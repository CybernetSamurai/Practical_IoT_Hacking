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
| Device        | Hardware Address  |
|---------------|-------------------|
| SW3 (VLAN 10) | 3c:df:1e:cf:81:c1 |
| SW3 (VLAN 20) | 3c:df:1e:cf:81:c2 |
| Attacker      | 8c:ec:4b:c3:9a:12 |
| Camera1       | b8:27:eb:9d:64:9a |
| Server        | b8:27:eb:2f:1c:13 |

## Cisco IOS Configuration

### VLAN Configuration
Create two VLANs, RED_NET (10), and BLUE_NET (20), per the network topology.
```console
vlan 10
 name RED_NET
vlan 20
 name BLUE_NET
```

Configure the switch interfaces as access ports for the VLANs.
```console
interface range Gi0/17 - 32
 switchport access vlan 10
 switchport mode access
 switchport nonegotiate
 no shutdown
interface range Gi0/33 - 48
 switchport access vlan 20
 switchport mode access
 switchport nonegotiate
 no shutdown
```

Assign static IP addresses to the VLAN interfaces on SW3. These will serve as the deafult gateways for clients in both subnets.
```console
interface vlan 10
 ip address 10.0.10.254 255.255.255.0
 no shutdown
interface vlan 20
 ip address 10.0.20.254 255.255.255.0
 no shutdown
```

In global configuration mode, enable routing for inter-vlan communication.
```console
ip routing
```

### DHCP Configuration
Set up the DHCP lease pools for both subnets. The 'default-router' parameter points to the static addresses assigned to SW3's VLAN interfaces.
```console
ip dhcp pool RED_NET
 network 10.0.10.0 255.255.255.0
 default-router 10.0.10.254
 lease 0 0 15
ip dhcp pool BLUE_NET
 network 10.0.20.0 255.255.255.0
 default-router 10.0.20.254
 lease 0 0 15
```

## Verify Network Connectivity
List the current addresses that have been leased. Attacker and Camera1 should be assigned addresses in VLAN 10, and Server should receive addresses in VLAN 20.
```
SW3# show ip dhcp binding

IP address        Hardware address        Type
10.0.10.1         01b8.27eb.9d64.9a       Automatic
10.0.10.2         018c.ec4b.c39a.12       Automatic
10.0.20.1         01b8.27eb.2f1c.13       Automatic
```

Check the default gateway of Camera1; it should point to the switch's VLAN interface address.
```bash
ip route show dev eth0
```

![camera1s default gateway](assets/figure4.png)

Ensure that the Attacker can ping Camera1 and Server.
```bash
fping -I eth0 10.0.10.1 10.0.20.1
```

![testing inter-vlan connectivity](assets/figure5.png)

### Prepare Camera1 for Exploit
Forcing a Linux client to send a new DHCP Discovery request can be difficult because both the DHCP server and client cache previously assigned addresses. However, this step is essential for Camera1 to accept an address from the rogue DHCP server.

For the lab, delete the DHCP entry on the switch, even though it will be cleared automatically when the lease expires. Replace <ip_addr> with Camera1's current assignment.
```console
clear ip dhcp binding <ip_addr>
```

Disable Camera1's interface. In this example, the eth0 connection is named 'CI670Net', although it might be labeled as 'Wired Connection 1' by default.
```bash
nmcli connection down CI670Net
```

To prevent the device from requesting the same address, clear the NetworkManager daemon's previous lease cache.
```bash
rm -f /var/lib/NetworkManager/*.lease
```

Shutdown Camera1 until the exploit is ready.
```bash
shutdown now
```

## Exploit

### dhcpstarv
`dhcpstarv` is a security auditing tool designed to exhaust all IP addresses on the local network. The time required to deplete the server depends on the target DHCP server's speed and capabilities and may take several minutes.

Install `dhcpstarv` on the Attacker machine.
```bash
apt -y install dhcpstarv
```

Execute the script using the interface connected to VLAN 10. Optional parameters include setting a destination MAC address and ignoring requests for specific addresses.
```bash
dhcpstarv -i eth0
```

![dhcp discover denial of service attack](assets/figure6.png)

After a few minutes, check the lease count for RED_NET. When the `leased addresses` value reaches 253, all addresses in this subnet are leased and cannot be assigned to valid clients.
```
SW3# show ip dhcp pool | include Pool|Total addresses|Leased addresses

Pool RED_NET :
 Total addresses  : 254
 Leased addresses : 253
Pool BLUE_NET :
 Total addresses  : 254
 Leased addresses : 1
```

### Yersinia
Yersinia is a security tool that supports a variety of Layer 2 network attacks. One of these attacks is configuring a rogue DHCP server to serve clients after the primary server has exhausted its address pool.

Start Yersinia in `Interactive Mode`.
```bash
yersinia -I
```

Press `G` to display the list of protocols you can target and select DHCP. Then, press `X` to view available DHCP attacks, and choose `2` to create a DHCP rogue server.

![yersinia dhcp mode](assets/figure7.png)

Fill in the parameters for the new DHCP server as shown  below. The `Router` and `Server ID` should match the IP address assigned to the Attacker's machine. In this case, the Attacker's IP address is 10.0.10.2.

### Rogue DHCP Server
| Parameter   | Value         |
|-------------|---------------|
| Server ID   | 10.0.10.2     |
| Start IP    | 10.0.10.50    |
| End IP      | 10.0.10.100   |
| Lease Time  | 3600          |
| Renew Time  | 3600          |
| Subnet Mask | 255.255.255.0 |
| Router      | 10.0.10.2     |
| DNS Server  | 10.0.10.2     |
| Domain      | hacked.net    |

![rogue dhcp server configuration](assets/figure8.png)

### Force DHCP Discovery
Boot Camera1 to begin a fresh DHCP Discovery request. After a few seconds, check if the default gateway has been updated. You might need to restart Yersinia's DHCP server. If Camera1 still gets legitimate IP addresses from the switch, clear the local leases, delete the IP-MAC binding on SW3, and reboot Camera1. Multiple attempts could be necessary for a successful attack.

![camera1 default gateway changed](assets/figure9.png)

## Traffic Interception

### ICMP Redirects
To prevent ICMP redirect messages from updating Camera1's routing tables, set `net.ipv4.conf.all.accept_redirects` to 0. Make the following kernel change in Camera1.
```bash
sysctl --write net.ipv4.conf.all.accept_redirects=0
```

Flush Camera1's routing tables.
```bash
sysctl --write net.ipv4.route.flush=1
```

Now that the default gateway for Camera1 is pointed towards Attacker, all egress traffic will be forwarded to the Attacker's computer. Try pinging Server from Camera1.

![camera1 network disruption](assets/figure10.png)

Camera1 cannot ping the server because the attacker has not configured packet forwarding. Although it fails, sniffing traffic on the attacker shows ICMP packets from Camera1 passing through.

![attacker receiving traffic from camera1](assets/figure11.png)

### IP Forwarding
Check whether IP forwarding is active on the Attacker (1 = enabled, 0 = disabled).
```bash
cat /proc/sys/net/ipv4/ip_forward
```

Enable IP forwarding.
```bash
sysctl --write net.ipv4.ip_forward=1
```

Enable `Promiscuous Mode` on eth0 to prevent the NIC from dropping packets.
```bash
ip link set eth0 promisc on
```

Pinging the Server from Camera1 will now be successful.

![ip forwarding enabled](assets/figure12.png)

On the Attacker's machine, we can see the ICMP echo requests (type 8) and ICMP redirections (type 5), but no replies. This is because the attack only lets us see half the conversation.

![traffic interception successful](assets/figure13.png)

### Exploring MQTT
This attack can be used to steal plaintext credentials and intercept other sensitive data transmitted over the network. To illustrate this, set up Server as an MQTT broker and Camera1 as an MQTT subscriber. In this setup, the broker uses basic password authentication without network encryption.

Publish a message from Camera1 and intercept it. Use `tshark` to analyze packets efficiently. Although only half the coversation is captured, the Attacker can see session credentials and messages.
```bash
tshark -i eth0 -n -Y 'mqtt'
```

![reading mqtt messages captured over the network](assets/figure14.png)

Extract the username and password used by Camera1.

![intercepting the mqtt session username and password](assets/figure15.png)

Extract the message that was published to the broker.

![decoding the mqtt message from hexadecimal](assets/figure16.png)

## Mitigations

### DHCP Snooping
To prevent rogue DHCP attacks, Cisco introduced a security feature called `DHCP snooping`. This feature permits DHCP responses solely from trusted ports. If an untrusted port attempts to send DHCP offer packets, the switch discards the packet.

Make the following configuration changes in SW3 to enable DHCP snooping.
```console
ip dhcp snooping
ip dhcp snooping vlan 10,20
```

With DHCP Snooping enabled, repeat the steps from this lab to try to hijack Camera1. Even if the legitimate DHCP server were turned off, the attack would still fail.

In SW3, see the number of DHCP Offers that were rejected from the Attacker.
```
SW3# show ip dhcp snooping statistics

 Packets Forwarded                       = 52
 Packets Dropped                         = 3
 Packets Dropped from untrusted ports    = 3
```

DHCP Snooping also drops packets if the Client MAC address in the DHCP layer does not match the source MAC address in the Ethernet layer. This helps prevent tools like `dhcpstarv`, which spoofs client's MAC addresses en masse but does not alter the source MAC address.

### Port Security
For enhanced network security, Cisco provides port security. Port Security on a Cisco switch restricts the number of unique MAC addresses a port can learn. It defends against unauthorized endpoints and MAC flooding attacks, such as DHCP starvation.

To enable port security on SW3, configure it to allow only one MAC address per port. The settings `Maximum 1` and `Violation shutdown` mean that if more than one MAC address attempts to connect, the port will be disabled.
```console
interface range Gi0/17 - 48
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation shutdown
```
