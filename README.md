> **Disclaimer:**  
This guide is provided for educational purposes only.  
> The author assumes no responsibility for any network issues, data loss, or damages resulting from the use of this information.  
> You are solely responsible for your network security and compliance with your local laws and ISP terms of service.

> [!CAUTION]
> **Use at your own risk.**

# Linux VPN Gateway for the Entire Home Network

Problem: some devices (for example, TVs and set-top boxes) do not support VPN client installation, and some VPN solutions cannot be shared across the whole local network out of the box.

Solution: set up a dedicated Linux gateway and route selected devices through it, so VPN works even where direct client installation is impossible.

You can use either of these options:
- an old laptop;
- or a Linux virtual machine with `2 GB RAM`.

This guide uses **Ubuntu Desktop** to keep compatibility with GUI-only VPN clients.

Tasks:
- ensure compatibility with any VPN type (OpenVPN, WireGuard, etc.);
- implement a kill switch so traffic cannot bypass VPN if the tunnel fails;
- minimize DNS/WebRTC/IPv6 leaks;
- split devices: some through VPN, some without VPN;
- keep admin access to the gateway from the local network (SSH/RDP);
- keep the gateway stable for 24/7 operation.

## Important Before You Start

> [!CAUTION]  
> **Before you proceed:** Modifying system networking rules can cause loss of connectivity.  
> Please ensure you have physical access to the device or a backup connection method to revert changes if necessary.

> [!WARNING]
This guide assumes an IPv4-only setup (no IPv6).
If your home network uses IPv6, ask AI to adapt these instructions.

Example network parameters:
- router: `192.168.0.1`
- home subnet: `192.168.0.0/24`
- VPN gateway: `192.168.0.3`

## Step 1. Assign a Static IP to the Linux Gateway

1. Click the network icon (top right) and open `Settings`.
2. Go to `Network.`
3. Click the gear icon next to your Wired/Ethernet connection.
4. Open the `IPv4` tab.
5. Change mode from Automatic (DHCP) to `Manual`.
6. Fill in these values:
   - Address: `192.168.0.3`
   - Netmask: `255.255.255.0`
   - Gateway: `192.168.0.1`
   - DNS: `192.168.0.3`
7. Click `Apply` and reboot the system to apply changes.

## Step 2. Create a systemd Service for Gateway Rules

Create `vpn-gateway.service` in `/etc/systemd/system/`:

```bash
[Unit]
Description=VPN Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/bin/bash /opt/vpn-rules.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

## Step 3. Create the Rules Script `/opt/vpn-rules.sh`

File content:

```bash
#!/bin/bash
export PATH=/usr/sbin:/usr/bin:/sbin:/bin

# fail on errors
set -e

VPN_INTF="tun0" # Set your vpn interface name
LAN_INTF="eth0" # Set your lan inerface name (usually eth0)
LAN_IP="192.168.x.x" # Set your gateway ip
LAN_SUBNET="192.168.x.0/24" # Set your lan subnet
VPN_IP="xxx.xxx.xxx.xxx" # Set your VPN server external/public IP (outside your local network)
MSS="<VPN MTU - 80>" # Check VPN interface MTU and calculate MSS with a safety margin
TTL="64" # Set fixed TTL value

echo "Setup started..."

### SYSTEM ###

# IPv4 forwarding
sysctl -w net.ipv4.ip_forward=1

# do not redirect clients to another gateway via ICMP
sysctl -w net.ipv4.conf.all.send_redirects=0
sysctl -w net.ipv4.conf.default.send_redirects=0
sysctl -w net.ipv4.conf."$LAN_INTF".send_redirects=0

# fully disable IPv6 (to prevent leaks)
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.default.disable_ipv6=1
sysctl -w net.ipv6.conf.lo.disable_ipv6=1

### IPTABLES RESET ###
iptables -F
iptables -t nat -F
iptables -t mangle -F
iptables -X

### DEFAULT POLICIES ###
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

# Loopback
iptables -A INPUT  -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# ICMP
iptables -A INPUT  -p icmp -j ACCEPT
iptables -A OUTPUT -p icmp -j ACCEPT
iptables -A FORWARD -p icmp -j ACCEPT

### ESTABLISHED / RELATED ###
iptables -A INPUT   -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT  -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

### RDP (ONLY FROM LAN, PORTS 3389 & 3390) ###
for port in 3389 3390; do
  iptables -A INPUT  -p tcp -s "$LAN_SUBNET" --dport $port -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
  iptables -A OUTPUT -p tcp --sport $port -d "$LAN_SUBNET" -m conntrack --ctstate ESTABLISHED -j ACCEPT
done


### SSH (ONLY FROM LAN, PORT 22 ) ###

# incoming SSH from LAN
iptables -A INPUT \
  -i "$LAN_INTF" \
  -p tcp \
  -s "$LAN_SUBNET" \
  --dport 22 \
  -m conntrack --ctstate NEW,ESTABLISHED \
  -j ACCEPT

# outgoing SSH replies to LAN
iptables -A OUTPUT \
  -o "$LAN_INTF" \
  -p tcp \
  -d "$LAN_SUBNET" \
  --sport 22 \
  -m conntrack --ctstate ESTABLISHED \
  -j ACCEPT

### ALLOW DIRECT ACCESS TO VPN SERVER (ANY TRAFFIC) ###
iptables -A OUTPUT -o "$LAN_INTF" -d "$VPN_IP" -j ACCEPT

### ALLOW LAN CLIENTS TO ACCESS VPN SERVER DIRECTLY ###
iptables -A FORWARD \
  -i "$LAN_INTF" \
  -s "$LAN_SUBNET" \
  -d "$VPN_IP" \
  -j ACCEPT

iptables -t nat -A POSTROUTING \
  -s "$LAN_SUBNET" \
  -d "$VPN_IP" \
  -o "$LAN_INTF" \
  -j MASQUERADE

### WAIT FOR VPN INTERFACE ###
echo "Waiting for VPN interface $VPN_INTF..."
while ! ip link show "$VPN_INTF" >/dev/null 2>&1; do
    sleep 1
done
echo "VPN interface $VPN_INTF is up, applying VPN setup rules..."

### RP_FILTER
for i in all default "$LAN_INTF" "$VPN_INTF"; do
  # this is required specifically for VPN routing
  # (otherwise Linux drops asymmetric routes)
  sysctl -w net.ipv4.conf.$i.rp_filter=0
done

### DNS THROUGH VPN ONLY ###
iptables -A INPUT -i "$LAN_INTF" -d "$LAN_IP" -s "$LAN_SUBNET" -p udp --dport 53 -j ACCEPT
iptables -A INPUT -i "$LAN_INTF" -d "$LAN_IP" -s "$LAN_SUBNET" -p tcp --dport 53 -j ACCEPT
iptables -A OUTPUT -o "$VPN_INTF" -p udp --dport 53 -j ACCEPT
iptables -A OUTPUT -o "$VPN_INTF" -p tcp --dport 53 -j ACCEPT

### LAN -> VPN (ALL TRAFFIC) ###
iptables -A FORWARD -i "$LAN_INTF" -o "$VPN_INTF" -s "$LAN_SUBNET" -j ACCEPT

### Ubuntu -> VPN ###
iptables -A OUTPUT -o "$VPN_INTF" -j ACCEPT

### NAT ###
iptables -t nat -A POSTROUTING -s "$LAN_SUBNET" -o "$VPN_INTF" -j MASQUERADE

### TTL FIX ###
iptables -t mangle -A POSTROUTING -o "$VPN_INTF" -j TTL --ttl-set "$TTL"

### MSS FIX ###
iptables -t mangle -A FORWARD -o "$VPN_INTF" -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss "$MSS"
iptables -t mangle -A FORWARD -i "$VPN_INTF" -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss "$MSS"

echo "[OK] Setup completed: VPN gateway rules fully applied"
```

After saving the file, enable and start the service:

```bash
systemctl enable vpn-gateway.service
systemctl start vpn-gateway.service
```

## Step 4. Configure DNS via systemd-resolved

Open `/etc/systemd/resolved.conf`.

In the DNS section, use the following values (copy the part after `#` too):

```text
DNS=1.1.1.1#cloudflare-dns.com 1.0.0.1#cloudflare-dns.com
DNS=8.8.8.8#dns.google 8.8.4.4#dns.google
```

Disable fallback DNS:

```bash
FallbackDNS=
```

Also ensure these parameters are set:

```bash
DNSOverTLS=yes
DNSStubListener=yes
DNSStubListenerExtra=<YOUR_GATEWAY_IP> # In our case it is 192.168.0.3
LLMNR=no
MulticastDNS=no
Cache=no-negative
CacheFromLocalhost=no
```

Apply the settings:

```bash
sudo systemctl restart systemd-resolved
```

## Step 5. Disable System Sleep

To prevent the gateway from sleeping:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

## Step 6. Configure Client Devices (TV, etc.)

On each device that should access the internet through the VPN gateway, set these Wi-Fi network parameters:
- DNS: `<YOUR_GATEWAY_IP>`
- Gateway: `<YOUR_GATEWAY_IP>`
- Subnet mask: `255.255.255.0` for `/24`

For this guide's example, use `192.168.0.3`.

> [!CAUTION]
Do not mix up the subnet mask, or a phone may work at first and then lose internet access after a few hours.

## 🟢 Congratulations! 🟢
After that, the device traffic will go through your Linux VPN gateway.
