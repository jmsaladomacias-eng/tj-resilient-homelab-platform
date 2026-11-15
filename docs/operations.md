# Operations and Troubleshooting

## Container roles and storage mounts

The supplied Docker output showed these services running at capture time: Plex, qBittorrent, Radarr, Sonarr, Overseerr, Prowlarr, FlareSolverr, Uptime Kuma and Portainer.

| Service | Storage mapping observed | Operational role |
|---|---|---|
| Plex | Pool movies to `/movies`; TV library to `/tv`; `/dev/shm` to `/transcode` | Media library and playback |
| qBittorrent | Pool to `/data` | Download client |
| Radarr | Pool to `/data` | Movie workflow management |
| Sonarr | Pool to `/data` | Series workflow management |

The application configuration directories were separate from media storage in the captured mappings. Exact host-specific configuration, IPs, credentials, user IDs and private service endpoints are intentionally excluded.

## Selected troubleshooting evidence

### SQLite and storage I/O

The TJ-015 history describes high I/O delay after application databases and small SQLite writes were placed on a mechanical disk over NFS. The documented response was to keep application configuration/database data on the Ubuntu VM's local SSD and keep large media/download data on the HDD-backed storage. The note reports I/O delay falling from a measured 20–45% range to below 1% and lower guest CPU/iowait. These are historical measurements from that intervention, not a current performance benchmark.

### MergerFS-backed media workflow

TJ-018 records a Sonarr/qBittorrent import for a series and a Radarr/qBittorrent workflow for a movie, with Plex visibility/playback. For selected files, matching inode/link-count evidence confirmed physical hardlinks between download and library paths. The 11 November movie test confirmed playback and a hardlink for the tested 2160p file. These tests are specific examples; they do not prove all imports or branch combinations create hardlinks.

The same record leaves cold-boot/restart validation pending and documents an unresolved systemd verification warning. Radarr's metadata lookup timeout later recovered, but the cause was not established.

### Plex stale network file handle

Plex previously reported a transcoder error followed by a file-read error; the container showed `StaleNetworkFileHandle` (errno 116). Ubuntu and Proxmox could access the media through their respective mounts. Checks covered the mounts, exports, MergerFS and disks. The existing media branch passed the available SMART checks; SMART access for the recently added media branch could not be completed through the USB adapter. After checks and actions on Plex configuration/deployment, the container listed `/movies` and accessed the tested media file again.

**Root cause remains unconfirmed.** Recovery after intervention does not establish which layer caused the stale handle. If the issue recurs, preserve logs and compare access from Proxmox, Ubuntu and the container before changing mount or application configuration.

## Backup and monitoring

Historical Proxmox notes describe capacity-aware guest backup schedules and a lightweight host configuration archive with retention. Another record documents a guest backup failing when the USB backup target ran out of space. These notes do not establish current schedules or successful restore validation. No successful restore test is claimed here.

Netdata and Telegram notification notes exist, but current monitoring coverage and alert delivery were not established by the storage command capture. Treat these as documented operational work, not a guarantee of active monitoring.

## Limitations and planned validation

- Cold-boot resilience for Proxmox, the NFS automount and Docker startup has not been tested.
- SMART health for the recently added media branch remains unknown because the attempted USB/SAT access was unsupported.
- The previous Plex stale-file-handle root cause is unknown.
- Backup restore validation is not established by available evidence.
- A historical systemd verification warning and NFS `lockd` startup message need characterization.
- MergerFS provides a unified namespace, not redundancy or disaster recovery.

Prioritized follow-up is to obtain SMART data for the recently added media branch through a supported interface, clarify mount and Docker startup dependencies, investigate the systemd/NFS warnings, and schedule a controlled cold-boot and restore validation. Such tests should be documented only after they are performed.


