# CUBE 1V1-EU (Smarthub)

This hub is based on a custom carrier board with a Raspberry pi CM1. The carrier board has a neat micro usb that we can use to put the board into boot mode, making the pi show up as a USB mass storage device and letting us both read and write to the filesystem.

This guide will be based on [this rooting guide by Home Assistant community member Dan333.](https://community.home-assistant.io/t/how-to-gain-root-to-futurehome-hub-activate-z-wave-js-ui-with-futurehome-app-as-a-fallback/839340)

### Prerequisites
* Linux host system, to run rpiboot and manipulate the rootfs
* Micro-USB Cable (With data pins)

---

#### Step 1 - **Disassemble the smarthub**
The hub is opened by unscrewing 5 screws on the back of the unit.

<img src="CUBE 1V1-EU/assets/hub_1_back.jpg" alt="Hub" height="300">

Removing the circuit board we find a few additional USB ports that are hidden behind the enclosure. However, most importantly on the top of the board we have the CM1 and to the right we have the Micro-USB port.

<img src="CUBE 1V1-EU/assets/hub_1_board_top.jpg" alt="Hub" height="300">
<img src="CUBE 1V1-EU/assets/hub_1_board_bottom.jpg" alt="Hub" height="300">

#### Step 2 - **Set up rpiboot**
Many distros can be used for this project, but I would recommend [Ubuntu Desktop](https://ubuntu.com/download/desktop) here for simplicity. The default Ubuntu packages repo has rpiboot, letting you install it simply by running
```sudo apt install rpiboot```. Alternative installation methods can be found [here](https://github.com/raspberrypi/usbboot).


#### Step 3 - **Put the CM1 into boot mode**
First connect the Micro-USB Cable between the host and hub, and then you connect the power supply (DC connector). Normal boot should light a LED on the board after a few seconds, but if this LED stays off then it is likely to be in the correct mode.
Now run ```rpiboot``` and you should see some progress in the terminal once it connects. Once it finishes two new partitions will show on your machine "boot" and "rootfs". In Ubuntu desktop this is very easy to see as the partitions will pop up in the sidebar.

I found that many of my cables were charge only cables, make sure your cable supports data transfer.

#### Step 4 - **Write the necessary changes to rootfs**
As the script by Dan333 works very well I will only be rewriting it to use the Ubuntu default mount points.

First you mount your drive, in Ubuntu Desktop this is performed automatically when you click the device "rootfs" in your sidebar.
Once the device is mounted it will likely mount to /media/ubuntu/rootfs, which is what this script assumes.

First we generate a SSH key that is stored in plaintext. Remove the -N parameter if you want to protect it with a password.

```
ssh-keygen -f ~/.ssh/key -N ""
```
Add symlinks and firewall rules to enable SSH
```
sudo ln -s /lib/systemd/system/ssh.service /media/ubuntu/rootfs/etc/systemd/system/multi-user.target.wants/ssh.service
sudo sed -i '/^[[:space:]]*exit 0/i iptables -I INPUT -p tcp --dport 22 -j ACCEPT' /media/ubuntu/rootfs/etc/rc.local
```
Upload the publickey to the hub, assumes no preexisting keys
```
sudo mkdir -p /media/ubuntu/rootfs/root/.ssh
sudo cp ~/.ssh/key.pub /media/ubuntu/rootfs/root/.ssh/authorized_keys
sudo chmod 700 /media/ubuntu/rootfs/root/.ssh
sudo chmod 600 /media/ubuntu/rootfs/root/.ssh/authorized_keys
sudo chown -R 0:0 /media/ubuntu/rootfs/root/.ssh
```
Then unmount
```
sudo umount /media/ubuntu/rootfs
```

#### Step 5 - **Root!**
Now you can disconnect the hub from micro-usb, unplug the DC power supply and plug it back in. The hub should now boot into normal mode with the LED active. Hopefully if everything has been done correctly you will now be able to connect to it and log in to the root account. If your connection is link-local the hostname ```cube.local``` could work. Otherwise you need to find the IP by checking ARP tables, scanning your network or through your router.

---

I have only had limited time to experiment with root access to the device, but here is what I have found so far:

The device has an advanced local UI that futurehome calls [Thingsplex/FIMPUI](https://support.futurehome.no/hc/en-no/articles/360033271432-Thingsplex-Advanced-UI-version-13-0). FIMP is short for Futurehome IoT Messaging Protocol. It most likely is possible to install directly on the device. Maybe through the ```fimp-upgrade``` binary installed on the device? Otherwise the documentation states to install it through the app, which might be possible as long as you are inside of the 30 day free period.

I have not explored this yet as my second-hand device already had FIMP installed. Instead I had to reset it, which was surprisingly simple.

In order to reset you navigate to ```/opt/fimpui```, and if the folder ```/opt/fimpui/data``` exists then delete all contents inside of it and copy ```/opt/fimpui/defaults/config.json``` to ```/opt/fimpui/data/config.json```. Run ```systemctl restart fimpui``` to restart the service.

The Thingiplex/FIMPUI web interface can now be viewed at port 8081 in a browser. The design of this interface is not great, but functionally it seems to work.

Here is a zigbee smart plug that is paired from Thingsplex, appearing in the app. This demonstrates that devices should be available through this local interface.

<img src="CUBE 1V1-EU/assets/zigbee_adapters.png" alt="Thingsplex" height="300">
<img src="CUBE 1V1-EU/assets/futurehome_app.jpg" alt="Futurehome app" height="300">

---

### Future improvements
I want to fully disable updates to the hub, we could directly block internet access but some integrations might require internet.
One example is the [tibber](https://github.com/tskaard/fh-tibber/) integration. It appears to interface directly with the Tibber API, however the access token might be generated through the app? Integrations seem to be possible to install through local Thingsplex as well, and parameters such as tokens can be configured there.

This is very interesting, but I still assume these talk to futurehome servers for the installation process.

<img src="CUBE 1V1-EU/assets/tibber.png" alt="Tibber integration" height="700">
