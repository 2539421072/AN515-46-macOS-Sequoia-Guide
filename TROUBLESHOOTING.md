# 排障手册：从「玄学启动失败」到定位 KASLR slide 抽签

配套主教程：[README.md](README.md)。本文专门讲**系统起不来时怎么查**，包含一整套可复用的流程和这次踩过的每一个误判。

---

## 0. 先记住五个结论

1. **`EXITBS:START` 不是卡死点**。它是 OpenCore 能写的最后一行，成功和失败的日志都以它结尾。看到它不等于卡在这里，失败发生在 XNU 接管之后。
2. **同一份 config 有时能进有时不能，先怀疑运行时随机量**（slide / 内存布局），不要急着回滚配置。
3. **设备「突然坏了」时，先查是不是被自己关掉了**。本项目两次大弯路（USB、触控板）都源于排障做减法时关掉 kext 忘记加回。
4. **「有画面 / 没画面」是一条独立于启动问题的排查线**。NullMoth 显卡 kext 装在系统卷 `/Library/Extensions`、由 AuxKC 加载，**不经过 `EFI/OC/Kexts`**，所以「改 config 做减法」根本影响不到它。黑屏或无 Metal 时按 §11 查，别去动 boot-args 里的 slide。
5. **「屏幕冻在 -v、但 SSH 能进、WindowServer 还 running」是一个专属信号**：不是显示配置问题，是 AuxKC 里驱动路径畸形（跨版本重建时 `-p` 被剥掉了 `-R` 前缀）。见 §11.4。**这时候关任何 kext 都没用，只会把水搅浑。**

---

## 1. 决策树：系统起不来

```
系统起不来 / 没画面
│
├─ 有开机画面但进桌面后黑屏、分辨率固定、无 Metal？
│   └─ 这是显卡驱动问题，不是启动问题 → 直接跳到 §11
│
├─ 能否进 Windows / Recovery / macOS Installer？
│   └─ 能 → 挂载 ESP，先看有没有「新日志」
│
├─ ESP 上没有新日志？
│   ├─ 先数 opencore-*.txt 数量：>20 个就清旧日志（OC 会写不进新文件）
│   ├─ 确认改的是「实际启动的那块 ESP」（双盘双 ESP 极易挂错）
│   └─ 用 nvram boot-path 反查 OC 从哪个分区加载
│
├─ 有新日志 → 提取 slide 值，与上一次成功的值对比
│   └─ 值不同 → 高度怀疑 slide 抽签，跳到 §3
│
├─ 日志显示 Prelinked status - Success，且所有 kext 都注入成功？
│   └─ 是 → OC 阶段没问题，失败在 XNU 之后，别再改 config 结构
│
└─ 还不行 → 做减法（§5），但必须先整包备份 + 列关闭清单
```

---

## 2. 拿到日志的正确姿势

### 2.1 找到并挂载 ESP

```bash
# 盘号每次都可能变，必须现查
diskutil list | grep -i EFI

# macOS 下 diskutil mount 常误报 "volume damaged"；fsck 往往无错误
sudo fsck_msdos -n /dev/rdiskXsY
sudo mkdir -p /Volumes/ESPX
sudo mount -t msdos /dev/diskXsY /Volumes/ESPX

# 确认这就是 OC 实际加载的那块
nvram 4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102:boot-path
```

用 `boot-path` 里的 `HD(n,...)` 对应回 `diskutil list` 的分区号，别凭记忆认盘。

### 2.2 找「最新」日志的坑

- **文件名是 UTC**：北京时间 16:05 的日志叫 `opencore-2026-11-07-080531.txt`。
- **FAT32 时间戳不可靠**：Windows 写入时会按本地时间记 mtime，`ls -t` 可能把新日志排到很后面。

```bash
# 按文件名排序才靠谱
ls -1 /Volumes/ESPX/opencore-*.txt | sort | tail -5
```

### 2.3 日志写不进来

曾出现 ESP 根目录堆了 **113 个** `opencore-*.txt` 之后，连续十几次启动零日志，一度误判成「OpenCore 根本没运行」。清理旧日志后立刻恢复写入。

```bash
# 只保留最近 5 个（先另存归档）
ls -1t /path/to/archive/opencore-*.txt | tail -n +6 | xargs rm -f
```

长期方案：把 `Misc/Debug/Target` 里的文件输出位（`0x40`）关掉，或定期清理。`Target=67` 是包含文件输出的。

---

## 3. 定位 slide 抽签（本次的决定性突破）

### 3.1 提取每次启动的 slide

```bash
for f in /Volumes/ESPX/opencore-*.txt; do
  printf "%-46s " "$(basename "$f")"
  tr -d '\000' < "$f" | grep -o 'slide=[0-9]*' | head -1
done
```

日志里的关键片段：

```
OCABC: ALRBL 1 RTDFRG 1 DEVMMIO 0 ... CSLIDE 1 MSLIDE 0 ... VMAP 1
OCABC: MAT support is 1
OCABC: Only 176/256 slide values are usable!
OCABC: Valid slides - 80-255
AAPL: #[EB|MBA:OUT] <"... alcid=13 slide=142">
```

`Only 176/256` 说明 80 个 slide 值因 MMIO 冲突被判为不可用。常见原因是 `DevirtualiseMmio=False` 让 `MmioWhitelist` 全部失效——**白名单只在这个 quirk 开启时才生效**。

### 3.2 规范化 diff

OpenCore 日志每行带相对时间戳，还混杂地址与 CPU 频率抖动。直接 `diff` 会得到近 2000 行噪声，规范化后只剩几十行。完整 Python 脚本见 [README 附录 B](README.md#附录-b日志规范化对比脚本)。

本项目的实测结果：

| 时间 | slide | 结果 |
|---|---|---|
| 11:06 | `122` | ✅ 成功 |
| 14:42:33 | `185` | ❌ 挂 |
| 14:45:06 | `98` | ❌ 挂 |
| 14:45:40 | `142` | ✅ 成功 |

同一份 config、同样 `Prelink size 74993664` / `kext offset 66134016` / `reserved 8859648`、同样的 kext 列表与 ACPI 表顺序，**实质差异只剩 `slide=`**。

> 判读要点：三个 Prelink 数值一致 ⇒ 内存布局和注入 kext 完全相同；此时剩下的唯一变量就是运行时追加的 slide。

### 3.3 修复

OpenCore 官方配置文档：

> Provide custom KASLR slide for low memory maps. **This option has no effect when `slide=value` boot argument is present**, which needs to be removed for this quirk to work.

所以把 ` slide=N` 写进 `NVRAM/Add/<boot-args GUID>/boot-args` 即可关掉随机抽签。

**N 必须取你自己机器上一次成功启动日志里的值**，不要照抄别人的。写完后重启，从日志 `MBA:OUT` 确认值恒定，再连续做多次冷启动验证。

次选方案：`DevirtualiseMmio = True` 让白名单生效，减少被排除的 slide 值；但会改变内存映射，风险高于固定 slide。

---

## 4. 常见误判表

| 现象 | 容易得出的错误结论 | 正确解释 |
|---|---|---|
| 日志停在 `EXITBS:START` | 「卡在 ExitBootServices」 | OC 日志的正常结尾，成功也一样 |
| 加某个 kext 后启动失败，去掉就好 | 「这个 kext 有毒」 | 可能只是这次抽到了坏 slide；固定 slide 再测 |
| 改回原 config 还是起不来 | 「config 被写坏了 / 文件缺了」 | 先做逐文件 MD5 对比；若一致，就是运行时变量 |
| reset NVRAM 后偶尔能进 | 「NVRAM 残留是根因」 | 只是又抽了一次签，恰好抽到可用值 |
| 触控板/USB 突然不可用 | 「硬件或 ACPI 坏了」 | 先查 kext 是不是被自己关掉了 |
| `diskutil mount` 报 volume damaged | 「ESP 文件系统损坏」 | `fsck_msdos -n` 往往无错误，用底层 `mount -t msdos` 即可 |
| `SetupVirtualMap=False` 后能进一次 | 「找到 AMD 固件修复了」 | 该值在本机 True/False 都成功过也都挂过，非决定变量 |
| Recovery 也卡死 | 「系统卷坏了」 | Recovery 同样会注入第三方 kext，只能排除系统卷，不能排除 kext |
| 装完系统更新后画面没了 / 无 Metal | 「OpenCore config 被改坏了」 | macOS 更新重建了 AuxKC，`/Library/Extensions` 里的显卡 kext 不再加载；重装驱动即可，与 EFI 无关 |
| 把 NVRM/NVAccel 等 kext 复制进 `EFI/OC/Kexts` 想「修黑屏」 | 「多注入一份更保险」 | 官方明确禁止：这些 kext 只能从系统卷 AuxKC 加载，双份会冲突 |
| 首次装驱动时卡在 `PCI configuration end` 约 1 分钟 | 「又死机了，断电重来」 | GPU 上电的正常等待，官方说明最长约 1 分钟 |

---

## 5. 做减法（最小启动集）与它的陷阱

当怀疑问题在 kext 层时，关掉大部分 kext，只留最小必需集：

```
Lilu · AMFIPass · AppleALC · AppleMCEReporterDisabler
VirtualSMC · VoodooPS2 四件套 · WhateverGreen
```

本项目用这 9 个成功进了桌面，从而证明问题确实在 kext 层。之后逐项加回（每次一层备份 + 冷/热启动验证）。

> ⚠️ 注意 `WhateverGreen` 被保留在最小集里：它虽不是显卡驱动本体，但与 NullMoth 共存时
> `boot-args` 里的 `gfxrst=4` 依赖它存在。**真正驱动这张 RTX 3060 的四个 `com.nullmoth.*` kext
> 不在这张清单里**，它们在系统卷，做减法关不到，也不该去关。

**但这个方法有个致命陷阱**：

> 关掉之后**必须列清单逐项加回**。本项目加回了网卡、VoodooI2C 五件套，却漏了 `XHCILagFix`，于是用户报「USB 还是不能用」时，被当成一个全新的硬件问题从头排查，越改越乱，最后还误判成启动失败元凶。

建议做法：

```bash
# 关之前先把启用列表存下来
plutil -extract Kernel.Add json -o /tmp/kext-on-before.json EFI/OC/config.plist
# 或
/usr/libexec/PlistBuddy -c "Print :Kernel:Add" EFI/OC/config.plist > /tmp/kext-before.txt
```

每次加回后对照清单打勾，全部加完再判断。

---

## 6. 整包对比：确认「只换 config 够不够」

只备份 `config.plist` 是不充分的——`Kexts/`、`ACPI/`、`Drivers/` 里任何文件被改动或缺失，config 换回去也没用。

```bash
# 两边各打包整个 EFI
tar -czf /tmp/efi-now.tgz -C <ESP挂载点> EFI
tar -czf /tmp/efi-gold.tgz -C <备份解包目录> EFI

# 逐文件 MD5
python3 - <<'EOF'
import os, hashlib
def h(root):
    m = {}
    for dp, _, fn in os.walk(root):
        for f in fn:
            p = os.path.join(dp, f)
            m[os.path.relpath(p, root)] = hashlib.md5(open(p,'rb').read()).hexdigest()
    return m
a, b = h('/tmp/cmp/now/EFI'), h('/tmp/cmp/gold/EFI')
print('now', len(a), 'gold', len(b))
print('only now   :', sorted(set(a)-set(b))[:20])
print('only gold  :', sorted(set(b)-set(a))[:20])   # 这里若是空，说明没缺文件
print('differs    :', sorted(k for k in set(a)&set(b) if a[k]!=b[k])[:20])
EOF
```

本项目的对比结果：当前 ESP 188 文件，黄金包 179 文件；多出的 9 个全是自己加的 `config.plist.bak-*`；缺失 0 个；内容不同的**只有 `OC/config.plist`**。据此才能确定「只换 config」成立，并把精力集中到 config 差异与运行时变量上。

---

## 7. 精确对比两份 config

数组索引位移会制造大量假差异（在中间插入一个 kext，后面所有项都「变了」）。**按 `BundlePath` 做集合对比**才准：

```python
import plistlib
a = plistlib.load(open('gold.plist','rb'))
b = plistlib.load(open('bt.plist','rb'))
def m(c): return {k['BundlePath']: bool(k.get('Enabled')) for k in c['Kernel']['Add']}
A, B = m(a), m(b)
print('新增 :', [k for k in B if k not in A])
print('删除 :', [k for k in A if k not in B])
print('开关变化 :', [k for k in A if k in B and A[k] != B[k]])
for sec in ['ACPI','Booter','DeviceProperties','Misc','PlatformInfo','UEFI']:
    print(sec, '相同' if a.get(sec) == b.get(sec) else '不同')
```

本项目正是用这个办法证明了：当初「装蓝牙导致起不来」的那份配置，与黄金版的差异**只有** 3 个蓝牙 kext + 2 个 NVRAM 变量 + Delete 列表，其余区段逐字节相同、零个 kext 被误关。既然配置没问题，失败就只能是运行时因素。

---

## 8. NVRAM 与 BIOS 时间的坑

- **reset NVRAM 之后绝对不能打断**，让它完整跑完一次启动循环。半写状态比不 reset 更糟。
- **FAT32 上的文件时间戳存 UTC**，Windows 若没设 `RealTimeIsUniversal=1`，两边交替把 RTC 当本地时/UTC 写，时间会跳 8 小时。看到「BIOS 时间不对」先换算时区，别急着判硬件时钟坏。
- Windows 修复：`reg add "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_DWORD /d 1 /f`
- `boot-args` 由 OC 每次启动从 `NVRAM/Add` 重新注入，reset NVRAM 不会丢 config 里写的参数。

---

## 9. 回滚矩阵

| 场景 | 回滚目标 | 说明 |
|---|---|---|
| 只是启动失败，配置刚改过 | 上一次的 `config.plist.bak-*` | 记得保留固定 slide，否则又回到抽签 |
| 怀疑 kext 文件被改 | 整个 `EFI/` 的 tgz | 整包还原，不要只换 config |
| 无蓝牙版仍想排除变量 | 23 kext 的黄金版 | 蓝牙变量全部移除 |
| 完全不知道改了什么 | 最小启动集（9 kext） | 先能进系统，再逐项加回 |

回滚后务必校验：`plutil -lint config.plist`、MD5 核对、`nvram boot-args` 确认 slide 值。

---

## 10. 这份手册的来历

一天之内，同一台机器上出现过：

- 装蓝牙三件套 → 冷启动必挂 → 回滚仍挂
- reset NVRAM → 偶尔能进 → 判定「NVRAM 半写」
- 逐文件 MD5 全量对比 → config 与黄金版逐字节一致却仍挂
- Recovery 也卡死、断电放电无效
- 做减法到 9 个 kext → 成功进桌面
- 逐项加回 → 网卡、触控板、WiFi 陆续恢复，但漏加了 USB kext
- 加 TPAD 补丁后触控板可用（**这一条后来被证明是对的**，见 README §3.3）→ 但当时误以为它也解释了启动挂死
- 清掉 113 个旧日志 → OC 恢复写日志 → 拿到关键证据
- 规范化 diff → 唯一变量是 `slide=`
- 固定 slide + 原封不动加回蓝牙三件套 → **一次成功**

最后回头看：**「XHCILagFix 有毒」「SetupVirtualMap 是正解」「装蓝牙必挂」** 这三条结论全是错的。真正的问题只有一个——**把运行时随机性当成了因果**。

> 触控板补丁需要单独澄清一下：这个补丁**是必需的**（不加以 `ELAN050A` 建不出 nub，见 README §3.3），当时错的不是「加补丁」，而是**「补丁也解释了启动挂死」这个推论**。补丁只管触控板，跟能不能进桌面毫无关系。

> 所以这份手册最想留下的一句话是：
> **先消除随机性，再谈因果；先用实测定义现象，再去找根因。**

---

## 11. 显卡 / 显示异常速查（与启动排障分开）

主教程 [README.md §3.0](README.md) 有完整安装与配置说明。这里只列「出问题时按什么顺序查」。

**前提认知**：这台机器 BIOS 是 Discrete GPU only，内屏物理走 NVIDIA。显卡驱动失效的表现是
「黑屏 / 只有固件帧缓冲 / 无 Metal / 分辨率锁死」，**但它和 `EXITBS:START`、slide 抽签毫无关系**。
两者混在一起查会浪费大量时间。

### 11.1 三条命令先定性

```bash
kmutil showloaded --list-only | grep nullmoth      # 期望 4 行
kextstat | grep -i nullmoth                        # 4 个 kext，版本一致
system_profiler SPDisplaysDataType | grep -E 'Chipset Model|Metal Support|UI Looks like|Kernel Extension Info'
nvram nullmoth-nv-ok                              # 1 = 安装器成功标记；不存在也可人工核对
```

| 结果 | 判断 | 下一步 |
|---|---|---|
| `kmutil` 0 行，但 `/Library/Extensions/NVRM.kext` 等 4 个文件都在 | AuxKC 没重建 / 系统扩展未被允许 | 系统设置 → 隐私与安全性允许扩展，重启；仍无效则重跑 1401 或 `install.sh` |
| `kmutil` 4 行，但 `SPDisplaysDataType` 无 Metal | Metal 插件或 AMFI 参数问题 | 核对 `amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0`、`SecureBootModel=Disabled` |
| 能看到 NVIDIA 但分辨率固定、无加速 | `Kernel/Block` 缺 `IONDRVSupport / Exclude`，固件帧缓冲抢了 display index 0 | 补上后重启 |
| 开机立刻 panic，日志提到 WEG / boot screen | 装了 WhateverGreen 却没加 `gfxrst=4` | 补 `gfxrst=4`（官方点名的修复） |
| 画面闪烁 / 撕裂 | 零拷贝 scan-out 时序 | 先试 `-nvrmnoflip` |
| 睡眠醒来后 GPU 丢失 | 电源树问题 | `-nvrmnopm`，并读官方 `docs/SLEEP-WAKE.md` |
| 卡在 `PCI configuration end` | **首次启动正常现象** | 等最多约 1 分钟，不要断电 |
| 屏幕冻在 `-v` 最后一行不动，但 SSH 能进、WindowServer 显示 `state = running` | **AuxKC 里 4 个驱动的路径是畸形的**，驱动一个都没装进去 | 立刻跳到 §11.4，别关 kext、别换 SMBIOS |
| 同上症状，但 `kmutil showloaded \| grep nullmoth` **是 4 行** | **不是 AuxKC 问题**：`NVRMNVDAFramebuffer` 没挂上，NVKMS 拿不到显示头，WindowServer 只建了个虚拟屏 | **关机 → 拔电源 → 长按电源键 15 秒放电 → 开机**（普通重启无效） |

> **⚠️ 同一个「卡在 -v、没有桌面」有两种完全不同的病，判据只有一条命令：**
>
> ```bash
> kmutil showloaded --list-only | grep -c nullmoth
> #   0 → AuxKC 路径畸形，去 §11.4
> #   4 → 继续往下查 framebuffer
> ```
>
> 第二种（framebuffer 竞态）的三行判据：
>
> ```bash
> ioreg -rc NVRMNVDAFramebuffer -w0 | grep -c "class NVRMNVDAFramebuffer"    # 故障 = 0（正常 = 1）
> ioreg -rc NVRMBootFB -w0 | grep -c "class NVRMBootFB"                       # 故障 = 0
> sudo log show --last 5m --style compact | grep "display stack"
> #   正常：display stack: heads 0x1 accel 4 agdc 1 k5 1 iop 1
> #   故障：display stack: heads 0x0 accel 0 agdc -99 k5 -99 iop -99
> sudo log show --last 5m --style compact | grep "boot guard"
> #   正常：boot guard cleared: the framebuffer has its scanout memory and a display mode
> ```
>
> 成因：`NVRMBootFB` 与系统内置的 `IONDRVFramebuffer` 同为 `IOMatchCategory=IOFramebuffer`，一个卡只能挂一个槽位；
> NV 那几个 kext 在 `/Library/Extensions`、经 AuxKC 加载较晚，**连续快速重启最容易把时序打乱导致抢不到槽位**。
> 与 config、kext、SMBIOS 全都无关——**所以别再去关 kext 做二分**（这是最容易走偏的地方）。
>
> 唯一有效解法：**完全关机 → 拔电源适配器 → 长按电源键 15 秒放电 → 插回电源开机**。
> 普通 `shutdown -r now`、关机后直接按电源键，都试过无效。
>
> 另一个必须知道的保护机制：驱动在 NVRAM 记 `nullmoth-nv-starts`，**连续 2 次启动没拿到显示就永久自禁用**。
> 排查期间每次重启前先清一次：`sudo nvram -d nullmoth-nv-starts`（保留 `nullmoth-nv-ok=1`）。

### 11.2 常见配置遗漏（逐项对照官方 README）

```text
NVRAM/Add/csr-active-config            = 430A0000   (0x0A43，SIP 不能全开)
Misc/Security/SecureBootModel          = Disabled
Kernel/Block                           = com.apple.iokit.IONDRVSupport / Exclude
boot-args 必含                          = nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth
                                         amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0
boot-args 有 WhateverGreen 时必加      = gfxrst=4
boot-args 禁止                          = nv_disable=1 / -wegnoegpu / agdpmod=pikera
BIOS                                   = Above 4G Decoding ON，CSM OFF
SMBIOS                                 = iMacPro1,1
ResizeGpuBars / ResizeAppleGpuBars     = 保持固件原值，不要抄别人的
DeviceProperties                       = NVIDIA 卡不需要任何属性
```

**别把显卡 kext 放进 `EFI/OC/Kexts`**。这是与 OpenCore 其他驱动最不同的一点：它们由系统卷的
Auxiliary Kernel Collection 加载，OpenCore 只负责 SIP 和 boot-args。

### 11.3 系统更新后的恢复流程

macOS 安全更新/大版本会重建 AuxKC，`com.nullmoth.*` 可能不再加载：

```bash
# 1. 先确认文件还在
ls -d /Library/Extensions/{NVRM,NVAccel,NVRMFB,NVRMAGDC}.kext
ls -d /Library/GPUBundles/NVMTLDriver.bundle /Library/GPUBundles/nvmtl

# 2. 重跑官方安装器（推荐，它会同时检查 config）
#    或手动执行 release 包里的 install.sh，然后重启两次

# 3. 验证
kmutil showloaded --list-only | grep nullmoth
```

用 1401 安装的话，OC 启动菜单里那条 **「1401: Remove NVIDIA driver」** 是可逆入口；
需要临时回到无驱动状态排查时选它，比手工删文件安全得多。

### 11.4 跨版本重建 AuxKC：`-p` 会剥掉 `-R` 前缀（macOS 26 实测）

在 macOS 15 里给 macOS 26 卷重建 AuxKC 时，如果目标卷名带空格（APFS Data 卷通常叫 `TAHOE - Data`），
`kmutil create` 会把 `-p` 的路径**按 `-R` 前缀剥掉、剩下的当相对路径拼到当前工作目录**：

```bash
# 在 15 里执行，cwd 是 /Users/acer
kmutil create -R "/Volumes/TAHOE" -p "/Volumes/TAHOE - Data/Library/Extensions/NVRMAGDC.kext" ...
# 剥掉 /Volumes/TAHOE 后剩 " - Data/Library/Extensions/NVRMAGDC.kext"（相对路径）
# 拼 cwd → /Users/acer/ - Data/Library/Extensions/NVRMAGDC.kext   ← 不存在
```

**怎么发现**：`kernelmanagerd` 日志里会明确写出来，注意路径中间那个突兀的 ` - Data/`：

```
Installation check: missing bundle for com.nullmoth.NVRMAGDC at /Users/acer/ - Data/Library/Extensions/NVRMAGDC.kext
```

四个驱动（`NVAccel` / `NVRM` / `NVRMAGDC` / `NVRMFB`）会全部被拒装。后果是内核、用户态、
WindowServer 全都正常启动，只有屏幕永远不动 —— 因为拿不到 framebuffer。

**正确做法**：到目标系统本机去跑，`cd /`，`-p` 全部用绝对路径：

```bash
cd /
PB=$(ls -d /System/Volumes/Preboot/*/boot | head -1)
sudo kmutil create -n aux -A /tmp/aux.kc -R / \
  -p /Library/Extensions/NVAccel.kext  -p /Library/Extensions/NVRM.kext \
  -p /Library/Extensions/NVRMAGDC.kext -p /Library/Extensions/NVRMFB.kext \
  -k /System/Library/Kernels/kernel \
  -B "$PB/System/Library/KernelCollections/BootKernelExtensions.kc" \
  -S /System/Library/KernelCollections/SystemKernelExtensions.kc \
  --build "$(sw_vers -buildVersion)" \
  --allow-missing-kdk --no-authentication --no-authorization --force
sudo cp /tmp/aux.kc /Library/KernelCollections/AuxiliaryKernelExtensions.kc
sudo chown root:wheel /Library/KernelCollections/AuxiliaryKernelExtensions.kc
sudo chmod 744 /Library/KernelCollections/AuxiliaryKernelExtensions.kc
```

三条走不通的路，别重复试：`kmutil install --volume-root`（打印 `rebuild done` 但文件不变）、
`--update-all`（会连 BootKC 重建，版本不匹配报 `com.apple.kpi.iokit not found`）、
`-L`（漏 `IOGraphicsFamily` 等隐式依赖）。

### 11.5 与其他章节的边界

- 如果 **连 OC picker 都看不到**：那是固件/启动项问题，回到 §1 决策树
- 如果 **picker 能显示、但进系统后黑屏 / 无 Metal**：本节
- 如果 **画面正常、只是随机起不来**：§3 slide 抽签
- 如果 **Type-C 接外接屏没信号**：物理 MUX 切断，无解，见 README §3.8，不要试图靠改显卡驱动解决
