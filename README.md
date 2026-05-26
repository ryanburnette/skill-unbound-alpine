# skill-unbound-alpine

Set up and manage local Unbound DNS resolver on Alpine Linux. Use when installing, configuring, debugging, or hardening Unbound on Alpine, including OpenRC service management, FD limits, remote control, and local DNS records.

## Usage

This is an [Agent Skills](https://agentskills.io/) compatible skill. Load it with your agent harness and invoke via `skill:unbound-alpine`.

## Why run your own DNS resolver

A local Unbound resolver gives your network its own recursive DNS cache. Every device on the network queries Unbound, and Unbound resolves names by walking the DNS hierarchy from the root. This has real benefits:

- Local records: define DNS entries for internal services (printers, NAS, dashboards, development VMs) that only exist on your network. No external resolver knows about these hosts, and no public DNS zone is needed. Unbound serves them directly from `local-data` entries in its config.
- Caching: every answer Unbound resolves gets cached. Repeated lookups for the same domain return from cache instead of hitting the internet again. On a home or lab network with multiple devices hitting the same services, this cuts latency and reduces outbound DNS traffic.
- No third-party resolver: without a local resolver, your devices query whatever DNS server the DHCP hands out, usually your ISP's resolver or a public one like Google or Cloudflare. Those resolvers see every domain you look up. Running your own means no single third party has a log of every domain your network resolves. Unbound's `qname-minimisation` sends only the minimal part of the query needed at each level of the DNS hierarchy, reducing what any single authoritative server learns about you.
- ISP visibility: in Unbound's default recursive mode, queries to root and authoritative servers travel as plain text on UDP/TCP port 53. Your ISP can observe these queries if they inspect traffic. Running your own resolver does not hide DNS activity from the ISP. To encrypt outbound queries, configure Unbound in forwarding mode with `forward-tls-upstream: yes` (DNS over TLS) to an upstream resolver that supports DoT. The skill's config template uses recursive mode for full control and caching; add a `forward-zone` with `forward-tls-upstream: yes` if you want encrypted upstream queries instead.

## Structure

- `SKILL.md` -- Skill instructions and frontmatter
