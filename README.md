# vaani.setup

This repo is an Express server that runs on the Vaani device and
handles the first-time setup required to get the device working:

- since the device is not on the local wifi network when it is first
  turned on, the device broadcasts its own wifi access point and runs
  the server on that. The user then connects their phone or laptop to
  that wifi network and uses a web browser (not a native app!) to
  connect to the device at the URL `vaani.local`. The user can select
  then their home wifi network and enter the password on a web page
  and transfer it to the web server running on the device. At this
  point, the device can turn off its private network and connect to
  the internet using the credentials the user provided.

- after re-connecting to their home wifi, the user can reload the
  `vaani.local` page and handle the second part of setup, which is to
  perform OAuth authentication with Evernote to obtain an access
  token. (The server saves the token and related values, such as the
  expiration date, in a file named `oauthToken.json`.)

The code is Linux-specific, depends on systemd, and has so far only
been tested on a Raspberry Pi 3. It requires hostapd and udhcpd to be
installed and properly configured. Here are the steps I followed to
configure and run this server. Note that the steps include
instructions for Raspberry Pi and Edison, but that I have not yet been
able to successfully run on Edison

### Step 0: clone and install

First, clone this repo and download its dependencies from npm:

```
$ git clone https://github.com/mozilla/vaani.setup.git
$ cd vaani.setup
$ npm install
```

Next, you need to create a config file:

```
$ cd vaani.setup
$ cp evernoteConfig.json.template evernoteConfig.json
```

Edit evernoteConfig.json to add your Evernote API "consumer key" and
"consumer secret" values. You need to register your app with Evernote
to get these.

### Step 1: Edison specific setup

If you're running this software on an Intel Edison instead of a
Raspberry Pi, you'll probably need to modify the default yocto Linux
build, as follows:

If you don't already have node 4.4, update your node and npm with
commands like these:

```
# curl https://nodejs.org/dist/v4.4.7/node-v4.4.7-linux-x86.tar.xz | zcat | tar xf - -C /usr/local
# cat <<EOF >> ~/.profile
export PATH=/usr/local/node-v4.4.7-linux-x86/bin:$PATH
EOF
# source ~/.profile
# node --version
v4.4.7
```

If your Edison is running mdnsd, you'll probably need to disable that
and install avahi instead. These software packages are both supposed
to do mdns aka zeroconf aka bonjour so that you can refer to your
device by the name 'hostname.local'. But the mdns package doesn't work
on my Edison, so I've swapped it out for avahi, which is what
Raspberry pi uses. Commands like these should work:

```
# systemctl disable mdns
# systemctl stop mdns
# opkg install avahi
# reboot
```

By default, my Edison was already running an HTTP server on port 80,
so this vaani.setup server was not able to run. I disabled the
edison_config server like this:

```
# systemctl disable edison_config
# systemctl stop edison_config
```

### Step 2: AP mode setup

Install software we need to host an access point, but
make sure it does not run by default each time we boot. For Raspberry
Pi, we need to do:

```
$ sudo apt-get install hostapd
$ sudo apt-get install udhcpd
$ sudo systemctl disable hostapd
$ sudo systemctl disable udhcpd
```

On my Edison device, hostapd and udhcpd are already installed and
disabled (but the udhcpd service is named `udhcpd-for-hostapd`) so
these steps are not necessary.

### Step 3: configuration files
Next, configure the software:

- On Raspberry Pi, edit /etc/default/hostapd to add the line:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```
this step is not necessary on Edison.

- Copy `config/hostapd.conf` to `/etc/hostapd/hostapd.conf`.  This
  config file defines the access point name "Vaani Setup". Edit it if
  you want to use a different name. On Edison
  `/etc/hostapd/hostapd.conf` alread exists. You may want to rename
  the existing file rather than overwriting it.

- On Raspberry Pi (but not Edison) edit the file `/etc/default/udhcpd`
  and comment out the line:

```
DHCPD_ENABLED="no"
```

- On Edison (but not Raspberry Pi) edit the file
  `/lib/systemd/system/udhcpd-for-hostapd.service` and modify this
  line:

```
ExecStartPre=/sbin/ifconfig wlan0 192.168.42.1 up
```
changing `192.168.42.1` to `10.0.0.1`. This is necessary because
`config/udhcpd.conf` and `wifi.js` use 10.0.0.1 as the local IP
address when we're broadcasting an access point.

- On Raspberry Pi, copy `config/udhcpd.conf` to `/etc/udhcp.conf`.
On Edison, rename `/etc/hostapd/udhcpd-for-hostapd.conf` to
`/etc/hostapd/udhcpd-for-hostapd.conf.orig`, and then copy
`config/udhcpd.conf` to `/etc/hostapd/udhcpd-for-hostapd.conf`.

### Step 4: set up the other Vaani services

Once the vaani.setup server has connected to wifi and has gotten an
oauth token, it will start an auto-update service. That auto-update
service will start the Vaani client software. In order for this all to
work, you need to have both of these pieces of software installed:

```
$ git clone https://github.com/andrenatal/git-auto-updater.git
$ git clone git@github.com:mozilla/vaani.client.git
```

You'll need to create appropriate systemd .service files for both of
these and put them in `/lib/systemd/system/vaani.service` and
`/lib/systemd/system/git-auto-updater.service`.

Importantly, this vaani.setup service stores the OAUTH token in
an environment variable in
`/lib/systemd/system/vaani.service.d/evernote.conf`. In order to do
this, you need to ensure that the directory exists:

```
$ sudo mkdir /lib/systemd/system/vaani.service.d
```

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
`config/vaani-setup.service` to `/lib/systemd/system`, edit it to set
the correct paths for node and for the server code, and then enable
the service with systemd:

```
$ sudo cp config/vaani-setup.service /lib/systemd/system
$ sudo vi /lib/systemd/system/vaani-setup.service # edit paths as needed
$ sudo systemctl enable vaani-setup
```

At this point, the server will run each time you reboot.  If you want
to run it manually without rebooting, do this:

```
$ sudo systemctl start vaani-setup
```

Any output from the server is sent to the systemd journal, and you can
review it with:

```
$ sudo journalctl -u vaani-setup
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
