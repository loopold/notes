# Banana Pi BPI-R3



Stack:

- MediaTek MT7986 Quad Core Processor Motherboard 2G DDR4 RAM 8G eMMC Flash Electronic Control Board Smart Router - [link][board]
- UART
- SFP GPON
- SFP RJ45
- HDD SSD NVMe M.2

![Banana set](banana-set.jpeg)


## Connection

### USB-UART converter
Set jumper to 3V
- `GND` black
- board `RX` orange -> converter `TXD` 
- board `TX` yellow -> converter `RXD`

### SFP
- `SFP1` WAN <- GPON ONU
- `SFP2` LAN <- RJ45 <- NAS


## TODO

- [ ] 2.5G LAN
- [ ] GPON
- [ ] boot from eMMC (8 GB)
- [ ] HDD PCIe NVMe M.2


## Boot

### use UART interface to login

```sh
minicom -D /dev/tty.usbserial-0001
```
Firstboot
```sh
 ----------------------------------------------------
 | OpenWrt 23.05-SNAPSHOT, r23780-6f70e09a00        |
 | Build time: 2024-03-09 13:45 CET                 |
 | Cezary Jackiewicz, https://eko.one.pl            |
 ----------------------------------------------------
 | Machine: Bananapi BPI-R3                         |
 | Uptime: 0d, 00:01:18                             |
 | Load: 0.35 0.15 0.06                             |
 | Flash: total: 88.3MB, free: 51.4MB, used: 42%    |
 | Memory: total: 1.9GB, free: 1.9GB, used: 4%      |
 | Leases: 0                                        |
 | lan: static, 192.168.10.1                        |
 | wan: dhcp, ?                                     |
 | wan6: dhcpv6, ?                                  |
 ----------------------------------------------------

root@OpenWrt:/# df -h
Filesystem                Size      Used Available Use% Mounted on
/dev/root                 8.8M      8.8M         0 100% /rom
tmpfs                   996.8M    880.0K    995.9M   0% /tmp
/dev/mmcblk0p66          88.3M     37.0M     51.4M  42% /overlay
overlayfs:/overlay       88.3M     37.0M     51.4M  42% /
tmpfs                   512.0K         0    512.0K   0% /dev
```
Updated image
```sh
----------------------------------------------------
 | OpenWrt 23.05-SNAPSHOT, r24014-76a0c2932c        |
 | Build time: 2024-07-20 09:42 CEST                |
 | Cezary Jackiewicz, https://eko.one.pl            |
 ----------------------------------------------------
 | Machine: Bananapi BPI-R3                         |
 | Uptime: 0d, 00:01:28                             |
 | Load: 0.02 0.01 0.00                             |
 | Flash:                                           |
 | Memory: total: 1.9GB, free: 1.9GB, used: 4%      |
 | Leases: 0                                        |
 | lan: static, 192.168.1.1                         |
 | wan: dhcp, ?                                     |
 | wan6: dhcpv6, ?                                  |
 ----------------------------------------------------
```
Do not touch the keys :smile:
![boot](openwrt-boot.png)

This (initramfs only) was probably the cause of the above effect.
```sh
root@OpenWrt:/# df -h
Filesystem                Size      Used Available Use% Mounted on
tmpfs                   996.8M     25.7M    971.1M   3% /
tmpfs                   996.8M    860.0K    996.0M   0% /tmp
tmpfs                   512.0K         0    512.0K   0% /dev
```

Now looks good
```sh
root@OpenWrt:/# df -h
Filesystem                Size      Used Available Use% Mounted on
/dev/root                 8.8M      8.8M         0 100% /rom
tmpfs                   996.8M   1008.0K    995.8M   0% /tmp
/dev/mmcblk0p66          88.3M     36.5M     51.8M  41% /overlay
overlayfs:/overlay       88.3M     36.5M     51.8M  41% /
tmpfs                   512.0K         0    512.0K   0% /dev
```

## Configure

### Network (temporary ethernet wan port)

Insert `SFP RJ45` connector into the SFP2.

```sh
# root@OpenWrt:/# 
vi /etc/config/network
```
Move **temporarily** `list ports 'sfp2'` to `config device | option name 'br-wan'` section. Then `/etc/init.d/network restart`

#### Access to Luci from wan port (it is local network)

```sh
uci add firewall rule
uci set firewall.@rule[-1].name=luciWAN
uci set firewall.@rule[-1].src=wan
uci set firewall.@rule[-1].src_ip=192.168.101.12
uci set firewall.@rule[-1].target=ACCEPT
uci set firewall.@rule[-1].proto=tcp
uci set firewall.@rule[-1].dest_port=80

uci commit firewall   # write changes to /etc/config/firewall
/etc/init.d/firewall restart
```

Use a Sysupgrade image to update a router that already runs OpenWrt. The image can be used with the LuCI web interface or the terminal.

```
Using ethernet@15100000 device
TFTP from server 192.168.1.254; our IP address is 192.168.1.1
Filename 'openwrt-mediatek-filogic-bananapi_bpi-r3-initramfs-recovery.itb'.
Load address: 0x46000000
Loading: *
ARP Retry count exceeded; starting again
Wrong Image Format for bootm command
ERROR: can't get kernel image!
```

### SSH

- add a key
  ```sh
  cat > /etc/dropbear/authorized_keys  # ssh-copy-id doesn't work for now
  ```
- disable SSH password authentication
  ```sh
  uci show dropbear
  uci set dropbear.@dropbear[0].PasswordAuth='off'
  uci commit dropbear
  service dropbear restart
  ```

I need login through WAN interface
```sh
uci add firewall rule
uci set firewall.@rule[-1].name=sshWAN
uci set firewall.@rule[-1].src=wan
uci set firewall.@rule[-1].src_ip=192.168.101.12
uci set firewall.@rule[-1].target=ACCEPT
uci set firewall.@rule[-1].proto=tcp
uci set firewall.@rule[-1].dest_port=22

uci commit firewall   # write changes to /etc/config/firewall
/etc/init.d/firewall restart
```



### Tools

```sh
# root@OpenWrt:/# 
opkg update

opkg install lsblk pciutils ethtool vim parted
```

### Boot from onboard eMMC

According to the [Banana Pi Wiki][wiki banana-pi]
>  Before burning image to eMMC, please prepare a SD card with flashed bootable image and a USB disk. Let's take OpenWrt image
> - mtk-bpi-r3-SD-WAN1-SFP1-20220619-single-image.img, 
> - mtk-bpi-r3-NAND-WAN1-SFP1-20220619-single-image.bin, 
> - bl2_emmc.img, 
> - mtk-bpi-r3-EMMC-WAN1-SFP1-20220619-single-image.img

#### Need to build packages
```sh
# loop@ubuntu-zamiast  ~  
sudo apt install gcc binutils bzip2 flex python3 perl make grep diffutils unzip gawk subversion libz-dev libc-dev rsync libncurses5-dev g++
git clone https://github.com/BPI-SINOVOIP/BPI-R3-OPENWRT-V21.02.3.git
```


Boot from SD card
```sh
fw_setenv bootcmd "run ubi_init ; env default bootcmd ; saveenv ; reset"
reboot
```

Wait for the reboot to complete. Then switch off the power supply. Set jumper 2/B to the low position. After power on, the unit will **boot from NAND**

After boot from NAND copy it eMMC

Perform sysupgrade because `fw_setenv` command is not found. Copy `squashfs-sysupgrade.itb` file to usb drive
```sh
root@OpenWrt:/# sysupgrade -n /mnt/sda1/luci-23.05-snapshot-r24014-76a0c2932c-mediatek-filogic-bananapi_bpi-r3-squashfs-sysupgrade.itb
```

```sh
fw_setenv bootcmd "run emmc_init ; env default bootcmd ; saveenv ; saveenv ; reset"
reboot
# /bin/ash: fw_setenv: not found

opkg update && opkg install uboot-envtools mmc-utils
```

```sh
...
```

```sh
mmcblk0      179:0    0  58.2G  0 disk
├─mmcblk0p1  179:1    0     4M  0 part
├─mmcblk0p2  179:2    0   512K  0 part
├─mmcblk0p3  179:3    0     2M  0 part
├─mmcblk0p4  179:4    0     4M  0 part
├─mmcblk0p5  179:5    0    32M  0 part
├─mmcblk0p6  179:6    0    20M  0 part
├─mmcblk0p7  179:7    0   104M  0 part
├─mmcblk0p65 259:0    0   8.7M  1 part /rom
└─mmcblk0p66 259:1    0  90.3M  0 part /overlay
```
```
Using /dev/mmcblk0
(parted) print free
Error: The backup GPT table is corrupt, but the primary appears OK, so that will
be used.
Number  Start   End     Size    File system  Name        Flags
 1      17.4kB  4194kB  4177kB               bl2         hidden, legacy_boot
 2      4194kB  4719kB  524kB                ubootenv    hidden
 3      4719kB  6816kB  2097kB               factory     hidden
 4      6816kB  11.0MB  4194kB               fip         boot, hidden, esp
        11.0MB  12.6MB  1573kB  Free Space
 5      12.6MB  46.1MB  33.6MB               recovery    boot, hidden, esp
 6      46.1MB  67.1MB  21.0MB               install     boot, hidden, esp
 7      67.1MB  176MB   109MB                production
        176MB   62.5GB  62.3GB  Free Space
```
```sh
opkg update
opkg install uvol autopart
uvol create userdata $(uvol free) rw
```
```sh
# new:
/dev/mmcblk0p8 344064 122075102 121731039   58G Linux LVM

mkfs.ext4 /dev/mmcblk0p8
mkdir /mnt/emmc-root
mount /dev/mmcblk0p8 /mnt/emmc-root
opkg install rsync
rsync -a /overlay/* /mnt/emmc-root


setenv bootargs 'console=ttyS0,115200 root=/dev/mmcblk1p1 rootfstype=ext4 rootwait'
saveenv

```

### nvme

## Links

- [wiki banana-pi][wiki banana-pi]
- [wiki junicast](https://wiki.junicast.de/en/junicast/review/bananapi-BPI-R3)
- [firmware](https://dl.eko.one.pl/firmware/?version=luci-23.05-SNAPSHOT&target=mediatek%2Ffilogic&id=bananapi_bpi-r3)
- [Rafał Bielawski @YouTube](https://www.youtube.com/watch?v=PmTMmande14)
- [Accessing LuCI web interface securely](https://openwrt.org/docs/guide-user/luci/luci.secure)
- [OpenWrt Support for Banana Pi BPI-R3](https://forum.openwrt.org/t/openwrt-support-for-banana-pi-bpi-r3/154294/134?page=3)

[wiki banana-pi]: https://wiki.banana-pi.org/Getting_Started_with_BPI-R3
[board]: https://pl.aliexpress.com/item/1005004886608696.html?spm=a2g0o.order_list.order_list_main.5.43e91c24iRBFf1&gatewayAdapt=glo2pol
[OpenWrt config]: https://openwrt.org/docs/guide-user/base-system/start
