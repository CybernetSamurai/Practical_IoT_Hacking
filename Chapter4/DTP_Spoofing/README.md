# VLAN Hopping: Switch Spoofing

## Lab Topology
![](assets/net-topo.png)

## Initial Setup
### Switch Port Layout
| Switch   | Port  | Connected Device | VLAN    | Port Mode       |
|----------|-------|------------------|---------|-----------------|
| C2950_01 | Fa0/0 | C2950_02         | N/A     | Trunk (Dynamic) |
| C2950_01 | Fa0/1 | CAMERA_01        | VLAN 20 | Access          |
| C2950_01 | Fa0/2 | ATTACKER         | VLAN 10 | Access          |
| C2950_02 | Fa0/0 | C2950_01         | N/A     | Trunk (Dynamic) |
| C2950_02 | Fa0/1 | CAMERA_02        | VLAN 20 | Access          |
| C2950_02 | Fa0/2 | GUEST_PC         | VLAN 10 | Access          |

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
  Switch(config-line)# enable password [passowrd]
  Switch(config-line)# exit
</pre>

## Verify Connectivity

## Verify Subnet Segmentation

## Exploit

## Mitigations
