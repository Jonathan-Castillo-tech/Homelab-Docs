Raspberry Pi 4B Home Network Infrastructure: Rocky Linux + AdGuard Home
Overview
This project documents the complete migration and setup of a Raspberry Pi 4B as a network‑wide DNS filtering and encryption server. The Pi originally ran Fedora Linux with Pi‑hole, but was migrated to Rocky Linux 9 for both principled reasons (Fedora’s direction on age verification compliance) and technical improvements (native DNS‑over‑HTTPS support, modern web interface). The final system provides:

Rocky Linux 9 Minimal as the base OS

AdGuard Home as the DNS filter and resolver

DNS‑over‑TLS (DoT) enabled with a valid TLS certificate from ZeroSSL

SSH hardened (key‑only, root login disabled)

Firewalld configured for essential services

All DNS queries from devices on the local network are filtered for ads and trackers, and encrypted queries (DoT) are accepted on port 853 for clients that support it.


Why I Did This
Replace Pi‑hole with AdGuard Home
Pi‑hole served well, but AdGuard Home natively supports DNS‑over‑HTTPS (DoH) and DNS‑over‑TLS (DoT). This allows browsers like Brave (which use DoH by default) to have their queries filtered while remaining encrypted—something Pi‑hole cannot do.

Move from Fedora to Rocky Linux
Fedora’s recent policies on age verification raised concerns about future compliance mandates. Rocky Linux is a 1:1 RHEL rebuild, stable, and perfect for RHCSA study.

Encrypt DNS Traffic
Default DNS queries are sent in plaintext, exposing browsing habits to the ISP. DoT wraps queries in TLS, preventing eavesdropping.

Centralized Configuration
One device manages DNS filtering for the entire home network, including IoT devices that cannot be individually configured for encrypted DNS.


Hardware
Component	Details
Raspberry Pi 4B	2GB RAM, headless
External SSD	Boot drive for Rocky Linux
Network	Static IP 192.168.8.110, connected via Ethernet
Laptop (client)	Fedora Linux 43 XFCE (systemd‑resolved)



Project Steps
1. Verified Existing Pi‑hole Installation
Verified Pi‑hole v6 was running directly on the host (no container).

Discovered the Pi’s own DNS pointed to the ISP router (192.168.8.1), meaning it was bypassing its own filter.

Documented upstream DNS (Cloudflare) and overall health.

2. Flash Rocky Linux to External SSD
Downloaded Rocky Linux 9 ARM64 image (.xz).

Decompressed and wrote to /dev/sda with dd.

Booted the Pi from the SSD, performed initial setup (user creation, sudo access).

3. Harden SSH
Enabled sshd and opened firewall for SSH.

Generated Ed25519 keypair on laptop, copied public key to Pi.

Disabled password authentication and root login over SSH.

4. Install AdGuard Home
Used the official install script to place AdGuard Home in /opt/AdGuardHome/.

Registered as a systemd service and started it.

Opened firewall ports: 53/tcp+udp, 80/tcp, 3000/tcp, 443/tcp, 853/tcp+udp.

5. Configure AdGuard Home
Ran the setup wizard via http://192.168.8.110:3000.

Set DNS listening on all interfaces, port 53.

Web interface on port 80.

Configured upstream DNS servers (e.g., Cloudflare 1.1.1.1).

6. Enable DNS‑over‑TLS (DoT)
Registered a free subdomain (kukimonji55.duckdns.org) pointing to the Pi’s local IP.

Used acme.sh with DuckDNS plugin to obtain a ZeroSSL certificate via DNS‑01 challenge.

Installed certificate and private key in /etc/adguardhome/certs/.

Configured AdGuard Home’s encryption settings with the fullchain certificate.

Opened firewall port 853/tcp and 853/udp.

7. Test Encrypted DNS on Client
Configured systemd‑resolved on the Fedora laptop to use 192.168.8.110#kukimonji55.duckdns.org.

Verified with resolvectl query google.com that the transport was encrypted.

8. Gradual Migration
Kept plain DNS (port 53) enabled to avoid breaking existing devices.

Migrated devices one by one to DoT as they support it.

Key Learnings & Pitfalls
Sequence matters: Install AdGuard Home before disabling the system resolver to avoid losing DNS during installation.

Check firewall on the correct machine: Opened port 853 on the Pi, not on the client.

Do not disable plain DNS prematurely: Devices with cached entries will appear to work, but new DNS lookups will fail.

Use ss -tulpn to verify which process owns a port (essential for troubleshooting DNS conflicts).

known_hosts errors after OS reinstall are expected; remove the old entry with ssh-keygen -R <IP>.

DNS‑01 challenge avoids exposing the Pi to the internet; the TXT record proves domain ownership.

Current State
Service	Status
Rocky Linux 9.7	Running, fully updated
SSH	Key‑only authentication, root disabled
firewalld	Allowed ports: 22, 53, 80, 443, 853, 3000
AdGuard Home	v0.107.73, listening on port 53 (plain) and 853 (DoT)
DoT Certificate	ZeroSSL, valid until 2026‑06‑27, auto‑renewal via acme.sh cron
Clients	Plain DNS for all devices; DoT for Fedora laptop; others to migrate
Next Steps / Future Improvements
Migrate all devices to DoT or DoH to eliminate plain DNS traffic.

Set up DoH for browsers that support custom DoH URLs, pointing to https://192.168.8.110/dns-query.

Implement DHCP in AdGuard Home to simplify DNS assignment for new devices.

Add monitoring (e.g., Prometheus node exporter) to track system health.

Document hardware inventory in a separate file for clarity.

Detailed Documentation
Each phase of this project is documented in detail in the following files (in order of execution):

Pi‑hole Audit (Pre‑Migration)
– Before‑migration audit of existing Pi‑hole v6 installation.

Fedora Pi Command Records
– Raw terminal logs verifying systemd‑resolved, port usage, etc.

Migrating to Rocky Linux
– Flashing Rocky Linux, creating user, enabling SSH.

AdGuard Home Installation
– Installing AdGuard Home, opening firewall ports.

Setting Up AdGuard Home with Rocky Linux
– Complete guide including systemd‑resolved disable, firewall, and client configuration.

Encrypted DNS (DoT) with AdGuard Home
– DuckDNS, acme.sh, ZeroSSL certificate, and enabling DoT.

For hardware specifics, see HARDWARE.md.

Date: March 28, 2026
Environment: Raspberry Pi 4B, Rocky Linux 9.7, AdGuard Home v0.107.73