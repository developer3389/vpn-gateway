> **Disclaimer:**  
This guide is provided for educational purposes only.  
> The author assumes no responsibility for any network issues, data loss, or damages resulting from the use of this information.  
> You are solely responsible for your network security and compliance with your local laws and ISP terms of service.

> [!CAUTION]
> **Use at your own risk.**

## Table of Contents
- [Linux VPN Gateway for the Entire Home Network](#linux-vpn-gateway-for-the-entire-home-network)
- [Important Before You Start](#important-before-you-start)
- [Step 1. Assign a Static IP](#step-1-assign-a-static-ip-to-the-linux-gateway)
- [Step 2. Disable System Sleep](#step-2-disable-system-sleep)
- [Step 3. Enable Auto-Login (Optional)](#step-3-enable-auto-login-optional)
- [Step 4. Prepare a Test Device](#step-4-prepare-a-test-device)
- [Step 5. Create the Rules Script](#step-5-create-the-rules-script-optvpn-rulessh)
- [Step 6. Create a systemd Service](#step-6-create-a-systemd-service-for-gateway-rules)
- [Step 7. Configure DNS via systemd-resolved](#step-7-configure-dns-via-systemd-resolved)
- [Step 8. Final Verification](#step-8-final-verification)
- [Appendix: How to set up a VPN cascade](#appendix-how-to-set-up-a-vpn-cascade)

---

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
> **VPN server compatibility**:  
> This guide assumes an `IPv4-based` VPN server. If your VPN provider or server requires `IPv6` connectivity, these instructions will not work as written. In that case, please ask AI to adapt this guide for an `IPv6-capable` gateway.

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

## Step 2. Disable System Sleep

To prevent the gateway from sleeping:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

## Step 3. Enable Auto-Login (Optional)

> [!NOTE]
> This is a practical approach for GUI-based VPN clients that require an active desktop session to start.
> If you are using a command-line (CLI) VPN client, you can skip this step and run the VPN as a systemd service instead.

- Open `Settings` > `Users`.
- Unlock the settings (top right).
- Toggle `Automatic Login` to `ON`.

This way, the VPN client will start automatically after every reboot.

## Step 4. Prepare a Test Device

You will use the same test device several times while verifying the setup.
Before you test routing, configure it with:
- DNS: `8.8.8.8 8.8.4.4`
- Gateway: `<YOUR_GATEWAY_IP>`
- Subnet mask: `255.255.255.0` for `/24`

For this example, use `192.168.0.3` as the gateway IP.

> [!CAUTION]
Do not mix up the subnet mask, or a phone may work at first and then lose internet access after a few hours.

Later, you will reconfigure this same device again in Step 7 for DNS verification, and again in Step 8 for the final client setup.

## Step 5. Create the Rules Script `/opt/vpn-rules.sh`

> [!TIP]
> If you use `nano`:
> - Paste text: `Ctrl+Shift+V` (or right-click -> Paste in many terminals).
> - Remove string: `Ctrl+K`
> - Delete everything: Keep pressing `Ctrl+K` until the file is empty
> - Exit with saving: `Ctrl+X`, then press `Y`, then `Enter`.
> - Exit without saving: `Ctrl+X`, then press `N`.


> [!IMPORTANT]
Set the following variables in the script before running it:
- `VPN_INTF="tun0"` - VPN interface name.
- `LAN_INTF="eth0"` - LAN interface name (usually `eth0`).
- `LAN_IP="192.168.x.x"` - Gateway IP address on your LAN.
- `LAN_SUBNET="192.168.x.0/24"` - LAN subnet.
- `VPN_IP="xxx.xxx.xxx.xxx"` - Public IP address of your VPN server (outside your local network).
- `MSS="<VPN MTU - 80>"` - MSS value derived from VPN MTU with a safety margin.
- `TTL="64"` - Fixed TTL value.

File content (`sudo nano /opt/vpn-rules.sh`):

```bash
#!/bin/bash
export PATH=/usr/sbin:/usr/bin:/sbin:/bin

# fail on errors
set -e

VPN_INTF="tun0" # Set your vpn interface name
LAN_INTF="eth0" # Set your lan interface name (usually eth0)
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

### Verification of the script
Run the script:
```bash
sudo chmod +x /opt/vpn-rules.sh && \
sudo /opt/vpn-rules.sh
```

Use the test device from Step 3 for this check.

Then make sure the device's traffic goes through the VPN.
This means the routing part is configured correctly.

#### Troubleshooting
If the device does not see VPN traffic:
- discuss the issue with AI, 
- fix the rules in `/opt/vpn-rules.sh`,
- reboot the gateway with `sudo reboot`,
- and run the script again with `sudo bash /opt/vpn-rules.sh`.
 
Repeat this process until the routing works correctly.

## Step 6. Create a systemd Service for Gateway Rules

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

After saving the file, enable and start the service:

```bash
systemctl enable vpn-gateway.service
systemctl start vpn-gateway.service
```

Check the service status:
```bash
systemctl status vpn-gateway.service
```

If the status is OK (active/success), reboot the gateway to ensure the service starts correctly on startup:

```bash
sudo reboot
```

After the reboot, use your test device from Step 3 to verify that the VPN rules are active. Check that traffic is being routed through the VPN before proceeding.

## Step 7. Configure DNS via systemd-resolved

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

You can check your DNS settings with this command:
```bash
resolvectl status
```

### Verification of DNS settings
Reconfigure the same test device from Step 3 with DNS already set to `<YOUR_GATEWAY_IP>` for this step.

Then make sure the device can resolve domain names.  
This means the DNS part is configured correctly.

## Step 8. Final Verification
Once the configuration is complete, verify your setup with these three steps:

1. **Routing Check:** On your test device, check your public IP (e.g., via `ifconfig.me`). It should match the VPN server's location, not your ISP.
2. **Leak Check:** Run a [Browser Leak Test](https://browserleaks.com/). It should confirm that your DNS, IPv6, and WebRTC traffic are all routed through the VPN tunnel, and that your home ISP's identity is completely masked.
3. **Kill Switch Check:** Stop your VPN client on the gateway. For example, for OpenVPN, run `sudo systemctl stop openvpn@client`. Your test device should immediately lose internet access. 
   *Note: If it stays online, re-check your `iptables` rules in `/opt/vpn-rules.sh`.*



## 🟢 Congratulations! 🟢
After that, traffic for the selected devices will go through your Linux VPN gateway.

## Appendix: How to set up a VPN cascade

> [!NOTE]
> This section is not required for a standard VPN Gateway setup. It is intended for regions where persistent network interference necessitates multi-hop VPS chaining to maintain reliable and open internet access.

In reality, it is quite difficult to provide a "one-size-fits-all" instruction for this domain. The configuration for an OpenVPN-to-OpenVPN cascade differs significantly from a WireGuard-to-OpenVPN setup.  
However, we can provide a general framework:

Cascading scenario: `VPN GATEWAY -> VPS1 -> VPS2`

> [!IMPORTANT]  
> **Prerequisites**: This rules assume that the basic VPN tunnels (`VPN GATEWAY` to `VPS1` and `VPS1` to `VPS2`) are already established and verified as functional point-to-point connections.

### VPN GATEWAY (Entry Node)
This part has already been configured in the previous step.

### VPS1 (Intermediate Node)
***Gateway** for `tun1` and **client** for `tun2`*

> [!CAUTION]  
> Do not forget to account for the rules already set by your OpenVPN/WireGuard installation scripts.  
> They may also require adjustments.

* main interface - `eth0`
- first vpn interface - `tun1`
    - subnet `10.255.255.0/24`
    - gateway `10.255.255.1`
* second vpn interface - `tun2`
    - subnet `10.0.0.0/24`
    - gateway `10.0.0.1`

```bash
(
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# Routing logic: set up a custom routing table for traffic from the client subnet
sudo ip rule add from 10.255.255.0/24 table 100 priority 100

# Routing logic: define the default gateway for the custom routing table
# All traffic matching the rule above will be routed through the secondary tunnel (tun2)
sudo ip route add default via 10.0.0.1 dev tun2 table 100

# Masquerade outbound traffic from the client subnet towards the secondary tunnel
sudo iptables -t nat -A POSTROUTING -s 10.255.255.0/24 -o tun2 -j MASQUERADE

# Masquerade traffic exiting through the primary tunnel
sudo iptables -t nat -A POSTROUTING -o tun1 -j MASQUERADE

# Allow traffic forwarding from the primary tunnel to the secondary tunnel
sudo iptables -A FORWARD -i tun1 -o tun2 -j ACCEPT

# Allow return traffic from the secondary tunnel back to the primary tunnel
sudo iptables -A FORWARD -i tun2 -o tun1 -m state --state RELATED,ESTABLISHED -j ACCEPT
)
```

### VPS2 (Exit Node)
***Gateway** for `tun2`*

* main interface - `eth0`
- vpn interface - `tun1`
    - subnet `10.0.0.0/24`
    - gateway `10.0.0.1`


```bash
(
# Enable forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# Enable NAT for outbound internet traffic
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

# Allow traffic forwarding from the tunnel to the external interface
sudo iptables -A FORWARD -i tun1 -o eth0 -j ACCEPT

# Allow return traffic from the internet back into the tunnel.
# Only permit established or related connections to maintain security
sudo iptables -A FORWARD -i eth0 -o tun1 -m state --state RELATED,ESTABLISHED -j ACCEPT
)
```
