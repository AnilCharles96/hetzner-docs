# Install DHCP server 

```
sudo apt update
sudo apt install isc-dhcp-server
```

# Set the interface 
```
INTERFACESv4="eth0" # vmbr0, enp0s31f6 etc
```

# Configure DHCP server 
```
sudo nano /etc/dhcp/dhcpd.conf


subnet 10.0.0.0 netmask 255.255.255.0 {
    range 10.0.0.100 10.0.0.200;
    option routers 10.0.0.1;
    option domain-name-servers 1.1.1.1, 8.8.8.8;

    # Static IP reservation
    host <CUSTOM_DEVICE_NAME> {
        hardware ethernet <MAC_ADDRESS>;
        fixed-address 10.0.0.100;
    }
}
```

# Restart DHCP server 
```
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
```

# Request static IP from DHCP server
```
# Client
dhclient <INTERFACE> 

# eg: 
# dhclient ens18 
````

# Verify IP is assigned
```
# client
ip addr show | grep inet 

# server 
arp -a
```