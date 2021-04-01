************
Denial of Service (DoS)
************

Um ataque de negação de serviço (DoS) é uma tentativa de fazer com que algum serviço fique indisponível. Diferente de outros tipos de ataque, que estabelecem pontos de apoio ou sequestram dados, ataques DoS não ameaçam dados sensíveis. Este tipo de ataque tem apenas um objetivo de fazer com que sistemas fiquem indisponíveis para usuários legítimos. Contudo, alguns ataques DoS pode também serem usados para criar outros tipos de ataque para outras atividades maliciosas, como, por exemplo, derrubar firewalls de aplicativos da web, etc.).


.. Note::

    **Nesta demo você irá aprender como realizar o ataque de DoS:** 

    **Requisitos:** 
    
    - Containernet - https://github.com/ramonfontes/containernet
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
                            dimage="ramonfontes/dos-attack")
        h2 = net.addHost('h2', ip='10.0.0.2', cls=Docker,
                         dimage="ramonfontes/dos-attack")
        h3 = net.addHost('h3', ip='10.0.0.254', cls=Docker,
                         dimage="ramonfontes/dos-attack")
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


Então, após executar o código, um simples ataque DoS (não DDoS) poderia ser realizado através do comando abaixo:


.. code:: console

  mininet-wifi> h1 hping3 -S --flood -V -p 80 10.0.0.254


.. Note::

   - hping3: carrega o hping3
   - S: especifica pacotes SYN
   - flood: respostas serão ignoradas (é por isso que as respostas não serão mostradas) e os pacotes serão enviados o mais rápido possível
   - V: Verbosity
   - p 80: porta 80
   - 170.155.9.185: IP alvo

