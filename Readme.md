# VPN setup between Nexar - lscapes - ubuntu server

## Server choise
Considering the updated requirements—using OpenVPN for the site-to-site VPN between Nexar-lan and lscapes-lan, with one site routing Netflix streaming through the tunnel (potentially up to 25Mbps for 4K) and PLEX media streaming 4 days per week (around 10-20Mbps per session)—the server needs to handle sustained encryption/decryption and forwarding without bottlenecks. OpenVPN is CPU-intensive, and while Starlink provides decent bandwidth (up to 100-200Mbps), the VPS must manage the throughput efficiently.
The CX23 (2 vCPUs, 4GB RAM, 40GB SSD, €2.99/mo) could work for light variable loads, but benchmarks show low-end VPS like this often limit OpenVPN to 20-50Mbps sustained throughput due to older hardware and shared resources. With Netflix and PLEX, you risk CPU spikes, latency, or dropped performance during peaks, especially since OpenVPN relies on AES encryption (supported via AES-NI on Hetzner).
To balance low cost with reliable performance, I recommend the CX33 (€4.99/mo): 4 vCPUs, 8GB RAM, 80GB SSD. This provides more headroom for multi-threaded OpenVPN handling, ensuring smoother streaming without throttling. The 20TB traffic limit remains ample (average <10Mbps over a month, with bursts fine on Hetzner's network).
If testing shows issues (e.g., via iperf or speedtests through the VPN), step up to the Regular Performance CPX32 (€10.49/mo): 4 vCPUs (AMD, newer gen), 8GB RAM, 160GB SSD. It offers higher baseline CPU performance for variable but sustained loads like this.

CX33 (€4.99/mo): 4 vCPUs, 8GB RAM, 80GB SSD. Will be used for the VPN server.


## Server initial setup.
Add user peter
```
adduser peter
usermod -aG sudo peter
groups peter
su - peter
```

Add ssh keys to user peter for remote login
```
ssh-copy-id peter@vpn.nexar.be
```


Disable root login via ssh
```
pico /etc/ssh/sshd_config
PermitRootLogin no
```

Disable password authentication for SSH
```
pico /etc/ssh/sshd_config
PasswordAuthentication no
systemctl restart ssh
```

## Setup fail2ban with recidive on ubuntu server
```
apt update
apt upgrade
apt install fail2ban
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
pico /etc/fail2ban/jail.local
```
```
[DEFAULT]
bantime = 1d
maxretry = 3
findtime = 1h
banaction = iptables-multiport
backend = systemd

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = %(sshd_log)s
maxretry = 3
findtime = 1h
bantime = 1d
banaction = iptables-multiport
backend = systemd

[recidive]
enabled = true
port = ssh
filter = recidive
logpath = %(sshd_log)s
maxretry = 3
findtime = 1h
bantime = 1d
banaction = iptables-multiport
backend = systemd
```
```
systemctl restart fail2ban
```

## Setup OpenVPN on ubuntu server (vpn.nexar.be)

```
sudo apt update && sudo apt upgrade -y
sudo apt install openvpn easy-rsa -y
sudo mkdir /etc/openvpn/easy-rsa && sudo cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa/
cd /etc/openvpn/easy-rsa/
sudo nano vars
EASYRSA_REQ_COUNTRY="BE"
EASYRSA_REQ_CITY="Antwerp"
```

Generate certificates and keys
```
sudo ./easyrsa init-pki
sudo ./easyrsa build-ca #Ons Nora is echt een schatje
sudo ./easyrsa build-server-full server nopass
sudo ./easyrsa build-client-full nexar-lan nopass
sudo ./easyrsa build-client-full lscapes-lan nopass
sudo ./easyrsa gen-dh
sudo openvpn --genkey secret /etc/openvpn/ta.key
```

## Building the vpn route between nexar and the ubuntu server

To set up and test the OpenVPN tunnel from the Nexar-lan (behind Starlink, no public IP) to the Ubuntu server (vpn.nexar.be), with the Draytek router initiating the connection as the client, we'll configure a basic one-way client-server setup first. No site-to-site routing or lscapes-lan involvement yet—just establish the tunnel and verify connectivity. Work as peter with sudo on the server.

```
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn
sudo nano /etc/openvpn/server.conf
```

```
port 1194
proto udp
dev tun
ca /etc/openvpn/easy-rsa/pki/ca.crt
cert /etc/openvpn/easy-rsa/pki/issued/server.crt
key /etc/openvpn/easy-rsa/pki/private/server.key
dh /etc/openvpn/easy-rsa/pki/dh.pem
tls-auth /etc/openvpn/ta.key 0
topology subnet
server 10.8.0.0 255.255.255.0  # VPN subnet
ifconfig-pool-persist /var/log/openvpn/ipp.txt
keepalive 10 120
cipher AES-256-GCM
auth SHA256
user nobody
group nogroup
persist-key
persist-tun
status /var/log/openvpn/openvpn-status.log
verb 3
```

```
sudo sysctl -w net.ipv4.ip_forward=1
sudo pico /etc/sysctl.conf
```
Add
```
net.ipv4.ip_forward=1
```
```
sudo sysctl -p
```

```
sudo systemctl restart openvpn@server
sudo systemctl status openvpn@server
```

From the server, securely copy these files to your local machine (e.g., via scp as peter):
```
scp peter@vpn.nexar.be:~/tempcerts/ca.crt .
scp peter@vpn.nexar.be:~/tempcerts/nexar-lan.crt .
scp peter@vpn.nexar.be:~/tempcerts/nexar-lan.key .
scp peter@vpn.nexar.be:~/tempcerts/ta.key .
```

Create a client config file (nexar-client.ovpn) on your local machine with this content (adjust paths if needed):
```
client
dev tun
proto udp
remote vpn.nexar.be 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert nexar-lan.crt
key nexar-lan.key
remote-cert-tls server
tls-auth ta.key 1
cipher AES-256-GCM
auth SHA256
verb 3
```

STILL NEED TO UPLOAD THIS TO THE DRAITEK AND ENABLE THIS
Don't know how to do this at the moment.
