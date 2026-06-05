---
name: unbound-alpine
description: Set up and manage local Unbound DNS resolver on Alpine Linux. Use when installing, configuring, debugging, or hardening Unbound on Alpine, including OpenRC service management, FD limits, remote control, and local DNS records.
---

## Install

```sh
apk add unbound unbound-openrc openssl
rc-update add unbound default
```

## OpenRC Configuration

Create `/etc/conf.d/unbound`:

```sh
# Configuration for /etc/init.d/unbound

# Path of the configuration file.
#cfgfile="/etc/unbound/$RC_SVCNAME.conf"

# Raise file descriptor limit — prevents FD exhaustion under load
rc_ulimit="-n 4096"

# Do NOT uncomment supervise-daemon. The init script uses
# command_background=yes, which is only compatible with
# start-stop-daemon (the default supervisor).
#supervisor=supervise-daemon
```

**Why `rc_ulimit="-n 4096"`:** The default soft limit is 1024. Unbound with
`outgoing-range` > 512 and `num-threads` > 1 can exhaust FDs, causing the
process to be killed by the OS and producing "unbound stopped by something else"
errors in syslog.

**Why not `supervise-daemon`:** OpenRC 0.62+ defaults to `supervise-daemon` for
crash recovery, but the Alpine init script (`/etc/init.d/unbound`) uses
`command_background=yes`, which only works with `start-stop-daemon`. If
`supervise-daemon` takes over, it cannot properly track the daemon PID, causing
restart failures and cascading crash loops on boot.

## Remote Control Setup

Unbound-control is required for `unbound-control reload` (used by logrotate),
runtime stats, and cache flushes. TLS certs must be generated.

```sh
unbound-control-setup -d /etc/unbound
```

This creates in `/etc/unbound/`:
- `unbound_server.key` / `unbound_server.pem`
- `unbound_control.key` / `unbound_control.pem`

Add to `unbound.conf` `remote-control:` section:

```
remote-control:
    control-enable: yes
    control-interface: 127.0.0.1
    control-port: 8953
    server-key-file: "/etc/unbound/unbound_server.key"
    server-cert-file: "/etc/unbound/unbound_server.pem"
    control-key-file: "/etc/unbound/unbound_control.key"
    control-cert-file: "/etc/unbound/unbound_control.pem"
```

Bind to `127.0.0.1` only — never `0.0.0.0` for remote control.

## Configuration Template

```
server:
    verbosity: 1
    num-threads: 1
    interface: 0.0.0.0
    port: 53
    outgoing-range: 64
    so-rcvbuf: 425984
    so-sndbuf: 425984
    msg-cache-size: 16m
    num-queries-per-thread: 512
    rrset-cache-size: 32m
    cache-max-ttl: 86400
    infra-host-ttl: 60
    do-ip4: yes
    do-ip6: no
    do-udp: yes
    do-tcp: yes
    access-control: 127.0.0.0/7 allow
    access-control: 10.0.0.0/8 allow
    access-control: 172.16.0.0/12 allow
    access-control: 192.168.0.0/16 allow
    username: unbound
    directory: "/etc/unbound"
    use-syslog: no
    log-queries: yes
    log-replies: yes
    logfile: "/var/log/unbound.log"
    hide-version: yes
    qname-minimisation: yes
    auto-trust-anchor-file: "/var/lib/unbound/root.key"

remote-control:
    control-enable: yes
    control-interface: 127.0.0.1
    control-port: 8953
    server-key-file: "/etc/unbound/unbound_server.key"
    server-cert-file: "/etc/unbound/unbound_server.pem"
    control-key-file: "/etc/unbound/unbound_control.key"
    control-cert-file: "/etc/unbound/unbound_control.pem"
```

## Tuning for Small VMs (<= 256MB RAM)

- `num-threads: 1` — threading overhead exceeds benefit on 1-2 vCPU containers
- `outgoing-range: 64` — sufficient for home/LAB DNS; 512 causes FD exhaustion
- `msg-cache-size: 16m` / `rrset-cache-size: 32m` — rrset should be 2x msg-cache
- `num-queries-per-thread: 512` — halved from default to match reduced cache

For larger VMs (>= 1GB RAM), increase:
- `num-threads: 2` (match vCPU count)
- `outgoing-range: 256`
- `msg-cache-size: 64m` / `rrset-cache-size: 128m`
- `rc_ulimit="-n 8192"` in `/etc/conf.d/unbound`

## Local Records

Add local DNS records under `server:`:

```
    domain-insecure: "example.net"
    local-zone: "example.net" nodefault
    local-data: "unifi.example.net. IN A 192.168.1.1"
    local-data: "pve.example.net. IN A 192.168.5.63"
```

- `domain-insecure` disables DNSSEC for the zone (required if the zone is
  internal-only and has no real DNSSEC)
- `local-zone ... nodefault` prevents Unbound from treating the zone as a
  static fallback; allows forwarding to an upstream resolver for names not
  defined in `local-data`
- Trailing dots on names (`host.example.net.`) are recommended for clarity but
  not required — Unbound treats `local-data` names as absolute either way

### Short Names Don't Resolve at Unbound

`local-data` entries must use FQDNs (`pve.example.net`, not `pve`). Unbound
will not append search domains to bare hostnames — that's a client-side
function. Tools like `dig` send the literal query, so `dig @unbound pve`
returns nothing.

Short names work on clients that support search domains (macOS, Linux with
`search example.net` in `/etc/resolv.conf`) because the OS appends the domain
before querying. `dscacheutil -q host -a name pve` on macOS resolves correctly
for this reason.

If you need short names to resolve directly at Unbound (e.g., for IoT devices
or tools that don't use search domains), add separate `local-data` entries:

```
    local-data: "pve IN A 192.168.5.63"
    local-data: "tlsrouter IN A 192.168.10.252"
```

This duplicates records, so the default approach is to rely on client-side
search domains and only add bare records if a device requires it.

## Stub Zones (Forward to Upstream DNS)

Forward a zone to another resolver (e.g., a UniFi gateway):

```
stub-zone:
    name: "example.net"
    stub-host: 192.168.1.1
    stub-first: yes
```

`stub-first: yes` means stub is queried before the internet roots — required
when the upstream holds records that don't exist in public DNS.

`unbound-checkconf` will warn about `stub-host` being an IP; this is harmless
and expected for internal resolvers.

## Logrotate

Alpine's unbound package installs `/etc/logrotate.d/unbound`:

```
/var/log/unbound.log {
    daily
    maxage 180
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    create 0644 unbound unbound
    postrotate
        /usr/sbin/unbound-control reload > /dev/null 2>&1 || true
    endscript
}
```

The `postrotate` script requires `control-enable: yes`. Without it, logrotate
runs but the reload silently fails.

To disable query logging (reduce log volume on stable setups):

```
    log-queries: no
    log-replies: no
```

## Service Management

```sh
# Validate config before restarting
unbound-checkconf /etc/unbound/unbound.conf

# Restart (always use rc-service, not kill -HUP)
rc-service unbound restart

# Check status
rc-service unbound status
unbound-control status

# Reload config without restart (after unbound.conf changes)
unbound-control reload

# Flush cache
unbound-control flush_zone example.net

# View stats
unbound-control stats

# List local zones
unbound-control list_local_zones
```

## Debugging Boot Crashes

Symptom: Unbound is "started" per OpenRC but DNS doesn't work, or syslog shows
"unbound stopped by something else" repeated.

Check in order:

1. **FD exhaustion:**
   ```sh
   cat /proc/$(cat /run/unbound.pid)/limits | grep "Max open"
   ls /proc/$(cat /run/unbound.pid)/fd | wc -l
   ```
   If FDs used approaches the soft limit, raise `rc_ulimit` and/or reduce
   `outgoing-range`.

2. **Supervisor conflict:**
   ```sh
   grep supervisor /etc/conf.d/unbound
   ps -ef | grep supervise-daemon
   ```
   Ensure `supervisor=supervise-daemon` is NOT set in `/etc/conf.d/unbound`.

3. **Config syntax:**
   ```sh
   unbound-checkconf /etc/unbound/unbound.conf
   ```

4. **Syslog errors:**
   ```sh
   grep unbound /var/log/messages | grep -E "ERROR|WARNING|stop"
   ```

5. **OOM (Proxmox LXC):**
   ```sh
   cat /sys/fs/cgroup/memory.events   # cgroup v2
   cat /sys/fs/cgroup/memory/memory.failcnt 2>/dev/null   # cgroup v1
   free -m
   ```

## Access Control

Only allow RFC 1918 and loopback networks. Never use `access-control: 0.0.0.0/0 allow`.

```
    access-control: 127.0.0.0/7 allow
    access-control: 10.0.0.0/8 allow
    access-control: 172.16.0.0/12 allow
    access-control: 192.168.0.0/16 allow
```

## Proxmox LXC Notes

- Unbound in LXC shares the host kernel; `dmesg` is unavailable from inside the
  container
- Check `memory.events` in cgroup v2 for OOM kills
- `so-rcvbuf` / `so-sndbuf` values may be capped by `net.core.rmem_max` on the
  host — if Unbound logs socket buffer warnings, raise on the Proxmox host:
  ```sh
  sysctl -w net.core.rmem_max=8388608
  sysctl -w net.core.wmem_max=8388608
  ```
