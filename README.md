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
- For the test suite: Docker, plus the pins in `requirements-test.txt`. See
  [Testing](#testing).

## Role variables

See [`defaults/main.yml`](defaults/main.yml) for the full list. The important
ones:

| Variable | Default | Purpose |
| --- | --- | --- |
| `caddy_install_method` | `apt` | `apt` or `xcaddy` |
| `caddy_apt_package_state` | `present` | `latest` to let a run upgrade the packaged Caddy |
| `caddy_apt_repo_url` | cloudsmith | Base URL of Caddy's apt repository |
| `caddy_extra_modules` | `[]` | Module import paths, only used with `xcaddy` |
| `caddy_version` | `""` | Caddy git tag to build. Empty builds whatever is latest — pin it for anything serving traffic |
| `caddy_go_version` | `"1.27.1"` | Go toolchain for `xcaddy` builds. Bumping it replaces an installed toolchain |
| `caddy_go_checksum` | `""` | Optional `sha256:…` for the Go tarball |
| `caddy_go_arch_map` | amd64/arm64 | `ansible_facts.architecture` → Go release architecture |
| `caddy_xcaddy_version` | `latest` | xcaddy version used as the build tool |
| `caddy_build_dir` | `/usr/local/src/caddy` | Staging binary + `build.stamp` for the rebuild decision |
| `caddy_build_workspace` | `<caddy_build_dir>/workspace` | Disposable Go caches and build files; removed after every build attempt |
| `caddy_xcaddy_env` | `{}` | Extra env for the `xcaddy` build (e.g. `GOPRIVATE` for private modules) |
| `caddy_caddyfile_template` | `Caddyfile.j2` | Template to render. `""` skips deployment (caller manages the Caddyfile) |
| `caddy_sites` | `[]` | List of reverse-proxy sites (see below) |
| `caddy_snippets` | `{}` | Named reusable Caddy snippets (see below) |
| `caddy_acme_email` | `""` | ACME contact for Let's Encrypt |
| `caddy_acme_dns` | `""` | Global DNS-01 provider + args (`acme_dns <value>`). Needed for wildcard certs and split-horizon/internal HTTPS |
| `caddy_global_extra` | `""` | Raw directives injected verbatim into the global options block (e.g. a clustered `storage` backend) |
| `caddy_systemd_env` | `{}` | Env vars injected via a systemd drop-in |
| `caddy_metrics_bind` | `""` | Bind address for the metrics server. `""` disables it |
| `caddy_metrics_port` | `9090` | Metrics server port |
| `caddy_metrics_allow_ips` | `[]` | IPs/CIDRs allowed to reach `/metrics` |

### `caddy_sites` entry schema

```yaml
caddy_sites:
  - host: "app.example.com"          # required — public hostname
    upstream: "10.0.0.5:8080"        # required — backend host:port, space-separated for several
    upstream_tls_skip_verify: true    # optional — proxy over HTTPS to a self-signed upstream
    lb_policy: round_robin            # optional — policy across multiple upstreams
    lb_retries: 2                     # optional — retries against other upstreams
    health_uri: /healthz              # optional — enables ACTIVE health checks
    health_interval: 10s              # optional — probe frequency (default 30s)
    health_timeout: 3s                # optional — per-probe timeout
    health_port: 9000                 # optional — probe another port than the upstream's
    health_status: 2xx                # optional — acceptable status (default 200)
    health_body: "OK"                 # optional — regexp the probe body must match
    fail_duration: 30s                # optional — how long a failed upstream stays out
    max_fails: 2                      # optional — failures within fail_duration before removal
    unhealthy_status: [5xx]           # optional — statuses counted as a failure
    unhealthy_latency: 2s             # optional — a slower response counts as a failure
    unhealthy_request_count: 20       # optional — concurrent requests that mean unhealthy
    lb_try_duration: 5s               # optional — how long to keep retrying other upstreams
    lb_try_interval: 250ms            # optional — wait between those retries
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

- `allow_ips` on the site → the whole site is locked to those clients. It is
  rendered as a deny for everyone else, so everything else declared on the site —
  `protected_paths`, imports, `extra_directives`, load balancing and health
  checks — still applies to the clients that are allowed in.
- `upstream_tls_skip_verify: true` → the upstream is spoken to over HTTPS with a
  `transport http { tls_insecure_skip_verify }` block, so Caddy accepts a
  self-signed or hostname-mismatched backend cert (e.g. Proxmox on `:8006`, PBS
  on `:8007`). Omit it (default `false`) for the plain `reverse_proxy`.
- `lb_*` / `health_*` / `fail_duration` / `max_fails` / `unhealthy_*` → emitted
  verbatim inside the `reverse_proxy` block, so they are Caddy's own semantics
  and Caddy's own defaults. They fall into three groups:
  - **Load balancing** — `lb_policy`, plus how hard to work at a failing
    request: `lb_retries` (retry count), or `lb_try_duration` /
    `lb_try_interval` (retry window and spacing).
  - **Active health checks** — `health_uri` is the one worth setting
    deliberately: without it Caddy only learns an upstream is down by failing a
    real request at it, so the first user after a backend dies eats the error.
    With it Caddy polls out of band and pulls the backend before that happens.
    `health_port`, `health_interval`, `health_timeout`, `health_status` and
    `health_body` shape the probe.
  - **Passive health checks** — judged from live traffic instead of probes.
    `fail_duration` is the base: how long an upstream that failed stays out of
    rotation, and Caddy's default of `0` means passive checks are off entirely.
    `max_fails` sets how many failures within that window it takes, and
    `unhealthy_status`, `unhealthy_latency` and `unhealthy_request_count` decide
    what counts as a failure in the first place.
- `access_log: true` → the role pre-creates `{{ caddy_log_dir }}/<host>.access.log`
  as `caddy:caddy` mode `0600` before reload and configures the site to write JSON
  access logs there. Characters that cannot appear in a filename are replaced with
  `_`, so a site addressed as `http://app.example.com` or `:8080` still gets a
  usable path.
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

## Building with xcaddy: pinning and rebuilds

The build is expensive and it replaces the binary that serves live traffic, so it
is driven by a stamp rather than run on every play. `{{ caddy_build_dir }}/build.stamp`
records the version, module list and Go toolchain the installed binary came from,
and the build only re-runs when the request no longer matches it:

```yaml
caddy_install_method: xcaddy
caddy_version: v2.11.4                     # pin it
caddy_go_version: "1.26.7"
caddy_extra_modules:
  - github.com/pberkel/caddy-storage-redis
```

Editing any of those three triggers exactly one rebuild; running the playbook
again does not. Leaving `caddy_version` empty means "whatever xcaddy resolves as
latest **at build time**", which is fine for a lab and a bad idea for an edge
node: the next unrelated deploy that happens to rebuild will move Caddy forward
and restart the service to install it.

Bumping `caddy_go_version` replaces an installed toolchain rather than being
ignored, so the pin is real on hosts that already have Go.

## Secrets and file modes

Two of this role's inputs are routinely secret — `caddy_global_extra` (a storage
backend password) and `caddy_systemd_env` (ACME provider tokens) — so the files
they land in are not world-readable:

| Path | Owner | Mode |
| --- | --- | --- |
| `{{ caddy_config_dir }}` | `root:caddy` | `0750` |
| `{{ caddy_caddyfile }}` | `root:caddy` | `0640` |
| `caddy.service.d/override.conf` | `root:caddy` | `0640` |
| `{{ caddy_data_dir }}` (certs, ACME keys) | `caddy:caddy` | `0750` |
| `<host>.access.log` | `caddy:caddy` | `0600` |

Neither install path prints those variables into the journal. The unit this role
installs on the `xcaddy` path runs Caddy without `--environ`; on the `apt` path
the unit belongs to the package and does carry the flag, so when
`caddy_systemd_env` is non-empty the drop-in resets `ExecStart` and redefines it
without it. Set `caddy_systemd_hide_environ: false` to keep the package's
invocation verbatim — worth doing if a future package revision adds a flag to
`ExecStart` that matters to you.

## Tags

- `install` — repo/binary, user, directories, systemd unit + drop-in, service
- `config` — render and (validated) deploy the Caddyfile, reload on change

Preflight assertions (install method, modules present for an `xcaddy` build,
mapped architecture) carry both tags, so neither entry point can run against an
unusable combination of inputs.

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

Served over plain HTTP on the given bind address, so a scraper needs no
certificate handling. An empty `caddy_metrics_allow_ips` exposes it to anything
that can reach the bind address.

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

## Testing

Two Molecule scenarios run against Debian 13 in Docker, both ending in an
idempotence check:

| Scenario | Covers |
| --- | --- |
| `default` | the `apt` path, and the full site matrix — allowlists, per-path rules, snippets, load balancing, health checks, access logs, metrics, and that the drop-in keeps its variables out of the journal |
| `xcaddy` | the build path: a pinned Caddy + module, the rebuild stamp, the role-owned unit, and certificates stored in Redis rather than on disk |

```sh
python3 -m venv .venv && . .venv/bin/activate
python3 -m pip install -r requirements-test.txt

ansible-lint                 # production profile
molecule test                # default scenario
molecule test -s xcaddy      # build path (downloads a Go toolchain, compiles Caddy)
molecule test --all          # what CI runs
```

The `default` scenario addresses its sites as `http://…` and sets `auto_https off`
so the container never reaches for a certificate: Caddy puts TLS on any site whose
address carries a port other than 80, and a plain request to that listener is
answered with 400.

The `xcaddy` scenario does issue certificates, through `local_certs` and a Redis
store, and asserts they land in Redis with nothing left under the data directory.
Real ACME needs a publicly reachable address, so the issuer differs from
production while the storage path — the part that lets two nodes behind one
floating IP share a certificate store — is the same. What a single container
cannot show is the second node reading that store, or a floating IP moving
between them; the VIP belongs to keepalived, not to this role.

---

In this repo the role is consumed by `playbooks/caddy/main.yaml`, which keeps
all jjstreams-specific data (the `caddy_sites` list, the admin-panel blocklist
file + apply wrapper, the forced-command SSH key) in the **playbook**, not the
role — so the role stays reusable across hosts.
