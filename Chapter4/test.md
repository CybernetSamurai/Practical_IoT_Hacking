test

## 🖥️ Lab Topology

![Lab Topology](assets/double-tagging-topo.png)


Create new vlan
<pre>
  SWITCH1> enable
  SWITCH1# configure terminal
  SWITCH1(config)# vlan 20
  SWITCH1(config-vlan)# name BLUE
  SWITCH1(config-vlan)# exit
  SWITCH1# show vlan brief
  
  VLAN Name                             Status    Ports
  ---- -------------------------------- --------- -------------------------------
  1    default                          active    Et0/0, Et0/1, Et0/2, Et0/3
                                                  Et1/0, Et1/1, Et1/2, Et1/3
                                                  Et2/0, Et2/1, Et2/2, Et2/3
                                                  Et3/0, Et3/1, Et3/2, Et3/3
  20   BLUE                             active
</pre>
