# Raspberry Pi Setup

This guide configures the Raspberry Pi 4B as an on-site gateway. It connects to
the internet via Starlink WiFi (wlan0), sits on the client's local network via
Ethernet (eth0), and runs a WireGuard tunnel to the VPS.

## Prerequisites

- Raspberry Pi 4B with 4 GB RAM
- A microSD card (16 GB or larger) flashed with **Raspberry Pi OS Lite (64-bit)**
- An Ethernet cable connecting the Pi to the same switch/network as the NVR
- The Starlink Gen 2 router powered on and broadcasting WiFi
- SSH access to the Pi (either via keyboard+monitor or headless setup)
- From the VPS setup guide:
  - VPS public IP address (`VPS_PUBLIC_IP`)
  - VPS server public key (`SERVER_PUBLIC_KEY`)

## Step 1: Flash Raspberry Pi OS

1. Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/) on your
   computer.
2. Insert the microSD card.
3. Open Raspberry Pi Imager and choose:
   - **OS:** Raspberry Pi OS Lite (64-bit) — under "Raspberry Pi OS (other)"
   - **Storage:** Your microSD card
4. Click the **gear icon** (⚙) to open Advanced Options:
   - **Enable SSH:** Yes, use password authentication (or add your SSH key)
   - **Set username and password:** e.g., user `pi`, password `your-secure-password`
   - **Configure WiFi:** Enter the Starlink WiFi name (SSID) and password
   - **Set locale:** Your timezone and keyboard layout
5. Click **Write** and wait for it to finish.
6. Insert the microSD card into the Pi and power it on.

## Step 2: Find and Connect to the Pi

If you configured WiFi in Step 1, the Pi should connect to Starlink automatically.
Find its IP address by one of these methods:

- **From your router:** Check the Starlink app for connected devices.
- **From another computer on the same network:**
  ```bash
  # On Linux/Mac
  ping raspberrypi.local

  # Or scan the network (install nmap first)
  nmap -sn 192.168.1.0/24
  ```

SSH into the Pi:

```bash
ssh pi@RASPBERRY_PI_IP
```

## Step 3: Update the System

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

Wait a minute, then SSH back in.

## Step 4: Configure the Network Interfaces

The Pi has two network connections:
- **wlan0** — WiFi, connected to Starlink (internet access)
- **eth0** — Ethernet, connected to the local network (where the NVR is)

We need to give eth0 a **static IP** on the client's local network and make sure
the Pi uses wlan0 (Starlink) as its default route to the internet.

### Configure eth0 with a static IP

Edit the network configuration using NetworkManager (default on recent Raspberry
Pi OS):

```bash
sudo nmcli connection add \
  type ethernet \
  con-name "local-lan" \
  ifname eth0 \
  ipv4.method manual \
  ipv4.addresses 192.168.1.10/24 \
  ipv4.never-default yes
```

> **Key detail:** `ipv4.never-default yes` ensures the Pi does NOT try to use
> eth0 as its internet gateway. All internet traffic goes through wlan0
> (Starlink). Without this, the Pi could lose internet access.

Bring up the connection:

```bash
sudo nmcli connection up local-lan
```

### Verify both interfaces

```bash
ip addr show eth0
ip addr show wlan0
```

You should see:
- **eth0:** `192.168.1.10/24`
- **wlan0:** An IP assigned by Starlink via DHCP

> **Warning — Possible IP Conflict:** Starlink Gen 2 routers often use the
> `192.168.1.0/24` range by default — the same range as the client's local
> network. If both wlan0 and eth0 end up on `192.168.1.x`, routing will break.
>
> **Check now:** If `ip addr show wlan0` shows a `192.168.1.x` address, you must
> fix this before continuing. See [Troubleshooting — Starlink and Local Network
> IP Conflict](04-troubleshooting.md#4-starlink-and-local-network-ip-conflict)
> for two solutions (change the Starlink subnet or the local network subnet).

### Test connectivity

```bash
# Test internet (through wlan0 / Starlink)
ping -c 3 8.8.8.8

# Test local network (through eth0)
ping -c 3 192.168.1.64
```

Both should succeed. If the NVR ping fails, check the Ethernet cable and that
the NVR is on the same network.

## Step 5: Install WireGuard

```bash
sudo apt install -y wireguard
```

## Step 6: Generate Keys

Secure the WireGuard directory and generate keys with restricted permissions:

```bash
sudo chmod 700 /etc/wireguard
sudo sh -c 'umask 077; wg genkey > /etc/wireguard/pi_private.key'
sudo sh -c 'cat /etc/wireguard/pi_private.key | wg pubkey > /etc/wireguard/pi_public.key'
```

View the public key:

```bash
sudo cat /etc/wireguard/pi_public.key
```

**Write down the public key.** You need to paste it into the VPS configuration
(replace `RASPBERRY_PI_PUBLIC_KEY` in `wg0.conf` on the VPS).

> **Security note:** Only display the private key when you actually need to paste
> it into the config file. Do not share it via email, chat, or notes.

## Step 7: Create the WireGuard Configuration

```bash
sudo nano /etc/wireguard/wg0.conf
```

Paste the following. Replace the placeholder values:

First, read the private key (you will paste it into the config):

```bash
sudo cat /etc/wireguard/pi_private.key
```

```ini
[Interface]
# Raspberry Pi's WireGuard address
Address = 10.0.0.2/24

# Reduce MTU to prevent fragmentation (WireGuard adds ~60 bytes overhead)
MTU = 1420

# Pi's private key
PrivateKey = PI_PRIVATE_KEY

# Enable forwarding from WireGuard to the local network
PostUp = sysctl -w net.ipv4.ip_forward=1
PostUp = iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

# ── Access control ──────────────────────────────────────────────────
# Android phone (10.0.0.3): ONLY allow access to NVR on port 8000
PostUp = iptables -A FORWARD -i wg0 -o eth0 -s 10.0.0.3 -d 192.168.1.64 -p tcp --dport 8000 -j ACCEPT
PostUp = iptables -A FORWARD -i wg0 -o eth0 -s 10.0.0.3 -j DROP

# Windows service station (10.0.0.4): full LAN access (all devices, all ports)
PostUp = iptables -A FORWARD -i wg0 -o eth0 -s 10.0.0.4 -j ACCEPT

# Default: allow other VPN peers to reach the LAN (for future peers)
PostUp = iptables -A FORWARD -i wg0 -o eth0 -j ACCEPT

# Allow return traffic from LAN back into the tunnel
PostUp = iptables -A FORWARD -i eth0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Block IPv6 on the tunnel to prevent leaks
PostUp = ip6tables -A FORWARD -i wg0 -j DROP
PostUp = ip6tables -A FORWARD -o wg0 -j DROP

# ── Cleanup ─────────────────────────────────────────────────────────
PostDown = iptables -t nat -D POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -o eth0 -s 10.0.0.3 -d 192.168.1.64 -p tcp --dport 8000 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -o eth0 -s 10.0.0.3 -j DROP
PostDown = iptables -D FORWARD -i wg0 -o eth0 -s 10.0.0.4 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -o eth0 -j ACCEPT
PostDown = iptables -D FORWARD -i eth0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
PostDown = ip6tables -D FORWARD -i wg0 -j DROP
PostDown = ip6tables -D FORWARD -o wg0 -j DROP

[Peer]
# VPS public key (from the VPS setup guide, Step 7)
PublicKey = SERVER_PUBLIC_KEY

# Only the VPN subnet is routed through the tunnel
AllowedIPs = 10.0.0.0/24

# VPS public IP and WireGuard port
Endpoint = VPS_PUBLIC_IP:51820

# Keep the tunnel alive — critical because Starlink uses CGNAT
# Without this, the NAT mapping expires and the tunnel breaks
PersistentKeepalive = 25
```

> **Before saving:** Verify the `Endpoint` line shows an actual IP address
> (e.g., `203.0.113.45:51820`), not the placeholder text `VPS_PUBLIC_IP`.

Set correct permissions on the config file:

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
```

### What the iptables rules do (plain English)

- **MASQUERADE:** When traffic arrives from the VPN tunnel (10.0.0.x) and goes
  out to the local network (eth0), the Pi rewrites the source address to its own
  (192.168.1.10). This way, the NVR sees the traffic as coming from the Pi and
  knows where to send the reply.
- **Android restriction (10.0.0.3):** The phone can ONLY reach the NVR
  (192.168.1.64) on TCP port 8000. All other traffic from the phone to the LAN
  is dropped. This follows the **principle of least privilege** — the phone only
  needs iVMS-4500 access, so that is all it gets.
- **Windows full access (10.0.0.4):** The service station can reach any device
  on any port — it needs full access for remote administration.
- **Return traffic:** Only replies to connections initiated from the VPN side are
  allowed back through. Devices on the LAN cannot initiate connections into the
  tunnel.
- **IPv6 block:** Prevents traffic from leaking outside the encrypted tunnel via
  IPv6.

## Step 8: Update the VPS Configuration

Now go back to the VPS and replace the placeholder in `/etc/wireguard/wg0.conf`:

```bash
# On the VPS:
ssh wgadmin@VPS_PUBLIC_IP

# Edit the config
sudo nano /etc/wireguard/wg0.conf
```

Replace `RASPBERRY_PI_PUBLIC_KEY` with the actual public key from Step 6 above.

Then reload WireGuard on the VPS:

```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

## Step 9: Start WireGuard on the Pi

```bash
sudo wg-quick up wg0
```

If there are no errors, enable it to start automatically on boot:

```bash
sudo systemctl enable wg-quick@wg0
```

## Step 10: Verify the Tunnel

On the Pi:

```bash
sudo wg show
```

You should see the VPS listed as a peer with a recent handshake:

```
peer: (VPS public key)
  endpoint: VPS_PUBLIC_IP:51820
  allowed ips: 10.0.0.0/24
  latest handshake: X seconds ago
  transfer: X.XX KiB received, X.XX KiB sent
```

Test connectivity through the tunnel:

```bash
# Ping the VPS through WireGuard
ping -c 3 10.0.0.1
```

On the VPS, also verify:

```bash
# Ping the Raspberry Pi through WireGuard
ping -c 3 10.0.0.2

# Ping the NVR through the tunnel (should work if forwarding is correct)
ping -c 3 192.168.1.64
```

If the VPS can ping 192.168.1.64, the full path is working.

## Step 11: Make It Survive Reboots

WireGuard auto-start was already enabled in Step 9. Additionally, ensure the
network comes up properly on boot:

```bash
# Verify services are enabled
sudo systemctl is-enabled wg-quick@wg0
sudo systemctl is-enabled NetworkManager
```

Both should say `enabled`.

## Summary

At this point you have:

- [x] Raspberry Pi connected to Starlink via WiFi (wlan0)
- [x] Raspberry Pi connected to local network via Ethernet (eth0, 192.168.1.10)
- [x] WireGuard tunnel active between the Pi and the VPS
- [x] Traffic forwarding with per-peer access control (phone restricted to NVR:8000)
- [x] IPv6 leak prevention on the tunnel
- [x] Everything starts automatically on boot

**What you need for the next step:**
- The **VPS public IP address**
- The **VPS server public key**
- Confirmation that `ping 192.168.1.64` works from the VPS

Next: [Android Client Setup](03-android-client.md)
