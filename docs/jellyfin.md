# Jellyfin

[Jellyfin](https://jellyfin.org) is deployed from the [official Helm chart](https://github.com/jellyfin/jellyfin-helm) as a subchart of [charts/homeserver](../charts/homeserver). It is exposed **only on the tailnet** through the [Tailscale operator](tailscale.md): there is no LAN or public ingress.

URL: `https://jellyfin.<tailnet>.ts.net`, with a valid certificate. Find the exact address with:

```sh
kubectl -n homeserver get ingress homeserver-jellyfin
```

## Where things are

| What | Where |
|---|---|
| Media library | `/srv/data/media` on the server, mounted at `/media` in the pod (hostPath) |
| Config, users, metadata | 5 Gi PVC `homeserver-jellyfin-config`, a directory under `/var/lib/rancher/k3s/storage` on the fast volume. A PVC created before the storage move stays in `/srv/data/k3s-storage` (a PV's path is fixed) |
| Transcode cache | `emptyDir` (lost on pod restart, on the root disk) |

Copy files into `/srv/data/media` on the server, for example `media/movies/` and `media/shows/`, then add the folders as libraries under `/media/...` in Jellyfin. The pod runs as root, so file ownership does not matter.

## First start

1. Open the URL from a device on your tailnet **right away**. The setup wizard is open until the first admin account exists.
2. Create the admin user, add libraries, finish.
3. On Android install the Jellyfin app (or Findroid) and use the same `https://` URL, with the Tailscale app connected.

## Operating

- Settings are in the `jellyfin:` block of [values.yaml](../charts/homeserver/values.yaml); redeploy with `ansible-playbook ansible/playbooks/site.yml -K --tags chart`.
- Upgrade: raise the chart `version` in `Chart.yaml`, run `helm dependency update`, redeploy. The Jellyfin version follows the chart.
- The hostname is the first label of `ingress.tls[].hosts`; changing it creates a new Tailscale machine, delete the old one in the admin console.
- Back up the config PVC and `/srv/data/media`.

## Limits of this setup

- No DLNA or local network discovery (they need host networking). Clients must be given the URL.
- No hardware transcoding. For Intel Quick Sync, the pod needs `/dev/dri`; add it via `volumes`/`volumeMounts` and a `securityContext` when you need it.
- `/srv/data` is an LVM volume on the system disk until the RAID replaces it, see the setup guide 2.6 and 3.6.
