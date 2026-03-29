# AdGuard Home — Encrypted DNS on a Raspberry Pi 4B

Rocky Linux 9 · DNS-over-TLS · ZeroSSL Cert · DuckDNS · firewalld

---

## What This Is

I built a self-hosted DNS filtering server on a Raspberry Pi 4B that blocks ads and trackers for every device on my network and encrypts all DNS traffic using DNS-over-TLS. No cloud service, no third party — just my Pi sitting on my desk handling DNS for my whole network.

This wasn't a clean tutorial run either. I flashed Rocky Linux onto a brand new external hard drive after the original drive malfunctioned, repartitioned it, debugged why sshd wouldn't start on a fresh flash, tracked down a certificate path mismatch in AdGuard, and worked through a TLS handshake issue between my laptop and the Pi. By the end of it everything was running — encrypted, filtered, and verified.

This is part of my homelab build as I work toward a career in Infrastructure Security Engineering.

---

## How It Works

```
Your Device
    │
    │  DNS-over-TLS (port 853, encrypted)
    ▼
Raspberry Pi 4B
├── Rocky Linux 9.7 Minimal (headless, no GUI)
├── AdGuard Home — filters ads and trackers
├── ZeroSSL ECC cert tied to kukimonji55.duckdns.org
└── firewalld — only the ports that need to be open are open
    │
    │  Upstream DNS queries (filtered)
    ▼
AdGuard Default Upstream Resolvers
```

---

## Hardware

| Thing | What it is |
|-------|------------|
| Device | Raspberry Pi 4B 2GB |
| Storage | External HDD (new drive, flashed via `dd`) |
| OS | Rocky Linux 9.7 Minimal |
| Network | Wired ethernet |

---

## What I Built

- DNS ad and tracker blocking via AdGuard Home
- DNS-over-TLS on port 853 with a real ECC certificate — not self-signed
- Dynamic DNS via DuckDNS so the domain stays pointed at my Pi even if my public IP changes
- Automatic cert renewal via acme.sh using the DNS-01 challenge — no need to open port 80 to the internet
- SSH locked down to key-based auth only — no passwords, no root login
- firewalld configured with only the ports the services actually need

---

## How I Built It

### Step 1 — Flash Rocky Linux to the New Drive

First unmount the drive if it's already mounted:

```bash
sudo umount /dev/sda1
sudo umount /dev/sda5
```

Then flash it:

```bash
sudo dd if=/path/to/RockyRpi_9.x.img of=/dev/sda bs=4M status=progress conv=fsync
```

### Step 2 — Expand the Root Partition

The image is only a few gigabytes. After flashing, the partition table thinks the disk ends there. You have to expand the partition and then grow the filesystem to use the rest of the drive.

Rocky Linux 9 uses XFS, not ext4, so `resize2fs` won't work here — use `xfs_growfs` instead.

```bash
sudo growpart /dev/sda 3
sudo xfs_growfs /
df -h /
```

### Step 3 — Get SSH Working

On a fresh flash the SSH host keys don't exist yet. That was the first thing I ran into — sshd was crashing at startup with `status=255/exception`. Turned out the host keys were missing and there was also a corrupted line in `sshd_config` from a terminal power-save interrupt that injected garbage text right into the middle of a config line while I was editing it.

Generate the missing host keys:

```bash
sudo ssh-keygen -A
```

Validate the config before starting the service:

```bash
sudo /usr/sbin/sshd -t
```

If that comes back clean, start it:

```bash
sudo systemctl enable sshd
sudo systemctl start sshd
```

### Step 4 — Harden SSH

Generate a key pair on your client machine:

```bash
ssh-keygen -t ed25519 -C "your-label"
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@<pi-ip>
```

Test that key login works, then lock down `/etc/ssh/sshd_config`:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
X11Forwarding no
MaxAuthTries 3
```

```bash
sudo systemctl reload sshd
```

Don't disable password auth before confirming key login works or you'll lock yourself out.

### Step 5 — Update the System

```bash
sudo dnf upgrade --refresh -y
sudo reboot
```

### Step 6 — Install AdGuard Home

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v

sudo systemctl enable AdGuardHome
sudo systemctl start AdGuardHome
```

Do the initial setup through the web UI at `http://<pi-ip>:3000`.

### Step 7 — Set Up DuckDNS

This keeps your domain pointed at your Pi even when your public IP changes.

```bash
mkdir -p ~/duckdns

cat > ~/duckdns/duck.sh << 'EOF'
#!/bin/bash
echo url="https://www.duckdns.org/update?domains=YOUR_DOMAIN&token=YOUR_TOKEN&ip=" | curl -k -o ~/duckdns/duck.log -K -
EOF

chmod +x ~/duckdns/duck.sh
~/duckdns/duck.sh
cat ~/duckdns/duck.log
```

If it worked it says `OK`. Then schedule it to run every 5 minutes:

```bash
crontab -e
# add this line:
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```

### Step 8 — Issue the TLS Certificate

I used acme.sh with the DNS-01 challenge through the DuckDNS API. This means the cert gets issued without needing to open port 80 to the internet — acme.sh just updates a DNS record on your DuckDNS domain to prove you own it.

```bash
curl https://get.acme.sh | sh -s email=your@email.com
source ~/.bashrc

export DuckDNS_Token="YOUR_TOKEN"

~/.acme.sh/acme.sh --issue --dns dns_duckdns -d YOUR_DOMAIN.duckdns.org
```

Create the cert folder and install the cert:

```bash
sudo mkdir -p /etc/adguardhome/cert
sudo chown $USER:$USER /etc/adguardhome/cert

~/.acme.sh/acme.sh --install-cert -d YOUR_DOMAIN.duckdns.org \
  --cert-file /etc/adguardhome/cert/cert.pem \
  --key-file /etc/adguardhome/cert/key.pem \
  --fullchain-file /etc/adguardhome/cert/fullchain.pem \
  --reloadcmd "sudo systemctl restart AdGuardHome"
```

acme.sh sets up its own cron job for renewals automatically. Certs renew before they expire — no manual work needed.

### Step 9 — Configure AdGuard Home TLS

This is where I hit a wall. AdGuard was showing "certificate chain is invalid" even though the cert files were there and valid. The issue was that I had the paths in the wrong fields in the yaml — there are two sets of fields (`certificate_chain`/`private_key` and `certificate_path`/`private_key_path`) and AdGuard was looking at the wrong ones.

Edit `/opt/AdGuardHome/AdGuardHome.yaml` and find the `tls:` section:

```yaml
tls:
  enabled: true
  server_name: YOUR_DOMAIN.duckdns.org
  force_https: false
  port_https: 443
  port_dns_over_tls: 853
  port_dns_over_quic: 853
  certificate_chain: ""
  private_key: ""
  certificate_path: /etc/adguardhome/cert/fullchain.pem
  private_key_path: /etc/adguardhome/cert/key.pem
```

Note on `force_https: false` — setting this to true will break your web UI access. Learned that the hard way.

```bash
sudo systemctl restart AdGuardHome
```

### Step 10 — Open the Firewall Ports

```bash
sudo firewall-cmd --permanent --add-port=53/tcp
sudo firewall-cmd --permanent --add-port=53/udp
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=853/tcp
sudo firewall-cmd --permanent --add-port=853/udp
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

### Step 11 — Point Your Client at the Pi

On Fedora with systemd-resolved:

```bash
sudo nmcli connection modify "YOUR_CONNECTION" \
  ipv4.dns "PI_IP" \
  ipv4.ignore-auto-dns yes

sudo nmcli connection up "YOUR_CONNECTION"
```

Edit `/etc/systemd/resolved.conf`:

```ini
[Resolve]
DNS=PI_IP#YOUR_DOMAIN.duckdns.org
DNSOverTLS=opportunistic
DNSSEC=no
```

```bash
sudo systemctl restart systemd-resolved
```

`opportunistic` is the right setting here — it tries DoT first and falls back gracefully if the handshake doesn't complete. Setting it to `yes` causes hard failures and breaks your internet. Found that out too.

---

## Verification

Confirm the TLS handshake is actually working:

```bash
openssl s_client -connect PI_IP:853 -servername YOUR_DOMAIN.duckdns.org
# Look for: Verify return code: 0 (ok)
```

Confirm DNS is resolving:

```bash
dig google.com @PI_IP
# Should come back clean in under 100ms
```

Confirm blocking is working:

```bash
dig doubleclick.net
# Should return SERVFAIL — AdGuard blocked it
```

Confirm encryption is active:

```bash
resolvectl status
# Look for: DNSOverTLS=opportunistic
# And: Current DNS Server: PI_IP#YOUR_DOMAIN.duckdns.org
```

---

## Things That Broke and How I Fixed Them

**sshd crashing on startup with status=255**
Fresh flash — SSH host keys don't exist yet. Fix: `sudo ssh-keygen -A`

**Corrupted sshd_config line**
Terminal power-save interrupt fired while I was editing the config and injected text mid-line. `PermitRootLogin` and an unrecognized option ended up on the same line. Fix: always run `sudo /usr/sbin/sshd -t` to validate the config before trying to start the service.

**AdGuard showing "certificate chain is invalid"**
The cert files existed and were valid but AdGuard couldn't read them. Two problems — wrong yaml fields (`certificate_chain` instead of `certificate_path`) and cert files owned by the wrong user. Fix: use the `_path` variant fields in the yaml and make sure cert files are owned by root since AdGuard runs as root.

**DNS breaking when pointing laptop at Pi**
firewalld wasn't open on port 53 yet. Fix: open the firewall rules before pointing anything at the Pi.

**DoT hard failing with DNSOverTLS=yes**
Resolved refuses to fall back to plain DNS in this mode — if the TLS handshake doesn't complete, DNS just stops working entirely. Fix: use `opportunistic`.

---

## Ports Open on the Pi

| Port | Protocol | What It's For |
|------|----------|---------------|
| 22 | TCP | SSH |
| 53 | TCP/UDP | Plain DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 853 | TCP/UDP | DNS-over-TLS |
| 3000 | TCP | AdGuard Home web UI |

---

## Certificate Renewal

acme.sh handles this automatically. It checks daily and renews when the cert is within 30 days of expiry. The `--reloadcmd` restarts AdGuard after renewal so the new cert loads without any manual work needed.

Verify the cron job is there:

```bash
crontab -l | grep acme
```

Current cert expiry: **2026-06-28**

---

## Related

- [Homelab-Docs](https://github.com/YOUR_USERNAME/Homelab-Docs) — main homelab repo with network topology, VLAN design, and Cisco 2960 router-on-a-stick setup

---

*Part of an active homelab build toward Infrastructure Security Engineering. Tools: Rocky Linux 9, AdGuard Home, acme.sh, firewalld, systemd-resolved.*
