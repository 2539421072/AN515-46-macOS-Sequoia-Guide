# 暗影骑士·龙 2022 / Acer Nitro 5 AN515-46 安装 macOS 15 Sequoia 与驱动修复全记录

> 一台 **AMD Ryzen 7 6800H + RTX 3060 Laptop** 笔记本的黑苹果完整攻坚记录。
> 最终：**WiFi、蓝牙（含耳机音频）、触控板与手势、USB、声卡、有线网、键盘、电池、亮度键全部可用**。
> 环境：macOS Sequoia 15.8.2（Darwin 24.6.0）/ OpenCore / SMBIOS `iMacPro1,1`。

这不是一份「照抄 EFI 就能用」的教程，而是一份**排障经验总结**：每一项都写清楚「症状 → 根因 → 具体做法 → 验证命令」，并如实记录走过的弯路。很多坑不是硬件问题，而是**判断方法**的问题，这也是本文最有价值的部分。

> ⚠️ 本文记录的所有结论均来自一台实机的反复验证。不同 BIOS 版本、不同无线卡子型号、不同 macOS 小版本都可能让结果不同。**请务必先整包备份自己的 EFI，再逐项尝试。**

## 目录

- [0. 一页结论表](#0-一页结论表)
- [1. 机器与环境](#1-机器与环境)
- [2. 安装与基础 OpenCore 配置](#2-安装与基础-opencore-配置)
- [3. 驱动逐项修复](#3-驱动逐项修复)
- [4. 启动「玄学」的真正根因：KASLR slide 抽签](#4-启动玄学的真正根因kaslr-slide-抽签)
- [5. 排障方法论与铁律](#5-排障方法论与铁律)
- [6. 最终配置快照](#6-最终配置快照)
- [7. 一键自检清单](#7-一键自检清单)
- [8. 备份与回滚策略](#8-备份与回滚策略)
- [9. 已知问题与后续方向](#9-已知问题与后续方向)
- [10. 参考项目与资料（全部仓库链接）](#10-参考项目与资料全部仓库链接)
- [附录 A：ACPI 补丁字节](#附录-aacpi-补丁字节)
- [附录 B：日志规范化对比脚本](#附录-b日志规范化对比脚本)

更深入的启动失败排查流程见 [TROUBLESHOOTING.md](TROUBLESHOOTING.md)。

---

## 0. 一页结论表

| 功能 | 最终状态 | 关键手段 | 验证判据 |
|---|---|---|---|
| WiFi（MT7922 / RZ616） | ✅ | `AirportMtkWlan.kext` v1.6.0（fork），`MinKernel=24.2.0` / `MaxKernel=24.99.99` | `system_profiler SPAirPortDataType` 显示 MT7922；实测 400–600 Mbps |
| 蓝牙（同一张卡，USB `0489:e0e4`） | ✅ | `BlueToolFixup` + `SimpleMtkBtFw` + `SimpleMtkBtPatcher` + 2 个 NVRAM 变量 | `Chipset: THIRD_PARTY_DONGLE` / `State: On` / Address 非空 |
| 触控板 + 多点手势 | ✅ | VoodooI2C 五件套 + `VoodooI2CHID`，**不需要 ACPI 补丁** | `VoodooI2CHID` / `VoodooInput` 加载，`AppleMultitouchDevice` 注册 |
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
| 独显 | NVIDIA RTX 3060 Laptop；本环境内屏由第三方 NullMoth 2.7.2 方案驱动 |
| 内存 / 屏幕 | 32 GB / 1920×1080 @165 Hz |
| 无线卡 | MediaTek MT7922 / AMD RZ616，M.2 2230，WiFi（PCIe `14c3:0616`）+ 蓝牙（USB `0489:e0e4`）二合一 |
| 声卡 | Realtek ALC287 |
| 系统 | macOS Sequoia 15.8.2 |
| SMBIOS | `iMacPro1,1` |
| 引导 | OpenCore（ESP 上 `EFI/OC/`） |

> 关于显卡：本文重点是**驱动修复**，显卡/显示输出的实现细节不在本次记录范围内，请勿据此推断通用做法。请单独参考对应驱动项目的文档。

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

其中 `alcid=13` 对应本机 ALC287 的 layout。最终还要在末尾追加固定的 `slide=N`（见 §4），**不要把它写死成 142**。

---

## 3. 驱动逐项修复

每一项都按「症状 → 根因 → 做法 → 验证」组织。

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

### 3.3 触控板：不需要 ACPI 补丁

**症状**：曾出现「触控板完全无反应」，日志里有：

```
VoodooI2CDeviceNub::TPAD Could not find an I2C Serial Bus declaration
Could not initialise nub for ELAN050A
```

**走过的一个弯路**：从 DSDT 里解出 `TPAD._CRS` 有 `If (OSYS < 0x07DC) Return (SBFI)` 这样的分支（`SBFI` 只有中断、没有 I2C 地址），于是写了一个 8 字节等长补丁把条件改成恒假：

```
Find    : 95 4f 53 59 53 0b dc 07    (If OSYS < 0x07DC)
Replace : 95 4f 53 59 53 0b 00 00    (If OSYS < 0 → 恒假)
```

加补丁那次触控板确实好了，但后来用**黄金版配置（不含此补丁）**实测：`VoodooI2CHID` + `VoodooInput` 都加载、`AppleMultitouchDevice` 正常注册、30 列传感器齐全。

> **所以这个补丁不是必需的，相关不等于因果。** 当时触控板不工作，真正原因是排障做减法时把 VoodooI2C 五件套关掉了，后来重启顺带完成了 kext 缓存重建。**不建议往 config 里加这条补丁。**

**正确做法**：保持 VoodooI2C 五件套（`VoodooI2CServices` / `VoodooGPIO` / `VoodooI2C` / `VoodooInput` 插件 / `VoodooI2CHID`）全部启用，且 `Lilu` 在前。

**验证**：

```bash
kextstat | grep -E "VoodooI2CHID|VoodooInput"
ioreg -rc AppleMultitouchDevice | grep -c AppleMultitouchDevice
ioreg -l -w0 | grep -c VoodooI2CDeviceNub
```

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

**启用 26 个 kext**（顺序即 `Kernel/Add` 顺序）：

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

**NVRAM/Add**：蓝牙两个变量 + boot-args（末尾追加 ` slide=N`）。
**NVRAM/Delete**：同名两个蓝牙变量。

---

## 7. 一键自检清单

进系统后按顺序跑一遍，任何一项不符就回到对应章节：

```bash
# 蓝牙
system_profiler SPBluetoothDataType | grep -E 'Chipset:|State:|Address:|Transport:'
kextstat | grep -iE 'BlueTool|SimpleMtkBt'

# WiFi
networksetup -listallhardwareports | grep -A2 -i Wi-Fi
system_profiler SPAirPortDataType | grep -E 'Card Type|Firmware Version|Status:'

# 触控板
kextstat | grep -cE 'VoodooI2CHID|VoodooInput'
ioreg -rc AppleMultitouchDevice | grep -c AppleMultitouchDevice

# USB
ioreg -r -c XHCILagFix -l -w0 | grep -E '"Active"|"Failure"|"Mode"'
ioreg -p IOService -w0 -r -c IOUSBHostDevice | grep busy

# 启动参数（确认 slide 已固定）
nvram boot-args
```

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

### 10.5 证据强度说明

| 强度 | 内容 |
|---|---|
| **实测（本机）** | 蓝牙三件套 + NVRAM 变量 + 固定 slide 的组合；XHCILagFix 的 `Active=Yes` 与 busy 时间；UPCN/UPCC 补丁；`alcid=13`；VoodooI2C 无补丁可用 |
| **官方/上游文档** | `slide=value` 使 `ProvideCustomSlide` 失效；`MmioWhitelist` 需 `DevirtualiseMmio=True`；AirportMtkWlan 的 Min/MaxKernel 要求 |
| **他人实测案例** | MT7921K/RZ608 在 macOS 15 + iMacPro1,1 完整工作；MT7922 `0489:E0D8` 在 Tahoe 26 能连 BLE；同款 6800H 的机型 AirPods 可连 |
| **推测 / 未验证** | 摄像头根因指向 XHCILagFix iso 缺陷（未做关 kext 对比）；Type-C 切 Optimus 方案；`DevirtualiseMmio=True` 的效果 |

---

## 附录 A：ACPI 补丁字节

| 名称 | Find | Replace | 作用 |
|---|---|---|---|
| `_OSI → XOSI` | 见 SSDT-XOSI 配套说明 | — | 配合 `SSDT-XOSI.aml`，让 ACPI 把系统识别成指定 Windows 版本 |
| `UPCN` | `08 55 50 43 4E 12 07 04 00 0A FF 00 00` | `08 55 50 43 4E 12 07 04 <b>01</b> 0A FF 00 00` | XHC0/XHC1 端口标记可连接 |
| `UPCC` | `08 55 50 43 43 12 07 04 00 0A FF 00 00` | `08 55 50 43 43 12 07 04 <b>01</b> 0A FF 00 00` | XHC2.PRT1 / 摄像头 UCAM |
| ~~TPAD / OSYS~~ | `95 4f 53 59 53 0b dc 07` | `95 4f 53 59 53 0b 00 00` | ⚠️ **已证实非必需，不要加**。见 §3.3 |

AML opcode 速查：`08` NameOp · `12` PackageOp · `0A` BytePrefix · `95` If 里的 LNotEqual/比较上下文 · `0b` WordPrefix。做等长替换时，**Find 必须在 DSDT 里唯一**，否则会误伤其它表。

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
