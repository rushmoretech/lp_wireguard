# Network Overview

This document describes the full network layout, IP addressing plan, and the role
of each device. Read this first so the rest of the guides make sense.

## Why Do We Need a VPN?

Starlink Gen 2 routers use something called **CGNAT** (Carrier-Grade NAT). In
simple terms, your Starlink connection does not get its own public IP address on
the internet. Many customers share the same public IP. Because of this:

- You **cannot** open ports on Starlink.
- You **cannot** set up port forwarding to reach devices behind it.
- Apps like iVMS-4500 **cannot** connect directly to the NVR from outside.

The solution is a **WireGuard VPN**. WireGuard creates an encrypted tunnel between
devices. We use a cheap cloud server (VPS) as a meeting point that both the
Raspberry Pi and the Android phone can reach.

## Network Diagram

```
                          ┌──────────────────────────┐
                          │        INTERNET           │
                          └──┬──────────┬──────────┬──┘
                             │          │          │
                             │          │          │
                   ┌─────────▼──┐ ┌─────▼────────┐ │
                   │ Vultr VPS  │ │ Android Phone│ │
                   │            │ │              │ │
                   │ Public IP: │ │ Mobile data  │ │
                   │ (assigned) │ │ or any WiFi  │ │
                   │            │ │              │ │
                   │ wg0:       │ │ wg0:         │ │
                   │ 10.0.0.1   │ │ 10.0.0.3    │ │
                   └──┬─────▲──┘ └──────────────┘ │
                      │     │                      │
           WireGuard  │     │  WireGuard           │
           tunnel     │     │  tunnels             │
                      │     │                      │
                      │     │    ┌─────────────────▼───┐
                      │     │    │ Windows 10 PC       │
                      │     │    │ (Service Station)   │
                      │     │    │                     │
                      │     │    │ wg0: 10.0.0.4       │
                      │     │    │ Full access to LAN  │
                      │     │    └─────────────────────┘
                      │     │
             ┌────────▼─────┴──────────────────────┐
             │         Raspberry Pi 4B              │
             │                                      │
             │  wlan0: ──► Starlink (WiFi)          │
             │             (gets IP via DHCP)        │
             │                                      │
             │  wg0:   10.0.0.2                     │
             │                                      │
             │  eth0:  192.168.1.10 ──► Local LAN   │
             └──────────────────┬───────────────────┘
                                │
                     ┌──────────▼──────────┐
                     │   Local Network     │
                     │   192.168.1.0/24    │
                     │                     │
                     │   Hikvision NVR     │
                     │   192.168.1.64      │
                     │   Port: 8000        │
                     │                     │
                     │   + all other LAN   │
                     │     devices         │
                     └─────────────────────┘
```

## IP Addressing Plan

| Device | Interface | IP Address | Purpose |
|---|---|---|---|
| Vultr VPS | enp1s0 (varies) | (public, assigned by Vultr) | Internet-facing |
| Vultr VPS | wg0 | 10.0.0.1/24 | WireGuard hub |
| Raspberry Pi | wlan0 | (DHCP from Starlink) | Internet via Starlink WiFi |
| Raspberry Pi | eth0 | 192.168.1.10 | Connected to client's local network |
| Raspberry Pi | wg0 | 10.0.0.2/24 | WireGuard tunnel to VPS |
| Android Phone | wg0 | 10.0.0.3/24 | WireGuard tunnel to VPS |
| Windows 10 PC | wg0 | 10.0.0.4/24 | WireGuard tunnel to VPS (service station) |
| Hikvision NVR | eth | 192.168.1.64 | Local network, port 8000 |

## WireGuard Port

All WireGuard traffic uses **UDP port 51820** (the default). This port must be
open on the VPS firewall.

## How Traffic Flows

When you open iVMS-4500 on the Android phone and connect to the NVR, here is
what happens step by step:

1. The phone sends a packet to **192.168.1.64:8000**.
2. The WireGuard app on the phone sees this is a tunneled address and encrypts
   the packet, sending it to the **VPS public IP on UDP 51820**.
3. The VPS receives the encrypted packet, decrypts it, and sees it is destined
   for **192.168.1.64** which belongs to the Raspberry Pi's network.
4. The VPS forwards the decrypted packet through the WireGuard tunnel to the
   **Raspberry Pi (10.0.0.2)**.
5. The Raspberry Pi receives the packet on its **wg0** interface. IP forwarding
   and NAT rules send it out through **eth0** to **192.168.1.64:8000**.
6. The NVR responds, and the reply follows the exact same path in reverse.

All of this happens in milliseconds. The phone behaves as if it were physically
plugged into the client's local network.

The **Windows 10 service station** works the same way, but with full access to
the entire 192.168.1.0/24 network on **all ports** (not just port 8000). This
allows a technician to remotely manage any device on the client's LAN — access
NVR web interfaces, SSH into switches, configure routers, run network scans, etc.

## Subnets Summary

| Subnet | Purpose | Used By |
|---|---|---|
| 10.0.0.0/24 | WireGuard VPN tunnel | VPS, Raspberry Pi, Android Phone, Windows PC |
| 192.168.1.0/24 | Client local network | Raspberry Pi (eth0), NVR, other local devices |

## Security Model

### What this setup protects against

- **Unauthorized access to the NVR:** The NVR is never exposed to the internet.
  Only devices with a valid WireGuard key pair can reach it.
- **Eavesdropping:** All traffic between devices is encrypted with modern
  cryptography (Curve25519, ChaCha20, Poly1305).
- **Brute-force attacks on SSH:** The VPS uses key-only authentication and
  fail2ban to block repeated login attempts.
- **Overly broad access:** The Android phone is restricted to the NVR on port
  8000 only. It cannot reach other devices on the LAN. Only the Windows service
  station has full LAN access.
- **IPv6 leaks:** IPv6 forwarding is blocked on the tunnel so traffic cannot
  bypass the VPN.

### What this setup does NOT protect against

- **VPS provider compromise:** The VPS provider (Vultr) has physical access to
  the server. If they are compromised, an attacker could extract WireGuard keys.
  This is a risk with any cloud server.
- **Compromised endpoint device:** If the Android phone or Windows PC is infected
  with malware, the attacker could use the VPN tunnel while it is active. Keep
  devices updated and use antivirus software.
- **Physical access to the Raspberry Pi:** Anyone who can physically reach the Pi
  can extract the microSD card and read the WireGuard private key. Place the Pi
  in a locked cabinet or enclosure if possible.
- **Starlink outages:** If Starlink goes down, the VPN tunnel breaks. There is
  no fallback internet connection in this setup.

### Per-Peer Access Control

| Peer | WireGuard IP | LAN Access |
|---|---|---|
| Android Phone | 10.0.0.3 | **Restricted:** only 192.168.1.64 on TCP port 8000 |
| Windows Service Station | 10.0.0.4 | **Full:** all devices on 192.168.1.0/24, all ports |

This enforcement happens on the Raspberry Pi via iptables rules. Even if a phone
user tries to access another device on the LAN, the Pi will drop the traffic.

### Key Management

- Each device has its own unique key pair (private + public key).
- Private keys **never** leave the device they were generated on.
- If a device is lost or compromised, remove its public key from the VPS and Pi
  configs and restart WireGuard. That instantly revokes its access.
- See [Troubleshooting — Revoking a Peer](04-troubleshooting.md#8-revoking-a-compromised-peer)
  for step-by-step instructions.
