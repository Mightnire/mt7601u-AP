## Orange Pi - MT7601U Access Point and Station Mode support. 

根据 [Vyacheslav-S/mt7601u-AP](https://github.com/Vyacheslav-S/mt7601u-AP) 仓库思路，对 OpenWrt 中如下文件进行修改，将编出来的 mt7601u.ko 覆盖到 /lib/modules/6.12.112/mt7601u.ko（必须用同一份 backports 源码 + 同 SDK 编，符号 CRC 才和系统里的 cfg80211.ko/mac80211.ko 对得上）。即可在 OpenWrt 中使用 MT7601U 网卡开启无线网。参考 DeepSeek V4.1-Flash-1017 编写的 [GUIDE.md](./GUIDE.md)。

```
--- a/drivers/net/wireless/mediatek/mt7601u/init.c
+++ b/drivers/net/wireless/mediatek/mt7601u/init.c
@@ -609,7 +609,8 @@
 	SET_IEEE80211_PERM_ADDR(hw, dev->macaddr);
 
 	wiphy->features |= NL80211_FEATURE_ACTIVE_MONITOR;
-	wiphy->interface_modes = BIT(NL80211_IFTYPE_STATION);
+	wiphy->interface_modes = BIT(NL80211_IFTYPE_STATION) |
+				 BIT(NL80211_IFTYPE_AP);
 	wiphy->flags |= WIPHY_FLAG_SUPPORTS_TDLS;
 
 	wiphy_ext_feature_set(wiphy, NL80211_EXT_FEATURE_CQM_RSSI_LIST);
```

<img width="763" height="195" alt="image" src="https://github.com/user-attachments/assets/9d05e0b8-c08c-400b-b519-b110e2efc188" />


This fixes the driver issues for mt7601u

Tested on Orange Pi PC2, Armbian 23.02.2 (kernel 5.15.93).


### Build module:
```
$ make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
    # Remove device
$ sudo rmmod mt7601u
$ sudo insmod ./mt7601u.ko
    # Insert device
```
``` 
Check there are no errors in dmesg and interface appeared in ip link, check connection stability.
To make change persistent till next kernel upgrade: 
backup original module and replace with compiled.
To find out where is original module run modinfo mt7601u
(view string filename: /lib/modules/KERNEL_VERSION/kernel/drivers/net/wireless/mediatek/mt7601u/mt7601u.ko).
```
```
$ sudo iw list
..............
  Supported interface modes:
      * managed
      * AP
      * AP/VLAN
      * monitor
.................      
```
