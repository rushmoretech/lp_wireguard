# WireGuard VPN Setup: Remote Access to Hikvision NVR via Starlink

This project documents a complete WireGuard VPN configuration that allows remote
access to a Hikvision NVR from an Android phone, using a Vultr VPS as the relay
server and a Raspberry Pi 4B as the on-site gateway.

## The Problem

A client has a Hikvision NVR (Network Video Recorder) on a local network behind
a Starlink Gen 2 router. Starlink uses CGNAT (Carrier-Grade NAT), which means
there is no way to open ports or forward traffic directly to the NVR from the
internet. The client wants to view their cameras remotely using the iVMS-4500
Android app.

## The Solution

We build a WireGuard VPN tunnel that bypasses Starlink's CGNAT:

```
┌─────────────────┐         ┌──────────────────┐         ┌─────────────────────────────┐
│  Android Phone  │         │   Vultr VPS      │         │   Client Site               │
│  (iVMS-4500)    │◄───────►│   WireGuard Hub  │◄───────►│   Raspberry Pi 4B           │
│  10.67.0.3       │ tunnel  │   10.67.0.1       │ tunnel  │   10.67.0.2                  │
└─────────────────┘         │                  │         │                             │
                            │                  │         │  wlan0 ─► Starlink (WiFi)   │
┌─────────────────┐         │                  │         │  eth0  ─► Local Network     │
│  Windows 10 PC  │◄───────►│                  │         │            192.168.1.0/24   │
│  (Service Stn)  │ tunnel  │                  │         │               │             │
│  10.67.0.4       │         │                  │         │           Hikvision NVR     │
└─────────────────┘         └──────────────────┘         │           192.168.1.64:8000 │
                                                         └─────────────────────────────┘
```

The Android phone connects to the VPS, the VPS relays traffic to the Raspberry Pi,
and the Pi forwards it to the NVR on the local network. All traffic is encrypted.

## Documentation

Follow the guides in order:

1. [Network Overview](docs/00-overview.md) — IP addressing plan and how everything connects
2. [VPS Setup (Vultr)](docs/01-vps-setup.md) — Create and configure the cloud server
3. [Raspberry Pi Setup](docs/02-raspberry-pi-setup.md) — Configure the on-site gateway
4. [Android Client Setup](docs/03-android-client.md) — Set up the phone for remote access
5. [Windows Service Station](docs/05-windows-workstation.md) — Set up the Windows 10 PC for full LAN access
6. [Troubleshooting](docs/04-troubleshooting.md) — Common problems and how to fix them

## Requirements

| Component | Details |
|---|---|
| Vultr VPS | Cheapest plan, Debian 13 (Trixie) |
| Raspberry Pi 4B | 4 GB RAM, Raspberry Pi OS Lite (64-bit) |
| Starlink | Gen 2 router (WiFi only, no Ethernet port) |
| Hikvision NVR | IP: 192.168.1.64, management port: 8000 |
| Android phone | iVMS-4500 app + WireGuard app |
| Windows 10 PC | WireGuard app (service station — full LAN access) |
