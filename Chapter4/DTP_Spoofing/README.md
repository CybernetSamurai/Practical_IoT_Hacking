# VLAN Hopping: Switch Spoofing
Switch spoofing is a VLAN hopping technique that abuses Dynamic Trunking Protocol (DTP), a Cisco proprietary protocol used to automatically negotiate trunk links between switches. In this attack, an adversary connects a device to an access port on a switch and configures their interface to actively participate in DTP negotiations. If the switch port is configured to negotiate (which is enabled by default), the attacker's device can spoof itself as a switch by sending forged DTP packets. This can trick the legitimate switch into establishing a trunk link, allowing the attacker to receive traffic from multiple VLANs, including those they shouldn’t have access to. Once a trunk is established, the attacker can craft and analyze 802.1Q-tagged packets, making it possible to launch further attacks or passively eavesdrop on inter-VLAN traffic. This vulnerability can be mitigated by disabling DTP on all client-facing switch ports.

> This lab was inspired by *Practical IoT Hacking* (O'Reilly), Chapter 4: **Network Assessments**, Section *"Hopping into the IoT Network"*. It assumes basic knowledge of GNS3 and networking.

## Lab Topology
![](assets/lab-topo.png)

## Initial Setup
### Switch Port Layout
| Switch  | Port  | Connected Device | VLAN    | Port Mode       |
|---------|-------|------------------|---------|-----------------|
| SWITCH1 | Fa0/0 | SWITCH2          | N/A     | Trunk (Dynamic) |
| SWITCH1 | Fa0/1 | ATTACKER         | VLAN 10 | Access          |
| SWITCH1 | Fa0/2 | CAMERA_01        | VLAN 20 | Access          |
| SWITCH2 | Fa0/0 | SWITCH1          | N/A     | Trunk (Dynamic) |
| SWITCH2 | Fa0/1 | GUEST_LAPTOP     | VLAN 10 | Access          |
| SWITCH2 | Fa0/2 | CAMERA_02        | VLAN 20 | Access          |

### VPCS Static IPv4 Configuration
To keep the lab simple, both **CAMERA_01** and **CAMERA_02** are implemented using [VPCS](https://docs.gns3.com/docs/emulators/vpcs/) (Virtual PC Simulator) appliances. This provides a lightweight way to simulate basic IoT device connectivity within the network. The following commands assign an IP address and saves it across reboots.
<pre>
  VPCS> ip 192.168.0.x/24
  VPCS> save
</pre>

### Docker Container IPv4 Configuration
The **ATTACKER** machine is a Docker container. The imaged used in this lab is [finchsec/scapy](https://hub.docker.com/r/finchsec/scapy), which gives access to the Scapy program for crafting custom network packets to perform this attack. Configuring IP addresses for Docker appliances in GNS3 is easy as shown below:

**Right-click**, then select `Edit Config` from the context menu.

![Attacker Interfaces](assets/attacker-interfaces.png)

```
apt update && apt -y install yersinia iproute2 inetutils-ping kmod linux-modules-$(uname -r)
```

### Cisco IOS
**SWITCH1** and **SWITCH2** are virtual Cisco IOS Layer 2 devices. You can use either the IOSvL2 or IOU L2 GNS3 appliance templates for this lab. They should work as standard switches 'out of the box', no configurations necessary. Note that since these are virtualized appliances, they may not behave exactly like their hardware counterparts.

## Verify Connectivity

### Enabling telnet on Cisco switch
Configure management IP
<pre>
  Switch> enable
  Switch# configure terminal
  Switch(config)# interface vlan1
  Switch(config-if)# ip address [ip_address] [subnet]
  Switch(config-if)# no shutdown
  Switch(config-if)# exit
</pre>

Enable remote management
<pre>
  Switch# configure terminal
  Switch(config)# line vty 0 4
  Switch(config-line)# no login
  Switch(config-line)# transport input telnet
  Switch(config-line)# exit
  Switch(config)# enable password [passowrd]
  Switch(config)# exit
</pre>

## IOS VLAN Configuration

## Verify Subnet Segmentation

## Exploit

## Mitigations
