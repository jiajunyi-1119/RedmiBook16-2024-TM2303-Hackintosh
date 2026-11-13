# Redmi Book 16 2024 (TM2303) 黑苹果 EFI — i5-12450H / Sequoia 15.8.3

小米 Redmi Book 16 2024（TM2303，i5-12450H + UHD Graphics 48EU）的黑苹果 EFI 与完整安装记录。Windows 11 Pro + macOS Sequoia 15.8.3 双系统，核显 Metal 加速正常，日常可用。

> **来源声明**：本 EFI 基于 [TigerShota/RedmiBook-2024-TM2205](https://github.com/TigerShota/RedmiBook-2024-TM2205)（RedmiBook 2024 焕新版，同为 i5-12450H / 0x46A3）修改适配而来——正是在该 EFI 的基础上才成功点亮 macOS 安装界面。针对本机（TM2303）的差异做了大量修改与实测，详见下文。

> **免责声明**：本 EFI 仅针对本机型调校，其它机器直接套用可能无法启动。安装前务必备份，一切操作风险自负。

---

## 硬件配置

| 项目 | 型号 |
|---|---|
| 机器 | XIAOMI Redmi Book 16 2024 (TM2303) |
| CPU | Intel Core i5-12450H（8 核 12 线程，Alder Lake-H） |
| 核显 | Intel UHD Graphics `8086:46A3`，48 EU，rev 0x0C |
| 内存 | 16 GB LPDDR5-5200 |
| 硬盘 | FORESEE XP2000F512G 512 GB NVMe |
| 屏幕 | 1920×1200，面板 CSO163B |
| 无线 | Intel AX203 Wi-Fi 6 `8086:51F0` |
| 蓝牙 | Intel AX203 蓝牙 `8087:0026` |
| 声卡 | Realtek ALC256 |
| 触控板 | Goodix GXT7863（I2C-HID，控制器 `8086:51E8`） |
| 引导 | OpenCore 1.0.8 + OpenCanopy |
| 系统 | Windows 11 Pro + macOS Sequoia 15.8.3 |

## 驱动状态

| 功能 | 状态 | 方案 |
|---|---|---|
| 核显 Metal 加速 | ✅ 完美 | WhateverXe 0.7.0 + Graphics 0.4.1，显存 1536 MB |
| Wi-Fi | ✅ | Intel AX203，itlwm 族驱动（见下文说明） |
| 蓝牙 | ✅ | IntelBluetoothFirmware 2.5.0 + IntelBTPatcher + BlueToolFixup 2.7.3 |
| 触控板 | ⚠️ 可用 | VoodooI2C + VoodooI2CHID，轮询模式（见已知问题） |
| 声音 | ✅ | AppleALC，layout-id=13 |
| 电池 | ✅ | SMCBatteryManager + ECEnabler |
| USB | ✅ | USBToolBox + USBInjectAll + UTBDefault |
| 睡眠 | ⚠️ | 合盖不睡眠已修复（见 ACPI patch 说明），开盖睡眠未验证 |
| 隔空投送/AirDrop | ✅ | ForbiddenItlwm 0.5.1 + BTLEAdvFixup（本次更新新增） |

---

## ⚠️ 使用前必读：生成你自己的三码

本包 `EFI/OC/config.plist` 中的 `PlatformInfo → Generic` 三码是**占位符**（`REPLACE-ME-WITH-GENSMBIOS`），直接引导会异常。

1. 用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 生成一套三码，机型选 **MacBookPro16,2**；
2. 填入 `config.plist` 的 `SystemSerialNumber` / `MLB` / `SystemUUID`；
3. ROM 保持默认（`112233445566` 是占位假 MAC，可换成自己网卡 MAC）；
4. **绝对不要使用别人分享的三码**（会导致 iMessage/FaceTime 串号甚至 Apple ID 风险）。

无线预填配置同样已清空：如需 itlwm 开机自动连 Wi-Fi，编辑 `EFI/OC/Kexts/itlwm.kext/Contents/Info.plist` 中 `IOKitPersonalities → itlwm → WiFiConfig`（本机实际使用 ForbiddenItlwm + 系统内连接，此步可选）。

---

## 安装流程

### 1. 分区

只从 D 盘末尾收缩，**不动 C 盘、Windows ESP、MSR、Recovery**。参考布局：

| 分区 | 内容 | 大小 |
|---|---|---|
| 1-4 | Windows EFI / MSR / C: / WinRE | 不动 |
| 5 | Windows D:（收缩后） | 93.58 GB |
| 6 | OC FAT32（盘符 O:） | 2 GB |
| 7 | APFS（macOS 容器） | ~108 GB |

```text
select disk 0
select partition 5
shrink desired=104448
create partition primary size=2048
format quick fs=fat32 label="OC"
assign letter=O
create partition primary size=102000
set id=7C3457EF-0000-11AA-AA11-00306543ECAC
```

> 分区编号可能不同，操作前必须重新确认，不要照抄。

### 2. BIOS 设置

本机 BIOS：`RMAAD6B0P0606`（Insyde）。

- Secure Boot：**Disabled**
- Boot Mode：UEFI only，CSM Disabled
- VT-d：可关则关，否则 OpenCore 里 `DisableIoMapper=true`
- CFG Lock：可关则关，否则 `AppleXcpmCfgLock=true`
- DVMT：本机不可调，按 60 MB 处理（boot-args 里 `-igfxdvmt` + stolenmem patch 补偿）
- 本机 BIOS 无 Hyper-Threading 开关，用 `CpuTopologyRebuild` 正常跑 12 线程

### 3. OpenCore 与 Recovery

- EFI 放到 O: 盘（`O:\EFI`），本包已含完整可用 EFI
- `O:\com.apple.recovery.boot` 放 `BaseSystem.dmg` + `BaseSystem.chunklist`（建议下载 15.8.x 的 Recovery，安装器会自动拉最新 15.8.3）
- 固件启动顺序：OpenCore 第一，Windows Boot Manager 第二
- 若 macOS"启动磁盘"把 OpenCore 写到 Windows ESP（表现为直接进 Windows），管理员 PowerShell 修复：

```powershell
bcdedit /set {你的OC启动项GUID} device partition=O:
bcdedit /set {你的OC启动项GUID} path \EFI\OC\OpenCore.efi
bcdedit /set {fwbootmgr} displayorder {你的OC启动项GUID} {bootmgr}
```

> 不要动 `{bootmgr}` 的 path。

### 4. 安装 macOS

1. OpenCore 里选 `macOS Base System`
2. 连网过认证（itlwm 连接后 Safari 过 Portal）
3. `diskutil list` 确认目标分区，只初始化 APFS 分区：

```bash
diskutil apfs create disk0s7 "macOS"
```

4. `Reinstall macOS Sequoia` → 安装，多次重启都选 `macOS Installer` / `macOS`
5. 设置助手：不迁移数据、先不登 Apple ID、本地账户、不开 FileVault

### 5. 安装后

- **核显**：安装 `WhateverXe-Graphics-0.4.1.pkg`，重启后关于本机应显示 Intel UHD Graphics 1536 MB 且 Metal 可用。注意：WEX 官方标定 15.8.1，本机在 **15.8.3 实测可用**；不要升级 15.9/Tahoe，可能失效
- **时间同步**：Windows 注册表 `RealTimeIsUniversal=1` + 时区 China Standard Time，双系统时间即一致
- **引导页**：`HideAuxiliary=true`、`ShowPicker=true`、`Timeout=20`、`PickerMode=External`，默认只显示 Windows/macOS，按**空格**可进 Recovery / Reset NVRAM

---

## 本次更新（2026-11-13）新增内容

相对 2026-11-12 的包，以下为验证通过的改动：

### 1. 合盖防睡眠

新增 ACPI Patch：`Disable lid PNP0C0D -> PNP0000`（已启用）。

合盖不再触发睡眠，外接显示器使用不受影响；需要睡眠时从菜单手动睡。

### 2. Intel Wi-Fi / AirDrop 栈升级

- 新增 `ForbiddenItlwm.kext` 0.5.1（定制 itlwm 变体）+ `BTLEAdvFixup.kext`
- `itlwm.kext` 已停用（保留在包内仅作备用，Info.plist 已清空网络预填）
- boot-args 新增一批 `-itlawdl*` 蓝牙/Wi-Fi 共存调优参数（AirDrop 稳定性）

### 3. config 清理

boot-args 确认保留 `-vi2c-force-polling`（触控板轮询模式）。

---

## 已知问题与实验记录

### 触控板：Goodix GXT7863（轮询模式可用，滑动会跳）

GXT7863 是双模式 I2C-HID 设备（描述符同时含相对鼠标 rid 1 + Win8 PTP rid 4）。当前状态：

- **轮询模式**（`-vi2c-force-polling`）：能正常移动、点击、多指手势，但指针偶发跳动、拖动不稳定
- **GPIO 中断模式**：VBT 里没有 GPI0 控制器映射，无法使用（`SSDT-GPI0` 方案已验证失败）
- **APIC 中断模式**：**已验证可以打通**，方法如下，供后续实验参考：
  1. SSDT 写 `GNVS.TPDM = 1`（本机 ACPI 的 TPD0 `_CRS` 三分支，`TPDM==0` 走 GPIO、否则走 APIC）
  2. 中断号为 GSI 81（0x51），超出 VoodooI2C 的 `validateAPICInterrupt` 上限 0x2F(47)，需二进制补丁放行
  3. 打通后触摸板可用，但**滑动跳动依旧**——已定性为与传输方式无关
- 跳动根因排查方向（供后来者）：VoodooI2C 的 Force Touch 仿真（关掉系统设置的"用力点按和触感反馈"测试）+ 固件 Confidence 置信位误判（palm rejection 过敏）。未完成修复
- 结论：**当前轮询模式 + USB 鼠标备用是最稳方案**，等待 VoodooI2C 上游对 Goodix 的完整支持

### 核显：WhateverXe 0.8.x 在 0x46A3 上的回归

- WEX 0.8.2 + Graphics 0.4.3 在 0x46A3（rev 0x0C，48 EU）上无法加速（8 MB VRAM、stub IOAccelerator），已回退 0.7.0
- 社区 contrib 分支（`xps9720-igpu` 构建）在本机实测**可满速加速**（Metal 2、1536 MB），需 `-wex-edp-keep -wex-no-edp-hpd`，但存在**背光调节失效**的回归，尚未解决
- 讨论详见 [WhateverXe issue #114](https://github.com/b00t0x/WhateverXe/issues/114)
- 结论：**0x46A3 用户请使用 0.7.0**，等 0.8.3+ 正式修复

---

## EFI 结构

```text
EFI/
├── BOOT/BOOTx64.efi
└── OC/
    ├── OpenCore.efi                # OpenCore 1.0.8
    ├── config.plist                # 三码已脱敏为占位符
    ├── ACPI/                       # 14 个 SSDT（PLUG-ALT/PMC/RHUB/XOSI/PNLF/HPET/AWAC/EC-USBX 等）
    ├── Drivers/                    # HfsPlus / OpenCanopy / OpenRuntime / ResetNvramEntry
    ├── Kexts/                      # 27 个 kext（详见上方驱动状态表）
    └── Resources/                  # OpenCanopy 主题
```

## 更新日志

- **2026-11-13**：新增合盖防睡眠 ACPI patch；Wi-Fi 栈切换 ForbiddenItlwm + BTLEAdvFixup（AirDrop 可用）；新增 itlwm coex boot-args；触控板 APIC 中断模式实验记录整理；三码与 Wi-Fi 预填全部脱敏；补全来源声明（基于 TigerShota TM2205 EFI 修改）
- **2026-11-12**：初始打包（完整安装流程总结）

## 致谢

- [TigerShota/RedmiBook-2024-TM2205](https://github.com/TigerShota/RedmiBook-2024-TM2205) —— 本 EFI 的直接基础，RedmiBook 2024 焕新版（TM2205）的 EFI 与踩坑记录；VoodooI2C 对 `8086:51e8` 的魔改支持也源于此
- [b00t0x/WhateverXe](https://github.com/b00t0x/WhateverXe)（含 jkbuha、TrDiTu 在 issue #114/#76/#35 的分析）
- [VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) 团队
- [acidanthera](https://github.com/acidanthera)（Lilu/OpenCore/VirtualSMC 全家桶）
- zxystd（itlwm）、OpenCore 社区与黑果小兵教程

---

*本 EFI 基于个人机器实测整理，随缘更新。*
