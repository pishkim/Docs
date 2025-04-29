### Mikrotik Remote Access
**1. Set Up a DigitalOcean Droplet (Ubuntu Server)**

<mark>Step 1</mark>

* Create a Droplet:

  - Go to [DigitalOcean](https://www.digitalocean.com/) and create a new Ubuntu droplet.
  
  * Choose a basic plan (e.g., $5/month).
  
  * Enable **IPv6** (optional but useful).
  
  * Add SSH keys for secure access.

_Connect to the Droplet:_

```bash
ssh root@your_droplet_ip
```

2. Install and Configure OpenVPN Server
We'll use OpenVPN to create a secure tunnel to the MikroTik.

Install OpenVPN & Easy-RSA
bash
sudo apt update && sudo apt upgrade -y
sudo apt install openvpn easy-rsa -y
Set Up PKI (Certificates)
bash
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
Edit vars file:

bash
nano vars
Update:

```ini
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="SanFrancisco"
export KEY_ORG="YourOrg"
export KEY_EMAIL="admin@example.com"
export KEY_OU="MyOrgUnit"
export KEY_NAME="server"
Generate certificates:
```

```bash
source vars
./clean-all
./build-ca          # (Press Enter for defaults)
./build-key-server server
./build-dh
./build-key client1 # For client device
Configure OpenVPN Server
bash
cd ~/openvpn-ca/keys
sudo cp server.crt server.key ca.crt dh2048.pem /etc/openvpn
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/
sudo gzip -d /etc/openvpn/server.conf.gz
sudo nano /etc/openvpn/server.conf
```

* Modify:

```ini
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh2048.pem
server 10.8.0.0 255.255.255.0
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
keepalive 10 120
comp-lzo
user nobody
group nogroup
persist-key
persist-tun
status openvpn-status.log
verb 3
Enable IP Forwarding
bash
sudo nano /etc/sysctl.conf
```

### Uncomment:

```ini
net.ipv4.ip_forward=1
```
* Apply:

```bash
sudo sysctl -p
```

### Start OpenVPN
```bash
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
```

3. Configure Port Forwarding on DigitalOcean
Go to DigitalOcean Firewall:

Add a UDP rule for port 1194 (OpenVPN).

(Optional) Add TCP forwarding if needed (e.g., port 8291 for MikroTik WinBox).

4. Set Up OpenVPN Client on MikroTik
Export client config from the OpenVPN server:

bash
cd ~/openvpn-ca/keys
nano client1.ovpn
Paste:

ini
client
dev tun
proto udp
remote your_droplet_ip 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client1.crt
key client1.key
remote-cert-tls server
comp-lzo
verb 3
Transfer client1.ovpn, ca.crt, client1.crt, client1.key to MikroTik.

Configure OpenVPN Client on MikroTik:

bash
/interface ovpn-client add name=ovpn-out1 connect-to=your_droplet_ip port=1194 mode=ip user=client1 password="" certificate=client1.crt_0 key=client1.key_0 cipher=aes256 auth=sha1 add-default-route=yes
5. Access MikroTik Remotely via OpenVPN
Once the VPN is connected:

MikroTik will get an IP like 10.8.0.6.

From your local machine (connected to OpenVPN), access MikroTik via:

bash
ssh admin@10.8.0.6
or WinBox via 10.8.0.6.

6. (Optional) Port Forwarding to MikroTik
If you need to expose MikroTik services (e.g., HTTP, WinBox) through the droplet:

bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 8291 -j DNAT --to-destination 10.8.0.6:8291
sudo iptables -A FORWARD -p tcp -d 10.8.0.6 --dport 8291 -j ACCEPT
Now, accessing your_droplet_ip:8291 will forward to MikroTik.

Final Notes
Security: Use firewall rules (ufw/iptables) to restrict access.

Persistent iptables: Save rules:

bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
Dynamic DNS: If the MikroTik has a changing IP, use a DDNS script.

This setup allows secure remote access to your MikroTik via a DigitalOcean VPN gateway. 🚀

###### Multiple mikrotiks
Step-by-Step Guide
1. Assign Static IPs to Each MikroTik in OpenVPN
Modify /etc/openvpn/server.conf to assign fixed IPs:

ini
client-config-dir /etc/openvpn/ccd
Create a config file for each MikroTik:

bash
sudo mkdir /etc/openvpn/ccd
sudo nano /etc/openvpn/ccd/client1
Add:

ini
ifconfig-push 10.8.0.6 255.255.255.0
Repeat for client2, client3, etc., assigning different IPs (e.g., 10.8.0.7, 10.8.0.8).

2. Set Up Port Forwarding Rules
Forward different ports on the DigitalOcean droplet to each MikroTik:

MikroTik	VPN IP	Service Port	Droplet Port
MikroTik 1	10.8.0.6	WinBox (8291)	10001
MikroTik 2	10.8.0.7	WinBox (8291)	10002
MikroTik 3	10.8.0.8	SSH (22)	10003
Run these commands:

bash
# MikroTik 1 (WinBox)
sudo iptables -t nat -A PREROUTING -p tcp --dport 10001 -j DNAT --to-destination 10.8.0.6:8291
sudo iptables -A FORWARD -p tcp -d 10.8.0.6 --dport 8291 -j ACCEPT

# MikroTik 2 (WinBox)
sudo iptables -t nat -A PREROUTING -p tcp --dport 10002 -j DNAT --to-destination 10.8.0.7:8291
sudo iptables -A FORWARD -p tcp -d 10.8.0.7 --dport 8291 -j ACCEPT

# MikroTik 3 (SSH)
sudo iptables -t nat -A PREROUTING -p tcp --dport 10003 -j DNAT --to-destination 10.8.0.8:22
sudo iptables -A FORWARD -p tcp -d 10.8.0.8 --dport 22 -j ACCEPT
3. Allow Ports in DigitalOcean Firewall
Go to DigitalOcean Networking > Firewall.

Add TCP rules for 10001, 10002, 10003.

4. Access Each MikroTik Remotely
MikroTik 1 (WinBox):
Connect to your_droplet_ip:10001 → Forwards to 10.8.0.6:8291.

MikroTik 2 (WinBox):
Connect to your_droplet_ip:10002 → Forwards to 10.8.0.7:8291.

MikroTik 3 (SSH):

bash
ssh admin@your_droplet_ip -p 10003
Optional: Persistent IPTables Rules
To avoid losing rules after reboot:

bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
Alternative: Use SSH Tunneling (No Port Forwarding)
If you prefer SSH tunneling instead of exposing ports:

bash
ssh -L 10001:10.8.0.6:8291 root@your_droplet_ip
Now, access WinBox via localhost:10001.


Summary
✅ Each MikroTik gets a unique VPN IP.

✅ Different ports on the droplet map to different MikroTiks.

✅ Secure access without exposing MikroTiks directly to the internet.


This method scales easily—just add more iptables rules for additional devices! 🚀
