# JuiceFS setup

JuiceFS provides the shared filesystem mounted at `/mnt/data`, backed by the
`filestore` PostgreSQL database (metadata) created in [02-postgres](02-postgres.md).

> [!IMPORTANT]
> Replace `<filestore-password>` below with the real `filestore` role password.
> The metadata database must be reachable before formatting or mounting.

## 1. Install

```bash
curl -sSL https://d.juicefs.com/install | sh -
sudo mkdir -p /mnt/data
```

## 2. Format the volume (run once)

This initializes the JuiceFS metadata in the `filestore` database. Run it only
once per volume.

```bash
juicefs format --storage postgres --bucket "localhost/filestore" --access-key filestore --secret-key <filestore-password> "postgres://filestore:<filestore-password>@localhost/filestore" filestore
```

## 3. Test mount

```bash
juicefs mount "postgres://filestore:<filestore-password>@localhost/filestore" /mnt/data
```

Verify with `df -h /mnt/data`, then stop it with `Ctrl+C` before setting up the
service.

## 4. Persistent mount via systemd

Create `/etc/systemd/system/juicefs.service`:

```ini
[Unit]
Description=JuiceFS Mount
Requires=postgresql.service
After=postgresql.service network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment="DSN=postgres://filestore:<filestore-password>@localhost/filestore?sslmode=disable"
ExecStart=/usr/local/bin/juicefs mount --writeback --cache-size=204800 --max-uploads=50 --no-usage-report ${DSN} /mnt/data -o allow_other,writeback_cache,max_read=99
ExecStop=/usr/local/bin/juicefs umount /mnt/data
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now juicefs.service
```

## 5. Verify

```bash
systemctl status juicefs.service
df -h /mnt/data
```
