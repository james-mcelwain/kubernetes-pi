# Installation

## Raspbian

Raspbian comes in two flavors: desktop and lite. Since we won't be using a desktop environment, we can grab the lite version: 
   ```
   curl -L https://downloads.raspberrypi.org/raspbian_lite_latest \
   -o raspbian.zip
   ```

RaspberryPi provides a graphical installer for flashing SD cards "NOOBS." Several guides suggested we use the install script for HypriotOS, available at `github.com/hypriot/flash`, or installed:
```
curl -O https://raw.githubusercontent.com/hypriot/flash/master/$(uname -s)/flash
chmod +x flash
sudo mv flash /usr/local/bin/flash
```
NB: On Arch, we'll want to install the binary to /usr/bin rather than /usr/local/bin, since the later isn't on our path by default.

I ran into a few issues with `flash`. 

First, user error: an error message `BLKRRPART failed: Invalid argument`. This one is simple. `flash` expects a disk rather than a partition. I made the mistake of passing the single partition on a new sd card `/dev/sda1` rather than the device itself.

The second was a little more cryptic: `mount: /tmp/mnt.2396: can't find in /etc/fstab.` Looking at the script itself, it seems that there is additional coordination that happens on the host machine after flashing the actual image to the SD card, mostly involving user confs. Since we're not using Hypriot, I decided to flash the SD cards just using `dd`:
```
dd bs=1M if=~/Downloads/raspbian.zip o=/dev/sda
```

This seemed to work fine, and is easy to understand from experience, i.e. the Arch live install process.

Booting into Raspbian we can log into the device with the username `pi` and the password `raspberry`. I'm using an ethernet switch to connect my laptop to the pis, so we'll need to set up some networking rules. On my laptop, I'm using a usb ethernet port, and will be using my wireless network interface as the gateway to the internet. We can set up some rules using `iptables`:
```
sudo -i

ETHERNET_INTERFACE=...
WIRELESS_INTERFACE=...

sysctl net.ipv4.ip_forward=1

ip link set up dev $ETHERNET_INTERFACE 
ip addr add 192.168.123.100/24 dev net0 # arbitrary address

iptables -t nat -A POSTROUTING -o $WIRELESS_INTERFACE -j MASQUERADE
iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i $ETHERNET_INTERFACE -o $WIRELESS_INTERFACE -j ACCEPT

```

There's probably more granular ways to deal with this -- especially because these configurations won't persist after restart, but it's good enough for now.

Now, on each of the pis, we'll assign a static ip:
```
ip addr add 192.168.123.20{n..5}/24 dev eth0 
ip link set up dev eth0
ip route add default via 192.168.123.100 dev eth0 # the static ip we assigned to the laptop earlier
```

If we want to have this persist after restart, we can add a profile to `/etc/dhcpcd.conf`. Technically, there are better places to put configurations for a static ip, but we may want to set up dhcpcd later, so this is fine for now. At the bottom of the config, we can add:
```
profile static_eth0
static ip_address=192.168.123.200/24
static routes=192.168.123.100

interface eth0
fallback static_eth0
```

We'll also want to make sure that sshd starts on boot with systemd:
```
sudo systemctl enable ssh
sudo systemctl start ssh
```


## Ansible
Ansible is available in the Arch user repository:
`pacaur -S ansible`

Ansible uses a push based system for maintaining configurations by default, so we'll need to tell it about where our pis live.
