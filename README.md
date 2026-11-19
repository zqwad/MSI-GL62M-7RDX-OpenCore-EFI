# MSI GL62M 7RDX（MS-16J9）OpenCore EFI

适用于 **MSI GL62M 7RDX 系列 / 主板 MS-16J9** 的黑苹果 EFI，在 **macOS Ventura 13.0.1** 上实机验证可用。本机为 `GL62M 7RDX-1286CN` 子型号，同一模具的其它 7RD/7RDX 配置（同为 i5-7300HQ + HD 630）基本通用。

- OpenCore **1.0.7**（官方 RELEASE 二进制，无任何自编译文件）
- SMBIOS：`MacBookPro14,2`
- 配置校验：`ocvalidate 1.0.7` → `No issues found`
- 规模：130 个文件 / 34.5MB

> ⚠️ **开工前必读**
> 1. `config.plist` 里的三码全是占位符，**必须自己生成**，否则 iCloud / App Store / iMessage 全部不可用（见「五、使用步骤」第 1 步）。
> 2. 刷写前请**整份备份原有 EFI 分区**。改 EFI 导致无法开机是最常见的翻车方式，而回滚只需要把备份拷回去。
> 3. **`HackrNVMeFamily.kext` + `ssdt-nvme.aml` 是给三星 PM981a 这一块盘定制的，不是通用组件。** 如果你的硬盘不是 PM981 / PM981a / PM991（大部分人不是），**必须先删掉这两项**，否则会卡在苹果 Logo。判断方法与删除步骤见 **4.9 节**。
> 4. 同样是本机专属的还有 `MSIFanService.kext`（MSI 风扇/EC 服务），换品牌机器请删除。

---

## 一、硬件配置

| 部件 | 型号 | 状态 |
|---|---|---|
| CPU | Intel Core i5-7300HQ（4C4T，Kaby Lake） | ✅ 原生电源管理 |
| 核显 | Intel HD Graphics 630（device-id `0x591B`） | ✅ Metal 3 硬件加速 |
| 独显 | NVIDIA GTX 1050 | ⛔ 完全屏蔽（macOS 无 Pascal 驱动） |
| 内存 | DDR4 2400 | ✅ |
| 主硬盘 | Samsung PM981a 1TB（`MZVLB1T0HBLR-00000`） | ⚠️ **本机为 PM981a，需要专用驱动**；用其他硬盘请删掉对应两项（见 4.9） |
| 副硬盘 | HGST HTS721010A9E630 1TB | ✅ |
| 无线网卡 | Intel Dual Band Wireless-AC 3168 | ✅ 上网正常；隔空投送/接力不支持 |
| 有线网卡 | Atheros AR8171/8175 | ✅ |
| 声卡 | Realtek ALC（HDA） | ✅ `layout-id = 98` |
| 蓝牙 | Intel 蓝牙 | ✅ |
| 触摸板 | PS2 + I2C 触摸板 | ✅ 多指手势可用 |
| 屏幕 | 内置 15.6" 1920×1080 | ✅ 亮度可调 |

---

## 二、功能可用性（哪些能用、哪些不能）

### ✅ 完全可用

| 功能 | 说明 |
|---|---|
| 图形加速 | `system_profiler` 显示 `Metal Support: Metal 3`，界面流畅、可跑 GPU 加速应用 |
| 内屏显示 | 1920×1080，eDP 通道，亮度可调 |
| HDMI 外接 | 走核显，实测最高 **2560×1440@60**，30-bit 色深 |
| 音频 | 内建扬声器、耳机、HDMI 音频均正常 |
| 有线网络 | 插网线即用 |
| 无线网络 | Wi-Fi 正常连接 |
| 蓝牙 | 可配对键鼠、耳机 |
| USB | 含 USB 3.0 端口映射，U 盘/移动硬盘正常 |
| 触摸板/键盘 | 多指手势、双指滚动、三指拖移可用 |
| **摄像头** | ✅ 内建摄像头正常，FaceTime / 拍照可用 |
| 电池 | 电量百分比、充电状态正常 |
| 风扇/温度 | 温度传感器可读，风扇策略可用 |
| **睡眠 / 唤醒** | ✅ **正常**。合盖休眠、开盖唤醒、电源键唤醒均可用（采用内存休眠，不写盘休眠） |

### ❌ 不可用 / 受限

| 功能 | 状态 | 原因 |
|---|---|---|
| **mini-DP 外接输出** | ❌ 不出图 | 详见「六、已知问题 1」 |
| **4K 分辨率** | ❌ 拿不到 | HDMI 口硬件规格 4K@30、EDID 只报 4K60，macOS 直接不给任何 4K 模式 |
| 独显（GTX 1050） | ⛔ 已屏蔽 | macOS 无 NVIDIA Pascal 驱动 |
| 隔空投送 / 接力 / Handoff | ❌ | Intel 网卡不支持 |
| Sidecar / 随航 | ❌ | 同上 |
| iCloud / App Store | ⚠️ 需自备三码 | 占位符状态下无法登录 |
| 写盘休眠（Hibernate） | ⚠️ 未启用 | `HibernateMode = None`，用内存休眠替代；**日常睡眠唤醒是完全正常的** |

---

## 三、目录结构

```
EFI/
├── BOOT/
│   └── BOOTx64.efi                 # 引导入口，OpenCore 1.0.7 官方
└── OC/
    ├── OpenCore.efi                 # OpenCore 1.0.7 官方
    ├── config.plist                 # 全部配置（三码已脱敏）
    ├── ACPI/                        # 7 个 SSDT
    ├── Drivers/                     # 4 个 UEFI 驱动
    ├── Kexts/                       # 21 个 kext
    ├── Resources/                   # 官方主题（Acidanthera/GoldenGate 为默认）
    └── Tools/
        └── OpenShell.efi            # 救援用 UEFI Shell
```

**已刻意剔除**：`EFI/APPLE`、`EFI/Microsoft`（那是本机 Windows 双系统的引导残留，不属于 EFI 内容）。**要保留 Windows 双系统启动项，把原来的 `EFI/Microsoft` 放回 ESP 根目录即可。**

---

## 四、config.plist 逐项说明（哪些开着、为什么）

### 4.1 ACPI（7 个 SSDT，全部启用）

| 文件 | 作用 | 能否删 |
|---|---|---|
| `SSDT-PLUG.aml` | CPU 电源管理，启用 `X86PlatformPlugin`，缺失会导致 CPU 无法降频、发热高 | ❌ 必须 |
| `SSDT-EC-USBX.aml` | 模拟 Embedded Controller + 修正 USB 供电参数（充电、外设识别） | ❌ 必须 |
| `SSDT-PNLF.aml` | 背光控制，缺失则亮度无法调节、亮度键失效 | ❌ 必须 |
| `SSDT-dGPU-Off.aml` | 在 ACPI 层把独显的 `_ON/_OFF` 置为关闭，与 `-wegnoegpu` 双保险屏蔽 GTX 1050 | ⭕ 本机专属 |
| `ssdt-nvme.aml` | PM981a 专用，配合 `HackrNVMeFamily` 使用 | ⭕ 本机专属 |
| `SSDT-UIAC.aml` | USB 端口映射（传统方式），与 `UTBMap.kext` 功能重叠 | ⭕ 二选一 |
| `SSDT-XOSI.aml` | 操作系统判断伪装成 Windows，供部分驱动/补丁使用 | ⭕ 当前未生效 |

> 说明：`SSDT-XOSI` 通常需要配合 `ACPI/Patch` 里的 `_OSI → XOSI` 改名补丁才生效，而本 config 的 `ACPI/Patch` 为空，所以它当前**实际不起作用**，属历史遗留，删掉无影响。

### 4.2 Drivers（UEFI 驱动，4 个，全部官方）

| 驱动 | 作用 | 能否删 |
|---|---|---|
| `OpenRuntime.efi` | OpenCore 运行必需（NVRAM、内存服务） | ❌ 必须 |
| `OpenHfsPlus.efi` | 开源 HFS+ 读取驱动，用于识别 HFS 卷 | ⭕ 纯 APFS 环境可删 |
| `OpenCanopy.efi` | GUI 图形启动界面（配 `Misc/Boot/PickerMode = External`） | ⭕ 想用文字界面可删 |
| `ResetNvramEntry.efi` | 提供「Reset NVRAM」启动项，排障必备 | ⭕ 建议保留 |

**已移除**：`BiosVideo.efi`（仅 Legacy BIOS 引导需要）、`HfsPlus.efi`（Apple 私有文件，已替换为官方开源 `OpenHfsPlus.efi`）、`AudioDxe.efi`（未启用音频辅助，纯属冗余）。

### 4.3 Kexts（26 个加载项 / 21 个 kext，全部启用）

**核心（不可删）**

| kext | 版本 | 作用 |
|---|---|---|
| `Lilu` | 1.7.2 | 所有补丁类 kext 的加载框架，必须最先加载 |
| `VirtualSMC` | 1.3.8 | 模拟 SMC 芯片，没有它系统根本起不来 |
| `WhateverGreen` | 1.7.1 | 显卡补丁框架（核显加速、外接屏、独显屏蔽） |
| `AppleALC` | 1.9.7 | 声卡补丁，配 `layout-id 98` |
| `SMCProcessor` / `SMCSuperIO` / `SMCBatteryManager` / `SMCLightSensor` | 1.3.8 | CPU 温度、风扇转速、电池、环境光传感器 |

**功能（按需）**

| kext | 版本 | 作用 |
|---|---|---|
| `NVMeFix` | 1.1.4 | NVMe 电源管理修正，减少发热与掉盘 |
| `HibernationFixup` | 1.5.5 | 休眠/唤醒修正 |
| `USBToolBox` + `UTBMap` | 1.1.1 / 1.1 | USB 端口映射（现代方式） |
| `VoodooPS2Controller`（含 Keyboard/Mouse/Trackpad 三个插件） | 2.3.8 | PS2 键盘、触摸板基础驱动 |
| `VoodooRMI`（含 RMISMBus、VoodooInput） | 1.4.3 | I2C 精密触摸板，多指手势靠它 |
| `ECEnabler` | 1.0.6 | 让系统能读取超过 8 位的 EC 字段（电池信息） |
| `IntelBluetoothFirmware` | 2.5.0 | Intel 蓝牙固件上传 |
| `BlueToolFixup` | 2.6.4 | 修正 macOS 12+ 蓝牙协议栈兼容性 |
| `AirportItlwm` | 2.2.0 | Intel 无线网卡驱动 |
| `AtherosE2200Ethernet` | 2.2.2 | 有线网卡驱动 |

**本机专属（换机请删）**

| kext | 版本 | 说明 |
|---|---|---|
| `HackrNVMeFamily` | 92.1.0 | **PM981a 专用**。三星 PM981/PM991 系列在 macOS 下会因原生 NVMe 驱动超时导致内核崩溃（`IONVMeController: Command timeout. Write`），此 kext 替换原生驱动解决该问题。**如果你的盘不是这三个型号，必须删掉它**（见 4.9） |
| `MSIFanService` | 1.0.0 | MSI 风扇/EC 服务，其他品牌机器无用 |

> 上面两项（`HackrNVMeFamily` + `ssdt-nvme.aml`）是**为这一块 PM981a 定制的**，不是黑苹果通用组件。绝大多数 NVMe 硬盘（三星 970/980 系列、西数 SN 系列、铠侠、海力士等）在 macOS 下是原生支持的，**多装反而可能出问题**。具体判断与删除方法见 4.9。

### 4.4 Kernel / Quirks（6 项开启）

| 键 | 值 | 作用 |
|---|---|---|
| `AppleXcpmCfgLock` | true | CPU 的 CFG Lock 未在 BIOS 解锁时的必需补丁，否则无法启动 |
| `DisableIoMapper` | true | BIOS 未关闭 VT-d 时必需，否则卡在 PCI 阶段 |
| `DisableLinkeditJettison` | true | 修正 Lilu 在旧系统上的内存错误 |
| `PanicNoKextDump` | true | 崩溃时不打印 kext 列表，日志更干净 |
| `PowerTimeoutKernelPanic` | true | 避免电源超时导致的内核 panic |
| `SetApfsTrimTimeout` | -1 | **关闭 APFS Trim**。PM981a 上开启 Trim 会导致开机极慢甚至卡死 |

### 4.5 DeviceProperties（注入属性）

```
PciRoot(0x0)/Pci(0x1F,0x3)   → layout-id = 98        （声卡）
PciRoot(0x0)/Pci(0x2,0x0)    → 核显相关：
    AAPL,ig-platform-id      = 00001b59              （KBL 移动平台帧缓冲）
    framebuffer-stolenmem    = 00003001              （显存 64MB，规避 BIOS DVMT 限制）
    framebuffer-fbmem        = 00009000              （帧缓冲 9MB）
    framebuffer-patch-enable = 01000000              （开启帧缓冲补丁）
    framebuffer-con1-busid   = 06                    （HDMI 连接器指向端口 6）
    framebuffer-con1-type    = 00080000              （HDMI 类型）
    framebuffer-con2-type    = 00040000              （DP 类型）
```

> **重要说明**：`framebuffer-con1-busid` / `con2-type` 这几个键，实测**在本机没有被 WhateverGreen 应用**（ioreg 里端口表始终是 `0 / 6 / 7`）。原因已查明：WEG 的帧缓冲连接器补丁需要配套的 `framebuffer-conX-enable = 01000000` 开关键，而原始 EFI 里这些键被拼错成 `framebuffter-conX-enable`（多了一个 `t`），WEG 从未识别。想继续折腾 DP 的同学，把 `-enable` 键名改正、配上 `-alldata` 完整 12 字节记录再试。

### 4.6 UEFI

| 段 | 关键项 | 值 | 说明 |
|---|---|---|---|
| Quirks | `RequestBootVarRouting` | true | Boot 变量走 OpenRuntime 路由，多系统共存必需 |
| | `EnableVectorAcceleration` | true | 加速 UEFI 内存操作，缩短开机时间 |
| | `ResizeGpuBars` | -1 | 不改 GPU BAR，KBL 无需 |
| APFS | `EnableJumpstart` | true | 从固件加载 APFS 驱动，识别 APFS 卷必需 |
| | `MinDate` / `MinVersion` | -1 | 不限制 APFS 版本（-1 = 不检查） |
| | `HideVerbose` | true | 隐藏 APFS 驱动的啰嗦输出 |

### 4.7 Misc

| 段 | 键 | 值 | 说明 |
|---|---|---|---|
| Boot | `ShowPicker` | true | 开机显示启动菜单 |
| | `PickerMode` | `External` | 使用 OpenCanopy 图形界面 |
| | `PickerVariant` | `Acidanthera\GoldenGate` | 官方主题（原为第三方 `blackosx\BsxM1`，已移除） |
| | `HideAuxiliary` | true | 默认隐藏工具项，**按空格键**显示（Reset NVRAM、OpenShell 都在里面） |
| | `Timeout` | 5 | 菜单停留 5 秒后自动进默认启动项 |
| | `HibernateMode` | `None` | 不使用休眠，规避唤醒问题 |
| Security | `SecureBootModel` | `Disabled` | 关闭安全启动模型，黑苹果必需 |
| | `ScanPolicy` | `0` | 扫描所有类型分区（含 NTFS），双系统能看到 Windows 盘 |
| | `Vault` | `Optional` | 不做 vault 校验，改 config 后无需重建 |
| Debug | `AppleDebug` / `ApplePanic` | false | **发布版关闭**，不往 ESP 写调试日志；排障时临时改 true 并加 `-v` |

### 4.8 NVRAM

| 键 | 值 | 说明 |
|---|---|---|
| `boot-args` | `agdpmod=pikera -wegnoegpu -igfxonln=1 -cdfon` | 见下表 |
| `run-efi-updater` | `No` | 阻止固件自动更新（避免刷坏主板固件） |
| `prev-lang:kbd` | `en-US:0` | 默认输入法语言 |

**boot-args 逐条**

| 参数 | 作用 |
|---|---|
| `agdpmod=pikera` | AGDP 板卡 ID 白名单修正，避免外接屏黑屏，本机保留无副作用 |
| `-wegnoegpu` | WhateverGreen 彻底关闭独显，省电、降热 |
| `-igfxonln=1` | 强制外接显示器「在线」，改善 HDMI 热插拔识别 |
| `-cdfon` | CoreDisplay 补丁，修正外接屏色彩与刷新率协商 |

---

### 4.9 硬盘兼容性：不是 PM981 的用户必读

**这份 EFI 里的 `HackrNVMeFamily.kext` + `ssdt-nvme.aml` 是给本机那块三星 PM981a 定制的，不是通用组件。** 换别的硬盘装上大概率会出问题，请按下面的流程处理。

#### 先判断你的盘是哪一类

| 硬盘类型 | macOS 支持情况 | 要不要动那两项 |
|---|---|---|
| **普通 NVMe（绝大多数）**：三星 970 EVO/980/990、西数 SN550/SN750/SN850、铠侠 RC10/RC20、英睿达 P 系列、海力士等 | ✅ 原生支持，用系统自带的 `IONVMeFamily` 即可 | **必须删掉** |
| **问题 NVMe**：三星 **PM981 / PM981a / PM991**、部分 Intel 600p、个别早期群联主控盘 | ❌ 原生驱动会超时 → 卡 Logo 或内核 panic（`IONVMeController: Command timeout. Write`） | 保留，或改成对应型号的补丁 |
| **SATA 硬盘 / SATA SSD** | ✅ 走 AHCI，兼容性最好 | **必须删掉** |

判断方法最简单的两种：

1. **直接试**：不动 NVMe 相关配置引导一次，卡在苹果 Logo 走一半不动、或者日志末尾出现 `IONVMeController: Command timeout`，就是问题盘；
2. **查型号**：把硬盘型号（不是品牌）搜一下有没有黑苹果踩坑记录。

#### 不是 PM981 的话，怎么删干净

三处都要改，缺一不可：

1. `config.plist` → `Kernel` → `Add`：把 `HackrNVMeFamily.kext` 这一项的 `Enabled` 改成 `false`（或整项删除）；
2. `config.plist` → `ACPI` → `Add`：把 `ssdt-nvme.aml` 的 `Enabled` 改成 `false`；
3. `config.plist` → `Kernel` → `Quirks` → `SetApfsTrimTimeout`：由 `-1` 改回 `0`（PM981 才需要关 Trim，其他盘不开 Trim 反而掉性能）。

顺手可以把 `EFI/OC/Kexts/HackrNVMeFamily.kext` 和 `EFI/OC/ACPI/ssdt-nvme.aml` 两个文件也删掉，包体更干净。

#### 如果你的盘恰好也是问题盘

三条路，按省事程度排：

| 方案 | 做法 | 评价 |
|---|---|---|
| **换盘** | 换一块原生支持的 NVMe（如 SN750、970 EVO） | 最省事、最稳，长期建议 |
| **装到 SATA 位** | 本机有 2.5" SATA 位，SATA SSD 几乎无兼容问题 | 成本最低，速度慢一些 |
| **用补丁 kext** | 保留本项目的 `HackrNVMeFamily`（针对 PM981 系列），或到社区找对应主控的 NVMe 补丁 | 能用，但换盘/升级系统时容易失效 |

> 提醒：`HackrNVMeFamily` 的原理是**伪装成系统原生的 `IONVMeFamily`** 并替换它。所以它一旦启用，就是「全局接管所有 NVMe 盘」——这正是为什么非问题盘必须删掉它，否则连正常硬盘也可能识别不了。

---

## 五、使用步骤（从零到能进系统）

### 第 1 步：生成自己的三码（必做）

打开 `config.plist` → `PlatformInfo` → `Generic`，替换四个占位符：

| 键 | 当前占位 | 说明 |
|---|---|---|
| `SystemSerialNumber` | `PLACEHOLDER-SERIAL` | 序列号 |
| `MLB` | `PLACEHOLDER-MLB00` | 主板序列号 |
| `SystemUUID` | `00000000-...` | SMBIOS UUID |
| `ROM` | `000000000000` | 网卡 MAC 地址（6 字节十六进制） |

**做法**：
1. 用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS)（脚本选 3 → 输入 `MacBookPro14,2`）或 OpenCore Auxiliary Tools 生成；
2. 生成后到 [Apple 保修查询页](https://checkcoverage.apple.com/) 查一次，返回「此序列号无效」说明没被占用，可以用；
3. 把结果粘回 config 的对应四项；`ROM` 可直接用网卡 MAC（`ifconfig en0 | grep ether`）。

> 不改也能开机，但 iCloud/App Store 会登录失败，且全零 `SystemUUID` 可能引起 NVRAM 异常行为。

### 第 2 步：写入 EFI 分区

**方式 A：直接写内置硬盘 ESP（已确认能用时）**

```bash
diskutil list                       # 找到 EFI 分区，通常是 disk0s1
sudo diskutil mount disk0s1         # 挂载到 /Volumes/EFI
# 先整份备份！改名保留，出问题可改回来
sudo mv /Volumes/EFI/EFI /Volumes/EFI/EFI_backup
sudo cp -R /path/to/EFI /Volumes/EFI/
sudo diskutil unmount disk0s1
```

**方式 B：先做 U 盘引导（推荐首次使用）**

1. 准备一个 FAT32 格式的 U 盘（16GB 以上），用「磁盘工具」抹成 `MS-DOS (FAT)`、方案 `主引导记录`；
2. 把整个 `EFI` 文件夹拷到 U 盘根目录；
3. 开机按 **F11**（MSI 机型的启动菜单键），选带 `UEFI:` 前缀的 U 盘；
4. 进入 OpenCore 菜单后**按空格键**，先执行一次 **Reset NVRAM**，再选 macOS 启动；
5. 确认能进系统后，再把 U 盘里的 EFI 拷到内置硬盘 ESP（U 盘保留作救砖备用）。

### 第 3 步：BIOS 设置（MSI 机型按 Del 进 BIOS）

| 项目 | 设置为 | 原因 |
|---|---|---|
| Secure Boot | **Disabled** | 开启会直接拒绝加载 OpenCore |
| Fast Boot | **Disabled** | 快速启动会跳过 USB 初始化，U 盘引导失败 |
| SATA Mode | **AHCI** | RAID 模式 macOS 无法识别硬盘 |
| VT-d | **Disabled**（或保留但必须保留 `DisableIoMapper`） | 否则卡在 PCI 枚举阶段 |
| CFG Lock | 无需解锁 | config 已开 `AppleXcpmCfgLock` 绕过 |
| 核显 DVMT | 保持默认 | `stolenmem` / `fbmem` 已在 config 补齐 |
| Boot 顺序 | 把 `UEFI: OpenCore` 或硬盘 ESP 放第一位 | — |

### 第 4 步：首次进系统后检查

```bash
# 1. 加速是否真的开了
system_profiler SPDisplaysDataType | grep -i metal      # 应显示 Metal 3

# 2. 第三方 kext 是否全部加载
kextstat | grep -v com.apple | awk '{print $6}' | sort

# 3. 声卡
system_profiler SPAudioDataType | head -20

# 4. 网卡 / 蓝牙
ifconfig en0 | grep status
system_profiler SPBluetoothDataType | head -10
```

### 第 5 步：双系统（如需保留 Windows）

把原 `EFI/Microsoft` 目录放回 ESP 根目录，OpenCore 菜单会出现「Windows Boot Manager」条目。`ScanPolicy = 0` 已保证能扫描到 NTFS 分区。

### 第 6 步：注意事项清单（开工前逐条对照）

| # | 注意事项 | 不做的后果 |
|---|---|---|
| 1 | **先整份备份原 EFI 分区**再动手 | 改坏后无路可回，只能拆硬盘 |
| 2 | **首次一定用 U 盘引导**，验证通过后再写内置 ESP | 直接写内置盘，一旦失败连 OC 菜单都进不去 |
| 3 | **三码必须自己生成**，不要用本仓库占位符 | iCloud / App Store / iMessage 登录失败、NVRAM 异常 |
| 4 | **硬盘不是 PM981/PM991 就删掉 `HackrNVMeFamily` + `ssdt-nvme`**（见 4.9） | 卡 Logo 或识别不了硬盘 |
| 5 | 删了那两项后，`SetApfsTrimTimeout` 要改回 `0` | Trim 长期关闭，SSD 性能衰减 |
| 6 | BIOS 里 **Secure Boot 必须关、SATA 必须 AHCI、Fast Boot 必须关** | 分别对应：拒绝加载 OC / 认不到硬盘 / U 盘引导失败 |
| 7 | 首次进 OC 菜单**按空格键**，先跑一次 `Reset NVRAM` | 残留旧 NVRAM 可能导致启动项错乱、外接屏异常 |
| 8 | **每次改 `config.plist` 都用 `ocvalidate` 校验**再重启 | 语法错误直接五国，且往往看不出错在哪 |
| 9 | 本包 **Wi-Fi kext 是 Ventura 专用版**，跨大版本升级要先换 kext | 升级到 Sonoma/Sequoia 后 Wi-Fi 直接失效 |
| 10 | **不要开 APFS Trim**（PM981 上会卡死），不要启用写盘休眠 | 开机 2 分钟以上 / 睡死 |
| 11 | 系统升级后 kext 可能要重装并重建缓存 | 升级后黑屏、加速消失 |
| 12 | `EFI/APPLE`、`EFI/Microsoft` 已被剔除，**双系统需自行放回** | OC 菜单看不到 Windows |

> **一句话总结**：这包的定位是「同一型号、同一块盘」的可用配置。换硬盘、换机型、升级大版本这三件事，都会让它从「开箱能进」变成「需要改配置」。

---

## 六、已知问题

### 1. mini-DP 接口在 macOS 下不出图（未解决）

**现象**：

- 插入 DP 线，核显**能**收到热插拔信号：`[Hot_Plug] Event insert`、`isHPDLow = 0`；
- 但 AUX 通道完全无响应：`[AGDC] Failed with status e00002bc for stream access aux`，5 分钟内刷 251 条超时；
- `sinkCount = 0` → 读不到 EDID → 系统认为「什么都没插」；
- 同一根线、同一个 DP→mini-DP 转接头，在 **Windows 下画面正常**。

**已尝试并排除**：

| 尝试 | 结果 |
|---|---|
| `framebuffer-con2-busid` 07 → 05 | 未生效（ioreg 端口表仍为 0/6/7） |
| `con1` / `con2` 端口对调 | 未生效 |
| `framebuffer-conX-alldata` 完整 12 字节替换 | 未生效 |

**根因线索（留给后来者）**：WEG 的连接器补丁除 `-alldata` 外，还需要配套的 `framebuffer-conX-enable = 01000000`；原 EFI 里这些键被拼错成 `framebuffter-conX-enable`，导致补丁从未被应用。修正键名后可以继续尝试把 DP 连接器改到正确 busid（KBL 移动平台常见为 04 或 05）。

**其它可疑因素**：被动式 DP→mini-DP 转接头对 AUX 差分信号质量敏感，Windows 驱动容忍度更高。换一根 **mDP 直转 DP 的原生线**是最廉价的验证手段。

### 2. HDMI 口拿不到 4K

- 显示器（P2710V）EDID 只报了 `3840×2160@60 / 像素时钟 594MHz` 一个 4K 时序；
- 核显 HD 630 的 HDMI 规格上限是 **4096×2160@30**，该口属 HDMI 1.4 级别；
- macOS 枚举出的模式表最大只有 `2560×1440@60`，**完全没有 3840×2160 条目**。

结论：**HDMI 口的 macOS 天花板是 2560×1440@60**。想上 4K@60 只能走 mini-DP（核显 DP 规格上限 4096×2304@60），而 DP 目前不通。

### 3. 其它限制

- Intel 网卡不支持隔空投送、接力、Handoff、随航；
- 独显完全屏蔽，所有外接屏走核显；
- 写盘休眠（Hibernate）未启用（`HibernateMode = None`），日常睡眠唤醒不受影响；
- **换了非 PM981 系列的硬盘，必须先按 4.9 删掉那两项定制配置，否则可能卡 Logo**。

---

## 七、常见故障排查（FAQ）

| 症状 | 可能原因 | 处理办法 |
|---|---|---|
| **卡苹果 Logo 不动 / 进度条走一半停住** | kext 顺序错误、USB 端口问题、缓存未更新 | 加 `-v` 看最后一行卡在哪；确认 `Lilu` 排第一；换一个 USB 口 |
| **五国 panic，日志里有 `IONVMeController: Command timeout. Write`** | **问题 NVMe 盘（PM981/PM991 类）原生驱动超时**——本机真实踩过的坑 | 本机的 PM981a：同时启用 `HackrNVMeFamily.kext` **和** `ssdt-nvme.aml`，任一禁用就必然五国 |
| **换了别的硬盘后卡 Logo / 五国** | 忘了删 PM981 专用的两项 | 按 4.9 关掉 `HackrNVMeFamily` 与 `ssdt-nvme.aml`，并把 `SetApfsTrimTimeout` 改回 `0` |
| **开机后黑屏但风扇在转、硬盘在读** | 帧缓冲配置错、外接屏抢占主显示 | 检查 `AAPL,ig-platform-id` 是否为 `00001b59`；拔掉外接屏再启动 |
| **外接 HDMI 无画面** | 分辨率/刷新率协商失败 | 先热插拔一次；`-igfxonln=1` 已默认开启，仍不行换线 |
| **mini-DP 无画面** | 见第六节，未解决 | — |
| **没声音 / 只有耳机有声** | `layout-id` 不匹配 | 试 `11`、`13`、`28`、`56`、`98`，每次改完重启验证 |
| **蓝牙打不开 / 搜不到设备** | 加载顺序或端口映射问题 | 确认 `IntelBluetoothFirmware` 在 `BlueToolFixup` 之前加载；确认 USB 端口映射包含蓝牙所在内部端口 |
| **Wi-Fi 连不上 / 速度慢** | `AirportItlwm` 与系统版本不匹配 | 本项目为 Ventura 专用版；跨大版本升级需换对应版本 |
| **触摸板只能用单指** | `VoodooRMI` 的 SMBus 通道没通 | 确认 `RMISMBus.kext` 插件已加载；确认 `SSDT-EC-USBX` 生效 |
| **合盖睡醒黑屏 / 睡死** | 本机实测睡眠唤醒**正常**；若出现异常，多半是外接屏或 USB 外设导致 | 先拔掉外接屏和 USB 设备再试；保持 `HibernateMode = None`；`HibernationFixup` 已装 |
| **开机特别慢（2 分钟以上）** | APFS Trim 在 PM981a 上卡住 | 保持 `SetApfsTrimTimeout = -1` |
| **OpenCore 菜单看不到 macOS 盘** | APFS 驱动没加载 / 扫描策略过滤 | 确认 `UEFI/APFS/EnableJumpstart = true`；确认有 `OpenRuntime.efi` |
| **OC 菜单看不到 Windows 盘** | `ScanPolicy` 限制 | 保持 `ScanPolicy = 0` |
| **改了 config 之后进不去** | 语法或逻辑错误 | 改前备份；改后用 `ocvalidate` 校验 |
| **想临时看详细启动日志** | 发布版关掉了调试 | `Misc/Debug` 改 `AppleDebug=true`、`Target=3`，boot-args 加 `-v keepsyms=1 debug=0x100` |

**通用救援手段**：OpenCore 菜单按**空格键**展开辅助项 → `Reset NVRAM`；`OpenShell` 救援 Shell 可手动挂载/替换 EFI 文件；实在不行用 Windows 侧 `diskpart` 挂载 ESP 换回备份。

---

## 八、本版相对原始 EFI 的整理记录

| 项目 | 改动 |
|---|---|
| 二进制 | 全部替换为 **OpenCore 1.0.7 官方 RELEASE**（原文件与官方 0.8.7/1.0.6/1.0.7/1.0.8 哈希全不符，且 `OpenCore.efi` 内残留 `REL-XXX-YYYY-MM-DD` 自编译占位串） |
| 引导驱动 | `HfsPlus.efi`（Apple 私有）→ **`OpenHfsPlus.efi`**（官方开源）；删除 `BiosVideo.efi`、`AudioDxe.efi`；Drivers 7 → **4** |
| 三码 | 序列号 / MLB / UUID / ROM 全部替换为占位符，全目录扫描零残留 |
| 调试项 | `AppleDebug`、`ApplePanic` 关闭，`Target = 0`；boot-args 去掉 `keepsyms=1 debug=0x100` |
| 主题 | 第三方 `blackosx\BsxM1` → 官方 `Acidanthera\GoldenGate`，并删除第三方主题目录 |
| 资源 | 删除 `Resources/Audio`（432KB，未启用音频辅助）；Resources 3.3MB → **1.7MB** |
| 引导内容 | 剔除 `EFI/APPLE`、`EFI/Microsoft` |
| 冗余项 | Kernel/Add 32→26、Kernel/Patch 7→0、ACPI/Delete 2→0、ACPI/Patch 2→0、Booter/Patch 1→0、MmioWhitelist 2→0、Misc/Entries 1→0、UEFI/ReservedMemory 2→0 |
| 显示配置 | 删除 4 个拼写错误的键 `framebuffter-conX-*`（WEG 从未识别）；`con2-type` 由 HDMI 改为 DP |
| 孤儿文件 | 删除 `OC/oldConfig.plist`（73KB）、`SSDT-PLUG-DRTNIA.aml`、`VoodooSMBus.kext`、`ApplePS2SmartTouchPad.kext`、13 个未用 Tools |
| 功能修复 | boot-args 移除 `igfxfw=2`，修复核显加速（原本 Metal 不可用，现为 Metal 3） |

---

## 九、校验

```bash
./ocvalidate EFI/OC/config.plist
# → Completed validating ... No issues found
```

| 项目 | 值 |
|---|---|
| 文件数 | 130 |
| 体积 | 约 34.5MB（Kexts 31.8MB、Resources 0.6MB） |
| OpenCore | 1.0.7 官方 REL |

---

## 十、免责声明

- 本项目仅供**个人学习与研究**使用。在非 Apple 硬件上安装 macOS 可能违反 Apple 的软件许可协议，请自行判断并承担风险。
- 刷写 EFI 有导致无法开机的风险，请务必**先备份**。
- 作者不对任何数据丢失、硬件损坏或系统异常负责。
- **请自己生成三码**，不要使用他人的序列号，也不要将本仓库的占位值用于日常使用。

## 十一、参考

- [Dortania OpenCore Install Guide — Laptop Kaby Lake](https://dortania.github.io/OpenCore-Install-Guide/config-laptop.plist/kaby-lake)
- [Dortania — Disabling GPU](https://dortania.github.io/OpenCore-Install-Guide/extras/spoof.html)
- [Dortania — 启动流程与故障排查](https://dortania.github.io/OpenCore-Install-Guide/troubleshooting/boot.html)
- [OpenCore 官方文档（Configuration.pdf）](https://github.com/acidanthera/OpenCorePkg)

## 十二、致谢

- [Acidanthera](https://github.com/acidanthera)：OpenCore、Lilu、WhateverGreen、VirtualSMC、AppleALC、NVMeFix、HibernationFixup、VoodooPS2、OpenHfsPlus
- [OpenIntelWireless](https://github.com/OpenIntelWireless)：AirportItlwm、IntelBluetoothFirmware、BlueToolFixup
- [USBToolBox](https://github.com/USBToolBox)：USB 端口映射
- [VoodooRMI](https://github.com/VoodooSMBus/VoodooRMI)、[ECEnabler](https://github.com/1Revenger1/ECEnabler)
- [corpnewt](https://github.com/corpnewt)：GenSMBIOS

## 十三、许可

`config.plist`、`ACPI` 中的 SSDT 与本文档采用 **MIT** 许可；各二进制与 kext 遵循其原始许可证。
