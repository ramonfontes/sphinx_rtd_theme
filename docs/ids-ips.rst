************
IDS/IPS
************


.. Note::

    **Nesta demo você irá aprender como executar um IDS/IPS:** 

    **Requisitos:** 
    
    - Containernet - https://github.com/ramonfontes/containernet
    - Snort
    
    **A VM disponível em https://github.com/intrig-unicamp/mininet-wifi deverá possuir todos os recursos necessários para reproduzir este documento. Porém, para utilizar o containernet você deverá entrar no diretório ~/containernet e dentro dele executar o comando `sudo make install` Caso alguma dependência esteja faltando ela terá de ser resolvida.**
    

Antes de tudo você precisa identificar a topologia de rede que será gerada através do código abaixo:


.. code:: python

    #!/usr/bin/python
     
    """@author: Ramon Fontes
    @email: ramon.fontes@imd.ufrn.br"""

    from containernet.net import Containernet
    from containernet.node import Docker
    from containernet.cli import CLI
    from mininet.log import info, setLogLevel


    def topology():
        net = Containernet()

        info('*** Adding docker containers\n')
        h1 = net.addHost('h1', ip='10.0.0.1', cls=Docker,
                         dimage="ramonfontes/ids-ips")
        h2 = net.addHost('h2', ip='10.0.0.2', cls=Docker,
                         dimage="ramonfontes/ids-ips")
        h3 = net.addHost('h3', ip='10.0.0.254', cls=Docker,
                         dimage="ramonfontes/ids-ips")
        s1 = net.addSwitch('s1', failMode="standalone")

        info('*** Configuring WiFi nodes\n')
        net.configureWifiNodes()

        info("*** Associating Stations\n")
        net.addLink(h1, s1)
        net.addLink(h2, s1)
        net.addLink(h3, s1)

        info("*** Starting network\n")
        net.build()
        s1.start([])

        info('*** Running CLI\n')
        CLI(net)

        info('*** Stopping network\n')
        net.stop()


    if __name__ == '__main__':
        setLogLevel('info')
        topology()


Então, após executar o código, o Snort pode ser iniciado da seguinte forma:


.. code:: console

  mininet-wifi> xxxx


