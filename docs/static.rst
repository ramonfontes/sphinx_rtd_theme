************
Firewall
************


.. Note::

    **Nesta demo você irá aprender como realizar roteamento estático:** 

    **Requisitos:** 
    
    - Mininet - https://github.com/mininet/mininet
      

Antes de tudo você precisa identificar a topologia de rede que será gerada através do código abaixo:


.. code:: python

      #!/usr/bin/python

      """@author: Ramon Fontes
      @email: ramon.fontes@imd.ufrn.br"""

      from mininet.node import Controller
      from mininet.log import setLogLevel, info
      from mininet.cli import CLI
      from mininet.net import Mininet


      def topology():
          "Create a network."
          net = Mininet()

          info("*** Creating nodes\n")
          h1 = net.addHost('h1', ip='10.0.0.1/8')
          h2 = net.addHost('h2', ip='192.168.0.2/24')
          r1 = net.addHost('r1')
          r2 = net.addHost('r2')

          info("*** Creating Links\n")
          net.addLink(h1, r1)
          net.addLink(r1, r2)
          net.addLink(h2, r2)    

          info("*** Starting network\n")
          net.build()

          #r1.cmd('sysctl net.ipv4.ip_forward=1')
          #r2.cmd('sysctl net.ipv4.ip_forward=1')
          #r1.cmd('ifconfig r1-eth0 10.0.0.2/8')
          #r1.cmd('ifconfig r1-eth1 172.16.0.1/30')
          #r2.cmd('ifconfig r2-eth0 172.16.0.2/30')
          #r2.cmd('ifconfig r2-eth1 192.168.0.1/24')
          #r1.cmd('route add -net 192.168.0.0/24 gw 172.16.0.2')
          #r2.cmd('route add -net 10.0.0.0/8 gw 172.16.0.1')
          #h1.cmd('route add default gw 10.0.0.2')
          #h2.cmd('route add default gw 192.168.0.1')

          info("*** Running CLI\n")
          CLI(net)

          info("*** Stopping network\n")
          net.stop()


      if __name__ == '__main__':
          setLogLevel('info')
          topology()


Então, execute o código acima e experimente criar rotas estáticas de forma que `h1` consiga se comunicar com `h2`.


