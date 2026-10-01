# Personal WireGuard VPN

Turn your own Linux VPS into a private VPN for you and a few friends with **one script** (`vpn.sh`) and
**one settings file** (`vpn.json`).

```
 Your laptop ─┐
 Your phone  ─┼══ encrypted WireGuard tunnel ══► your VPS ──► internet
 A friend    ─┘
```

While connected, all your traffic is encrypted to the VPS, and websites see the **VPS's IP and
location** instead of yours.

---

## What you need

- A VPS running **Ubuntu 22.04 or 24.04** with root SSH access. 1 CPU and 1 GB RAM are plenty.
- **ufw** (Ubuntu's firewall) turned on. If it isn't, allow SSH **first**, or you'll lock yourself out:
  ```bash
  ufw allow OpenSSH
  ufw enable
  ```
- The free **WireGuard app** on each device: [wireguard.com/install](https://www.wireguard.com/install/)
  (Windows, macOS, iOS, Android, Linux).

---

## Quick start

**1. Edit `vpn.json`** and list the people/devices you want to allow:

```json
{
  "server": {
    "host": "auto",
    "port": 51820,
    "subnet": "10.8.0.0/24",
    "dns": ["1.1.1.1", "1.0.0.1"]
  },
  "peers": ["my-laptop", "my-phone"]
}
```

**2. Copy both files to the VPS and run setup** (replace `<vps-ip>`):

```bash
ssh root@<vps-ip> mkdir -p /root/vpn
scp vpn.sh vpn.json root@<vps-ip>:/root/vpn/
ssh root@<vps-ip> bash /root/vpn/vpn.sh setup
```

Setup installs WireGuard, opens UDP port 51820 and starts the VPN (also after reboots). It creates
one config file per person in `/etc/wireguard/clients/`.

**3. Get each config to its device:**

- **Computer:** download it, then in the WireGuard app choose **Import tunnel(s) from file**.
  ```bash
  scp root@<vps-ip>:/etc/wireguard/clients/my-laptop.conf .
  ```
- **Phone:** show a QR code in the VPS terminal and scan it with the WireGuard app (**+** → **Scan from QR code**):
  ```bash
  qrencode -t ansiutf8 < /etc/wireguard/clients/my-phone.conf
  ```

**4. Click Activate** in the WireGuard app. Check [whatismyipaddress.com](https://whatismyipaddress.com):
it should now show your VPS's IP.

**5. Delete the configs from the server** once they've been imported:

```bash
ssh root@<vps-ip> rm /etc/wireguard/clients/*.conf
```

---

## vpn.json settings

| Setting | Meaning |
|---|---|
| `server.host` | `"auto"` uses the VPS's public IP. Set a domain or IP to override it. |
| `server.port` | UDP port WireGuard listens on. Default `51820`. |
| `server.subnet` | Private addresses inside the VPN. Must be a `/24`. **Don't change it after setup.** |
| `server.dns` | DNS servers the devices use while connected. |
| `peers` | Names of the people/devices allowed in: lowercase letters, digits and dashes. |

---

## Commands

Run on the VPS as root, from the folder with `vpn.sh` and `vpn.json`:

| Command | What it does |
|---|---|
| `bash vpn.sh setup` | Install or update everything from `vpn.json`. Safe to re-run. |
| `bash vpn.sh add <name>` | Add a person (also adds them to `vpn.json`). Add `--qr` to show a QR code. |
| `bash vpn.sh remove <name>` | Remove a person. Their device can't connect anymore, immediately. |
| `bash vpn.sh sync` | Make the server match the `peers` list in `vpn.json` (after editing it). |
| `bash vpn.sh list` | Everyone on the VPN: address, last seen, data used. |
| `bash vpn.sh uninstall` | Remove the VPN and its firewall rules. Old config is kept in `/root/backups/`. |

---

## Good to know

- **A config only works with the server it came from.** It holds that server's key and the person's own
  private key. To use a different VPS, run setup there and import the new config as a separate tunnel.
- **Each config is a private key.** Anyone who has the file can use your VPN as that person. Send it
  privately, and remove the person (`bash vpn.sh remove <name>`) if a device is lost.
- **All traffic uses your VPS's bandwidth**, and anything your friends do online comes from your
  server's IP. Only add people you trust.
- **The VPN changes your IP, not everything.** Sites can still learn your real location if you allow
  location access, and your accounts and time zone stay the same.
- People on the VPN can't reach each other's devices; they can only reach the internet.

## What setup changes on the server

- Installs `wireguard-tools`, `jq` and `qrencode`.
- Creates `/etc/wireguard/` (server keys, `peers/`, `clients/`, `wg0.conf`).
- Turns on IPv4 forwarding (`/etc/sysctl.d/99-wireguard.conf`).
- ufw: allows the VPN port in, and allows VPN traffic out to the internet. Nothing else is opened.
- Adds NAT for the VPN subnet while the VPN is running.
- Backs up the previous firewall state to `/root/backups/` first.

## Troubleshooting

| Problem | Try |
|---|---|
| App says connected but no internet | `bash vpn.sh list`: if "last seen" says `never`, UDP 51820 is blocked. Check your VPS provider's firewall panel. |
| Internet stopped completely | Click **Deactivate** in the WireGuard app, then check the server with `bash vpn.sh list`. |
| "This script expects ufw to be active" | Turn on ufw (see *What you need*). |
| Some sites load slowly | Re-run `bash vpn.sh setup`; it re-applies the packet-size fix. |
