# homeserver-demo

Base for a homelab server running on an old ThinkPad with a USB drive bay (up to 4 disks, software RAID), Debian and k3s.

- [docs/setup-guide.md](docs/setup-guide.md): preparing the machine, installing k3s, deploying software
- [charts/homeserver/](charts/homeserver/): the "chart of charts" — one Helm umbrella chart that installs everything
- [ansible/](ansible/): playbook that configures this machine as the homeserver (`ansible-playbook ansible/playbooks/site.yml -K`)
- [docs/tailscale.md](docs/tailscale.md): reaching services from anywhere with the Tailscale operator, and where to keep secrets
- [docs/secrets.md](docs/secrets.md): committing secrets encrypted with SOPS and age
