# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Documentation-only project. Contains step-by-step guides for setting up a WireGuard VPN
that allows remote access to a Hikvision NVR (192.168.1.64:8000) from an Android phone
(iVMS-4500 app), bypassing Starlink Gen 2 CGNAT via a Vultr VPS relay.

## Architecture

Four devices form the VPN (subnet 10.0.0.0/24):

- **Vultr VPS (10.0.0.1)** — Debian 13 (Trixie), WireGuard hub with a public IP, relays traffic between peers
- **Raspberry Pi 4B (10.0.0.2)** — on-site gateway: wlan0 connects to Starlink WiFi for
  internet, eth0 (192.168.1.10) connects to the local LAN where the NVR sits. Forwards
  and NATs traffic between the WireGuard tunnel and the local network.
- **Android phone (10.0.0.3)** — mobile client running WireGuard + iVMS-4500
- **Windows 10 PC (10.0.0.4)** — service station with full access to 192.168.1.0/24 on all ports

## Documentation Structure

All guides are in `docs/` and meant to be followed in order:

- `docs/00-overview.md` — IP plan, network diagram, traffic flow explanation
- `docs/01-vps-setup.md` — Vultr VPS creation, WireGuard server config, firewall
- `docs/02-raspberry-pi-setup.md` — Pi networking (dual WiFi+Ethernet), WireGuard client, IP forwarding
- `docs/03-android-client.md` — WireGuard app config, iVMS-4500 device setup
- `docs/05-windows-workstation.md` — Windows 10 service station with full LAN access
- `docs/04-troubleshooting.md` — Diagnostics for tunnel, routing, NVR connectivity issues

## Key Details

- VPS runs Debian 13 — interface name is NOT eth0 (typically enp1s0 on Vultr, check with `ip addr`)
- VPS SSH user is `wgadmin` (root login disabled)
- WireGuard port: UDP 51820
- NVR: Hikvision at 192.168.1.64, management port 8000
- Starlink Gen 2 has no Ethernet — Pi connects via WiFi only
- PersistentKeepalive = 25 is critical on all peers behind NAT/CGNAT
- Pi must use `ipv4.never-default yes` on eth0 to avoid routing internet through the LAN
