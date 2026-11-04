# ImmortalWrt 24.10 / Linux 6.6 —— KST WF3000A & Redmi AX6000

基于 [padavanonly/immortalwrt-mt798x-6.6](https://github.com/padavanonly/immortalwrt-mt798x-6.6) 的 **fork 源码仓库**。
机型移植改动已直接写进源码，可直接用 GitHub Actions 云编译，**不需要在 CI 里跑查找/替换补丁脚本**。

> ✅ **已内置 NMBM oobsize 对齐修复**（`mediatek,bmt-mtd-overridden-oobsize = <64>`）。
> 它解决的是「同一份固件、同一份 U-Boot、同样的刷法，有的机器正常，有的机器刷完反复重启或一直停在 U-Boot Web」的问题。
> 详见下文「NMBM / oobsize 对齐修复」章节，**首次编译前建议通读一遍**。

固件定位：**纯净**。官方默认包 + LuCI（中文）+ HNAT 硬件加速 + MTK 无线驱动/固件，不含第三方插件；需要什么刷完自己 `opkg install`。

---

## 一、与旧仓库的关系

旧仓库 [KST-WF3000A-ImmortalWrt-6.6](https://github.com/wangwang623/KST-WF3000A-ImmortalWrt-6.6) 已停止维护，请统一迁移到本仓库。

| | 旧仓库 | **本仓库** |
|---|---|---|
| 源码 | 无，CI 里 checkout 上游官方源 | **有，fork 完整源码** |
| 移植改动 | `scripts/prepare-source.py` 靠正则/锚点注入 | **已写进源码，5 个文件** |
| 上游改文件格式 | 锚点失配 → 编译前报错 | 无此风险 |
| 本地验证 | 只能 `simulate.py` 模拟 | 可直接 `diff`、`sh -n` |
| 维护成本 | 需跟着上游调整锚点 | 只管自己那几处改动 |
| NMBM oobsize 修复 | 早期产物未内置 | **已内置** |

> ⚠️ **两个仓库不要叠加使用。** 本仓库已含机型定义，若再跑旧仓库的 `prepare-source.py`，会因「机型重复定义」失败。
> 同理，若把 CI 的 `SOURCE_REPO` 指向本仓库，必须停用补丁脚本。

---

## 二、硬件与布局

| 项目 | KST WF3000A | Redmi AX6000 |
|---|---|---|
| SoC | MediaTek MT7981B（双核 A53 @ 1.3GHz） | MT7986 |
| 内存 | 256 MB | — |
| Flash | **128 MB SPI-NAND（Winbond W25N01KV，带 NMBM）** | 128 MB SPI-NAND（ESMT） |
| 无线 | MT7976，2.4G + 5G 双频 AX3000 | MT7976 |
| 本工程布局 | **114M（expand）** | 按本仓库对应机型定义 |

**WF3000A 只编译 `expand(114m)` 一种布局**，刷机时 U-Boot 布局也必须选 **114m**，两者必须对应。

```text
default:      nmbm0:1024k(bl2),512k(u-boot-env),2048k(factory),2048k(fip),49152k(ubi),49152k(ubi1)
expand(114m): nmbm0:1024k(bl2),512k(u-boot-env),2048k(factory),2048k(fip),116736k(ubi)
```

| 分区 | 起始 | 大小 |
|---|---:|---:|
| bl2 | `0x0000000` | `0x0100000` |
| u-boot-env | `0x0100000` | `0x0080000` |
| factory | `0x0180000` | `0x0200000` |
| fip | `0x0380000` | `0x0200000` |
| ubi | `0x0580000` | `0x07200000`（116736 KiB） |

DTS 里 `ubi` 为 `0x07200000`，`IMAGE_SIZE` 比它少约 1 MiB，留给 UBI 开销与坏块处理。

---

## 三、本仓库相对上游的改动

以下改动**已直接写在源码里**，不需要任何 CI 注入。路径以仓库实际文件为准。

| # | 文件 | 改动 |
|---|---|---|
| ① | `target/linux/mediatek/dts/mt7981b-kst-wf3000a.dts` | 新增设备树；含 `mediatek,bmt-mtd-overridden-oobsize = <64>`（见第四章） |
| ② | `target/linux/mediatek/image/filogic.mk` | 追加 `kst,wf3000a` 机型定义、`IMAGE_SIZE`、无线包引用 |
| ③ | `target/linux/mediatek/filogic/base-files/etc/board.d/02_network` | 新增 `kst,wf3000a)` 分支，从 factory 推导 LAN/WAN MAC |
| ④ | `target/linux/mediatek/filogic/base-files/lib/upgrade/platform.sh` | 新增机型，支持 sysupgrade |
| ⑤ | `package/mtk/applications/mtwifi-cfg/files/mtwifi.sh` | 整文件替换，生成无线配置时按 factory 偏移写 MAC |
| ⑥ | `config/wf3000a.config` | 构建配置（意图清单），显式打开 MTK 闭源无线包 |

自查（任选其一，能确认改动在源码里）：

```bash
grep -n "kst,wf3000a" target/linux/mediatek/image/filogic.mk
grep -n "mediatek,bmt-mtd-overridden-oobsize" target/linux/mediatek/dts/mt7981b-kst-wf3000a.dts
grep -n "kmod-mt_wifi" config/wf3000a.config
grep -n "kst,wf3000a" target/linux/mediatek/filogic/base-files/etc/board.d/02_network
```

---

## 四、NMBM / oobsize 对齐修复（必读）

### 4.1 它解决什么问题

一台 WF3000A 刷本固件后可能出现：

- U-Boot Web UI 能进、能刷入 `factory.bin`，但重启后**网口反复连接/断开、指示灯熄灭→闪几下→常亮→循环**；
- 或者刷完**稳定停在 U-Boot Web UI、灯常亮**，就是不进系统；
- 但 **Web UI 里上传 `initramfs-kernel.bin` 能正常进系统**；
- 而**另一台同批次、同 U-Boot、同刷法、同一个固件文件却完全正常**。

这类现象极易被误判成「这台机器 flash 坏了 / 硬件有问题」。实际上多数是**固件与 U-Boot 的 NMBM 参数不一致**，属于固件缺陷，只是**零坏块的机器侥幸不发作**。

### 4.2 根因链

```text
① U-Boot 的 drivers/mtd/nand/spi/winbond.c
   W25N01KV → NAND_MEMORG(1, 2048, 64, ...)   → 把 64 写进 NMBM 签名 spare_size
   签名落在 flash 末尾签名块（最后一块；若是坏块则向前顺延）

② Linux 6.6 upstream 的 winbond.c
   同一颗 W25N01KV → NAND_MEMORG(1, 2048, 96, ...) → mtd->oobsize = 96

③ 内核 mtk_bmt.c → nmbm_attach() 四项校验
   ni->signature.spare_size (64) != bmtd.mtd->oobsize (96)
   → 打印 "NMBM configuration mismatch" → return -EINVAL

④ mtk_bmt_init_nmbm() 对 attach 失败是「静默跳过」，不替换 MTD 回调、不停机
   → 内核按 DTS fixed-partitions 在裸物理设备 spi0.0 上按物理偏移建分区
```

于是形成两套视图：

| | U-Boot | 内核（mismatch 后） |
|---|---|---|
| 底座 | `nmbm0` 逻辑设备，坏块透明重映射 | 裸 `spi0.0` 物理设备 |
| 看到的块 | 912 个「全好」逻辑块 | 907 好 + 5 坏（UBI 自己跳过） |
| NMBM 管理区 | 末尾 1/16（8 MiB，`0x7800000` 起），签名 + 主备信息表 + 替换块池 | 完全不处理 |

- **零坏块的机器**：逻辑视图 == 物理视图，两边读到的字节完全一样 → 怎么刷都正常。
- **有物理坏块的机器**：NMBM 已把坏块重映射到末尾备用区（日志特征：`Signature found at block 1022`、`info table 960/963`、`writecount 2`）→ 视图分叉 → U-Boot 写的固件内核读不到、内核写的固件 U-Boot 读不到。
- **initramfs 一直能起**，因为它 `bootm` 的是内存镜像，压根不碰 flash 上的 UBI 分区。

> `NMBM`（NAND mapping block management）是联发科的纯软件坏块映射层：末尾划 1/16 容量做签名 + 主备信息表 + 替换块池，逻辑块号 → 物理块号按 **eraseblock（128 KiB）** 映射。它让 `bl2`/`fip`/`u-boot-env`/`factory` 这些按固定偏移读的裸分区也能假设「块块都好」。
> 前提是 **Bootloader 与内核必须复用同一张映射表**，否则按物理偏移去读被映射改过的位置就会读错。

### 4.3 修复代码

在 `mt7981b-kst-wf3000a.dts` 的 `spi_nand: spi_nand@0` 节点内，紧跟 `mediatek,nmbm;` 加一行：

```dts
&spi_nand {
	...
	mediatek,nmbm;
	mediatek,bmt-mtd-overridden-oobsize = <64>;   /* ← 本仓库已内置 */
	mediatek,bmt-max-ratio = <1>;
	mediatek,bmt-max-reserved-blocks = <64>;
	...
};
```

生效依赖内核侧已合入 OpenWrt `e585ae7` / 24.10 `37cc725b`（本仓库所用 24.10 分支已包含）。编译前可确认：

```bash
grep -rn "bmt-mtd-overridden-oobsize" target/linux/generic/files/drivers/mtd/nand/
```

必须命中 `mtk_bmt.c` 里的 `of_property_read_u32(...)`。那段代码在 `nmbm_attach()` **之前**把 `bmtd.mtd->oobsize` 从 96 覆盖成 64，四项校验随即通过，内核成功 attach、与 U-Boot 复用 flash 上同一张映射表。

> 如果 `grep` 搜不到：光改 DTS 无效，属性只是一段没人读的字符串，需要先 backport `e585ae7`。

### 4.4 这个 `64` 到底是什么

**不是芯片参数，不是 datasheet 值，不是内核驱动值。** 它是 **U-Boot 写进 NMBM 签名并持久化下来的 `spare_size`**，即「账本上的数」。DTS 属性只是入参，`mtk_bmt.c` 才是执行者，作用是把内核用于对账的值改成账本上的值。

属性名 `overridden-oobsize` 容易误导，理解成 `align-to-signature-spare-size` 更准确。它只在 `val && val < mtd->oobsize` 时生效，改的只是 NMBM 签名比对用的那个字段，不碰芯片驱动表、不碰 ooblayout、不碰 ECC 实际使用的 spare 区。

### 4.5 不写 / 写错的后果

| 写法 | 最终 oobsize | 与签名 64 | NMBM | 后果 |
|---|---|---|---|---|
| 不写 | 96 | ≠ | ❌ | 静默回退物理层；零坏块机器侥幸跑，有坏块机器必炸 |
| **`<64>`** | **64** | **=** | **✅** | **修复成功，两边共用映射表** |
| `<65>`～`<95>` | 65～95 | ≠ | ❌ | 会打印 `overridden to xxB` 但仍 mismatch，**假修复** |
| `<96>` 及以上 | 96（不覆盖） | ≠ | ❌ | 同「不写」 |
| `<1>`～`<63>` | 1～63 | ≠ | ❌ | 修不好，且共享的 `mtd->oobsize` 被压小，可能牵连 ECC / UBI 异常 |
| `<0>` | 96（`val` 为 0 不生效） | ≠ | ❌ | 同「不写」 |
| 换用 ESMT/Foresee（驱动本就 64） | 64 | = | ✅ | 加不加都行，加了因 `64 < 64` 为假而不生效 |

**所以这个数必须严格等于 U-Boot 写进签名的那个值，不能凭「比内核小」随便猜。**

### 4.6 换 NAND flash 怎么办

`64` 不是写死的硬件常量，换料后要重新确认三件事：

| 量 | 谁决定 |
|---|---|
| 内核驱动表的 oobsize | 对应厂商驱动 `NAND_MEMORG` 的第三个参数，靠 JEDEC ID 探测选中 |
| **U-Boot 签名的 spare_size** | U-Boot 那份驱动表，编译期写死、落盘后固定 |
| DTS 的 override 值 | **必须等于第二项**，与 datasheet 值、内核值都无关 |

- 新料两边声明一致（如 ESMT，U-Boot 64 / 内核 64）→ 这行可删，留着也不生效。
- 新料又不一致（如 U-Boot 64 / 内核 128）→ DTS 改成 **U-Boot 写的那个值**。
- **换料 / 换 OOB 布局必须先擦掉 NMBM 签名块让它重建**，否则旧账本里的 `spare_size` 还在，新驱动照样 mismatch。别一上来怀疑驱动。
- 若 page size / 块数 / 容量也变了，末尾 1/16 管理区起点、签名块位置、`mtdparts` 里的 `116736k(ubi)` 与 DTS 的 `0x07200000` 都要跟着重算，不只是改这一行。

> `oobsize` 是每 **page**（2 KiB 主区）尾部附带的 spare 区字节数，出厂就定、厂商/型号/后缀相关；映射粒度是 **eraseblock**（128 KiB），两者差 64 倍，不是一回事。raw NAND 都有 OOB，NOR 没有，eMMC/UFS 对主机不可见。

### 4.7 编译期自检（推荐，避免刷完才发现问题）

「同固件 A 机好 B 机坏」最容易把人带去怀疑机器。建议把 NMBM 兼容性做成**编译流水线里的硬门禁**，而不是刷机后的排查笔记：

- **编译前 check**：解析内核驱动表 oobsize、U-Boot 签名基准、每个启用 `mediatek,nmbm` 的 DTS，三方比对；命中「有风险固件」直接 fail，不产出镜像。
- **编译后 verify**：检查生成的 `.dtb` 里属性是否真的编进去、内核对象里是否真有 override 逻辑（防止源码改了但没重编、补丁放错目录）。
- **报告随 artifact 发布**，刷机前先看结论。

门禁必须校验三件事同时成立，只查「属性存在」会放过 `<65>` 这种假修复：

1. 内核源码里确实有 `of_property_read_u32(np, "mediatek,bmt-mtd-overridden-oobsize", ...)`；
2. 属性值**严格等于 U-Boot 签名基准值**（本机 64，换料后是别的数）；
3. 值落在 `(0, 内核驱动声明值)` 区间，才会被代码真正采用。

脚本可放 `scripts/check-nmbm-compat.sh`（`check` / `verify` 两个阶段），在 workflow 里加两个 step：源码就绪后、`feeds update` 前跑 `check`，镜像生成后跑 `verify`。解析不到证据时应按风险阻断，不要静默放过。

### 4.8 刷机前必做 + 刷后验证

代码修复管的是「两边用同一张映射表」，**不负责清理 flash 上的旧数据**。若这台机器曾反复擦写过 UBI、或写过别的机器的 ubi 备份，可能残留旧 UBI EC header（`image_seq` 与当前镜像不一致），UBI 会拒绝 attach：

```text
MT7981> mtd erase ubi
MT7981> ubi part ubi
# 期望：empty MTD device detected / image sequence number: 0
# 不应再出现 bad image sequence number ... in PEB xxx
```

编译后**先用 initramfs 验证**（不写 flash，零风险），SSH 进去：

```sh
dmesg | grep -i nmbm
```

期望看到：

```text
NMBM: mtd OOB size has been overridden to 64B
Signature found at block 1022        # 块号视实际坏块情况
NMBM has been successfully attached
```

且**不再出现** `NMBM configuration mismatch`。

> ⚠️ `5 fixed-partitions found on MTD device spi0.0` 修复成功后**仍会打印**，因为这版 `mtk_bmt` 是在原 mtd 上替换 ops、设备名不变。**不能靠设备名判断 NMBM 有没有生效**，只认 `NMBM has been successfully attached`。

确认后再刷正式固件。此后 U-Boot Web UI 刷 `factory.bin` 与系统内 `sysupgrade` 都会落在同一视图，不再互相对不上。

### 4.9 其它机型适用性

| 机型 | 闪存 | 是否需要该属性 |
|---|---|---|
| KST WF3000A | Winbond W25N01KV | **需要**（本仓库已加） |
| 小米 AX3000T（W25N01KV 版） | Winbond W25N01KV | **需要**；OpenWrt `3299d19` / 24.10 `db30b60` 已官方修复 |
| 小米 AX3000T（ESMT / Foresee 版） | 64B OOB | 不需要，加了也不生效 |
| Redmi AX6000 | ESMT F50L1G41LB | 不需要，天然对齐 |

判断依据看 U-Boot 启动日志：

```text
spi-nand: Winbond SPI NAND was found. ... OOB size: 64   → 加 override
spi-nand: ESMT / FORESEE ... OOB size: 64                → 不用加
```

**决定因素是闪存型号 + 内核版本，不是 SoC、也不是「U-Boot 是不是 hanwckf 系」**。AX6000 自己的坑是 stock / 110m / ubootmod 三种分区布局互不兼容，与本问题无关。

### 4.10 排查归因规则（建议写进 issue 模板）

> 同一份固件、同一个 U-Boot、同样刷法，A 机正常 B 机异常时：
> 只要 B 机出现 `Signature found at block 1022`（非 1023）、`info table ... writecount > 0`、
> `NMBM configuration mismatch`、`bad PEBs > 0`、
> 「Web UI 能刷、initramfs 能起、正式固件起不来」中的任意几条，
> **优先判定为固件与 U-Boot 的 NMBM 参数不一致，而不是机器硬件故障**。
> 零坏块时两个视图重合，所以「侥幸能跑」不代表固件没问题。

---

## 五、刷机前提

固件是 114M 大分区版本，**必须先刷入多分区 U-Boot**。

1. **解锁 SSH**：在路由器配置项中导入 `开ssh和救砖/nl_wr8103-sysbackup-ssh.bin`，然后**拔电源重启**。
2. **刷入 U-Boot**：推荐 PuTTY + WinSCP 操作，下载地址见 [VIKINGYFY/UBOOT-CI](https://github.com/VIKINGYFY/UBOOT-CI)（`release/Yuzhii0718-SP2-26.10.12`，文件名 `fip-mt7981_kst_wf3000a_SP2-Yuzhii-dhcpd-fixed-parts-multi-layout_md5-...bin`）。
   U-Boot 源码基于 [Yuzhii0718/bl-mt798x-dhcpd](https://github.com/Yuzhii0718/bl-mt798x-dhcpd)。
3. 串口救砖方法见 `开ssh和救砖/MediaTek Filogic 系列路由器串口救砖教程.md`。

> U-Boot Web 刷机页面地址 `http://192.168.1.1/`，布局务必选 **expand(114m)**。

---

## 六、云编译

1. 本仓库 → **Settings → Actions → General → Workflow permissions** 勾 **Read and write**。
2. **Actions → 对应构建 workflow → Run workflow**，填写输入项（源码分支/commit、是否发布 Release 等，以 workflow 实际输入框为准）。
3. 等 1～2 小时，产物在 **Artifacts** / Release。

产物文件名带 `kst_wf3000a` 即说明选对了机型：

```text
immortalwrt-...-mediatek-filogic-kst_wf3000a-initramfs-kernel.bin
immortalwrt-...-mediatek-filogic-kst_wf3000a-squashfs-factory.bin
immortalwrt-...-mediatek-filogic-kst_wf3000a-squashfs-sysupgrade.bin
```

**源码版本**：输入分支名会跟着上游变动，可跟进安全更新；若要复现某次成功构建，把那次 Release body 里记录的 `Commit`（短 SHA）与 `Kernel` 版本填回去锁死版本。因为本仓库是 fork，**换版本时留意自己那几处改动是否需要跟着调整**。

---

## 七、刷机与升级

### 首次（强烈建议先走内存启动）

1. U-Boot Web，布局选 **114m**。
2. 先上传 **`initramfs-kernel.bin`** —— 内存启动，不写 flash，零风险。
3. SSH 进去确认无线、NMBM（见 4.8）正常后再刷正式包：

   ```sh
   dmesg | grep -iE "FLASH OFFSET|NVM is FLASH|eeprom"
   # 看到 FLASH OFFSET [0x0] → 读到了原厂校准数据
   dmesg | grep -i nmbm
   ```

4. 确认无误后，U-Boot Web 上传 **`factory.bin`**；若之前 UBI 有残留，先按 4.8 执行一次 `mtd erase ubi`。

### 后续升级

```sh
sysupgrade -n /tmp/xxx-squashfs-sysupgrade.bin
```

⚠️ 三个文件不能混用：

| 文件 | 用途 |
|---|---|
| `initramfs-kernel.bin` | 内存启动验证（不写 flash） |
| `factory.bin` | U-Boot Web 首次刷入 |
| `sysupgrade.bin` | 系统内升级 |

---

## 八、MAC：从 factory 推导，不硬编码

### 分配规律（WF3000A 实测）

| 来源 | 值 | 说明 |
|---|---|---|
| factory `0x04` | `48:DA:35:50:70:F7` | **2.4G 无线** ← 基准 MAC |
| factory `0x0a` | `48:DA:35:50:70:F8` | **5G 无线** |
| 推导 `base-1` | `48:DA:35:50:70:F6` | **LAN** |
| 推导 `base-2` | `48:DA:35:50:70:F5` | **WAN**（与机身标签一致） |

```text
base = factory 分区偏移 0x04 处的 MAC   ← 2.4G 无线
LAN  = base - 1
WAN  = base - 2
```

> ⚠️ factory 里**没有** F5：`48:DA:35:50:70:F5` 只出现在机身标签。必须从 `0x04` 读基准再往下推算，不能直接读。
> 硬编码会把本机 MAC 写死进固件，别人刷了就撞 MAC；从每台自己的 factory 读取，固件才能公开分发。

`02_network` 里的实现：

```sh
kst,wf3000a)
	base_mac=$(mtd_get_mac_binary factory 4)
	lan_mac=$(macaddr_add "$base_mac" -1)
	wan_mac=$(macaddr_add "$base_mac" -2)
	label_mac=$lan_mac
	;;
```

> **为什么用 `macaddr_add` 传负数**：这份源码的 `/lib/functions/system.sh` 只有 `macaddr_add`，没有 `macaddr_sub`。其内部是 `(0x<nic> + val) & 0xffffff`，传负数靠补码回绕等价减法。源码自带的 `02_network` 里就有 `macaddr_add "$lan_mac" -2` 的用法，实测可行。

### 无线 MAC

有线由 `02_network` 处理，无线由 `mtwifi.sh` 的 `detect_mtwifi()` 在生成 `/etc/config/wireless` 时决定：

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

> ⚠️ **闭源驱动仍会忽略 5G 的 macaddr**：实测设 `48:...:F8`、`b1.dat` 里也写成 F8，但 BSSID 恒为 `LAA(2.4G)`。这是驱动固有行为，改配置解决不了。本动作的实际收益是**让 2.4G 默认即为 F7**。
> 生效时机：只在生成 `/etc/config/wireless` 时（全新刷机，或删掉该文件后重启）；sysupgrade 保留配置时沿用旧配置。

---

## 九、无线：MTK 闭源驱动（这份源码没有 mt76）

上游源码里 `package/kernel/mt76` 不存在，也没有任何 Makefile 定义 `mt7981-firmware` / `mt7981-wo-firmware`；`filogic.mk` 里其它机型写的 `kmod-mt7915e` 等是**死引用**，会被 `make defconfig` 静默丢弃——这正是「固件没有无线」的根因。

本源码真正可用的无线是 MTK 闭源驱动：

| 包 | 路径 | 作用 |
|---|---|---|
| `kmod-mt_wifi` | `package/mtk/drivers/mt_wifi` | mt_wifi.ko 闭源驱动 |
| `kmod-warp` | `package/mtk/drivers/warp` | WED 硬件卸载 |
| `kmod-conninfra` | `package/mtk/drivers/conninfra` | 连接基础设施 |
| `wifi-dats` | `package/mtk/drivers/wifi-profile` | mt7981 profile(.dat) |
| `mtwifi-cfg` | `package/mtk/applications/mtwifi-cfg` | 无线配置工具 |

**无线不会自动带出来**：`filogic/target.mk` 的 `DEFAULT_PACKAGES` 和各机型 `DEVICE_PACKAGES` 都没有它，必须在 `config/wf3000a.config` 里显式写 `=y`。

### 闭源驱动的固有行为（不是 bug）

- LuCI 里 `iwinfo` 不可用，无线状态看 `luci-app-mtwifi-cfg` 或 `iwpriv ra0`
- 不能改 `/etc/config/wireless`，要改 `/etc/wireless/mt7981/` 下的 `.dat` profile
- 中继扫描时同频段设备会短暂断开
- IGMP Snooping 可能有问题（新版源码已默认关闭）

---

## 十、config 是「意图清单」，不是全量快照

只写要的，不写不要的；依赖由 `make defconfig` 沿依赖树自动展开：

```sh
CONFIG_PACKAGE_luci=y
# 不写的包，默认就是不编译
```

### ⚠️ 唯一例外：MTK 驱动符号必须连 `is not set` 一起抄

`config/wf3000a.config` 里的 **MTK 驱动符号块**逐行照抄源码自带的 `defconfig/mt7981-ax3000.config`，**包括 `# CONFIG_X is not set` 行**。这批符号在 `mt_wifi/config.in` 里是 `default y`，不显式禁用就会被默认打开，曾导致：

```text
eeprom.c:508  error: 'PRE_CAL_SIZE_ONE_CARD' undeclared
eeprom.c:539  error: 'PRECALPART_OFFSET'     undeclared
ERROR: package/mtk/drivers/mt_wifi failed to build.
```

**只比对「我写了什么」不够，还要比对「上游禁用了什么」**——建议保留 CI 里的双向比对步骤。

### 编进固件 vs 事后安装

| 类型 | 编进去？ | 理由 |
|---|---|---|
| `kmod-*` | **必须** | 与内核版本强绑定，事后 `opkg install` 必然失败 |
| `ca-bundle` / `ca-certificates` | **必须** | 否则 opkg 连不上 https 源 |
| `CONFIG_LUCI_LANG_zh_Hans` | **必须** | 不是插件，是 LuCI 语言配置项 |
| 无线驱动 / 固件 | **必须** | 否则没有 WiFi |
| `luci` | 建议 | 让机器可用的基础 |
| `luci-app-*` / 主题 | 不必 | 纯脚本，随时 `opkg install` |

> `luci-ssl` 是**入站** HTTPS（浏览器 → 管理页），`ca-bundle` 是**出站** HTTPS（路由器 → 软件源），方向相反。本工程保留 `ca-bundle`、省略 `luci-ssl`。

以后装插件：`opkg update && opkg install luci-app-upnp luci-app-ddns luci-app-wol luci-app-package-manager zram-swap`。带 `kmod-*` 的包必须重新编译后刷入。

---

## 十一、验证命令

```sh
cat /sys/class/net/lan1/address   # 本机示例 48:da:35:50:70:f6
cat /sys/class/net/wan/address    # 本机示例 48:da:35:50:70:f5
hexdump -C -n 16 /dev/mtd2        # 基准：0x04 = ...F7
cat /proc/mtd                     # ubi 应为 0x07200000 / 116736k
dmesg | grep -i nmbm              # 见 4.8，必须出现 NMBM has been successfully attached
dmesg | grep -iE "mt_wifi|warp|conninfra|eeprom"
opkg list-installed               # 确认无线包确实编进去了
```

---

## 十二、本地编译（可选）

本仓库走云编译，不需要本地构建。若在本机编译，参考上游 README：

```bash
git clone -b openwrt-24.10-6.6 --single-branch --filter=blob:none \
  https://github.com/padavanonly/immortalwrt-mt798x-6.6
cd immortalwrt-mt798x-6.6
./scripts/feeds update -a
./scripts/feeds install -a
cp -f defconfig/mt7981-ax3000.config .config
make -j$(nproc)
```

- 用**非 root 用户**，不要 sudo
- PATH 和工作目录**不能有空格或非 ASCII 字符**
- 至少 4GB 内存、25GB 磁盘，需联网；Debian 11 + AMD64 优先
- WSL 下需移除 PATH 中的 Windows 目录
- 若用本仓库（fork）编译，直接把 `config/wf3000a.config` 拷成 `.config`

---

## 十三、参考地址

| 项目 | 地址 |
|---|---|
| 上游源码 | https://github.com/padavanonly/immortalwrt-mt798x-6.6 |
| 本项目介绍（hanwckf） | https://cmi.hanwckf.top/p/immortalwrt-mt798x/ |
| U-Boot 源码 | https://github.com/Yuzhii0718/bl-mt798x-dhcpd |
| U-Boot 预编译 | https://github.com/VIKINGYFY/UBOOT-CI |
| WF3000A 移植早期版本 | https://github.com/Anser-y/WF3000A-ImmortalWrt-6.6 |
| 旧仓库（已归档，勿用） | https://github.com/wangwang623/KST-WF3000A-ImmortalWrt-6.6 |

### 与参考工程的差异

| | Anser-y | 恩山「小圆」 | **本仓库** |
|---|---|---|---|
| Flash 布局 | 大分区 | 114M（另有大分区变体） | **114M（单一）** |
| 插件 | 大量 | 较多 | **无（纯净）** |
| MAC | 硬编码 | 随机（DTS 缺 nvmem-cells） | **每台机器各自推导** |
| NMBM oobsize | 未处理 | 未处理 | **已对齐** |

> ⚠️ Anser-y 那份 config 里没有任何无线配置，不能作为「无线正常」的参照物。
