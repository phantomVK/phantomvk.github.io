---
layout:     post
title:      "树莓派设置USB无线网卡"
date:       2017-10-21
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - RaspberryPi
---

## 一、注意和准备

### 1.1 注意事项

1. 使用无线网卡前请检查电源能否提供足够电压和电流；

2. 使用最新系统：假如系统恰好增加新网卡的支持，能免去自己编译驱动的麻烦；

3. 仅针对RaspberryPi 3B以下的设备，这些设备没有配备无线网卡；

4. 请用有线ssh登入，设置过程中树莓派是需要重启无线网络的。

### 1.2 硬件信息

先看我使用的无线网卡：

```shell
pi@raspberrypi:~ $ lsusb
Bus 001 Device 004: ID 148f:760b Ralink Technology, Corp. MT7601U Wireless Adapter
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp. SMSC9512/9514 Fast Ethernet Adapter
Bus 001 Device 002: ID 0424:9514 Standard Microsystems Corp. SMC9514 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

上面显示的`Ralink Technology, Corp. MT7601U Wireless Adapter`是我的无线网卡。

在`Raspbian Stretch Lite 2017-09-07 Kernel 4.9`系统镜像中已经自带该网卡的驱动。如果你的无线网卡在系统中无法识别，请自行编译安装驱动。

### 1.3 查看已经连接的网络

使用`iwconfig`查看`wlan0`的详情：

```shell
pi@raspberrypi:~ $ iwconfig

eth0      no wireless extensions.
lo        no wireless extensions.
wlan0     IEEE 802.11  ESSID:"NETGEAR76"
          Mode:Managed  Frequency:2.462 GHz  Access Point: 10:DA:43:7A:A3:E8
          Bit Rate=24 Mb/s   Tx-Power=20 dBm
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:off
          Link Quality=39/70  Signal level=-71 dBm
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:32   Missed beacon:0
```

上面已经显示了连接到`NETGEAR76`，连接频段和信号质量等等信息。

## 二、配置无线网络

### 2.1 扫描可见SSID

通过`sudo iwlist wlan0 scan`扫描附近可见的SSID，并截取两个无线路由器的信息。后者`NETGEAR76`是已经连接的，现用前者`MERCURY_4E1A`演示。

```shell
pi@raspberrypi:~ $ sudo iwlist wlan0 scan
wlan0     Scan completed :
          Cell 08 - Address: E4:F3:F5:0F:4E:1A
                    Channel:11
                    Frequency:2.462 GHz (Channel 11)
                    Quality=47/70  Signal level=-63 dBm
                    Encryption key:on
                    ESSID:"MERCURY_4E1A"
                    Bit Rates:1 Mb/s; 2 Mb/s; 5.5 Mb/s; 11 Mb/s; 6 Mb/s
                              9 Mb/s; 12 Mb/s; 18 Mb/s
                    Bit Rates:24 Mb/s; 36 Mb/s; 48 Mb/s; 54 Mb/s
                    Mode:Master
                    Extra:tsf=000004fb6621e986
                    Extra: Last beacon: 20ms ago
                    IE: Unknown: 000C4D4552435552595F34453141
                    IE: Unknown: 010882848B960C121824
                    IE: Unknown: 03010B
                    IE: Unknown: 2A0100
                    IE: Unknown: 32043048606C
                    IE: Unknown: 2D1A6E1003FFFF000000000000000000000000000000000000000000
                    IE: Unknown: 3D160B071100000000000000000000000000000000000000
                    IE: IEEE 802.11i/WPA2 Version 1
                        Group Cipher : CCMP
                        Pairwise Ciphers (1) : CCMP
                        Authentication Suites (1) : PSK
                    IE: WPA Version 1
                        Group Cipher : CCMP
                        Pairwise Ciphers (1) : CCMP
                        Authentication Suites (1) : PSK
                    IE: Unknown: DD180050F2020101000003A4000027A4000042435E0062322F00
                    IE: Unknown: DD05000AEB0100
          Cell 09 - Address: 10:DA:43:7A:A3:E8
                    Channel:11
                    Frequency:2.462 GHz (Channel 11)
                    Quality=41/70  Signal level=-69 dBm
                    Encryption key:on
                    ESSID:"NETGEAR76"
                    Bit Rates:1 Mb/s; 2 Mb/s; 5.5 Mb/s; 11 Mb/s; 18 Mb/s
                              24 Mb/s; 36 Mb/s; 54 Mb/s
                    Bit Rates:6 Mb/s; 9 Mb/s; 12 Mb/s; 48 Mb/s
                    Mode:Master
                    Extra:tsf=00000119cffe47dc
                    Extra: Last beacon: 20ms ago
                    IE: Unknown: 00094E4554474541523736
                    IE: Unknown: 010882840B162430486C
                    IE: Unknown: 03010B
                    IE: Unknown: 2A0100
                    IE: Unknown: 2F0100
                    IE: IEEE 802.11i/WPA2 Version 1
                        Group Cipher : CCMP
                        Pairwise Ciphers (1) : CCMP
                        Authentication Suites (1) : PSK
                    IE: Unknown: 32040C121860
                    IE: Unknown: 0B0501001D0000
                    IE: Unknown: 7F080400080000000040
                    IE: Unknown: DD7B0050F204104A0001101044000102103B0001031047001021A685A13B70898A47777B80AB552F671021000D4E4554474541522C20496E632E102300055236323530102400055236323530104200033637391054000800060050F2040001101100055236323530100800022008103C0001031049000600372A000120
                    IE: Unknown: DD090010180201000C0000
                    IE: Unknown: DD180050F2020101880003A4000027A4000042435E0062322F00
                    IE: Unknown: 46057208010000
```

### 2.2 设置SSID和对应密码

使用命令`wpa_passphrase SSID 密码`生成SSID和对应的密码，复制内容备用：

```shell
pi@raspberrypi:~ $ wpa_passphrase MERCURY_4E1A blackcar
network={
	ssid="MERCURY_4E1A"
	#psk="blackcar"
	psk=7763537e8128e46f91200bab0a3d9adc08d701a0e1a555d0285fd1e10b185ab9
}
```

或通过重定向命令写入到文件备用:

```shell
pi@raspberrypi:~ $ wpa_passphrase MERCURY_4E1A blackcar>~/wifi.conf
```

### 2.3 修改wpa_supplicant.conf

修改`wpa_supplicant.conf`需要管理员权限，请切换到`root`:

```shell
pi@raspberrypi:~ $ su
Password:
```

把内容粘贴到`/etc/wpa_supplicant/wpa_supplicant.conf`：

```shell
root@raspberrypi:/home/pi# nano /etc/wpa_supplicant/wpa_supplicant.conf
```

或重定向的方式给`wpa_supplicant.conf`追加`wifi.conf`：

```shell
root@raspberrypi:/home/pi# less wifi.conf >> /etc/wpa_supplicant/wpa_supplicant.conf
```
### 2.4 修改配置文件

配置`/etc/network/interfaces`:

```shell
sudo nano /etc/network/interfaces
```


下面是我的配置，同时支持`eth0`和`wlan0`。当网线和Wifi同时连接时会独立获得ip，`wlan0`的配置表示通过DHCP获取IP地址。

```
# interfaces(5) file used by ifup(8) and ifdown(8)

# Please note that this file is written to be used with dhcpcd
# For static IP, consult /etc/dhcpcd.conf and 'man dhcpcd.conf'

# Include files from /etc/network/interfaces.d:
source-directory /etc/network/interfaces.d

auto lo
iface lo inet loopback

allow-hotplug eth0
iface eth0 inet dhcp

allow-hotplug wlan0
auto wlan0
iface wlan0 inet dhcp
pre-up wpa_supplicant -B w -D wext -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
post-down killall -q wpa_supplicant
```

### 2.5 重启网络

设置完成后保存退出，并重启树莓派的的网络：

```shell
sudo /etc/init.d/networking restart
sudo ifup wlan0
```

## 三、 查看连接后信息

连接成功后，用`iwconfig`查看信息

```shell
pi@raspberrypi:~ $ iwconfig
eth0      no wireless extensions.

lo        no wireless extensions.

wlan0     IEEE 802.11  ESSID:"MERCURY_4E1A"
          Mode:Managed  Frequency:2.437 GHz  Access Point: E4:F3:F5:0F:4E:1A
          Bit Rate=24 Mb/s   Tx-Power=20 dBm
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:off
          Link Quality=46/70  Signal level=-64 dBm
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:5   Missed beacon:0
```

用`ifconfig`查看信息，获得了IP地址`192.168.0.101`

```shell
pi@raspberrypi:~ $ ifconfig
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1  (Local Loopback)
        RX packets 8  bytes 312 (312.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8  bytes 312 (312.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.101  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::236:76ff:fe06:8115  prefixlen 64  scopeid 0x20<link>
        ether 00:36:76:06:81:15  txqueuelen 1000  (Ethernet)
        RX packets 112  bytes 12015 (11.7 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 101  bytes 16500 (16.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

