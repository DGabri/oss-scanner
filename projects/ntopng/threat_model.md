# Threat model

## What this project does and where untrusted input enters
- ntopng is a network traffic monitoring tool written in C++ (core, `src/`, `include/`) with a Lua scripting layer
  (`scripts/lua/`) and a web UI (Lua pages + Vue.js, `httpdocs/`). It embeds the nDPI deep packet inspection library.
- Untrusted input, in decreasing order of importance:
  1. Network traffic: packets from capture interfaces, and flow records from NetFlow/sFlow/IPFIX/ZMQ collectors
     (parsers for DNS, HTTP, TLS, SSH, SNMP, Modbus, S7comm, Profinet, etc.). Any remote host can send crafted
     traffic that ntopng will parse.
  2. HTTP requests to the embedded web server (CivetWeb): REST API (`scripts/lua/rest/`), Lua pages, parameter
     handling, authentication/session/CSRF logic.
  3. Data from external services: SNMP devices, syslog/Wazuh/Suricata alert feeds, LDAP/RADIUS responses, DNS names.
  4. Uploaded files and imported configurations (pcap, JSON config, backups).

## Components that matter most / least
- Most: packet/flow parsers in C++, the HTTP/REST layer and authentication, input sanitization in Lua code that
  builds shell commands, SQL (SQLite, MySQL, ClickHouse) or file paths, and anything reachable without authentication.
- Less: the admin-only configuration pages, and CLI tooling under `tools/`.
- Out of scope: `third-party/`, `httpdocs/dist/`, `tests/`, `attic/`, `contrib/`, and bugs that are only in nDPI
  itself (report those to the nDPI project).

## How to exercise it
- Binary: `/src/ntopng`. It needs a running Redis server (`redis-server`) and can read a pcap with `-i file.pcap`.
- The web UI is served on port 3000; default credentials are admin/admin.
- The Lua REST API is reachable at `/lua/rest/v2/...` once the process is running.

## How you rate severity
- Memory corruption (buffer overflow, use-after-free, double free) reachable from network traffic or an
  unauthenticated request: high at minimum; critical if it plausibly leads to remote code execution.
- Authentication bypass, unauthenticated command/SQL injection or path traversal: critical.
- Post-authentication command injection or SQL injection by a low-privilege (non-admin) user: high. By an admin
  user it is medium, since admins can already change system settings.
- Stored/reflected XSS: medium; high if it can be triggered by network traffic content (e.g. a hostname or HTTP
  header shown in the UI) without any user interaction beyond viewing the page.
- CSRF on state-changing endpoints: medium.
- Denial of service: medium, unless a single unauthenticated packet or request crashes the process (high).
- Null dereferences and assertion failures without a security impact beyond a crash: low.

## Anything to leave alone
- Do not report missing hardening flags or the default admin/admin password.
- Do not report issues that require local root or write access to the ntopng data directory.
