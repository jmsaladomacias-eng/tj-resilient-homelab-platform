# Storage Architecture

## Physical filesystems and pool

The captured Proxmox inventory showed two media branches formatted as ext4 and mounted separately. The existing media branch contains historical data, while the recently added media branch was added more recently and was mostly free in the captured capacity snapshot. Their different utilization levels are expected from this history and are not, by themselves, a fault.

At the 14 November 2026 command capture, the reported figures were approximately:

| Filesystem view | Total | Used | Available | Use |
|---|---:|---:|---:|---:|
| Existing media branch | 1.8 TiB | 1.6 TiB | 292 GiB | 85% |
| Recently added media branch | 1.8 TiB | 23 GiB | 1.7 TiB | 2% |
| MergerFS pool | 3.6 TiB | 1.6 TiB | 2.0 TiB | 44% |

Values are rounded from a point-in-time `df -hT` output. No serial numbers, UUIDs or device paths are published.

## MergerFS

The captured unit combined the two media directories into one pool and specified:

```ini
Type=fuse.mergerfs
Options=defaults,allow_other,use_ino,category.create=epmfs
```

`category.create=epmfs` was present in the unit configuration. Historical tests described files created in tested download scenarios landing on the recently added media branch. The outcome of a creation policy can depend on branch contents and the specific path; the tests do not justify a universal guarantee for every directory.

MergerFS creates a unified namespace across branches. It does **not** mirror files, protect against disk failure, or replace independent backups.

## NFSv4.2 client integration

Proxmox exported the pool over NFS. Ubuntu mounted it at `/media-pool` using NFSv4.2. The captured client configuration used a systemd automount, read/write access and a hard mount for this pool. Private export ranges and live server addresses are omitted.

The command capture showed the automount active and the NFS mount triggered and mounted when accessed. It also showed a separate direct NFS entry for the existing media branch with different `soft,intr` options. That second mount was not the Plex pool path in the captured bind mounts; whether it remains necessary should be clarified in future operations documentation.

## Systemd ordering and persistence

On Proxmox, the captured MergerFS mount unit declared `Requires=` and `After=` relationships for both branch mount units and `Before=nfs-server.service`. The branch mounts and pool were active at capture time.

On Ubuntu, the generated automount unit triggered the NFS mount on access. These observations establish runtime state at inspection time only. No cold-boot test was performed. Historical notes also identify an unresolved `systemd-analyze verify` warning for the generated mount unit for the recently added media branch and a need to inspect Docker's dependency on NFS at startup.

## SMART status and limits

- **Existing media branch:** `smartctl -H` returned an overall-health self-assessment of `PASSED` at capture time. This is not a guarantee against future failure.
- **Recently added media branch:** the attempted SMART query through USB/SAT failed because the bridge path returned an unsupported SCSI command field. This is unsupported access through that path, not evidence of disk failure. No SMART pass was obtained for this branch; its health remains unknown.

## Related operational incident

A previous Plex playback incident included `StaleNetworkFileHandle` inside the container. Hosts could access the media, and Plex later regained access after checks and Plex configuration/deployment actions. The available evidence does not establish the root cause. See [operations](operations.md).


