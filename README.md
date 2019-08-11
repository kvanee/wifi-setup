# wifi-setup

This repo is an Express server designed to run on an IoT device and
handles the first-time setup required to get the device working
connected to the user's home wifi network.

- since the device is not on the local wifi network when it is first
  turned on, the device broadcasts its own wifi access point and runs
  the server on that. The user then connects their phone or laptop to
  that wifi network and uses a web browser (not a native app!) to
  connect to the device at the URL 10.0.0.1 or `<hostname>.local`. The
  user can select then their home wifi network and enter the password
  on a web page and transfer it to the web server running on the
  device. At this point, the device can turn off its private network
  and connect to the internet using the credentials the user provided.

The code is Linux-specific, depends on systemd, and has so far only
been tested on a Raspberry Pi 3. It requires hostapd and udhcpd to be
installed and properly configured. Here are the steps I followed to
configure and run this server.

### Step 0: clone and install

First, clone this repo and download its dependencies from npm:

```
$ git clone https://github.com/kvanee/wifi-setup.git
$ cd wifi-setup
$ npm install
```

### Step 1: AP mode setup

Install software we need to host an access point, but
make sure it does not run by default each time we boot. For Raspberry
Pi, we need to do:

```
$ sudo apt install dnsmasq hostapd
$ sudo systemctl disable hostapd
$ sudo systemctl unmask hostapd
$ sudo systemctl disable dnsmasq
```

### Step 2: configuration files
Next, configure the software:

- Edit /etc/default/hostapd to add the line:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

- Copy `config/hostapd.conf` to `/etc/hostapd/hostapd.conf`.  This
  config file defines the access point name "Wifi Setup". Edit it if
  you want to use a more descriptive name for your device.

### Step 3: Configuring the DHCP server (dnsmasq)

The DHCP service is provided by dnsmasq. By default, the configuration file contains a lot of information that is not needed, and it is easier to start from scratch. Rename this configuration file, and edit a new one:

```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
sudo nano /etc/dnsmasq.conf
```
Type or copy the following information into the dnsmasq configuration file and save it:

```
interface=wlan0      # Use the require wireless interface - usually wlan0
dhcp-range=192.168.1.2,192.168.1.20,255.255.255.0,24h
```

So for wlan0, we are going to provide IP addresses between 192.168.1.2 and 192.168.1.20, with a lease time of 24 hours. If you are providing DHCP services for other network devices (e.g. eth0), you could add more sections with the appropriate interface header, with the range of addresses you intend to provide to that interface.

There are many more options for dnsmasq; see the [dnsmasq documentation|http://www.thekelleys.org.uk/dnsmasq/doc.html] for more details.

Reload dnsmasq to use the updated configuration:

```
sudo systemctl reload dnsmasq
```

### Step 4: set up the other services you want your device to run

Once the wifi-setup server has connected to wifi, it will exit. But if
you want, it can run a command to make your device start doing
whatever it is your device does. If you want to use this feature, edit
`platforms/default.js` to define the `nextStageCommand` property.

### Step 5: run the server

If you have a keyboard and monitor hooked up to your device, or have a
serial connection to the device, then you can try out the server at
this point:

```
sudo node index.js
```

If you want to run the server on a device that has no network
connection and no keyboard or monitor, you probably want to set it up
to run automatically when the device boots up. To do this, copy
`config/wifi-setup.service` to `/lib/systemd/system`, edit it to set
the correct paths for node and for the server code, and then enable
the service with systemd:

```
$ sudo cp config/wifi-setup.service /lib/systemd/system
$ sudo vi /lib/systemd/system/wifi-setup.service # edit paths as needed
$ sudo systemctl enable wifi-setup
```

At this point, the server will run each time you reboot.  If you want
to run it manually without rebooting, do this:

```
$ sudo systemctl start wifi-setup
```

Any output from the server is sent to the systemd journal, and you can
review it with:

```
$ sudo journalctl -u wifi-setup
```

Add the -b option to the line above if you just want to view output
from the current boot.  Add -f if you want to watch the output live as
you interact with the server.

If you want these journals to persist across reboots (you probably do)
then ensure that the `/var/log/journal/` directory
exists:

```
$ sudo mkdir /var/log/journal
```
