# UniFi Network Application

UniFi runs as two workloads on the `floor=eg` node (Raspberry Pi 5, 8 GB):

| Workload | Image | Data on the node |
| --- | --- | --- |
| `unifi` | `lscr.io/linuxserver/unifi-network-application` | `/home/he/unifi-network-application` → `/config` |
| `unifi-db` | `mongo` (pinned major, see `helm/values.yaml`) | `/home/he/unifi-db` → `/data/db` |

`unifi` uses host networking, so devices reach it on the node IP (`8080` inform, `8443` UI, `3478/udp` STUN, `10001/udp` discovery). `unifi-db` is only reachable in-cluster through the `unifi-db` ClusterIP service.

The MongoDB user is created by `init-mongo.sh` (ConfigMap `unifi-db-init`) **only on first start with an empty `/data/db`**. Passwords come from `--set unifi.mongoPassword` / `--set unifi.mongoRootPassword` (GitHub secrets `UNIFI_MONGO_PASSWORD` / `UNIFI_MONGO_ROOT_PASSWORD`). Changing them later does not change the database; see Troubleshooting.

Memory: the Java heap is capped at 1 GB (`MEM_LIMIT`), the pod at 2 Gi; the MongoDB WiredTiger cache is capped at 256 MB, the pod at 1 Gi.

## Migration from `jacobalberty/unifi`

The old image bundled MongoDB; the new one does not, so the data directory cannot be reused. Migrate through a UniFi backup. The old data in `/home/he/unifi` is left untouched for rollback.

1. **Check the node.** On the Pi:
   ```sh
   id he              # uid/gid should be 1000, otherwise adjust PUID/PGID in deployment-unifi.yaml
   getconf PAGESIZE   # 16384 on the Pi 5 default kernel; see Troubleshooting if unifi-db crashes
   ```
2. **Create the GitHub secrets.** Use alphanumeric values (`,` and `=` break `helm --set`):
   ```sh
   gh secret set UNIFI_MONGO_PASSWORD --body "$(openssl rand -hex 24)"
   gh secret set UNIFI_MONGO_ROOT_PASSWORD --body "$(openssl rand -hex 24)"
   ```
3. **Back up the old controller.** In the old UI (`https://<pi-ip>:8443`) open Settings → System → Backups and download a backup (`.unf`). Copy it off the Pi. As a fallback, auto-backups are in `/home/he/unifi/data/backup/autobackup/`.
4. **Deploy.** Merge to `main`. Helm replaces the `unifi` pod (old image stops, new image starts on the same ports) and starts `unifi-db`. UniFi is down until the restore is complete.
5. **Watch startup.**
   ```sh
   kubectl logs statefulset/unifi-db -f   # first start: init-mongo.sh runs, createUser returns { ok: 1 }
   kubectl logs statefulset/unifi -f      # wait for "[ls.io-init] done."
   ```
   First start on the Pi takes a few minutes.
6. **Restore.** Open `https://<pi-ip>:8443`. In the setup wizard choose **Restore Server from a Backup** and upload the `.unf`. The application restarts and you log in with your old credentials. Backups from 8.6.x restore into 10.x directly.
7. **Check devices.** The node IP and port `8080` are unchanged, so adopted devices should reconnect within a few minutes. If a device stays offline, search the settings for *Inform Host* (it should be empty or the Pi IP), or re-point it manually:
   ```sh
   ssh <device-user>@<device-ip>
   set-inform http://<pi-ip>:8080/inform
   ```
8. **Clean up** after a few days of stable operation: `sudo rm -rf /home/he/unifi`.

### Rollback

Revert the migration commit and redeploy. The old image starts again on `/home/he/unifi`. Changes made in the new controller are lost. `/home/he/unifi-network-application` and `/home/he/unifi-db` can then be deleted.

## Troubleshooting

- **`unifi` logs show MongoDB authentication errors.** `unifi-db` was initialised with different passwords, or the init script did not run because `/data/db` was not empty. Before the restore you lose nothing by resetting it: `kubectl scale statefulset/unifi-db --replicas=0`, `sudo rm -rf /home/he/unifi-db`, then redeploy.
- **`unifi-db` crash-loops (segfault / tcmalloc errors).** This is most likely the Pi 5's 16K-page kernel. Switch to the 4K-page kernel by adding `kernel=kernel8.img` to `/boot/firmware/config.txt` and rebooting.
- **Upgrading MongoDB.** Only move one major version at a time (7.0 → 8.0). After each step, set `featureCompatibilityVersion`. Take a UniFi backup first.
