# 在 OpenWrt 上把 MT7601U 变成可用 AP（完整操作手册）

**适用环境**（本手册实测通过的环境）：

- 设备：x86_64 小主机 / 虚拟机，OpenWrt **25.12.6**（r33311-981c978eff），内核 **6.12.112**
- 网卡：MT7601U USB（`148f:7601`），驱动后接口名 `phy0-ap0`
- 编译机：Debian 10 / Ubuntu 等 x86_64 Linux
- 结果：Windows 11、iPhone 都能连上，能正常上网

---

## 0. 一页速查（老手直接看这里）

```bash
# 1) 准备 SDK（与设备同版本）
wget https://downloads.openwrt.org/releases/25.12.6/targets/x86/64/openwrt-sdk-25.12.6-x86-64_gcc-14.3.0_musl.Linux-x86_64.tar.zst
zstd -d openwrt-sdk-*.tar.zst -o sdk.tar && tar xf sdk.tar
cd openwrt-sdk-*

# 2) 放入 mac80211 包源码（版本要与 SDK 的 feeds 锁一致，见 3.3）
cp -a /path/to/package/kernel/mac80211 package/kernel/

# 3) 打 AP 模式补丁（见 3.4，文件内容在附录 B）
mkdir -p package/kernel/mac80211/patches/mt7601u
cp mt7601u-add-ap-mode.patch package/kernel/mac80211/patches/mt7601u/002-mt7601u-add-ap-mode.patch

# 4) 配置并编译
make defconfig
echo 'CONFIG_PACKAGE_kmod-mac80211=m' >> .config
echo 'CONFIG_PACKAGE_kmod-mt7601u=m'  >> .config
make package/kernel/mac80211/compile -j8

# 5) 取出模块
find build_dir -path '*kmod-mt7601u*' -name mt7601u.ko

# 6) 部署到设备（见第 4 节），配置 /etc/config/wireless，重启
```

核心就一句话：**不要用厂商 DPA 驱动，用 OpenWrt 自带 mac80211 版 mt7601u，
给它补上 `NL80211_IFTYPE_AP`。**

---

## 1. 背景：两条路线，只有一条能走通

MT7601U 是一颗很老的 2.4G 单流 USB 网卡。Linux 里有两套驱动：

| 路线 | 驱动 | 结论 |
|---|---|---|
| A | 厂商 DPA 驱动（[Anthony96922/mt7601u-ap](https://github.com/Anthony96922/mt7601u-ap)） | **在 6.12 上用不了**：能起 AP、能发 beacon、能回 probe response，但客户端始终不发 auth 帧，连不上 |
| B | mac80211 版 mt7601u（[Vyacheslav-S/mt7601u-AP](https://github.com/Vyacheslav-S/mt7601u-AP) 的思路）+ OpenWrt backports | **实测可用**：hostapd 正常起 AP，Windows/iPhone 都能关联、拿 IP、上网 |

路线 B 的好处：它就是标准的 mac80211 驱动，配置走 UCI/LuCI/hostapd，
加密、802.11w、多 SSID 这些都能用；而且 OpenWrt 自带的 `kmod-mt7601u`
本来就是它，我们只是**重新编译并打开 AP 模式**。

> 路线 A 的完整记录放在附录 A，供好奇/排错参考，**不建议再用**。

---

## 2. 先确认设备状态

SSH 上路由器（本例 root 密码为空，直接回车）：

```bash
ssh root@192.168.1.1
```

确认几件事：

```bash
cat /etc/openwrt_release | head -3      # 版本，本手册是 25.12.6
uname -r                                 # 内核，本手册是 6.12.112
lsusb | grep -i 7601                     # 应看到 148f:7601
ls /lib/firmware/mediatek/mt7601u.bin    # 固件文件应存在（mac80211 驱动要用）
```

固件文件如果不存在，先装 `kmod-mt7601u` 对应的固件包，或从
`linux-firmware` 里把 `mt7601u.bin` 放到 `/lib/firmware/mediatek/`。

---

## 3. 编译带 AP 模式的 mt7601u.ko

### 3.1 编译机装依赖

Debian/Ubuntu：

```bash
sudo apt-get update
sudo apt-get install -y build-essential git zstd xz-utils bc rsync wget \
     python3 unzip libncurses5-dev libssl-dev flex bison gawk python3-distutils
```

> Debian 10 上如果 `make defconfig` 报
> `Please install GNU 'awk'` / `Please install the Python3 distutils module`，
> 就是缺 `gawk`、`python3-distutils` 这两个包。

### 3.2 下载与设备同版本的 SDK

SDK 版本必须和设备一致（同内核、同 gcc），否则编出来的模块加载不了。

```bash
mkdir -p ~/sdk && cd ~/sdk
wget https://downloads.openwrt.org/releases/25.12.6/targets/x86/64/openwrt-sdk-25.12.6-x86-64_gcc-14.3.0_musl.Linux-x86_64.tar.zst
zstd -d openwrt-sdk-*.tar.zst -o sdk.tar
tar xf sdk.tar
cd openwrt-sdk-*/
```

其他设备的对应 SDK 在 `https://downloads.openwrt.org/releases/<版本>/targets/<target>/`，
找文件名里带 `sdk` 的那个。

### 3.3 取 `package/kernel/mac80211` 源码

SDK 里默认**没有** mac80211 这个包，要从 openwrt 仓库取，而且要用 SDK
`feeds.conf.default` 里锁定的那个 commit（`base ... ^<commit>`），保证和系统里的
backports 版本一致。

```bash
cd ~
git clone --filter=blob:none --no-checkout https://github.com/openwrt/openwrt.git owrt-src
cd owrt-src
git config core.sparseCheckout true
mkdir -p .git/info
printf 'package/kernel/mac80211/\n' > .git/info/sparse-checkout
# 换成本机 SDK 里 feeds.conf.default 中 base 后面的 commit：
git checkout 64900e756efa667774214f9c6b17c113222e01f7

cp -a package/kernel/mac80211 ~/sdk/openwrt-sdk-*/package/kernel/
```

查看本机 SDK 锁定的 commit：

```bash
grep '^src-git.*base' ~/sdk/openwrt-sdk-*/feeds.conf.default
# 形如：src-git --root=package base https://git.openwrt.org/openwrt/openwrt.git^64900e75...
#                                                                              ^^^^^^^^^^ 这个
```

### 3.4 加 AP 模式补丁

新建 `package/kernel/mac80211/patches/mt7601u/002-mt7601u-add-ap-mode.patch`，
内容见 **附录 B**。就一处改动：在 `init.c` 里把 `wiphy->interface_modes`
加上 `BIT(NL80211_IFTYPE_AP)`。

（同目录下已经有 `001-wifi-mt7601u-update-firmware-path.patch`，是 OpenWrt 自己
改固件路径的，别动它。）

### 3.5 配置 SDK

```bash
cd ~/sdk/openwrt-sdk-*/
make defconfig
echo 'CONFIG_PACKAGE_kmod-mac80211=m' >> .config
echo 'CONFIG_PACKAGE_kmod-mt7601u=m'  >> .config
```

> 包管理从 opkg 换成 apk 之后，kmod 仍然编译成 `.ko`，这里只是让 SDK 把
> mac80211 包和 mt7601u 变体都编出来。

### 3.6 编译

```bash
make package/kernel/mac80211/compile -j$(nproc)
```

第一次会先编一遍内核 kmod 树，再编 backports，8 核大约 20–40 分钟。
backports 源码包（`backports-6.18.55.tar.zst`）会自动下载到 `dl/`；
如果编译机不能上外网，可以提前把 tarball 放进 `dl/`。

### 3.7 取出编好的模块

```bash
find build_dir -path '*kmod-mt7601u*' -name mt7601u.ko
# 典型路径：
# build_dir/target-x86_64_musl/linux-x86_64/mac80211-regular/backports-6.18.55/.pkgdir/kmod-mt7601u/lib/modules/6.12.112/mt7601u.ko
```

确认一下身份（本手册产物）：

```bash
ls -l  mt7601u.ko        # 668280 字节
md5sum mt7601u.ko        # 984a7a2b2bf4063eeea0a5b83eda94a7
strings mt7601u.ko | grep -E '^vermagic|^depends'
# vermagic=6.12.112 SMP mod_unload
# depends=mac80211,cfg80211,compat
```

`vermagic` 必须等于 `uname -r`，`depends` 必须是 `mac80211,cfg80211,compat`。

---

## 4. 部署到设备

### 4.1 传文件

OpenWrt 的 Dropbear 不一定带 sftp，`scp` 可能报
`subsystem request failed`。用 `cat` 走 ssh 最稳：

```bash
ssh root@192.168.1.1 'cat > /tmp/mt7601u.ko' < mt7601u.ko
ssh root@192.168.1.1 'ls -l /tmp/mt7601u.ko'
```

### 4.2 备份并替换模块

```bash
ssh root@192.168.1.1

# 备份原厂模块
cp /lib/modules/$(uname -r)/mt7601u.ko /root/mt7601u.ko.stock

# 换成带 AP 模式的
cp /tmp/mt7601u.ko /lib/modules/$(uname -r)/mt7601u.ko
sync
```

### 4.3 写 `/etc/config/wireless`

先看 USB 网卡的 path（不同机器不一样）：

```bash
ls -l /sys/class/ieee80211/ 2>/dev/null
# 或者用之前备份里的 path：
# pci0000:00/0000:00:11.0/0000:02:03.0/usb2/2-1/2-1:1.0
```

然后写配置（也可以直接用 `uci` 命令，见 4.4）：

```bash
cat > /etc/config/wireless <<'EOF'
config wifi-device 'radio0'
	option type 'mac80211'
	option path 'pci0000:00/0000:00:11.0/0000:02:03.0/usb2/2-1/2-1:1.0'
	option band '2g'
	option channel '6'
	option htmode 'HT20'
	option country 'CN'
	option cell_density '0'

config wifi-iface 'default_radio0'
	option device 'radio0'
	option network 'lan'
	option mode 'ap'
	option ssid 'MT7601U-AP'
	option encryption 'psk2'
	option key 'mt7601uap123'
EOF
```

说明：

- `option network 'lan'`：把 AP 接口桥接进 `br-lan`，DHCP、防火墙、NAT 全部复用 LAN，
  客户端拿到的就是 `192.168.1.x`，直接能上网。
- `option channel '6'` + `option htmode 'HT20'`：2.4G 最稳的组合。
  想用 13 信道也可以，但 **13 信道不能配 HT40**（hostapd 会自动退回 HT20）。
- `option encryption 'psk2'` 就是 WPA2-PSK；要开放网络改成 `none`。

### 4.4 用 uci 改（等价写法）

```bash
uci set wireless.radio0.channel='6'
uci set wireless.radio0.htmode='HT20'
uci set wireless.default_radio0.ssid='MT7601U-AP'
uci set wireless.default_radio0.encryption='psk2'
uci set wireless.default_radio0.key='mt7601uap123'
uci commit wireless
```

### 4.5 如果之前装过厂商驱动，清掉它

```bash
# 停用厂商驱动的开机脚本
/etc/init.d/mt7601uap disable 2>/dev/null
rm -f /etc/rc.d/S19mt7601uap

# 把厂商模块挪走，避免误加载
mv /lib/modules/mt7601Uap.ko /root/ 2>/dev/null

# 把 ra0 从 br-lan 的端口列表里删掉（如果加过）
uci -q del_list network.@device[0].ports='ra0'
uci commit network

# 确保 mac80211 的 mt7601u 会开机自动加载
ls /etc/modules.d/mt7601u || echo mt7601u > /etc/modules.d/mt7601u
```

### 4.6 重启

```bash
sync
reboot
```

---

## 5. 验证

### 5.1 驱动和接口

```bash
lsmod | grep mt7601u
iw dev
```

正常应看到：

```
phy#0
	Interface phy0-ap0
		ifindex 5
		wdev 0x1
		addr e8:4e:06:93:52:b5
		ssid MT7601U-AP
		type AP
		channel 6 (2437 MHz), width: 20 MHz, center1: 2437 MHz
		txpower 20.00 dBm
```

`type AP` 就说明补丁生效了。接口名是 `phy0-ap0`（netifd 的命名）。

### 5.2 hostapd 状态

```bash
logread | grep hostapd | tail
```

应看到：

```
phy0-ap0: interface state COUNTRY_UPDATE->ENABLED
phy0-ap0: AP-ENABLED
```

### 5.3 桥接

```bash
ls /sys/class/net/br-lan/brif/
# eth0  phy0-ap0
```

### 5.4 客户端连接

手机/电脑搜 `MT7601U-AP`，密码 `mt7601uap123`。连上后：

```bash
# 看关联的客户端
iw dev phy0-ap0 station dump

# 看 DHCP 租约（拿到的 IP）
cat /tmp/dhcp.leases

# 路由器侧 ping 客户端
ping -c 3 192.168.1.x
```

`station dump` 里应该有 `authorized: yes / authenticated: yes / associated: yes`，
`signal` 在 -30 ~ -40 dBm 左右。

### 5.5 确认真的能上网

在路由器上看 NAT 会话（把 IP 换成客户端拿到的地址）：

```bash
grep <客户端IP> /proc/net/nf_conntrack | head
```

能看到类似：

```
src=192.168.1.225 dst=114.110.96.26 dport=80
  -> src=114.110.96.26 dst=192.168.198.136 (WAN 地址)
```

就说明流量确实经 AP → br-lan → NAT → WAN 出去了。

---

## 6. 日常维护

### 6.1 改 SSID / 密码 / 信道

LuCI：网络 → 无线，直接改。
命令行：

```bash
uci set wireless.default_radio0.ssid='MyAP'
uci set wireless.default_radio0.key='newpassword'
uci set wireless.radio0.channel='11'
uci commit wireless
wifi reload
```

### 6.2 看当前连接

```bash
iw dev phy0-ap0 station dump
cat /tmp/dhcp.leases
```

### 6.3 升级 OpenWrt 之后

覆盖进 `/lib/modules/<版本>/` 的 `mt7601u.ko` 会随系统升级丢失，
升级后要**用新版本的 SDK 重新编一次**（第 3 节流程不变，只换 SDK 版本），
再覆盖回去。原厂模块备份在 `/root/mt7601u.ko.stock`。

### 6.4 想恢复原样

```bash
cp /root/mt7601u.ko.stock /lib/modules/$(uname -r)/mt7601u.ko
# 再把 /etc/config/wireless 删掉/改回原样，重启
```

---

## 7. 排错

| 现象 | 原因 / 处理 |
|---|---|
| `insmod: disagrees about version of symbol` | 模块和系统的 `cfg80211.ko`/`mac80211.ko` 不是同一份源码/配置编的。必须用**同一版本 SDK + 同一 backports 源码**重编 |
| `iw dev` 只有 `wlan0`，`iw list` 里没有 `AP` | 加载的还是原厂模块，或补丁没打上。`md5sum` 对一下，确认 `/lib/modules/$(uname -r)/mt7601u.ko` 是新文件，重启 |
| `iw list` 里有 AP，但 hostapd 起不来 | `logread \| grep hostapd` 看报错；确认 `/etc/config/wireless` 的 `path` 和实际 USB 路径一致；确认 `wpad`/`hostapd` 已安装 |
| AP 起来了但客户端连不上 | 先确认 `type AP` 和 `AP-ENABLED`；再看 `iw dev phy0-ap0 station dump` 有没有关联记录。没有的话用手机试，排除电脑的 WLAN 黑名单 |
| 连上了但拿不到 IP | `ls /sys/class/net/br-lan/brif/` 里要有 `phy0-ap0`；检查 `dnsmasq` 是否在跑、`option network 'lan'` 是否写对 |
| 能拿 IP 但不能上网 | 检查防火墙 zone：`lan` 要包含 `br-lan`、`wan` 要开 masq；`/etc/config/firewall` 默认就是对的 |
| dmesg 有 `Vendor request req:07 off:0730 failed:-110` | MT7601U 的 RXIQ 校准超时，**不影响使用**，可以忽略 |
| `scp` 传文件失败 | Dropbear 没开 sftp，用 `ssh root@ip 'cat > /tmp/x' < x` |

---

## 附录 A：厂商 DPA 驱动路线（已放弃，留作记录）

一开始按 [Anthony96922/mt7601u-ap](https://github.com/Anthony96922/mt7601u-ap)
（MediaTek/Ralink 的 SoftAP 厂商驱动）来做。它在 6.12 上需要 5 处小改才能编译/运行：

1. `asm/unaligned.h` 在 6.12 被删 → 改 `<linux/unaligned.h>`
2. `skb->tail` 已是偏移量（`sk_buff_data_t`）→ 宏改成 `head + tail` / `skb_set_tail_pointer()`
3. `rt_linux.c` 里 7 处把 `skb->tail` 当指针用 → 改 `skb_tail_pointer(skb)`
4. 内核没开 `CONFIG_CFG80211` 时没有 `wireless_send_event` → 6.12 上把 WEXT 事件调用编掉
5. `RTMPSetCountryCode()` 对非 NUL 结尾的 `CountryCode[3]` 调 `strlen()`，FORTIFY 会 WARN → 改判首字节

编译（用 SDK 的内核树）：

```bash
SDK=~/sdk/openwrt-sdk-*
K=$SDK/build_dir/target-x86_64_musl/linux-x86_64/linux-6.12.112
TC=$SDK/staging_dir/toolchain-x86_64_gcc-14.3.0_musl/bin/x86_64-openwrt-linux-musl-
cd mt7601u-ap
make LINUX_SRC="$K" ARCH=x86_64 CROSS_COMPILE="$TC" KCFLAGS="-Wno-error" -j8
# 产物 os/linux/mt7601Uap.ko
```

部署后 `ra0` 能起来、能桥接、beacon 和 probe response 都正常（客户端也会 ACK），
但**客户端始终不发 auth 帧**，Windows 和 Android 都一样；换成最简单的开放 11b、
去掉 WMM/HT/Country IE/Broadcom 兼容 IE、强制指定 BSSID 连接，现象完全一样。
抓包确认原因是客户端不进入关联阶段，属于该厂商 SoftAP 协议栈在 6.12 上的兼容性问题。

结论：**6.12 上不要用厂商驱动，走 mac80211 路线。**

（厂商驱动的 6.12 移植补丁和 .ko 都留在本仓库 `patches/mt7601u-ap-6.12.patch`、
`build/mt7601Uap.ko`，仅供对照。）

---

## 附录 B：AP 模式补丁内容

文件：`patches/mt7601u-add-ap-mode.patch`

```diff
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

打完补丁后 `iw list` 的 `Supported interface modes` 会多出 `AP`、`AP/VLAN`。

---

## 附录 C：本次用到的文件清单

| 文件 | 说明 |
|---|---|
| `patches/mt7601u-add-ap-mode.patch` | 本手册用的 AP 模式补丁 |
| `build/mt7601u-6.12.112-ap.ko` | 已编好的模块（md5 `984a7a2b…`），可直接用 |
| `build/wireless-mac80211` | `/etc/config/wireless` 参考配置 |
| `patches/mt7601u-ap-6.12.patch` | 厂商驱动的 6.12 移植补丁（留档，不推荐） |
| `build/mt7601Uap.ko` | 厂商驱动模块（留档，不推荐） |
| `scripts/debian/*.sh` | 编译/检查脚本 |
| `scripts/openwrt/*.sh` | 部署/切换/诊断脚本 |

---

## 附录 D：关键参数速查

| 项 | 值 |
|---|---|
| 设备系统 | OpenWrt 25.12.6 r33311-981c978eff，x86/64 |
| 内核 | 6.12.112 |
| SDK | openwrt-sdk-25.12.6-x86-64_gcc-14.3.0_musl |
| backports | 6.18.55 |
| 模块 | `/lib/modules/6.12.112/mt7601u.ko`（668280 字节） |
| 模块 md5 | `984a7a2b2bf4063eeea0a5b83eda94a7` |
| vermagic | `6.12.112 SMP mod_unload` |
| depends | `mac80211,cfg80211,compat` |
| AP 接口 | `phy0-ap0`，桥接进 `br-lan` |
| SSID / 密码 | `MT7601U-AP` / `mt7601uap123` |
| 信道 | 6（HT20）；也可 13（会回落 HT20） |
