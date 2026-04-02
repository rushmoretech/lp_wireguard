# Troubleshooting

Common problems and how to fix them.

## 1. WireGuard Tunnel Not Connecting

**Symptoms:** `wg show` on the Pi or VPS shows no recent handshake. Pings to
10.67.0.1 or 10.67.0.2 time out.

### Check the basics

```bash
# On the Pi — is WireGuard running?
sudo wg show

# Is the interface up?
ip addr show wg0
```

If wg0 does not exist, start it:

```bash
sudo wg-quick up wg0
```

### Check for key mismatches

The most common error is pasting the wrong key. Verify:

- The **VPS wg0.conf** has the Pi's **public** key (not private).
- The **Pi wg0.conf** has the VPS's **public** key (not private).
- The **Android config** has the VPS's **public** key.

Keys always come in pairs. A public key looks like `aB3cD4eF5gH6iJ7...=`. If you
see two identical keys in the same config file, something is wrong.

### Check the VPS firewall

```bash
# On the VPS
sudo ufw status
```

You should see `51820/udp ALLOW Anywhere`. If not:

```bash
sudo ufw allow 51820/udp
```

### Check the endpoint

On the Pi, verify the VPS IP and port in `/etc/wireguard/wg0.conf`:

```
Endpoint = VPS_PUBLIC_IP:51820
```

Make sure this matches the actual public IP shown in the Vultr dashboard.

## 2. Tunnel Works, but Cannot Reach the NVR (192.168.1.64)

**Symptoms:** Pinging 10.67.0.1 and 10.67.0.2 works, but pinging 192.168.1.64
from the VPS or phone fails.

### Check IP forwarding on the Pi

```bash
cat /proc/sys/net/ipv4/ip_forward
```

Should return `1`. If it returns `0`:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

### Check iptables rules on the Pi

```bash
sudo iptables -t nat -L -n
sudo iptables -L FORWARD -n
```

You should see:
- A MASQUERADE rule on the nat table for the `10.67.0.0/24` source going out eth0
- FORWARD rules allowing traffic between wg0 and eth0

If missing, restart WireGuard (which re-applies the PostUp rules):

```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

### Check the VPS routing

On the VPS, verify the Pi's AllowedIPs includes the local network:

```bash
sudo wg show
```

Under the Pi's peer entry, you should see:

```
allowed ips: 10.67.0.2/32, 192.168.1.0/24
```

If `192.168.1.0/24` is missing, edit `/etc/wireguard/wg0.conf` on the VPS and
add it, then reload:

```bash
wg-quick down wg0 && wg-quick up wg0
```

### Check the NVR is alive

From the Pi (not through the tunnel, but directly on eth0):

```bash
ping -c 3 192.168.1.64
```

If this fails, the problem is between the Pi and the NVR:
- Check the Ethernet cable.
- Verify the NVR is powered on.
- Verify the NVR's IP is actually 192.168.1.64 (check on the NVR screen or via
  the Hikvision SADP tool).

## 3. Starlink WiFi Issues

### Pi cannot connect to Starlink WiFi

```bash
# Check WiFi status
nmcli device status

# Check available networks
nmcli device wifi list
```

If the Starlink network is not visible, the Pi might be too far from the router.
Move it closer or use a USB WiFi adapter with better range.

To reconnect:

```bash
sudo nmcli device wifi connect "STARLINK_SSID" password "STARLINK_PASSWORD"
```

### Pi has internet on WiFi but tunnel does not work

Check that wlan0 is the default gateway (not eth0):

```bash
ip route
```

The default route should go through wlan0:

```
default via 192.168.1.1 dev wlan0 proto dhcp ...
```

If the default route goes through eth0, that means the Pi is trying to reach the
VPS through the local network (which will not work). Fix it:

```bash
sudo nmcli connection modify "local-lan" ipv4.never-default yes
sudo nmcli connection up "local-lan"
```

## 4. Starlink and Local Network IP Conflict

**Symptoms:** Both Starlink and the local network use `192.168.1.0/24`. This
causes routing confusion.

### Solution A: Change the local network subnet

If you control the local network router, change it to a different subnet, for
example `192.168.2.0/24`. Then update:

1. The NVR's IP (e.g., to `192.168.2.64`)
2. The Pi's eth0 IP (e.g., to `192.168.2.10`)
3. The VPS wg0.conf (AllowedIPs for the Pi: `10.67.0.2/32, 192.168.2.0/24`)
4. The Android AllowedIPs (`10.67.0.0/24, 192.168.2.0/24`)

### Solution B: Change the Starlink subnet

In the Starlink app:
1. Go to **Settings** > **Advanced** > **Custom DNS / DHCP**.
2. Change the LAN range to something like `192.168.0.0/24`.

This keeps the local network (192.168.1.0/24) unchanged.

## 5. iVMS-4500 Cannot Connect to the NVR

**Symptoms:** The VPN is on (key icon in status bar), but iVMS-4500 shows
"Device Offline" or connection timeout.

### Check VPN connectivity first

Open Termux (or any terminal app) on the phone:

```
ping 192.168.1.64
```

- **If ping fails:** The VPN tunnel or routing is broken. Go back to sections
  1–3 above.
- **If ping works:** The network path is fine. The problem is with iVMS-4500
  settings.

### Verify iVMS-4500 settings

1. Open iVMS-4500 > Menu > Devices.
2. Tap on your NVR device.
3. Verify:
   - **Address:** `192.168.1.64` (not a domain or public IP)
   - **Port:** `8000` (not 80, 443, or 8080)
   - **Username/Password:** Correct NVR admin credentials

### Test the NVR port directly

From the Pi:

```bash
nc -zv 192.168.1.64 8000
```

Should say `Connection ... succeeded!` or `open`. If it says connection refused,
the NVR's management port may be different. Check the NVR settings or try common
Hikvision ports: 8000, 8200, 80, 443.

## 6. VPN Works Initially but Drops After Some Time

### Check PersistentKeepalive

Both the Pi and the Android phone are behind NAT (Starlink CGNAT and mobile
carrier NAT). Without keepalive packets, the NAT mapping expires and the tunnel
breaks silently.

Verify `PersistentKeepalive = 25` is set in:
- Pi's `wg0.conf` under the `[Peer]` section
- Android WireGuard app under the peer settings

### Check Pi uptime and WireGuard status

```bash
# Is the Pi still running?
uptime

# Is WireGuard still active?
sudo systemctl status wg-quick@wg0
```

If WireGuard crashed, check the logs:

```bash
journalctl -u wg-quick@wg0 --no-pager -n 50
```

## 7. Slow Video Feed

### Check Starlink speed

On the Pi:

```bash
# Install speedtest
sudo apt install -y speedtest-cli
speedtest
```

Hikvision cameras at standard quality need about 2–4 Mbps per camera. If Starlink
upload speed is below this, the feed will be laggy.

### Reduce stream quality

In iVMS-4500:
1. During live view, tap the **SD/HD** button.
2. Switch to **Fluent** or **Sub-stream** instead of **Main stream**.

This significantly reduces bandwidth requirements.

## 8. Revoking a Compromised Peer

If a device is lost, stolen, or compromised, remove its access immediately.

### Step 1: Remove the peer from the VPS

```bash
ssh wgadmin@VPS_PUBLIC_IP
sudo nano /etc/wireguard/wg0.conf
```

Delete the entire `[Peer]` block for the compromised device (both the `PublicKey`
and `AllowedIPs` lines). Then reload:

```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

### Step 2: Remove from the Pi (if the peer had LAN access rules)

If you added per-peer iptables rules on the Pi (as documented in the Pi guide),
also remove those. Edit `/etc/wireguard/wg0.conf` on the Pi and remove the
relevant PostUp/PostDown lines, then reload:

```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

### Step 3: Verify

```bash
sudo wg show
```

The revoked peer should no longer appear. Even if the attacker has the old
private key, the VPS will reject their connection because their public key is
no longer in the config.

> **No need to regenerate other keys.** WireGuard's cryptography ensures that
> removing one peer does not affect the security of the remaining peers. Only
> regenerate ALL keys if you suspect the **VPS private key** was compromised.

## 9. Rotating Keys (Planned Maintenance)

Periodically rotating keys is good practice, even if no compromise is suspected.

### On the device you want to rotate (e.g., the Pi):

```bash
# Generate a new key pair
sudo sh -c 'umask 077; wg genkey > /etc/wireguard/pi_private_new.key'
sudo sh -c 'cat /etc/wireguard/pi_private_new.key | wg pubkey > /etc/wireguard/pi_public_new.key'

# Show the new public key
sudo cat /etc/wireguard/pi_public_new.key
```

### On the VPS:

Update the peer's `PublicKey` in `/etc/wireguard/wg0.conf` with the new public
key, then reload:

```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

### Back on the Pi:

Update `PrivateKey` in `/etc/wireguard/wg0.conf` with the new private key, then:

```bash
# Replace old key files
sudo mv /etc/wireguard/pi_private_new.key /etc/wireguard/pi_private.key
sudo mv /etc/wireguard/pi_public_new.key /etc/wireguard/pi_public.key

# Reload
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

> **Downtime:** The tunnel will be down for a few seconds during the reload on
> both sides. Coordinate the changes.

## Quick Reference: Key Commands

```bash
# ─── On the VPS ─────────────────────────────
sudo wg show                          # WireGuard status and connected peers
sudo wg-quick down wg0                # Stop WireGuard
sudo wg-quick up wg0                  # Start WireGuard
sudo systemctl restart wg-quick@wg0   # Restart WireGuard
sudo cat /etc/wireguard/wg0.conf      # View configuration (root only)
journalctl -u wg-quick@wg0 -n 20     # View recent logs

# ─── On the Raspberry Pi ────────────────────
sudo wg show                          # WireGuard status
sudo wg-quick down wg0                # Stop WireGuard
sudo wg-quick up wg0                  # Start WireGuard
ip route                              # Check routing table
ping -c 3 192.168.1.64                # Test NVR reachability
sudo iptables -t nat -L -n            # Check NAT rules
nmcli device status                   # Check network interfaces
```
