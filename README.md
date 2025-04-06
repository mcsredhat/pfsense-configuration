# pfsense-configuration
# PfSense Configuration Guide

This repository contains comprehensive documentation for configuring pfSense firewalls, including interface setup, firewall rules, VLANs, DHCP, VPN, and more. This guide offers step-by-step instructions for network administrators setting up secure network environments with pfSense.

## Table of Contents

- [Basic Configuration](#basic-configuration)
  - [SSH Configuration](#ssh-configuration)
  - [Firewall Settings](#firewall-settings)
  - [Disable IPv6](#disable-ipv6)
  - [System Updates](#system-updates)
- [Network Configuration](#network-configuration)
  - [Adding Interfaces](#adding-interfaces)
  - [Firewall Rules](#firewall-rules)
  - [Creating Aliases](#creating-aliases)
  - [DHCP Configuration](#dhcp-configuration)
- [VLAN Configuration](#vlan-configuration)
  - [Creating VLANs](#creating-vlans)
  - [VLAN Interface Assignment](#vlan-interface-assignment)
  - [VLAN DHCP Configuration](#vlan-dhcp-configuration)
- [Switch Configuration](#switch-configuration)
  - [VLAN Setup on Cisco Switch](#vlan-setup-on-cisco-switch)
  - [Trunk Configuration](#trunk-configuration)
- [DMZ and Port Forwarding](#dmz-and-port-forwarding)
  - [Port Forwarding Setup](#port-forwarding-setup)
  - [WAN Firewall Rules](#wan-firewall-rules)
- [Routing Protocols](#routing-protocols)
  - [OSPF Configuration](#ospf-configuration)
- [VPN Configuration](#vpn-configuration)
  - [IPsec VPN Setup](#ipsec-vpn-setup)
  - [IPsec Tunnel Configuration](#ipsec-tunnel-configuration)
  - [VPN Firewall Rules](#vpn-firewall-rules)

## Basic Configuration

### SSH Configuration

```
System --> Advanced --> Admin Access --> 
- Select HTTP
- Select "Display page name first in browser tab"
- Under Secure Shell:
  - Select "Enable Secure Shell" 
  - SSH Port = 2222
- Save
```

### Firewall Settings

```
System --> Advanced --> Firewall/NAT --> 
- Firewall Optimization options = conservative
- Save
```

### Disable IPv6

```
System --> Advanced --> Networking --> 
- Under IPv6 Option, uncheck "All IPv6"
- Save
```

### System Updates

```
System --> Advanced --> Update --> 
- Check for updates
```

## Network Configuration

### Adding Interfaces

```
Interface --> Assignment --> 
- Select opt3
- Under General Configuration:
  - Check "Enable interface"
  - Description: LAN
  - IPv4 configuration Type = static IPv4
  - Static IPv4 Configuration = 44.1.2.32/24
- Save & Apply
```

### Firewall Rules

Creating a pass rule:
```
Firewall --> Rules --> LAN -->
- Action = Pass
- Interface = LAN
- Protocol = ANY
- Source = any
- Destination = the firewall
- Destination port range = 10443 - 10443
- Description = Allow all
- Save & Apply changes
```

Creating a block rule:
```
Firewall --> Rules --> LAN -->
- Action = Block
- Interface = LAN
- Protocol = ANY
- Description = Block Web Interface
- Save & Apply changes
```

### Creating Aliases

```
Firewall --> Alias --> 
- Name = Myprivate_IP
- Type = Network
- Add the following networks:
  - 172.16.1.0/24 (DMZ)
  - 44.1.2.0/24 (LAN)
  - 44.1.12.0/24 (VLAN12)
  - 44.1.22.0/24 (VLAN22)
- Save & Apply changes
```

Creating a block rule using alias:
```
Firewall --> Rules --> LAN -->
- Action = Block
- Interface = LAN
- Protocol = ANY
- Source = any
- Destination = single host or alias (DMZ)
- Description = Block DMZ
- Save & Apply changes
```

### DHCP Configuration

```
Services --> DHCP Server --> LAN -->
- Check "Enable DHCP Server"
- Range = 44.1.2.10 - 44.1.2.40
- Save & Apply changes
```

## VLAN Configuration

### Creating VLANs

```
Interface --> VLAN --> Add -->
- Parent Interface = LAN
- VLAN Tag = 12
- Description = connect to VLAN 12
- Save & Apply
```

### VLAN Interface Assignment

```
Interface --> Assignment --> 
- Click on Available Network port
- Select VLAN12
- Add
- Under General Configuration:
  - Check "Enable interface"
  - Description: VLAN12
  - IPv4 configuration Type = static IPv4
  - Static IPv4 Configuration = 44.1.12.1/24
- Save & Apply
```

VLAN firewall rules:
```
Firewall --> Rules --> VLAN12 -->
- Action = Pass
- Interface = LAN
- Protocol = ANY
- Source = any
- Destination = any
- Description = Allow all
- Save & Apply changes
```

### VLAN DHCP Configuration

```
Services --> DHCP Server --> VLAN12 -->
- Check "Enable DHCP Server"
- Range = 44.1.12.100 - 44.1.12.120
- Save & Apply changes
```

## Switch Configuration

### VLAN Setup on Cisco Switch

```
en
conf t
vlan 12
name Work
exit

vlan 22
name Home
exit

vlan 99
name Trunk
exit

interface GigabitEthernet0/0
 switchport mode access
 switchport access vlan 99
 no shutdown
 exit

interface range GigabitEthernet0/1-3
 switchport mode access
 switchport access vlan 12
 no shutdown
 exit

interface range GigabitEthernet1/0-3
 switchport mode access
 switchport access vlan 22
 no shutdown
 exit

interface Vlan 99
ip address dhcp
no shutdown
do wr
exit
```

### Trunk Configuration

```
interface GigabitEthernet0/0
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk native vlan 99
switchport trunk allowed vlan 12,22
exit
show vlan brief 
show interfaces trunk 
wr
```

## DMZ and Port Forwarding

### Port Forwarding Setup

```
Firewall --> NAT --> Port Forward --> Add -->
- Interface: WAN
- Protocol: TCP (or as needed)
- Source: Any
- Destination: WAN address (24.1.2.32)
- Destination Port Range: HTTP (80) and HTTPS (443)
- Redirect Target IP: 172.16.1.190 (DMZ web server)
- Redirect Target Port: 80
- Description: allow client from outside connect with web server on DMZ network
- Save
```

### WAN Firewall Rules

```
Firewall --> Rules --> WAN --> Add -->
- Action: Pass
- Interface: WAN
- Address Family: IPv4
- Protocol: TCP
- Source: Any
- Destination: Single host (172.16.1.190)
- Destination Port Range: HTTP (80) and HTTPS (443)
- Description: Allow access to DMZ web server
- Save
```

## Routing Protocols

### OSPF Configuration

1. Install the package:
```
System --> Package Manager --> Available Packages --> 
- Search for "frr"
- Install
```

2. Configure OSPF:
```
Services --> FRR OSPF --> Areas -->
- Area = 0.0.0.0
- Description = OSPF PFsense backbone
- Save

Services --> FRR OSPF --> Interfaces --> Add -->
- Interface = LAN
- Description = to insideLAN
- Under OSPF Interface Handling:
  - Area = 0.0.0.0
- Save

Services --> FRR OSPF --> Global Settings -->
- Check "Enable FRR"
- Default Router ID = 24.1.2.1
- Password = *Libya4ever*
- Save

Services --> FRR OSPF --> OSPF -->
- Check "Enable OSPF Routing"
- Router ID = 24.1.2.1
- Under Route Redistribution:
  - Check "Connected Network"
- Under Default Route Redistribution:
  - Check "Redistribute a Default route to neighbors"
  - Check "Always Redistribute a default route, even if routing table contains no default"
- Save
```

## VPN Configuration

### IPsec VPN Setup

```
VPN --> IPsec --> Advanced Setting -->
- Check "Enable Maximum MMS"
- Save
```

### IPsec Tunnel Configuration

Create VPN aliases:
```
Firewall --> Rules --> Aliases --> IP --> ADD -->
- Name = IPSecVPNConnects
- Type = Hosts
- IP or FQDN = 23.1.2.31, 172.25.71.33
- Save & Apply changes
```

Create WAN rules for VPN traffic:
```
Firewall --> Rules --> WAN --> Add -->
- Action = Pass
- Interface = WAN
- Address Family = IPv4
- Protocol = UDP
- Source = single host or alias (IPSecVPNConnections)
- Destination = WAN ADDRESS
- Destination port Range = ISAKMP (500)
- Save & Apply changes
```

### VPN Firewall Rules

Configure IPsec Phase 1:
```
VPN --> IPsec --> Tunnels --> Add P1 -->
- Description = VPN connect with 10.2.0.0/24 FG-FW
- Key Exchange version = IKEv2
- Internet Protocol = IPv4
- Interface = WAN
- Remote Gateway = 23.1.2.31

Under Phase 1:
- Authentication Method = Mutual PSK
- My Identifier = My IP Address
- Peer Identifier = Peer IP Address
- Pre-shared key = *Libya4ever*

Under Phase 1 Proposal:
- Encryption Algorithm = AES 256bit
- Hash Algorithms = SHA1
- DH Group = 5 OR 14
- Save & Apply changes
```

Configure IPsec Phase 2:
```
Click on "Show Phase 2 Entries" --> Add P2 -->
- Remote Network = 10.2.0.0/24

Under Phase 2 Proposal:
- Protocol = ESP
- Encryption Algorithms = AES Auto
- Hash Algorithms = SHA1
- PFS Key Group = 5(1024) OR 14(1024)
- Save & Apply Changes
```

Configure IPsec firewall rules:
```
Firewall --> Rules --> IPSec --> ADD -->
- Action = Pass
- Interface = IPSec
- Address Family = IPv4
- Protocol = Any
- Save & Apply Changes
```

## Testing and Verification

- Check VPN status: Status --> IPSec --> Overview
- Test connectivity: ping 10.2.0.* (local LAN on the first site)
- Verify external connectivity: ping 8.8.8.8

## Contributing

Feel free to contribute to this guide by submitting pull requests or opening issues for improvements and corrections.


