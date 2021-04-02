************
IDS/IPS com Snort
************


O Snort é um Sistema de Detecção de Intrusão de código aberto que você pode usar em seus sistemas Linux. Este tutorial revisará a configuração básica do Snort IDS e ensinará como criar regras para detectar atividades no sistema.


.. Note::

    **Nesta demo você irá aprender como executar um IDS/IPS:** 

    **Requisitos:** 
    
    - Containernet - https://github.com/ramonfontes/containernet
    - Snort
    - nmap
    - hping3
    
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


Para este tutorial, a rede que usaremos é: 10.0.0.0/8. Edite seu arquivo /etc/snort/snort.conf e substitua o ```any``` ao lado de ipvar ```$HOME_NET``` com estas informações de rede, como mostrado na tela de exemplo abaixo:

.. image:: https://github.com/ramonfontes/sphinx_rtd_theme/blob/master/docs/imgs/snort1.png?raw=true

Alternativamente, você também pode definir endereços IP específicos separados por vírgula entre [], como mostrado nesta captura de tela:

.. image:: https://github.com/ramonfontes/sphinx_rtd_theme/blob/master/docs/imgs/snort2.png?raw=true

Agora vamos executar o comando abaixo na linha de comando:

# snort -d -l /var/log/snort/ -h 10.0.0.0/8 -A console -c /etc/snort/snort.conf

Onde:
   - d = tells snort to show data
   - l = determines the logs directory
   - h = specifies the network to monitor
   - A = instructs snort to print alerts in the console
   - c = specifies snort the configuration file


.. code:: console

  mininet-wifi> xxxx


