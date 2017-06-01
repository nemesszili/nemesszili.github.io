---
layout: post
title: "Double trouble"
categories: tech
tags: [thesis, config, rpi]
image:
  feature: pi.jpg
  teaser: pi.jpg
  credit:
  creditlink: ""
---

While working on my thesis, I came across a specific problem regarding the compatibility of a WiFi interface (TP-LINK TL-WN725N) and Raspberry Pi 3 Model B.
If you're not interested in the train of thought, skip to the **Summary**.

### The Pi and the USB WiFi Interface

I hope there is no need to introduce the most formidable pocket PC of our times: the one and only Raspberry Pi. It does come in various models, but since we had lots of
options for development in the project's infancy, we went for the best bang for the buck: Raspberry Pi 3 Model B. Take note that this is the first edition of this pocket
wonder that actually has a built-in WiFi interface and I was very glad for having this feature.

There are dozens of USB WiFi interfaces out there, so why TP-LINK TL-WN725N? Simple: its form factor is great and it was the cheapest I could find at the local retailer.

### Specific needs

It may be common that your Pi-based application needs wireless Internet access, but I'm fairly certain that it's not everyday you need two interfaces operating at the same time.
The built-in one needs to switch between client and AP modes, depending on the Pi's configuration state. To complicate things further, this extra interface of yours 
constantly needs to be in Access Point mode so your remote NodeMCU based sensors can have access to your Pi. No matter how you assign these roles, both interfaces will
need to enter Access Point mode at some point of this application.

### First attempt

I have had some success with NetworkManager (_**nmcli**_) for switching the state of the built-in interface when I started tinkering with the second interface. I naively thought this thing will work just the same. Quickly fired up the Pi, accessed its terminal via serial. _Note: I can't SSH to it because the session will suddenly end when **nmcli** tries to activate a connection - weird bug if you ask me..._

After flashing the latest image (_I recommend [Win32DiskImager](https://sourceforge.net/projects/win32diskimager/) - Windows only - or [Etcher](https://etcher.io) - cross-platform_), don't forget to add the following line to _/boot/config.txt_:

```
enable_uart=1
```

In Windows, use _Putty_ to access the Pi's terminal via serial. In Linux, screen is the way to go:

```
screen /dev/ttyUSB0 115200
```

Let's check if our interface is recognised by the Pi:

```
lsusb
```

![lsusb output](/images/2017-05-29-Double-trouble/lsusb_marked.png)

```
ifconfig
```

![ifconfig output](/images/2017-05-29-Double-trouble/ifconfig_marked.png)

The outputs seem alright. The first one shows that there is indeed a new USB device plugged in, while _ifconfig_ tells us that there is a new wireless interface available.
You can easily tell which is which, since the built-in one's MAC address begins with _b8:27:eb_. 

So let's make an Access Point, shall we? 

First, it's good to know that NetworkManager can only manage interfaces that are not present in _/etc/network/interfaces_. Before we can proceed, let's edit the file
and delete all entries except for the loopback (lo).

```shell
sudo nano /etc/network/interfaces
```

After editing, your file should look like this:

```
source-directory /etc/network/interfaces.d

auto lo
iface lo inet loopback
```

Now, let's install NetworkManager (you'll need internet access for this step):

```shell
sudo apt-get update
sudo apt-get install -y network-manager
```

Then configure a connection for _wlan1_:

```shell
sudo nmcli c add type wifi ifname wlan1 con-name hotspot autoconnect no ssid Pi_AP
sudo nmcli connection modify hotspot 802-11-wireless.mode ap 802-11-wireless.band bg ipv4.method shared
sudo nmcli connection modify hotspot wifi-sec.key-mgmt wpa-psk
sudo nmcli connection modify hotspot wifi-sec.psk "abcdefgh"
```

So now we have a connection configured for interface wlan1 named _hotspot_. When activated, it should create an access point with _Pi\_AP_ as SSID and some passphrase. 

Time to fire it up!

```shell
sudo nmcli c up hotspot
```

![Very specific...](/images/2017-05-29-Double-trouble/error2.png)

Okay, I guess I'll come back later then...

### Second attempt

So I figure life is just way too short to dig through Raspbian's syslogs to find out why _nmcli_ refuses to cooperate. Plan B goes like this: instead of relying on _nmcli_ 
to handle this interface, I turn directly to _**hostapd**_ to address the issue. The rationale behind this? I found the official [guide](https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md) on configuring a hotspot for the Pi and it uses _hostapd_.
This guide also configures a DHCP server for your hotspot, which I dearly need in my project.
I do expect that an officially endorsed method will show some results after the previous attempt... 

Before we begin, let's check which interfaces are managed by _nmcli_:

```
nmcli d
```

![nmcli d](/images/2017-05-29-Double-trouble/nmcli_d.png)

Seems about right. Now, let's follow the aforementioned [guide](https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md) to the letter.
If you do everything right (swapping _wlan0_ for _wlan1_), you should have your access point up and running.

```shell
sudo hostapd /etc/hostapd/hostapd.conf
```
![nmcli d](/images/2017-05-29-Double-trouble/hostapd_fail.png)

...or do you?

### Second attempt reloaded - slightly going mad

I was starting to get frustrated with my inability to make this thing work. Luckily, I stumbled across the [blog entry](https://jenssegers.com/43/realtek-rtl8188-based-access-point-on-raspberry-pi) of Jens Segers. So what we need to do now is remove the original _hostapd_,
install the one provided by Realtek and fine-tuned by Jens. Thing is, when you run `sudo make install`, the script will overwrite the previously edited 
_/etc/hostapd/hostapd.conf_ and you'll need to edit it again.

A keen eye will notice that Jens's _hostapd.conf_ contains a different driver entry than in the official guide: `driver=rtl871xdrv` - _this bug cost me another 2 hours..._

Follow the steps in Jens's entry and edit _hostapd.conf_ accordingly!

Now, let's restart _hostapd_:

```shell
sudo systemctl daemon-reload
sudo service hostapd restart
```

Seems like my phone can connect without a hassle. At last!

![phone](/images/2017-05-29-Double-trouble/phone.png)

### Delusion of victory

I was so happy for achieving the goal for the day, I haven't even thought of integrating this solution in the actual version of the project. Thing is, when
I tried to do so the day after, I had a mini heart attack when the built-in interface (_wlan0_) couldn't create a hotspot via nmcli. **A feature that worked perfectly before.**

![failed](/images/2017-05-29-Double-trouble/failed2.png)

_If only these error messages were more verbose..._

### Pyrrhic victory

Since I couldn't get any useful information out of _NetworkManager_, my last resort were the dreaded syslogs. After messing around for an hour or two to extract some useful logs
from the Pi and analyze them on my workstation, I found something quite useful:

```
May 13 14:04:52 raspberrypi NetworkManager[21128]: <info> Starting dnsmasq...
May 13 14:04:52 raspberrypi NetworkManager[21128]: <info> (wlan0): device state change: ip-config -> ip-check (reason 'none') [70 80 0]
May 13 14:04:52 raspberrypi NetworkManager[21128]: <info> Activation (wlan0) Stage 5 of 5 (IPv4 Commit) complete.
May 13 14:04:52 raspberrypi dnsmasq[25143]: failed to bind DHCP server socket: Address already in use
May 13 14:04:52 raspberrypi dnsmasq[25143]: FAILED to start up
May 13 14:04:52 raspberrypi NetworkManager[21128]: dnsmasq: failed to bind DHCP server socket: Address already in use
May 13 14:04:52 raspberrypi NetworkManager[21128]: <warn> dnsmasq exited with error: Network access problem (address in use; permissions; etc) (2)
```

I remembered that while installing _NetworkManager_, there was an entry for dnsmasq-base, which made me think that perhaps it has its own instance and that's what causes the
address conflict. So I figured, if the instance needed by _wlan1_ goes down while _wlan0_ is in AP mode, there should be no conflicts:

```
sudo service dnsmasq stop
sudo nmcli c up hotspot
```

Whenever I need the DHCP server running, I'll do the opposite:

```
sudo nmcli c down hotspot
sudo service dnsmasq restart
```

Quite a hack, but it works like a charm!

_Note: I also experimented with setting my own dnsmasq to be used by NetworkManager, without any luck..._

### Summary - the useful part

Let's summarize all the necessary steps for having two hotspots:

- Install _dnsmasq_ and _hostapd_

```
sudo apt-get install -y dnsmasq hostapd
```

- Halt the services

```
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```

- Edit _/etc/dhcpcd.conf_

Add this at the end of the file:
```
denyinterfaces wlan1
```

- Edit _/etc/network/interfaces_

Add this at the end of the file:
```
allow-hotplug wlan1
iface wlan1 inet static  
    address 192.168.0.1
    netmask 255.255.255.0
    network 192.168.0.0
```

- Restart _dhcpcd_ and bring up _wlan1_

```shell
sudo service dhcpcd restart
sudo ifdown wlan1
sudo ifup wlan1
```

- Install Jens's hostapd

```
sudo apt-get autoremove hostapd -y
wget https://github.com/jenssegers/RTL8188-hostapd/archive/v2.0.tar.gz
tar -zxvf v2.0.tar.gz
cd RTL8188-hostapd-2.0/hostapd
sudo make
sudo make install
sudo systemctl daemon-reload
```

- Edit _/etc/dnsmasq.conf_

The file should contain only the following:
```
interface=wlan1
dhcp-range=192.168.0.2,192.168.0.20,255.255.255.0,24h
```

- Create _/etc/hostapd/hostapd.conf_

The file should contain only the following:
```
interface=wlan1
driver=rtl871xdrv
ssid=Pi_AP
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=abcdefgh
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

- Edit _/etc/default/hostapd_

Find the line with `#DAEMON_CONF`, and replace it with this:
```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

- Restart the services

```
sudo service hostapd start  
sudo service dnsmasq start
```

- Install NetworkManager to control the built-in interface

```
sudo apt-get install network-manager
```

### Moral of the story

Make sure you have enough time, refreshments and nerves when dealing with Linux-related stuff.