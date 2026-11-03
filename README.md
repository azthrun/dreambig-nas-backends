# dreambig-nas-backends

PostgreSQL, Redis and RabbitMQ as one Container Manager Project on the NAS.

Target: DS220+ (Intel, `linux/amd64`), DSM 7.4.1, Container Manager (Docker Engine 24), 2 GB RAM.

| Service | Image | Host port(s) | Memory cap |
| --- | --- | --- | --- |
| PostgreSQL | `postgres:18.6-trixie` | 5432 | 512 MB |
| Redis | `redis:8.10.3-alpine` | 6379 | 256 MB |
| RabbitMQ | `rabbitmq:4.4.0-management-alpine` | 5672 (AMQP), 15672 (UI) | 768 MB |

Versions are the latest stable releases as of 2026-11-03 (PostgreSQL 19 is still a release candidate). All three publish `linux/amd64` images.

## Memory

The caps (512 + 256 + 768 MB) are limits, not reservations: on a 2 GB NAS they only stop one service from starving the others. Redis logs a "Memory overcommit must be enabled" warning at start; that is expected. If Redis later logs failed AOF rewrites or `BGSAVE` (fork errors), enable overcommit at every boot:

1. DSM > **Control Panel** > **Task Scheduler** > **Create** > **Triggered Task** > **User-defined script**.
2. **General**: user `root`, event **Boot-up**.
3. **Task Settings** > **Run command**: `sysctl -w vm.overcommit_memory=1`. **OK**, then select the task and **Run** once (or reboot).

## Setup

1. Create the data folders over SSH, not File Station: File Station folders carry Synology ACLs, which break PostgreSQL's and RabbitMQ's owner-only permissions.
   - DSM > **Control Panel** > **Terminal & SNMP** > **Enable SSH service** > **Apply**.
   - From the Mac: `ssh <admin-user>@<nas-ip>`, then:
     ```bash
     sudo mkdir -p /volume1/docker/dreambig-nas-backends/{postgres,redis,rabbitmq}
     sudo /usr/syno/bin/synoacltool -del /volume1/docker/dreambig-nas-backends/postgres
     sudo /usr/syno/bin/synoacltool -del /volume1/docker/dreambig-nas-backends/redis
     sudo /usr/syno/bin/synoacltool -del /volume1/docker/dreambig-nas-backends/rabbitmq
     ```
     `synoacltool -del` removes the ACL so normal Linux modes apply. Never open or put anything in these three folders via File Station (it adds `@eaDir`, and Redis then skips fixing `/data` ownership).
2. Copy `compose.yaml`, the `rabbitmq` folder (with `10-nas.conf`) and your `.env` into `docker/dreambig-nas-backends`. Make `.env` from `.env.example`; replace every `<password-...>` with the output of `openssl rand -hex 24`.
3. **Container Manager > Project > Create**: name `dreambig-nas-backends`, path `/docker/dreambig-nas-backends`, source **Use existing docker-compose.yml**. Next, skip Web Station, **Done**.
4. **Container Manager > Container**: all three containers show **healthy** after about a minute.
5. Check from a Mac on the LAN (replace `<nas-ip>`):
   - `nc -vz <nas-ip> 5432`, `nc -vz <nas-ip> 6379`, `nc -vz <nas-ip> 5672` all succeed.
   - `http://<nas-ip>:15672` shows the RabbitMQ sign-in; sign in with `RABBITMQ_USER` / `RABBITMQ_PASSWORD`.

## Troubleshooting

- "port is already allocated": another DSM package uses that port. Change its `*_HOST_PORT` in `.env` and rebuild the Project.
- "data directory has invalid permissions" (PostgreSQL) or "Cookie file must be accessible by owner only" (RabbitMQ): the data folder has a Synology ACL, and the paths the container created inside it (`18/docker`, `.erlang.cookie`) inherited it. **Project > Action > Stop**, strip the ACL from the folder and everything in it, e.g. `sudo find /volume1/docker/dreambig-nas-backends/postgres -exec /usr/syno/bin/synoacltool -del {} \;` (or `.../rabbitmq`; errors for paths that already have no ACL are harmless). For RabbitMQ, also `sudo rm /volume1/docker/dreambig-nas-backends/rabbitmq/.erlang.cookie`: stripping the ACL leaves the cookie's old mode bits and the entrypoint never chmods it; RabbitMQ regenerates it owner-only on next start (safe on this single node). Then **Start**. If nothing is stored yet, you can instead empty the folder and recreate it per Setup step 1.

## Connecting

- PostgreSQL: `postgresql://<POSTGRES_USER>:<POSTGRES_PASSWORD>@<nas-ip>:5432/<POSTGRES_DB>`
- Redis: `redis://:<REDIS_PASSWORD>@<nas-ip>:6379/0`
- RabbitMQ: `amqp://<RABBITMQ_USER>:<RABBITMQ_PASSWORD>@<nas-ip>:5672/%2F` (URL-encode the vhost; `/` is `%2F`)

## Security

Every port is published on all NAS interfaces, protected only by passwords, in plain text (no TLS). That is fine on the trusted home LAN and over Tailscale (WireGuard-encrypted). Never forward these ports on the router. DSM's firewall may not filter ports that Container Manager publishes, so do not count on it.

## Backups

The data folders are live database files; a Hyper Backup copy taken while PostgreSQL runs may not restore cleanly. For PostgreSQL, back up a dump instead (over SSH on the NAS; network clients always need the password, the socket inside the container does not). DSM's Docker socket is root-only, so `docker` needs `sudo`; the `>` redirect still runs as your admin user:

```bash
sudo docker exec dreambig-nas-backends-postgres-1 pg_dump -U <POSTGRES_USER> -Fc <POSTGRES_DB> > /volume1/docker/dreambig-nas-backends/backup.dump
```

## Upgrading

Edit the image tag in `compose.yaml`, then **Project > Action > Build**. Patch versions (18.6 → 18.7) are drop-in. A PostgreSQL major version (18 → 19) cannot read the old data folder: dump, start the new version on an empty folder, restore. RabbitMQ: upgrade one minor at a time (4.4 → 4.5); after each upgrade, once healthy, run `sudo docker exec dreambig-nas-backends-rabbitmq-1 rabbitmqctl enable_feature_flag all` (otherwise the next minor refuses to start).
