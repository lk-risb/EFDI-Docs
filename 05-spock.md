# Spock replication

Multi-master logical replication between the two swarm Postgres nodes using
[pgEdge Spock](https://docs.pgedge.com/spock_ext/).  
Built on the Postgres install from [02-postgres](02-postgres.md); nodes talk to
each other over the WireGuard tunnel from [01-vpn](01-vpn.md).

| Node | WireGuard IP |
| ---- | ----------- |
| `swarm1` | `10.10.0.1` |
| `swarm2` | `10.10.0.2` |

> [!IMPORTANT]
> `SPOCK_DB_PASSWORD` is a placeholder — use the real `spock` role password. Steps 1–6 run **on both nodes**; steps 7 onward are
> per-node as noted.

---

## 1. Install the Spock extension (both nodes)

```bash
sudo apt-get install -y pgedge-postgresql-18-spock50
```

## 2. Postgres configuration (both nodes)

Edit `/etc/postgresql/18/main/postgresql.conf`:

```conf
listen_addresses = '0.0.0.0'

wal_level = logical

max_worker_processes = 16
max_logical_replication_workers = 8

max_wal_senders = 10
max_replication_slots = 10
track_commit_timestamp = on

# PostgreSQL 18
max_active_replication_origins = 10
shared_preload_libraries = 'spock'
output_plugin_libraries = 'pgoutput, test_decoding, spock_output'
```

Check for config errors before restarting:

```bash
sudo -u postgres psql -c "SELECT sourcefile, name, sourceline, error FROM pg_file_settings WHERE error IS NOT NULL;"
```

Restart and verify the key settings applied:

```bash
sudo systemctl restart postgresql@18-main
sudo -u postgres psql -c "SHOW wal_level;"
sudo -u postgres psql -c "SHOW shared_preload_libraries;"
sudo -u postgres psql -c "SHOW max_active_replication_origins;"
sudo -u postgres psql -c "SHOW output_plugin_libraries;"
```

## 3. Access rules (both nodes)

The two nodes must be able to reach each other's Postgres as the `spock` role.
Edit `/etc/postgresql/18/main/pg_hba.conf` — add the **other** node's tunnel IP:

```conf
# on swarm1:
host    all    spock    10.10.0.2/32    scram-sha-256
# on swarm2:
host    all    spock    10.10.0.1/32    scram-sha-256
```

Reload after editing: `sudo systemctl reload postgresql@18-main`.

## 4. Create the replication role (both nodes)

```bash
sudo -u postgres psql -c "CREATE ROLE spock WITH LOGIN SUPERUSER REPLICATION PASSWORD 'SPOCK_DB_PASSWORD';"
```

---

> [!TIP]
> Steps 1–4 (OS packages, Postgres config, `pg_hba.conf`, the `spock` role) must
> be done over SSH. From step 5 on, everything is `spock.*` SQL — you can do it
> from **Spock Tower** (section 11) instead of the `psql` commands below. Install
> it now and follow the same steps in its UI.

## 5. Enable the extension in each replicated database (both nodes)

Run for every database that needs replication (here `postgres`):

```bash
sudo -u postgres psql -d postgres -c "CREATE EXTENSION IF NOT EXISTS spock;"
```

## 6. Initial data sync (optional, when bootstrapping a fresh node)

Spock replicates changes, not existing rows. If the databases are not already
identical, dump from the source node and load on the target:

```bash
# on the source node
sudo -u postgres pg_dumpall > postgres_full.sql
# copy the file over, then on the target node
sudo -u postgres psql -f postgres_full.sql
```

---

## 7. Register the nodes

Run each `node_create` **on its own node**. The local node needs no password
(peer/local auth); remote DSNs include the password.

```bash
# on swarm1
sudo -u postgres psql -c "SELECT spock.node_create(node_name := 'swarm1', dsn := 'host=10.10.0.1 port=5432 dbname=postgres user=spock password=SPOCK_DB_PASSWORD');"

# on swarm2
sudo -u postgres psql -c "SELECT spock.node_create(node_name := 'swarm2', dsn := 'host=10.10.0.2 port=5432 dbname=postgres user=spock password=SPOCK_DB_PASSWORD');"
```

## 8. Add tables to the replication set (both nodes)

```bash
sudo -u postgres psql -c "SELECT spock.repset_add_all_tables('default', ARRAY['public']);"
```

Re-run this after creating new tables, or manage sets from Spock Tower (below).

## 9. Create subscriptions

One subscription per direction. Each is created **on the node that receives** the
changes, pointing at the other node as provider.

```bash
# on swarm1 — receive from swarm2
sudo -u postgres psql -c "SELECT spock.sub_create(subscription_name := 'sub_swarm1_swarm2', provider_dsn := 'host=10.10.0.2 port=5432 dbname=postgres user=spock password=SPOCK_DB_PASSWORD');"

# on swarm2 — receive from swarm1 (multi-master)
sudo -u postgres psql -c "SELECT spock.sub_create(subscription_name := 'sub_swarm2_swarm1', provider_dsn := 'host=10.10.0.1 port=5432 dbname=postgres user=spock password=SPOCK_DB_PASSWORD');"
```

Wait for each to finish its initial sync:

```bash
sudo -u postgres psql -c "SELECT spock.sub_wait_for_sync('sub_swarm1_swarm2');"
```

## 10. Finalise and check

```bash
sudo -u postgres psql -c "GRANT USAGE ON SCHEMA spock TO spock;"
sudo -u postgres psql -c "SELECT * FROM spock.sub_show_status();"
```

`sub_show_status()` should report `replicating` for every subscription.

---

## 11. Spock Tower (recommended for day-to-day management)

Once steps 1–4 are done, **Spock Tower** can drive everything from step 5 onward —
enabling the extension, registering nodes, replication sets, subscriptions and
status — and it stays the recommended tool for ongoing management (schema compare,
config dump, subscription monitoring). Prefer it over running `spock.*` functions
by hand.

Install on a management host (or a swarm node):

```bash
git clone SPOCK_TOWER_REPO_URL /opt/spock_tower
cd /opt/spock_tower
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
cp config.yml config.yml.bak
```

Edit `config.yml` — list each node under `databases:` (host, port, `dbname`,
`user`, `password`) and set the `server` host/port. Then run:

```bash
cd /opt/spock_tower
.venv/bin/python3 app.py
```

UI listens on `server.host`/`server.port` from `config.yml` (default
`0.0.0.0:8080`). See the repo's `README.md` for details.
