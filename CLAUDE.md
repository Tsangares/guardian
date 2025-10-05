# VPN Gateway Router - Developer Documentation

## Project Overview

This project converts a Raspberry Pi (or similar Armbian device) into a VPN gateway router. It creates a WiFi Access Point that routes all client traffic through a VPN tunnel, providing encrypted internet access through the ISP connection.

## Architecture

```
Client Device
    ↓ (WiFi)
Access Point Interface (wlx98038eb6c140)
    ↓ (IP: 192.168.50.1/24)
DHCP/DNS (dnsmasq)
    ↓
VPN Tunnel (wg0 or tun0)
    ↓ (NAT/MASQUERADE)
WAN Interface (wlan0)
    ↓
Internet via ISP
```

## Traffic Flow

1. **Client → AP**: Client connects to WiFi AP (hostapd on wlx98038eb6c140)
2. **DHCP**: Client gets IP from dnsmasq (192.168.50.10-200)
3. **Routing**: Traffic is forwarded to VPN interface (wg0/tun0)
4. **NAT**: iptables MASQUERADE translates source IPs
5. **VPN**: Traffic exits through encrypted VPN tunnel
6. **WAN**: Encrypted traffic goes to internet via wlan0

## File Structure

```
.
├── setup                           # Main setup script
├── manage                          # Management/troubleshooting script
├── vpn-status.sh                   # Quick status display (optional)
├── /etc/hostapd/hostapd.conf      # AP configuration
├── /etc/dnsmasq.conf              # DHCP/DNS configuration
├── /etc/wireguard/wg0.conf        # WireGuard config (if using WireGuard)
├── /etc/openvpn/client.conf       # OpenVPN config (if using OpenVPN)
├── /etc/sysctl.d/99-vpn-router.conf    # IP forwarding config
├── /etc/systemd/network/10-*.network   # systemd-networkd configs
└── /usr/local/bin/vpn-gateway-init.sh  # Init script for AP interface
```

## Key Components

### 1. Access Point (hostapd)
- **Service**: `hostapd.service`
- **Config**: `/etc/hostapd/hostapd.conf`
- **Interface**: Uses `wlx98038eb6c140` (USB WiFi adapter)
- **Mode**: AP mode (802.11g/n)
- **Security**: WPA2-PSK

**Critical Requirements:**
- WiFi adapter MUST support AP mode
- Interface must be UP before hostapd starts
- Must not be managed by wpa_supplicant

### 2. DHCP Server (dnsmasq)
- **Service**: `dnsmasq.service`
- **Config**: `/etc/dnsmasq.conf`
- **Range**: 192.168.50.10 - 192.168.50.200
- **Gateway**: 192.168.50.1 (AP interface)
- **DNS**: 1.1.1.1, 1.0.0.1 (Cloudflare)

**Common Issues:**
- Port 53 conflict with systemd-resolved
- Interface not ready when dnsmasq starts
- No IP assigned to AP interface

### 3. VPN Tunnel

#### WireGuard (Recommended)
- **Service**: `wg-quick@wg0.service`
- **Config**: `/etc/wireguard/wg0.conf`
- **Interface**: `wg0`
- **Advantages**: Fast, low CPU, kernel-level

#### OpenVPN (Alternative)
- **Service**: `openvpn@client.service`
- **Config**: `/etc/openvpn/client.conf`
- **Interface**: `tun0`
- **Scripts**: `/etc/openvpn/up.sh`, `/etc/openvpn/down.sh`

### 4. Routing & NAT

**IP Forwarding:**
```bash
net.ipv4.ip_forward=1
```

**NAT Rules:**
```bash
# For WireGuard
iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
iptables -A FORWARD -i wlx98038eb6c140 -o wg0 -j ACCEPT
iptables -A FORWARD -i wg0 -o wlx98038eb6c140 -m state --state RELATED,ESTABLISHED -j ACCEPT

# For OpenVPN
iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
iptables -A FORWARD -i wlx98038eb6c140 -o tun0 -j ACCEPT
iptables -A FORWARD -i tun0 -o wlx98038eb6c140 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

## Current Known Issues

### Issue 1: dnsmasq Not Starting
**Symptoms:**
- `systemctl status dnsmasq` shows stopped/failed
- No DHCP leases being issued
- Clients can't get IP addresses

**Common Causes:**
1. **Port 53 conflict**: systemd-resolved is using port 53
2. **No IP on interface**: AP interface doesn't have 192.168.50.1
3. **Interface not ready**: dnsmasq starts before interface is up

**Debug Commands:**
```bash
# Check port 53
sudo ss -tulpn | grep :53
sudo lsof -i :53

# Check dnsmasq logs
sudo journalctl -u dnsmasq -n 50

# Test config
sudo dnsmasq --test

# Check interface IP
ip addr show wlx98038eb6c140
```

**Solutions:**
```bash
# Fix 1: Stop systemd-resolved
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved

# Fix 2: Assign IP to interface
sudo ip addr add 192.168.50.1/24 dev wlx98038eb6c140

# Fix 3: Add dependency in systemd
sudo systemctl edit dnsmasq
# Add:
[Unit]
After=vpn-gateway-init.service
Requires=vpn-gateway-init.service
```

### Issue 2: AP Interface Missing IP
**Symptoms:**
- Interface is UP but has no IPv4 address
- Only shows link-local IPv6 (fe80::...)

**Causes:**
- Network manager (NetworkManager/dhcpcd) is managing the interface
- systemd-networkd config missing or incorrect
- vpn-gateway-init.service not running

**Debug Commands:**
```bash
# Check what's managing the interface
systemctl status NetworkManager
systemctl status dhcpcd
systemctl status systemd-networkd

# Check for conflicts
cat /etc/dhcpcd.conf | grep wlx98038eb6c140

# Check systemd-networkd config
ls -la /etc/systemd/network/
cat /etc/systemd/network/10-wlx98038eb6c140.network
```

**Solutions:**
```bash
# Option A: Use systemd-networkd
sudo systemctl enable systemd-networkd
cat > /etc/systemd/network/10-wlx98038eb6c140.network <<EOF
[Match]
Name=wlx98038eb6c140

[Network]
Address=192.168.50.1/24
IPForward=yes
EOF
sudo systemctl restart systemd-networkd

# Option B: Deny interface in dhcpcd
echo "denyinterfaces wlx98038eb6c140" >> /etc/dhcpcd.conf
sudo systemctl restart dhcpcd

# Option C: Ensure init service runs
sudo systemctl enable --now vpn-gateway-init
```

### Issue 3: USB WiFi Adapter Power Issues
**Symptoms:**
- "unable to enumerate USB device" in dmesg
- Adapter disconnects randomly
- Interface disappears

**Causes:**
- Insufficient USB power (Pi Zero 2 W has ~500mA total)
- Power management putting adapter to sleep

**Debug Commands:**
```bash
# Check for USB errors
dmesg | grep -i usb | tail -20

# Check power issues
vcgencmd get_throttled

# Check power management
iw dev wlx98038eb6c140 get power_save
```

**Solutions:**
```bash
# Disable power management
sudo iw dev wlx98038eb6c140 set power_save off

# Add to /boot/firmware/config.txt or /boot/config.txt
max_usb_current=1

# Use powered USB hub (best solution for Pi Zero 2 W)
```

### Issue 4: hostapd Won't Start
**Symptoms:**
- hostapd fails to start
- "Could not configure driver mode" error
- "Device or resource busy"

**Causes:**
- wpa_supplicant is controlling the interface
- Interface doesn't support AP mode
- Another process using the interface

**Debug Commands:**
```bash
# Check hostapd logs
sudo journalctl -u hostapd -n 50

# Check if wpa_supplicant is running
ps aux | grep wpa_supplicant
systemctl status wpa_supplicant

# Check interface capabilities
iw list | grep -A 10 "Supported interface modes"
iw phy | grep -A 20 "interface modes"

# Check what's using the interface
sudo lsof | grep wlx98038eb6c140
```

**Solutions:**
```bash
# Stop wpa_supplicant
sudo systemctl stop wpa_supplicant
sudo systemctl disable wpa_supplicant

# Kill any processes using interface
sudo killall wpa_supplicant

# Verify AP mode support
iw phy$(iw dev wlx98038eb6c140 info | grep wiphy | awk '{print $2}') info | grep "* AP"
```

### Issue 5: VPN Not Connecting
**Symptoms:**
- VPN service running but no connectivity
- No handshake (WireGuard)
- No tunnel interface created

**Debug Commands:**
```bash
# WireGuard
sudo wg show
sudo journalctl -u wg-quick@wg0 -n 50
ping -c 3 -I wg0 1.1.1.1

# OpenVPN
sudo journalctl -u openvpn@client -n 50
ip addr show tun0
ping -c 3 -I tun0 1.1.1.1
```

**Common Issues:**
```bash
# Firewall blocking VPN
sudo ufw status
sudo iptables -L -n | grep 51820  # WireGuard port

# DNS issues
cat /etc/resolv.conf

# Routing issues
ip route show
ip route get 1.1.1.1
```

## Service Start Order

**Critical startup sequence:**
1. `vpn-gateway-init.service` - Sets IP on AP interface
2. VPN service (`wg-quick@wg0` or `openvpn@client`)
3. `dnsmasq.service` - DHCP/DNS server
4. `hostapd.service` - Access point

**Dependencies configured in systemd:**
- hostapd requires network-online.target
- dnsmasq requires vpn-gateway-init
- All services enable IP forwarding before starting

## Testing & Verification

### 1. Check All Services
```bash
sudo ./manage status
```

### 2. Test AP Functionality
```bash
# Check AP interface
ip addr show wlx98038eb6c140

# Check hostapd
sudo systemctl status hostapd
sudo iw dev wlx98038eb6c140 info

# See connected stations
sudo iw dev wlx98038eb6c140 station dump
```

### 3. Test DHCP
```bash
# Check leases
cat /var/lib/misc/dnsmasq.leases

# Monitor DHCP in real-time
sudo journalctl -u dnsmasq -f
```

### 4. Test VPN
```bash
# WireGuard
sudo wg show
ping -c 3 -I wg0 1.1.1.1
curl --interface wg0 ifconfig.me

# OpenVPN
ip addr show tun0
ping -c 3 -I tun0 1.1.1.1
```

### 5. Test Routing
```bash
# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward

# Check NAT rules
sudo iptables -t nat -L -n -v

# Check forwarding rules
sudo iptables -L FORWARD -n -v
```

### 6. End-to-End Test
1. Connect client to WiFi AP
2. Verify client gets IP (192.168.50.x)
3. Check client can reach internet
4. Verify traffic goes through VPN:
   ```bash
   # On client
   curl ifconfig.me  # Should show VPN server IP, not ISP IP
   ```

## Debugging Workflow

### Step 1: Check Service Status
```bash
sudo ./manage status
```

### Step 2: Check Logs
```bash
# Follow all logs
sudo ./manage follow

# Individual service logs
sudo ./manage logs hostapd
sudo ./manage logs dnsmasq
```

### Step 3: Check Network
```bash
# Interfaces
ip -brief addr

# Routes
ip route

# NAT
sudo iptables -t nat -L -n -v
```

### Step 4: Test Connectivity
```bash
# From Pi
ping -c 3 1.1.1.1
ping -c 3 google.com

# Through VPN
ping -c 3 -I wg0 1.1.1.1  # WireGuard
ping -c 3 -I tun0 1.1.1.1  # OpenVPN
```

## Configuration Variables

### Network Configuration
- **AP Interface**: `wlx98038eb6c140` (USB WiFi adapter)
- **WAN Interface**: `wlan0` (built-in WiFi)
- **AP IP**: `192.168.50.1/24`
- **DHCP Range**: `192.168.50.10` - `192.168.50.200`
- **Lease Time**: 24 hours

### VPN Configuration
- **WireGuard Port**: 51820 (default)
- **OpenVPN Port**: 1194 (default, configurable)
- **VPN Subnet**: Provider-specific (e.g., 10.66.66.0/24)

## Performance Considerations

### Raspberry Pi Zero 2 W Limitations
- **CPU**: 4-core ARM Cortex-A53 @ 1GHz
- **RAM**: 512MB
- **USB**: ~500mA total power budget
- **WiFi**: 2.4GHz only (built-in)

### Expected Throughput
- **WireGuard**: 80-100 Mbps
- **OpenVPN**: 15-25 Mbps
- **Real-world**: Depends on WiFi signal, USB adapter quality

### Optimization Tips
```bash
# Disable WiFi power saving
sudo iw dev wlan0 set power_save off

# Optimize WireGuard MTU
# Add to wg0.conf: MTU = 1420

# Monitor performance
sudo ./manage monitor  # If iftop installed
```

## Security Notes

1. **Change default AP password** - Don't use weak passwords
2. **Use WPA2** only (not WPA or WEP)
3. **VPN provider matters** - Choose privacy-focused providers
4. **DNS leaks** - Verify DNS goes through VPN
5. **Kill switch** - If VPN disconnects, traffic should not leak

## Useful Commands Reference

```bash
# Service management
sudo systemctl restart hostapd
sudo systemctl restart dnsmasq
sudo systemctl restart wg-quick@wg0

# Network debugging
ip addr show
ip route show
ip link show
sudo iw dev wlx98038eb6c140 info

# Firewall
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v

# Logs
sudo journalctl -u hostapd -f
sudo journalctl -u dnsmasq -f
sudo journalctl -u wg-quick@wg0 -f

# VPN
sudo wg show
sudo wg show wg0 endpoints
sudo wg show wg0 transfer

# DHCP leases
cat /var/lib/misc/dnsmasq.leases
sudo journalctl -u dnsmasq | grep DHCP
```

## Common Tasks

### Add/Change AP Password
```bash
sudo nano /etc/hostapd/hostapd.conf
# Change: wpa_passphrase=YourNewPassword
sudo systemctl restart hostapd
```

### Change AP Subnet
```bash
# 1. Update hostapd config (if IP is there)
sudo nano /etc/hostapd/hostapd.conf

# 2. Update interface IP
sudo nano /etc/systemd/network/10-wlx98038eb6c140.network
# Change: Address=192.168.X.1/24

# 3. Update dnsmasq
sudo nano /etc/dnsmasq.conf
# Change: dhcp-range=192.168.X.10,192.168.X.200,...

# 4. Update iptables rules in WireGuard config
sudo nano /etc/wireguard/wg0.conf
# Change IP route add line

# 5. Restart everything
sudo systemctl restart vpn-gateway-init
sudo systemctl restart dnsmasq
sudo systemctl restart hostapd
```

### Switch VPN Servers
```bash
# WireGuard
sudo nano /etc/wireguard/wg0.conf
# Update Endpoint line
sudo systemctl restart wg-quick@wg0

# OpenVPN
sudo nano /etc/openvpn/client.conf
# Update remote line
sudo systemctl restart openvpn@client
```

### View Connected Clients
```bash
sudo ./manage clients
# Or manually:
cat /var/lib/misc/dnsmasq.leases
sudo iw dev wlx98038eb6c140 station dump
```

## Recovery Procedures

### Full Reset
```bash
# Stop all services
sudo systemctl stop hostapd dnsmasq wg-quick@wg0

# Clear iptables
sudo iptables -F
sudo iptables -t nat -F

# Restart from scratch
sudo ./setup
```

### Backup Configuration
```bash
sudo ./manage backup
# Creates backup in /root/vpn-gateway-backup-YYYYMMDD-HHMMSS/
```

### Emergency Access
If locked out:
1. Connect via Ethernet or built-in WiFi
2. SSH into device
3. Check logs: `sudo journalctl -xe`
4. Restart services: `sudo ./manage restart`