# sensors-ansible

Ansible deployment for the LilLittleSensors Raspberry Pi fleet:

- **rpi4-db** — PostgreSQL server (implemented)
- **rpi4-api** — nginx + FastAPI via uv-managed Python (implemented)
- **rpi4-app** — Grafana + uv-managed Python + standalone systemd services/timers (implemented)

## Connectivity model

- SSH as **root**, password auth (no SSH keys).
- **No sudo anywhere.** The one exception: the PostgreSQL role/db/extension
  tasks in `roles/postgres/tasks/roles_db.yml` use Ansible `become` with
  `become_method: su` (not sudo) to run as the OS `postgres` user, since
  `community.postgresql.*` modules need peer auth over the local socket.
  Root can `su` to any user without a password, so this needs no sudo
  package or config.
- Secrets (root SSH password, Postgres `api` user password) are stored in
  an `ansible-vault`-encrypted file, never in plaintext.
- The control node (this Mac) connects via Ansible's `paramiko` plugin so
  password auth works without needing the `sshpass` CLI installed.

## Setup

```bash
# 1. Install the required collection
ansible-galaxy collection install -r requirements.yml

# 2. Create the vault files (see the vault.yml.example files for the keys they need)
ansible-vault create inventory/group_vars/all/vault.yml
#   vault_root_ssh_password: "<actual root SSH password>"
#   vault_postgres_api_password: "<actual password for the Postgres 'api' role>"

ansible-vault create inventory/group_vars/api/vault.yml
#   vault_api_secret_key: "<random secret for the app's confirm-email token signer>"
#   vault_dyson_serial / vault_dyson_credentials / vault_dyson_product_type / vault_dyson_name / vault_dyson_version

ansible-vault create inventory/group_vars/app/vault.yml
#   vault_emporia_username / vault_emporia_password
#   vault_tradfri_psk
#   vault_kasa_username / vault_kasa_password (only if your Kasa devices need cloud auth)
#   vault_smtp_user / vault_smtp_password (shared Gmail app password for all four
#     apps' manual/cron --sendemail reports; unrelated to --push-api)
```

`inventory/hosts.yml` has `rpi4-db` (`10.0.0.11`) and `rpi4-api`
(`10.0.0.13`) pinned to their real static IPs; `rpi4-app`'s entry is a
placeholder (`10.0.0.16`) -- confirm/update it before running `--limit app`.

**Before running `--limit app`**, fill in the `TODO`-marked placeholders
in `inventory/group_vars/app/vars.yml` (your actual timezone, TRADFRI
gateway IP, Kasa broadcast address, `sensors_email_to`, and the
`rtl_postgres_repo` URL once that repo exists) and
`inventory/group_vars/app/vault.yml`. Ring also needs a one-time
interactive `--auth` step -- see below. Note that `--sendemail`/`--json`
runs are always available manually or via your own cron job; only the
`--push-api` mode is wired into the ansible-managed timers, so a blank
`vault_smtp_user`/`vault_smtp_password` doesn't block the scheduled runs
-- it only blocks the email report path until you fill it in.

**Before the first `--limit api` run**, the Alembic baseline must already
be stamped against rpi4-db's real `api` database -- this is a one-time
manual step, not part of the playbook (see `sensors-backend-fastapi`'s
README): `DATABASE_URL=<real url> SECRET_KEY=<real key> uv run alembic stamp head`.

## Running

```bash
# rpi4-db only
ansible-playbook site.yml --limit db --ask-vault-pass

# rpi4-api only (needs rpi4-db already provisioned, and the Alembic
# baseline stamped -- see above)
ansible-playbook site.yml --limit api --ask-vault-pass

# rpi4-app only
ansible-playbook site.yml --limit app --ask-vault-pass

# rpi4-db + rpi4-api together
ansible-playbook site.yml --limit db,api --ask-vault-pass

# everything currently defined
ansible-playbook site.yml --ask-vault-pass
```

## What the rpi4-db run does

- **common role**: installs and enables `fail2ban` (with an `sshd` jail
  enabled -- Debian ships none enabled by default) and `chrony` as the NTP
  client (default public pool).
- **postgres role**:
  - Adds the official PGDG apt repository for the host's Debian release
    (`trixie-pgdg` on Debian 13) using the current deb822 `.sources` format.
  - Installs PostgreSQL (version pinned via `postgres_major_version`,
    currently 18) plus `postgresql-contrib` (needed for the `citext`
    extension) and `python3-psycopg2`.
  - Sets `listen_addresses` to `localhost` plus the host's own private IP
    on `10.0.0.0/24` (derived from facts, not `0.0.0.0`).
  - Configures `pg_hba.conf` to allow `scram-sha-256` connections from
    `10.0.0.0/24` (for rpi4-api/rpi4-app later) plus loopback.
  - Creates the `api` role (`LOGIN`, `SUPERUSER`, vaulted password) and the
    `api` database, idempotently -- it will not touch an existing database,
    so a `pg_restore` done afterwards is safe on repeat playbook runs.
  - Creates the `citext` extension inside the `api` database.

## What the rpi4-api run does

- **common role**: same as above (`fail2ban` + `chrony`).
- **api role**:
  - Creates an unprivileged `sensors-api` system user/group (`nologin`
    shell) that owns the app code and runs the service -- ansible itself
    still manages the host as root, but the network-facing app doesn't.
  - Installs `uv` system-wide (`/usr/local/bin`, via the official
    installer script, idempotent).
  - Adds the official `nginx.org` apt repository for the host's Debian
    release and installs `nginx` (deb822 format, same pattern as the
    PGDG repo in the postgres role) -- **not** Debian's bundled nginx
    package. nginx.org's package uses `/etc/nginx/conf.d/*.conf`, not
    Debian's `sites-available`/`sites-enabled` convention.
  - Deploys an nginx reverse-proxy site listening on `0.0.0.0:80`,
    proxying to `127.0.0.1:8000` (open to the whole network, not
    restricted to `10.0.0.0/24` like Postgres -- no TLS in this pass).
  - `git clone`/pulls `sensors-backend-fastapi` (public GitHub repo) into
    `/srv/sensors-api/app`, runs `uv sync --frozen` to install
    dependencies (provisions the pinned Python version automatically),
    templates the app's environment file to `/etc/sensors-api/api.env`
    (root-owned, `0640`, readable by the `sensors-api` group), and runs
    `uv run alembic upgrade head` -- safe/idempotent, but only *after*
    the one-time `alembic stamp head` baseline (see above) has been run
    against the real database.
  - The `git clone`/`uv sync`/`alembic upgrade` tasks run as
    `sensors-api` via `become_method: su` with `become_flags: '-s /bin/sh'`
    (same deliberate "no sudo" exception as the postgres role's
    `roles_db.yml`, needed here because `sensors-api`'s `nologin` shell
    would otherwise block `su -c` from running anything).
  - Deploys a systemd unit (`sensors-api.service`) running a single
    `uvicorn` worker bound to `127.0.0.1:8000`, `Restart=always`.

## What the rpi4-app run does

- **common role**: same as above (`fail2ban` + `chrony`).
- **app role**:
  - Creates a shared unprivileged `sensors-app` system user/group
    (`nologin` shell) for all five ingestion apps. Grafana is deliberately
    **not** folded into this user -- it keeps its own `grafana` account
    (created by Grafana's own official package), since merging them risks
    permission fights with Grafana's package management over
    `/var/lib/grafana` etc.
  - Installs `uv` system-wide, same as the api role.
  - **Grafana**: checks first (`dpkg-query`) and only adds the official
    apt repo + installs the package if it's genuinely missing -- it's
    already manually installed and working here, so this just makes it
    ansible-managed going forward (enabled+started) without risking an
    apt-triggered reinstall disturbing what's there. No dashboard
    import/provisioning in this pass.
  - **Four timer-based apps** (`emporia`, `kasa`, `ikea`, `ring`, driven by
    the `sensor_apps` list in `vars.yml`): `git clone`/pulls each repo into
    `/srv/sensors-app/<name>`, creates a `uv venv` + `uv pip install -r
    requirements.txt` (none of these repos have a `pyproject.toml`, so
    `uv sync` doesn't apply), templates that app's `.env`, and deploys a
    systemd **timer + oneshot service** pair
    (`sensors-app-<name>.timer`/`.service`) running
    `<entrypoint> --push-api` on its own schedule (`item.interval` --
    2-15 min depending on the app's API cost). Each `--push-api` mode
    POSTs current readings to `sensors-backend-fastapi` and skips the
    report/chart/email output those scripts also support for manual/cron
    use.
  - **rtl_433**: built from source (`cmake`, not an apt package, to track
    latest per this project's convention) into `/usr/local`. The role
    prints the Acurite protocol IDs for the built binary directly in the
    ansible output (`rtl_433 -R help | grep -i acurite`) since numeric IDs
    shift between versions -- copy them into `rtl433_protocols` in
    `vars.yml` and re-run to narrow reception to just Tower/Atlas (empty
    list = receive everything, the safe default until you've set this).
  - **rtl-postgres**: same clone+venv treatment as the four timer apps,
    but deployed as a **persistent** systemd service (`rtl433-postgres`,
    `Restart=always`), not a timer -- its `ExecStart` is a shell pipeline,
    `rtl_433 -F json [-R ...] | rtl_2_postgres.py`, continuously streaming
    readings into `POST /v1/thermohygrometers/{id}/log` (which dedups
    server-side, skipping unchanged consecutive readings). `sensors-app`
    is added to the `plugdev` group for USB SDR device access (the
    apt-installed `rtl-sdr` package's udev rules grant that group access;
    no need to run any of this as root).
  - The `git clone`/`uv venv`/`uv pip install` tasks use the same
    `become_method: su`, `become_flags: '-l -s /bin/sh'` pattern as the
    api role (`nologin` shell needs an explicit shell override; `-l`
    resets `HOME` so `uv` doesn't try to write into `/root`).

**Two apps need a one-time manual step before their scheduled runs can
work unattended** (same precedent as the Alembic baseline stamp -- deploy
first, then do this by hand, then re-run):

- **Ring**: interactive 2FA login. SSH into rpi4-app, `cd
  /srv/sensors-app/ring && .venv/bin/python Ring_Energy.py --auth` with a
  real TTY. Ansible preserves the resulting `.ring_token.cache` file
  rather than regenerating it.
- **IKEA/TRÅDFRI**: pairing. Ansible can't mint the PSK itself since it
  needs a live call to the gateway using the one-time sticker security
  code. After the first `--limit app` deploy (even with `vault_tradfri_psk`
  still blank), SSH in and run:
  ```bash
  cd /srv/sensors-app/ikea
  TRADFRI_HOST=<gateway-ip> TRADFRI_SECURITY_CODE=<sticker-code> .venv/bin/python ikea_energy.py --pair
  ```
  Put the printed PSK into `vault_tradfri_psk` (the security code itself
  doesn't need to go in the vault -- it's single-use), then re-run the
  playbook so the templated `.env` picks up the real PSK.
- **Tailscale** (rpi4-app only): the `tailscale` role installs and enables
  `tailscaled`, but doesn't authenticate the node -- ansible has no
  authkey to work with (deliberately, to avoid a long-lived secret in the
  vault). After a `--limit app` deploy, SSH in and run `tailscale up`,
  then approve the printed login link in a browser. This is truly
  one-time: `tailscaled` persists its node identity in
  `/var/lib/tailscale/` and reconnects automatically on every reboot from
  then on, with no further action needed.

## ufw

`roles/common` installs `ufw` and adds rules on every host (SSH plus each
host's own service port -- 5432 on db, 80 on api, 3000/Grafana on app),
scoped to `ufw_trusted_cidrs` (`inventory/group_vars/all/vars.yml`) and,
for each host's own service port only (never SSH), the `tailscale0`
interface once that's up. **It deliberately never runs `ufw enable`** --
review what's staged, then flip it on yourself:

```bash
ufw show added      # rules staged, not yet enforced
ufw status verbose  # will read "Status: inactive" until you enable it
ufw enable          # your call, once you're satisfied
```

## Verification

On rpi4-db:

```bash
ss -tlnp | grep 5432                    # listening on loopback + 10.0.0.11 only
su postgres -c "psql -c '\du'"           # role "api" shows Superuser
su postgres -c "psql -c '\l'"             # database "api" owned by api
su postgres -c "psql -d api -c '\dx'"      # extension "citext" present
apt-cache policy postgresql-18            # origin should be apt.postgresql.org
systemctl status postgresql fail2ban chrony
fail2ban-client status sshd
chronyc tracking
```

On rpi4-api:

```bash
systemctl status sensors-api nginx
journalctl -u sensors-api -n 50
curl -s http://127.0.0.1/v1/aqsensors        # through nginx
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/docs
ss -tlnp | grep -E ':(80|8000)'              # 8000 on 127.0.0.1 only, 80 on 0.0.0.0
```

From another host on the network: `curl http://10.0.0.13/v1/aqsensors`
should succeed; `curl http://10.0.0.13:8000/...` should fail/refuse
(uvicorn isn't directly reachable, only through nginx).

From another host on `10.0.0.0/24`:

```bash
psql -h 10.0.0.11 -U api -d api -W
```

On rpi4-app:

```bash
systemctl status sensors-app-emporia.timer sensors-app-kasa.timer \
  sensors-app-ikea.timer sensors-app-ring.timer
systemctl status rtl433-postgres grafana-server
journalctl -u rtl433-postgres -n 50
# force a timer-based app to run right now instead of waiting for its interval:
systemctl start sensors-app-kasa.service
journalctl -u sensors-app-kasa -n 30
```

From another host: `curl -X POST http://10.0.0.13/v1/energy-circuits/kasa/test-1/log -d '{"name":"Test","watts":42}' -H 'Content-Type: application/json'`
should land in Postgres; posting the same body to
`/v1/thermohygrometers/{id}/log` twice should only create one log row
(dedup working).
