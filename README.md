# ansible-role-caddy

Installs and configures the [Caddy](https://caddyserver.com/) reverse proxy on
Debian. Ships a generic Caddyfile template that turns a list of sites into
reverse-proxy blocks with optional IP allowlists, per-path restrictions, an
imported runtime blocklist, and a Prometheus metrics endpoint.

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
| `caddy_caddyfile_template` | `Caddyfile.j2` | Template to render. `""` skips deployment (caller manages the Caddyfile) |
| `caddy_sites` | `[]` | List of reverse-proxy sites (see below) |
| `caddy_snippets` | `{}` | Named reusable Caddy snippets (see below) |
| `caddy_acme_email` | `""` | ACME contact for Let's Encrypt |
| `caddy_systemd_env` | `{}` | Env vars injected via a systemd drop-in |
| `caddy_metrics_bind` | `""` | Bind address for the metrics server. `""` disables it |
| `caddy_metrics_port` | `9090` | Metrics server port |
| `caddy_metrics_allow_ips` | `[]` | IPs/CIDRs allowed to reach `/metrics` |

### `caddy_sites` entry schema

```yaml
caddy_sites:
  - host: "app.example.com"          # required — public hostname
    upstream: "10.0.0.5:8080"        # required — backend host:port
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
