## Setting up a Raspberry Pi as an access point in a wireless network

The Raspberry Pi can be used as a wireless access point, running a standalone network. This can be done using the inbuilt wireless features of the Raspberry Pi 3.

Note that this documentation was tested on a Raspberry Pi 3 with Raspbian Stretch, and it is possible that some USB dongles may need slight changes to their settings. If you are having trouble with a USB wireless dongle, please check the forums.

To add a Raspberry Pi-based access point to an existing network, see [this section](https://github.com/SurferTim/documentation/blob/6bc583965254fa292a470990c40b145f553f6b34/configuration/wireless/access-point.md#internet-sharing).

In order to work as an access point, the Raspberry Pi will need to have access point software installed, along with DHCP server software to provide connecting devices with a network address. Ensure that your Raspberry Pi is using an up-to-date version of Raspbian (dated 2017 or later).

Use the following to update your Raspbian installation:

```
sudo apt-get update
sudo apt-get upgrade
```

Install all the required software in one go with this command:

```
sudo apt-get install dnsmasq hostapd
```

Since the configuration files are not ready yet, turn the new software off as follows:

```
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```

### Configuring a static IP

We are configuring a standalone network to act as a server, so the Raspberry Pi needs to have a static IP address assigned to the wireless port. This documentation assumes that we are using the standard 192.168.x.x IP addresses for our wireless network, so we will assign the server the IP address 192.168.0.1. It is also assumed that the wireless device being used is `wlan0`.

To configure the static IP address, edit the dhcpcd configuration file with:

```
sudo vi /etc/dhcpcd.conf
```

Go to the end of the file and edit it so that it looks like the following:

```
interface wlan0
    static ip_address=192.168.4.1/24
```

Now restart the dhcpcd daemon and set up the new `wlan0` configuration:

```
sudo service dhcpcd restart 
```

### Configuring the DHCP server (dnsmasq)

The DHCP service is provided by dnsmasq. By default, the configuration file contains a lot of information that is not needed, and it is easier to start from scratch. Rename this configuration file, and edit a new one:

```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig  
sudo vi /etc/dnsmasq.conf
```

Type or copy the following information into the dnsmasq configuration file and save it:

```
interface=wlan0      # Use the require wireless interface - usually wlan0
  dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
```

So for `wlan0`, we are going to provide IP addresses between 192.168.4.2 and 192.168.4.20, with a lease time of 24 hours. If you are providing DHCP services for other network devices (e.g. eth0), you could add more sections with the appropriate interface header, with the range of addresses you intend to provide to that interface.

There are many more options for dnsmasq; see the [dnsmasq documentation](http://www.thekelleys.org.uk/dnsmasq/doc.html) for more details.

### Configuring the access point host software (hostapd)

You need to edit the hostapd configuration file, located at /etc/hostapd/hostapd.conf, to add the various parameters for your wireless network. After initial install, this will be a new/empty file.

```
sudo vi /etc/hostapd/hostapd.conf
```

Add the information below to the configuration file. This configuration assumes we are using channel 7, with a network name of `NameOfNetwork`, and a password `PasswordForAP`. Note that the name and password should **not** have quotes around them.

```
interface=wlan0
driver=nl80211
ssid=<NameOfNetwork>
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=<PasswordForAP>
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

We now need to tell the system where to find this configuration file.

```
sudo vi /etc/default/hostapd
```

Find the line with #DAEMON_CONF, and replace it with this:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

### Start it up

Now start up the remaining services:

```
sudo service hostapd start  
sudo service dnsmasq start  
```

### Add routing and masquerade

Edit /etc/sysctl.conf and uncomment this line:

```
net.ipv4.ip_forward=1
```

Add a masquerade for outbound traffic on eth0:

```
sudo iptables -t nat -A  POSTROUTING -o eth0 -j MASQUERADE
```

Save the iptables rule.

```
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

Edit /etc/rc.local and add this just above "exit 0" to install these rules on boot.

```
iptables-restore < /etc/iptables.ipv4.nat
```

Reboot

Using a wireless device, search for networks. The network SSID you specified in the hostapd configuration should now be present, and it should be accessible with the specified password.

If SSH is enabled on the Raspberry Pi access point, it should be possible to connect to it from another Linux box (or a system with SSH connectivity present) as follows, assuming the `pi` account is present:

```
ssh pi@192.168.4.1
```

By this point, the Raspberry Pi is acting as an access point, and other devices can associate with it. Associated devices can access the Raspberry Pi access point via its IP address for operations such as `rsync`, `scp`, or `ssh`.

## Using the Raspberry Pi as an access point to share an internet connection

One common use of the Raspberry Pi as an access point is to provide wireless connections to a wired Ethernet connection, so that anyone logged into the access point can access the internet, providing of course that the wired Ethernet on the Pi can connect to the internet via some sort of router.

## /etc/network/interfaces.d/station

```
allow-hotplug wlan1
iface wlan1 inet manual
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```

## Do not let DHCPCD manage wpa_supplicant!!

```
sudo rm -f /lib/dhcpcd/dhcpcd-hooks/10-wpa_supplicant
```

## Set up the client wifi (station) on wlan1.

Create `/etc/wpa_supplicant/wpa_supplicant.conf`. The contents depend on whether your home network is open, WEP or WPA. It is probably WPA, and so should look like:

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
country=GB

network={
    ssid="WiFi_1"
    scan_ssid=1
    psk="password_for_wifi_1"
    key_mgmt=WPA-PSK
    priority=1
}

network={
    ssid="WiFi_2"
    scan_ssid=1
    psk="password_for_wifi_2"
    key_mgmt=WPA-PSK
    priority=2
}
```

Replace `Wifi_x` with your home network SSID and `password_for_wifi_x` with your wifi password (in clear text).

## Restart DHCPCD

```
sudo systemctl restart dhcpcd
```

## Now restart the dns and hostapd services

```
sudo systemctl restart dnsmasq
sudo systemctl restart hostapd
```

## Restart the client interface

I dunno. The client interface went down for some reason (see below "bringup order"). Bring it back up:

```
sudo ifdown wlan0
sudo ifup wlan0
```

## Permanently deal with interface bringup order

```
# see https://unix.stackexchange.com/questions/396059/unable-to-establish-connection-with-mlme-connect-failed-ret-1-operation-not-p
```

Edit `/etc/rc.local` and add the following lines just before "exit 0":

```
sleep 5
ifdown wlan0
sleep 2
rm -f /var/run/wpa_supplicant/wlan0
ifup wlan0
```

## Bridge AP to cient side

This is optional. If you do this step, then someone connected to the AP side can browse the internet through the client side.

```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -s 192.168.4.0/24 ! -d 192.168.4.0/24 -j MASQUERADE
iptables-save > /etc/iptables/rules.v4
```



Now reboot the Raspberry Pi.

There should now be a functioning bridge between the wireless LAN0 and the wireless LAN1 connection on the Raspberry Pi, and any device associated with the Raspberry Pi access point will act as if it is connected to the access point's wireless network.
