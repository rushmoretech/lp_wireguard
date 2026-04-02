# VPS Setup (Vultr)

This guide walks you through creating a cloud server on Vultr and configuring it
as a WireGuard hub. The VPS is the central relay point — the Raspberry Pi,
Android phone, and Windows service station all connect to it.

## Step 1: Create a Vultr Account

1. Go to [vultr.com](https://www.vultr.com) and create an account.
2. Add a payment method (credit card or PayPal).

## Step 2: Deploy a Server

1. Click **Deploy New Server** (the blue "+" button).
2. Choose the following settings:

| Setting | Value |
|---|---|
| Type | Cloud Compute — Shared CPU |
| Location | Pick the one closest to the client (e.g., Frankfurt for Europe) |
| Image | Debian 13 x64 (Trixie) |
| Plan | The cheapest one (1 vCPU, 1 GB RAM, 25 GB SSD) — this is more than enough |
| Additional Features | Leave defaults (no auto-backups needed for this) |
| SSH Keys | Upload your SSH public key (recommended) or use password auth |
| Hostname | `wg-hub` (or any name you prefer) |

3. Click **Deploy Now** and wait 1–2 minutes for the server to be created.
4. Once ready, note the **public IP address** shown in the Vultr dashboard. We
   will call this `VPS_PUBLIC_IP` throughout this guide.

## Step 3: Connect to the VPS

Open a terminal on your computer and SSH into the server:

```bash
ssh root@VPS_PUBLIC_IP
```

If you used a password instead of an SSH key, enter the password shown in the
Vultr dashboard under the server's overview page.

## Step 4: Update the System and Install Security Tools

```bash
apt update && apt upgrade -y
```

Install essential packages that Debian does not include by default:

```bash
apt install -y sudo ufw iptables
```

> **Why these packages?**
> - `sudo` — lets the non-root user run admin commands (needed after we lock root)
> - `ufw` — simple firewall manager (preinstalled on Ubuntu, but not on Debian)
> - `iptables` — packet filtering used by WireGuard PostUp rules (Debian 13
>   defaults to nftables, but the `iptables` package provides a compatibility
>   layer that WireGuard expects)

Install automatic security updates so the server patches itself:

```bash
apt install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```

> Select **Yes** when asked. From now on, the server will automatically install
> security patches daily.

Install **fail2ban** to block brute-force SSH attacks:

```bash
apt install -y fail2ban
systemctl enable fail2ban
systemctl start fail2ban
```

fail2ban is now running with sensible defaults — it will ban any IP that fails
SSH login 5 times within 10 minutes.

Reboot if a kernel update was installed:

```bash
reboot
```

Wait 30 seconds, then SSH back in.

## Step 5: Harden SSH

By default, the VPS allows SSH login as root with a password. This is a common
attack target — automated bots scan the internet 24/7 trying root passwords. We
will lock it down by:

1. Creating a regular (non-root) user
2. Setting up SSH key authentication for that user
3. Disabling root login and password authentication entirely

After this step, the only way to log in will be with your SSH key as the
`wgadmin` user. No password, no root.

### 5a: Create a regular user

```bash
adduser wgadmin
usermod -aG sudo wgadmin
```

`adduser` will ask you to set a password and fill in some optional info. Set a
**strong password** (you will need it when running `sudo` commands). The rest
(Full Name, Room Number, etc.) can be left blank — just press Enter.

`usermod -aG sudo wgadmin` adds the user to the `sudo` group, which allows them
to run admin commands with `sudo`.

### 5b: Set up SSH key authentication for the new user

There are two scenarios depending on how you deployed the VPS at Vultr.

---

**Scenario A: You uploaded an SSH key to Vultr during deployment**

Vultr already placed your public key in `/root/.ssh/authorized_keys`. Copy it
to the new user:

```bash
mkdir -p /home/wgadmin/.ssh
cp /root/.ssh/authorized_keys /home/wgadmin/.ssh/
chown -R wgadmin:wgadmin /home/wgadmin/.ssh
chmod 700 /home/wgadmin/.ssh
chmod 600 /home/wgadmin/.ssh/authorized_keys
```

Skip ahead to **Step 5c**.

---

**Scenario B: You deployed without an SSH key (password only)**

You are currently logged in as root using the password from the Vultr dashboard.
You need to generate an SSH key pair on **your local computer** (not the VPS) and
upload the public key to the VPS.

**On your local computer** (Linux, Mac, or Windows with Git Bash / PowerShell):

```bash
# Check if you already have a key
ls ~/.ssh/id_ed25519.pub
```

If the file exists, you already have a key — skip the generation step below.

If it does not exist, generate one:

```bash
ssh-keygen -t ed25519 -C "wgadmin@wg-hub"
```

It will ask:
- **File location** — press Enter to accept the default (`~/.ssh/id_ed25519`)
- **Passphrase** — enter a strong passphrase (recommended) or press Enter for
  none. The passphrase protects the key if someone steals the file from your
  computer.

Now display the **public** key:

```bash
cat ~/.ssh/id_ed25519.pub
```

You will see one line that starts with `ssh-ed25519 AAAA...`. Copy this entire
line.

**Back on the VPS** (still logged in as root), paste the public key into the
new user's authorized_keys:

```bash
mkdir -p /home/wgadmin/.ssh
nano /home/wgadmin/.ssh/authorized_keys
```

Paste the public key line you copied, save, and exit (`Ctrl+X`, then `Y`, then
Enter).

Set the correct permissions:

```bash
chown -R wgadmin:wgadmin /home/wgadmin/.ssh
chmod 700 /home/wgadmin/.ssh
chmod 600 /home/wgadmin/.ssh/authorized_keys
```

> **What did we just do?** SSH key authentication works like a lock and key. The
> **private key** stays on your local computer (in `~/.ssh/id_ed25519`) — never
> share this file. The **public key** goes on the server (in `authorized_keys`)
> — it acts like a lock that only your private key can open. When you connect,
> SSH verifies that your private key matches the public key on the server. No
> password is sent over the network.

---

### 5c: Test the new user before locking root

This is critical — do NOT skip this step. If the key setup has a problem, you
could lock yourself out.

Open a **new** terminal window (keep the current root session open as a safety
net) and try to log in as the new user:

```bash
ssh wgadmin@VPS_PUBLIC_IP
```

This should log you in **without asking for a password** (or asking only for your
SSH key passphrase if you set one). If it asks for `wgadmin`'s account password,
the SSH key is not working — go back to Step 5b and check the file permissions
and contents.

Once logged in, verify sudo works:

```bash
sudo whoami
```

It will ask for the `wgadmin` password (the one you set in Step 5a). It should
print `root`.

> **If this fails:** Do NOT proceed. Debug the issue while you still have the
> root session open in the other terminal. Common problems:
> - Wrong permissions on `.ssh/` or `authorized_keys` (must be 700 and 600)
> - Public key not pasted correctly (must be one single line)
> - Connecting with a different SSH key than the one you generated

### 5d: Disable root login and password authentication

Now that you have confirmed the `wgadmin` user works with SSH key auth, lock
everything else down:

```bash
sudo nano /etc/ssh/sshd_config
```

Find and change (or add) these three lines:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

> **What each line does:**
> - `PermitRootLogin no` — no one can SSH as root, period.
> - `PasswordAuthentication no` — passwords are disabled. Only SSH keys work.
>   This stops all brute-force password attacks.
> - `PubkeyAuthentication yes` — ensures key-based login is explicitly enabled
>   (it is the default, but being explicit is safer).

Save and exit, then restart the SSH service:

```bash
sudo systemctl restart ssh
```

> **Debian note:** The SSH service is called `ssh` on Debian (not `sshd` as on
> some other distributions).

### 5e: Verify the lockdown (from another terminal)

```bash
# This should FAIL (root login disabled):
ssh root@VPS_PUBLIC_IP

# This should FAIL (password login disabled):
ssh -o PubkeyAuthentication=no wgadmin@VPS_PUBLIC_IP

# This should WORK (SSH key login as wgadmin):
ssh wgadmin@VPS_PUBLIC_IP
```

If all three tests pass, SSH is fully hardened.

> **From this point on**, always use `ssh wgadmin@VPS_PUBLIC_IP` and prefix
> admin commands with `sudo`.

## Step 6: Install WireGuard

```bash
sudo apt install -y wireguard
```

Verify the installation:

```bash
wg --version
```

You should see something like `wireguard-tools v1.0.20210914` (or newer).

## Step 7: Generate Server Keys

WireGuard uses a pair of cryptographic keys (like a lock and a key). The
**private key** stays on the server, and the **public key** is shared with the
other devices (Raspberry Pi, Android phone, and Windows PC).

First, secure the WireGuard directory:

```bash
sudo chmod 700 /etc/wireguard
```

Generate keys with restricted permissions from the start:

```bash
sudo sh -c 'umask 077; wg genkey > /etc/wireguard/server_private.key'
sudo sh -c 'cat /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key'
```

View the keys (you will need these later):

```bash
sudo cat /etc/wireguard/server_public.key
```

**Write down the public key.** You will need it when configuring the Raspberry Pi,
Android phone, and Windows PC. **Never share the private key.**

> **Security note:** Only display the private key when you actually need to paste
> it into the config file. Avoid copying it into chat messages, emails, or notes.

## Step 8: Identify the Network Interface Name

Before writing the WireGuard config, you need to know the name of the VPS's main
internet-facing network interface. **Debian does not use `eth0`** — it uses
predictable names like `enp1s0`, `ens3`, or similar.

Run this command to find it:

```bash
ip -o -4 addr show | grep -v '127.0.0.1'
```

You will see output like:

```
2: enp1s0    inet 203.0.113.45/23 brd 203.0.113.255 ...
```

The interface name is the second field — in this example, `enp1s0`. **Write it
down.** You will use it in the next step everywhere you see `enp1s0`.

> **On Vultr**, common Debian interface names are `enp1s0` or `ens3`. If you see
> something different, use whatever your output shows.

## Step 9: Create the WireGuard Configuration

First, read the private key (you will paste it into the config):

```bash
sudo cat /etc/wireguard/server_private.key
```

Create the configuration file:

```bash
sudo nano /etc/wireguard/wg0.conf
```

Paste the following content. You need to replace **two** placeholder values:
- `SERVER_PRIVATE_KEY` — with the actual private key from Step 7
- `enp1s0` — with your actual interface name from Step 8 (if different)

```ini
[Interface]
# VPS WireGuard address
Address = 10.67.0.1/24

# WireGuard listen port
ListenPort = 51820

# Reduce MTU to prevent fragmentation (WireGuard adds ~60 bytes overhead)
MTU = 1420

# Server private key
PrivateKey = SERVER_PRIVATE_KEY

# Enable IP forwarding when the interface comes up
PostUp = sysctl -w net.ipv4.ip_forward=1

# NAT only VPN traffic (restrict source to WireGuard subnet)
# *** Replace enp1s0 with YOUR interface name from Step 8 ***
PostUp = iptables -t nat -A POSTROUTING -s 10.67.0.0/24 -o enp1s0 -j MASQUERADE

# Stateful forwarding: allow new+established from wg0, only established back
PostUp = iptables -A FORWARD -i wg0 -o enp1s0 -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT
PostUp = iptables -A FORWARD -i enp1s0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Allow peer-to-peer traffic within the VPN
PostUp = iptables -A FORWARD -i wg0 -o wg0 -j ACCEPT

# Block IPv6 on the tunnel to prevent leaks
PostUp = ip6tables -A FORWARD -i wg0 -j DROP
PostUp = ip6tables -A FORWARD -o wg0 -j DROP

# Clean up when the interface goes down
PostDown = iptables -t nat -D POSTROUTING -s 10.67.0.0/24 -o enp1s0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -o enp1s0 -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -D FORWARD -i enp1s0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -o wg0 -j ACCEPT
PostDown = ip6tables -D FORWARD -i wg0 -j DROP
PostDown = ip6tables -D FORWARD -o wg0 -j DROP

# ─── Peer: Raspberry Pi ─────────────────────────────────────────────
[Peer]
# Replace with the Raspberry Pi's public key (generated in the Pi guide)
PublicKey = RASPBERRY_PI_PUBLIC_KEY

# The Pi gets 10.67.0.2 and also "owns" the 192.168.1.0/24 local network
AllowedIPs = 10.67.0.2/32, 192.168.1.0/24

# Keep the tunnel alive (important because Pi is behind Starlink CGNAT)
PersistentKeepalive = 25

# ─── Peer: Android Phone ────────────────────────────────────────────
[Peer]
# Replace with the Android phone's public key (generated in the Android guide)
PublicKey = ANDROID_PHONE_PUBLIC_KEY

# The phone only gets its own VPN address
AllowedIPs = 10.67.0.3/32

# ─── Peer: Windows 10 Service Station ──────────────────────────────
[Peer]
# Replace with the Windows PC's public key (generated in the Windows guide)
PublicKey = WINDOWS_PC_PUBLIC_KEY

# The PC only gets its own VPN address
AllowedIPs = 10.67.0.4/32
```

> **Important:** The `AllowedIPs` for the Raspberry Pi includes `192.168.1.0/24`.
> This tells the VPS that any traffic destined for the client's local network
> should be sent through the tunnel to the Pi.
>
> **Security notes on the iptables rules above:**
> - MASQUERADE is restricted to source `10.67.0.0/24` — only VPN traffic gets
>   NATed, not everything on the server.
> - FORWARD uses **stateful filtering** — new connections can only be initiated
>   from the VPN side, not from the internet back into the VPN.
> - IPv6 is blocked on the tunnel to prevent traffic leaking outside the
>   encrypted VPN.

### Double-check the interface name

Before saving, verify that every occurrence of the interface name in the file
matches your actual interface. A quick way to count:

```bash
# This should return nothing — if it does, you forgot to replace a placeholder
grep 'eth0' /etc/wireguard/wg0.conf
```

## Step 10: Open the Firewall

Configure UFW with a **deny-by-default** policy and only allow what is needed.

### 10a: Allow packet forwarding through UFW

UFW on Debian blocks forwarded packets by default. Since the VPS needs to forward
traffic between WireGuard peers, we must change this:

```bash
sudo nano /etc/default/ufw
```

Find the line:

```
DEFAULT_FORWARD_POLICY="DROP"
```

Change it to:

```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

Save and exit.

> **Why is this safe?** The VPS is not a router — it has only one internet
> interface and the WireGuard tunnel. The WireGuard PostUp iptables rules already
> enforce stateful forwarding (only VPN traffic is forwarded, and only in the
> correct direction). Changing the UFW default just prevents UFW from blocking
> WireGuard's own rules.

### 10b: Configure UFW rules

```bash
# Set default policies: block everything incoming, allow outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH first (CRITICAL: do this before enabling, or you get locked out)
sudo ufw allow 22/tcp comment "SSH"

# Allow WireGuard
sudo ufw allow 51820/udp comment "WireGuard"

# Enable the firewall
sudo ufw enable

# Verify — you should see both rules and "Status: active"
sudo ufw status verbose
```

You should see output like:

```
Status: active
Default: deny (incoming), allow (outgoing), allow (routed)

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
51820/udp                  ALLOW IN    Anywhere
```

Note that the `routed` line now says `allow` (not `disabled`). This confirms
that forwarded traffic is allowed.

> **Why deny incoming by default?** This ensures that only SSH and WireGuard are
> reachable from the internet. Even if other software gets installed later, it
> will not be accidentally exposed.

## Step 11: Start WireGuard

Set correct permissions on the config file (it contains the private key):

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
```

Start the WireGuard interface:

```bash
sudo wg-quick up wg0
```

If there are no errors, enable it to start automatically on boot:

```bash
sudo systemctl enable wg-quick@wg0
```

## Step 12: Verify

Check that WireGuard is running:

```bash
sudo wg show
```

You should see output like:

```
interface: wg0
  public key: (your server public key)
  private key: (hidden)
  listening port: 51820
```

No peers will show as connected yet — that is normal. They will appear after you
configure the Raspberry Pi and the Android phone.

## Summary

At this point you have:

- [x] A Vultr VPS running Debian 13 (Trixie)
- [x] SSH hardened (no root login, key-only authentication)
- [x] fail2ban protecting against brute-force attacks
- [x] Automatic security updates enabled
- [x] WireGuard installed and configured as the hub
- [x] Stateful iptables rules with restricted MASQUERADE
- [x] IPv6 leak prevention on the tunnel
- [x] Firewall default-deny with only SSH and WireGuard open
- [x] WireGuard set to start on boot

**What you need for the next steps:**
- The **VPS public IP address** (from the Vultr dashboard)
- The **server public key** (from Step 7)

Next: [Raspberry Pi Setup](02-raspberry-pi-setup.md)
