# Code on iPad using Raspberry Pi 4 and Code Server

## Setting up Raspberry Pi 4

After the OS is burnt, the SD card will be automatically ejected. Take it out and insert it again into your laptop's SD card reader because there are 2 more things we need to do before we proceed to the iPad.

1.  Write and empty `ssh` file to the Raspberry Pi.

    `$ touch /Volumes/boot/ssh`

2.  Write and edit `wpa_supplicant.conf` on the Raspberry Pi boot partition and add the content bellow to it. Replace the country code with your own and same for ssid and psk for your WiFi.

    `$ vim /Volumes/boot/wpa_supplicant.conf`

    ```
    country=NL
    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    update_config=1

    network={
        ssid="Name of your WiFi"
        psk="Password of you WiFi"
    }
    ```

3.  Now safely eject the SD card and insert it in you Raspberry Pi. Then connect the Raspberry Pi to the iPad using the USB-C cable.

4.  Now the Raspberry Pi 4 will boot up, so let it boot for a minute to be sure and fire up the Blink terminal. To ssh to the Pi via the terminal use:

    `$ ssh pi@raspberrypi.local`

    You will be prompted with a password request. Just type in `raspberry` and now you are connected to your Pi. You can change the password and the name of your Raspberry Pi using the `sudo raspi-config` command. You can find more about this online.

5.  Let's activate VNC on the Raspberry Pi 4. To do this enter the config screen using:

    `$ sudo raspi-config`

    Select Interfacing Options then VNC and now you are ready to connect to your Pi using VNC.
    Also while you are here make sure that you are in the correct time-zone by navigating to Localisation Options.

6.  Now let's update the Pi OS and install necessary packages by running:

    `$ sudo apt update`

    `$ sudo apt upgrade -y`

    `$ sudo apt install dnsmasq -y`

    `$ sudo apt install vim`

7.  Next step is to setup the Raspberry Pi as a USB-C gadget and it will show up as a ethernet device on the iPad.

    **7.1. Some basic config**

    `$ sudo echo "dtoverlay=dwc2" >> /boot/config.txt`

    `$ sudo echo "modules-load=dwc2" >> /boot/cmdline.txt`

    `$ sudo echo "libcomposite" >> /etc/modules`

    `$ sudo echo "denyinterfaces usb0" >> /etc/dhcpcd.conf`

    **NOTE:** If you get access denied to write on config.txt and the other files just use the command bellow. I know it is not the safest but only you will access this Pi ideally.

    `$ sudo chmod ugo+rwx /boot/config.txt`

    **7.2. Create the following files and add the configuration bellow to them:**

    `$ sudo vim /etc/dnsmasq.d/usb`

    ```
    interface=usb0
    dhcp-range=10.55.0.2,10.55.0.6,255.255.255.248,1h
    dhcp-option=3
    leasefile-ro
    ```

    `$ sudo vim /etc/network/interfaces.d/usb0`

    ```
    auto usb0
    allow-hotplug usb0
    iface usb0 inet static
        address 10.55.0.1
        netmask 255.255.255.248
    ```

    `$ sudo vim /root/usb.sh`

    ```
    #!/bin/bash
    cd /sys/kernel/config/usb_gadget/
    mkdir -p pi4
    cd pi4
    echo 0x1d6b > idVendor # Linux Foundation
    echo 0x0104 > idProduct # Multifunction Composite Gadget
    echo 0x0100 > bcdDevice # v1.0.0
    echo 0x0200 > bcdUSB # USB2
    echo 0xEF > bDeviceClass
    echo 0x02 > bDeviceSubClass
    echo 0x01 > bDeviceProtocol
    mkdir -p strings/0x409
    echo "fedcba9876543211" > strings/0x409/serialnumber
    echo "Ben Hardill" > strings/0x409/manufacturer
    echo "PI4 USB Device" > strings/0x409/product
    mkdir -p configs/c.1/strings/0x409
    echo "Config 1: ECM network" > configs/c.1/strings/0x409/configuration
    echo 250 > configs/c.1/MaxPower
    # Add functions here
    # see gadget configurations below
    # End functions
    mkdir -p functions/ecm.usb0
    HOST="00:dc:c8:f7:75:14" # "HostPC"
    SELF="00:dd:dc:eb:6d:a1" # "BadUSB"
    echo $HOST > functions/ecm.usb0/host_addr
    echo $SELF > functions/ecm.usb0/dev_addr
    ln -s functions/ecm.usb0 configs/c.1/
    udevadm settle -t 5 || :
    ls /sys/class/udc > UDC
    ifup usb0
    service dnsmasq restart
    ```

    Make `/root/usb.sh` executable:

    `$ chmod +x /root/usb.sh`

    Add `sh /root/usb.sh` to **_/etc/rc.local_** file before `exit 0` line.

    After reboot `$ sudo reboot` the Pi4 will show up as a ethernet device with an IP address of 10.55.0.1 and will assign the device you plug it into an IP address via DHCP. This means you can just ssh to pi@10.55.0.1 to start using it.

## Setting up Code-Server on Raspberry Pi 4

1. Install NodeJS

   To check the which type of processor you have (e.g. ARMv6, ARMv7 or ARMv8) run the following command:

   `$ uname -m`

   Go to [NodeJS download page](https://nodejs.org/en/download/) and copy the link address of the NodeJS for the type of processor you have. I have ARMv7 so I will run:

   `$ wget https://nodejs.org/dist/v14.16.1/node-v14.16.1-linux-armv7l.tar.gz`

   **NOTE:** If the extension of the file is tar.xz just change in the link to gz.

   `$ tar -xzf node-v14.16.1-linux-armv7l.tar.gz`

   `$ cd node-v14.16.1-linux-armv7l`

   `sudo cp -R * /usr/local/`

   Check if NodeJS and NPM were installed correctly:

   `$ node -v`

   `$ npm -v`

2. Install code-server

   2.1. Run the following command to install code-server. Be patient, it will take between 5 - 10 minutes, maybe even more.

   `$ curl -fsSL https://code-server.dev/install.sh | sh`

   2.2. Configure code-server to be make sure it can be accessed from your iPad's browser:

   `$ sudo mkdir /home/pi/.config/code-server`

   `$ sudo vim /home/pi/.config/code-server/config.yaml`

   Add the following to your config file:

   ```
   ---
   bind-addr: 0.0.0.0:8080
   auth: password
   password: <WHATEVER PASSWORD YOU WANT TO LOGIN TO CODE SERVER>
   cert: false
   ```

   2.3. Let's configure it as a service which will start when the Pi boots so that you don't always have to do it manually.

   `$ sudo vim /lib/systemd/system/code-server.service`

   Add the following to the service file:

   ```
   [Unit]
   Description=code-server
   After=network.target

   [Service]
   Type=exec
   Environment=PASSWORD=code-server-password
   ExecStart=/usr/bin/code-server --bind-addr 0.0.0.0:8080 --user-data-dir /var/lib/code-server --auth password
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```

   `$ sudo systemctl daemon-reload`

   `$ sudo systemctl start code-server`

   `$ sudo systemctl enable code-server`

   `$ sudo systemctl status code-server`

   To stop the service manually:

   `$ sudo systemctl stop code-server`

   Now everytime you boot the Pi, code-server will start automatically

## Access Code-Server from the iPad

On the iPad go to `http://10.55.0.1:8080` and login with the password you set above and that's it! Enjoy coding on your iPad!
