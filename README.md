<img src="https://avatars.githubusercontent.com/u/53193414?s=200&v=4" alt="logo" width="200" height="200" align="right">

# Project ImmortalWrt

ImmortalWrt is a fork of [OpenWrt](https://openwrt.org), with more packages ported, more devices supported, default optimized profiles and localization modifications for mainland China users.<br/>
Compared to upstream, we allow to use (non-upstreamable) modifications/hacks to provide better feature/performance/support.

Default login address: http://192.168.6.1 or http://immortalwrt.lan, username: __root__, password: _none_.

---

> ## ⚠️ 这是个人 fork，不是官方仓库
>
> 上游：<https://github.com/padavanonly/immortalwrt-mt798x-6.6>
>
> 本 fork 在 `openwrt-24.10-6.6` 分支上做了两件事：
>
> 1. **新增 KST WF3000A 板级支持**（源码改动 5 处，见第一～七节）
> 2. **新增 Redmi AX6000 编译流程**（**源码零改动**，上游已支持，见[第八节](#八redmi-ax6000)）
>
> 除上述外，其余内容与上游完全一致。
>
> ### 目录
>
> | 章节 | 内容 | 适用机型 |
> |---|---|---|
> | [一](#一这个-fork-改了什么) | 这个 fork 改了什么（5 处） | WF3000A |
> | [二](#二硬件与分区布局) | 硬件与分区布局 | WF3000A |
> | [三](#三mac-地址方案) | MAC 地址方案 | WF3000A |
> | [四](#四无线mtk-闭源驱动) | 无线：MTK 闭源驱动 | **两机型共用** |
> | [五](#五已知限制) | 已知限制 | **两机型共用** |
> | [六](#六本地验证改完自查) | 本地验证（改完自查） | WF3000A |
> | [七](#七同步上游) | 同步上游 | WF3000A |
> | **[八](#八redmi-ax6000)** | **Redmi AX6000（零改动编译）** | **AX6000** |
> | [九](#九通用性与参考链接) | 通用性与参考链接 | — |
>
> 只用 AX6000 → 直接跳到[第八节](#八redmi-ax6000)。
> 只看固件怎么来的 → 直接跳到[第九节 · 参考链接](#九通用性与参考链接)。

---

# KST-WF3000A 板级支持

## 一、这个 fork 改了什么

**共 5 处**，全部集中在板级移植，不涉及内核、工具链、通用包：

| # | 文件 | 类型 | 作用 |
|---|---|---|---|
| 1 | `target/linux/mediatek/dts/mt7981b-kst-wf3000a.dts` | **新增**（330 行） | 分区表、交换机端口、GPIO、MAC nvmem |
| 2 | `target/linux/mediatek/image/filogic.mk` | **末尾追加** | 机型定义 + `TARGET_DEVICES += kst_wf3000a` |
| 3 | `target/linux/mediatek/filogic/base-files/etc/board.d/02_network` | 中间插入 2 处 | 接口分组 + **有线 MAC** |
| 4 | `target/linux/mediatek/filogic/base-files/lib/upgrade/platform.sh` | 中间插入 1 处 | sysupgrade 分支 |
| 5 | `package/mtk/applications/mtwifi-cfg/files/mtwifi.sh` | **整文件替换**（103 行） | **无线 MAC** 从 factory 读取 |

> **为什么不用 CI 打补丁**：曾尝试过「源码保持原样 + 构建时脚本插入」，
> 但那套方案依赖正则/锚点定位，上游改格式就失配，而且报错往往在编译几十分钟后。
> 直接改源码可以本地 `sh -n` 验证、直接看 diff，**排查成本从"猜脚本为什么没匹配上"
> 降到"看文件对不对"**。代价只是同步上游时可能有冲突（见[第七节](#七同步上游)）。

### 各处改动要点

**① DTS（新增）** — 114MiB 分区表 + NMBM

```dts
partition@180000 { label = "factory"; reg = <0x0180000 0x0200000>; read-only; };
partition@580000 { label = "ubi";     reg = <0x0580000 0x07200000>; };
```

`factory` 必须带 `read-only`：万一 EEPROM 校验失败，驱动会试图把默认 EEPROM
写回 flash，`read-only` 能挡住这次写，保护原厂校准数据。

**② filogic.mk（末尾追加）** — ⚠️ 只能用 `>>`

```makefile
SUPPORTED_DEVICES := kst,wf3000a
DEVICE_PACKAGES := kmod-mt_wifi kmod-warp kmod-conninfra wifi-dats mtwifi-cfg
IMAGE_SIZE := 115712k
KERNEL_IN_UBI := 1
KERNEL = kernel-bin | lzma | \
	fit lzma $$(KDIR)/image-$$(firstword $$(DEVICE_DTS)).dtb
KERNEL_INITRAMFS = kernel-bin | lzma | \
	fit lzma $$(KDIR)/image-$$(firstword $$(DEVICE_DTS)).dtb with-initrd
```

用 `>` 覆盖会清掉上游全部 153 个机型。追加后 `define Device/` 应为 **153** 个。

**③ 02_network（2 处）**

- 第 58 行：接口分组，加进续行链 `kst,wf3000a|\`
- 第 246 行：MAC 分支（详见[第三节](#三mac-地址方案)）

**④ platform.sh（1 处，第 69 行）**

```sh
kst,wf3000a)
	CI_UBIPART="ubi"
	CI_KERNPART="kernel"
	CI_ROOTPART="rootfs"
	nand_do_upgrade "$1"
	;;
```

插入位置是 `platform_do_upgrade()` 里、`netcore,n60-pro|` 那个续行组的**组首之前**。
`platform_check_image()` 不用改——通用 `*)` 的 `nand_do_platform_check` 已覆盖。

**⑤ mtwifi.sh（整文件替换）** — 无线 MAC，详见[第三节](#三mac-地址方案)

---

## 二、硬件与分区布局

| 项目 | 参数 |
|---|---|
| SoC | MediaTek MT7981B（双核 A53 @ 1.3GHz） |
| RAM | 256 MB |
| Flash | **128 MB** SPI-NAND（启用 NMBM） |
| 无线 | MT7976，2.4G + 5G，AX3000 |
| 布局 | **114M（expand）** |

### U-Boot 的两种 MTD 布局（刷机时二选一）

```text
default:      nmbm0:1024k(bl2),512k(u-boot-env),2048k(factory),2048k(fip),49152k(ubi),49152k(ubi1)
expand(114m): nmbm0:1024k(bl2),512k(u-boot-env),2048k(factory),2048k(fip),116736k(ubi)
```

**本 fork 只编译 `expand(114m)` 这一种。** 刷机时 U-Boot 布局也必须选 **114m**，
两者必须一致，否则起不来。

分区算数核对：

```
1024 + 512 + 2048 + 2048 + 116736 = 122368 KiB = 0x07780000
IMAGE_SIZE = 116736 - 1024 = 115712k
```

---

## 三、MAC 地址方案

### 核心约束（这是整套方案的起点）

用 `hexdump` 读本机 factory 分区，只有**两个**无线 MAC：

```text
00000000  81 79 00 00 48 da 35 50  70 f7 48 da 35 50 70 f8  |.y..H.5Pp.H.5Pp.|
              ↑         ↑ 2.4G = ...:F7      ↑ 5G = ...:F8
              chipid 0x7981
```

- `0x04` = `48:DA:35:50:70:F7`（2.4G 无线）
- `0x0a` = `48:DA:35:50:70:F8`（5G 无线）
- **`...:F5`（机身标签）在 factory 里不存在** —— 所以只能推算

### 最终布局（五个地址升序连续）

| 用途 | 本机值 | 来源 |
|---|---|---|
| **LAN**（= 机身标签） | `...:F5` | 基准 = 2.4G − 2 |
| **WAN** | `...:F6` | 基准 + 1 |
| **label_mac** | `...:F5` | = LAN |
| 2.4G 无线 | `...:F7` | factory `0x04` 直读 |
| 5G 无线 | `...:F8` | factory `0x0a` 直读 |

即 **LAN(F5) → WAN(F6) → 2.4G(F7) → 5G(F8)**，与 factory 里 `0x04`/`0x0a` 的排布自洽。

> `label_mac`（写进 `/etc/config/system` 的 `label_macaddr`）是 LuCI 状态页显示的
> "设备身份"，取 LAN 是上游主流做法（14 处赋值里 4 处取 `lan_mac`、4 处取 `wan_mac`）。

### 有线 MAC（第 3 处：`02_network`）

```sh
kst,wf3000a)
	wlan_2g_mac=$(mtd_get_mac_binary factory 4)    # F7
	wlan_5g_mac=$(mtd_get_mac_binary factory 10)   # F8

	# 校验：排除空值 / 全 0 / 全 F，读不到就整体回退，绝不写脏值
	case "$wlan_2g_mac" in
		""|"00:00:00:00:00:00"|"ff:ff:ff:ff:ff:ff"|"FF:FF:FF:FF:FF:FF")
			wlan_2g_mac=""; wlan_5g_mac="" ;;
	esac

	if [ -n "$wlan_2g_mac" ]; then
		base_mac=$(macaddr_add "$wlan_2g_mac" -2)   # 基准 = F7 - 2 = F5
		lan_mac=$base_mac                            # LAN = 基准 = 机身标签
		label_mac=$lan_mac
		wan_mac=$(macaddr_add "$base_mac" 1)         # WAN = F6
	fi
	# ↓ 抢先把无线 MAC 写进 .dat profile（见下方「为什么必须抢先写」）
	;;
```

> 这份源码只有 `macaddr_add`，**没有** `macaddr_sub`。
> 传负数靠补码回绕实现减法，跨字节借位也正确（已实测 `...00:00:00 - 1 = ...ff:ff:ff`）。

### 为什么必须"抢先写" `.dat`

`mediatek_setup_macs()` 的**尾部**（`esac` 之后，所有机型共用）有这段：

```sh
if [ -n "$label_mac" ]; then
	base_mac="$label_mac"        # ← 优先级最高，会覆盖分支内的 base_mac
elif [ -n "$wan_mac" ]; then ...

if [ -f ${b0dat} ] && ! grep -q "MacAddress=" ${b0dat}; then
	b0mac="$(macaddr_add "$base_mac" 3)"    # 2.4G = label + 3
	b1mac="$(macaddr_add "$base_mac" 4)"    # 5G   = label + 4
fi
```

这是 padavanonly 上游**本来就有**的通用兜底（官方 OpenWrt 没有，因为它用开源 mt76）：
不管有线 MAC 是 ASCII、二进制、eMMC 还是内核值，无线一律在它上面 `+3`/`+4`，
保证五个地址不冲突——代价是**忽略 factory 里已有的无线 MAC**。

**这就是 2.4G 默认显示 `F9` 的根因**：

```
F9 = label_mac(F6) + 3 = (F7 - 1) + 3 = F7 + 2
```

修法不是删掉 `+3/+4`（会影响 AX6000 等几十个机型），而是**让守卫命中**：

```sh
# 在 kst 分支内先写入正确值
if [ -n "$wlan_2g_mac" ] && [ -f "$b0dat" ] && ! grep -q "MacAddress=" "$b0dat"; then
	echo "MacAddress=$wlan_2g_mac" >> "$b0dat"    # F7
fi
if [ -n "$wlan_5g_mac" ] && [ -f "$b1dat" ] && ! grep -q "MacAddress=" "$b1dat"; then
	echo "MacAddress=$wlan_5g_mac" >> "$b1dat"    # F8
fi
```

`.dat` 里已有 `MacAddress=` → 尾部的 `! grep -q` 守卫跳过 → `+3/+4` 不执行。

### 无线 MAC（第 5 处：`mtwifi.sh`）

原版 `detect_mtwifi()` 生成 `/etc/config/wireless` 时**不设 macaddr**，
驱动于是从 EEPROM 自行派生，实测结果是 `F9` / `4A:...:F9`，与 factory 记录不符。

改法是在生成配置时按 factory 偏移写入：

```sh
if [ $idx -eq 1 ]; then          # 2.4G
	macaddr="$(mtwifi_factory_mac 4)"      # factory 0x04 → F7
else                             # 5G
	macaddr="$(mtwifi_factory_mac 10)"     # factory 0x0a → F8
fi
```

配套读取函数带容错（读不到就不写，避免留空值）：

```sh
mtwifi_factory_mac() {
	local off="$1" m=""
	command -v mtd_get_mac_binary >/dev/null 2>&1 || return 1
	m="$(mtd_get_mac_binary factory "$off" 2>/dev/null)"
	case "$m" in
		""|"00:00:00:00:00:00"|"ff:ff:ff:ff:ff:ff"|"FF:FF:FF:FF:FF:FF") return 1 ;;
	esac
	echo "$m"
}
```

**⚠️ 5G 的实际表现见[第五节](#五已知限制)。**

### 效果与双保险

每台机器读自己的 flash，**固件可以公开分发，不会与他人撞 MAC**。

`02_network` 与 `mtwifi.sh` 现在**都读 `0x04`/`0x0a`**，不是冗余而是互补：

| | 时机 | 写入目标 | 读不到时 |
|---|---|---|---|
| `02_network` | 首次开机 | `.dat` 文件 | 回退 eth0 默认值 |
| `mtwifi.sh` | 生成无线配置 | `/etc/config/wireless` → mtwifi-cfg 覆盖 `.dat` | **不写 macaddr，保留前者写的值** |

正常情况两者都写 F7/F8；万一 `mtwifi.sh` 读 factory 失败，
它还留着 `02_network` 写进去的正确值。

---

## 四、无线：MTK 闭源驱动

### 这份源码没有 mt76

`package/kernel/mt76` **不存在**。无线来自 MTK 闭源驱动，必须在 `.config` 里显式选中：

```
CONFIG_PACKAGE_kmod-mt_wifi=y
CONFIG_PACKAGE_kmod-warp=y
CONFIG_PACKAGE_kmod-conninfra=y
CONFIG_PACKAGE_wifi-dats=y
CONFIG_PACKAGE_mtwifi-cfg=y
```

### ⚠️ 官方主线那套思路在这里不适用

`filogic.mk` 里其他机型写的 `kmod-mt7915e` / `kmod-mt7981-firmware` /
`mt7981-wo-firmware` 在**这份源码里全是死引用**——包不存在，
`make defconfig` 会静默丢弃。

（134 处 `DEVICE_PACKAGES` 里没有一处写 `kmod-mt_wifi`，
因为闭源驱动是靠 `.config` 选的，不是靠机型带出来的。）

### 闭源驱动的固有行为（不是 bug）

| 现象 | 说明 |
|---|---|
| LuCI 里 `iwinfo` 不可用 | 要看 `luci-app-mtwifi-cfg` 或 `iwpriv ra0` |
| 不能改 `/etc/config/wireless` | 要改 `/etc/wireless/mt7981/*.dat` |
| 中继扫描时同频段设备短暂断开 | 驱动特性 |
| IGMP Snooping 有问题 | 新版源码默认关闭 |

---

## 五、已知限制

### ⚠️ 5G 的 MAC 恒为 `LAA(2.4G)`，改不了（两机型都有）

实测现象（WF3000A）：

| 操作 | 5G 实际 BSSID |
|---|---|
| 修复前默认 | `4A:DA:35:50:70:F9` |
| 修复后默认 | `4A:DA:35:50:70:F7` ← 跟着 2.4G 走 |
| 5G 设为 `48:...:F8` | `4A:DA:35:50:70:F7`（**设置无效**） |
| 5G 设为 `4A:...:F9` | `4A:DA:35:50:70:F7`（**设置无效**） |

配置层面完全正常——`uci` 和 `.dat` 里确实写成了 F8：

```
b0.dat: MacAddress=48:DA:35:50:70:F7
b1.dat: MacAddress=48:DA:35:50:70:F8     ← 写进去了
```

但驱动**忽略 5G 的 macaddr**，只取 2.4G 的当前值再置 **LAA 本地管理位**
（`0x48 | 0x02 = 0x4A`）。这是闭源驱动的固有行为，改配置解决不了。

**影响**：`4A` 开头的 LAA 地址是 IEEE 802 规定的合法本地管理地址，
不影响连接、速度、稳定性，也不冲突——**只是和机身标签对不上，管理时不好认**。

**第 5 处改动的真实收益**：让 **2.4G 默认即为 F7**（不再是 F9，刷完不用手动设）。
5G 的 `4A:F7` 依然是驱动行为，**本次改动不影响**。

### ⚠️ AX6000 的 LAN MAC 上游没设

`02_network` 里 AX6000 分支只给了 `wan_mac` 和 `label_mac`：

```sh
xiaomi,redmi-router-ax6000-stock|\
xiaomi,redmi-router-ax6000-ubootmod)
	wan_mac=$(mtd_get_mac_ascii Bdata ethaddr_wan)
	label_mac=$wan_mac
	;;                        # ← 没有 lan_mac
```

所以 `ucidef_set_interface_macaddr "lan"` 被跳过，LAN 的 MAC 会回落到
网口默认值。**这是上游行为，不是本 fork 造成的**。若刷完发现 LAN MAC 与
标签不符，需单独给 AX6000 补一行 `lan_mac` 推算。

### 生效时机

`mtwifi.sh` 只在**生成 `/etc/config/wireless` 时**执行：

- ✅ 全新刷机（不保留配置）
- ✅ 删掉 `/etc/config/wireless` 后重启
- ❌ sysupgrade 保留配置 —— 沿用旧配置

### 其他待定项

- **功放类型** `CONFIG_MTK_FIRST_IF_IPAILNA`（当前按内置功放）。
  软件层面查不出来，只能拆机看 MT7981 旁有无外置前端芯片。
  **不影响有没有 WiFi，只影响发射功率**——信号明显偏弱就改
  `CONFIG_MTK_FIRST_IF_EPAELNA=y` 重编。

---

## 六、本地验证（改完自查）

这是直接改源码相比打补丁脚本的**最大优势**——改完立刻能验：

```bash
# shell 语法（最关键，记事本改过换行符就会挂在这）
sh -n target/linux/mediatek/filogic/base-files/etc/board.d/02_network
sh -n target/linux/mediatek/filogic/base-files/lib/upgrade/platform.sh
sh -n package/mtk/applications/mtwifi-cfg/files/mtwifi.sh

# 机型定义（都必须等于 1）
grep -c "define Device/kst_wf3000a" target/linux/mediatek/image/filogic.mk
grep -c "TARGET_DEVICES += kst_wf3000a" target/linux/mediatek/image/filogic.mk

# 确认没覆盖掉原有机型（应为 153）
grep -c "define Device/" target/linux/mediatek/image/filogic.mk

# MAC 逻辑
grep -c "mtd_get_mac_binary factory 4" .../02_network        # 1
grep -c "mtwifi_factory_mac" .../mtwifi.sh                    # 3
```

**GitHub 网页端自查**：仓库页搜索框输入 `kst_wf3000a` 或 `mtwifi_factory_mac`
（记得点 "In this repository"）。

---

## 七、同步上游

改动共 **5 个文件**（1 个新增、1 个末尾追加、3 处中间插入），
新增约 30 行，所以冲突很小：

| 文件 | 冲突概率 | 说明 |
|---|---|---|
| DTS | 零 | 新增文件，上游没有 |
| filogic.mk | 极低 | 末尾追加 |
| `02_network` | 中 | 上游常改这个文件 |
| `platform.sh` | 中 | 上游常改 |
| `mtwifi.sh` | 低 | 整文件替换，上游改版需重新对齐 |

同步后如有冲突，解完立刻跑[第六节](#六本地验证改完自查)的验证命令，
几分钟内就能知道有没有丢东西。

> 构建仓库的 workflow 第 5 步也会自动复核这几项，双重保险。

---

# 八、Redmi AX6000

> **本机型源码零改动。** 上游 padavanonly 已完整支持，
> 本 fork 只新增了**编译流程**（workflow + config），不动任何源码文件。

## 8.1 为什么零改动就能编

上游源码里机型定义、DTS、`02_network`、`platform.sh` 四件套**全都齐了**：

| 环节 | 上游状态 |
|---|---|
| 机型定义 `filogic.mk` | ✅ 有 `xiaomi_redmi-router-ax6000-ubootmod` |
| DTS | ✅ `mt7986a-xiaomi-redmi-router-ax6000-ubootmod.dts` |
| `02_network` | ✅ 接口分组 + WAN MAC（第 176 / 320 行两处） |
| `platform.sh` | ✅ `nand_do_upgrade` + `CI_UBIPART="ubi"` |

所以只需要一份 config + 一个 workflow。

## 8.2 分区布局：名字叫 ubootmod，实际是 hanwckf 110M

这是最容易踩的坑。上游把机型命名为 `-ubootmod`，但**里面的布局跟官方
OpenWrt 的 ubootmod 完全不同**：

| | 本源码 `-ubootmod` | 官方 OpenWrt `ubootmod` | hanwckf U-Boot |
|---|---|---|---|
| ubi 起始 | `0x600000` | `0x580000` | `0x600000` |
| ubi 大小 | **110 MiB** (`0x6e00000`) | 122.5 MiB (`0x7a80000`) | **110 MiB** |
| crash / crash_log | **保留** | 删除 | **保留** |
| NMBM | **开启** | 未开启 | **开启** |

**四项与 hanwckf U-Boot 完全一致，与官方 ubootmod 四项全不同。**

DTS 原文：

```dts
&spi_nand_flash {
        mediatek,nmbm;                              /* ★ hanwckf uboot 必需 */
        mediatek,bmt-max-ratio = <1>;
        mediatek,bmt-max-reserved-blocks = <64>;
};

&partitions {
	partition@580000 { label = "crash";     reg = <0x580000 0x40000>; read-only; };
	partition@5c0000 { label = "crash_log"; reg = <0x5c0000 0x40000>; read-only; };
	partition@600000 { label = "ubi";       reg = <0x600000 0x6e00000>; };
};
```

**结论：直接编 `-ubootmod` 即可刷 hanwckf U-Boot，不用自建 110M 变体。**
（24.10 只是把独立的 `-110m` 机型合并进了 `-ubootmod` 这个名字下，
机型少了但支持没丢——名字骗了不少人。）

## 8.3 `mt7986` 还是 `mt7986a`

**两个都在用，分属不同层面，写错会编译失败：**

| 层面 | 写法 | 位置 |
|---|---|---|
| 设备树 / 内核 | **`mt7986a`**（带 a） | DTS 文件名、`mt7986a.dtsi`、compatible |
| OpenWrt 目标平台 | `mt7986` | `CONFIG_TARGET_mediatek_mt7986=y` |
| MTK 闭源驱动 | **`MT7986`** | `CONFIG_MTK_CHIP_MT7986=y` 等 |
| conninfra | `MT7986` | `CONFIG_MTK_CONNINFRA_APSOC_MT7986=y` |
| WARP | `mt7986`（小写） | `CONFIG_WARP_CHIPSET="mt7986"` |

`a` / `b` 是芯片封装变体（MT7986A / MT7986B），内核需要不同 `.dtsi`；
**闭源驱动不区分变体**，按 SoC 系列编译，符号一律不带后缀字母。

已核实：`package/mtk/drivers/mt_wifi/config.in` 里 `MT7986A` 出现 **0 次**。

> 跟 WF3000A 的规律一致：**DTS 带后缀字母（`mt7981b`），构建符号不带。**

写 `CONFIG_MTK_CHIP_MT7986A=y` 的后果：符号不存在 → `make defconfig`
**静默丢弃** → conninfra 拿不到芯片宏 → **编译到第 50 分钟才失败**。

## 8.4 MAC 方案（上游原生，不动它）

```sh
xiaomi,redmi-router-ax6000-stock|\
xiaomi,redmi-router-ax6000-ubootmod)
	wan_mac=$(mtd_get_mac_ascii Bdata ethaddr_wan)   # 从 Bdata 读 ASCII
	label_mac=$wan_mac
	;;
```

与 WF3000A 走完全不同的路径（那边是 factory 二进制推算）：

| | 来源 | 分区 |
|---|---|---|
| AX6000 | `Bdata` 分区 ASCII 变量 `ethaddr_wan` | Bdata |
| WF3000A | `factory` 二进制偏移 `0x04` 推算 | factory |

无线 MAC 仍走 `02_network` 尾部的通用兜底 `wan+3` / `wan+4`——
**对 AX6000 这是正确的**，它的 factory 里未必有规范的无线 MAC。
（本 fork 只给 WF3000A 单独写了直读逻辑，没有动通用逻辑。）

## 8.5 编译

| 文件 | 路径 |
|---|---|
| Workflow | `.github/workflows/build-redmi-ax6000.yml` |
| 配置 | `defconfig/ax6000-hanwckf-110m.config` |

Actions 页 → **Build Redmi AX6000** → Run workflow。

两个必开项（workflow 第 ④ 步会强制打开、第 ⑤ 步会校验）：

```
CONFIG_TARGET_mediatek_filogic_DEVICE_xiaomi_redmi-router-ax6000-ubootmod=y
CONFIG_TARGET_ROOTFS_INITRAMFS=y
```

第二条是**硬性要求**：机型定义里 `ARTIFACT/initramfs-factory.ubi` 依赖
`initramfs-recovery.itb`，而 `ax6000-ubootmod` 块**没有** `ifneq(...INITRAMFS)`
守卫——不开反而会编译失败。

### 产物

该机型**没有 `factory.bin`**，只有：

```
*-initramfs-factory.ubi        ← 首次刷入（hanwckf uboot Web 界面直接认）
*-squashfs-sysupgrade.bin      ← 进系统后再升级
```

### 驱动

**只能选闭源**。`package/kernel/mt76` 在这份源码里 **404 不存在**
（`package/mtk/drivers/mt_wifi` 存在），所以 `filogic.mk` 里写的
`kmod-mt7915e` / `kmod-mt7986-firmware` **全是死引用**，
必须在 config 里显式写闭源那套：

```
CONFIG_PACKAGE_kmod-mt_wifi=y
CONFIG_PACKAGE_kmod-warp=y
CONFIG_PACKAGE_kmod-conninfra=y
CONFIG_PACKAGE_wifi-dats=y
CONFIG_PACKAGE_mtwifi-cfg=y
```

### ⚠️ 芯片符号块必须抄全（111 项）

这是踩过的坑：conninfra 的 Makefile 把 `CONFIG_MTK_*` 转成 `-D` 宏传给内核模块，
**只写 5 个 `CONFIG_PACKAGE_kmod-*` 会导致 conninfra 编译失败**。

必须从 `defconfig/mt7986-ax6000.config` 提取整块，关键项：

```
CONFIG_MTK_CHIP_MT7986=y
CONFIG_MTK_CONNINFRA_APSOC=y
CONFIG_MTK_CONNINFRA_APSOC_MT7986=y          ★
CONFIG_MTK_FIRST_IF_MT7986=y
CONFIG_MTK_MT7986_NEW_FW=y
CONFIG_MTK_WIFI_ADIE_TYPE="mt7976"           ← 射频芯片，与 SoC 无关
CONFIG_MTK_WIFI_SKU_TYPE="AX6000"
CONFIG_WARP_CHIPSET="mt7986"
CONFIG_first_card_name="MT7986"
# CONFIG_MTK_PRE_CAL_TRX_SET1_SUPPORT is not set   ★ 必须连 is not set 一起抄
# CONFIG_MTK_PRE_CAL_TRX_SET2_SUPPORT is not set
```

> 这批符号在 `config.in` 里是 `default y`，**不显式关就会被默认打开**。
> workflow 第 ⑤ 步会提前检查这 13 项，不用等 50 分钟。

## 8.6 刷机

1. 进 hanwckf U-Boot Web 恢复界面
2. **布局选 `immortalwrt-110m`**（不是 default）
3. 上传 `*-initramfs-factory.ubi`
4. 进系统后 `sysupgrade -n *-squashfs-sysupgrade.bin`
5. 验证：`cat /proc/mtd` → ubi 应为 `06e00000`

> 布局选错会 UBI volume-table 校验失败变砖。
> 另外 hanwckf 的 bl-mt798x ATF 开了 `CONFIG_ENABLE_NAND_NMBM=y`，
> 官方主线 ubootmod 不开，两层看到的 flash 大小不同，**不能混刷**。

---

## 九、通用性与参考链接

### 通用性

**所有 MAC 都从各台机器自己的 flash 分区读取并推算**：

| 机型 | 有线来源 | 无线来源 |
|---|---|---|
| WF3000A | factory `0x04` 推算 | factory `0x04` / `0x0a` 直读 |
| AX6000 | Bdata `ethaddr_wan` | 上游通用兜底 `wan+3/+4` |

所以这份源码编译出的固件可以公开分发，不同机器不会撞 MAC。

如果别人要用这个 fork：

- **WF3000A**：U-Boot 布局必须选 **114m**
- **AX6000**：U-Boot 布局必须选 **immortalwrt-110m**

否则分区对不上起不来。

### 本 fork 新增的文件清单

| 机型 | 文件 | 类型 |
|---|---|---|
| WF3000A | `target/linux/mediatek/dts/mt7981b-kst-wf3000a.dts` | 新增 |
| WF3000A | `target/linux/mediatek/image/filogic.mk` | 末尾追加 |
| WF3000A | `target/linux/mediatek/filogic/base-files/etc/board.d/02_network` | 插入 2 处 |
| WF3000A | `target/linux/mediatek/filogic/base-files/lib/upgrade/platform.sh` | 插入 1 处 |
| WF3000A | `package/mtk/applications/mtwifi-cfg/files/mtwifi.sh` | 整文件替换 |
| **AX6000** | `.github/workflows/build-redmi-ax6000.yml` | **新增** |
| **AX6000** | `defconfig/ax6000-hanwckf-110m.config` | **新增** |

> AX6000 两文件都是**纯新增**，不触碰任何源码。

### 参考链接

| 用途 | 地址 |
|---|---|
| **WF3000A 构建仓库**（方案 B：padavanonly 源码 + CI 打补丁） | <https://github.com/wangwang623/KST-WF3000A-ImmortalWrt-6.6> |
| **AX6000 workflow**（在本 fork 内） | `.github/workflows/build-redmi-ax6000.yml` |
| 本 fork 上游源码 | <https://github.com/padavanonly/immortalwrt-mt798x-6.6> |
| 多分区 U-Boot（WF3000A 刷机前提） | <https://github.com/VIKINGYFY/UBOOT-CI> |
| hanwckf 项目（AX6000 U-Boot） | <https://github.com/hanwckf> |
| hanwckf 项目文档 | <https://cmi.hanwckf.top> |
| ImmortalWrt 官方 | <https://github.com/immortalwrt> |

### 两个参考工程（本 fork 的部分思路来源）

| 工程 | 地址 | 说明 |
|---|---|---|
| Anser-y | <https://github.com/Anser-y/WF3000A-ImmortalWrt-6.6> | 注意：其 config 无无线配置，不能直接当参照 |
| 恩山「小圆」 | 同机型固件 | 实测可用；其 `DEVICE_PACKAGES` 里 `kmod-mt7981-firmware` 是死引用，无线实际来自 defconfig |

---

# 以下为 ImmortalWrt 官方内容

## Download

Built firmware images are available for many architectures and come with a package selection to be used as WiFi home router. To quickly find a factory image usable to migrate from a vendor stock firmware to ImmortalWrt, try the *Firmware Selector*.

- [ImmortalWrt Firmware Selector](https://firmware-selector.immortalwrt.org/)

If your device is supported, please follow the **Info** link to see install instructions or consult the support resources listed below.

## Development

To build your own firmware you need a GNU/Linux, BSD or macOS system (case sensitive filesystem required). Cygwin is unsupported because of the lack of a case sensitive file system.<br/>

  ### Requirements

  To build with this project, Debian 11 is preferred. And you need use the CPU based on AMD64 architecture, with at least 4GB RAM and 25 GB available disk space. Make sure the __Internet__ is accessible.

  The following tools are needed to compile ImmortalWrt, the package names vary between distributions.

  - Here is an example for Debian/Ubuntu users:<br/>
    - Method 1:
      <details>
        <summary>Setup dependencies via APT</summary>

        ```bash
        sudo apt update -y
        sudo apt full-upgrade -y
        sudo apt install -y ack antlr3 asciidoc autoconf automake autopoint binutils bison build-essential \
          bzip2 ccache clang cmake cpio curl device-tree-compiler ecj fastjar flex gawk gettext gcc-multilib \
          g++-multilib git gnutls-dev gperf haveged help2man intltool lib32gcc-s1 libc6-dev-i386 libelf-dev \
          libglib2.0-dev libgmp3-dev libltdl-dev libmpc-dev libmpfr-dev libncurses-dev libpython3-dev \
          libreadline-dev libssl-dev libtool libyaml-dev libz-dev lld llvm lrzsz mkisofs msmtp nano \
          ninja-build p7zip p7zip-full patch pkgconf python3 python3-pip python3-ply python3-docutils \
          python3-pyelftools qemu-utils re2c rsync scons squashfs-tools subversion swig texinfo uglifyjs \
          upx-ucl unzip vim wget xmlto xxd zlib1g-dev zstd
        ```
      </details>
    - Method 2:
      ```bash
      sudo bash -c 'bash <(curl -s https://build-scripts.immortalwrt.org/init_build_environment.sh)'
      ```

  Note:
  - Do everything as an unprivileged user, not root, without sudo.
  - Using CPUs based on other architectures should be fine to compile ImmortalWrt, but more hacks are needed - No warranty at all.
  - You must __not__ have spaces or non-ascii characters in PATH or in the work folders on the drive.
  - If you're using Windows Subsystem for Linux (or WSL), removing Windows folders from PATH is required, please see [Build system setup WSL](https://openwrt.org/docs/guide-developer/build-system/wsl) documentation.
  - Using macOS as the host build OS is __not__ recommended. No warranty at all. You can get tips from [Build system setup macOS](https://openwrt.org/docs/guide-developer/build-system/buildroot.exigence.macosx) documentation.
  - For more details, please see [Build system setup](https://openwrt.org/docs/guide-developer/build-system/install-buildsystem) documentation.

  ### Quickstart

  1. Run `git clone -b openwrt-24.10-6.6 --single-branch --filter=blob:none https://github.com/wangwang623/immortalwrt-mt798x-6.6 immortalwrt-mt798x-6.6` to clone **this fork** (already contains the KST WF3000A port).
  2. Run `cd immortalwrt-mt798x-6.6` to enter source directory.
  3. Run `./scripts/feeds update -a` to obtain all the latest package definitions defined in feeds.conf / feeds.conf.default
  4. Run `./scripts/feeds install -a` to install symlinks for all obtained packages into package/feeds/
  5. Copy the configuration file for your device from the `defconfig` directory to the project root directory and rename it `.config`

     ```bash
     # MT7981 (generic reference)
     cp -f defconfig/mt7981-ax3000.config .config

     # MT7986 (generic reference)
     cp -f defconfig/mt7986-ax6000.config .config

     # Redmi AX6000 with hanwckf U-Boot (110M layout) — in this repo
     cp -f defconfig/ax6000-hanwckf-110m.config .config
     ```

     > For KST WF3000A, use the config from the
     > [build repo](https://github.com/wangwang623/KST-WF3000A-ImmortalWrt-6.6)
     > instead — it selects the proprietary MTK wireless driver
     > (see [section 四](#四无线mtk-闭源驱动)).
     >
     > Note: the generic `defconfig/mt798*-*.config` templates are
     > **MULTI_PROFILE** (all devices of the chip selected at once).
     > `defconfig/ax6000-hanwckf-110m.config` selects **one** device only.

  6. Run `make` to build your firmware. This will download all sources, build the cross-compile toolchain and then cross-compile the GNU/Linux kernel & all chosen applications for your target system.

  ### Related Repositories

  The main repository uses multiple sub-repositories to manage packages of different categories. All packages are installed via the OpenWrt package manager called opkg. If you're looking to develop the web interface or port packages to ImmortalWrt, please find the fitting repository below.

  - [LuCI Web Interface](https://github.com/immortalwrt/luci): Modern and modular interface to control the device via a web browser.
  - [ImmortalWrt Packages](https://github.com/immortalwrt/packages): Community repository of ported packages.
  - [OpenWrt Routing](https://github.com/openwrt/routing): Packages specifically focused on (mesh) routing.
  - [OpenWrt Video](https://github.com/openwrt/video): Packages specifically focused on display servers and clients (Xorg and Wayland).

## Support Information

For a list of supported devices see the [OpenWrt Hardware Database](https://openwrt.org/supported_devices)

  ### Documentation
  - [Quick Start Guide](https://openwrt.org/docs/guide-quick-start/start)
  - [User Guide](https://openwrt.org/docs/guide-user/start)
  - [Developer Documentation](https://openwrt.org/docs/guide-developer/start)
  - [Technical Reference](https://openwrt.org/docs/techref/start)

  ### Support Community
  - Support Chat: group [@ctcgfw_openwrt_discuss](https://t.me/ctcgfw_openwrt_discuss) on [Telegram](https://telegram.org/).
  - Support Chat: group [#immortalwrt](https://matrix.to/#/#immortalwrt:matrix.org) on [Matrix](https://matrix.org/).

## License

ImmortalWrt is licensed under [GPL-2.0-only](https://spdx.org/licenses/GPL-2.0-only.html).

## Acknowledgements

<table>
  <tr>
    <td><a href="https://dlercloud.com/"><img src="https://user-images.githubusercontent.com/22235437/111103249-f9ec6e00-8588-11eb-9bfc-67cc55574555.png" width="183" height="52" border="0" alt="Dler Cloud"></a></td>
    <td><a href="https://www.jetbrains.com/"><img src="https://resources.jetbrains.com/storage/products/company/brand/logos/jb_square.png" width="120" height="120" border="0" alt="JetBrains"></a></td>
    <td><a href="https://sourceforge.net/"><img src="https://sourceforge.net/sflogo.php?type=17&group_id=3663829" alt="SourceForge" width=200></a></td>
  </tr>
</table>
