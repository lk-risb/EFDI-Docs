# Data directory (`/mnt/data`)

`/mnt/data` is the JuiceFS mount from [03-juicefs](03-juicefs.md). It holds every
Docker stack and its persistent data. The stack definitions live in the **swarm**
repository under `data/` and must be copied into `/mnt/data`.

```
swarm/data/*   ->   /mnt/data/*
```

> [!IMPORTANT]
> The swarm repo contains **no secrets**. Private keys and certificates are
> git-ignored, and every password / setup key / encryption key in the committed
> configs is an upper-case named placeholder (e.g. `NETBIRD_DB_PASSWORD`,
> `NETBIRD_AUTH_SECRET`, `NETBIRD_SETUP_KEY_LTU_TELIA`).

---

## 1. Get the stack files

```bash
git clone SWARM_REPO_URL /tmp/swarm
sudo cp -r /tmp/swarm/data/. /mnt/data/
rm -r /tmp/swarm
```

Layout after copy:

| Path | Service |
| ---- | ------- |
| `/mnt/data/dockhand/` | Dockhand container-management UI (HTTPS, Postgres-backed) |
| `/mnt/data/netbird/` | Self-hosted NetBird control plane (`nbio.fairytail.eu`) |
| `/mnt/data/routers/telia/`, `/mnt/data/routers/backbone/` | NetBird transit routers |
| `/mnt/data/zenoh/zenoh1..3/` | Zenoh routers (mTLS, port 7447) |
| `/mnt/data/volumes/` | Runtime bind-mount data (created by the stacks) |

---

## 2. Dockhand

Files: `/mnt/data/dockhand/` — `docker-compose.yml`, `swarm.crt`, `swarm.key`.

### 2.1 Database

Dockhand uses the `dockhand` role and database created in
[02-postgres](02-postgres.md#4-dockhand-database).

### 2.2 Secrets to fill in

| File | Placeholder | Value |
| ---- | ----------- | ----- |
| `docker-compose.yml` | `DOCKHAND_DB_PASSWORD` (in `DATABASE_URL`) | The `dockhand` DB password set in 02-postgres |

### 2.3 TLS certificate

`swarm.crt` / `swarm.key` are the HTTPS server cert for the Dockhand UI, issued by
the **EFDI LTU Intermediate CA**. They are git-ignored, so copy them onto the host
manually into `/mnt/data/dockhand/`:

```bash
sudo chmod 600 /mnt/data/dockhand/swarm.key
sudo chmod 644 /mnt/data/dockhand/swarm.crt
```

If the cert is missing or expired, issue a new one from the CA (see the CA docs)
for the host name the UI is served on, then drop both files in place.

### 2.4 Deploy

```bash
cd /mnt/data/dockhand
docker compose up -d
```

UI: `https://HOST:3000`.

---

## 3. NetBird control plane

Files: `/mnt/data/netbird/` — `docker-compose.yml`, `config.yaml`, `dashboard.env`.
Depends on the three `netbird_*` databases from
[02-postgres](02-postgres.md#3-netbird-databases).

### 3.1 `config.yaml`

| Placeholder | Meaning | How to set |
| ----------- | ------- | ---------- |
| `NETBIRD_AUTH_SECRET` | Secret for the embedded IdP / OAuth2 | Generate a random 32+ char string (`openssl rand -hex 32`) |
| `NETBIRD_STORE_ENCRYPTION_KEY` | Encrypts secrets at rest in the store DB | Generate with `openssl rand -base64 32` — **never change after first start** |
| `NETBIRD_DB_PASSWORD` (3×, in the store / authStore / activityStore DSNs) | `netbird` DB password | From 02-postgres, `netbird` role |

> [!WARNING]
> `NETBIRD_AUTH_SECRET` and `NETBIRD_STORE_ENCRYPTION_KEY` must stay identical for
> the life of the deployment. Losing or rotating them invalidates stored
> peer/auth data.

### 3.2 `dashboard.env`

| Key | How to set |
| --- | ---------- |
| `AUTH_CLIENT_SECRET` | Leave empty — the embedded IdP uses a public client (`USE_AUTH0=false`) unless an external IdP is configured |

Endpoints and OIDC authority are already set to `https://nbio.fairytail.eu`.

### 3.3 `docker-compose.yml` (Traefik)

DNS for `nbio.fairytail.eu` must point at this host and ports `80`, `443`,
`3478/udp` must be reachable so ACME and STUN work.

### 3.4 Deploy

```bash
cd /mnt/data/netbird
docker compose up -d
```

First run: open `https://nbio.fairytail.eu`, create the admin account, then
generate **setup keys** for clients (used in [01-vpn](01-vpn.md)) and for the
transit routers below.

---

## 4. Transit routers (optional)

Files: `/mnt/data/routers/telia/`, `/mnt/data/routers/backbone/`.

Each container has an `NB_SETUP_KEY` placeholder — replace with a setup key
generated in the matching NetBird control plane:

| File | Container | Placeholder | Management URL | Setup key source |
| ---- | --------- | ----------- | -------------- | ---------------- |
| `routers/telia/docker-compose.yml` | `ltu` | `NETBIRD_SETUP_KEY_LTU_TELIA` | `nbio.fairytail.eu` | local NetBird (step 3) |
| `routers/telia/docker-compose.yml` | `telia` | `NETBIRD_SETUP_KEY_TELIA_LTU` | Azure NetBird | that network's admin |
| `routers/backbone/docker-compose.yml` | `ltu` | `NETBIRD_SETUP_KEY_LTU_BACKBONE` | `nbio.fairytail.eu` | local NetBird (step 3) |
| `routers/backbone/docker-compose.yml` | `backbone` | `NETBIRD_SETUP_KEY_BACKBONE_LTU` | `netbird.efdi-backbone.net` | that network's admin |

Deploy: `cd /mnt/data/routers/NAME && docker compose up -d`.

---

## 5. Zenoh routers (optional)

Files: `/mnt/data/zenoh/zenoh1..3/`. Each needs an mTLS key pair and the EFDI CA
cert in its `certs/` folder (git-ignored — copy manually):

| File | Purpose |
| ---- | ------- |
| `certs/efdi_ca.crt` (or `ca.crt`) | CA that verifies peer certs |
| `certs/NAME.crt` / `certs/NAME.key` | This router's client/server cert, issued by the EFDI CA |

Set permissions (`chmod 600` on `*.key`), then
`cd /mnt/data/zenoh/NAME && docker compose up -d`.
