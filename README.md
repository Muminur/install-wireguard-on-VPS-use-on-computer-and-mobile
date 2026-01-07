# How to Set Up Your Own WireGuard VPN on a VPS: Complete Guide for Windows & Mobile

*Secure your internet connection with a self-hosted VPN in under 30 minutes*

---

## Introduction

In an era of increasing online surveillance and data breaches, having your own VPN is no longer a luxury—it's a necessity. While commercial VPN services are convenient, running your own VPN server gives you complete control over your data and privacy.

In this guide, I'll walk you through setting up **WireGuard VPN** on a Virtual Private Server (VPS), then connecting to it from both your Windows PC and mobile phone.

### Why WireGuard?

- **Fast**: Uses state-of-the-art cryptography and runs in the kernel
- **Simple**: ~4,000 lines of code vs. 100,000+ for OpenVPN
- **Modern**: Built from the ground up with security in mind
- **Cross-platform**: Works on Linux, Windows, macOS, iOS, and Android

### What You'll Need

- A VPS running Ubuntu 22.04 or later (from providers like DigitalOcean, Linode, Vultr, etc.)
- SSH access to your VPS (root or sudo privileges)
- WireGuard client for your devices
- About 20-30 minutes of your time

---

## Part 1: Server Setup

### Step 1: Connect to Your VPS

Open your terminal (or PuTTY on Windows) and SSH into your server:

```bash
ssh root@YOUR_SERVER_IP
```

### Step 2: Install WireGuard

Update your system and install WireGuard along with QR code tools for easy mobile setup:

```bash
apt-get update
apt-get install -y wireguard wireguard-tools qrencode
```

### Step 3: Generate Server Keys

Create the WireGuard directory and generate your server's keypair:

```bash
mkdir -p /etc/wireguard
chmod 700 /etc/wireguard
cd /etc/wireguard

# Generate private and public keys
wg genkey | tee server_private.key | wg pubkey > server_public.key
chmod 600 server_private.key

# View your keys (you'll need these later)
echo "Private Key: $(cat server_private.key)"
echo "Public Key: $(cat server_public.key)"
```

> **Important**: Keep your private key secret! Never share it with anyone.

### Step 4: Generate Client Keys

For each device you want to connect, generate a unique keypair:

```bash
# For your Windows PC
wg genkey | tee client1_private.key | wg pubkey > client1_public.key

# For your phone
wg genkey | tee client2_private.key | wg pubkey > client2_public.key

chmod 600 client1_private.key client2_private.key
```

### Step 5: Find Your Network Interface

Before creating the config, identify your server's main network interface:

```bash
ip -4 route show default | grep -oP 'dev \K\S+'
```

This will output something like `eth0`, `ens3`, or `ens6`. Note this down—you'll need it for the configuration.

### Step 6: Create Server Configuration

Create the WireGuard configuration file. Replace the placeholders with your actual values:

```bash
cat > /etc/wireguard/wg0.conf << 'EOF'
[Interface]
Address = 10.10.0.1/24
ListenPort = 51820
PrivateKey = YOUR_SERVER_PRIVATE_KEY
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o YOUR_INTERFACE -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o YOUR_INTERFACE -j MASQUERADE
SaveConfig = false

[Peer]
# Windows PC
PublicKey = YOUR_CLIENT1_PUBLIC_KEY
AllowedIPs = 10.10.0.2/32

[Peer]
# Phone
PublicKey = YOUR_CLIENT2_PUBLIC_KEY
AllowedIPs = 10.10.0.3/32
EOF

chmod 600 /etc/wireguard/wg0.conf
```

**Replace:**
- `YOUR_SERVER_PRIVATE_KEY` with your server's private key
- `YOUR_INTERFACE` with your network interface (e.g., `eth0`)
- `YOUR_CLIENT1_PUBLIC_KEY` with your PC's public key
- `YOUR_CLIENT2_PUBLIC_KEY` with your phone's public key

### Step 7: Enable IP Forwarding

Allow your server to forward traffic between the VPN and the internet:

```bash
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p
```

### Step 8: Configure the Firewall

If you're using UFW (Ubuntu's default firewall), allow WireGuard traffic:

```bash
ufw allow 51820/udp
```

For servers with Docker, you may also need to configure UFW for forwarding:

```bash
# Edit UFW default forward policy
sed -i 's/DEFAULT_FORWARD_POLICY="DROP"/DEFAULT_FORWARD_POLICY="ACCEPT"/' /etc/default/ufw

# Add NAT rules
cat >> /etc/ufw/before.rules << 'EOF'

# WireGuard NAT rules
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.10.0.0/24 -o eth0 -j MASQUERADE
COMMIT
EOF

# Reload firewall
ufw disable && ufw enable
```

### Step 9: Start WireGuard

Enable and start the WireGuard service:

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

Verify it's running:

```bash
wg show
```

You should see output showing your interface and peers.

---

## Part 2: Windows Client Setup

### Step 1: Download WireGuard

Download the official WireGuard client from [wireguard.com/install](https://www.wireguard.com/install/).

### Step 2: Create Client Configuration

Create a new text file with this configuration:

```ini
[Interface]
PrivateKey = YOUR_CLIENT1_PRIVATE_KEY
Address = 10.10.0.2/24
DNS = 8.8.8.8, 1.1.1.1

[Peer]
PublicKey = YOUR_SERVER_PUBLIC_KEY
Endpoint = YOUR_SERVER_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

**Replace:**
- `YOUR_CLIENT1_PRIVATE_KEY` with the private key you generated for your PC
- `YOUR_SERVER_PUBLIC_KEY` with your server's public key
- `YOUR_SERVER_IP` with your VPS IP address

### Step 3: Import and Connect

1. Open the WireGuard application
2. Click **"Add Tunnel"** → **"Import tunnel(s) from file"**
3. Select your configuration file
4. Click **"Activate"** to connect

You should now be connected! Verify by visiting [whatismyip.com](https://whatismyip.com)—it should show your VPS IP address.

---

## Part 3: Mobile Setup (iOS/Android)

### Step 1: Install the App

Download WireGuard from:
- **iOS**: [App Store](https://apps.apple.com/app/wireguard/id1441195209)
- **Android**: [Play Store](https://play.google.com/store/apps/details?id=com.wireguard.android)

### Step 2: Generate QR Code (Easiest Method)

On your server, create a client config and generate a QR code:

```bash
# Create phone config
cat > /etc/wireguard/phone.conf << 'EOF'
[Interface]
PrivateKey = YOUR_CLIENT2_PRIVATE_KEY
Address = 10.10.0.3/24
DNS = 8.8.8.8, 1.1.1.1

[Peer]
PublicKey = YOUR_SERVER_PUBLIC_KEY
Endpoint = YOUR_SERVER_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF

# Generate QR code
qrencode -t ansiutf8 < /etc/wireguard/phone.conf
```

### Step 3: Scan and Connect

1. Open the WireGuard app on your phone
2. Tap the **"+"** button
3. Select **"Scan from QR code"**
4. Scan the QR code displayed in your terminal
5. Name your tunnel and save
6. Toggle the switch to connect

---

## Troubleshooting

### "Handshake did not complete"

- **Check firewall**: Ensure port 51820/UDP is open
- **Verify keys**: Make sure public/private keys match on both ends
- **Check endpoint**: Confirm the server IP and port are correct

### "DNS not resolving" / "Can't access websites"

This usually means traffic isn't being forwarded properly:

```bash
# Verify IP forwarding is enabled
cat /proc/sys/net/ipv4/ip_forward
# Should return: 1

# Check NAT rules
iptables -t nat -L POSTROUTING -v -n
```

### Subnet Conflicts

If you're using Docker or other services that use the `10.0.0.0/24` subnet, change WireGuard to use a different range like `10.10.0.0/24` or `10.100.0.0/24`.

---

## Adding More Clients

To add another device:

### 1. Generate new keys on the server:

```bash
cd /etc/wireguard
wg genkey | tee client3_private.key | wg pubkey > client3_public.key
```

### 2. Add the peer to WireGuard:

```bash
wg set wg0 peer NEW_CLIENT_PUBLIC_KEY allowed-ips 10.10.0.4/32
```

### 3. Create the client config:

Use the next available IP (10.10.0.4, 10.10.0.5, etc.) and follow the same pattern as above.

---

## Useful Commands

| Command | Description |
|---------|-------------|
| `wg show` | Display WireGuard status and peers |
| `wg show wg0 latest-handshakes` | Show when each peer last connected |
| `systemctl status wg-quick@wg0` | Check service status |
| `systemctl restart wg-quick@wg0` | Restart WireGuard |
| `journalctl -u wg-quick@wg0` | View WireGuard logs |

---

## Security Best Practices

1. **Keep private keys private**: Never share or expose your private keys
2. **Use strong key generation**: Always use `wg genkey` for cryptographically secure keys
3. **Limit peer access**: Only add peers you trust
4. **Monitor connections**: Regularly check `wg show` for unexpected peers
5. **Keep software updated**: Run `apt update && apt upgrade` regularly
6. **Consider key rotation**: Periodically generate new keys for enhanced security

---

## Performance Tips

- **Choose a nearby server location**: Lower latency = better performance
- **Use a quality VPS provider**: Look for providers with good network connectivity
- **MTU tuning**: If you experience issues, try adjusting the MTU in your config:
  ```ini
  [Interface]
  MTU = 1380
  ```

---

## Conclusion

Congratulations! You now have your own private VPN server running WireGuard. Your internet traffic is encrypted and routed through your VPS, protecting you from snooping on public WiFi and giving you more control over your online privacy.

### What's Next?

- Set up [Pi-hole](https://pi-hole.net/) alongside WireGuard for network-wide ad blocking
- Configure split tunneling to only route specific traffic through the VPN
- Set up multiple VPN servers in different regions for geo-flexibility

---

## Quick Reference Card

| Setting | Value |
|---------|-------|
| VPN Port | 51820/UDP |
| VPN Subnet | 10.10.0.0/24 |
| Server VPN IP | 10.10.0.1 |
| First Client IP | 10.10.0.2 |
| Second Client IP | 10.10.0.3 |
| Protocol | UDP |
| Encryption | ChaCha20, Poly1305, Curve25519 |

---

*Have questions or run into issues? Drop a comment below and I'll help you out!*

---

**Tags**: #VPN #WireGuard #Privacy #Security #SelfHosted #Linux #Ubuntu #Tutorial
