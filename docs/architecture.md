# Architecture

## System roles

The observed design uses a Proxmox VE 9.2 host as the virtualization and storage layer. The host mounts two ext4 media branches, combines their media directories with MergerFS, and exports the resulting pool over NFS. An Ubuntu Server 24.04.4 LTS virtual machine mounts that export and runs the media application containers through Docker.

The supplied command capture showed the relevant mounts and containers active at inspection time. It did not test boot recovery or establish that every service remains available after a host or network failure.

## Data flow

1. Physical media disks are mounted as separate ext4 branches on Proxmox.
2. MergerFS presents the media directories as a single namespace at `/media-pool`.
3. Proxmox exports the pool; Ubuntu mounts it at `/media-pool` through NFSv4.2 and systemd automount.
4. Docker bind mounts expose the pool to application containers.
5. qBittorrent receives downloads; Radarr and Sonarr manage movie/series workflows; Plex reads the library paths.

The paths shown here are generic logical mount points. Live addresses, hostnames, disk identifiers and export client ranges are intentionally omitted.

## Application services

The capture listed Plex, qBittorrent, Radarr, Sonarr, Overseerr, Prowlarr, FlareSolverr, Uptime Kuma and Portainer running. The principal pool consumers were:

| Service | Container path | Purpose |
|---|---|---|
| Plex | `/movies`, `/tv` | Read media libraries; `/transcode` used a memory-backed temporary path in the captured mount list |
| qBittorrent | `/data` | Store downloads in shared pool namespace |
| Radarr | `/data` | Manage movie downloads and library imports |
| Sonarr | `/data` | Manage series downloads and library imports |

Application configuration paths were separate from the media pool in the captured Docker mounts. Portainer was also shown with Docker socket access; this is a privileged administrative capability and should not be exposed in public deployment examples.

## Design decisions and trade-offs

- **Host-managed disks:** Proxmox owns the physical disk mounts and composes the pool. This centralizes filesystem ownership at the hypervisor layer; Ubuntu consumes it as a network filesystem.
- **MergerFS namespace:** the pool gives applications one logical path over multiple branches. It does not mirror data or provide redundancy.
- **NFS guest access:** NFSv4.2 makes the host-managed pool available to Ubuntu without passing the physical drives directly into the VM. This adds mount availability and startup-order dependencies.
- **Shared container path:** qBittorrent, Radarr and Sonarr use a common `/data` view, enabling same-filesystem hardlinks in tested cases. The observed hardlink tests do not prove every file or branch combination will behave identically.
- **Automount:** systemd mounts the NFS pool on access. Docker's dependency on that mount during boot still needs explicit validation.
- **Capacity imbalance:** the existing media branch contains historical media, while the recently added media branch was added more recently. Uneven usage is expected and is not evidence of a fault.

See [the architecture diagram source](../diagrams/architecture.mmd) and [storage details](storage.md).


