   # SETTING THE RPi5 AS USB GADGET AND SHARING THE INTERNET THROUGH USB

   I had a lot of problems configuring everything, so I hope that if anyone happens to find this, it will prove useful. This guide is for my future reference. Everything worked fine for me, but your mileage may vary.

   <hr/>
      
   > [!TIP]
   > Perform all the operations as a `root`, so you will not be forced to use `sudo` and authenticate all the time.

   > [!NOTE]
   > You can edit the files with nano, vi, vim, or any editor of your liking.
   >
   > I chose nano because vi was reading some funny inputs when I was navigating the files.

   <hr/>
   <br/>

   ## Update the system

   <ol>
   <li> Check for updates

   ```bash
   apt-get update -y
   ```

   </li>
   <li> Download and apply the updates

   ```basha
   pt-get upgrade -y
   ```

   </li>
   </ol>
   <br/>

   ## IF REQUIRED - UPDATE THE FIRMWARE, BEAR IN MIND IT IS RISKY

   <ol>
      <li> Run the firmware update

   ```bash
   rpi-update
   ```

      </li>
   </ol>
   <br/>

   ## Add kernel entries

   <ol>
   <li> Edit the <code>config.txt</code>
      <ul><li>
      
   ```bash
   nano /boot/firmware/config.txt
   ```

   </li>
   <li> Add this code at the end of the file

   ```bash
   [all]
   # allowing the usb devices to draw more current, useful for LCD touchscreen
   max_usb_current=1
   # good measure to enable HDMI hot-plugging
   hdmi_force_hotplug=1
   # enable the USB functionality
   dtoverlay=dwc2
   ```

   </li></ul>

   <li> Edit the <code>cmdline.txt</code>
      <ul><li>

   ```bash
   nano /boot/firmware/cmdline.txt
   ```

   </li>
   <li> Add this entry at the end of line

   ```bash
   modules-load=dwc2
   ```

   </li>
   
   </ul>
   </li>
   <br/>
   <hr/>

   > [!CAUTION]
   > DON'T ADD NEW LINES!!!
   >
   > Everything has to be in one line!

   <hr/>
   <br/>

   <li> Edit the list of modules
   <ul>

      <li>
      
   ```bash
   nano /etc/modules
   ```

   </li>
      <li> Add this at the end of file

   ```bash
   libcomposite
   ```

   </li>
   </ul>
   </li>
   </ol>
   <br/>

   ## Create new USB Ethernet interfaces

   Two new interfaces are created (usb0 & usb1) to include both ECM and RNDIS Ethernet devices, so it will work with Linux, macOS, and Windows without the need to install any extra drivers.

   <hr/>

   > [!IMPORTANT]
   > Don't forget to make it executable, otherwise it won't work!

   <hr/>
   <br/>
   <ol>

   <li> Create the config file

   ```bash
   nano /usr/local/sbin/usb-gadget.sh
   ```

   </li>
   <li> Paste the following script and save

   ```bash
   #!/bin/bash

   cd /sys/kernel/config/usb_gadget/
   mkdir -p display-pi
   cd display-pi
   echo 0x1d6b > idVendor # Linux Foundation
   echo 0x0104 > idProduct # Multifunction Composite Gadget
   echo 0x0103 > bcdDevice # v1.0.3
   echo 0x0320 > bcdUSB # USB2
   echo 2 > bDeviceClass
   mkdir -p strings/0x409
   echo "fedcba9876543213" > strings/0x409/serialnumber
   echo "Ben Hardill" > strings/0x409/manufacturer
   echo "Display-Pi USB Device" > strings/0x409/product
   mkdir -p configs/c.1/strings/0x409
   echo "CDC" > configs/c.1/strings/0x409/configuration
   echo 250 > configs/c.1/MaxPower
   echo 0x80 > configs/c.1/bmAttributes

   #ECM
   mkdir -p functions/ecm.usb0
   HOST="00:dc:c8:f7:75:15" # "HostPC"
   SELF="00:dd:dc:eb:6d:a1" # "BadUSB"
   echo $HOST > functions/ecm.usb0/host_addr
   echo $SELF > functions/ecm.usb0/dev_addr
   ln -s functions/ecm.usb0 configs/c.1/

   #
   mkdir -p configs/c.2
   echo 0x80 > configs/c.2/bmAttributes
   echo 0x250 > configs/c.2/MaxPower
   mkdir -p configs/c.2/strings/0x409
   echo "RNDIS" > configs/c.2/strings/0x409/configuration

   echo "1" > os_desc/use
   echo "0xcd" > os_desc/b_vendor_code
   echo "MSFT100" > os_desc/qw_sign

   mkdir -p functions/rndis.usb0
   HOST_R="00:dc:c8:f7:75:16"
   SELF_R="00:dd:dc:eb:6d:a2"
   echo $HOST_R > functions/rndis.usb0/dev_addr
   echo $SELF_R > functions/rndis.usb0/host_addr
   echo "RNDIS" >   functions/rndis.usb0/os_desc/interface.rndis/compatible_id
   echo "5162001" > functions/rndis.usb0/os_desc/interface.rndis/sub_compatible_id

   ln -s functions/rndis.usb0 configs/c.2
   ln -s configs/c.2 os_desc

   udevadm settle -t 5 || :
   ls /sys/class/udc > UDC

   sleep 5

   nmcli connection up bridge-br0
   nmcli connection up bridge-slave-usb0
   nmcli connection up bridge-slave-usb1
   sleep 5
   service dnsmasq restart
   ```

   </li>
   <li> Make the script executable

   ```bash
   chmod +x /usr/local/sbin/usb-gadget.sh
   ```

   </li>
   </ol>
   <br/>

   ## Create a systemd service and enable it

   <ol>

   <li> Create the file

   ```bash
   nano /lib/systemd/system/usbgadget.service
   ```

   </li>
   <li> Paste the following script and save

   ```bash
   [Unit]
   Description=My USB gadget
   After=network-online.target
   Wants=network-online.target
   #After=systemd-modules-load.service

   [Service]
   Type=oneshot
   RemainAfterExit=yes
   ExecStart=/usr/local/sbin/usb-gadget.sh

   [Install]
   WantedBy=sysinit.target
   ```

   </li>
   <li> Reload the services (not necessary)

   ```bash
   systemctl daemon-reload
   ```

   </li>
   <li> Enable the service

   ```bash
   systemctl enable usbgadget.service
   ```

   </li>
   </ol>
   <br/>

   ## Create a bridge interface

   We create this bridge to combine both the ECM and the RNDIS driver and share the same IP address

   <ol>

   <li> Create the main bridge

   ```bash
   nmcli con add type bridge ifname br0
   ```

   </li>
   <li> Create the slave interface for ECM

   ```bash
   nmcli con add type bridge-slave ifname usb0 master br0
   ```

   </li>
   <li> Create the slave interface for RNDIS

   ```bash
   nmcli con add type bridge-slave ifname usb1 master br0
   ```

   </li>
   <li> Configure the IP address of the USB gadget

   ```bash
   nmcli connection modify bridge-br0 ipv4.method manual ipv4.addresses 10.55.0.1/24
   ```

   </li>
   <br/>
   <hr/>

   > [!IMPORTANT]
   > IP ADDRES CAN BE CHANGED, JUST REMEMBER TO ALSO CHANGE IT IN DNSMASQ CONFIG LATER!

   <hr/>
   </ol>
   <br/>

   ## Install and configure the DNSMASQ

   <ol>

   <li> Install dnsmasq

   ```bash
   apt-get install dnsmasq
   ```

   </li>
   <li> Create the DNS config for interface <code>br0</code>

   ```bash
   nano /etc/dnsmasq.d/br0
   ```

   </li>
   <li> Paste the following script and save

   ```bash
   dhcp-authoritative
   dhcp-rapid-commit
   no-ping
   interface=br0
   dhcp-range=10.55.0.2,10.55.0.6,255.255.255.248,1h
   dhcp-option=3
   leasefile-ro
   ```

   </li>
   <br/>
   <hr/>

   > [!IMPORTANT]
   > IF YOU CHANGED THE IP IN THE PREVIOUS STEP, ADJUST THE RANGE ACCORDINGLY!

   <hr/>
   <br/>
   </ol>

   ## Reboot the raspberry and test

   <ol>

   <li> Restart RPi
   <ul>
      <li> If the raspberry is already powered by the USB cable connected to the PC

   ```bash
   systemctl reboot
   ```

   </li>
      <li> If the raspberry is connected to the power supply

   ```bash
   shutdown now
   ```

   then connect it to the PC using a USB cable.

   </li>
   </ul>
   </li>
   <li> Momentarily turn off the Wi-Fi on the PC to see if the USB connection is working

   </li>
   <li> Open a terminal and try ping
   <ul>
      <li>
      
   ```bash
   ping raspberrypi.local
   ```

   </li>
      <li>
      
   ```bash
   ping 10.55.0.1
   ```
   (or whatever IP you chose in the previous steps)

   </li>
   </ul>
   </li>
   <br/>
   <hr/>

   > [!NOTE]
   > I'm assuming here that the hostname of the raspberry is set to `raspberrypi`

   <hr/>
   <br/>
   <li> If ping worked fine, try SSH (remember to use a proper username, e.g. <code>pi</code>)
   <ul>
      <li>

   ```bash
   ssh pi@raspberrypi.local
   ```

   </li>
      <li>

   ```bash
   ssh pi@10.55.0.1
   ```

   (or whatever IP you chose in the previous steps)

   </li>
   </ul>
   </li>
   </ol>
   <br/>
   <br/>
   <br/>
   <br/>

   # Add routing from the RPi to the internet through a shared USB connection

   <br/>

   ## Windows - configured through control panel

   > [!WARNING]
   > DID NOT TEST, BUT WAS WORKING FINE WITH RPi4 AS USB GADGET SOME TIME AGO!

   <hr/>
   <br/>

   To enable Internet Connection Sharing in Windows 10, follow the steps below:

   <ol>

   <li> Press Windows key + X to open the Power User menu and select Network Connections. </li>

   <li> Right-click the network adapter with an Internet connection (Ethernet or wireless network adapter), then select Properties. </li>

   <li> Click Sharing. </li>

   <li> Put a check mark on Allow other network users to connect through this computer’s Internet connection. </li>

   <li> From the Home networking connection drop-down menu, select the adapter with internet connection. </li>

   <li> Click OK to finish. </li>

   </ol>
   <br/>

   ## Linux - oh boy...

   > [!IMPORTANT]
   > This setup is much more complicated, but I tested it several times, and it worked every time.
   >
   > This instruction assumes that both systems are managed by NetworkManager (Fedora 40 KDE and Raspberry Pi OS Bookworm in my case).

   > [!NOTE]
   > You can edit the files with nano, vi, vim, or any editor of your liking.
   >
   > I chose nano on both machines because vi was reading some funny inputs when I was navigating the files.

   <hr/>
   <br/>

   #### Linux PC

   <hr/>

   > [!TIP]
   > Perform all the operations as a `root`, so you will not be forced to use `sudo` and authenticate all the time.

   <hr/>
   <br/>
   <ol>

   <li> Create default config file for nftables

   ```bash
   nano /etc/nftables.conf
   ```

   </li>
   <li> Paste the following script and save

   ```bash
   #!/usr/bin/nft -f
   # vim:set ts=2 sw=2 et:

   # IPv4/IPv6 Simple & Safe firewall ruleset.
   # More examples in /usr/share/nftables/ and /usr/share/doc/nftables/examples/.

   destroy table inet filter
   table inet filter {
      chain input {
         type filter hook input priority filter
         policy drop

         ct state invalid drop comment "early drop of invalid connections"
         ct state { established, related } accept comment "allow tracked connections"
         iif lo accept comment "allow from loopback"
         ip protocol icmp accept comment "allow icmp"
         meta l4proto ipv6-icmp accept comment "allow icmp v6"
         tcp dport ssh accept comment "allow sshd"
         pkttype host limit rate 5/second counter reject with icmpx type admin-prohibited
         counter
      }
      chain forward {
         type filter hook forward priority filter
         policy drop
      }
   }
   ```

   </li>
   <li> Delete all of the rules

   ```bash
   nft flush ruleset
   ```

   </li>
   <li> Verify that the list is empty

   ```bash
   nft list ruleset
   ```

   </li>
   <li> Restore the default config from file you just created

   ```bash
   nft -f /etc/nftables.conf
   ```

   </li>
   <li> Verify that the rules were added

   ```bash
   nft list ruleset
   ```

   </li>
   <li> Create a new table for NAT

   ```bash
   nft add table inet nat
   ```

   </li>
   <li> Create "postrouting chain"

   ```bash
   nft add chain inet nat postrouting '{ type nat hook postrouting priority 100 ; }'
   ```

   </li>
   <li> Check the names of the interfaces

   ```bash
   ifconfig
   ```

   </li>
   <br/>
   <hr/>

   > [!NOTE]
   > Change your interfaces accordingly
   >
   > e.g. in my case `enp7s0f3u2` is the interface of the raspberry connected via USB;
   >
   > `wlo1` is the Wi-Fi interface with internet access which I want to share.

   <hr/>
   <br/>

   <li> Masquerade the enp7s0f3u2 addresses for wlo1

   ```bash
   nft add rule inet nat postrouting oifname wlo1 masquerade
   ```

   </li>
   <li> Allow forwarding NAT traffic (default policy of the 'filter' table's 'forward' chain is set to 'drop'):
   <ul>
      <li>
      
   ```bash
   nft add rule inet filter forward ct state related,established accept
   ```

   </li>
      <li>
      
   ```bash
   nft add rule inet filter forward iifname enp7s0f3u2 oifname wlo1 accept
   ```

   </li>
   </ul>
   </li>
   <li> Backup the config, read the backup and verify the ruleset

   ```bash
   nft -s list ruleset >> /etc/my_nftables.conf && nft flush ruleset && nft -f /etc/my_nftables.conf && nft list ruleset
   ```

   </li>
   <li> Open the <code>sysctl.conf</code> file

   ```bash
   nano /etc/sysctl.conf
   ```

   </li>
<br/>
<hr/>

> [!IMPORTANT]
> This is the most important sequence of steps. Without it, all our work will be pointless.
> 
> We want to enable IP forwarding from Pi to the Internet.
>
> This setting needs to be persistent, so it won't be reset after a reboot.

<hr/>
<br/>
   <li> Paste the following code at the end of the file (only the 1st line is really required)

   ```bash
   net.ipv4.ip_forward=1
   net.ipv4.conf.all.forwarding=1
   net.ipv6.conf.all.forwarding=1
   ```

   </li>
   <li> Apply new configuration immediately

   ```bash
   sysctl -p
   ```

   </li>
   <li> Verify that forwarding is enabled (set to "1")

   ```bash
   sysctl net.ipv4.ip_forward
   ```

   </li>
   <li> Restart the NetworkManager service

   ```bash
   systemctl restart NetworkManager
   ```

   </li>
   <li> Chceck if the NetworkManager service is working

   ```bash
   systemctl status NetworkManager
   ```

   </li>
   </ol>
   <br/>

   #### Raspberry Pi

   <hr/>

   > [!TIP]
   > Perform all the operations as a `root`, so you will not be forced to use `sudo` and authenticate all the time.

   <hr/>
   <br/>
   <ol>

   <li> Add a default route to PC

   ```bash
   nmcli connection modify bridge-br0 ipv4.routes "0.0.0.0/0 10.55.0.6"
   ```

   which works the same as this command

   ```bash
   ip route add default via 10.55.0.6 dev br0
   ```

   </li>
   <br/>
   <hr/>

   > [!WARNING]
   > `bridge-br0` is set by the `usbgadget.service`,
   >
   > `10.55.0.6` is the IP my laptop is getting, you may be forced to change that.

   > [!NOTE]
   > The main difference in nmcli command (which is required in our case) is the changes are persistent, so this will work after the reboot.

   > [!TIP]
   > Routes are separated by commas. More info [here](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-static-routes_configuring-and-managing-networking#configuring-a-static-route-by-using-nmcli_configuring-static-routes).

   <hr/>
   <br/>

   <li> Verify the rules

   ```bash
   ip r
   ```

   </li>
   <li> Add public DNS from Google or change it to your needs (I used the private DNS, which is running on my PiHole)

   ```bash
   nmcli connection modify bridge-br0 ipv4.dns "8.8.8.8 8.8.4.4"
   ```

   </li>
   <br/>
   <hr/>

   > [!TIP]
   > IP addresses are separated by space. More info [here](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/networking_guide/sec-configuring_ip_networking_with_nmcli#sec-Adding_and_Configuring_a_Static_Ethernet_Connection_with_nmcli).

   <hr/>
   <br/>

   <li> Restart the service

   ```bash
   service NetworkManager restart
   ```

   </li>
   <li> Enable the connection (not necessary)

   ```bash
   nmcli connection up bridge-br0
   ```

   </li>
   <li> Verify the rules

   ```bash
   cat /etc/resolv.conf
   ```

   </li>
   </ol>
   <br/>
   <br/>

   ## SOURCES

   [https://blog.hardill.me.uk/2023/12/23/pi5-usb -c-gadget/](https://blog.hardill.me.uk/2023/12/23/pi5-usb-c-gadget/)

   [https://forums.raspberrypi.com/viewtopic.php?t=358573](https://forums.raspberrypi.com/viewtopic.php?t=358573)

   [https://wiki.archlinux.org/title/Internet_sharing](https://wiki.archlinux.org/title/Internet_sharing)

   [https://wiki.nftables.org/wiki-nftables/index.php/Performing*Network_Address_Translation*(NAT)](<https://wiki.nftables.org/wiki-nftables/index.php/Performing_Network_Address_Translation_(NAT)>)

   [https://wiki.archlinux.org/title/Nftables](https://wiki.archlinux.org/title/Nftables)

   [https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/networking_guide/sec-configuring_ip_networking_with_nmcli#sec-Adding_and_Configuring_a_Static_Ethernet_Connection_with_nmcli](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/networking_guide/sec-configuring_ip_networking_with_nmcli#sec-Adding_and_Configuring_a_Static_Ethernet_Connection_with_nmcli)

   [https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-static-routes_configuring-and-managing-networking#configuring-a-static-route-by-using-nmcli_configuring-static-routes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-static-routes_configuring-and-managing-networking#configuring-a-static-route-by-using-nmcli_configuring-static-routes)    
