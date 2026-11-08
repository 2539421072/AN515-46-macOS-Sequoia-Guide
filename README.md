# 暗影骑士·龙 2022 / Acer Nitro 5 AN515-46 安装 macOS 15 Sequoia / 26 Tahoe 与驱动修复全记录

> 一台 **AMD Ryzen 7 6800H + RTX 3060 Laptop** 笔记本的黑苹果完整攻坚记录。
> 最终：**RTX 3060 Laptop 显卡加速（Metal 3 / 内屏 165Hz）、WiFi、蓝牙（含耳机音频）、触控板与手势、USB、声卡、有线网、键盘、电池、亮度键全部可用**。
> 环境：macOS Sequoia 15.8.2（Darwin 24.6.0）→ **macOS 26.7.2 Tahoe（build 25G332）** / OpenCore / SMBIOS 15 用 `iMacPro1,1`、26 用 `MacPro7,1`。

这不是一份「照抄 EFI 就能用」的教程，而是一份**排障经验总结**：每一项都写清楚「症状 → 根因 → 具体做法 → 验证命令」，并如实记录走过的弯路。很多坑不是硬件问题，而是**判断方法**的问题，这也是本文最有价值的部分。

> ⚠️ 本文记录的所有结论均来自一台实机的反复验证。不同 BIOS 版本、不同无线卡子型号、不同 macOS 小版本都可能让结果不同。**请务必先整包备份自己的 EFI，再逐项尝试。**

> 📌 **2026-11-08 更新**：本文已验证可在 **macOS 26 Tahoe** 上复现全部结果（触控板 / USB / 硬件加速 / WiFi / 蓝牙 / 声卡）。升级过程踩了两个大坑，单独写在 [§11 AuxKC 路径陷阱](#11-桌面起不来但-windowserver-在跑auxkc-路径陷阱)、[§12 无桌面时离线建账户并开-ssh](#12-没有桌面时怎么进去离线建账户--开-ssh)、[§13 选单排序](#13-opencore-选单顺序没有排序逻辑)。**同时修正了本文一处旧错误：触控板的 ACPI 补丁是必需的（见 §3.3）。**

## 目录

- [0. 一页结论表](#0-一页结论表)
- [1. 机器与环境](#1-机器与环境)
- [2. 安装与基础 OpenCore 配置](#2-安装与基础-opencore-配置)
- [3. 驱动逐项修复](#3-驱动逐项修复)
  - [3.0 显卡（前置条件）](#30-显卡rtx-3060-laptop--nullmoth-nvidia-驱动前置条件)
- [4. 启动「玄学」的真正根因：KASLR slide 抽签](#4-启动玄学的真正根因kaslr-slide-抽签)
- [5. 排障方法论与铁律](#5-排障方法论与铁律)
- [6. 最终配置快照](#6-最终配置快照)
- [7. 一键自检清单](#7-一键自检清单)
- [8. 备份与回滚策略](#8-备份与回滚策略)
- [9. 已知问题与后续方向](#9-已知问题与后续方向)
- [10. 参考项目与资料（全部仓库链接）](#10-参考项目与资料全部仓库链接)
- [11. 桌面起不来但 WindowServer 在跑：AuxKC 路径陷阱](#11-桌面起不来但-windowserver-在跑auxkc-路径陷阱)
- [12. 没有桌面时怎么进去：离线建账户 + 开 SSH](#12-没有桌面时怎么进去离线建账户--开-ssh)
- [13. OpenCore 选单顺序没有排序逻辑](#13-opencore-选单顺序没有排序逻辑)
- [附录 A：ACPI 补丁字节](#附录-aacpi-补丁字节)
- [附录 B：日志规范化对比脚本](#附录-b日志规范化对比脚本)

更深入的启动失败排查流程见 [TROUBLESHOOTING.md](TROUBLESHOOTING.md)。

---

## 0. 一页结论表

| 功能 | 最终状态 | 关键手段 | 验证判据 |
|---|---|---|---|
| **显卡 / 内屏显示 + Metal 加速** | ✅ | **`nullmoth/nvidia-macos-driver` 2.7.2**（kext 装 `/Library/Extensions`，不从 EFI 注入）+ 一组 boot-args / SIP 设置 | `kmutil showloaded \| grep nullmoth` 有 4 行；`SPDisplaysDataType` 显示 `Metal Support: Metal 3` || WiFi（MT7922 / RZ616） | ✅ | `AirportMtkWlan.kext` v1.6.0（fork），`MinKernel=24.2.0` / `MaxKernel=24.99.99` | `system_profiler SPAirPortDataType` 显示 MT7922；实测 400–600 Mbps |
| 蓝牙（同一张卡，USB `0489:e0e4`） | ✅ | `BlueToolFixup` + `SimpleMtkBtFw` + `SimpleMtkBtPatcher` + 2 个 NVRAM 变量 | `Chipset: THIRD_PARTY_DONGLE` / `State: On` / Address 非空 |
| 触控板 + 多点手势 | ✅ | VoodooI2C 五件套 + `VoodooI2CHID` + **ACPI `TPAD._CRS` 的 OSYS 旁路补丁（必需，见 §3.3）** | `TPAD` 是 `VoodooI2CDeviceNub` 且 `Interrupt Mode = GPIO`；`VoodooI2CPrecisionTouchpadHIDEventDriver` active |
| USB（不卡顿） | ✅ | 2 条 `UPCN`/`UPCC` ACPI 补丁 + `XHCILagFix.kext` | `Active=Yes`、`Failure=""`，鼠标 `busy` 数百 ms |
| 声卡（ALC287） | ✅ | `AppleALC` + `alcid=13` | 扬声器 / 耳机 / 内置麦 / Line In 均可选 |
| 有线网、键盘、电池、亮度键 | ✅ | `RealtekRTL8111` / VoodooPS2 / VirtualSMC 系列 / `BrightnessKeys` | 系统信息里可见 |
| 冷启动随机失败 | ✅ 已规避 | boot-args 固定 `slide=<已验证值>` | 日志 `MBA:OUT` 中 slide 恒定，连续启动均成功 |
| 摄像头出流 | ⚠️ 半通 | 枚举与供电正常，被 `XHCILagFix` 的 iso 缺陷阻塞 | 有设备节点但 0 帧 |
| Type-C 视频输出 | ❌ 无解 | BIOS 的 Discrete GPU only 物理切断 DP Alt Mode | 只有内屏 + HDMI 两个 connector |
| 右侧第 4 个 USB-A 的 USB 2.0 通道 | ❌ 未完成 | ACPI 端口表未定义该通道 | 有线鼠标插上零节点，U 盘（USB3）可用 |

**最重要的一条经验**：这台机器曾经被判定「装蓝牙就必然启动失败」，折腾了一整天；最后发现蓝牙配置从头到尾都是对的，真正的元凶是 **OpenCore 每次启动随机抽取的 KASLR slide 值**。把 slide 固定后，同一份「有毒」的配置一次就成功。详见 [§4](#4-启动玄学的真正根因kaslr-slide-抽签)。

---

## 1. 机器与环境

| 项 | 值 |
|---|---|
| 机型 | Acer Nitro 5 AN515-46（国内名：暗影骑士·龙 2022） |
| CPU | AMD Ryzen 7 6800H（Rembrandt / Zen3+，8C16T） |
| 核显 | Radeon 680M（未启用；EFI 里有 `RembrandtGfx.kext` 但未加入 `Kernel/Add`） |
| 独显 | NVIDIA RTX 3060 Laptop（`10de:2520`，6 GB，PCIe x8）；**内屏物理走它**，由 NullMoth 2.7.2 驱动，详见 §3.0 |
| 内存 / 屏幕 | 32 GB / 1920×1080 @165 Hz |
| 无线卡 | MediaTek MT7922 / AMD RZ616，M.2 2230，WiFi（PCIe `14c3:0616`）+ 蓝牙（USB `0489:e0e4`）二合一 |
| 声卡 | Realtek ALC287 |
| 系统 | macOS Sequoia 15.8.2 |
| SMBIOS | `iMacPro1,1` |
| 引导 | OpenCore（ESP 上 `EFI/OC/`） |

> ⚠️ **顺序提醒**：这台机器 BIOS 处于 Discrete GPU only（MUX 硬切），内屏信号由 NVIDIA 独显直接输出。
> 所以**显卡驱动是安装的前置条件，不是最后才修的一项**——没有它连桌面都看不到，后面所有 kext 调试都无从谈起。先做 §3.0，再做 3.1 起的其他项。

### 隐私说明

本文已移除所有本机凭据、内网地址、设备 MAC 与私有路径。文中出现的 `0489:e0e4`、`14c3:0616` 是硬件 ID（公开信息），`slide=142` 是**这台机器**的实测值，**不要直接照抄**，读法见 §4。

---

## 2. 安装与基础 OpenCore 配置

### 2.1 磁盘布局与「双 ESP」陷阱

本机是双 NVMe：一块装 Windows，一块单独装 macOS，**两块盘各有自己的 EFI 分区**。这带来一个反复踩中的坑：

> **ESP 的盘号每次启动都可能变化**（`disk0s3` ↔ `disk1s3`），卷名也不固定（`NO NAME` / `EFI` / 其它）。
> 曾出现过误以为挂的是 macOS ESP、实际指向 Windows 分区的情况。**每次动手前必须现查**：

```bash
diskutil list | grep -i EFI          # 先看清楚这次是哪个 diskXsY
ls -1 /Volumes/                     # 卷名也要现查
```

macOS 下 `diskutil mount diskXsY` 常报 `volume damaged`，但 `fsck_msdos -n` 检查往往没有任何错误。此时用底层挂载可以成功：

```bash
sudo mkdir -p /Volumes/ESPX
sudo mount -t msdos /dev/diskXsY /Volumes/ESPX     # 盘号替换成现查结果
```

Windows 侧挂载 ESP 后是另一个盘符，改 config 更方便；但要注意**你改的是哪块盘上的 OC**。可以用 NVRAM 里的 `boot-path` 反查 OpenCore 实际从哪个分区加载：

```bash
nvram 4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102:boot-path
```

### 2.2 基础 kext 与 SMBIOS

基础组合是 AMD 平台的常规集合：`Lilu`（1.7.3）、`VirtualSMC` + 三个 SMC 插件、`WhateverGreen`、`AppleALC`、`AMFIPass`、`AppleMCEReporterDisabler`、`NVMeFix`、`RestrictEvents`、`BrightnessKeys`、`RealtekRTL8111`、VoodooPS2 四件套、VoodooI2C 五件套。完整加载顺序见 [§6](#6-最终配置快照)。

SMBIOS 用 `iMacPro1,1`（无核显加速的机型，适合独显/无 Metal 核显驱动的场景）。序列号、MLB、ROM 等请自行生成合法值，**本文不提供任何真实平台信息**。

> `SetupVirtualMap`：Dortania 文档建议 2020 年后的 AMD 新固件设为 `False`。但在这台机器上，`True` 和 `False` 都成功过、也都挂过，**它不是决定变量**（真凶见 §4）。所以本文保留原值 `True`，不把它当作修复手段。

### 2.3 Booter Quirks（AMD 平台）

```
ProvideCustomSlide      = True       ← 每次启动随机抽 slide，§4 的关键
ProvideMaxSlide         = 0
RebuildAppleMemoryMap   = True
SyncRuntimePermissions  = True
SetupVirtualMap         = True
DevirtualiseMmio        = False      ← 导致 MmioWhitelist 全部失效
MmioWhitelist           = 3 条 Enabled=True
```

**`MmioWhitelist` 只在 `DevirtualiseMmio=True` 时生效。** 本机三条白名单（`0xFED80000` / `0xFDE00000` / `0xFF000000`）虽然都勾了 Enabled，但因为总开关是关的，一直是摆设。日志里因此出现：

```
OCABC: Only 176/256 slide values are usable!
OCABC: Valid slides - 80-255
```

80 个 slide 值因 MMIO 区域冲突被排除，这就是抽签容易抽到「坏值」的背景。白名单注释里其实早就记着故障地址 `0xFFF8B04C`——地址定位过，只是没让白名单生效。

### 2.4 boot-args

```
-v keepsyms=1 debug=0x100 rtcfx_exclude=58-59,80-AB,B0-B7 \
nvfb=1 nvcursor=0 nvaccel=1 nvfbheads=4 -nvkmsnosmooth \
amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0 \
gfxrst=4 alcid=13
```

拆开看，前面这一大串里**有一半是给显卡的**（详见 §3.0）：

| 参数组 | 用途 |
|---|---|
| `nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth` | NullMoth 帧缓冲 + 加速器 + 4 个显示头 |
| `amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0` | 让 WindowServer 能加载未签名的 Metal 驱动 bundle |
| `gfxrst=4` | **与 WhateverGreen 共存时必加**，否则开机画面拷贝会读到已释放内存导致 panic |
| `nvcursor=0` | 本机额外参数（光标相关） |
| `alcid=13` | 声卡 layout |

最终还要在末尾追加固定的 `slide=N`（见 §4），**不要把它写死成 142**。
注意 `boot-args` 里**不能**出现 `nv_disable=1`、`-wegnoegpu`、`agdpmod=pikera`——它们会隐藏 NVIDIA 卡。

---

## 3. 驱动逐项修复

每一项都按「症状 → 根因 → 做法 → 验证」组织。

### 3.0 显卡：RTX 3060 Laptop + NullMoth NVIDIA 驱动（前置条件）

**为什么必须先做这一项**：这台机器 BIOS 是 Discrete GPU only，内屏通过 MUX 硬连到 NVIDIA 独显。
AMD Radeon 680M 核显虽然在 PCI 总线上可见（`1002:1681`），但 `No Kext Loaded`，且被
`disable-gpu=1` 主动禁用。**没有任何显卡驱动 = 没有桌面**，安装盘之后的所有调试都做不了。

**症状（未装驱动时）**：能进系统但只有固件帧缓冲、分辨率固定、无 Metal、WindowServer 无加速，
或者干脆黑屏。macOS 自带 NVIDIA 驱动早在 Mojave 之后就被移除，Web Driver 也早已停更——传统路线全部无效。

**根因与解法**：用 `nullmoth/nvidia-macos-driver`（**2.7.2**，2026-11-05 发布）。它不是「补个 kext」，
而是把 NVIDIA 官方开源 GPU 内核模块（r610）移植到 macOS，再叠一套自研 Metal 实现：

```
Metal app ─► NVMTLDriver.bundle ─► 着色器翻译器 (Apple AIR → SPIR-V)
          ─► NVK (Mesa Vulkan + NAK 编译器) ─► kexts ─► GPU
```

| 组件 | 磁盘位置（本机实测均存在） | 作用 |
|---|---|---|
| `NVRM.kext` / `NVAccel.kext` / `NVRMFB.kext` / `NVRMAGDC.kext` | `/Library/Extensions/` | 内核侧四个 kext，全部 `2.7.2` |
| `NVMTLDriver.bundle` | `/Library/GPUBundles/` | Metal 驱动插件 |
| `nvmtl/` | `/Library/GPUBundles/` | Vulkan 后端（NVK） |
| GSP 固件 | `/Library/NullMoth/nvfw/nvidia/615.78.08` | NVIDIA 原版固件，未修改 |

> ⚠️ **最容易搞错的一点**：这四个 kext 装在**系统卷** `/Library/Extensions`，由 Auxiliary Kernel Collection 加载。
> **绝对不要再把它们复制进 `EFI/OC/Kexts` 并从 `Kernel/Add` 注入**——双份加载会冲突。
> OpenCore 在这里的职责只有两个：设置 SIP 和 boot-args。

#### 3.0.1 OpenCore 必需设置（本机逐项核对过官方 README）

| 位置 | 值 | 为什么 |
|---|---|---|
| `NVRAM → Add → csr-active-config` | `430A0000`（Data，即 `0x0A43`） | 允许未签名 kext + root patch；SIP 不能全开 |
| `Misc → Security → SecureBootModel` | `Disabled` | Apple 安全启动会拒绝非 Apple 签名的 kext |
| `boot-args` | `nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0` | 帧缓冲 + 加速器 + 4 显示头；AMFI 参数让 WindowServer 加载驱动 bundle |
| `boot-args` 追加 | `gfxrst=4` | **装了 WhateverGreen 就必加**，否则 NVIDIA 显示打开时 WEG 的开机画面拷贝会读已释放内存 → 内核崩溃 |
| `Kernel → Add` | `Lilu.kext` + `AMFIPass.kext` 启用 | AMFI 豁免 |
| `Kernel → Block` | `com.apple.iokit.IONDRVSupport`，`Strategy=Exclude` | 否则固件帧缓冲会抢走 display index 0 |
| `SMBIOS` | `iMacPro1,1` | 能跑 macOS 15 且带独显的机型（也是官方安装器使用的机型） |
| `DeviceProperties` | **NVIDIA 卡不需要任何属性** | 官方明确说明；不要抄别人的 `AAPL,ig-platform-id` 之类 |
| `UEFI → Quirks → ResizeGpuBars` / `Booter → Quirks → ResizeAppleGpuBars` | 保持固件原值（本机均为 `-1`） | 强制改 BAR 会让固件在 picker 之前卡住；**不要照抄别的主板** |
| BIOS | Above 4G Decoding **ON**、CSM **OFF** | 显卡需要映射 4 GB 以上内存 |
| 禁止出现 | `nv_disable=1`、`-wegnoegpu`、`agdpmod=pikera`、WhateverGreen 的 NVIDIA 补丁 | 都会隐藏/干扰这张卡 |

同时把这些键列入 `NVRAM → Delete`，让每次启动都重新写入。本机 `DeviceProperties` 里只有三条，
**没有一条是给 NVIDIA 的**：`acpi-wake-type=1`（网卡唤醒）、`disable-gpu=1`（禁用 AMD 核显）、
`built-in` + `layout-id`（声卡）。

#### 3.0.2 安装

**推荐**：官方 [1401](https://github.com/nullmoth/1401) Mac app。它会找到启动用的 OpenCore，
改 config 前逐项展示变更、自动备份，并安装驱动；还会在 OC 启动菜单加一条
**「1401: Remove NVIDIA driver」**，选它即可在下次启动时完整卸载并还原 config（含 OpenCore 设置）。
这个一键回滚入口在不稳定机器上非常有价值。

**手动**（自己管理 OpenCore 时）：

```bash
# 从 Releases 下载 nullmoth-nvidia-<version>.tar.gz
tar -xzf nullmoth-nvidia-*.tar.gz && cd pkgroot
shasum -a 256 -c SHA256SUMS      # 每个文件必须 OK
sudo ./install.sh                # 复制文件并重建 AuxKC
sudo shutdown -r now             # 必须重启，注销不会加载驱动
```

首次会卡在 **「PCI configuration end」最长约 1 分钟**（GPU 上电），这是正常的，别急着断电。
重启后到「系统设置 → 隐私与安全性」**允许这些系统扩展**，然后**再重启一次**。
本机还有一个 `nullmoth-nv-ok=1` 的 NVRAM 变量，是安装成功标记。

> 官方要求 macOS **15.5+ Sequoia 或 26 Tahoe**、x86_64、Turing 及更新的卡。
> 开发验证主要在 RTX 5060 上进行；官方 `CARD-SUPPORT.md` 把 **RTX 30 笔记本卡标为 `Pending`**，
> 也就是说本机属于「实测可用、但不在官方确认表内」。同代不同型号结果可能不同。

#### 3.0.3 验证

```bash
kmutil showloaded --list-only | grep nullmoth       # 期望 4 行
kextstat | grep -i nullmoth                         # 4 个 kext，均为 2.7.2
system_profiler SPDisplaysDataType | head -25       # Metal Support: Metal 3
```

本机实测输出：

```
NVIDIA GeForce RTX 3060 Laptop GPU:
  Vendor: NVIDIA (0x10de)   Device ID: 0x2520   VRAM: 6 GB   Bus: PCIe x8
  Metal Support: Metal 3
  Resolution: 1920 x 1080   UI Looks like: 1920 x 1080 @ 165.00Hz
  Framebuffer Depth: 24-Bit Color (ARGB8888)   Connection Type: Internal   Online: Yes

AMD (0x1002) 0x1681 → Kernel Extension Info: No Kext Loaded   ← 核显按预期禁用
```

内核日志里的正常渲染证据：`NVMTL over NVIDIA GeForce RTX 3060 Laptop GPU (NVK GA106-A)`、
`NVRMFB: flipToMemory ... -> ok 1 rc 0`。看到这些就说明 Metal 与窗口合成真的在跑，
不只是「设备被枚举」。

#### 3.0.4 已知代价与风险

| 项 | 说明 |
|---|---|
| **安全性下降** | 未签名 kext 要求 SIP 降到 `0x0A43`、`SecureBootModel=Disabled`。FileVault、DRM、部分系统更新行为会受影响，请自行权衡 |
| **系统更新后可能失效** | 安装 macOS 安全更新/大版本会重建 AuxKC，驱动可能不再加载，需要重新跑安装器。更新前先整包备份 EFI |
| **睡眠 / 唤醒** | 官方有专门的 `docs/SLEEP-WAKE.md`；睡醒丢 GPU 时可加 `-nvrmnopm` 让 NVRM 退出电源树 |
| **画面闪烁 / 撕裂** | 官方建议先试 `-nvrmnoflip`（走拷贝路径，不做零拷贝 scan-out） |
| **笔记本输出路线** | 官方明确提醒「卡被支持 ≠ 这台笔记本能用」，还要有 NVIDIA 能驱动的输出通路。本机是 MUX 直连内屏，所以成立；**Type-C 视频输出仍无解**（见 §3.8） |
| **与 WhateverGreen** | 必须 `gfxrst=4`，这是官方点名的 panic 修复 |

可选排障参数还有 `-nvrmnogo`（驱动待命，手动 `nvrmctl go` 启动）、`-nvrmnobootscreen`（不快照开机画面）。

---

### 3.1 WiFi：MT7922 并非无解

**症状**：系统里完全没有 WiFi 接口；网上大量「MT7922 / RZ616 黑苹果无驱动，只能换卡」的说法。

**根因**：上游项目不支持 PCI ID `14c3:0616`，但**社区 fork 已经支持了**。不要用「上游没支持」推断「无解」，先搜 fork 和 PR。

**做法**：使用 `AirportMtkWlan.kext` v1.6.0（源自 `e5u/SimpleMtkWlan` 这一分支），加入 `Kernel/Add`：

```
BundlePath : AirportMtkWlan.kext
MinKernel  : 24.2.0
MaxKernel  : 24.99.99       ← 必须设！装 25.x/Tahoe 版本会直接 panic
```

前提条件（缺一不可）：`DisableIoMapper = true`；`Kernel/Block` 里没有屏蔽 `IOSkywalkFamily`；没有 OCLP 的无线补丁；网卡 `vendor-id` 没被隐藏；主机有 MSI 中断。

**已知限制**：不支持 WPA3-SAE（用 WPA2）；不支持 AirDrop / Handoff / AWDL / Continuity，因为它不是原生 Airport 卡。

**验证**：

```bash
ifconfig en2 | grep -E 'status|inet '
networksetup -getairportnetwork en2
system_profiler SPAirPortDataType | head -30
# 期望：Firmware 显示 MT7922，协商 780 Mbps，实测 400–600 Mbps
```

### 3.2 蓝牙：一次「冤案」与最终解法 ★

**症状**：USB 设备 `0489:e0e4` 能枚举出 `BT_ACL_If` / `BT_SCO_If` / `BT_ISO_If` 三个接口，但系统蓝牙显示 `Chipset: BCM_4350C2`、`Transport: UART`、`Address: NULL`、`State: Off`，完全不可用。

**根因**：`bluetoothd` 走了「内置 Apple 蓝牙模块」的路径，而本机是第三方 USB 蓝牙。需要让它改走 `THIRD_PARTY_DONGLE` 路径，并由固件加载器把 MT7922 的 RAM code 灌进芯片。

**做法**（最终成功方案，kext 版本与顺序很关键）：

1. 三个 kext 按此顺序放在 `Lilu` 之后：
   ```
   BlueToolFixup.kext       2.7.2    （BrcmPatchRAM 项目自带）
   SimpleMtkBtFw.kext       1.1.0    ← Info.plist 含 MT7922A_0489_e0e4 personality
   SimpleMtkBtPatcher.kext  1.1.0    （Lilu 插件，修 HCI 行为）
   ```
2. `NVRAM/Add` 增加两个变量，并同时写入 `NVRAM/Delete`（让每次启动重新注入）：
   ```
   bluetoothExternalDongleFailed    = 00
   bluetoothInternalControllerInfo  = 14 × 00
   ```
3. `boot-args` 末尾固定 `slide=<已验证值>`（见 §4）。**这一步不是蓝牙的一部分，但它决定了你能不能稳定进系统去验证蓝牙。**
4. 其它区段（Booter / ACPI / DeviceProperties / PlatformInfo）保持不动。

**验证**（官方判据）：

```bash
system_profiler SPBluetoothDataType
# 期望：
#   Chipset: THIRD_PARTY_DONGLE   ← 最关键
#   State: On
#   Address: <非空真实 MAC>
#   Transport: USB
#   Firmware Version: v9489 c6166
#   Supported services: HFP AVRCP A2DP HID ...
kextstat | grep -iE "BlueTool|SimpleMtkBt"
```

成功后蓝牙耳机会出现在 `system_profiler SPAudioDataType` 里，`Transport: Bluetooth`，输出 44.1 kHz 双声道、输入 8 kHz（HFP），实测可正常听音与通话。

> **这段最值得复盘**：上午装上这三个 kext 后连续启动失败，当时判定「蓝牙方案有毒」并整体回滚。下午把成功与失败的 OpenCore 日志规范化逐行对比，才发现同一份 config、同样的 `Prelink size`、同样的 kext 列表，**唯一变量是随机抽取的 slide 值**。固定 slide 后，原封不动地把那三件套加回来，**一次就成功**。
> 结论：**先排除运行时随机性，再判断因果**，否则会把运气问题当成方案问题。

**备选方案（本次未使用，仅存档）**：`andy-walker-idfa/RZ616-Bluetooth-macOS` 是 `BlueToolFixup` 的 fork，走 SRS / 用户态灌固件路线，二进制里硬编码 `0489:e0d8` 与 `0e8d:0616` 两组 ID。若要用于 `0489:e0e4`，需要改 2 个字节（`d8 e0 → e4 e0`）。该仓库 0 stars、无实测案例、用较新 SDK 编译，**风险中高**，方案一成功后无需再试。

### 3.3 触控板：ACPI OSYS 旁路补丁**是必需的**（2026-11-08 修正）

> ⚠️ **本节此前写错了。** 旧版说「不需要 ACPI 补丁、相关不等于因果」，那个结论是在 macOS 15 上热启动、`OSYS` 恰好被抬高的偶然情况下验证的，把运气当成了规律。macOS 26 冷启动能从反面证明：没有这条补丁，`ELAN050A` 根本建不出 nub。

**症状**：触控板完全无反应，日志里有：

```
VoodooI2CDeviceNub::TPAD Could not find an I2C Serial Bus declaration
Could not initialise nub for ELAN050A
```

**机理**：`TPAD._CRS` 里有这样一个门：

```asl
If (OSYS < 0x07DC) { Return (SBFI) }     // 老系统走这条
Return (SBFB + SBFG)                     // 新系统走这条：I2C Serial Bus + GpioInt
```

`SBFI` 这个 buffer 里**只有 GPIO 中断，没有 I2C 从机地址描述符**，所以 VoodooI2C 拿不到总线地址，nub 建不出来，后面整条链（`VoodooI2CHID` → `VoodooInput` → 触控板）全部断掉。

而 `OSYS` 是个 SystemMemory 字段，由操作系统写进去、靠内存残留维持。**冷启动时它是低值，热启动时可能还留着上一次的高值** —— 这就解释了那种「有时候好有时候坏、重启一次又好了」的随机感。别把它当成 kext 不稳定。

**补丁（8 字节等长，只把比较常量改成 0，让条件恒假）**：

```
Find    : 95 4F 53 59 53 0B DC 07      ; If (OSYS < 0x07DC)
Replace : 95 4F 53 59 53 0B 00 00      ; If (OSYS < 0) → 恒假
```

字节解读：`4F 53 59 53` = "OSYS"，`0B DC 07` = Integer 0x07DC，`0B 00 00` = Integer 0。前面那个 `95` 是比较操作码。**把比较常量归零，使这个分支不再成立**；DSDT 全表**只命中 1 处**，`Count = 1`，不会误伤。判断补丁是否真生效，**不要看反编译结果，直接看 `ioreg` 里 `Interrupt Mode` 是不是变成了 `GPIO`**。

**前提**：`VoodooGPIO` 必须加载 —— 补丁让 `_CRS` 返回 `SBFG`（GpioInt），而 GpioInt 资源要挂到 `\_SB.GPIO` 上才解析得出来，那个控制器就是 `VoodooGPIO` 提供的。

**正确做法**：五件套全部启用，且顺序上 `Lilu` / `VoodooI2CServices` / `VoodooGPIO` 在 `VoodooI2C` 之前，`VoodooInput` 插件在 `VoodooI2C` 之后：

```
Lilu.kext
VoodooI2C.kext/Contents/PlugIns/VoodooI2CServices.kext
VoodooI2C.kext/Contents/PlugIns/VoodooGPIO.kext
VoodooI2C.kext                                  ← 主驱动
VoodooI2C.kext/Contents/PlugIns/VoodooInput.kext
VoodooI2CHID.kext
```

**验证（macOS 26 实测输出，按这个对）**：

```bash
# 1) nub 建出来了没有 —— 这一条最关键
ioreg -p IOService -n TPAD -r -d 3
#   +-o TPAD  <class VoodooI2CDeviceNub, ... registered, matched, active>
#       "IOName"         = "ELAN050A"
#       "Interrupt Mode" = "GPIO"        ← 不是 "APIC"，也不是缺失
#       "gpioPin"        = 9
#       "i2cAddress"     = 21
#       "sclHz"          = 400000

# 2) 驱动链起来了没有
ioreg -c IOHIDDevice -r -w0 | grep -i "magic trackpad"
#   +-o VoodooI2CPrecisionTouchpadHIDEventDriver <... active>
#       "Product"      = "Magic Trackpad 2"      ← VoodooInput 的模拟名，不是真苹果硬件
#       "SerialNumber" = "Voodoo Magic Trackpad 2 Simulator"

# 3) kext 加载情况（macOS 26 上 kextstat 已不推荐，用 kmutil）
kmutil showloaded --list-only | grep -iE "voodooi2c|voodooinput|voodoogpio"
```

**反面判据**：只要 `ioreg -p IOService -n TPAD` 里看不到 `VoodooI2CDeviceNub`，或者 `Interrupt Mode` 不是 `GPIO`，就是补丁没打上 / `VoodooGPIO` 没加载，**跟 kext 版本无关，别去换 VoodooI2C 的版本**。

> 顺带一条纪律：`AppleMultitouchDevice` 注册成功**不能**当成触控板可用的证据。`VoodooInput` 只要加载就会注册那个模拟设备，哪怕一条真实数据都收不到。

### 3.4 USB：两条 ACPI 补丁 + XHCILagFix

本机有 5 个 XHCI 控制器，症状分两类，别混为一谈。

#### (a) 端口「不可连接」→ 设备完全不出现

ACPI 里 `UPCN` / `UPCC` 的第一个字节是 `0`（不可连接）时，macOS 根本不给控制器上电：

```
CurrentPowerState = 0
kPowerStateOff = 700690 ms (99%)     ← 99% 时间处于关机态
```

两条等长补丁，只改第一个元素 `0 → 1`：

```
# UPCN：救活 XHC0 / XHC1 的端口
Find    : 08 55 50 43 4E 12 07 04 00 0A FF 00 00
Replace : 08 55 50 43 4E 12 07 04 01 0A FF 00 00

# UPCC：救活 XHC2.PRT1 与摄像头 UCAM
Find    : 08 55 50 43 43 12 07 04 00 0A FF 00 00
Replace : 08 55 50 43 43 12 07 04 01 0A FF 00 00
```

字节解读：`08` = NameOp，`55 50 43 4E` = "UPCN"，`12 07 04` = Package(4)，`00`/`01` = 首元素，`0A FF` = Byte 255。

#### (b) 设备能用但极度卡顿 → XHCILagFix

**症状（与 (a) 完全不同）**：鼠标插上能识别，但**移动一下要 3–4 秒光标才动、且是瞬移**；U 盘能识别却读不出数据。这是 AMD 平台 XHCI 完成事件通知路径约 50 ms 延迟的典型表现。

**做法**：`XHCILagFix.kext`（Lilu 插件，实验性项目）把 Control/Bulk/Interrupt 传输的完成事件从 Interrupter 1 重定向回 Interrupter 0 / Ring 0。

```
BundlePath     = XHCILagFix.kext
ExecutablePath = Contents/MacOS/XHCILagFix
PlistPath      = Contents/Info.plist
Arch           = x86_64
```

安装前提：放在 `Lilu` 之后；`Lilu >= 1.7.3`；没有 `GenericUSBXHCI` 等重复改写 USB 栈的旧插件。它支持用 `-xhcilagfixoff` 启动参数临时禁用，方便对比测试而不用反复改 config。

**验证**：

```bash
ioreg -r -c XHCILagFix -l -w0 | grep -E '"Active"|"Failure"|"Mode"'
ioreg -p IOService -w0 -r -c IOUSBHostDevice | grep "busy"
# Active=Yes / Failure 为空 = 三个 Hook 装上了
# 鼠标节点 busy 从 7949 ms 降到 320 ms 左右 = 延迟问题已解决
```

**代价（重要）**：它把**等时（iso）传输**也重定向了，而 iso 完成必须走 `AppleUSBXHCI::filterOccurred → isochronousFilterInterrupt()`。落到 Ring 0 的 iso 永不完成，于是**摄像头有供电、有枚举、就是拿不到帧**。这是一个二选一：要 USB 不卡顿，就得接受摄像头不出流（上游 issue #3 有未合并的修法）。

> ⚠️ 项目自述为 **experimental**，作者只在 Ryzen 9 8945HS / ROG Zephyrus G14 上验证过。本机 6800H 属于同一机理但不在官方验证范围内，实测可用。

**遗留**：右侧第 4 个 USB-A 的 USB 3.0 通道正常（U 盘可用），但 USB 2.0 通道在 ACPI 端口表里根本没定义（有线鼠标插上零节点）。要定性是硬件还是 ACPI 缺项，可在 Windows 下插同一个鼠标对比。

### 3.5 声卡

`AppleALC.kext` + boot-args `alcid=13`，对应 Realtek ALC287。扬声器、耳机、内置麦、Line In 均可用。换 layout 时记得清 NVRAM 或用 `alcid=` 覆盖，避免旧 layout 缓存。

```bash
system_profiler SPAudioDataType | grep -iE "Output Source|Default Output|Transport"
```

### 3.6 键盘 / 电池 / 亮度 / 有线网

| 组件 | kext | 备注 |
|---|---|---|
| 内置键盘 | `VoodooPS2Controller` + Keyboard/Mouse/Trackpad 插件 | `VoodooInput` 插件保持关闭，避免与 I2C 版冲突 |
| 电池 / 光感 | `SMCBatteryManager` / `SMCLightSensor` | 依赖 `VirtualSMC` |
| 亮度键 | `BrightnessKeys` | 无需 DSDT 补丁 |
| 有线网 | `RealtekRTL8111.kext` | — |

> **教训**：排障时做减法关掉 kext 之后，必须列清单逐项加回。本项目里「USB 突然不能用」「触控板突然不能用」两次假故障，都是关掉后忘记加回造成的，白白绕了一大圈。

### 3.7 摄像头（半通）

设备 `ACER HD User Facing` 能枚举、能供电，但 0 帧。当前根因指向 `XHCILagFix` 的 iso 缺陷（见 3.4）。临时关掉 `XHCILagFix` 可以验证这个判断，但会让 USB 卡顿回归，验证完请立刻加回。

### 3.8 Type-C 视频输出（无解）

Acer 官方说明：**启用 BIOS 的「Discrete GPU only」时，Type-C 不支持 DisplayPort 视频输出**——这是 MUX 物理层面的切断，不是驱动能补的。

三条佐证：内屏挂在 RTX 3060 上（当前就是 Discrete 模式）；日志恒定 `connected display(s) of 2 handle(s)`，connector 只有内屏 `0x2` 和 HDMI `0x1`；Type-C 屏在跑码阶段（UEFI 帧缓冲）能显示，一进桌面就冻结在最后一帧。

理论上切 BIOS 到 Optimus 模式并启用 `RembrandtGfx.kext` 可能恢复，但作者自述该 kext「还没优化好」，切模式后内屏改走核显很可能直接黑屏且无法进系统改回，**风险远大于收益，不建议尝试**。实用替代是 DisplayLink USB 显卡扩展坞。

---

## 4. 启动「玄学」的真正根因：KASLR slide 抽签

### 4.1 现象

- 同一份 config，有时能进、有时卡在 `EXITBS:START` 或白苹果后自动重启
- 每次卡点还不一样：`VoodooGPIOAMD`、`VoodooI2CACPIController`、直接黑屏都出现过
- reset NVRAM 有时「有效」、放电无效、改 BIOS 时间无效 → 看起来完全随机

### 4.2 定位

`ProvideCustomSlide = True` 会让 OpenCore 每次从有效范围里**随机**选一个 KASLR slide 追加到 boot-args。把一份失败日志与一份成功日志**规范化后逐行 diff**（剥掉相对时间戳、ISO 时间、十六进制地址、CPU 频率抖动），得到决定性结论：

| 时间 | slide | 结果 |
|---|---|---|
| 11:06 | `122` | ✅ 成功 |
| 14:42:33 | `185` | ❌ 挂 |
| 14:45:06 | `98` | ❌ 挂 |
| 14:45:40 | `142` | ✅ 成功 |

两次的 `Prelink size` / `kext offset` / `reserved` 三个数值完全相同（74993664 / 66134016 / 8859648），注入的 kext 列表、ACPI 表顺序也完全一致，**实质差异只剩 `slide=`**。失败发生在 XNU 接管之后，所以 OpenCore 日志里看不出任何区别。

完整流程、脚本和「常见误判表」见 [TROUBLESHOOTING.md](TROUBLESHOOTING.md)。

### 4.3 修复

Dortania / OpenCore 配置文档原文：

> Provide custom KASLR slide for low memory maps. **This option has no effect when `slide=value` boot argument is present**, which needs to be removed for this quirk to work.

也就是**手动写 `slide=N` 会让 `ProvideCustomSlide` 自动失效**，随机抽签被彻底关掉——这是官方机制，不是 hack。

步骤：

1. 从**你自己机器上一次成功启动**的日志里读出 slide 值：
   ```bash
   tr -d '\000' < /Volumes/ESPX/opencore-*.txt | grep -o 'slide=[0-9]*' | tail -1
   ```
2. 把 ` slide=N` 追加到 `NVRAM/Add/<boot-args GUID>/boot-args` 末尾。
3. 重启后从日志 `AAPL: #[EB|MBA:OUT]` 确认最终值确实是 N 且不再变化，再连续验证多次冷启动。

> **不要照抄 142**。slide 值取决于你自己的内存映射和 MMIO 布局，必须取本机实测成功值。

另一个可选方向是让 `DevirtualiseMmio = True` 使三条 `MmioWhitelist` 生效，减少被排除的 slide 值；但它会引入新的内存映射变化，优先级低于固定 slide。

### 4.4 三个让排查走偏的坑

1. **`EXITBS:START` 不是卡死判据**。它是 OpenCore 能写的**最后一行**，成功和失败的日志都是 1014 行左右、都以它结尾。真正的失败发生在这一行之后。
2. **ESP 根目录堆太多日志会让 OC 写不进新文件**。曾出现堆了 113 个 `opencore-*.txt` 后，十几次启动零日志，一度误判成「OC 没运行」。清理后立刻恢复。建议保留最近 5–10 个，或关掉文件日志输出。
3. **日志文件名是 UTC，且 FAT32 时间戳不可靠**。北京时间 16:05 的日志叫 `...-080531.txt`，`ls -t` 的排序会被 Windows 写入的本地时间戳骗到。**按文件名排序找最新日志**。

---

## 5. 排障方法论与铁律

1. **改之前先整包备份**，不要只备份 `config.plist`。三层：ESP 上 `config.plist.bak-xxx` → 整个 `EFI/` 打包成 tgz → 复制到另一台机器/云盘，MD5 三方核对。
   > 但也要会做逐文件对比：本项目验证过除 `config.plist` 外，`Kexts/` `ACPI/` `Drivers/` `BOOT/` 共 179 个文件与黄金版逐字节一致。确认了文件层没被动过，单换 config 才成立。
2. **排除随机性之后再判断因果**。固定 slide 之前，任何「加了这个就挂、去掉就好」的结论都可能是抽签。
3. **一次只改一个变量**，并记下改了什么。
4. **不要用「用户报了某个设备坏了」倒推硬件根因**，先查它是不是被自己关掉了。本项目的 USB 和触控板两个大坑都是这么来的。
5. **实测优先**：用 `ioreg` / `kextstat` / `system_profiler` / `log show`，不要用截图或推测判断设备状态。
6. **对比日志前先规范化**，否则 diff 会被时间戳和地址淹没（实测原始 diff 1986 行 → 规范化后 35 行）。脚本见[附录 B](#附录-b日志规范化对比脚本)。
7. **reset NVRAM 之后绝对不能打断**，让它完整跑完一次启动循环；半写状态比不 reset 更糟。
8. **改完 ESP 卸载卷**再重启，避免写入冲突。
9. **冷启动 ≠ 热启动**，验证稳定性必须用「关机 → 开机」。
10. **不要用「上游不支持」推断「无解」**，先搜 fork、PR 和 issue（MT7922 的 WiFi 和蓝牙都是这么找到出路的）。

---

## 6. 最终配置快照

**启用 26 个 kext**（顺序即 `Kernel/Add` 顺序）——注意这里**不包含** NullMoth 的 4 个 kext：
它们装在系统卷 `/Library/Extensions`，由 AuxKC 加载，不经过 OpenCore。见 §3.0。

```
 0  Lilu.kext                                    ← 依赖基座 1.7.3
 1  BlueToolFixup.kext          2.7.2          ┐
 2  SimpleMtkBtFw.kext          1.1.0          ├ 蓝牙（本次修复）
 3  SimpleMtkBtPatcher.kext     1.1.0          ┘
 4  AirportMtkWlan.kext         Min=24.2.0 Max=24.99.99   ← WiFi
 5  XHCILagFix.kext                           ← USB 不卡顿
 6  AMFIPass.kext               Min=20.0.0
 7  AppleALC.kext                             ← alcid=13
 8  AppleMCEReporterDisabler.kext
 9  BrightnessKeys.kext
10  NVMeFix.kext
11  RealtekRTL8111.kext
12  RestrictEvents.kext
13  VirtualSMC.kext
14  SMCBatteryManager.kext
15  SMCLightSensor.kext
16  VoodooI2C/PlugIns/VoodooI2CServices.kext   ┐
17  VoodooI2C/PlugIns/VoodooGPIO.kext          │
18  VoodooI2C.kext                             ├ 触控板
19  VoodooI2C/PlugIns/VoodooInput.kext         │
20  VoodooI2CHID.kext                         ┘
21  VoodooPS2Controller.kext                   ┐
22  VoodooPS2/PlugIns/VoodooPS2Keyboard.kext   │ 内置键盘
23  VoodooPS2/PlugIns/VoodooPS2Mouse.kext      │
24  VoodooPS2/PlugIns/VoodooPS2Trackpad.kext   ┘
25  WhateverGreen.kext
```

**关闭的条目**（历史遗留，不要动）：`USBToolBox.kext`、`UTBDefault.kext`、`VoodooPS2/PlugIns/VoodooInput.kext`、`USBMap.kext`（文件不存在）、`RTCMemoryFixup.kext`、`RembrandtGfx.kext`（未加入列表）。

**ACPI**：3 条 Patch（`_OSI→XOSI` rename、`UPCN`、`UPCC`）+ 4 个启用 SSDT（`SSDT-EC` / `SSDT-PLUG-ALT` / `SSDT-USBX` / `SSDT-XOSI`）。
`SSDT-USB-Reset.aml` 保持禁用——它在 Darwin 下把 `RHUB._STA` 置 0 会干掉所有 USB 端口。

> **注意：没有 TPAD / OSYS 补丁。** 见 §3.3。

**NVRAM/Add**：蓝牙两个变量 + `csr-active-config = 430A0000` + boot-args（含 §3.0 的显卡参数，末尾追加 ` slide=N`）。
**Kernel/Block**：`com.apple.iokit.IONDRVSupport` / `Exclude`（显卡必需）。
**Misc/Security/SecureBootModel**：`Disabled`（显卡必需）。
**NVRAM/Delete**：同名两个蓝牙变量。

---

## 7. 一键自检清单

进系统后按顺序跑一遍，任何一项不符就回到对应章节：

```bash
# 显卡（最先查：没它就没有桌面）
kmutil showloaded --list-only | grep nullmoth                 # 期望 4 行
system_profiler SPDisplaysDataType | grep -E 'Chipset Model|VRAM|Metal Support|UI Looks like'
nvram nullmoth-nv-ok                                       # 1 = 安装器成功标记

# 蓝牙
system_profiler SPBluetoothDataType | grep -E 'Chipset:|State:|Address:|Transport:'
kmutil showloaded --list-only | grep -iE 'BlueTool|SimpleMtkBt'

# WiFi
networksetup -listallhardwareports | grep -A2 -i Wi-Fi
system_profiler SPAirPortDataType | grep -E 'Card Type|Firmware Version|Status:'

# 触控板（两层都要查，缺一不可）
ioreg -p IOService -n TPAD -r -d 3 | grep -E "VoodooI2CDeviceNub|Interrupt Mode|gpioPin"
ioreg -c IOHIDDevice -r -w0 | grep -i "magic trackpad"
kmutil showloaded --list-only | grep -iE "voodooi2c|voodooinput|voodoogpio"   # 期望 5 行

# USB（唯一判据，别用别的指标替）
ioreg -r -c XHCILagFix -l -w0 | grep -E '"Active"|"Failure"'
#   正常：Active = Yes 且 Failure = ""
#   异常：Active = No / Failure = "waiting for AppleUSBXHCI"  → 补丁没装上

# 启动参数
nvram boot-args
```

**关于 USB 判据的一点经验**：端口数对不对、设备枚举全不全、日志里没有 error —— **这些全都不能证明 USB 正常**。补丁没装时，这些指标一样漂亮，但鼠标照样延迟三四秒、U 盘只能识别不能读。只有 `Active = Yes / Failure = ""` 才算数。

**关于显卡判据**：`WindowServer` 在跑 ≠ 显卡正常。这个组合（内核起得来、用户态起得来、WindowServer 也在跑、屏幕却冻在 `-v`）几乎都是 AuxKC 里驱动没装进去，去查 `/Library/KernelCollections/AuxiliaryKernelExtensions.kc` 是不是被重建过，见 §11。

一个容易忽略的小问题：MT7922 是 WiFi/蓝牙二合一卡，蓝牙固件上传后偶尔会出现开机时 WiFi 电源为 Off 的情况。驱动和固件都正常，只是电源状态：

```bash
networksetup -setairportpower en2 on     # 接口名用上面查到的实际值
```

若每次冷启动都需要手动开，可以在系统设置里固定或加到登录项。

---

## 8. 备份与回滚策略

```
改之前：
  1) cp EFI/OC/config.plist EFI/OC/config.plist.bak-<日期-原因>
  2) tar -czf EFI-<状态>-<日期>.tgz EFI/           # 整个 EFI，含 Kexts/ACPI/Drivers
  3) scp 到第二台机器 / 云盘，MD5 三方核对
  4) plutil -lint EFI/OC/config.plist                # 语法校验
```

回滚时**优先整包还原**，而不是只换 config——除非你已经用逐文件 MD5 确认过其它文件没被改过。本项目里正是因为做了那次全量对比，才敢断定「只换 config 就够」。

同时保留三档已知可用状态：

| 档位 | 含义 |
|---|---|
| 全功能版（最终） | 26 kext + 蓝牙变量 + 固定 slide |
| 无蓝牙版 | 23 kext，其余功能齐全，用于隔离蓝牙变量 |
| 最小启动集 | 9 个必需 kext，用于判断问题是否在 kext 层 |

---

## 9. 已知问题与后续方向

- [ ] **显卡驱动与系统更新**：macOS 安全更新会重建 AuxKC，更新后需确认 `kmutil ... | grep nullmoth` 仍是 4 行，失效则重跑 1401/install.sh
- [ ] **睡眠唤醒后的 GPU**：若睡醒丢显卡，试 `-nvrmnopm`；参考官方 `docs/SLEEP-WAKE.md`
- [ ] **HDMI 外接显示器**：本机只枚举出 2 个 connector（内屏 + HDMI），多屏 / 热插拔行为未系统验证
- [ ] **摄像头出流**：等 `XHCILagFix` 上游修 iso 重定向（issue #3），或临时关闭它做取舍验证
- [ ] **Type-C 视频输出**：物理切断，实用解法是 DisplayLink 扩展坞
- [ ] **右侧 USB-A 的 USB 2.0 通道**：先在 Windows 下对比，判定是硬件还是 ACPI 缺项
- [ ] **连续多次冷启动验证**：固定 slide 后理论上应每次都进，值得再做一轮压力确认
- [ ] `DevirtualiseMmio = True` 让白名单生效是否比固定 slide 更优雅（风险中等）
- [ ] 清理 `NVRAM/Delete` 里遗留的 `rtcfx_exclude` 参数（`RTCMemoryFixup` 已禁用，参数无效）
- [ ] Windows 与 macOS 交替把 RTC 当本地时/UTC 写，时间会跳 8 小时；可在 Windows 设 `RealTimeIsUniversal=1`

---

## 10. 参考项目与资料（全部仓库链接）

以下是本次攻坚中**实际查证、下载、比对或采用**的项目与资料。所有 GitHub 链接都经过 API 核实存在（2026-11-07）。标注「采用」的是最终 EFI 里真正在用的。

### 10.1 引导与基础组件

| 项目 | 链接 | 用途 |
|---|---|---|
| OpenCorePkg | https://github.com/acidanthera/OpenCorePkg | 引导器本体、官方配置文档 `Docs/Configuration.pdf` |
| Lilu | https://github.com/acidanthera/Lilu | 补丁基座，本机 1.7.3（**采用**） |
| VirtualSMC | https://github.com/acidanthera/VirtualSMC | SMC 仿真 + 电池/光感插件（**采用**） |
| WhateverGreen | https://github.com/acidanthera/WhateverGreen | 显卡相关补丁（**采用**） |
| AppleALC | https://github.com/acidanthera/AppleALC | 声卡，`alcid=13`（**采用**） |
| NVMeFix | https://github.com/acidanthera/NVMeFix | NVMe 电源管理（**采用**） |
| RestrictEvents | https://github.com/acidanthera/RestrictEvents | 屏蔽不兼容的提示/更新（**采用**） |
| BrightnessKeys | https://github.com/acidanthera/BrightnessKeys | 亮度键（**采用**） |
| VoodooInput | https://github.com/acidanthera/VoodooInput | 多点触控统一层（**采用**） |
| VoodooPS2 | https://github.com/acidanthera/VoodooPS2 | 内置键盘（**采用**） |
| BrcmPatchRAM | https://github.com/acidanthera/BrcmPatchRAM | **`BlueToolFixup.kext` 出处**（**采用**） |
| VoodooI2C | https://github.com/VoodooI2C/VoodooI2C | I2C 触控板全套（**采用**；注意仓库已迁到组织下，旧地址 `alexandred/VoodooI2C` 会重定向） |
| USBToolBox/tool | https://github.com/USBToolBox/tool | USB 端口定制工具；本项目未启用，仅作备选 |
| OpenCore Install Guide | https://github.com/dortania/OpenCore-Install-Guide · https://dortania.github.io/OpenCore-Install-Guide/ | 安装与排障总参考 |
| Dortania kernel issues | https://dortania.github.io/OpenCore-Install-Guide/troubleshooting/extended/kernel-issues.html | `slide` / `ProvideCustomSlide` / `DevirtualiseMmio` 的官方说明 |

> `AMFIPass`、`AppleMCEReporterDisabler`、`RealtekRTL8111` 也在最终 EFI 中，但本次记录没有单独核实它们的上游仓库地址（属 AMD 平台常见组件）。请通过 OpenCore 官方 kext 索引或对应项目主页获取，**本文不提供未经核实的链接**。

### 10.2 WiFi（MT7922 / RZ616）

| 项目 | 链接 | 结论 |
|---|---|---|
| laobamac/SimpleMtkWlan | https://github.com/laobamac/SimpleMtkWlan | 上游实验版，不支持 `14c3:0616`、无 release |
| **e5u/SimpleMtkWlan** | https://github.com/e5u/SimpleMtkWlan | ★ **采用**：v1.6.0，README 实测表列有 MT7922 / 460–500 Mbps |
| flxxxxddd/SimpleMtkWlan | https://github.com/flxxxxddd/SimpleMtkWlan | 同源改进分支；最终 kext 的 bundle id 为 `com.flxxxxddd.AirportMtkWlan` |
| e5u/FA608FM-Hackintosh-EFI | https://github.com/e5u/FA608FM-Hackintosh-EFI | 同为 MT7922 的 AMD 笔记本 EFI，配置参考 |

### 10.3 蓝牙（MT7922 USB `0489:e0e4`）

| 项目 / 资料 | 链接 | 结论 |
|---|---|---|
| laobamac/SimpleMtkBtFw | https://github.com/laobamac/SimpleMtkBtFw | v1.0.0 基础版（2026-05），同一代码线 |
| **flxxxxddd/SimpleMtkBtFw** | https://github.com/flxxxxddd/SimpleMtkBtFw | ★ **采用**：v1.1.0（2026-10），分支 `mt7921-bt-fixes`，含固件加载器 + Lilu patcher |
| ├ issue #3（laobamac） | https://github.com/laobamac/SimpleMtkBtFw/issues/3 | **最有价值的一篇**：作者本人写的 Sequoia 上 `THIRD_PARTY_DONGLE` 路径、NVRAM 变量、AirPods 音频修复长文 |
| ├ issue #4（laobamac） | https://github.com/laobamac/SimpleMtkBtFw/issues/4 | MT7922 `04ca:3804` 在 Tahoe 26 上固件已载但 `Address NULL` / pipe stalled 的失败案例（无解答） |
| ├ issue #2（flxxxxddd） | https://github.com/flxxxxddd/SimpleMtkBtFw/issues/2 | MT7922 / Tahoe 的实测反馈 |
| andy-walker-idfa/RZ616-Bluetooth-macOS | https://github.com/andy-walker-idfa/RZ616-Bluetooth-macOS | **备选（未采用）**：BlueToolFixup fork，SRS 路线，目标 `0E8D:0616` |
| └ release v2.7.3-rz616 | https://github.com/andy-walker-idfa/RZ616-Bluetooth-macOS/releases/tag/v2.7.3-rz616 | 二进制内含 `-bluetoothmediatek` / `kMtkChipsetVendorPatched`；硬编码 `0489:e0d8` 与 `0e8d:0616` |
| tonymacx86 原帖 | https://www.tonymacx86.com/threads/wifi-6-e-bluetooth-5-x-mediatek-chipset-on-native-sequoia-tahoe.332855/ | SRS 方案出处（论坛附件需登录，故改从 GitHub 获取等价实现） |
| OpenIntelWireless/IntelBluetoothFirmware | https://github.com/OpenIntelWireless/IntelBluetoothFirmware | 对照项目：只支持 Intel，**不支持 MediaTek**，用来排除错误方向 |

固件 `BT_RAM_CODE_MT7922_1_1_hdr.bin` 来自 Linux 的 `linux-firmware` 仓库 `mediatek/` 目录，Windows 驱动包里也有同名文件。

### 10.4 USB

| 项目 / 资料 | 链接 | 结论 |
|---|---|---|
| **hbyonly1/XHCILagFix** | https://github.com/hbyonly1/XHCILagFix | ★ **采用**：实验性 AppleXHCI 完成事件重定向，`OpenCore/Kernel-Add.plist` 里有官方注入条目可对照 |
| └ issue #3 | https://github.com/hbyonly1/XHCILagFix/issues/3 | iso 传输永不完成 = 摄像头 0 帧的根因，作者给了未合并的修法 |
| AMD-OSX 论坛线索 | 帖名 `amd-rayon-r7-6800h-install-ventura-kernel-panic`（用户 `redhorse1990`） | 最初发现 XHCILagFix 的渠道；**未保存完整 URL，此处不提供猜测地址** |

### 10.5 显卡（NullMoth NVIDIA）★ 本次补充

| 项目 / 资料 | 链接 | 结论 |
|---|---|---|
| **nullmoth/nvidia-macos-driver** | https://github.com/nullmoth/nvidia-macos-driver | ★ **采用 v2.7.2**：Metal driver for NVIDIA GeForce on macOS Sequoia，源码开放（4193 stars） |
| └ Releases / v2.7.2 | https://github.com/nullmoth/nvidia-macos-driver/releases/tag/v2.7.2 | 本次所用版本（2026-11-05），含 `nullmoth-nvidia-2.7.2.tar.gz` 与 SHA256SUMS |
| └ CARD-SUPPORT | https://github.com/nullmoth/nvidia-macos-driver/blob/main/docs/CARD-SUPPORT.md | 支持列表；**RTX 30 笔记本卡标为 `Pending`**，本机属实测可用但非官方确认 |
| └ TROUBLESHOOTING | https://github.com/nullmoth/nvidia-macos-driver/blob/main/docs/TROUBLESHOOTING.md | 官方排障手册 |
| └ HOW-IT-WORKS | https://github.com/nullmoth/nvidia-macos-driver/blob/main/docs/HOW-IT-WORKS.md | NVMTLDriver → SPIR-V → NVK → kexts 的结构说明 |
| └ SLEEP-WAKE | https://github.com/nullmoth/nvidia-macos-driver/blob/main/docs/SLEEP-WAKE.md | 睡眠唤醒与 `-nvrmnopm` |
| └ CHANGELOG | https://github.com/nullmoth/nvidia-macos-driver/blob/main/CHANGELOG.md | 版本差异（2.7.2 之前改动很多，排障时先确认版本） |
| **nullmoth/1401** | https://github.com/nullmoth/1401 | 官方安装器（Mac app / 从 Windows 装 macOS），会自动改 OC config 并加「Remove NVIDIA driver」启动项 |
| laobamac/nvidia-macos-driver | https://github.com/laobamac/nvidia-macos-driver | 早期 fork，仅作对照（2 stars，未采用） |
| pubglite55/ROG-GU603ZM-Hackintosh | https://github.com/pubglite55/ROG-GU603ZM-Hackintosh | 同为 **RTX 3060 Laptop + Sequoia 15 + NullMoth** 的 EFI 参考 |
| CGR-MIX/AN515-macOS-Tahoe | https://github.com/CGR-MIX/AN515-macOS-Tahoe | 同系列 AN515 机型 + NullMoth 的 Tahoe 26 案例 |
| GeekAudio/nm185-diag | https://github.com/GeekAudio/nm185-diag | BAR1 aperture 诊断收集器（排同类问题时参考） |
| bananan11ss/nullmoth-updater | https://github.com/bananan11ss/nullmoth-updater | 第三方安装/更新器，声称修了 Metal 插件；**未采用**，仅列出 |
| acidanthera/WhateverGreen | https://github.com/acidanthera/WhateverGreen | 已在 10.1；与 NullMoth 共存时**必须** `gfxrst=4` |

> 本组链接与 §3.0 的全部配置要求，均在 2026-11-07 用官方 README 与本机的
> `kextstat` / `system_profiler` / `config.plist` 逐项比对过，不是推测。

### 10.6 证据强度说明

| 强度 | 内容 |
|---|---|
| **实测（本机）** | NullMoth 2.7.2 的 4 个 kext + Metal 3 / 165Hz 内屏；蓝牙三件套 + NVRAM 变量 + 固定 slide；XHCILagFix 的 `Active=Yes` 与 busy 时间；UPCN/UPCC 补丁；`alcid=13`；VoodooI2C 五件套 + `TPAD._CRS` OSYS 旁路补丁（`Interrupt Mode = GPIO`）；macOS 26 上 `MacPro7,1` 全功能复现 |
| **官方/上游文档** | NullMoth 官方 README 的 boot-args / SIP / Kernel-Block / `gfxrst=4` 要求；`slide=value` 使 `ProvideCustomSlide` 失效；`MmioWhitelist` 需 `DevirtualiseMmio=True`；AirportMtkWlan 的 Min/MaxKernel 要求 |
| **他人实测案例** | MT7921K/RZ608 在 macOS 15 + iMacPro1,1 完整工作；MT7922 `0489:E0D8` 在 Tahoe 26 能连 BLE；同款 6800H 的机型 AirPods 可连 |
| **推测 / 未验证** | 摄像头根因指向 XHCILagFix iso 缺陷（未做关 kext 对比）；Type-C 切 Optimus 方案；`DevirtualiseMmio=True` 的效果 |

---

## 11. 桌面起不来但 WindowServer 在跑：AuxKC 路径陷阱

这是 macOS 26 (Tahoe) 升级过程中最难的一段，症状具有极强的迷惑性，值得单独成章。

### 11.1 症状组合（记住这个组合，它会重复出现）

- 屏幕永远冻在 `-v` 的最后一行，常见停在 `LUPCIConfigurator::configure() PCI configuration PCI0`；
- **SSH 能连进来**，内核完全启动，网络、DNS、NTP、APNs 全部正常；
- `launchctl print system/com.apple.WindowServer` 明确显示 `state = running`；
- 但 `log show` 里反复出现：

```
Failed to look up the port for "com.apple.windowserver.active" (1102)
```

**「内核好 + 用户态好 + WindowServer 在跑 + 屏幕不动」这个组合，指向的几乎一定是 GPU 驱动没加载进 AuxKC，而不是显示配置或 ACPI 的问题。** 别去改 framebuffer、别去换 SMBIOS、更别去关 kext。

### 11.2 根因：`kmutil create` 的 `-p` 参数会剥掉 `-R` 前缀

真正的错误藏在 `kernelmanagerd` 日志里：

```
Installation check: missing bundle for com.nullmoth.NVRMAGDC at /Users/acer/ - Data/Library/Extensions/NVRMAGDC.kext
```

注意那个 **`/Users/acer/ - Data/`** —— 这是畸形的。原因：

- 目标系统卷名是 `TAHOE - Data`（APFS Data 卷，名字本身就带空格和连字符）；
- 跨版本重建时用了 `-R /Volumes/TAHOE -p /Volumes/TAHOE - Data/Library/Extensions/xxx.kext`；
- `kmutil` 会先用 `-R` 的前缀去剥 `-p` 的路径，剥完剩下 ` - Data/Library/Extensions/xxx.kext`（相对路径）；
- 再把这个相对路径**拼上当前工作目录** —— 当时 cwd 是 `/Users/acer`，于是得到 `/Users/acer/ - Data/Library/Extensions/xxx.kext`。

这个路径当然不存在，四个驱动全部被拒装。于是：内核起得来、用户态起得来、WindowServer 也起得来，但**拿不到 framebuffer**，屏幕就永远停在那里。

### 11.3 修复：在目标系统本机跑，cwd 必须是 `/`，`-p` 用绝对路径

```bash
cd /                                    # ← 关键，cwd 必须是 /
PB=$(ls -d /System/Volumes/Preboot/*/boot | head -1)

sudo kmutil create -n aux -A /tmp/aux-good.kc -R / \
  -p /Library/Extensions/NVAccel.kext \
  -p /Library/Extensions/NVRM.kext \
  -p /Library/Extensions/NVRMAGDC.kext \
  -p /Library/Extensions/NVRMFB.kext \
  -k /System/Library/Kernels/kernel \
  -B "$PB/System/Library/KernelCollections/BootKernelExtensions.kc" \
  -S /System/Library/KernelCollections/SystemKernelExtensions.kc \
  --build "$(sw_vers -buildVersion)" \
  --allow-missing-kdk --no-authentication --no-authorization --force

# 产物约 19–20 MB，装回去
sudo cp /tmp/aux-good.kc /Library/KernelCollections/AuxiliaryKernelExtensions.kc
sudo chown root:wheel /Library/KernelCollections/AuxiliaryKernelExtensions.kc
sudo chmod 744 /Library/KernelCollections/AuxiliaryKernelExtensions.kc
sudo reboot
```

**怎么确认修好了**：

```bash
kmutil showloaded --list-only | grep -c nullmoth          # 4
ioreg -l -w0 | grep -A2 IOFramebuffer | head -20          # 应出现 NVRMNVDAFramebuffer
ioreg -l -w0 | grep -c NVMTLDevice                        # Metal 插件类名
```

### 11.4 三条走不通的路（别再试）

| 做法 | 结果 |
|---|---|
| `kmutil install --volume-root / --allow-missing-kdk` | 打印 `rebuild done`，但文件 mtime/大小不变，因为它认为已是最新 |
| 加 `--update-all` | 会连 BootKC 一起重建，宿主机版本与目标卷不匹配时报 `com.apple.kpi.iokit not found` |
| 用 `-L` 指定 kext 列表 | 会漏掉 `IOGraphicsFamily` 等隐式依赖，驱动仍拿不到 framebuffer |

**顺带的结论**：系统更新（安全更新 / 小版本升级）都会重建 AuxKC，所以每次更新后都要跑一遍 `kmutil showloaded --list-only | grep nullmoth`，不够 4 行就按上面的命令重建。

---

## 12. 没有桌面时怎么进去：离线建账户 + 开 SSH

装完系统之后，如果因为显卡驱动没起来而进不了桌面，会陷入死锁：**没有桌面 → 打不开「系统设置 → 通用 → 共享 → 远程登录」→ 没法 SSH → 没法修驱动**。

这个死锁可以在**另一台 Mac 上离线破解**：直接往目标卷里写账户和 launchd 覆盖配置。

### 12.1 先验证「用户覆盖能压过 plist 默认值」

不要凭空猜。拿一个**已经能用 SSH 的系统**做对照：

```bash
# 系统自带的 sshd 服务定义写着 Disabled=true
plutil -p /System/Library/LaunchDaemons/ssh.plist | grep -i disabled
# → "Disabled" => true

# 但用户覆盖里是 0（false）
sudo plutil -p /var/db/com.apple.xpc.launchd/disabled.plist | grep sshd
# → "com.openssh.sshd" => false

# 而 SSH 实际是能用的
```

结论：**`/var/db/com.apple.xpc.launchd/disabled.plist` 里的用户覆盖优先于服务自身的 `Disabled` 键。** 往目标卷写同样的 `false`，ssh 就会起来。

### 12.2 落地步骤（全部只写目标卷，不动源系统）

记目标卷的 Data 分区挂载点为 `$T`（例如 `/Volumes/TAHOE - Data`）。

**① 建用户**

```bash
$T/private/var/db/dslocal/nodes/Default/users/acer.plist
```

字段：`uid=501`、`gid=20`、`home=/Users/acer`、`shell=/bin/zsh`、`GeneratedUID` 用一个新的 UUID。

密码这一步最省事的是**复用另一台机器上同一账户的 `ShadowHashData`**（933 字节，Base64）。但如果源账户带 Secure Token，必须把 `authentication_authority` 数组里的这几项剔掉，否则目标系统认证会异常：

```
;SecureToken;
;Kerberosv5;;acer@LKDC:...
HeimdalSRPKey
KerberosKeys
```

**② 加进 admin 组**

```
$T/private/var/db/dslocal/nodes/Default/groups/admin.plist
  users          = ['root', 'acer']
  GroupMembership= ['root', 'acer']
  GroupMembers   += <新用户 GeneratedUID>
```

**③ 放公钥**

```bash
$T/Users/acer/.ssh/authorized_keys     # 权限 600，属主 acer:staff
```

**④ 跳过设置助手**

```bash
touch $T/private/var/db/.AppleSetupDone
touch $T/Users/acer/.skipbuddy
```

再写 15 个 `DidSee*` 键（`com.apple.SetupAssistant` 域）和 9 个 `Skip*` 的 managed plist。

**⑤ 开 sshd**

```bash
$T/private/var/db/com.apple.xpc.launchd/disabled.plist
  com.openssh.sshd = false
```

**⑥ 预生成 host key（容易漏，漏了会握手失败）**

全新系统在第一次开 SSH 前没有任何 host key。离线补上：

```bash
ssh-keygen -q -t rsa     -b 3072 -N '' -f $T/private/etc/ssh/ssh_host_rsa_key
ssh-keygen -q -t ecdsa   -b 256  -N '' -f $T/private/etc/ssh/ssh_host_ecdsa_key
ssh-keygen -q -t ed25519         -N '' -f $T/private/etc/ssh/ssh_host_ed25519_key
```

**⑦ 验证（不启动目标系统就能验）**

```bash
dscl -f $T/private/var/db/dslocal/nodes/Default localhost -read /Local/Default/Users/acer
# 能读出完整记录 = 目录服务能识别
```

### 12.3 三个坑

- **不要**手工创建 `com.apple.access_ssh` 组。这个组**不存在**时表示「所有用户都可以 SSH」，建了反而要维护成员列表，写错就谁都进不去。
- `ShadowHashData` 不能逐字段重编，必须整块照搬（933 字节）。自己拼 `SALTED-SHA512-PBKDF2` 基本进不去。
- 那些 `DidSee*` / `Skip*` 少写几个只会让设置助手多弹几屏，**不会**导致 SSH 失败；但 `.AppleSetupDone` 漏了会卡在「创建账户」界面出不来。所以先保证 `.AppleSetupDone`，其余可选。

---

## 13. OpenCore 选单顺序没有排序逻辑

**OpenCore 不会对启动项排序。** 显示顺序 = 扫描与注册的顺序：

1. 固件 `BootOrder` 里已有的条目；
2. APFS 扫描发现的 macOS 卷；
3. `Misc/Entries` 里手工加的自定义条目 —— **永远排在最后**。

让 macOS 排第一的可行组合：

```
Misc / Boot / HideAuxiliary   = true      # 隐藏 Recovery、Time Machine 等辅助项
Misc / Security / ScanPolicy  = 19464963  # 收紧：去掉 ESP / NTFS 扫描位
```

`ScanPolicy = 19464963` 的副作用很实在：自动探测出来的 Windows 条目会消失，于是必须自己往 `Misc/Entries` 加一条 Windows，而且它一定排在 macOS 后面 —— 这刚好就是想要的顺序。

> ⚠️ **做安装 U 盘时记得临时把 `ScanPolicy` 改回 `0`**，否则 U 盘根本不出现在选单里，找半天以为 U 盘没做好。

**另一个独立的坑：F12 里两个「Ubuntu」死条目。** 两个 ESP 里都残留着 `EFI/ubuntu`（Kubuntu 的 shim + grub），但磁盘上 Linux 分区早已删除，`grub.cfg` 里 `search.fs_uuid 2514fd6e-...` 指向不存在的卷。选中只会掉进 `grub rescue>`。它跟 OpenCore 无关（`ScanPolicy` 关掉 ESP 扫描后 OC 选单里根本不显示），要清只能进 Windows：

```cmd
bcdedit /enum firmware
bcdedit /delete {对应标识符}
```

只删 ESP 里的 `EFI/ubuntu` 目录**不会**清掉 NVRAM 里的引导项，F12 里照样在。

---

## 附录 A：ACPI 补丁字节

| 名称 | Find | Replace | 作用 |
|---|---|---|---|
| `_OSI → XOSI` | 见 SSDT-XOSI 配套说明 | — | 配合 `SSDT-XOSI.aml`，让 ACPI 把系统识别成指定 Windows 版本 |
| `UPCN` | `08 55 50 43 4E 12 07 04 00 0A FF 00 00` | `08 55 50 43 4E 12 07 04 <b>01</b> 0A FF 00 00` | XHC0/XHC1 端口标记可连接 |
| `UPCC` | `08 55 50 43 43 12 07 04 00 0A FF 00 00` | `08 55 50 43 43 12 07 04 <b>01</b> 0A FF 00 00` | XHC2.PRT1 / 摄像头 UCAM |
| `TPAD / OSYS` | `95 4f 53 59 53 0b dc 07` | `95 4f 53 59 53 0b 00 00` | **必需**。绕开 `TPAD._CRS` 里的 OSYS 门，让 `_CRS` 返回 `SBFB`+`SBFG`（I2C 总线地址 + GPIO 中断）。不加则 `ELAN050A` 建不出 nub，触控板完全无反应。见 §3.3 |

AML opcode 速查：`08` NameOp · `12` PackageOp · `0A` BytePrefix · `0B` WordPrefix · `95` LNotEqual（比较类操作码）。做等长替换时，**Find 必须在 DSDT 里唯一**，否则会误伤其它表。改完务必用 `ioreg` 验证效果，不要只依赖反编译判断。

## 附录 B：日志规范化对比脚本

```python
import re, difflib

def norm(line):
    s = line.rstrip('\n')
    s = re.sub(r'^\d{2}:\d{3} \d{2}:\d{3} ', '', s)   # 行首相对时间戳
    s = re.sub(r'2026-11-07T[\d:]+', 'TS', s)        # ISO 时间（换成你的日期）
    s = re.sub(r'2026\.11\.07 [\d:]+', 'TS', s)
    s = re.sub(r'\b[0-9A-F]{8}\b', 'ADDR', s)        # 32 位地址
    s = re.sub(r'0x[0-9a-fA-F]+', 'HEX', s)
    s = re.sub(r'\b\d{4,}\b', 'NUM', s)              # 长数字（含频率抖动）
    return s

def load(path):
    # OC 日志尾部有大段 \x00 填充，先清掉
    raw = open(path, 'rb').read().replace(b'\x00', b'').decode('utf8', 'replace')
    return [norm(l) for l in raw.splitlines() if l.strip()]

a, b = load('fail.txt'), load('ok.txt')
for line in difflib.unified_diff(a, b, n=1, lineterm=''):
    print(line[:160])

# 单独提取 slide
raw = open('ok.txt', 'rb').read().replace(b'\x00', b'').decode('utf8', 'replace')
print(re.search(r'slide=(\d+)', raw).group(1))
```

**判读要点**：若 `Prelink size` / `kext offset` / `reserved` 三个值一致，说明内存布局与注入 kext 完全相同，此时剩下的唯一变量通常就是 `slide=`。

---

## 许可与免责

- 本文为个人实机经验记录，欢迎转载与二次整理，请注明出处并保留「已知问题」与「证据强度」标注。
- 文中不含任何可直接使用的 EFI、序列号、平台信息或个人凭据。请自行生成合法的 SMBIOS 数据。
- 黑苹果涉及修改引导与内核扩展，**可能导致数据丢失或系统无法启动**。操作前请完整备份，风险自负。
- 部分项目（尤其 `XHCILagFix`、`RZ616-Bluetooth-macOS` fork）为实验性或缺乏第三方验证，请自行评估。
