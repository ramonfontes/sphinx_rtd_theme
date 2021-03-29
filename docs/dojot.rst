************
Dojot
************

In this activity we will see some examples of how to simulate a 6loWPAN network for the Internet of Things. 6LoWPAN (IPv6 over Low power Wireless Personal Area Networks) is an IP protocol that creates and maintains the specifications that allow us to use the IPv6 protocol on the IEEE 802.15.4 standard. It has been widely used in the implementation of sensor networks with energy limitations, low signal range, low transmission rates and low cost. The integration of sensor networks with the Internet is seen as essential for the Internet of Things (IoT), allowing the use of distributed sensor applications.

.. Note::

    First of all, you will need to install _mosquitto_, _mosquitto-clients_ and _paho-mqtt_ packages as well as _Dojot_ (http://www.dojot.com.br/).

    Follow the detailed install instructions available here: https://docs.google.com/document/d/1tbeC0T-zd7yNnSS16IYYdp4UXXk-ePKg1OZm6_XaDAQ/edit?usp=sharing


**Activity 1 - MQTT basics with Mosquitto**

.. Note::

    As per the Mininet-WiFi Book


**Activity 2 - Dojot**

.. Note::

    How to install Dojot?

    You can follow the instructions available at https://dojotdocs.readthedocs.io/en/latest/installation-guide.html#installation. However, I strongly suggest you to use the docker-compose.yml file available at https://gist.github.com/ramonfontes/fd8360e1c7a1d8f0f9b5e8e1d3f555de. That said, you have to replace the docker-compose.ym file provided by dojot with the suggested one.


In this activity we will simulate some sensors that send the current temperature to Dojot. The code will find below should work with the most recent version of Mininet-WiFi, but you can consider the commit **db71bf47cb** if you face problems with it.

You can run the following commands if you want to work with the commit **db71bf47cb**:


.. code:: console
    ~/mininet-wifi$ git checkout db71bf47cb
    ~/mininet-wifi$ sudo make install


Now that we already have our environment ready to be used, let's take the scripts we will use in this activity. You can find three codes below:

**lowpan.py**: the Mininet-WiFi script (We will only run this script. The others will be loaded automatically)
**temperature-simulator.py**: our temperature simulator
**controller-sub.py**: code of the controller that will receive messages sent by Dojot


**lowpan.py**:

.. code:: python

  #!/usr/bin/python

  """
  @author: Ramon Fontes
  @email: ramon.fontes@imd.ufrn.br

  lowpan.py: works with both fakelb or mac802154_hwsim modules
  mac802154_hwsim is only supported from kernel 4.18
  """

  import os
  import sys

  from mininet.log import setLogLevel, info
  from mininet.node import Controller
  from mn_wifi.cli import CLI
  from mn_wifi.net import Mininet_wifi
  from mn_wifi.sixLoWPAN.node import OVSSensor
  from mininet.term import makeTerm


  def topology():
      "Create a network."
      net = Mininet_wifi(iot_module='fakelb', apsensor=OVSSensor,
                         disable_tcp_checksum=True,
                         controller=Controller)

      DEVICE_ID = sys.argv[1]
      TOPIC = sys.argv[2]

      info("*** Creating nodes\n")
      sensor1 = net.addSensor('sensor1', ip6='2001::1/64',
                              panid='0xbeef', position='20,80,0')
      sensor2 = net.addSensor('sensor2', ip6='2001::2/64',
                              panid='0xbeef', position='20,50,0')
      sensor3 = net.addSensor('sensor3', ip6='2001::3/64',
                              panid='0xbeef', position='20,20,0')
      ap1 = net.addAPSensor('ap1', panid='0xbeef', datapath='user',
                            position='50,50,0')
      c1 = net.addController('c1')

      info("*** Configuring wireless nodes\n")
      net.configureWifiNodes()

      info("*** Plotting graph\n")
      net.plotGraph(max_x=100, max_y=100)

      info("*** Starting network\n")
      net.build()
      net.addNAT(name='wan0', linkTo='ap1').configDefault()
      c1.start()
      ap1.start([c1])

      info("*** Configuring the network environment\n")
      ap1.cmd('sysctl net.ipv6.conf.all.forwarding=1')
      ap1.cmd('sysctl net.ipv6.conf.all.proxy_ndp=1')
      sensor1.cmd('route add -A inet6 default gw 2001::6')
      sensor2.cmd('route add -A inet6 default gw 2001::6')
      sensor3.cmd('route add -A inet6 default gw 2001::6')
      ap1.cmd('ip -6 addr add 2001::6/64 dev ap1-pan0')
      os.system('ip -6 addr add 2002::1/64 dev wan0-eth0')
      ap1.cmd('ip -6 addr add 2002::2/64 dev ap1-eth5')

      info("*** Configuring ip6tables rules\n")
      iface = 'wan0-eth0'
      ip6 = '2002::'
      os.system('ip6tables -I FORWARD -i {} -d {} -j DROP'.format(iface, ip6))
      os.system('ip6tables -A FORWARD -i {} -s {} -j ACCEPT'.format(iface, ip6))
      os.system('ip6tables -A FORWARD -o {} -d {} -j ACCEPT'.format(iface, ip6))
      os.system('ip6tables -t nat -A POSTROUTING -s {} \'!\' -d {} -j MASQUERADE'.format(ip6, ip6))

      info("*** Starting publishers\n")
      cmd = "bash -c 'python temperature-simulator.py {} /admin/{}/attrs {};'"
      makeTerm(sensor1, title='', cmd=cmd.format(sensor1.name, DEVICE_ID, TOPIC))
      makeTerm(sensor2, title='', cmd=cmd.format(sensor1.name, DEVICE_ID, TOPIC))
      makeTerm(sensor3, title='', cmd=cmd.format(sensor1.name, DEVICE_ID, TOPIC))
      makeTerm(c1, title='controller', cmd="bash -c 'python controller-sub.py %s;'" % DEVICE_ID)

      info("*** Running CLI\n")
      CLI(net)

      info("*** Killing xterm\n")
      os.system('pkill -f \"xterm -title\"')

      info("*** Stopping network\n")
      net.stop()


  if __name__ == '__main__':
      setLogLevel('info')
      topology()

   
**temperature-simulator.py**:
    
.. code:: python

    #!/usr/bin/python

    """
    @author: Ramon Fontes
    @email: ramonreisfontes@gmail.com
    """

    import os
    import random
    import logging

    from time import sleep
    from sys import argv

    logging.basicConfig(level="INFO")


    i = 25
    node = argv[1]
    topic = argv[2]
    attribute = argv[3]
    attr = '{\"%s-%s\":' % (node, attribute)
    character = '}'
    sleep(10)
    while True:
        data = random.randint(i-5, i+5)
        i = data
        cmd = "mosquitto_pub -h 2002::1 -t {} -m \'{}{}{}\'"
        cmd = cmd.format(topic, attr, data, character)
        logging.info(cmd)
        os.system(cmd)
        sleep(5)


**controller-sub.py**:


.. code:: python

    #!/usr/bin/python

    """
    @author: Ramon Fontes
    @email: ramonreisfontes@gmail.com
    """

    import sys
    import logging
    from paho.mqtt.client import Client

    logging.basicConfig(level="INFO")
    DEVICE_ID = sys.argv[1]


    def on_connect(client, userdata, flags, rc):
        topic_list = ['/admin/{}/config'.format(DEVICE_ID)]
        for topic in topic_list:
            client.subscribe(topic)


    def on_message(client, userdata, msg):
        logging.info("Received " + str(msg.payload))


    client = Client()
    client.on_connect = on_connect
    client.on_message = on_message


    while True:
        try:
            client.connect("2002::1", 1883, 60)
            logging.info("Connected")
            break
        except:
            pass
    client.loop_forever()



Then, considering that you already have installed Dojot, open its dashboard in a browser of your choice.

.. warning::
    Please make sure that Dojot is working correctly and mosquitto server is not running!



Now, in the dashboard you need to do the following steps:

  - create a new template and an attribute called _sensor1-temperature_ with value type _integer_
  - now open the _device_ menu and add the template created previously
  - go to _flows_ and add an _event device_  as input flow. Select the device you have created and check both _actuation_ and _publication_ checkboxes as well. 
  - add a _change function_ and configure the set field as below:
  ```msg.payload.data.attrs.sta1-temperature```
  - in the "to" field, write:
  ```Alert message!```
  - finally, add a _multi actuate node_ and select _Specific Devices_ in the action field. Then select your device and configure the source field as below:
  ```msg.payload.data.attrs.sta1-temperature```
  - create a link between the _event device_ and the _change function_ and another link between the _change function_ and the _multi actuate node_
  - save the changes in the dashboard!
  - close the web browser

Now, let's run our network topology. To do so you need to run `lowpan.py` as below:

.. code:: console
    ~/mininet-wifi$ sudo python lowpan.py df7327 temperature


* **df7327**: device id created by Dojot - you have to set the id of a device
* **temperature**: topic


Four terminals should appear: one for each sensor and one for the controller


Open **the Dojot's dashboard** and you will be able to see some values being received by Dojot. You will also be able to see the "Alert message" being received by the controller's terminal.


**Activity 3 - Dojot (event trigger API)**

Now that you have run the script and you already know how MQTT works, as well as Dojot, in this activity you must create an event in Dojot so that different messages are received by the controller. For example, when the temperature of the sensors exceeds a certain value, a message informing about the high temperature should be sent to the controller. Also, think of some action that the controller can take based on the message received by Dojot.
