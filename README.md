# ImmortalWrt 24.10 / Linux 6.6 · KST WF3000A & Redmi AX6000

本仓库是 **6.6 内核工程**（源码 + 云编译一体，分支 `openwrt-24.10-6.6`），板级移植已直接写进源码，Actions 里编译即可，不用补丁脚本。无线用 MTK 闭源 `mt_wifi`（这份源码没有 mt76）。

另有 **5.4 内核工程**（hanwckf 21.02，多一台 AX3000T）：[编译工程](https://github.com/wangwang623/immortalwrt-hanwckf-firmware-build) · [源码 fork](https://github.com/wangwang623/immortalwrt-mt798x)。同一机型可在两个工程间互刷（布局要对应），换内核建议不保留配置。

## 机型与布局

| | KST WF3000A | Redmi AX6000 |
|---|---|---|
| SoC / 内存 | MT7981B / 256 MB | MT7986 / 512 MB |
| U-Boot 布局 | `expand(114m)` | `immortalwrt-110m` |
| U-Boot Web 地址 | `192.168.1.1` | `192.168.31.1` |
| 配置文件 | `defconfig/wf3000a.config` | `defconfig/ax6000-hanwckf-110m.config` |
| 编译流 | `build-kst-wf3000a.yml` | `build-redmi-ax6000.yml` |

## 固件里有什么

一套最小完整系统：MTK 闭源无线驱动 + HNAT 硬件加速、系统工具与库、LuCI 中文、Argon 主题、zram-swap。**不含第三方插件**。

- `kmod-*` 和无线驱动与内核绑定，必须编进固件，事后装不上；中文、Argon 可以后装。
- zram 默认 **64M / lzo**，防内存突发占用；`kmod-lib-lz4` 也已编入，LuCI 里改选 lz4 才生效。

首次启动统一写入：LAN `192.168.31.1`、DHCP `100~249`（start 100 / limit 150）。之后以 LuCI / UCI 设置为准；重刷或 `sysupgrade -n` 会再写一次。想按自己习惯改，只动 `base-files/etc/uci-defaults/99-default-settings` 一个文件。

## 编译与刷机

1. 仓库 → **Actions → 对应 workflow → Run workflow**，无输入项；源码分支固定，编译成功自动发 Release，产物在 Artifacts / Release。
2. WF3000A：U-Boot Web 布局选 **114m**，传 `factory.bin`。建议先传 `initramfs-kernel.bin` 内存启动，确认网口、无线正常后再刷正式包。
3. AX6000：布局选 **immortalwrt-110m**，传 `*-initramfs-factory.ubi`，进系统后 `sysupgrade -n *-squashfs-sysupgrade.bin`。
4. 刷完管理地址是 `192.168.31.1`。作二级路由时若上级也是 31.1 网段，先把本机 LAN 改成别的网段。

三种文件不要混用：initramfs 验机、factory 首次刷入、sysupgrade 系统内升级。U-Boot 与救砖资料见文末。

## 两点提醒

- **NMBM**：WF3000A 用 Winbond 闪存，与 U-Boot 的 OOB 参数对不上时，会出现「同一份固件、有的机器正常，有的反复重启或一直停在 U-Boot Web」。本仓库 DTS 已内置 `mediatek,bmt-mtd-overridden-oobsize = <64>` 对齐，**别删**。刷正式包前用 initramfs 看 `dmesg | grep -i nmbm`，应出现 `NMBM has been successfully attached`。**5.4 工程不要加这一行**。
- **MAC**：每台机器从自己的 factory 分区推导有线和无线 MAC，不写死在固件里，公开分发不会撞 MAC。

## 验收

```sh
uci get network.lan.ipaddr                        # 192.168.31.1
uci get dhcp.lan.start; uci get dhcp.lan.limit   # 100 / 150
/etc/init.d/zram status                           # 64 MiB / lzo
dmesg | grep -i nmbm                              # WF3000A 必须 attached
dmesg | grep -iE 'mt_wifi|warp|conninfra'        # 无线驱动已加载
```

## 参考

| | |
|---|---|
| 上游 6.6 源码 | https://github.com/padavanonly/immortalwrt-mt798x-6.6 |
| 5.4 编译工程 / 源码 | https://github.com/wangwang623/immortalwrt-hanwckf-firmware-build · https://github.com/wangwang623/immortalwrt-mt798x |
| U-Boot 预编译 / 源码 | https://github.com/VIKINGYFY/UBOOT-CI · https://github.com/Yuzhii0718/bl-mt798x-dhcpd |
| 项目介绍 / AX3000T 教程 | https://cmi.hanwckf.top/p/immortalwrt-mt798x/ · https://okhk.net/xiaomi-ax3000t-router-with-hanwckf-immortalwrt |
| 旧仓库（索引页，勿叠加使用） | https://github.com/wangwang623/KST-WF3000A-ImmortalWrt |
