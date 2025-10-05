# VPN Gateway Router

Turn your Raspberry Pi into a privacy-focused WiFi router that encrypts all traffic through a VPN tunnel.

## 🎯 What This Does

Creates a WiFi access point where **all connected devices automatically route through an encrypted VPN**, protecting your traffic from your ISP and providing privacy on public networks.

```
Your Phone/Laptop → WiFi AP → VPN Encryption → Internet
                    (Pi Router)
```

**Perfect for:**
- 🏠 Home privacy from ISP tracking
- ☕ Secure public WiFi usage
- 🌍 Bypassing geo-restrictions for all devices
- 🔒 Encrypting IoT device traffic
- 🚐 Travel router with VPN

## ⚡ Quick Start

### Requirements

**Hardware:**
- Raspberry Pi (Zero 2 W, 3, 4, or similar)
- USB WiFi adapter (for the access point)
- MicroSD card (8GB+)
- Power supply

**Software:**
- Armbian or Raspberry Pi OS
- VPN subscription (Mullvad, ProtonVPN, etc.) with WireGuard or OpenVPN support

**Network:**
- The Pi must have internet access (WiFi or Ethernet)
- USB WiFi adapter must support AP mode

### Installation

1. **Download the scripts:**
   ```bash
   # Clone or download this repository
   git clone <repository-url>
   cd vpn-gateway-router
   
   # Or copy the scripts manually
   chmod +x setup
   chmod +x manage
   ```

2. **Run the setup:**
   ```bash
   sudo ./setup
   ```

3. **Follow the prompts:**
   - Select your network interfaces
   - Choose WiFi name (SSID) and password
   - Select VPN type (WireGuard recommended)
   - Enter your VPN credentials
   - Confirm installation

4. **Connect and enjoy!**
   - Find your new WiFi network on your devices
   - Connect with the password you set
   - All traffic now goes through the VPN automatically

## 📋 Usage

### Check Status
```bash
sudo ./manage status
```
Shows service status, connected clients, VPN connection, and system resources.

### View Connected Clients
```bash
sudo ./manage clients
```

### Test VPN Connection
```bash
sudo ./manage vpn-test
```

### View Logs
```bash
# Follow all logs in real-time
sudo ./manage follow

# View specific service logs
sudo ./manage logs hostapd
sudo ./manage logs dnsmasq
```

### Restart Services
```bash
sudo ./manage restart
```

### Stop/Start Services
```bash
sudo ./manage stop
sudo ./manage start
```

### Backup Configuration
```bash
sudo ./manage backup
```
Creates a timestamped backup in `/root/vpn-gateway-backup-*/`

### Show Help
```bash
sudo ./manage help
```

## 🔧 Configuration

### Change WiFi Password
```bash
sudo nano /etc/hostapd/hostapd.conf
# Edit: wpa_passphrase=YourNewPassword
sudo systemctl restart hostapd
```

### Change WiFi Name (SSID)
```bash
sudo nano /etc/hostapd/hostapd.conf
# Edit: ssid=YourNetworkName
sudo systemctl restart hostapd
```

### Change VPN Server
```bash
# For WireGuard
sudo nano /etc/wireguard/wg0.conf
# Edit the Endpoint line
sudo systemctl restart wg-quick@wg0

# For OpenVPN
sudo nano /etc/openvpn/client.conf
# Edit the remote line
sudo systemctl restart openvpn@client
```

### Configuration Files
- **Access Point**: `/etc/hostapd/hostapd.conf`
- **DHCP/DNS**: `/etc/dnsmasq.conf`
- **WireGuard**: `/etc/wireguard/wg0.conf`
- **OpenVPN**: `/etc/openvpn/client.conf`

## 🐛 Troubleshooting

### WiFi Network Not Visible

**Check if hostapd is running:**
```bash
sudo systemctl status hostapd
```

**View hostapd logs:**
```bash
sudo ./manage logs hostapd
```

**Common fixes:**
```bash
# Restart hostapd
sudo systemctl restart hostapd

# Check interface is up
ip link show wlx98038eb6c140
```

### Can't Get IP Address (DHCP Issues)

**Check if dnsmasq is running:**
```bash
sudo systemctl status dnsmasq
```

**Common fixes:**
```bash
# Check for port conflicts
sudo ss -tulpn | grep :53

# Stop conflicting service
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved

# Restart dnsmasq
sudo systemctl restart dnsmasq
```

### No Internet Connection

**Check VPN status:**
```bash
sudo ./manage vpn-test
```

**For WireGuard:**
```bash
sudo wg show
# Should show recent handshake
```

**For OpenVPN:**
```bash
sudo systemctl status openvpn@client
```

**Check routing:**
```bash
sudo iptables -t nat -L -n -v
# Should show MASQUERADE rules
```

### USB WiFi Adapter Issues

**Check if adapter is detected:**
```bash
lsusb
ip link show
```

**Check for power issues:**
```bash
dmesg | grep -i usb | tail -20
```

**Solutions:**
```bash
# Disable power management
sudo iw dev wlx98038eb6c140 set power_save off

# Use powered USB hub (recommended for Pi Zero 2 W)
```

### VPN Not Connecting

**Check credentials:**
```bash
# WireGuard
sudo nano /etc/wireguard/wg0.conf
# Verify keys and endpoint

# OpenVPN
sudo nano /etc/openvpn/client.conf
# Verify server and credentials
```

**Test connectivity:**
```bash
# Can you reach the VPN server?
ping <vpn-server-address>

# Check firewall
sudo iptables -L -n | grep <vpn-port>
```

**View detailed logs:**
```bash
# WireGuard
sudo journalctl -u wg-quick@wg0 -n 100

# OpenVPN
sudo journalctl -u openvpn@client -n 100
```

### Complete Reset

If things are really broken:
```bash
# Stop all services
sudo systemctl stop hostapd dnsmasq wg-quick@wg0

# Backup your config first!
sudo ./manage backup

# Re-run setup
sudo ./setup
```

## ❓ FAQ

**Q: Which VPN should I use, WireGuard or OpenVPN?**  
A: WireGuard is recommended for Raspberry Pi - it's 4-5x faster and uses much less CPU.

**Q: Can I use my existing VPN subscription?**  
A: Yes! Most VPN providers (Mullvad, ProtonVPN, NordVPN, etc.) support WireGuard. You'll need to get your configuration from them.

**Q: How many devices can connect?**  
A: The DHCP server is configured for 190 devices (192.168.50.10-200), but realistically 10-20 devices depending on your Pi model.

**Q: Will this work on Raspberry Pi Zero W (1st gen)?**  
A: It will work but will be very slow. The Zero 2 W is minimum recommended.

**Q: Can I use Ethernet instead of WiFi for internet?**  
A: Yes! During setup, just select your Ethernet interface (eth0) as the WAN interface.

**Q: How do I know if my USB WiFi adapter supports AP mode?**  
A: Run: `iw list | grep -A 10 "Supported interface modes"` and look for "AP" in the list.

**Q: Can I run this on Ubuntu/Debian instead of Armbian?**  
A: Yes! The scripts work on any Debian-based system.

**Q: Is my traffic really encrypted?**  
A: Yes! All traffic from connected clients goes through the VPN tunnel. You can verify by checking your public IP: `curl ifconfig.me`

**Q: What if the VPN disconnects?**  
A: Currently, traffic will try to route directly. For a "kill switch" that blocks non-VPN traffic, you'll need to add additional firewall rules.

**Q: Can I access my local network devices?**  
A: By default, all traffic goes through VPN. To access local devices, you'd need to add routing exceptions.

## 🔐 Security Notes

- **Change default password**: Use a strong WiFi password
- **Keep updated**: Regularly update your system: `sudo apt update && sudo apt upgrade`
- **Trust your VPN**: Your VPN provider can see your traffic - choose a reputable provider
- **DNS leaks**: The setup uses VPN DNS servers to prevent leaks
- **No kill switch**: If VPN fails, traffic may leak. Monitor with `sudo ./manage status`

## 📊 Performance

Expected throughput on Raspberry Pi Zero 2 W:
- **WireGuard**: 80-100 Mbps
- **OpenVPN**: 15-25 Mbps

For better performance, use a Pi 4 or similar.

## 🛠️ Advanced

### Custom DHCP Range
Edit `/etc/dnsmasq.conf`:
```bash
# Change this line:
dhcp-range=192.168.50.10,192.168.50.200,255.255.255.0,24h
```

### Custom DNS Servers
Edit `/etc/dnsmasq.conf`:
```bash
# Change these lines:
server=1.1.1.1
server=1.0.0.1
```

### Enable 5GHz WiFi (if supported)
Edit `/etc/hostapd/hostapd.conf`:
```bash
hw_mode=a
channel=36
ieee80211ac=1
```

### View Detailed Status
A more detailed status script is available:
```bash
./vpn-status.sh
```

## 📝 Logs

All services log to systemd journal:
```bash
# View logs
sudo journalctl -u hostapd
sudo journalctl -u dnsmasq
sudo journalctl -u wg-quick@wg0
sudo journalctl -u openvpn@client

# Follow logs in real-time
sudo ./manage follow
```

## 🤝 Contributing

Issues and pull requests welcome!

## 📄 License

[Your chosen license]

## ⚠️ Disclaimer

This software is provided as-is. Use at your own risk. The authors are not responsible for any misuse or damage caused by this software. Always respect local laws and your VPN provider's terms of service.

## 🔗 Resources

- [WireGuard Documentation](https://www.wireguard.com/)
- [hostapd Documentation](https://w1.fi/hostapd/)
- [dnsmasq Documentation](http://www.thekelleys.org.uk/dnsmasq/doc.html)
- [Raspberry Pi Documentation](https://www.raspberrypi.org/documentation/)

## 📧 Support

For issues and questions, please check:
1. This README's troubleshooting section
2. The `CLAUDE.md` file for technical details
3. System logs: `sudo ./manage logs`
4. GitHub issues (if applicable)

---

**Built for privacy. Powered by open source.** 🔒