# Home Server

A living project document for the Raspberry Pi 5 home-server build. This
README records the current hardware, operating system, Docker layout,
services, configuration decisions, troubleshooting notes, and remaining
implementation work.

> **Project status (2026-11-25):** 
> **Currently functional:*** PostgreSQL, and n8n.
> **Planned deployments:*** Nextcloud, Beszel (including Discord webhook 
> notifications), Home Assistant + Matter Server, Pi-hole, Cloudflare 
> Tunnel, and a formal backup process remain planned (Pi-hole deferred 
> to last).

------------------------------------------------------------------------

## 1. Hardware and operating system

  -----------------------------------------------------------------------
  Component           Current details      Planned deployment
  ------------------- -------------------- ------------------------------
  Host                Custom build         ThinkCentre M920q

  RAM                 256 GB               16 GB

  Architecture        x64                  x64

  Operating system    Omarchy OS           Proxmox

  OS storage          1tb M.2 NVMe         256 GB

  Data storage        4tb internal M.2     8 TB Usable (RAID 1)

  filesystem          Btrfs(1tb),Ext4(4tb) ZFS

  Data mnt point      `/mnt/4tb`           TBD

  label               `4tb`                TBD

  Docker Engine       29.9.2               TBD

  Docker Compose      5.6.1                TBD
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 2. Project goals

Run the following services on custom built test computer with future 
deployment on a dedicated 'server' M920q. Currently using Docker Compose 
and keeping resource use low. Potentially making a switch to K3 at a later 
time to allow for server redundancies and better uptime.

1.  **PostgreSQL** --- one shared database server for applications that
    support PostgreSQL.
2.  **n8n** --- workflow automation.
3.  **Nextcloud** --- personal cloud files and collaboration.
4.  **Home Assistant + Matter** --- local smart-home control (basic
    Wi-Fi Matter smart plugs).
5.  **Pi-hole** --- LAN DNS filtering.
6.  **Cloudflare Tunnel (`cloudflared`)** --- one outbound tunnel for
    securely publishing selected web apps without opening inbound router
    ports.
7.  **Monitoring** --- host and container metrics, temperatures,
    storage, and service availability (Beszel, with Discord notifications).
8.  **ollama** --- serve text embedding model for AI RAG - currently only 
    for n8n

The design favors separate Compose projects, persistent data on the HDD,
and private Docker networking where appropriate. Host networking is used
for Home Assistant and Matter Server (required for mDNS/device discovery).

------------------------------------------------------------------------

## 3. Directory layout

Compose project files live under `/mnt/4tb/gitproject/myserver`

``` text
~/
├── postgres/
│   ├── compose.yml
│   └── secrets/*.txt
├── n8n/
│   ├── compose.yml
│   ├── secrets/*.txt
│   └── local-files/
├── nextcloud/
│   ├── compose.yml
│   ├── secrets/*.txt
│   ├── html/
│   ├── data/
│   └── redis/
├── homeassistant/
│   ├── config/
│   ├── matter/
│   └── compose.yml
├── pihole/
├── cloudflared/
├── ollama/
│   └── compose.yml
└── beszel/
    └── compose.yml 
```

-------------------------------------------------------------------------

## 4. Docker networking

A defined Docker bridge network for postgres named `backup-shared-db-net` has 
been created (in production will remove 'backup' from name). Additional bridge 
network created for text embedding ai model, named 'ai-bridge':

``` bash
sudo docker network ls
sudo docker network inspect backup-shared-db-net
sudo docker network inspect ai-bridge
```

Its purpose is to let application containers reach shared services by
Docker network name, without publishing internal database
ports to the LAN.

Current network design:

-   `backup-shared-db-net`: PostgreSQL, n8n, Nextcloud, and Nextcloud Redis.
-   'ai-bridge': n8n, ollama
-   Host networking: Home Assistant and Matter Server (required for
    Matter/mDNS).
-   A separate `proxy` network is planned for `cloudflared` and web
    applications that should be reachable through the tunnel.
-   Pi-hole networking will be handled separately because DNS needs
    access to port 53 on the LAN.

PostgreSQL is not published to the host. Its port `5432` is available to
containers attached to `backend`.

------------------------------------------------------------------------

## 5. Deployed services

### 5.1 PostgreSQL

  Setting                    Value
  -------------------------- ------------------------------
  Compose project            `postgres`
  Container name             `backup-shared-postgres`
  Image                      `pgvector/pgvector:pg16`
  Version observed in logs   **need to confirm**
  Docker network             `backup-shared-db-net`
  Published host port        None **need to confirm, it may be exposed to LAN 
                             currently***
  Health                     Healthy at last verification


PostgreSQL's administrator account is `postgres`. Its password is stored
in `/~/myserver/postgres/secrets/postgres_password.txt`, which 
has mode `600`. **Do not put passwords in this README or commit `secrets/*.*` files 
to source control.**

#### Databases and roles

Two separate databases and least-privilege application roles have been
created:

  Application   Database      Owner / login role
  ------------- ------------- --------------------
  n8n           `n8n`         `n8n_user`
  Nextcloud     `nextcloud`   `nextcloud_user`

The role passwords were set interactively with PostgreSQL's `\password`
command. They are not recorded here.

Useful PostgreSQL administration commands:

``` bash
# Open a PostgreSQL shell
sudo docker exec -it postgres psql -U postgres

# From the psql prompt
\du
\l
\q
```

### 5.2 n8n

  Setting                     Value
  --------------------------- ----------------------------------
  Compose project             `backup-n8n` (will remove 'backup' when in production)
  Container name              `backup-n8n` (will remove 'backup' when in production)
  Image                       `docker.n8n.io/n8nio/n8n:latest`
  Web port                    `5450`
  Database                    PostgreSQL database `n8n`
  Database role               `n8n_user`
  Docker networks             `ai-bridge`, 'backup-shared-db-net'


The secrets are located here `/~/myserver/n8n/secrets/` contains:

-   `n8n-encryption-key.txt`
-   `postgres-n8n-password.txt`

These files are restricted to mode `600`:

``` bash
sudo chmod 600 *.txt
```

**Important:** Back up the n8n encryption key securely. Losing or
changing it can prevent n8n from decrypting stored credentials.

The n8n port is currently published on the host as `5450:5678`, for LAN
access. Do not expose it directly to the public internet.


### 5.3 Nextcloud - **not started yet**

  -----------------------------------------------------------------------
  Setting                             Value
  ----------------------------------- -----------------------------------
  Compose project                     `nextcloud`
  Container name                      `nextcloud`
  Image                               `nextcloud:apache`
  PHP version observed                PHP 8.5.12
  Web port                            Host `5480` → container `80`
  Persistent application directory    `nextcloud/html` →
                                      `/var/www/html`
  Persistent user data directory      `nextcloud/data` →
                                      `/var/www/html/data`
  Database                            Existing PostgreSQL database
                                      `nextcloud`
  Database host / role                `postgres` / `nextcloud_user`
  Redis container                     `nextcloud-redis`
  Docker network                      `backup-shared-db-net`
  -----------------------------------------------------------------------

Nextcloud reuses the existing PostgreSQL server and the `nextcloud`
database/`nextcloud_user` role. PostgreSQL is not separately exposed to
the LAN. A Redis container is included on the private `backend` network
for caching and transactional file locking.

Nextcloud will be reachable on the LAN at:

``` text
<myip>:5480
```

The Compose project uses a separate environment file at
`nextcloud/secrets/nextcloud-db-password.txt` for `NEXTCLOUD_DB_PASSWORD`. Keep it
restricted:

``` bash
sudo chmod 600 *.txt
```

### 5.4 Beszel

Beszel Hub and Agents are not yet deployed, including Discord webhook
notifications.

Current confirmed state:

-   Beszel Hub container: `beszel`
-   Hub image: `henrygd/beszel:latest`
-   Hub web interface: `<myip>:5490`
-   Notifications: webhook will be configured to send alerts to Discord.

Beszel provides host and Docker resource monitoring, including CPU,
memory, storage, network, and container metrics. Temperature and USB HDD
S.M.A.R.T. visibility depend on what the host kernel and USB enclosure
expose to the agent.

The exact Agent Compose configuration, credentials, and persistent host
paths should be documented from the live deployment. Do not record
Beszel tokens or private credentials in this README.


### 5.5 Home Assistant + Matter Server

  Setting                          Value
  -------------------------------- ---------------------------------------
  Compose project                  `homeassistant`
  Home Assistant container         `homeassistant`
  Home Assistant image             `ghcr.io/home-assistant/home-assistant:stable`
  Home Assistant web port          `5423` (host networking)
  Matter Server container          `matter-server`
  Matter Server image              `ghcr.io/matter-js/python-matter-server:stable`
  Matter Server port               `5580` (host networking)
  Persistent config directory      `/homeassistant/config`
  Persistent Matter data           `/homeassistant/matter`
  Docker network                   host (both containers)
  Time zone                        `America/Denver`
  Status                           pending

Home Assistant will be reachable on the LAN at:

``` text
<myip>:5423
```

Matter Server uses host networking for proper mDNS and device discovery.
The Matter integration in Home Assistant is configured to connect to the
local Matter Server WebSocket (typically `ws://127.0.0.1:5580/ws` or
`ws://<myip>:5580/ws`).

Intended use is basic automation of Wi-Fi Matter smart plugs. Thread
devices are not currently in scope (no OpenThread border router deployed).

Do not expose the Home Assistant or Matter Server interfaces directly to
the public internet.

------------------------------------------------------------------------

## 7. Planned services and architecture 
**need to confirm everything below, may be out of date - carried forward 
from old project**

### 7.1 Pi-hole

Planned Compose project: `~/myserver/pihole`

Deferred to last. Pi-hole will provide DNS filtering for devices on the
local network. DNS generally requires port `53` (TCP and UDP). 

Before changing router DNS settings:

1.  Confirm the Pi has a stable LAN address (DHCP reservation is already
    in place).
2.  Check for port 53 conflicts.
3.  Deploy Pi-hole and test DNS manually.
4.  Only then change the router or client DNS configuration.

If Pi-hole is also expected to provide DHCP, that requires additional
network and router planning.

### 7.2 Cloudflare Tunnel

Planned Compose project: `/~/myserver/cloudflared`

The goal is **one Cloudflare Tunnel** for multiple hostnames, rather
than one tunnel per application. For example:

-   `n8n.example.com` → n8n's internal Docker address and port.
-   `cloud.example.com` → Nextcloud's internal Docker address and port.

The exact hostnames and routes will be configured once the domain and
Cloudflare setup are confirmed.

The planned `proxy` Docker network will connect `cloudflared` to the web
application containers. PostgreSQL will remain on the private `backend`
network and will not be exposed through the tunnel. Home Assistant and
Matter Server are not planned for public exposure.

No inbound router port forwarding should be needed for the tunnel. Keep
tunnel credentials or tokens in a restricted environment/configuration
file, not in this README.

### 7.3 Backups

Planned. High priority before storing important data.

Goal: automated, tested backups of PostgreSQL databases (especially
`n8n` and `nextcloud`), critical configuration files, and the n8n
encryption key, stored on a separate device or remote destination.

Priority items:

1.  **PostgreSQL logical dumps** (`pg_dump`) of the `n8n` and
    `nextcloud` databases. n8n workflows, credentials, and execution
    history live in the database.
2.  **n8n encryption key** — required to decrypt stored credentials.
3.  **Nextcloud data directory and configuration**.
4.  **Home Assistant configuration**
    (`/~/myserver/homeassistant/config`).

### 7.4 Monitoring: Beszel

Monitoring is deployed with Beszel only. Notifications are configured
via webhook to Discord.

-   **Beszel** collects host and Docker resource metrics and can be used
    for service availability checks.

Beszel is intended to show:

-   Raspberry Pi CPU usage and load.
-   RAM and swap usage.
-   Disk capacity and disk I/O.
-   Network throughput.
-   Host temperature sensors, when exposed by the kernel.
-   Docker container status and per-container resource statistics.
-   Disk S.M.A.R.T. health where the drive and USB enclosure expose it.
-   Historical metrics and configurable alerts.

Beszel documentation: <https://www.beszel.dev/guide/what-is-beszel>

Monitoring dashboards are currently LAN-accessible. Decide separately
whether to publish them through Cloudflare Tunnel; exposing
administrative dashboards publicly is not recommended without a
deliberate security review.

**Hardware identification:** Monitoring dashboards can show live
resource metrics, but they may not automatically provide a complete
hardware inventory. This README records the Pi model, RAM, CPU
architecture, OS, and storage devices. A small host-inventory report can
be added if desired.

**Temperature caveat:** Beszel can report temperatures exposed through
Linux sensor interfaces. Verify that the Pi 5's temperature sensor is
visible to the agent. USB HDD S.M.A.R.T. passthrough depends on the USB
enclosure/adapter.

------------------------------------------------------------------------

## 8. Roadmap

-   [x] Install Docker Engine and Docker Compose plugin.
-   [x] Create Compose project directories.
-   [x] Create the Docker `backup-shared-db-net` network.
-   [x] Deploy PostgreSQL 16 and verify health.
-   [x] Create separate `n8n` and `nextcloud` databases and roles.
-   [x] Deploy n8n using PostgreSQL.
-   [x] Verify n8n initial account setup and normal operation.
-   [ ] Deploy Nextcloud using the shared PostgreSQL server.
-   [ ] Deploy Beszel Hub and Agent; add services for monitoring.
-   [ ] Deploy Cloudflared tunnel
-   [ ] Deploy Ollama
-   [ ] Create the Docker `ai-bridge` network.
-   [ ] Deploy Pihole
-   [ ] Configure Beszel notifications (Discord webhook).
-   [ ] Deploy Home Assistant Container + Matter Server for basic Wi-Fi
    Matter smart-plug automation.
-   [ ] Deploy Pi-hole; verify port 53 and stable LAN addressing first
    (deferred to last).
-   [ ] Configure one Cloudflare Tunnel for n8n and Nextcloud.
-   [ ] Configure and test backups for databases, app data, and
    encryption keys.
-   [ ] Review memory consumption and tune container limits based on
    observed usage.


------------------------------------------------------------------------



## 9. Security and backup notes

-   Keep `secrets/*.txt` files mode `600`.
-   Never paste passwords, encryption keys, Cloudflare tokens, or other
    secrets into this README.
-   Keep PostgreSQL off the public internet and off published host
    ports.
-   Publish only the web applications that actually need remote access,
    through Cloudflare Tunnel.
-   Keep monitoring dashboards and Home Assistant private unless there
    is a clear reason to expose them.
-   Use unique database credentials for each application.
-   Back up PostgreSQL and application data to a separate device or
    remote destination. A second folder on the same HDD is not a backup.
-   Back up n8n's encryption key along with its database.
-   Back up Nextcloud's database, data directory, and configuration
    consistently.
-   Back up Home Assistant config.

------------------------------------------------------------------------

## 11. Project notes / decisions to confirm

-   Confirm the final domain names and Cloudflare DNS arrangement before
    creating tunnel routes.
-   Confirm whether Pi-hole should provide DNS only or DNS plus DHCP
    (deferred to last).
-   Decide on an independent backup destination (another disk, NAS, or
    remote storage) and implement the backup process.
-   Confirm whether the USB enclosure supports S.M.A.R.T. passthrough.
-   Reassess whether the 1 TB HDD is sufficient for Nextcloud and
    database workloads or whether an SSD would be a worthwhile future
    upgrade.
-   Record Beszel Hub/Agent Compose paths, persistent storage paths, and
    the agent connection method.


------------------------------------------------------------------------

*Last updated: 2026-11-25*

