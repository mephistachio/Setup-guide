# Homeserver setup guide

Target: an old ThinkPad, clean Debian install, USB drive bay with up to four disks in a RAID array, running k3s (single-node Kubernetes). Everything you deploy afterwards is described in one Helm umbrella chart in this repo.

Commands run as root (`sudo -i`) unless noted.

## 0. Automated setup with Ansible

Sections 2–5 are automated by the playbook in [ansible/](../ansible), which configures the machine it runs on (`localhost`). The manual commands below stay as reference for what it does. Run from the repo root (`ansible.cfg` points Ansible at `ansible/`):

```sh
sudo apt install -y ansible
ansible-galaxy collection install -r ansible/requirements.yml
ansible-playbook ansible/playbooks/site.yml -K            # everything
ansible-playbook ansible/playbooks/site.yml -K --tags k3s # one part
```

Settings live in [ansible/inventory/group_vars/homeserver.yml](../ansible/inventory/group_vars/homeserver.yml); every role's tunables are in `ansible/roles/<role>/defaults/main.yml`.

| Tag | Role | Guide section |
|---|---|---|
| `base` | packages, hostname, `/etc/hosts`, unattended-upgrades, panic reboot | 2, 2.2, 2.3, 4.2 (local only) |
| `power` | mask sleep, logind lid/keys, kernel cmdline (`consoleblank`, USB autosuspend, UAS quirks), no resume, battery thresholds | 1 (thresholds), 2.1, 3.1 |
| `ssh` | no password or root login over SSH | 2.4 |
| `firewall` | ufw rules | 2.5 |
| `raid` | create array (opt-in), ext4, fstab with `nofail`, mdadm.conf, alerts, smartd, data dirs | 3.2–3.5 |
| `k3s` | config, wait-for-mount drop-in, install, graceful shutdown (`k3s-shutdown@<node>.service` drains the node before k3s stops) | 4 |
| `helm` | helm binary, `KUBECONFIG` in `/etc/profile.d` | 4.1 |
| `chart` | `helm upgrade --install homeserver` | 5.1 |

The RAID is only built when `raid_devices` lists the `/dev/disk/by-id/...` paths, and since that wipes the disks it also needs `-e raid_create_confirm=true`. Without disks, `/srv/data` is a plain directory on the root disk.

Still manual: BIOS settings (section 1, except battery thresholds), the power-loss test, DNS on the router, a mail relay for mdadm alerts, replacing failed disks, and backups. Reboot after the first run so the kernel command line and logind settings take effect.

## 1. BIOS / firmware

Enter setup with **F1** (or Enter, then F1) at the ThinkPad logo.

| Setting | Value | Why |
|---|---|---|
| Config → Power → **Power On with AC Attach** | Enabled | Server boots by itself when mains returns after an outage. Many ThinkPads have no "After Power Loss" option; this is the equivalent. If yours lacks it, check for a BIOS update, or use Wake on LAN / a smart plug as fallback. |
| Config → Power → Sleep State | Linux (S3) or irrelevant | We disable sleep in the OS anyway. |
| Config → Network → **Wake On LAN** | AC only (optional) | Lets you power the machine on remotely. |
| Config → Power → Adaptive Thermal Management | Maximize performance, or leave | Optional. |
| Config → Power → Battery charge thresholds | Start 40% / Stop 80% (if offered) | Prolongs battery life if you leave it plugged in forever. The battery then acts as a tiny UPS. Alternatively remove the battery if it is swollen or dead. |
| Startup → Boot order | Internal disk first | USB drives must not be tried before the system disk. |
| Security → Secure Boot | Your choice (Debian supports it) | |
| Startup → CSM / UEFI | UEFI only | Matches a normal Debian UEFI install. |

Test it: shut down, pull the AC plug (and battery, if installed), plug back in, and confirm the machine boots on its own.

## 2. Debian base configuration

Install Debian with only "SSH server" and "standard system utilities" (no desktop). Give the machine a fixed address: either a DHCP reservation on your router (preferred) or static config. Prefer Ethernet over Wi-Fi.

```sh
apt update && apt full-upgrade -y
apt install -y sudo curl git vim htop smartmontools mdadm lvm2 \
  nfs-common open-iscsi ufw unattended-upgrades ca-certificates gnupg
hostnamectl set-hostname homeserver
```

### 2.1 Disable sleep, suspend and hibernate

```sh
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Make sure logind ignores the lid and power key (a closed laptop lid is the usual server position):

```sh
mkdir -p /etc/systemd/logind.conf.d
cat > /etc/systemd/logind.conf.d/homeserver.conf <<'CONF'
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
HandlePowerKey=ignore
HandleSuspendKey=ignore
HandleHibernateKey=ignore
IdleAction=ignore
CONF
systemctl restart systemd-logind
```

Drop the swap-backed hibernation resume target as well: if the installer created a swap partition, remove `resume=` from `/etc/default/grub` (and `/etc/initramfs-tools/conf.d/resume`) if present, then `update-grub && update-initramfs -u`.

Disable screen blanking on the console so the panel does not wake and sleep forever (optional, saves backlight power by turning it off instead):

```sh
# /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet consoleblank=60"
update-grub
```

Verify: `systemctl status sleep.target` shows `masked`; closing the lid does not suspend the machine.

### 2.2 Reboot automatically after a kernel panic / hang (optional)

```sh
echo 'kernel.panic = 10' > /etc/sysctl.d/90-panic.conf
sysctl --system
```

### 2.3 Automatic security updates

```sh
dpkg-reconfigure -plow unattended-upgrades
```

### 2.4 SSH

Copy your public key (`ssh-copy-id user@homeserver`), then in `/etc/ssh/sshd_config.d/10-hardening.conf`:

```
PasswordAuthentication no
PermitRootLogin no
```

`systemctl reload ssh`.

### 2.5 Firewall

k3s needs these reachable from the LAN: 22 (SSH), 80/443 (ingress), 6443 (Kubernetes API, only if you use `kubectl` remotely). Pod/service networks must be allowed through.

```sh
ufw default deny incoming
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 6443/tcp
ufw allow from 10.42.0.0/16 to any   # pods
ufw allow from 10.43.0.0/16 to any   # services
ufw enable
```

## 3. The USB RAID array

### 3.1 Caveats of RAID over USB

- A RAID enclosure with one USB cable is a single point of failure: if the cable or hub glitches, md kicks drives out. Use a good cable, a powered enclosure, and USB 3 port directly on the laptop.
- If the bay presents disks individually (JBOD / "pass-through" mode), use Linux `mdadm`. If the enclosure does its own hardware RAID, you can skip this section and just treat the result as one disk — but then recovery needs the same enclosure.
- Some USB-SATA bridges misbehave with UAS. If you see `uas_eh_abort_handler` resets in `dmesg`, add a quirk to disable UAS for that bridge: find IDs with `lsusb` and add `usb-storage.quirks=VVVV:PPPP:u` to `GRUB_CMDLINE_LINUX_DEFAULT`.
- Disable USB autosuspend so disks don't vanish: add `usbcore.autosuspend=-1` to `GRUB_CMDLINE_LINUX_DEFAULT`, then `update-grub`.
- RAID is **not a backup**. Keep an offsite/second copy of what matters.

### 3.2 Identify disks

```sh
lsblk -o NAME,SIZE,MODEL,SERIAL,TRAN
ls -l /dev/disk/by-id/ | grep -v part
```

Use `/dev/disk/by-id/...` names below (stable across reboots; `sdX` letters are not). Check each disk's health first: `smartctl -a -d sat /dev/sdX`.

### 3.3 Create the array

Pick a level: RAID1 (2 disks, mirror), RAID5 (3–4 disks, 1 disk redundancy), RAID10 (4 disks, faster rebuilds). With large disks, RAID6 or RAID10 are safer than RAID5.

**This destroys all data on the listed disks.**

```sh
mdadm --create /dev/md0 --level=5 --raid-devices=4 \
  /dev/disk/by-id/usb-DISK1 /dev/disk/by-id/usb-DISK2 \
  /dev/disk/by-id/usb-DISK3 /dev/disk/by-id/usb-DISK4
cat /proc/mdstat    # initial sync takes hours; the array is usable meanwhile
```

Filesystem and mount:

```sh
mkfs.ext4 -L data /dev/md0
mkdir -p /srv/data
blkid /dev/md0                         # copy the UUID
echo 'UUID=<uuid> /srv/data ext4 defaults,nofail,x-systemd.device-timeout=30 0 2' >> /etc/fstab
mount -a
```

`nofail` matters: if the array is missing at boot, the server must still come up and be reachable over SSH.

Persist the array definition and rebuild the initramfs:

```sh
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u
```

Enable monitoring with email, and scheduled checks (Debian ships the monthly `checkarray` cron job):

```sh
# /etc/mdadm/mdadm.conf
MAILADDR you@example.com
```
Install a mail relay such as `msmtp-mta` or `postfix` for this to deliver, or point `PROGRAM` at a script that sends a push notification.

Also enable SMART monitoring: `systemctl enable --now smartmontools`.

### 3.4 Replacing a failed disk

```sh
mdadm --detail /dev/md0
mdadm /dev/md0 --fail /dev/disk/by-id/usb-OLD --remove /dev/disk/by-id/usb-OLD
# swap the physical disk
mdadm /dev/md0 --add /dev/disk/by-id/usb-NEW
```

### 3.5 Directory layout

```sh
mkdir -p /srv/data/{k3s-storage,backups,media}
```

`k3s-storage` will hold all Kubernetes persistent volumes.

## 4. Install k3s

k3s is a single-binary Kubernetes with Traefik ingress, a local-path storage provisioner and a service load balancer built in. That is all this setup needs.

Write config **before** installing so persistent volumes land on the RAID:

```sh
mkdir -p /etc/rancher/k3s
cat > /etc/rancher/k3s/config.yaml <<'CONF'
write-kubeconfig-mode: "0640"
write-kubeconfig-group: "sudo"
default-local-storage-path: /srv/data/k3s-storage
tls-san:
  - homeserver
  - homeserver.home.lan
CONF
```

Install (pin a version; see <https://github.com/k3s-io/k3s/releases> for current stable):

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=stable sh -
```

Verify:

```sh
systemctl status k3s
kubectl get nodes          # Ready
kubectl get pods -A        # traefik, coredns, local-path-provisioner Running
```

If the RAID is mounted late, make k3s wait for it so volumes aren't created on the root disk:

```sh
mkdir -p /etc/systemd/system/k3s.service.d
cat > /etc/systemd/system/k3s.service.d/wait-for-data.conf <<'CONF'
[Unit]
RequiresMountsFor=/srv/data
CONF
systemctl daemon-reload
```

(With `nofail` in fstab, `RequiresMountsFor` makes k3s fail rather than start without the data — which is what you want.)

### 4.1 Tools on your workstation

Install `kubectl` and `helm` on your laptop/desktop, and copy the kubeconfig:

```sh
scp homeserver:/etc/rancher/k3s/k3s.yaml ~/.kube/homeserver.yaml   # needs read permission
sed -i 's/127.0.0.1/homeserver.home.lan/' ~/.kube/homeserver.yaml
export KUBECONFIG=~/.kube/homeserver.yaml
kubectl get nodes
```

On the server itself: `curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`, and `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` (add to your shell profile).

### 4.2 DNS

Services are exposed by hostname through Traefik on ports 80/443. Make `*.home.lan` resolve to the server's IP: a wildcard record on your router/Pi-hole/AdGuard, or entries in `/etc/hosts` on each client for a start:

```
192.168.1.10  whoami.home.lan
```

## 5. The "helm chart of helm charts"

[charts/homeserver](../charts/homeserver) is an **umbrella chart**. It has no templates of its own; each piece of software is a *dependency* in `Chart.yaml`, switched on or off in `values.yaml`:

```
charts/homeserver/
├── Chart.yaml          # list of dependencies (subcharts)
├── values.yaml         # global values + one block per subchart, with enabled: true/false
└── charts-local/       # subcharts that live in this repo (e.g. whoami)
```

### 5.1 Deploy it

```sh
cd charts/homeserver
helm dependency update          # fetches subcharts into charts/, writes Chart.lock
helm lint .
helm upgrade --install homeserver . \
  --namespace homeserver --create-namespace
```

Check: `kubectl -n homeserver get pods,ingress`, then open `http://whoami.home.lan`.

Everything is one Helm release. To change something, edit `values.yaml` and re-run the same `helm upgrade --install`. To see what would change first: `helm diff upgrade` (plugin `databus23/helm-diff`) or `helm template . | less`.

### 5.2 Add software

1. Find the chart (<https://artifacthub.io>).
2. Add it to `dependencies:` in `Chart.yaml`, pinning the version, with `condition: <name>.enabled`.
3. Add a `<name>:` block to `values.yaml` with `enabled: true` and the chart's values. Use `global.domain` for hostnames and `global.storageClass` for persistence.
4. `helm dependency update && helm upgrade --install homeserver . -n homeserver`.

Example, enabling Nextcloud:

```yaml
nextcloud:
  enabled: true
  nextcloud:
    host: cloud.home.lan
  ingress:
    enabled: true
  persistence:
    enabled: true
    storageClass: local-path
```

Remove software by setting `enabled: false` and upgrading. Note that PVCs usually survive removal; delete them explicitly if you want the data gone.

### 5.3 Notes

- **Namespaces.** A single release installs everything into one namespace. Charts that need cluster-wide resources (cert-manager CRDs, operators) are better installed as separate releases first. If that grows, switch to Argo CD or Flux with one Application per subchart — the umbrella pattern here maps over directly.
- **Secrets.** Never commit plaintext secrets to `values.yaml`. Keep them in a gitignored `secrets.yaml` (`-f secrets.yaml`), or use SOPS/age or Sealed Secrets.
- **Persistent data.** The `local-path` volumes are plain directories under `/srv/data/k3s-storage`, on the RAID. They are node-bound, which is fine for a single node.
- **Commit `Chart.lock`.** It pins exact dependency versions; the downloaded `charts/` directory is gitignored.

## 6. Backups and maintenance

- k3s state (SQLite): `k3s etcd-snapshot` is for etcd only; for the default SQLite datastore, back up `/var/lib/rancher/k3s/server/db/` while stopped, or use a cron job with `sqlite3 .backup`. Everything declarative is already in this git repo; only PVC contents and the k3s token/database need backing up.
- Copy `/srv/data` to a separate disk or remote with `restic` or `borg` on a systemd timer.
- Upgrade k3s by rerunning the install script; upgrade Debian with unattended-upgrades plus occasional `apt full-upgrade`.
- Watch `cat /proc/mdstat`, `smartctl -H`, and `df -h /srv/data` periodically.

## 7. Checklist

- [ ] BIOS: power on with AC attach, boot order
- [ ] Power loss test passed
- [ ] Sleep/hibernate masked, lid ignored
- [ ] Static IP / DHCP reservation, SSH keys only
- [ ] RAID created, in fstab with `nofail`, mdadm monitoring on
- [ ] k3s installed with storage path on the RAID
- [ ] DNS wildcard for `*.home.lan`
- [ ] `helm upgrade --install homeserver` works, whoami reachable
- [ ] Backups configured
