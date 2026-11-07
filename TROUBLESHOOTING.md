# 排障手册：从「玄学启动失败」到定位 KASLR slide 抽签

配套主教程：[README.md](README.md)。本文专门讲**系统起不来时怎么查**，包含一整套可复用的流程和这次踩过的每一个误判。

---

## 0. 先记住三个结论

1. **`EXITBS:START` 不是卡死点**。它是 OpenCore 能写的最后一行，成功和失败的日志都以它结尾。看到它不等于卡在这里，失败发生在 XNU 接管之后。
2. **同一份 config 有时能进有时不能，先怀疑运行时随机量**（slide / 内存布局），不要急着回滚配置。
3. **设备「突然坏了」时，先查是不是被自己关掉了**。本项目两次大弯路（USB、触控板）都源于排障做减法时关掉 kext 忘记加回。

---

## 1. 决策树：系统起不来

```
系统起不来
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

---

## 5. 做减法（最小启动集）与它的陷阱

当怀疑问题在 kext 层时，关掉大部分 kext，只留最小必需集：

```
Lilu · AMFIPass · AppleALC · AppleMCEReporterDisabler
VirtualSMC · VoodooPS2 四件套 · WhateverGreen
```

本项目用这 9 个成功进了桌面，从而证明问题确实在 kext 层。之后逐项加回（每次一层备份 + 冷/热启动验证）。

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
- 加 TPAD 补丁后触控板可用 → 误以为找到了 ACPI 根因
- 清掉 113 个旧日志 → OC 恢复写日志 → 拿到关键证据
- 规范化 diff → 唯一变量是 `slide=`
- 固定 slide + 原封不动加回蓝牙三件套 → **一次成功**

最后回头看：**触控板补丁、XHCILagFix 有毒、SetupVirtualMap 是正解、装蓝牙必挂**这四条结论全是错的。真正的问题只有一个——**把运行时随机性当成了因果**。

> 所以这份手册最想留下的一句话是：
> **先消除随机性，再谈因果；先用实测定义现象，再去找根因。**
