# ansible-role-caddy

Installs and configures the [Caddy](https://caddyserver.com/) reverse proxy on
Debian. Ships a generic Caddyfile template that turns a list of sites into
reverse-proxy blocks with optional IP allowlists, per-path restrictions, an
imported runtime blocklist, a Prometheus metrics endpoint, and multi-backend
load balancing with health checks.

Two install paths:

- **`apt`** (default) — Caddy's official cloudsmith repo. Covers everything the
  bundled HTTP handlers do (`reverse_proxy`, `metrics`, `file_server`,
  `respond`, `remote_ip`, …).
- **`xcaddy`** — builds a custom binary when you need third-party modules
  (e.g. DNS providers for ACME-DNS, `cache-handler`, geoip). Opt in with
  `caddy_install_method: xcaddy` + `caddy_extra_modules`.

## Requirements

- Debian (trixie). Needs `community.general` and `ansible.posix` collections
  for the calling playbook's surrounding tasks (the role itself only uses
  builtins).
- For `xcaddy`: outbound access to `go.dev` and GitHub to fetch the toolchain
  and modules.

## Role variables

See [`defaults/main.yml`](defaults/main.yml) for the full list. The important
ones:

| Variable | Default | Purpose |
| --- | --- | --- |
| `caddy_install_method` | `apt` | `apt` or `xcaddy` |
| `caddy_extra_modules` | `[]` | Module import paths, only used with `xcaddy` |
| `caddy_go_version` | `"1.22"` | Go toolchain for `xcaddy` builds |
| `caddy_xcaddy_env` | `{}` | Extra env for the `xcaddy` build (e.g. `GOPRIVATE` for private modules) |
| `caddy_caddyfile_template` | `Caddyfile.j2` | Template to render. `""` skips deployment (caller manages the Caddyfile) |
| `caddy_sites` | `[]` | List of reverse-proxy sites (see below) |
| `caddy_snippets` | `{}` | Named reusable Caddy snippets (see below) |
| `caddy_acme_email` | `""` | ACME contact for Let's Encrypt |
| `caddy_acme_dns` | `""` | Global DNS-01 provider + args (`acme_dns <value>`). Needed for wildcard certs and split-horizon/internal HTTPS |
| `caddy_systemd_env` | `{}` | Env vars injected via a systemd drop-in |
| `caddy_metrics_bind` | `""` | Bind address for the metrics server. `""` disables it |
| `caddy_metrics_port` | `9090` | Metrics server port |
| `caddy_metrics_allow_ips` | `[]` | IPs/CIDRs allowed to reach `/metrics` |

### `caddy_sites` entry schema

```yaml
caddy_sites:
  - host: "app.example.com"          # required — public hostname
    upstream: "10.0.0.5:8080"        # required unless `upstreams` is set
    upstream_tls_skip_verify: true    # optional — proxy over HTTPS to a self-signed upstream
    access_log: true                  # optional — JSON access log under caddy_log_dir
    allow_ips:                        # optional — whole-site allowlist (403 otherwise)
      - 203.0.113.10
      - 10.0.0.0/24
    protected_paths:                  # optional — per-path allowlists
      - path: "/admin/*"
        allow_ips:
          - 203.0.113.10
    import_snippets:                  # optional — pull in named caddy_snippets
      - security_headers
      - compression
    import_files:                     # optional — import external files by path
      - /etc/caddy/blocklist.caddy
    extra_directives: |               # optional — raw Caddyfile, injected verbatim
      header /healthz Cache-Control "no-store"
```

- `allow_ips` on the site → the whole site is locked to those clients.
- `upstream_tls_skip_verify: true` → the upstream is spoken to over HTTPS with a
  `transport http { tls_insecure_skip_verify }` block, so Caddy accepts a
  self-signed or hostname-mismatched backend cert (e.g. Proxmox on `:8006`, PBS
  on `:8007`). Omit it (default `false`) for the plain `reverse_proxy`.

### Multiple backends, load balancing & health checks

```yaml
caddy_sites:
  - host: "app.example.com"
    upstreams:                        # optional — list of backends; overrides `upstream`
      - "10.0.0.10:8080"
      - "10.0.0.11:8080"
      - "10.0.0.12:8080"
    lb_policy: round_robin            # optional — see policies below
    lb_try_duration: "5s"             # optional — total retry window on failure
    lb_try_interval: "250ms"          # optional — delay between retries
    health_check:                     # optional — active + passive health checks
      uri: /healthz                   # active: path Caddy polls periodically
      interval: 10s
      timeout: 5s
      expect_status: 200
      expect_body: "OK"
      fail_duration: 30s              # passive: count failures on live traffic
      max_fails: 2
      unhealthy_status: [500, 503]
      unhealthy_latency: 1s
      unhealthy_request_count: 1
```

- `upstreams` → a list of `host:port` backends. When set (non-empty), it's used
  instead of `upstream`; a single `upstream` still works unchanged for the
  common one-backend case, so nothing above needs to migrate.
- `lb_policy` → written verbatim as `lb_policy <value>`, so any policy Caddy
  supports works, including:
  - `round_robin` — Caddy's default; only need to set this explicitly to be
    explicit about it.
  - `random`, `random_choose <n>` — spread load randomly.
  - `least_conn` — send to the backend with the fewest active requests.
  - `first` — always prefer the first available backend (hot standby).
  - `ip_hash` — sticky by client IP: the same client keeps hitting the same
    backend as long as it stays healthy.
  - `uri_hash` — sticky by request URI (useful for caching proxies).
  - `header <field>` — sticky by a request header value.
  - `cookie [<name> [<secret>]]` — sticky sessions via cookie: Caddy sets a
    cookie on the first response and routes that client to the same backend
    for the life of the session. Use this when the app keeps in-memory state
    per backend (websockets, server-side sessions) and can't tolerate a
    client bouncing between instances.
- `lb_try_duration` / `lb_try_interval` → control how long/often Caddy retries
  a request against another backend after a failure, before giving up.
- `health_check.{uri,port,interval,timeout,expect_status,expect_body}` →
  **active** health checks: Caddy polls each backend on its own schedule and
  takes it out of rotation on failure, independent of live traffic.
- `health_check.{fail_duration,max_fails,unhealthy_status,unhealthy_latency,unhealthy_request_count}`
  → **passive** health checks: Caddy watches live request outcomes and
  temporarily marks a backend unhealthy after it crosses the given
  thresholds. Cheaper than active checks (no extra polling traffic) and
  usually enough unless you need to catch a dead backend before real traffic
  hits it.
- All of the above are optional and independent — set only what you need.
  `upstreams` with none of the load-balancing/health-check fields renders a
  plain `reverse_proxy a b c` line, letting Caddy round-robin with its
  built-in defaults.
- `access_log: true` → the role pre-creates `{{ caddy_log_dir }}/<host>.access.log`
  as `caddy:caddy` before reload and configures the site to write JSON access
  logs there.
- `protected_paths` → only those path prefixes are locked; the rest stays open.
  Each `path` is expanded to match `X`, `X/`, and `X/*`.
- `import_snippets` → emits `import <name>` for each listed snippet (defined in
  `caddy_snippets`). The clean way to share `header`/`encode`/etc. across sites.
- `import_files` → emits `import <path>` for each file. Use this for a snippet
  whose *content* is owned by another process at runtime (e.g. a panel that
  writes a per-user 403 blocklist and reloads caddy). Ship a default-empty file
  from your playbook, or `caddy validate` fails on the missing import.
- `extra_directives` → raw Caddyfile lines injected verbatim into the site block,
  just before `reverse_proxy`. A last-resort escape hatch for anything the schema
  doesn't model (ad-hoc `handle` blocks, one-off `header` rules, …).

### Reusable snippets (`caddy_snippets`)

Define named blocks once, `import` them from any site. They render as Caddy
`(name) { … }` snippets at the top of the Caddyfile. Two ready-to-use examples:

```yaml
caddy_snippets:
  security_headers: |
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
        Referrer-Policy strict-origin-when-cross-origin
        -Server
    }
  compression: |
    encode zstd gzip

caddy_sites:
  - host: "app.example.com"
    upstream: "127.0.0.1:8080"
    import_snippets: [security_headers, compression]
```

### Named IP allowlist groups (caller-side pattern)

`allow_ips` always takes a plain list — the role has no concept of "groups". But
because the value is just a list, you can keep one canonical set of IPs in a var
and reference it by name everywhere, instead of duplicating the same IPs across
sites and paths. Define the groups (in a `vars/` file, group_vars, or inline)
and reference them with Jinja:

```yaml
# vars/allowlists.yml
caddy_allow_groups:
  admins:
    - 203.0.113.10   # alice
    - 203.0.113.11   # bob
  office:
    - 198.51.100.0/24

# playbook
caddy_sites:
  - host: "admin.example.com"
    upstream: "127.0.0.1:8080"
    allow_ips: "{{ caddy_allow_groups.admins }}"
  - host: "intra.example.com"
    upstream: "127.0.0.1:9000"
    # combine groups (or add one-off IPs) with the + operator
    allow_ips: "{{ caddy_allow_groups.admins + caddy_allow_groups.office }}"
```

Add a person in one place; every site referencing the group picks it up. This is
how the jjstreams playbook in this repo factors out its shared admin allowlist
(`playbooks/caddy/vars/allowlists.yml`).

## Tags

- `install` — repo/binary, user, directories, systemd unit + drop-in, service
- `config` — render and (validated) deploy the Caddyfile, reload on change

## Example: minimal reverse proxy

```yaml
- hosts: web
  become: true
  vars:
    caddy_acme_email: "admin@example.com"
    caddy_sites:
      - host: "app.example.com"
        upstream: "127.0.0.1:8080"
  roles:
    - ansible-role-caddy
```

## Example: with a Prometheus metrics endpoint

```yaml
- hosts: web
  become: true
  vars:
    caddy_metrics_bind: "10.0.0.2"
    caddy_metrics_allow_ips: ["10.0.0.0/24"]
    caddy_sites:
      - host: "app.example.com"
        upstream: "127.0.0.1:8080"
  roles:
    - ansible-role-caddy
```

## Example: load-balanced app with sticky sessions

```yaml
- hosts: web
  become: true
  vars:
    caddy_sites:
      - host: "app.example.com"
        upstreams:
          - "10.0.0.10:8080"
          - "10.0.0.11:8080"
          - "10.0.0.12:8080"
        lb_policy: "cookie app_session"   # sticky sessions via cookie
        health_check:
          uri: /healthz
          interval: 10s
          timeout: 5s
          expect_status: 200
          fail_duration: 30s
          max_fails: 2
  roles:
    - ansible-role-caddy
```

Renders as:

```caddy
app.example.com {
    reverse_proxy 10.0.0.10:8080 10.0.0.11:8080 10.0.0.12:8080 {
        lb_policy cookie app_session
        health_uri /healthz
        health_interval 10s
        health_timeout 5s
        health_status 200
        fail_duration 30s
        max_fails 2
    }
}
```

## Example: custom binary with a DNS module

```yaml
- hosts: web
  become: true
  vars:
    caddy_install_method: xcaddy
    caddy_extra_modules:
      - github.com/caddy-dns/cloudflare
    caddy_sites:
      - host: "app.example.com"
        upstream: "127.0.0.1:8080"
  roles:
    - ansible-role-caddy
```

## Example: wildcard / DNS-01 certificates (split-horizon)

Use the DNS-01 challenge when the certificate hostnames don't (or can't) resolve
to this server publicly — wildcards, or an internal Caddy whose public A record
points at an external ingress while an internal resolver overrides it to the LAN.
`caddy_acme_dns` sets the challenge provider once, globally, so every site
inherits it; no per-site `tls` block needed.

The DNS provider module must be compiled in (`xcaddy`), and its API token passed
through the systemd environment:

```yaml
- hosts: web
  become: true
  vars:
    caddy_install_method: xcaddy
    caddy_extra_modules:
      - github.com/caddy-dns/cloudflare      # swap for your provider
    caddy_systemd_env:
      CF_API_TOKEN: "{{ vault_cf_token }}"    # keep in ansible-vault
    caddy_acme_email: "admin@example.com"
    caddy_acme_dns: "cloudflare {env.CF_API_TOKEN}"
    caddy_sites:
      - host: "*.home.example.com"           # wildcard now works
        upstream: "127.0.0.1:8080"
  roles:
    - ansible-role-caddy
```

Renders as the global block below; the token is read from the service
environment at runtime, never written into the Caddyfile:

```caddy
{
    email admin@example.com
    acme_dns cloudflare {env.CF_API_TOKEN}
}
```

## Using your own Caddyfile

If the generic template doesn't fit, point the role at your own template
(relative to the calling playbook's `templates/`, or an absolute path):

```yaml
caddy_caddyfile_template: my-Caddyfile.j2
```

or set it to `""` and write `/etc/caddy/Caddyfile` yourself in the playbook —
the role still handles install, the systemd drop-in, and the service.

---

In this repo the role is consumed by `playbooks/caddy/main.yaml`, which keeps
all jjstreams-specific data (the `caddy_sites` list, the admin-panel blocklist
file + apply wrapper, the forced-command SSH key) in the **playbook**, not the
role — so the role stays reusable across hosts.
