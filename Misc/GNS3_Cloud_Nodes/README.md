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
