# Android Client Setup

This guide configures the Android phone to connect to the WireGuard VPN and
access the Hikvision NVR through the iVMS-4500 app.

## Prerequisites

- An Android phone with internet access (mobile data or WiFi)
- From previous guides:
  - VPS public IP address (`VPS_PUBLIC_IP`)
  - VPS server public key (`SERVER_PUBLIC_KEY`)

## Step 1: Install the WireGuard App

1. Open **Google Play Store** on the phone.
2. Search for **WireGuard**.
3. Install the app by **WireGuard Development Team** (the official one).

## Step 2: Create a New Tunnel

1. Open the **WireGuard** app.
2. Tap the **blue "+" button** at the bottom right.
3. Choose **Create from scratch**.

## Step 3: Fill In the Configuration

### Interface section (your phone)

| Field | Value |
|---|---|
| Name | `nvr-vpn` (or any name you like) |
| Private key | Tap **Generate** — the app creates a key pair automatically |
| Addresses | `10.67.0.3/24` |
| DNS servers | `1.1.1.1` (or leave empty — only needed if you route all traffic) |

> **Important:** After tapping Generate, the **public key** appears below the
> private key. You will need this public key for the VPS configuration. Tap it
> to copy it.

### Peer section (the VPS)

Tap **Add Peer** and fill in:

| Field | Value |
|---|---|
| Public key | `SERVER_PUBLIC_KEY` (the VPS public key from Step 6 of the VPS guide) |
| Endpoint | `VPS_PUBLIC_IP:51820` |
| Allowed IPs | `10.67.0.0/24, 192.168.1.0/24` |
| Persistent keepalive | `25` |

> **What does AllowedIPs mean here?** It tells the phone which traffic should go
> through the VPN tunnel:
> - `10.67.0.0/24` — traffic to other VPN devices
> - `192.168.1.0/24` — traffic to the client's local network (where the NVR is)
>
> All other traffic (web browsing, YouTube, etc.) goes through the phone's normal
> internet connection. This is called **split tunneling**.

4. Tap the **save icon** (floppy disk) at the top right.

## Step 4: Update the VPS Configuration

The VPS needs to know the phone's public key. SSH into the VPS:

```bash
ssh wgadmin@VPS_PUBLIC_IP
sudo nano /etc/wireguard/wg0.conf
```

Replace `ANDROID_PHONE_PUBLIC_KEY` with the public key you copied from the
WireGuard app in Step 3.

Reload WireGuard on the VPS:

```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

## Step 5: Connect the VPN

1. Open the **WireGuard** app on the phone.
2. Toggle the switch next to **nvr-vpn** to **ON**.
3. Android will ask you to allow a VPN connection — tap **OK**.
4. The toggle should turn green, and you will see a small **key icon** in the
   status bar at the top of the screen.

### Quick test

Open a browser on the phone and go to:

```
http://10.67.0.1
```

If the VPS has nothing running on port 80, you will get a connection refused
error — that is fine. The important thing is that the request does not time out.
A timeout means the tunnel is not working.

Better test — open a terminal app (like Termux) and run:

```
ping 10.67.0.1
ping 10.67.0.2
ping 192.168.1.64
```

All three should respond.

## Step 6: Install and Configure iVMS-4500

1. Open **Google Play Store**.
2. Search for **iVMS-4500** and install it (by Hangzhou Hikvision Digital Technology).
3. Open the app and go through the initial setup (accept permissions, etc.).

### Add the NVR device

1. Tap the **menu icon** (☰) at the top left.
2. Tap **Devices**.
3. Tap the **"+"** button at the top right.
4. Choose **Manual Adding**.
5. Fill in the following:

| Field | Value |
|---|---|
| Alias | `Client NVR` (or any name) |
| Register Mode | **IP/Domain** |
| Address | `192.168.1.64` |
| Port | `8000` |
| User Name | The NVR admin username (usually `admin`) |
| Password | The NVR admin password |

6. Tap **Save**.

> **Why 192.168.1.64 and not a public IP?** Because the WireGuard VPN makes your
> phone behave as if it is on the same local network as the NVR. You use the
> NVR's local IP address directly.

### View the cameras

1. Go back to the main screen.
2. Tap **Live View**.
3. Tap the **"+"** button to add cameras.
4. Select your NVR from the device list.
5. Choose the cameras you want to view and tap **Start Live View**.

You should now see the camera feeds.

## Step 7: Daily Usage

- **To view cameras remotely:** Open the WireGuard app, toggle the VPN on, then
  open iVMS-4500 and go to Live View.
- **When done:** Toggle the VPN off in the WireGuard app to save battery.
- **The VPN works on any internet connection** — mobile data, home WiFi, hotel
  WiFi, etc. It does not matter where the phone is.

### Optional: Auto-connect with iVMS-4500

If the client wants, you can set the WireGuard tunnel to be "always on":

1. Go to Android **Settings** > **Network & Internet** > **VPN**.
2. Tap the gear icon next to **WireGuard**.
3. Enable **Always-on VPN**.

This keeps the VPN running at all times. Note that all traffic to `10.67.0.0/24`
and `192.168.1.0/24` will always go through the tunnel, but since we used split
tunneling, regular internet use is not affected.

## Security Note

The Android phone's access is **restricted on the Raspberry Pi side**. Even
though the phone's WireGuard config routes the full `192.168.1.0/24` subnet
through the tunnel, the Pi only allows traffic from this phone to reach
**192.168.1.64 on TCP port 8000** (the NVR). All other traffic from the phone
to the client's LAN is dropped.

This means if the phone is lost or compromised, an attacker can only reach the
NVR's management port — not the entire network.

## Summary

At this point you have:

- [x] WireGuard app installed and configured on the Android phone
- [x] VPN tunnel connecting the phone to the VPS (and through it, to the Pi)
- [x] iVMS-4500 configured to connect to the NVR at 192.168.1.64:8000
- [x] Access restricted to NVR port 8000 only (enforced by the Pi)
- [x] Remote camera viewing working from anywhere

If something is not working, see: [Troubleshooting](04-troubleshooting.md)
