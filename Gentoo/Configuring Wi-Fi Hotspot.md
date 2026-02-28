# Configuring Wi-Fi Hotspot

Throughout this guide we will turn our `ARM` SoC with `Gentoo` to a Wi-Fi Router.

## Initial setup

Firstly, enable `script` use flag for `dnsmasq`:

```sh
echo 'net-dns/dnsmasq script' | sudo tee /etc/portage/package.use/dnsmasq
```

Then emerge required packages:

```sh
sudo emerge --ask net-wireless/hostapd net-dns/dnsmasq net-firewall/iptables
```

Set static IP for the wireless interface (replace `wlu1` with the name of your wireless adapter):

```sh
echo 'config_wlu1="192.168.1.1/24"' | sudo tee /etc/conf.d/net
```

Then create symlink for the networking service:

```sh
sudo ln -s /etc/init.d/net.lo /etc/init.d/net.wlu1
```

Then start the service:

```sh
sudo rc-service net.wlu1 start
```

Enable `ip forwarding` in current session for enabling traffic routing from one interface to another:

```sh
sudo sysctl -w net.ipv4.ip_forward=1
```

To persist `ip forwarding`:

```sh
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.conf
```

Create `iptables` rule for replacing source ip address with the ip address of the wired interface:

```sh
sudo iptables -t nat -A POSTROUTING -o 'enp49s0' -j MASQUERADE
```

Create `iptables` rule to allow forwarding from wired interface to wireless for established connections (and to an intermediate interface created by proxy):

```sh
sudo iptables -A FORWARD -i enp49s0 -o wlu1 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i Meta -o wlu1 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Create `iptables` rule to allow forwarding from wireless interface to wired (and to an intermediate interface created by proxy):

```sh
sudo iptables -A FORWARD -i wlu1 -o enp49s0 -j ACCEPT
sudo iptables -A FORWARD -i wlu1 -o Meta -j ACCEPT
```

Configure `ip address` range and interface for `dnsmasq`:

```sh
sudo tee -a /etc/dnsmasq.conf > /dev/null << EOF
interface=wlu1
dhcp-range=192.168.1.10,192.168.1.100,12h
EOF
```

Start `dnsmasq` service:

```sh
sudo rc-service dnsmasq start
```

Create `hostapd` configuration file:

```sh
sudo tee /etc/hostapd/hostapd.conf > /dev/null << EOF
# the interface used by the AP
interface=wlu1
# "g" simply means 2.4GHz band
hw_mode=g
# the channel to use
channel=10
# limit the frequencies used to those allowed in the country
ieee80211d=1
# the country code
country_code=RU
# 802.11n support
ieee80211n=1
# QoS support, also required for full speed on 802.11n/ac/ax
wmm_enabled=1

# the name of the AP
ssid=opiap
# 1=wpa, 2=wep, 3=both
auth_algs=1
# WPA2 only
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
wpa_passphrase=foobarbazqux
EOF
```

Start `hostapd`:

```sh
sudo hostapd /etc/hostapd/hostapd.conf
```

::: tip Customizing init script for `hostapd`

To run `hostapd` as a service, if your wireless adapter name is different from the default one, change the init script at `/etc/init.d/hostapd` like this:

```sh
#!/sbin/openrc-run
# Copyright 1999-2014 Gentoo Foundation
# Distributed under the terms of the GNU General Public License v2

pidfile="/run/${SVCNAME}.pid"
command="/usr/sbin/hostapd"
command_args="-P ${pidfile} -B ${OPTIONS} ${CONFIGS}"

extra_started_commands="reload"

depend() {
  local myneeds=  # [!code --]
  local myneeds="net.wlu1"  # [!code ++]
  for iface in ${INTERFACES}; do  # [!code --]
    myneeds="${myneeds} net.${iface}"  # [!code --]
  done  # [!code --]

  [ -n "${myneeds}" ] && need ${myneeds}
  use logger
}

start_pre() {
  local file

  for file in ${CONFIGS}; do
    if [ ! -r "${file}" ]; then
      eerror "hostapd configuration file (${CONFIG}) not found"
      return 1
    fi
  done
}

reload() {
  start_pre || return 1

    ebegin "Reloading ${SVCNAME} configuration"
    kill -HUP $(cat ${pidfile}) > /dev/null 2>&1
    eend $?
}
```

:::

Assign `ip` address to the wireless interface **after** starting `hostapd`:

```sh
sudo ip addr add 192.168.1.1/24 dev wlu1
```

## Reboot

After reboot, start `net` service, then `hostapd` in a separate window:

```sh
sudo rc-service net.wlu1 start
sudo hostapd /etc/hostapd/hostapd.conf
```

Then apply these commands to start the access point:

```sh
sudo iptables -t nat -A POSTROUTING -o enp49s0 -j MASQUERADE
sudo iptables -A FORWARD -i enp49s0 -o wlu1 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i Meta -o wlu1 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlu1 -o enp49s0 -j ACCEPT
sudo iptables -A FORWARD -i wlu1 -o Meta -j ACCEPT
sudo ip addr add 192.168.1.1/24 dev wlu1
sudo rc-service dnsmasq start
```

::: tip Saving iptables

To save iptable rules, run the following command:

```sh
sudo rc-service iptables save
```

And make sure that `iptables` service is added to the `default` runlevel:

```sh
sudo rc-update add iptables default
```

:::

## Run on startup

To initialize access point on device startup, create a custom script located `/etc/local.d/wifi` with content like this:

```sh
#!/bin/bash

echo 0 > /sys/bus/usb/devices/13-1/authorized
sleep 1
echo 1 > /sys/bus/usb/devices/13-1/authorized

# Wait until interface exists
for i in $(seq 1 10); do
  if ip link show wlu1 >/dev/null 2>&1; then
    break
  fi
  sleep 1
done

# Bring interface up explicitly
ip link set wlu1 up

# Start wpa_supplicant (client mode)
# rc-service wpa_supplicant start

# Start hostapd (router mode)
rc-service net.wlu1 start
sleep 1
rc-service dnsmasq start
sleep 1
rc-service hostapd start
sleep 10
ip addr add 192.168.1.1/24 dev wlu1
```

## Redirect from one wireless interface to another

Configure which intefrace `wpa_supplicant` should use in client mode by editing `/etc/conf.d/wpa_supplicant`:

```sh
wpa_supplicant_args=""  # [!code --]
wpa_supplicant_args="-i wlan0"  # [!code ++]
```

Create script for automatic initialization of `wpa_supplicant`, put it at `/etc/local.d/dlink.start`, for example:

```sh
#!/bin/bash

IFACE="wlan0"

while true; do
  if ifconfig "$IFACE"; then
    # Check if the interface has an IP assigned
    IP=$(ifconfig "$IFACE" 2>/dev/null | grep 'inet ' | awk '{print $2}')

    if [ -z "$IP" ]; then
      echo "$(date) No IP found on $IFACE. Restarting wpa_supplicant..." >> /tmp/validate-wireless-interfase-log.txt
      sudo rc-service wpa_supplicant restart
    else
      echo "$(date) $IFACE has IP: $IP" >> /tmp/validate-wireless-interfase-log.txt
      break
    fi
    sleep 60
  else
    echo "$(date) Interface $IFACE is not available" >> /tmp/validate-wireless-interfase-log.txt
    sleep 1
  fi
done
```

Update `iptables` rules:

```sh
sudo iptables -t nat -D POSTROUTING -o enp49s0 -j MASQUERADE
sudo iptables -D FORWARD -i enp49s0 -o wlu1 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -D FORWARD -i wlu1 -o enp49s0 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i wlan0 -o wlu1 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlu1 -o wlan0 -j ACCEPT
```

Persist changes:

```sh
sudo rc-service iptables save
```

## Troubleshooting

If interfaces are failing to start up after reboot, try to disconnect the adapters and connect them back. To start access point manually run the command:

```sh
sudo rc-service net.wlu1 restart; sleep 1; sudo rc-service dnsmasq restart; sleep 1; sudo rc-service hostapd restart; sleep 10; sudo ip addr add 192.168.1.1/24 dev wlu1
```

To start `wpa_supplicant` manually:

```sh
sudo rc-service wpa_supplicant start
```
