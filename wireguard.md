# Introduction

In order for a server to join as a worker node in kubeadm cluster they both have to be on the same network. 

To make them part of the same network VPN needs to be configured. This docs will show how to put them on the same network using wireguard. 

# Install wireguard
```
sudo apt update
sudo apt install wireguard
```

# Generate public and private key in both client and server
```
wg genkey | tee privatekey | wg pubkey > publickey
```

# Create server config 
```
# server 
# nano /etc/wireguard/wg0.conf
[Interface]
Address = 10.2.2.1/24
SaveConfig = true
PostUp = iptables -A FORWARD -i wg0 -o vmbr0 -j ACCEPT; iptables -A FORWARD -i vmbr0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT; iptables -t nat>
PostDown = iptables -D FORWARD -i wg0 -o vmbr0 -j ACCEPT; iptables -D FORWARD -i vmbr0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT; iptables -t n>
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>
```

# Run wireguard in server to create tunnel
```
# if already up 
# wg-quick down wg0

wg-quick up wg0

# ip addr show # check wg0 

# Get public key
wg  # use this in the client
```


# Create client config
```
[Interface]
PrivateKey = kIa1Y0V1lf9Q8ncsiiM1gaSnGgMwfEl4FHNdRCnXlks= 
Address = 10.2.2.2/32           
DNS = 1.1.1.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY> # we get this from running the above command wg in server
Endpoint =  <SERVER_PUBLIC_IP>:51820  
AllowedIPs = 10.2.2.0/24,10.0.0.100/24          
PersistentKeepalive = 25
````


# Run wireguard in client to create tunnel
```
# if already up 
# wg-quick down wg0

wg-quick up wg0

# ip addr show # check wg0 

# Get public key
wg  # use this in the server
```


# Add peer section in server conf
```
# server 
# nano /etc/wireguard/wg0.conf
[Interface]
Address = 10.2.2.1/24
SaveConfig = true
PostUp = iptables -A FORWARD -i wg0 -o vmbr0 -j ACCEPT; iptables -A FORWARD -i vmbr0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT; iptables -t nat>
PostDown = iptables -D FORWARD -i wg0 -o vmbr0 -j ACCEPT; iptables -D FORWARD -i vmbr0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT; iptables -t n>
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>


[Peer]
PublicKey = <CLIENT_PUBLIC_KEY> 
AllowedIPs = 10.2.2.2/32
Endpoint = 49.12.120.60:57551
```


# Verify
```
# client and server
wg show

# eg: output 
interface: wg0
  public key: <CLIENT_PUBLIC_KEY>
  private key: (hidden)
  listening port: 57551

peer: <SERVER_PUBLIC_KEY>
  endpoint: <SERVER_PUBLIC_IP>:51820
  allowed ips: 10.2.2.0/24, 10.0.0.100/32
  latest handshake: 16 seconds ago
  transfer: 97.18 KiB received, 216.13 KiB sent
  persistent keepalive: every 25 seconds
```