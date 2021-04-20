************
Key Reinstallation Attacks
************

Intro
-------------

.. Note::
  **Inspired in https://www.krackattacks.com/**
  
  **Requirements:** 
  
  - Mininet-WiFi - https://github.com/intrig-unicamp/mininet-wifi
  - Hostap - http://w1.fi/hostap.git (commit df949062017bacae8095edeb73647ef97e7566bc )
  - libnl-route-3-dev
  
  
The result of this demo will be quite similar to the video below:

.. raw:: html

    <div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%; height: auto;">
        <iframe src="//www.youtube.com/embed/XbUxH5zQPTc" frameborder="0" allowfullscreen style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;"></iframe>
    </div>


Preparing the environment
-------------

.. code:: console

   # creating workind directory
   ~$ mkdir working_dir && cd working_dir
   
   # installing dependencies
   ~/working_dir$ sudo apt install libnl-route-3-dev
   
   # configuring hostap
   ~/working_dir$ git clone http://w1.fi/hostap.git
   ~/working_dir$ cd hostap && git reset df949062017
   ~/working_dir/hostap$ cd wpa_supplicant/
   ~/working_dir/hostap/wpa_supplicant$ cp defconfig .config
   ~/working_dir/hostap/wpa_supplicant$ sudo make install
   ~/working_dir/hostap/wpa_supplicant$ cd ../hostapd
   
   # uncomment the line with CONFIG_IEEE80211R=y in defconfig
   ~/working_dir/hostap/hostapd$ cp defconfig .config
   ~/working_dir/hostap/hostapd$ sudo make install
   
   ~/working_dir/hostap/hostapd$ cd ~/working_dir
   ~/working_dir$ git clone https://github.com/ramonfontes/krackattack
   ~/working_dir$ cd krackattack && sudo ./pysetup.sh
   

First of all trying to identify the network topology that will be generated through the code below:

.. code:: python

    #!/usr/bin/python

    """This code tests if APs are affected by CVE-2017-13082 (KRACK attack) and
    determine whether an implementation is vulnerable to attacks."""

    __author__ = "Ramon Fontes"
    __credits__ = ["https://github.com/vanhoefm/krackattacks-test-ap-ft"]

    from time import sleep

    from mininet.log import setLogLevel, info
    from mininet.term import makeTerm
    from mn_wifi.net import MininetWithControlWNet, Mininet_wifi
    from mn_wifi.cli import CLI
    from mn_wifi.link import wmediumd
    from mn_wifi.wmediumdConnector import interference


    def topology():

        "Create a network."
        net = Mininet_wifi(link=wmediumd, wmediumd_mode=interference)

        info("*** Creating nodes\n")
        sta1 = net.addStation('sta1', ip='10.0.0.1/8', position='50,0,0',
                              encrypt='wpa2')
        ap1 = net.addStation('ap1', mac='02:00:00:00:01:00',
                             ip='10.0.0.101/8', position='10,30,0')
        ap2 = net.addStation('ap2', mac='02:00:00:00:02:00',
                             ip='10.0.0.102/8', position='100,30,0')

        info("*** Configuring Propagation Model\n")
        net.setPropagationModel(model="logDistance", exp=3.5)

        info("*** Configuring wifi nodes\n")
        net.configureWifiNodes()

        ap1.setMasterMode(intf='ap1-wlan0', ssid='handover', channel='1',
                          ieee80211r=True, bssid_list=[['ap2']], mobility_domain='a1b2',
                          passwd='123456789a', encrypt='wpa2')
        ap2.setMasterMode(intf='ap2-wlan0', ssid='handover', channel='6',
                          ieee80211r=True, bssid_list=[['ap1']], mobility_domain='a1b2',
                          passwd='123456789a', encrypt='wpa2')

        info("*** Plotting Graph\n")
        net.plotGraph(min_x=-100, min_y=-100, max_x=200, max_y=200)

        info("*** Starting network\n")
        net.build()

        sta1.cmd("iw dev sta1-wlan0 interface add mon0 type monitor")
        sta1.cmd("ifconfig mon0 up")

        sleep(10)
        makeTerm(sta1, title='Scanning', cmd="bash -c 'iw dev sta1-wlan0 scan;'")
        makeTerm(sta1, title='KrackAttack', cmd="bash -c 'cd krackattack && python krack-ft-test.py;'")

        info("*** Running CLI\n")
        CLI(net)

        info("*** Stopping network\n")
        net.stop()


    if __name__ == '__main__':
        setLogLevel('info')
        topology()



So considering that you have named the code above as ```krack-attack```, run it as follows:

.. code:: console

    cd ~/working_dir
    ~/working_dir$ sudo python krack-attack.py
    

You should see now two terminals and you can use the Mininet-WiFi CLI to roam between ```ap1``` and ```ap2```:
    
.. code:: console

    mininet-wifi> sta1 wpa_cli -i sta1-wlan0 roam 02:00:00:00:01:00
    mininet-wifi> sta1 wpa_cli -i sta1-wlan0 roam 02:00:00:00:02:00
    
And finally you can see the vulnerability after pinging to ```ap2```.
    
.. code:: console    
    
    mininet-wifi> sta1 ping 10.0.0.102
