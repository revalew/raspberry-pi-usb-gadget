# SETTING THE RPi5 AS USB GADGET AND SHARING THE INTERNET THROUGH USB

I had a lot of problems configuring everything, so I hope that if anyone happens to find this, it will prove useful. This guide is for my future reference. Everything worked fine for me, but your mileage may vary.

> [!TIP]
> Perform all the operations as a `root`, so you will not be forced to use `sudo` and authenticate all the time.

> [!NOTE]
> You can edit the files with nano, vi, vim, or any editor of your liking.
>
> I chose nano because vi was reading some funny inputs when I was navigating the files.

## Update the system

`apt-get update -y`

`apt-get upgrade -y`

## IF REQUIRED - UPDATE THE FIRMWARE, BEAR IN MIND IT IS RISKY

`rpi-update`

## Add kernel entries

1. Edit the `config.txt`

   - `nano /boot/firmware/config.txt`

   - Add this code at the end of the file

     ```bash
     [all]
     # allowing the usb devices to draw more current, useful for LCD touchscreen
     max_usb_current=1
     # good measure to enable HDMI hot-plugging
     hdmi_force_hotplug=1
     # enable the USB functionality
     dtoverlay=dwc2
     ```

2. Edit the `cmdline.txt`

   - `nano /boot/firmware/cmdline.txt`

   - Add this entry at the end of line

     > [!CAUTION]
     > DON'T ADD NEW LINES!!!
     >
     > Everything has to be in one line!

     `modules-load=dwc2`

3. Edit the list of modules

   - `nano /etc/modules`

   - Add this at the end of file

     `libcomposite`

## Create new USB Ethernet interfaces

Two new interfaces are created (usb0 & usb1) to include both ECM and RNDIS Ethernet devices, so it will work with Linux, macOS, and Windows without the need to install any extra drivers.

> [!IMPORTANT]
> Don't forget to make it executable, otherwise it won't work!

1. Create the config file

   `nano /usr/local/sbin/usb-gadget.sh`

2. Paste the following script and save

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

3. Make the script executable

   `chmod +x /usr/local/sbin/usb-gadget.sh`

## Create a systemd service and enable it

1. Create the file

   `nano /lib/systemd/system/usbgadget.service`

2. Paste the following script and save

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

3. Reload the services (not necessary)

   `systemctl daemon-reload`

4. Enable the service

   `systemctl enable usbgadget.service`

## Create a bridge interface

We create this bridge to combine both the ECM and the RNDIS driver and share the same IP address

1. Create the main bridge

   `nmcli con add type bridge ifname br0`

2. Create the slave interface for ECM

   `nmcli con add type bridge-slave ifname usb0 master br0`

3. Create the slave interface for RNDIS

   `nmcli con add type bridge-slave ifname usb1 master br0`

4. Configure the IP address of the USB gadget

   > [!IMPORTANT]
   > IP ADDRES CAN BE CHANGED, JUST REMEMBER TO ALSO CHANGE IT IN DNSMASQ CONFIG LATER!

   `nmcli connection modify bridge-br0 ipv4.method manual ipv4.addresses 10.55.0.1/24`

## INSTALL AND CONFIGURE THE DNSMASQ

1. Install dnsmasq

   `apt-get install dnsmasq`

2. Create the DNS config for interface `br0`

   `nano /etc/dnsmasq.d/br0`

3. Paste the following script and save

   > [!IMPORTANT]
   > IF YOU CHANGED THE IP IN THE PREVIOUS STEP, ADJUST THE RANGE ACCORDINGLY!

   ```bash
   dhcp-authoritative
   dhcp-rapid-commit
   no-ping
   interface=br0
   dhcp-range=10.55.0.2,10.55.0.6,255.255.255.248,1h
   dhcp-option=3
   leasefile-ro
   ```

## REBOOT THE RASPBERRY AND TEST

1. Restart RPi

   - If the raspberry is already powered by the USB cable connected to the PC

     `systemctl reboot`.

   - If the raspberry is connected to the power supply

     `shutdown now`

     then connect it to the PC using a USB cable.

2. Momentarily turn off the Wi-Fi on the PC to see if the USB connection is working

3. Open a terminal and try ping

   - `ping raspberrypi.local`

   - `ping 10.55.0.1` (or whatever IP you chose in the previous steps)

4. If ping worked fine, try SSH (remember to use a proper username, e.g. `pi`)

   - `ssh pi@raspberrypi.local`

   - `ssh pi@10.55.0.1` (or whatever IP you chose in the previous steps)

# Add routing from the RPi to the internet through a shared USB connection

## Windows - configured through control panel.

> [!WARNING]
> DID NOT TEST, BUT WAS WORKING FINE WITH RPi4 AS USB GADGET SOME TIME AGO!

To enable Internet Connection Sharing in Windows 10, follow the steps below:

1. Press Windows key + X to open the Power User menu and select Network Connections.

2. Right-click the network adapter with an Internet connection (Ethernet or wireless network adapter), then select Properties.

3. Click Sharing.

4. Put a check mark on Allow other network users to connect through this computer’s Internet connection.

5. From the Home networking connection drop-down menu, select the adapter with internet connection.

6. Click OK to finish.

## Linux - oh boy...

> [!IMPORTANT]
> This setup is much more complicated, but I tested it several times, and it worked every time.

> [!IMPORTANT]
> This instruction assumes that both systems are managed by NetworkManager (Fedora 40 KDE and Raspberry Pi OS Bookworm in my case).

> [!NOTE]
> You can edit the files with nano, vi, vim, or any editor of your liking.
>
> I chose nano on both machines because vi was reading some funny inputs when I was navigating the files.

1.  Linux machine:

    > [!TIP]
    > Perform all the operations as a `root`, so you will not be forced to use `sudo` and authenticate all the time.

    - Create default config file for nftables

      `nano /etc/nftables.conf`

    - Paste the following script and save

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

    - Delete all of the rules

      `nft flush ruleset`

    - Verify that the list is empty

      `nft list ruleset`

    - Restore the default config from file you just created

      `nft -f /etc/nftables.conf`

    - Verify that the rules were added

      `nft list ruleset`

    - Create a new table for NAT

      `nft add table inet nat`

    - Create "postrouting chain"

      `nft add chain inet nat postrouting '{ type nat hook postrouting priority 100 ; }'`

    - Check the names of the interfaces

      `ifconfig`

      > [!NOTE]
      > e.g. in my case `enp7s0f3u2` is the interface of the raspberry connected via USB;
      >
      > `wlo1` is the Wi-Fi interface with internet access which I want to share.

    - Masquerade the enp7s0f3u2 addresses for wlo1

      `nft add rule inet nat postrouting oifname wlo1 masquerade`

      > [!NOTE]
      > `enp7s0f3u2` is the interface of the raspberry;
      >
      > `wlo1` is the interface with internet access.

    - Allow forwarding NAT traffic (default policy of the 'filter' table's 'forward' chain is set to 'drop'):

      - `nft add rule inet filter forward ct state related,established accept`

      - `nft add rule inet filter forward iifname enp7s0f3u2 oifname wlo1 accept`

      > [!NOTE]
      > AGAIN `enp7s0f3u2` is the interface of the raspberry;
      >
      > `wlo1` is the interface with internet access.

    - Backup the config, read the backup and verify the ruleset

      `nft -s list ruleset >> /etc/my_nftables.conf && nft flush ruleset && nft -f /etc/my_nftables.conf && nft list ruleset`

2.  On RPi:

    > [!TIP]
    > Perform all the operations as a `root`, so you will not be forced to use `sudo` and authenticate all the time.

    - Set up the routing:

      > [!WARNING]
      > `bridge-br0` is set by the `usbgadget.service`,
      >
      > `10.55.0.6` is the IP my laptop is getting, you may be forced to change that.

      - `nmcli connection modify bridge-br0 ipv4.routes "0.0.0.0/0 10.55.0.6"`

        which works the same as this command

        `ip route add default via 10.55.0.6 dev br0`

        > [!NOTE]
        > The main difference in nmcli command (which is required in our case) is the changes are persistent, so this will work after the reboot.

        > [!TIP]
        > Routes are separated by commas. More info [here](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-static-routes_configuring-and-managing-networking#configuring-a-static-route-by-using-nmcli_configuring-static-routes).

      - Verify the rules

        `ip r`

    - Set up the DNS:

      - Add public DNS from Google or change it to your needs (I used the private DNS, which is running on my PiHole)

        `nmcli connection modify bridge-br0 ipv4.dns "8.8.8.8 8.8.4.4"`

        > [!TIP]
        > IP addresses are separated by space.

      - Restart the service

        `service NetworkManager restart`

      - Enable the connection (not necessary)

        `nmcli connection up bridge-br0`

      - Verify the rules

        `cat /etc/resolv.conf`

## BASED ON:

[https://blog.hardill.me.uk/2023/12/23/pi5-usb -c-gadget/](https://blog.hardill.me.uk/2023/12/23/pi5-usb-c-gadget/)

[https://forums.raspberrypi.com/viewtopic.php?t=358573](https://forums.raspberrypi.com/viewtopic.php?t=358573)

[https://wiki.archlinux.org/title/Internet_sharing](https://wiki.archlinux.org/title/Internet_sharing)

[https://wiki.nftables.org/wiki-nftables/index.php/Performing_Network_Address_Translation_(NAT)](https://wiki.nftables.org/wiki-nftables/index.php/Performing_Network_Address_Translation_(NAT))

[https://wiki.archlinux.org/title/Nftables](https://wiki.archlinux.org/title/Nftables)
