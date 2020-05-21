



# 树莓派4B实现无线路由（开启AP）

```
从树莓派官网下载镜像
https://www.raspberrypi.org/downloads/raspbian/
```

**这是目前官网最新的 2020-02-13-raspbian-buster-full.zip**

**解压后得到2020-02-13-raspbian-buster-full.img**

**再通过工具（比如Etcher）将镜像写入microSD卡中（我的是一张16G的TF卡,读取速度97M/S）**



```
root@raspberrypi:/etc/hostapd# uname -a
Linux raspberrypi 4.19.97-v7l+ #1294 SMP Thu Jan 30 13:21:14 GMT 2020 armv7l GNU/Linux
root@raspberrypi:/etc/hostapd# lsb_release -a
No LSB modules are available.
Distributor ID: Raspbian
Description:    Raspbian GNU/Linux 10 (buster)
Release:        10
Codename:       buster
这是我的系统版本
```

***网上找的关于树莓派4B的无线WiFi配置***：==802.11n 无线2.4GHz /5GHz 双频WiFi==
#### IP设置
**下面是在网络的配置文件中增加相应的网卡信息，eth0是我这个树莓派有线网卡接口（address是我这里的局域网IP地址 这个地址需要有能访问互联网的权限、 metmask是子网掩码 gateway默认网关 dns-nameservers 公网DNS服务器IP 地址。wlan0是无线wifi 这里地址自己设置的，要注意不能与eth0在同一个网段！**
[IP]

```vim /etc/network/interfaces 
vim /etc/network/interfaces
auto lo
iface lo inet loopback
auto eth0
iface eth0 inet static
        address 192.168.92.39
        netmask 255.255.255.0
        gateway 192.168.92.254
        dns-nameservers 202.106.0.20
auto wlan0
allow-hotplug wlan0
iface wlan0 inet static
        address 192.168.10.10
        netmask 255.255.255.0
```

为了安装软件速度更快，更换成清华的镜像源。之前的用#号注释掉，增加倒数着两行。

```
root@raspberrypi:~# cat /etc/apt/sources.list
#deb http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi
# Uncomment line below then 'apt-get update' to enable 'apt-get source'
#deb-src http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi
deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main contrib non-free rpi
deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main contrib non-free rpi
```

还可以更改一下系统源，还是将之前的用#号注释掉。增加最后两行清华的源。                

```
root@raspberrypi:~# cat /etc/apt/sources.list.d/raspi.list 
#deb http://archive.raspberrypi.org/debian/ buster main
# Uncomment line below then 'apt-get update' to enable 'apt-get source'
#deb-src http://archive.raspberrypi.org/debian/ buster main

deb http://mirrors.tuna.tsinghua.edu.cn/raspberrypi/ buster main ui
deb-src http://mirrors.tuna.tsinghua.edu.cn/raspberrypi/ buster main ui
```



更新一下系统的可用软件列,再通过安装/升级 软件来更新新系统

```
root@raspberrypi:~# apt update 
root@raspberrypi:~# apt upgrade
```

下面是安装要用到的软件包

`root@raspberrypi:~# apt install hostapd` <font size=2>==能使无线网卡工作在软AP（Access Point）模式，即无线路由器==</font>

`root@raspberrypi:~# apt install dnsmasq` <font size=2>==能够同时提供DHCP和DNS服务==</font>

==还可以使用isc-dhcp-server和bind9这两个包，分别是服务DHCP和DNS,但是使用dnsmasq已经足够了。==

在最新的树莓派版本中，所有的网络接口默认使用dhcpcd.service服务进行配置，因为wlan0工作在AP模式，所以我们要手动给他配置静态IP地址，先在配置文件/etc/dhcpcd.conf中最下面添加一行禁用wlan0 ,否则wlan0和eth0会发生冲突。

root@raspbian:~# echo "denyinterfaces wlan0" >> /etc/dhcpcd.conf

==另外还需要在/etc/network/interfaces 中增加[IP地址信息](#IP设置)，这个我们在前面已经做过了。==



OK我们现在配置hostapd服务吧，也就是wifi信息。 

```
root@raspberrypi:/etc/hostapd# vim /etc/hostapd/hostapd.conf
interface=wlan0
driver=nl80211
ssid=benben #ssid名字自己定义
hw_mode=g #网卡工作在802.11G模式
channel=6 #无线网卡选用6通道
wmm_enabled=1
macaddr_acl=0
auth_algs=3
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=123456 #这个是访问wifi的密码，也就是你用手机连接wifi时输入的密码。
wpa_key_mgmt=WPA-PSK #认证方式WPA-PSK
rsn_pairwise=CCMP #加密方式CCMP
```

这时我们可以验证一下 hostapd服务

```
root@raspberrypi:/home/RaspberryPi# systemctl restart hostapd.service
root@raspberrypi:/home/RaspberryPi# systemctl status hostapd.service
* hostapd.service - Advanced IEEE 802.11 AP and IEEE 802.1X/WPA/WPA2/EAP Authenticator
   Loaded: loaded (/lib/systemd/system/hostapd.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2020-05-20 14:26:19 CST; 1s ago
  Process: 1911 ExecStart=/usr/sbin/hostapd -B -P /run/hostapd.pid -B $DAEMON_OPTS ${DAEMON_CONF} (code=exited, status=0/SUCCESS)
 Main PID: 1913 (hostapd)
    Tasks: 1 (limit: 4915)
   Memory: 540.0K
   CGroup: /system.slice/hostapd.service
           `-1913 /usr/sbin/hostapd -B -P /run/hostapd.pid -B /etc/hostapd/hostapd.conf

5月 20 14:26:19 raspberrypi systemd[1]: Starting Advanced IEEE 802.11 AP and IEEE 802.1X/WPA/WPA2/EAP Authenticator...
5月 20 14:26:19 raspberrypi hostapd[1911]: Configuration file: /etc/hostapd/hostapd.conf
5月 20 14:26:19 raspberrypi hostapd[1911]: wlan0: Could not connect to kernel driver
5月 20 14:26:19 raspberrypi hostapd[1911]: Using interface wlan0 with hwaddr dc:a6:32:53:d8:bb and ssid "benben"
5月 20 14:26:19 raspberrypi hostapd[1911]: wlan0: interface state UNINITIALIZED->ENABLED
5月 20 14:26:19 raspberrypi hostapd[1911]: wlan0: AP-ENABLED
5月 20 14:26:19 raspberrypi systemd[1]: Started Advanced IEEE 802.11 AP and IEEE 802.1X/WPA/WPA2/EAP Authenticator.
```

再来配置dnsmasq服务,应该是你手机端配分配的IP 等设置

```
root@raspberrypi:~# vim /etc/dnsmasq.conf
interface=wlan0 #wifi的接口
listen-address=192.168.10.10 #wifi接口的IP地址，我个人觉得也应该是手机被分配IP时的网关地址。
bind-interfaces
server=202.106.0.20
server=4.2.2.2
domain-needed
bogus-priv
dhcp-range=192.168.10.11,192.168.10.254,12h #分配给手机的IP地址范围，12h是IP地址时间租约。
```

验证一下dnsmasq服务

```
root@raspberrypi:~# systemctl restart dnsmasq.service
root@raspberrypi:~# systemctl status dnsmasq.service
* dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server
   Loaded: loaded (/lib/systemd/system/dnsmasq.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2020-05-20 14:34:40 CST; 1s ago
  Process: 1992 ExecStartPre=/usr/sbin/dnsmasq --test (code=exited, status=0/SUCCESS)
  Process: 1993 ExecStart=/etc/init.d/dnsmasq systemd-exec (code=exited, status=0/SUCCESS)
  Process: 2002 ExecStartPost=/etc/init.d/dnsmasq systemd-start-resolvconf (code=exited, status=0/SUCCESS)
 Main PID: 2001 (dnsmasq)
    Tasks: 1 (limit: 4915)
   Memory: 1.6M
   CGroup: /system.slice/dnsmasq.service
           `-2001 /usr/sbin/dnsmasq -x /run/dnsmasq/dnsmasq.pid -u dnsmasq -r /run/dnsmasq/resolv.conf -7 /etc/dnsmasq.d,.dpkg-dist,.dpkg-old,.dpkg-new --local-service --trust-an

5月 20 14:34:40 raspberrypi dnsmasq-dhcp[2001]: DHCP, sockets bound exclusively to interface wlan0
5月 20 14:34:40 raspberrypi dnsmasq[2001]: using nameserver 4.2.2.2#53
5月 20 14:34:40 raspberrypi dnsmasq[2001]: using nameserver 202.106.0.20#53
5月 20 14:34:40 raspberrypi dnsmasq[2001]: reading /run/dnsmasq/resolv.conf
5月 20 14:34:40 raspberrypi dnsmasq[2001]: using nameserver 4.2.2.2#53
5月 20 14:34:40 raspberrypi dnsmasq[2001]: using nameserver 202.106.0.20#53
5月 20 14:34:40 raspberrypi dnsmasq[2001]: using nameserver 202.106.0.20#53
5月 20 14:34:40 raspberrypi dnsmasq[2001]: read /etc/hosts - 5 addresses
5月 20 14:34:40 raspberrypi dnsmasq[2002]: Too few arguments.
5月 20 14:34:40 raspberrypi systemd[1]: Started dnsmasq - A lightweight DHCP and caching DNS server.

```

接下来开启树莓派的路由转发功能

```
root@raspberrypi:~# vim /etc/sysctl.conf
root@raspberrypi:~# egrep -v '^$|^#' /etc/sysctl.conf
net.ipv4.ip_forward=1 #将设条前面的#好去掉就行了
```

最后设置防火墙

```
root@raspberrypi:~# iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
root@raspberrypi:~# iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
root@raspberrypi:~# iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
root@raspberrypi:~# sh -c "iptables-save > /etc/iptables.ipv4.nat"
root@raspberrypi:~# echo "up iptables-restore < /etc/iptables.ipv4.nat" >> /etc/network/interfaces #为了防止系统重启后防火墙信息清除，需要执行最后这两条
```

reboot后就可以连接wifi了
