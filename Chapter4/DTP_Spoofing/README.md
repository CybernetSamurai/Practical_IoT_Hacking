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

## Verify Connectivity

## Verify Subnet Segmentation

## Exploit

## Mitigations
