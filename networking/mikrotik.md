# Mikrotik

Learn how to set up your MikroTik router covering initial configuration, secure network setup. Master your MikroTik router with this all-in-one guide! From initial setup to securing your network and turbocharging performance.

<details>

<summary>Prerequisites</summary>

* Console access to the MikroTik
* No Configuration on the Mikrotik

</details>

{% hint style="info" %}
In this configuration we use **ether1** as the link to the internet.
{% endhint %}

## Basic Configuration

### LAN

#### Bridge Interface

Create a bridge interface to unify all Ethernet ports into a single network.

```shell
/interface bridge
add name=bridge1 protocol-mode=none
/interface bridge port
add bridge=bridge1 interface=ether2
add bridge=bridge1 interface=ether3
add bridge=bridge1 interface=ether4
add bridge=bridge1 interface=ether5
add bridge=bridge1 interface=ether6
add bridge=bridge1 interface=ether7
add bridge=bridge1 interface=ether8
/interface list
add name=LAN
/interface list member
add interface=bridge1 list=LAN
/ip address
add address=192.168.1.1/24 interface=bridge1 network=192.168.1.0
```

#### DHCP

Set up a DHCP server to assign IP addresses to connected devices.

```shell
/ip dhcp-server network
add address=192.168.1.0/24 dns-server=192.168.1.1 gateway=192.168.1.1 netmask=24
/ip pool
add name=dhcp_pool0 ranges=192.168.1.100-192.168.1.254
/ip dhcp-server
add address-pool=dhcp_pool0 interface=bridge1 lease-time=1d name=dhcp1
```

#### DNS

Enable DNS queries and forward them to the upstream DNS server.

```shell
/ip dns
set allow-remote-requests=yes
```

### Firewall

This firewall will block all traffic by default, allowing only incoming data that is established, related, or untracked, while permitting outgoing traffic exclusively from the LAN side.

```shell
/ip firewall filter
add action=accept chain=forward comment="Allow established,related,untracked" connection-state=established,related,untracked
add action=drop chain=forward comment="drop invalid traffic" connection-state=invalid
add action=accept chain=forward comment="port forwarding" connection-nat-state=dstnat
add action=accept chain=forward comment="internet traffic" in-interface-list=LAN out-interface-list=WAN
add action=drop chain=forward comment="drop all else"
add action=accept chain=input comment="Allow established,related,untracked" connection-state=established,related,untracked
add action=drop chain=input comment="Drop invalid" connection-state=invalid
add action=accept chain=input comment="Allow traffic from LAN interface list to the router" in-interface-list=LAN
add action=drop chain=input comment="Drop all else"
/ip firewall service-port
set ftp disabled=yes
set tftp disabled=yes
set h323 disabled=yes
set sip disabled=yes
```

### WAN

Set up the necessary WAN interface to obtain an IP address from your ISP.

{% tabs %}
{% tab title="DHCP with VLAN" %}
```bash
/interface vlan add interface=ether1 name=internet vlan-id=<Enter ISP VLAN ID> 
/ip dhcp-client add interface=internet disabled=no use-peer-ntp=no add-default-route=yes 
/interface list add name=WAN 
/interface list member add interface=internet list=WAN
```

> * Change `<Enter ISP VLAN ID>` to the needed VLAN ID for your ISP
{% endtab %}

{% tab title="DHCP" %}
```bash
/interface ethernet set ether1 name=internet 
/ip dhcp-client add interface=internet add-default-route=yes disabled=no use-peer-ntp=no 
/interface list add name=WAN 
/interface list member add interface=internet list=WAN`
```
{% endtab %}

{% tab title="PPPoE with VLAN" %}
```bash
/interface add interface=ether1 name=vlan_int vlan-id=<Enter ISP VLAN ID> 
/interface pppoe-client add add-default-route=yes disabled=no interface=vlan_int name=internet use-peer-dns=yes user=<username> password=<password>
/interface list add name=WAN 
/interface list member add interface=internet list=WAN`
```

> * Change `<Enter ISP VLAN ID>` to the needed VLAN ID for your ISP
> * Change `<username>` to the needed username for your PPPoE Connection
> * Change `<password>` to the needed password for your PPPoE Connection
{% endtab %}

{% tab title="PPPoE" %}
```bash
/interface pppoe-client add add-default-route=yes disabled=no interface=ether1 name=internet use-peer-dns=yes user=<username> password=<password> 
/interface list add name=WAN 
/interface list member add interface=internet list=WAN`
```

> * Change `<username>` to the needed username for your PPPoE Connection
> * Change `<password>` to the needed password for your PPPoE Connection
{% endtab %}

{% tab title="Static IP" %}
```bash
/interface ethernet set ether1 name=internet 
/ip address add address=<Your IP Address> interface=internet 
/ip route add gateway=<Your IP Gateway> 
/ip dns set servers=<Your DNS Server> 
/interface list add name=WAN 
/interface list member add interface=internet list=WAN`
```

> * Change `<Your IP Address>` to IP Address given by your ISP
> * Change `<Your IP Gateway>` to IP Gateway given by your ISP
> * Change `<Your DNS Server>` to DNS Servers given by your ISP or any Public DNS you want to use.
{% endtab %}
{% endtabs %}

### NAT

Set up a NAT rule to translate all outgoing traffic to your public IP address.

```shell
/ip firewall nat
add action=masquerade chain=srcnat comment="Enable NAT on WAN interface" out-interface-list=WAN
```

### NTP

Configure the correct timezone to ensure the router displays the accurate time.

```shell
/system ntp client
set enabled=yes
/system ntp client servers
add address=0.pool.ntp.org
/system clock
set time-zone-name=Europe/Amsterdam # Change this to your timezone
```

### System

#### Hostname

```shell
/system identity
set name=<Your hostname>
```

#### User Account

```bash
/user add name=<Username> password=<Password> group=full
/user disable admin
```

## WireGuard

WireGuard is a modern, simple, and fast VPN protocol that uses state-of-the-art cryptography to provide a secure and efficient connection.

### Interface

Create a WireGuard interface and add it to the LAN group

```shell
/interface wireguard
add comment="Remote VPN" listen-port=13231 mtu=1420 name=wireguard1
/interface list member
add interface=wireguard1 list=LAN
```

### IP Address

Give the interface an IP Address. Make this is not in the same range as your LAN subnet

```
/ip address
add address=192.168.2.1/24 interface=wireguard1 network=192.168.2.0
```

### Firewall

Allow the WireGuard traffic in the firewall.

```shell
/ip firewall filter
add action=accept chain=input comment="Allow WireGuard traffic" dst-port=13231 protocol=udp
add action=accept chain=forward comment="Allow WireGuard to LAN" in-interface=wireguard1 out-interface=bridge1
```

{% hint style="warning" %}
Make sure the rules are before the `DROP` rule of the belonging chain.
{% endhint %}

### Client Configuration

Create a client in order to connect to the WireGuard tunnel

```shell
/interface wireguard peers
add allowed-address=192.168.2.2/32 interface=wireguard1 private-key=auto
```

> Ensure that each client has a unique `allowed-address`.

## Port forwarding

Port forwarding is a network technique that redirects incoming traffic from a specific port on a router to a designated device or service within a local network, enabling external access to internal network resources.

```shell
/ip firewall nat
add action=dst-nat chain=dstnat comment="Your Comment" dst-port=<Port to forward> in-interface-list=WAN protocol=tcp to-addresses=<IP To Forward to>
```

## Static DHCP

Static DHCP, also known as DHCP reservation, is a network configuration where a DHCP server assigns a specific IP address to a device based on its MAC address, ensuring the device always receives the same IP address within the network.

```shell
/ip dhcp-server lease
add address=<IP Adress> mac-address=<Client Mac Address> server=dhcp1
```
