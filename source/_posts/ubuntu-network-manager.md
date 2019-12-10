---
title: Manage Network on Ubuntu Without GUI
date: 2019-12-10 18:51:03
tags:
- Ubuntu
- 工具
- 工作
---

### 0. Gogals
- manage network on Ubuntu without GUI.
- which kind of network type
    - ethernet
    - wifi

The topic is useful for developer who want develop a linux application, for example snap application in ubuntu.

### 1. Network Manager On Ubuntu
Use the [NetworkManager](https://docs.ubuntu.com/core/en/stacks/network/network-manager/docs/) to manage the network device and connections. NetworkManager can manage network types:
- ethenet
- wifi
- mobile boardband(WWAN)
- PPPoE device for VPN

### 2. Usage Of NetworkManager

#### 2.1 [Install NetworkManager](https://docs.ubuntu.com/core/en/stacks/network/network-manager/docs/installation)
```
# a. find the application
snap find network-manager
# b. install
snap install network-manager
# c. view the plugs
snap interfaces network-manager
# d. check the service status
systemctl status network-manager
```
when execute the command `c` the output looks like:
```
name:    network-manager
summary: allows operating as the NetworkManager service
plugs:
  - network-manager:nmcli
slots:
  - core
  - network-manager:service
```
we can find that the network manager command-line interface(nmcli) will be installed. the `nmcli` is key command what is used to manage network. If you wan to learn more about `nmcli`, you can use `nmcli help` command.

#### 2.2 [Manage WIFI](https://docs.ubuntu.com/core/en/stacks/network/network-manager/docs/configure-wifi-connections)
```
# check whether the wifi is open
nmcli radio wifi
# if return `disabled`, please enable the wifi radio
nmcli radio wifi on
# list available wifi
nmcli device wifi list
# connect to wifi
nmcli d wifi connect my_wifi password <password>
# check the connection
nmcli connection show
```
Other wifi configuration:
- powersave option(0: default, 1: ignore, 2: disable, 3: enable)
```
nmcli c modify <name> 802-11-wireless.powersave 2
```

### 2.3 [Manage Ethernet]()
before modify the Ethernet, backup the contents of /etc/netplan is a best practice.

```sh
# check whether support enthernet by network manager
sudo snap get network-manager ethernet.enable
# if return `false`, please enable ethernet support
snap set network-manager ethernet.enable=true
sudo reboot

```
after the reboot, you can edit the ethernet with `nmcli`. There two way to midfy the ethernet:
- interactive console: print/set/save
- connection modify command

We choose the second way because it can be warpped as shell script or api.
```sh
# list all connection
nmcli connection show
# show detail a connection
nmcli connection show <myconnection name or uuid>
# change the connection name
nmcli connection modify myconnection connection.id myconn
```
You can get usage detail by executing `nmcli connection modify help`, looks like below:
```
Usage: nmcli connection modify { ARGUMENTS | help }

ARGUMENTS := [id | uuid | path] <ID> ([+|-]<setting>.<property> <value>)+

Modify one or more properties of the connection profile.
The profile is identified by its name, UUID or D-Bus path. For multi-valued
properties you can use optional '+' or '-' prefix to the property name.
The '+' sign allows appending items instead of overwriting the whole value.
The '-' sign allows removing selected items instead of the whole value.

Examples:
nmcli con mod home-wifi wifi.ssid rakosnicek
nmcli con mod em1-1 ipv4.method manual ipv4.addr "192.168.1.2/24, 10.10.1.5/8"
nmcli con mod em1-1 +ipv4.dns 8.8.4.4
nmcli con mod em1-1 -ipv4.dns 1
nmcli con mod em1-1 -ipv6.addr "abbe::cafe/56"
nmcli con mod bond0 +bond.options mii=500
nmcli con mod bond0 -bond.options downdelay
```

### 3. Program
- [python-networkmanager](https://github.com/seveas/python-networkmanager): Easy communication with NetworkManager
- [dbus-network-manager](https://github.com/LGSInnovations/node-dbus-network-manager): A node.js API to interact with NetworkManager via DBus. This is currently extremely new and incomplete