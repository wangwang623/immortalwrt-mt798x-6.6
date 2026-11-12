# ImmortalWrt 24.10 / Linux 6.6 · KST WF3000A & Redmi AX6000

本仓库是 **6.6 内核工程**（源码 + 云编译一体，分支 `openwrt-24.10-6.6`），板级移植已直接写进源码，Actions 里编译即可，不用补丁脚本。无线用 MTK 闭源 `mt_wifi`（这份源码没有 mt76）。

另有 **5.4 内核工程**（hanwckf 21.02，多一台 AX3000T）：[编译工程](https://github.com/wangwang623/immortalwrt-hanwckf-firmware-build) · [源码 fork](https://github.com/wangwang623/immortalwrt-mt798x)。同一机型可在两个工程间互刷（布局要对应），换内核建议不保留配置。

## 机型与布局

| | KST WF3000A | Redmi AX6000 |
|---|---|---|
| SoC / 内存 | MT7981B / 256 MB | MT7986 / 512 MB |
| U-Boot 布局 | `expand(114m)` | `immortalwrt-110m` |
| U-Boot Web（刷机页） | `192.168.1.1` | `192.168.31.1` |
| 固件管理地址 | `192.168.6.1` | `192.168.6.1` |
| 配置文件 | `defconfig/wf3000a.config` | `defconfig/ax6000-hanwckf-110m.config` |
| 编译流 | `build-kst-wf3000a.yml` | `build-redmi-ax6000.yml` |

## 固件里有什么

一套最小完整系统：MTK 闭源无线驱动 + HNAT 硬件加速、系统工具与库、LuCI 中文、Argon 主题、zram-swap，外加 **TurboACC 状态页**（网络 → Turbo ACC，用来看 HNAT 是否生效、PPE 绑定数）。**不含其它第三方插件**。

- `kmod-*` 和无线驱动与内核绑定，必须编进固件，事后装不上；中文、Argon、TurboACC 可以后装。
- zram-swap 已编入，容量与算法走源码默认；`kmod-lib-lz4` 也已编入，LuCI 里改选 lz4 才生效。需要固定容量，刷完用 UCI 设一次。
- TurboACC 是软包：主包、汉化包和源码 Makefile 里 16 个依赖全部显式 `=y`，校验只查主包（缺了只警告继续编译），依赖不校验。
- ⚠️ 源码 `package/mtk/applications/luci-app-turboacc-mtk/Makefile` 把 `+luci-app-ttyd` 和 15 个页面用不到的 kmod（wireguard / btrfs / bonding 等）写成强制依赖，选 TurboACC 就会一并编进固件；页面只读 `/sys/kernel/debug/hnat/hnat_stats`，并不调用它们。不想带 ttyd 可精简该 Makefile 的 `LUCI_DEPENDS`。
- `.config` 里硬包及可查的一层依赖都显式 `=y`：缺基本系统硬项中止编译，中文 / Argon / lz4 / TurboACC 缺了只警告。

LAN、DHCP、zram **全部沿用源码默认**（LAN `192.168.6.1`），编译流不再 sed 改源码、也不注入任何 uci-defaults。想改网段 / DHCP 范围 / zram 容量，刷完用 UCI 设一次即可，见文末。

## 编译与刷机

1. 仓库 → **Actions → 对应 workflow → Run workflow**，无输入项；编译环境 `ubuntu-24.04`，源码分支固定，编译成功自动发 Release，产物在 Artifacts / Release。
2. WF3000A：U-Boot Web 布局选 **114m**，传 `factory.bin`。建议先传 `initramfs-kernel.bin` 内存启动，确认网口、无线正常后再刷正式包。
3. AX6000：布局选 **immortalwrt-110m**，传 `*-initramfs-factory.ubi`，进系统后 `sysupgrade -n *-squashfs-sysupgrade.bin`。
4. 刷完管理地址是 `192.168.6.1`（源码默认）。作二级路由时若上级也是 6.1 网段，先把本机 LAN 改成别的网段。

三种文件不要混用：initramfs 验机、factory 首次刷入、sysupgrade 系统内升级。U-Boot 与救砖资料见文末。

## 两点提醒

- **NMBM**：WF3000A 用 Winbond 闪存，与 U-Boot 的 OOB 参数对不上时，会出现「同一份固件、有的机器正常，有的反复重启或一直停在 U-Boot Web」。本仓库 DTS 已内置 `mediatek,bmt-mtd-overridden-oobsize = <64>` 对齐，**别删**。刷正式包前用 initramfs 看 `dmesg | grep -i nmbm`，应出现 `NMBM has been successfully attached`。**5.4 工程不要加这一行**。
- **MAC**：每台机器从自己的 factory 分区推导有线和无线 MAC，不写死在固件里，公开分发不会撞 MAC。

## 验收

```sh
uci get network.lan.ipaddr                        # 源码默认 192.168.6.1
uci get dhcp.lan.start; uci get dhcp.lan.limit   # 源码默认值
/etc/init.d/zram status                           # 容量按内存自动算 / lzo
dmesg | grep -i nmbm                              # WF3000A 必须 attached
dmesg | grep -iE 'mt_wifi|warp|conninfra'        # 无线驱动已加载
```

想按自己的习惯固定（示例）：

```sh
uci set network.lan.ipaddr=192.168.31.1; uci commit network
uci set dhcp.lan.start=100; uci set dhcp.lan.limit=150; uci commit dhcp
uci set system.@system[0].zram_size_mb=64
uci set system.@system[0].zram_comp_algo=lzo
uci commit system; /etc/init.d/zram restart
```

## 参考

| | |
|---|---|
| 上游 6.6 源码 | https://github.com/padavanonly/immortalwrt-mt798x-6.6 |
| 5.4 编译工程 / 源码 | https://github.com/wangwang623/immortalwrt-hanwckf-firmware-build · https://github.com/wangwang623/immortalwrt-mt798x |
| U-Boot 预编译 / 源码 | https://github.com/VIKINGYFY/UBOOT-CI · https://github.com/Yuzhii0718/bl-mt798x-dhcpd |
| 项目介绍 / AX3000T 教程 | https://cmi.hanwckf.top/p/immortalwrt-mt798x/ · https://okhk.net/xiaomi-ax3000t-router-with-hanwckf-immortalwrt |
| 旧仓库（索引页，勿叠加使用） | https://github.com/wangwang623/KST-WF3000A-ImmortalWrt |
