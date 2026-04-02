# Windows 10 Service Station Setup

This guide configures a Windows 10 PC as a remote service station. Unlike the
Android phone (which only accesses the NVR on port 8000), this workstation gets
**full access to the entire 192.168.1.0/24 network on all ports**. This allows
a technician to:

- Access the NVR web interface (HTTP/HTTPS)
- SSH into network switches or other managed devices
- Use Hikvision SADP Tool to discover and configure cameras
- Run network diagnostics (ping, traceroute, port scans)
- Access any service on any device on the client's LAN

## Prerequisites

- A Windows 10 PC with internet access
- From previous guides:
  - VPS public IP address (`VPS_PUBLIC_IP`)
  - VPS server public key (`SERVER_PUBLIC_KEY`)

## Step 1: Download and Install WireGuard

1. Open a browser and go to the official WireGuard website:
   **https://www.wireguard.com/install/**
2. Click **Download Windows Installer**.
3. Run the downloaded `.msi` file.
4. Follow the installer — click **Next** through the prompts and then **Install**.
5. WireGuard will launch automatically after installation. You will see its icon
   in the system tray (bottom-right corner of the taskbar, near the clock).

## Step 2: Create a New Tunnel

1. Open the **WireGuard** application (click the tray icon or search for
   "WireGuard" in the Start menu).
2. Click **Add Tunnel** (bottom-left) > **Add empty tunnel...**
3. WireGuard will automatically generate a **private key** and **public key**.
   You will see them at the top of the window.

> **Important:** Copy the **public key** shown at the top. You need it for the
> VPS configuration (replace `WINDOWS_PC_PUBLIC_KEY` in the VPS wg0.conf).

## Step 3: Enter the Configuration

In the text editor area, **replace everything** with the following. Paste your
actual private key on the `PrivateKey` line (WireGuard already filled it in —
keep that value):

```ini
[Interface]
# Windows PC WireGuard address
Address = 10.0.0.4/24

# The private key was auto-generated — keep the one WireGuard created
PrivateKey = YOUR_AUTO_GENERATED_PRIVATE_KEY

# DNS is optional — only needed if you route all traffic through the VPN
# DNS = 1.1.1.1

[Peer]
# VPS public key
PublicKey = SERVER_PUBLIC_KEY

# Route VPN subnet + full client LAN through the tunnel
AllowedIPs = 10.0.0.0/24, 192.168.1.0/24

# VPS public IP and WireGuard port
Endpoint = VPS_PUBLIC_IP:51820

# Keep the tunnel alive through NAT
PersistentKeepalive = 25
```

4. In the **Name** field at the top, type: `nvr-service` (or any name you like).
5. Click **Save**.

### What does AllowedIPs mean here?

| Entry | What it does |
|---|---|
| `10.0.0.0/24` | Routes VPN traffic (to reach the VPS and Pi) through the tunnel |
| `192.168.1.0/24` | Routes the entire client LAN through the tunnel — **all ports, all devices** |

Everything else (web browsing, email, etc.) goes through the PC's normal internet
connection. This is **split tunneling** — only LAN and VPN traffic use the VPN.

## Step 4: Update the VPS Configuration

SSH into the VPS and add the Windows PC's public key:

```bash
ssh wgadmin@VPS_PUBLIC_IP
sudo nano /etc/wireguard/wg0.conf
```

Replace `WINDOWS_PC_PUBLIC_KEY` with the actual public key you copied in Step 2.

Reload WireGuard:

```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

## Step 5: Connect the VPN

1. In the WireGuard app on Windows, select the **nvr-service** tunnel.
2. Click **Activate**.
3. The status should change to **Active** with a green indicator. You will see
   data transfer counters start ticking.

### Verify the connection

Open **Command Prompt** (search for `cmd` in the Start menu) and run:

```cmd
ping 10.0.0.1
ping 10.0.0.2
ping 192.168.1.64
```

All three should reply. If they do, you have full access to the client's network.

## Step 6: Test Full Network Access

Now try accessing different services on the client's LAN:

### NVR Web Interface

Open a browser and go to:

```
http://192.168.1.64
```

You should see the Hikvision NVR login page. Log in with the NVR admin
credentials.

### NVR Management Port

If you have iVMS-4200 (the desktop version of Hikvision's app) installed:

1. Open iVMS-4200.
2. Go to **Device Management** > **Add Device**.
3. Enter:
   - **Address:** `192.168.1.64`
   - **Port:** `8000`
   - **User/Password:** NVR admin credentials
4. Click **Add**.

### Discover Other Devices

Open Command Prompt and scan the LAN:

```cmd
:: Ping sweep — find devices on the network
for /L %i in (1,1,254) do @ping -n 1 -w 200 192.168.1.%i | find "Reply"
```

This sends one ping to every address from 192.168.1.1 to 192.168.1.254 and shows
which ones respond. It takes about a minute to complete.

### Access Any Port on Any Device

Since the tunnel routes the full `192.168.1.0/24` subnet, you can access any
device on any port. Examples:

| Service | How to access |
|---|---|
| NVR web interface | `http://192.168.1.64` (browser) |
| NVR management | `192.168.1.64:8000` (iVMS-4200) |
| Router admin panel | `http://192.168.1.1` (browser) |
| SSH to a device | `ssh admin@192.168.1.x` (terminal) |
| RDP to a PC | `mstsc /v:192.168.1.x` (Remote Desktop) |
| Any TCP/UDP port | Works through the tunnel transparently |

## Step 7: Optional — Auto-Connect on Startup

If you want WireGuard to connect automatically when Windows starts:

1. Open the **WireGuard** app.
2. Go to the **nvr-service** tunnel.
3. Check the box **Activate on launch** (under the tunnel name, if available).

Alternatively, WireGuard runs as a Windows service. You can set it to start
automatically:

1. Press `Win+R`, type `services.msc`, press Enter.
2. Find **WireGuard Tunnel: nvr-service**.
3. Double-click it, set **Startup type** to **Automatic**.
4. Click **OK**.

> **Note:** Only enable auto-connect if this PC is always used as the service
> station. For occasional use, it is better to connect manually.

## Step 8: Windows Firewall Note

Windows Firewall may block incoming connections on the WireGuard interface. If
you are trying to reach this PC from another VPN device (e.g., ping 10.0.0.4
from the VPS), you may need to allow it:

1. Open **Windows Defender Firewall with Advanced Security** (search for it in
   the Start menu).
2. Click **Inbound Rules** > **New Rule**.
3. Choose **Custom** > **All programs**.
4. Protocol: **ICMPv4** (for ping) or **Any** (for all traffic).
5. Remote IP: `10.0.0.0/24`.
6. Action: **Allow**.
7. Apply to **all profiles** (Domain, Private, Public).
8. Name it `Allow WireGuard VPN` and click **Finish**.

This is only needed if you want other VPN peers to reach the Windows PC. It is
not required for the PC to reach the client's LAN (outbound connections work
without this).

## Summary

At this point you have:

- [x] WireGuard installed on Windows 10
- [x] VPN tunnel to the VPS configured and active
- [x] Full access to the client's 192.168.1.0/24 network on all ports
- [x] Ability to manage any device on the client's LAN remotely

### Differences from the Android Phone Setup

| Feature | Android Phone (10.0.0.3) | Windows Service Station (10.0.0.4) |
|---|---|---|
| Primary use | View cameras via iVMS-4500 | Full remote network management |
| LAN access enforced by Pi | **Restricted:** NVR port 8000 only | **Full:** all devices, all ports |
| AllowedIPs (client side) | `10.0.0.0/24, 192.168.1.0/24` | `10.0.0.0/24, 192.168.1.0/24` |
| Typical tools | iVMS-4500 | iVMS-4200, browser, SSH, RDP, SADP Tool |

> **Note:** Both devices have the same client-side `AllowedIPs`, but the
> Raspberry Pi enforces different access levels via iptables. The phone is
> restricted to the NVR on port 8000; the Windows PC has full LAN access.
> This is enforced server-side regardless of what the client device attempts.

### Security Warning

Because this workstation has **full access to the client's entire local network**,
treat it with the same security rigor as any remote administration tool:

- **Keep Windows updated** — enable automatic updates.
- **Use a strong Windows account password** and enable screen lock.
- **Enable Windows Defender** (or equivalent antivirus) — a compromised service
  station gives an attacker access to the entire client LAN.
- **Disconnect the VPN when not in use** — do not leave it always-on unless
  necessary. Minimize the window of exposure.
- **If this PC is lost or compromised**, revoke its key immediately. See
  [Troubleshooting — Revoking a Peer](04-troubleshooting.md#8-revoking-a-compromised-peer).

If something is not working, see: [Troubleshooting](04-troubleshooting.md)
