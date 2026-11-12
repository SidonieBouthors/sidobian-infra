# Immich with photo storage on a NAS

Immich runs with Docker Compose on the server. The photos live on the NAS, mounted over NFS.
Everything that is performance-sensitive or can be regenerated stays on the server's local disk.

| Data                                       | Location                 | Why                                              |
| ------------------------------------------ | ------------------------ | ------------------------------------------------ |
| `library`, `upload`, `profile`, `backups`  | NAS (`/mnt/immich/...`)  | Photos, videos and DB dumps (heavy, irreplaceable) |
| `thumbs`, `encoded-video`                  | server (`./library/...`) | Fast access, Immich can regenerate them          |
| PostgreSQL (`DB_DATA_LOCATION=./postgres`) | server                   | Immich doesn't support the database on a network share |
| ML model cache (`model-cache` volume)      | server                   | Local cache, re-downloaded if lost               |

## NAS

- Static IP on the NAS and on the server. The NAS grants NFS access by client IP, so the server must keep its address.
- Shared folder `immich`, exported over NFS at `/volume1/immich` (NFSv4.1 max).
- NFS rule: read/write for the server's IP only.
- Squash: all users are mapped to `admin`, so files belong to `admin` on the NAS whatever UID writes. No permission problems.

## Server

Mount the share:

```bash
sudo apt install nfs-common
sudo mkdir -p /mnt/immich
```

Add to `/etc/fstab` (`_netdev` waits for the network, `nofail` lets the server boot without the NAS, `x-systemd.automount` mounts on first access):

```
<NAS_IP>:/volume1/immich  /mnt/immich  nfs4  _netdev,nofail,x-systemd.automount  0  0
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart remote-fs.target
ls /mnt/immich
findmnt /mnt/immich    # should show <NAS_IP>:/volume1/immich, nfs4
```

In `docker-compose.yml`, `immich-server` keeps the standard `/data` mount (local) and overlays the four NAS folders on top. `.env` is standard (`UPLOAD_LOCATION=./library`, `DB_DATA_LOCATION=./postgres`), with `IMMICH_VERSION` pinned to an exact version (see Updating).

```yaml
    volumes:
      - ${UPLOAD_LOCATION}:/data
      - type: bind
        source: /mnt/immich/library
        target: /data/library
        bind:
          create_host_path: false
      # ...same block for upload, profile and backups
      - /etc/localtime:/etc/localtime:ro
```

`create_host_path: false` is a safety net: if the NAS isn't mounted, the container refuses to start instead of creating empty folders on the server's local disk.

Check: `docker exec immich_server df -h /data/library` should show the NAS.

## Rules

- Never modify files in `/mnt/immich` by hand. Immich keeps file paths in its database and doesn't rescan the folder, so manual changes desynchronise the two.
- After the NAS reboots or is updated, run `docker compose restart` so the containers pick up the NFS mount again.
- If the NAS is off, `immich-server` refuses to start (see `create_host_path: false` above). Power the NAS on, check that `findmnt /mnt/immich` shows it, then run `docker compose up -d`.

## Backups

- **Database:** Immich dumps its database daily at 2:00 AM and keeps the last 14 in `/mnt/immich/backups`. These contain metadata only, no photos.
- **Photos: TO DO.** Set up a second copy, ideally off-site, of `library`, `upload`, `profile` and `backups`, then document it here (tool, destination, schedule, how to restore).

## Updating Immich

The version is pinned in `.env` (`IMMICH_VERSION=v3.4.1`), so nothing upgrades by itself.

1. Read the release notes for breaking changes.
2. Trigger a manual database dump: Administration > Job Queues > Create job > Create Database Dump.
3. Change `IMMICH_VERSION` in `.env`.
4. `docker compose pull && docker compose up -d`, then check the logs and the web UI.

Use a separate admin account for administration and a normal account for photos.

## Rebuilding after losing the server

1. Reinstall Docker and `nfs-common`, recreate `/mnt/immich`, restore the fstab line and mount.
2. Restore the compose file and `.env` (ask admin for `DB_PASSWORD`, it is stored outside the server in case of loss), including the NAS overlay lines.
3. `docker compose up -d` with an empty `./postgres`.
4. In Immich: Administration > Maintenance > Restore database backup, and pick the latest dump.
5. If thumbnails or encoded videos are missing, regenerate them from Administration > Job Queues.

If the NAS itself is lost, restore from the second copy once it exists, then follow the steps above.