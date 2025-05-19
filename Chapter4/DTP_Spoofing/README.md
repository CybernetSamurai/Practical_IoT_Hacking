# VLAN Hopping: Switch Spoofing

## Lab Topology
![](assets/net-topo.png)

## Initial Setup
### Switch Port Layout
| Switch  | Port  | Connected Device | VLAN    | Port Mode       |
|---------|-------|------------------|---------|-----------------|
| SWITCH1 | Fa0/0 | SWITCH2          | N/A     | Trunk (Dynamic) |
| SWITCH1 | Fa0/1 | CAMERA_01        | VLAN 20 | Access          |
| SWITCH1 | Fa0/2 | ATTACKER         | VLAN 10 | Access          |
| SWITCH2 | Fa0/0 | SWITCH1          | N/A     | Trunk (Dynamic) |
| SWITCH2 | Fa0/1 | CAMERA_02        | VLAN 20 | Access          |
| SWITCH2 | Fa0/2 | GUEST_PC         | VLAN 10 | Access          |

## Verify Connectivity

## Verify Subnet Segmentation

## Exploit

## Mitigations
