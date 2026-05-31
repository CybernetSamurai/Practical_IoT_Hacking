# Gateway Hijacking: DHCP Starvation (2025.12)
The Dynamic Host Configuration Protocol (DHCP) automates the assignment of IP addresses and related information to network clients. It is set to distribute a limited number of IP addresses based on the subnet mask. When a new device connects to the network, it broadcasts a request for any available DHCP servers. Any server can respond with an offer that includes a client address, default route, search domain, DNS servers, and more. Leased addresses are logged by the server along with the client's MAC address. This lab demonstrates how to exploit this trust relationship by 'starving' the legitimate DHCP server of all available addresses, forcing new clients to obtain leases from a rogue server, and redirecting traffic to the attacker's machine.

## Lab Topology

![Network topolgoy](assets/figure1.png)

## Initial Setup
This lab uses two Raspberry Pi 3 Model B+ devices as IoT endpoints: Camera1 and Server. The Attacker machine runs Kali Linux on a typical x86_64 platform. All three devices are connected to a Cisco Catalyst 3560G Layer 3 switch. The 3560G switch segregates two VLANs to emulate simple network routing.

### Switch Port Layout
| Switch | Port   | Endpoint | VLAN | Port Mode |
|--------|--------|----------|------|-----------|
| SW3    | Gi0/17 | Attacker | 10   | Access    |
| SW3    | Gi0/18 | Camera1  | 10   | Access    |
| SW3    | Gi0/33 | Server   | 20   | Access    |

### Linux DHCP IPv4 Configuration
Configure the network clients to request dynamic addresses as depicted above.

Start the NetworkManager service's terminal user interface (TUI).
```bash
nmtui
```

Navigate to 'Edit a connection'.

![selecting network interfaces with NetworkManager](assets/figure2.png)

Under 'Ethernet', select the connection associated with the primary network card used, then select '<Edit...>'. Next, change 'IPv4 CONFIGURATION' from 'Manual' to 'Automatic'.

![configuring a network interface with networkmanager](assets/figure3.png)

If necessary, cycle the interface to force DHCP discovery.
```bash
nmcli connection down <conn_name>
nmcli connection up <conn_name>
```

When the DHCP server is online, verify that a new address has been applied.
```bash
ip --color --brief --family inet address show dev eth0
```

### IP Address Assignments
| Device        | Network      | IP Address  |
|---------------|--------------|-------------|
| SW3 (VLAN 10) | 10.0.10.0/24 | 10.0.10.254 |
| SW3 (VLAN 20) | 10.0.20.0/24 | 10.0.20.254 |
| Attacker      | 10.0.10.0/24 | DHCP        |
| Camera1       | 10.0.10.0/24 | DHCP        |
| Server        | 10.0.20.0/24 | DHCP        |

### Client MAC Addresses
| Device        | Hardware Address 
|---------------|
| SW3 (VLAN 10) |
| SW3 (VLAN 20) |
| Attacker      |
| Camera1       |
| Server
