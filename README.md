# ASUS UX501JW · macOS Sequoia 15.6 EFI / OpenCore 完整配置

> **English TL;DR** — A working OpenCore EFI for the **ASUS ZenBook Pro UX501JW (i7-4720HQ / HD 4600 / GTX 960M)** running **macOS 15 (Sequoia)**.
> The Haswell iGPU gets full **Metal 2** acceleration via **OCLP Post-Install Root Patches**, and the built-in Intel Wi-Fi gets the **native macOS Wi-Fi UI** via the **laobamac AirportItlwm fork — with zero Wi-Fi root patches needed**.
> Detailed instructions are in Chinese below; open an issue if you need an English translation.

## 一、机型与成果

| 项目 | 状态 |
|---|---|
| CPU | Intel Core i7-4720HQ (Haswell) ✅ 原生支持 |
| 核显 | Intel HD 4600 ✅ **Metal 2 硬件加速**,显存 1536 MB,1080p 原生,无花屏 |
| 独显 | GTX 960M ❌ macOS 无驱动(Hackintosh 常态,系统内隐藏即可) |
| Wi-Fi | 内置 Intel 网卡 ✅ **原生菜单栏 UI + 锁屏自动连接**(AirportItlwm) |
| 蓝牙 | ✅ IntelBluetoothFirmware + BlueToolFixup |
| 声卡 | ✅ AppleALC,layout-id 3(boot-arg alcid=27) |
| 隐私权限 | ✅ **AMFIPass 1.4.1**, 第三方 App 正常弹窗申请摄像头/麦克风权限 |
| 电池/亮度/USB/读卡器 | ✅ SMCBatteryManager / BrightnessKeys+PNLF / USBToolBox / Sinetek-rtsx |
| 开机画面 | ✅ 标准 Apple 标志+进度条(已关闭 verbose 跑码) |

运行系统:**macOS 15.6 (24G84)**,SMBIOS = **MacBookPro14,1**。

## 二、EFI 结构

```
EFI/
├── BOOT/                     # OpenCore 回落引导
└── OC/
    ├── ACPI/                 # 11 个 SSDT(含 SSDT-Disable_GPU_PEG0 屏蔽独显、SSDT-GPRW 睡眠防秒醒)
    ├── Drivers/              # OpenRuntime / HfsPlus / OpenCanopy / ResetNvramEntry
    ├── Kexts/                # 22 个内核驱动(见下表)
    ├── Resources/            # OpenCanopy 主题资源
    └── config.plist          # 主配置(⚠️ 个人序列号等已清空,见第四节)
```

| Kext | 版本 | 用途 |
|---|---|---|
| Lilu | 1.7.3 | 补丁引擎 |
| AMFIPass | 1.4.1 | 保持 AMFI 开启并放行 OCLP 显卡补丁, 解决第三方 App 摄像头/麦克风权限问题 |
| VirtualSMC(含 SMCProcessor/SMCSuperIO/SMCBatteryManager/SMCLightSensor) | 1.3.8 | SMC 模拟 / 电池 / 风扇 / 传感器 |
| WhateverGreen | 1.7.1 | 显卡修复 |
| AppleALC | 1.9.8 | 声卡 |
| **AirportItlwm** | **2.4.0(laobamac fork,Sequoia 构建)** | Intel Wi-Fi 原生 UI |
| itlwm | 2.3.0 | Wi-Fi 备用方案(config 中已 **Enabled=false**) |
| IntelBluetoothFirmware / IntelBTPatcher | 2.5.0 | Intel 蓝牙 |
| BlueToolFixup | 2.7.3 | macOS 12+ 蓝牙适配 |
| USBToolBox + UTBMap | 1.0 | USB 3.0 (5 Gbps) 物理端口定制(左1右2满速+防15端口截断) |
| VoodooPS2Controller | 2.3.8 | 触控板 |
| ECEnabler | 1.0.6 | EC 电池 |
| AsusSMC | 1.4.1 | 华硕机型电池/EC 特性 |
| BrightnessKeys | latest | 亮度键 |
| ForgedInvariant | 1.5.0 | Haswell invariant 修复 |
| Sinetek-rtsx | 9.0.0 | SD 读卡器 |

> ⚠️ **itlwm 与 AirportItlwm 绝不能同时启用**(config 里二选一)。

## 三、核显加速:OCLP Haswell Root Patch(最关键的一步)

本机 HD 4600 的 device-id 是 **0x0416**,而 OCLP 的 Haswell 支持表里只收 [0x0412, 0x0A26, 0x0A2E, 0x0D22, 0x0D26],所以:

1. **config 中给 IGPU 注入 `device-id = 0x0412`**(伪装桌面版 HD 4600)。
2. 打开 OCLP(OpenCore Legacy Patcher)2.5.1+:
   - ⚙️ Settings → **Target Model 手动选 `MacBookPro11,5`** —— 不选的话会提示 "No applicable patches"
3. **Post-Install Root Patch** → "Intel Haswell" 补丁集 → 重启生效。
4. **千万不要点 Build & Install OpenCore**,会覆盖你自己的 EFI!

每次 macOS 小版本更新(15.x → 15.y)后需要重新 Root Patch 一次;EFI 配置不用动。

### IGPU DeviceProperties(`PciRoot(0x0)/Pci(0x2,0x0)`)

| key | 值 | 作用 |
|---|---|---|
| `AAPL,ig-platform-id` | `BgAmCg==`(0x0A260006) | Haswell 笔记本 framebuffer |
| `device-id` | `EgQAAA==`(0x0412) | **伪装桌面 HD 4600,OCLP 补丁识别关键** |
| `hda-gfx` | `onboard-1` | HDMI 音频 |
| `framebuffer-patch-enable` | `AQAAAA==` | framebuffer 补丁开关 |
| `framebuffer-cursormem` | `AACQAA==`(9MB) | **修复花屏的经典开关** |
| `AAPL00,override-no-connect` | SAMSUNG 156HL01-102 完整 EDID | 强制内屏正确参数 |

### boot-args

```
alcid=27 alcverbs=1 -no_compat_check keepsyms=1 debug=0x100 -wegbeta -amfipassbeta
```

- 使用 `AMFIPass.kext` + `-amfipassbeta` 替代了传统的 `amfi_get_out_of_my_way=1`，保持系统原生 AMFI / TCC 权限校验正常运行，彻底解决第三方应用（微信、腾讯会议、Zoom 等）无法弹出摄像头/麦克风权限申请的问题，同时放行 OCLP Haswell 显卡根补丁。
- `-wegbeta`: 允许 WhateverGreen 在 macOS 15 Sequoia 上正常执行核显接口与补丁逻辑。
- `-v`(verbose 跑码)已移除;需要调试时临时加回。

## 四、⚠️ 使用本 EFI 前必做

1. **所有个人标识已替换为占位符**(序列号/MLB/UUID/ROM 均为 0),不能直接引导使用。
2. 请自行用 **GenSMBIOS**(CorpNewt)生成 MacBookPro14,1 的序列号、MLB、SystemUUID、ROM 并填回 `PlatformInfo → Generic`,否则 iCloud/imessage服务等可能异常。
3. USB 端口映射(USBToolBox)是按 UX501JW 本机做的,直接可用;其他机型请重做。

## 五、Wi-Fi:原生菜单栏体验(laobamac fork 方案)

Sequoia 下 Intel Wi-Fi 传统只有两条路:itlwm+HeliPort(第三方 UI),或 OCLP "Modern Wireless" 根补丁(要在 Kernel→Block 里禁 com.apple.iokit.IOSkywalkFamily + 注入一堆 kext + 每次更新重打,且可能影响 iServices)。本 EFI 走第三条路 —— **laobamac 的 itlwm fork 直接为 Sequoia 提供了 AirportItlwm**:

- **零 Wi-Fi Root Patch**:不动 IOSkywalk / IO80211FamilyLegacy / AMFIPass 任何系统组件
- 与 OCLP 的 Haswell 图形补丁**零冲突**
- 菜单栏原生 Wi-Fi 图标、系统设置原生 Wi-Fi 页面、锁屏/开机自动连接
- 蓝牙不受影响(IntelBluetoothFirmware 照常)

本 EFI 已完成集成;若想自行升级,去 [laobamac/itlwm Releases](https://github.com/laobamac/itlwm/releases) 下载 `AirportItlwm-Sequoia-*.zip` 放进 Kexts。注意 HeliPort 对此方案 **无用**(itlwm 的用户态 App),可以不装。

## 六、FAQ

**Q:重启后 Wi-Fi 搜不到网?** 检查 Kernel→Add 里 itlwm=false / AirportItlwm=true;`DisableIoMapper` 需开启。

**Q:开机花屏?** 检查第三节 framebuffer 三件套(patch-enable + cursormem + EDID)是否在位。

**Q:更新系统后显卡变 VESA(7MB 显存)?** 正常 —— 更新清掉了 Root Patch,重打 OCLP Haswell 补丁即可。

**Q:这份 EFI 能直接启动吗?** 不行,先按第四节生成自己的序列号等填回去。

**Q:Windows 10 会受影响吗?** 不会,`EFI/Microsoft` 未受影响;本仓库不包含 Windows 引导文件。

## 七、致谢

- [Dortania / OpenCore Legacy Patcher](https://github.com/dortania/OpenCore-Legacy-Patcher) — Haswell 图形补丁
- [laobamac](https://github.com/laobamac/itlwm) — Sequoia AirportItlwm fork
- [OpenIntelWireless](https://openintelwireless.github.io/) — itlwm / IntelBluetoothFirmware
- [Acidanthera](https://github.com/acidanthera) — OpenCore / Lilu / VirtualSMC / VoodooPS2 等
- [5T33Z0 / OCLP4Hackintosh](https://github.com/5T33Z0/OCLP4Hackintosh) — 思路指引
- [corpnewt](https://github.com/corpnewt) — GenSMBIOS 等配套工具
