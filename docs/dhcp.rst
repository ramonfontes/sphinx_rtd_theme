************
DHCP
************


.. Note::

    **Nesta demo você irá aprender como configurar um servidor DHCP:** 

    **Requisitos:** 
    
    - Mininet-WiFi - https://github.com/intrig-unicamp/mininet-wifi
    - isc-dhcp-server (`sudo apt install isc-dhcp-server`)      

Antes de tudo você precisa identificar a topologia de rede que será gerada através do código abaixo:


.. code:: python

  #!/usr/bin/python
     
  """@author: Ramon Fontes
  @email: ramon.fontes@imd.ufrn.br"""

  from mininet.log import setLogLevel, info
  from mininet.cli import CLI
  from mininet.net import Mininet


  def topology():

      net = Mininet()

      info("*** Creating nodes\n")
      h1 = net.addHost('h1', mac='00:00:00:00:00:02', ip='0/0')
      h2 = net.addHost('h2', ip='192.168.11.1/24', inNamespace=False)
      s1 = net.addSwitch('s1', failMode='standalone')

      info("*** Creating links\n")
      net.addLink(s1, h1)
      net.addLink(s1, h2)

      info("*** Starting network\n")
      net.start()

      h1.cmd("echo 1 > /proc/sys/net/ipv4/ip_forward")

      info("*** Running CLI\n")
      CLI(net)

      info("*** Stopping network\n")
      net.stop()

  if __name__ == '__main__':
      setLogLevel('info')
      topology()


Uma vez compreendida a topologia de rede, execute um código com conteúdo exatamente igual ao apresentado acima. Considerando que você tenha salvo o conteúdo acima em um arquivo com nome `dhcp.py`, execute conforme abaixo:


.. code:: console

  $ sudo python dhcp.py


Agora, em um novo terminal, edite o arquivo `/etc/dhcp/dhcpd.conf` de forma que ele possua apenas o conteúdo abaixo:

.. code:: console
  option domain-name-servers 192.168.11.1;
  subnet 192.168.11.0 netmask 255.255.255.0 {
      range 192.168.11.2 192.168.11.254;
      option routers 192.168.11.1;
      default-lease-time 6000;
      max-lease-time 72000;
      INTERFACES="h2-eth0";
  }


Finalmente, abra um terminal via xterm para `h2`, que é o nosso servidor dhcp, e reinicie o serviço dhcp de forma que as configurações realizadas acima sejam aplicadas.

.. code:: console

  mininet> xterm h2
  h2# sudo service isc-dhcp-server restart 


Neste momento, o servidor dhcp `h2` deve estar operacional. Portanto, caso `h1` solicite um endereço IP, este conseguirá obter um através do servidor `h2`. A solicitação de endereço IP pode ser realizada através da ferramente dhclient, conforme abaixo:

.. code:: console

  mininet> h1 dhclient
  
  
.. Note::

   - Experimente reiniciar o código da topologia do Mininet-WiFi utilizando o wireshark na porta `s1-eth1` e, após isso, executar o dhclient mais uma vez a partir de `h1`. Você conseguirá observar as mensagens do protocolo DHCP.
