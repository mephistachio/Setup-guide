# homeserver-demo

Base for a homelab server running on an old ThinkPad with a USB drive bay (up to 4 disks, software RAID), Debian and k3s.

- [docs/setup-guide.md](docs/setup-guide.md): preparing the machine, installing k3s, deploying software
- [charts/homeserver/](charts/homeserver/): the "chart of charts" — one Helm umbrella chart that installs everything
