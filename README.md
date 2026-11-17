# Nexcloud-Wireguard-Setup

# Self-Hosted Nextcloud with WireGuard: Setup and Hardening Guide

This guide documents a secure, reusable setup for accessing a self-hosted Nextcloud instance from a phone or computer through WireGuard, while also allowing controlled HTTPS access from a trusted computer on the home LAN.

> Replace these placeholders throughout:
>
> - `YOUR_DDNS_HOSTNAME` — your DDNS hostname, for example `example.ddns.net`
> - `NEXTCLOUD_SERVER_LAN_IP` — LAN IP of the Raspberry Pi running Nextcloud, for example `192.168.1.50`
> - `TRUSTED_PC_LAN_IP` — LAN IP of the computer allowed to access Nextcloud directly, for example `192.168.1.105`
> - `WIREGUARD_SUBNET` — WireGuard client subnet, for example `10.20.100.0/24`
> - `HOME_LAN_SUBNET` — home LAN subnet, for example `192.168.1.0/24`
> - `WIREGUARD_UDP_PORT` — WireGuard listening port, commonly `51820`

## Architecture

- **Raspberry Pi: hosts WireGuard and Nextcloud; Nextcloud data is stored on an external drive.
- **Nextcloud**: served by Apache.
- **Phone**: accesses Nextcloud remotely through WireGuard using `https://YOUR_DDNS_HOSTNAME/nextcloud`.
- **Arch Linux PC**: accesses Nextcloud locally through the same HTTPS hostname, using a local DNS override.
- **Router**: forwards TCP ports 80 and 443 to the Raspberry Pi for certificate issuance/renewal and optional public HTTPS access.
- **UFW**: controls inbound access on the Raspberry Pi.
- **firewalld**: controls inbound access on the Arch Linux PC.

## Install Nextcloud

DietPi provides Nextcloud through its software installer:

```bash
dietpi-software
```

Select the Nextcloud option and follow the installer prompts.

After installation, Nextcloud is commonly available at:

```text
http://pi.example.local/nextcloud
```

DietPi documents that its Nextcloud installation is accessible through HTTP/HTTPS ports 80/443 under the `/nextcloud` path. 

Create the Nextcloud administrator account during setup. Use a unique, long password and store it in a password manager.

## Move Nextcloud data to the external drive

Confirm the current data directory:

```bash
sudo -u www-data php /var/www/nextcloud/occ config:system:get datadirectory
```

For this setup, the data directory is:

```text
/mnt/external-drive/nextcloud/data
```

Confirm that the external drive is mounted and has enough space:

```bash
findmnt -T /mnt/external-drive
df -h /mnt/external-drive
```

A healthy result shows the drive mounted at `/mnt/external-drive`, with adequate free space and read-write access.

## Install WireGuard with PiVPN

DietPi supports PiVPN and WireGuard through its software menu:

```bash
dietpi-software
```

Choose the VPN option and select WireGuard as the VPN server. DietPi’s documentation describes PiVPN installation and notes that WireGuard server or client configurations can be managed with DietPi-WireGuard. 

After installation, create a client profile for the phone:

```bash
pivpn add
```

Follow the prompts to create a client profile. PiVPN provides commands for managing the VPN server and generating client configurations. 

## Configure the WireGuard network

Use a private address range that does not conflict with the home network.

Example:

```text
Home network: 192.168.1.0/24
WireGuard network: 10.0.0.0/24
```

The Pi acts as the WireGuard server. The phone is a peer.

A basic server configuration has this structure:

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

[Peer]
PublicKey = PHONE_PUBLIC_KEY
AllowedIPs = 10.0.0.2/32
```

A basic phone configuration has this structure:

```ini
[Interface]
PrivateKey = PHONE_PRIVATE_KEY
Address = 10.0.0.2/32
DNS = 192.168.1.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = home.example.ddns.net:51820
AllowedIPs = 192.168.1.0/24
PersistentKeepalive = 25
```

WireGuard configurations consist of an interface and one or more peers, with keys, allowed IP ranges, and endpoint details. 

Important: do not paste real private keys into documentation, chat messages, screenshots, or version control.

## Security model

The preferred remote-access path is WireGuard. Public HTTPS access is required only for Let’s Encrypt HTTP-01 certificate issuance and renewal.

| Access path                              | Protocol                |                 Port | Access control                 |
| ---------------------------------------- | ----------------------- | -------------------: | ------------------------------ |
| Remote phone access                      | HTTPS through WireGuard |                  443 | WireGuard client configuration |
| Trusted PC local access                  | HTTPS                   |                  443 | UFW source-IP rule             |
| Let’s Encrypt validation / HTTP redirect | HTTP                    |                   80 | Open to the internet           |
| WireGuard VPN                            | UDP                     | `WIREGUARD_UDP_PORT` | WireGuard keys and firewall    |

Keep TCP port 80 open if using Let’s Encrypt HTTP-01 certificate renewal. Restrict TCP port 443 on the LAN to only the trusted PC if desired.

## Prerequisites

Before starting, ensure:

- The Raspberry Pi runs DietPi and has a static DHCP reservation in the router.
- Nextcloud is installed and reachable locally at `http://NEXTCLOUD_SERVER_LAN_IP/nextcloud`.
- The external storage drive is mounted and used for Nextcloud data.
- WireGuard is installed and configured on the Raspberry Pi.
- A DDNS hostname is registered with a DNS provider such as No-IP.
- The router forwards the required TCP ports to the Raspberry Pi.
- The Arch Linux PC has a firewall installed, such as `firewalld`.

## Router port forwarding

Create two TCP forwarding rules in the router:

| External port | Protocol | Internal IP | Internal port | Purpose |
|---:|---|---|---:|---|
| 80 | TCP | `NEXTCLOUD_SERVER_LAN_IP` | 80 | Let’s Encrypt HTTP-01 validation and HTTP-to-HTTPS redirect |
| 443 | TCP | `NEXTCLOUD_SERVER_LAN_IP` | 443 | HTTPS access to Nextcloud |
| `WIREGUARD_UDP_PORT` | UDP | `NEXTCLOUD_SERVER_LAN_IP` | `WIREGUARD_UDP_PORT` | WireGuard VPN |

Important:

- Use TCP for ports 80 and 443.
- Use UDP for WireGuard.
- Do not put a port number in the internal IP field.
- If the router uses port 80 or 443 for its own administration interface, change the router administration port to avoid a conflict.
- If the router WAN IP differs from the public IP reported by an external IP-check service, the connection may be behind CGNAT and inbound forwarding may not work.

## Configure DDNS

Register a DDNS hostname, such as `YOUR_DDNS_HOSTNAME`, and ensure its public DNS record points to the home network’s current public IPv4 address.

On the Raspberry Pi, verify the public DNS record:

```bash
dig +short YOUR_DDNS_HOSTNAME
```

or:

```bash
nslookup YOUR_DDNS_HOSTNAME
```

Verify the public IP as seen from the Raspberry Pi:

```bash
curl -4 ifconfig.me
```

or:

```bash
curl -4 ipinfo.io/ip
```

The DDNS record and the public IP should match. If they differ, update the DDNS record in the DNS provider.

## Enable HTTPS on Nextcloud

Apache must listen on both TCP port 80 and TCP port 443.

Check current listeners:

```bash
sudo ss -tulpn | grep -E ':80|:443'
```

If only port 80 is listening, HTTPS has not been configured.

Run DietPi’s Let’s Encrypt tool:

```bash
sudo dietpi-letsencrypt
```

When prompted:

1. Install Certbot: **Yes**
2. Select Apache as the web server.
3. Enter `YOUR_DDNS_HOSTNAME`.
4. Provide an email address for certificate notices.
5. Enable the HTTP-to-HTTPS redirect.
6. Apply the configuration.

After completion, verify:

```bash
sudo ss -tulpn | grep -E ':80|:443'
```

Expected output should include both:

```text
*:80   apache2
*:443  apache2
```

## Let’s Encrypt requirements

The standard HTTP-01 certificate challenge requires Let’s Encrypt to reach the Raspberry Pi from the internet on TCP port 80.

- The public DDNS record must point to the current public IP.
- The router must forward external TCP port 80 to the Raspberry Pi’s internal TCP port 80.
- UFW on the Raspberry Pi must allow inbound TCP port 80.
- Apache must be listening on TCP port 80.

Do not use the WireGuard UDP port for certificate validation. Let’s Encrypt HTTP-01 validation uses TCP port 80.

Test from a phone using mobile data, with home Wi-Fi and WireGuard disabled:

```text
http://YOUR_DDNS_HOSTNAME/nextcloud
```

It should reach Apache and redirect to HTTPS.

If certificate issuance fails with `Timeout during connect (likely firewall problem)`, check:

1. The DDNS record points to the current public IP.
2. The router forwards TCP 80 and TCP 443 to the Raspberry Pi.
3. UFW allows inbound TCP 80 and TCP 443.
4. Apache is running and listening on port 80.
5. The ISP does not block inbound port 80 or use CGNAT.

Use [Let’s Debug](https://letsdebug.net/) to diagnose HTTP-01 validation issues.

## Configure Nextcloud trusted domains

Nextcloud rejects access through hostnames not listed in `trusted_domains`.

On the Raspberry Pi, identify the next unused trusted-domain index:

```bash
sudo -u www-data php /var/www/nextcloud/occ config:system:get trusted_domains
```

Add the DDNS hostname:

```bash
sudo -u www-data php /var/www/nextcloud/occ config:system:set trusted_domains 1 --value=YOUR_DDNS_HOSTNAME
```

If index `1` is already in use, use the next available index, such as `2`:

```bash
sudo -u www-data php /var/www/nextcloud/occ config:system:set trusted_domains 2 --value=YOUR_DDNS_HOSTNAME
```

Verify:

```bash
sudo -u www-data php /var/www/nextcloud/occ config:system:get trusted_domains
```

Recommended configuration:

```php
'trusted_domains' => [
    0 => 'localhost',
    1 => 'YOUR_DDNS_HOSTNAME',
],
```

If access will always use the DDNS hostname, the server’s private LAN IP does not need to be added. The hostname is what appears in the browser URL.

## Local DNS override on Arch Linux

A public DDNS hostname normally resolves to the home public IP. When at home, configure the PC to resolve that hostname to the Raspberry Pi’s LAN IP instead. This is called split-horizon DNS.

On the Arch Linux PC, edit `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Add:

```text
NEXTCLOUD_SERVER_LAN_IP YOUR_DDNS_HOSTNAME
```

Example:

```text
192.168.1.50 example.ddns.net
```

Important:

- Do not include `https://`.
- Do not include `/nextcloud`.
- Do not include a port such as `:51820`.
- The hostname maps only to an IP address; the browser determines the port from the URL.

Flush DNS only if `systemd-resolved` is active:

```bash
systemctl is-active systemd-resolved
```

If active:

```bash
sudo resolvectl flush-caches
```

If inactive, restart NetworkManager instead:

```bash
sudo systemctl restart NetworkManager
```

Verify the override:

```bash
getent hosts YOUR_DDNS_HOSTNAME
```

Expected result:

```text
NEXTCLOUD_SERVER_LAN_IP YOUR_DDNS_HOSTNAME
```

Also verify:

```bash
ping -c 3 YOUR_DDNS_HOSTNAME
```

## Access Nextcloud locally

With WireGuard disabled on the Arch PC, open:

```text
https://YOUR_DDNS_HOSTNAME/nextcloud
```

The browser automatically uses TCP port 443 for HTTPS.

Test from the terminal:

```bash
curl -Iv https://YOUR_DDNS_HOSTNAME/nextcloud
```

A successful connection should complete a TLS handshake and return an HTTP status such as `200`, `301`, or `302`.

If the browser shows **Access through untrusted domain**, add the DDNS hostname to Nextcloud’s `trusted_domains`.

## Configure UFW on the Raspberry Pi

UFW controls inbound access to the Raspberry Pi.

Check current rules:

```bash
sudo ufw status verbose
sudo ufw status numbered
```

### Required rules

Allow HTTP for Let’s Encrypt renewal and HTTP-to-HTTPS redirect:

```bash
sudo ufw allow 80/tcp comment 'Let’s Encrypt HTTP-01 renewal and HTTP redirect'
```

Allow HTTPS from the trusted PC only:

```bash
sudo ufw allow from TRUSTED_PC_LAN_IP to any port 443 proto tcp comment 'Nextcloud HTTPS from trusted PC'
```

Allow WireGuard clients to reach Nextcloud:

```bash
sudo ufw allow from WIREGUARD_SUBNET to any port 443 proto tcp comment 'Nextcloud HTTPS via WireGuard'
```

Allow the WireGuard VPN port:

```bash
sudo ufw allow WIREGUARD_UDP_PORT/udp comment 'WireGuard VPN'
```

Reload and verify:

```bash
sudo ufw reload
sudo ufw status numbered
```

### Avoid broad HTTPS access

Do not use this rule if the goal is to restrict direct LAN HTTPS access:

```bash
sudo ufw allow 443/tcp
```

Instead, use the source-specific rule for the trusted PC:

```bash
sudo ufw allow from TRUSTED_PC_LAN_IP to any port 443 proto tcp
```

### Keep port 80 open

Do not restrict TCP port 80 to only the trusted PC if using Let’s Encrypt HTTP-01 renewal. Let’s Encrypt validation servers must be able to reach port 80 from the internet.

Port 80 is used for:

- Let’s Encrypt HTTP-01 validation.
- Redirecting HTTP requests to HTTPS.
- Automatic certificate renewal.

## Remove an incorrect UFW rule

List rules with numbers:

```bash
sudo ufw status numbered
```

Delete a rule by number:

```bash
sudo ufw delete RULE_NUMBER
sudo ufw reload
```

Example:

```bash
sudo ufw delete 3
```

Always rerun `sudo ufw status numbered` after deleting a rule because remaining rules are renumbered.

Delete by exact rule syntax:

```bash
sudo ufw delete allow from HOME_LAN_SUBNET to any port 80 proto tcp
```

Avoid `sudo ufw reset` unless intentionally erasing all UFW rules; it can cause loss of remote access.

## Harden the Arch Linux PC

Arch Linux may include `firewalld` or `ufw`, depending on the edition, but the firewall is not enabled by default.

Check installed firewalls:

```bash
systemctl status firewalld
systemctl status ufw
command -v firewall-cmd
command -v ufw
```

If using `firewalld`, enable it:

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --state
```

View active rules:

```bash
sudo firewall-cmd --list-all
```

View persistent rules:

```bash
sudo firewall-cmd --permanent --list-all
```

### Prevent inbound SSH to the PC

If the PC does not need to accept SSH connections, remove the SSH service from `firewalld`:

```bash
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --list-all
```

`ssh` should no longer appear under `services`.

If a rich rule allows SSH on a custom port, remove it:

```bash
sudo firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" source address="SOURCE_IP" port port="CUSTOM_PORT" protocol="tcp" accept'
sudo firewall-cmd --reload
```

Check whether the SSH server is listening:

```bash
sudo ss -tulpn | grep ':22'
```

If SSH server access is not needed, disable it:

```bash
sudo systemctl disable --now sshd
```

Check for systemd socket activation:

```bash
systemctl status ssh.socket --no-pager
```

If active or enabled:

```bash
sudo systemctl disable --now ssh.socket
sudo systemctl disable --now sshd
```

Verify port 22 is no longer listening:

```bash
sudo ss -tulpn | grep ':22'
```

## Harden the Raspberry Pi

The Raspberry Pi hosts WireGuard and Nextcloud, so keep its attack surface small.

### SSH access

Allow SSH only from the trusted administration computer:

```bash
sudo ufw allow from TRUSTED_PC_LAN_IP to any port 22 proto tcp comment 'SSH from trusted admin PC'
```

Do not use a broad rule such as:

```bash
sudo ufw allow 22/tcp
```

Use SSH key authentication rather than passwords where possible. Do not place the PC’s primary private SSH key on the Raspberry Pi.

### Disable unnecessary SSH forwarding

Edit the SSH server configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

Set or add:

```text
AllowTcpForwarding no
AllowStreamLocalForwarding no
GatewayPorts no
PermitTunnel no
X11Forwarding no
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

These settings reduce the chance that a compromised Raspberry Pi can create tunnels back into the LAN. They do not protect against an attacker with root access to the Pi.

### Limit services

Review listening services:

```bash
sudo ss -tulpn
```

Disable services that are not needed:

```bash
sudo systemctl disable --now SERVICE_NAME
```

Keep the Raspberry Pi updated:

```bash
sudo apt update
sudo apt upgrade
```

## Update and maintain Nextcloud

On DietPi, use the recommended update process:

```bash
sudo dietpi-update
```

Also update Nextcloud through its administrative interface when prompted.

Maintain backups of:

- Nextcloud database.
- Nextcloud configuration directory.
- Nextcloud data directory, including the external storage drive.
- WireGuard configuration and private keys, stored securely and separately from the server.

Test restores periodically. A backup that has never been restored is not a verified backup.

## Certificate renewal

Check certificates:

```bash
sudo certbot certificates
```

Check the renewal timer:

```bash
sudo systemctl list-timers | grep certbot
```

Test renewal without making changes:

```bash
sudo certbot renew --dry-run
```

If using HTTP-01 validation, TCP port 80 must remain reachable from the internet for renewal.
