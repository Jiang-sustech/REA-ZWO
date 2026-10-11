# 01 · ASIAIR 到底是什么堆出来的

> 调查日期：2026-10-11。所有结论标注证据强度，**弱证据明确标为推断**。
> 本文件是二手调研的综合，尚未用实机固件验证。验证计划见 README 路线图。

---

## 一、总览：三层拼装

ASIAIR **不是 ZWO 自研的嵌入式系统**，而是在成熟开源栈上叠了一层闭源封装。

```
┌─────────────────────────────────────────────┐
│  Android App  com.zwoasi.asiair  v3.0.0     │  ← Java/Kotlin + 一堆 .so
│  明文 JSON-RPC  →  TCP 4040 / 4360 / 4700   │
├─────────────────────────────────────────────┤
│  zwoair_imager / zwoair_guider /            │  ← 闭源 C/C++ ELF
│  zwoair_updater   (/home/pi/ASIAIR/bin/)    │
├─────────────────────────────────────────────┤
│  魔改 INDI (C++)  +  gphoto2  +  Siril 代码  │  ← 开源底座，被改造
│  闭源: libASICamera2.so / libasi (EAF/EFW)  │
├─────────────────────────────────────────────┤
│  魔改 Raspberry Pi OS / Raspbian (armhf)    │  ← Debian
│  自编 kernel 4.19.x                         │
├─────────────────────────────────────────────┤
│  硬件: Pi 3/4 · CM4+32GB eMMC ·             │
│        RK3568B2 · RV1126 (Mini)             │
└─────────────────────────────────────────────┘
```

---

## 二、逐条证据

### 2.1 OS 层 = 魔改 Raspbian

| 证据 | 来源 |
|---|---|
| ASIAIR Pro 跑 Linux 4.19.97，平台 Raspberry Pi 4 Model B | ASIAIR Mini hacking 帖 |
| ASIAIR Plus（CM4）用户实际用 **32 位 Raspberry Pi OS（Buster 10 / Bullseye）** 做替换系统，并 fork 官方 Raspbian kernel 与 ZWO 的 kernel config 对比 | Cloudy Nights「Development – ASIAIR Plus & Compute Module 4」 |
| ZWO 改了 `.elf` 启动文件和 kernel，导致原厂镜像只能在那一个 4 GB CM4 变体上启动（8 GB CM4 报 `start4.elf is not compatible`） | 同上 |
| 内核配置可用 `modprobe configs` → `/proc/config.gz` 取出 | Hacking the ZWO ASIAIR Plus Kernel |
| ASIAIR Mini 内核 `4.19.111`，构建者 `jerry@ubuntu`；`/boot/config.txt` 为空但装了 `raspi-config` —— 沿用了树莓派约定 | ASIAIR Mini hacking 帖 |

**结论**：Debian / Raspberry Pi OS 血统，32 位 armhf（更新包名 `asiair_armhf.deb` 印证）。
**证据强度：强。**

### 2.2 应用层 = 魔改 INDI

| 证据 | 来源 |
|---|---|
| ASIAIR 被描述为基于**修改过的 INDI server**，另借用 astronomy.net 与 Siril 的部分代码 | Stargazers Lounge / webastro 论坛 |
| INDI 是 XML 协议，驱动为 C++（`lx200driver.cpp`、`lx200basic`）；存在 ZWO 的 INDI fork | Cloudy Nights「ASIAir LX200 Classic」|
| INDI 用在**赤道仪、单反、其他外设**上，**但不用于 ZWO 自家相机**（走专有 libasi / ASICamera2 SDK） | webastro 论坛 |
| 正是 INDI + gphoto2 给了 ASIAIR 广泛的赤道仪 / 单反兼容性，但被「上笼头」以独占 ZWO 硬件 | 同上 |
| ZWO 在压力下只公开了 libindi 与 astrometry.net 两个仓库，社区评为「法院不会认可的 GPL 合规」 | Cloudy Nights，billndotnet 帖 |

**结论**：行为参照系是公开的 INDI / PHD2 / Siril / astrometry.net 源码。
**证据强度：强**（多方独立佐证）。

### 2.3 二进制清单与形态

从 `asiair_armhf.deb` 解包后的实际目录（`ar x` → `unxz data.tar.xz` → `tar -xvf data.tar`）：

```
/home/pi/ASIAIR/
├── bin/     zwoair_imager, zwoair_guider, zwoair_updater,
│            zwoair_daemon.sh, asiair.sh, air_ble, bsa_server,
│            comets.py, planet.py, searchSSIDIndex.py
├── lib/     libyuv.so 等
├── config/
└── model/   nanotrack_backbone_127_pre.rknn
             nanotrack_backbone_255_pre.rknn
             nanotrack_head_pre.rknn
```

| 要点 | 证据 |
|---|---|
| 核心是编译好的原生二进制，设备上无源码 | FFmpeg ticket #10419 |
| `strings zwoair_imager \| grep -i ffmpeg` 暴露内嵌 **FFmpeg 4.2.2-Kodi** | 同上（原厂级证据） |
| 出现 `.rknn` 模型 → 用 Rockchip NPU 跑 nanotrack 目标跟踪 | Cloudy Nights 解包帖 |
| 夹带 Python 脚本（`comets.py` / `planet.py`）→ 部分功能是 Python 胶水 | 同上 |
| **未发现 Go 组件**；已知实现语言为 C / C++（INDI 驱动、RTS2）与 C 风格 API | 多源，**属"未发现"而非"已排除"** |

**证据强度：强**（有真实文件清单 + 原厂级 `strings` 结果）。

### 2.4 通信协议

| 项 | 内容 | 强度 |
|---|---|---|
| 协议 | **明文 JSON / JSON-RPC，无加密**（Bill Nash 逆向确认「no encryption in ASIAIR」） | 强 |
| 端口 | 4040 / 4360 / 4700，分别对应 guider / updater / imager | 中 |
| 攻击面 | 存在**未保护的 JSON-RPC 端点**，可远程以 root 执行命令 → 社区 `run_jailbreak.py` 即基于此 | 强 |
| 修复 | 该 jailbreak 在**固件 1.91 起失效** | 强 |
| 抓包手段 | `tcpflow` 分析 App ↔ 设备流量 | 中 |

### 2.5 App 侧（Android）

`com.zwoasi.asiair`，v3.0.0，约 1.17 GB，Android 9.0+，SHA-256 `e15b9c0b…d767`。

内嵌原生库（供参考）：

- **TUTK P2P 隧道**：`libIOTCAPIs.so`、`libRDTAPIs.so`、`libP2PTunnelAPIs.so`、`libTUTKGlobalAPIs.so`
- **IJKPlayer**：`libijkffmpeg.so`、`libijkplayer.so`、`libijksdl.so`
- **其他**：`libopencv_java4.so`、`libsqlcipher.so`、`libmmkv.so`、`libzwoNova.so`、`libzalgorithm.so`
- 服务：`com.zwoasi.kit.service.ConnectService`、`tv.danmaku.ijk.media.example.services.MediaPlayerService`

**证据强度：强**（来自 APK 清单）。

### 2.6 硬件分代

| 型号 | SoC | 存储 | 提取方式 |
|---|---|---|---|
| ASIAIR Pro | Raspberry Pi 3 / 4 | microSD | 拔卡 `dd` |
| ASIAIR Plus（早期） | Raspberry Pi CM4 | 32 GB eMMC | USB-C + `rpiboot`（reset 键接 `nRPI_BOOT`） |
| ASIAIR Plus 256GB | Rockchip RK3568B2 | 256 GB eMMC，**后期版无 SD 槽** | maskrom + `rkdeveloptool` |
| ASIAIR Mini | Rockchip RV1126 | — | `rkdeveloptool` |

**关键限制**：ZWO **不公开任何固件镜像**。已变砖用户索要完整镜像被拒，只能 RMA。
因此 **Rockchip 机型上，开机第一件事是整盘备份**——那是回到官方固件的唯一退路。

### 2.7 附带发现：GPL 合规争议

- FFmpeg 官方 ticket **#10419**「Use of ffmpeg in the ASIair」——ZWO 分发内嵌 FFmpeg 的二进制却不提供对应源码。
- 至少 7 处 GPL / LGPL 义务未履行，涉及 libindi、FFmpeg 等。
- `ZWODevTeam` GitHub 仓库被社区普遍视为合规挡箭牌，**其中包含的并非 ASIAIR 实际使用的改动**。

**对逆向的意义**：不要指望 ZWO 的公开仓库；以官方 INDI / PHD2 / Siril 的行为为准。

---

## 三、尚未验证 / 待实机确认

- [ ] Rockchip 代是否用 arm64 包（观察到的是 `armhf`，与 Pi 代一致）
- [ ] `zwoair_imager` 的依赖库清单（`readelf -d` 未跑过）
- [ ] 4040 / 4360 / 4700 端口映射的**一手**确认（当前为论坛转述）
- [ ] 具体机型的实际分区表与 eMMC 布局
- [ ] `zwoair_*` 是否静态链接、是否有符号表残留

---

## 四、来源

- [ASIAIR Mini hacking（Cloudy Nights）](https://www.cloudynights.com/forums/topic/907326-some-help-with-asiair-mini-hacking/)
- [Seestar S50 / ASIAIR Jailbreak + SSH — Baksmaling classes3.dex（Cloudy Nights）](https://www.cloudynights.com/forums/topic/900861-seestar-s50asiair-jailbreak-ssh/page/4/?forcePrint=1)
- [Use of ffmpeg in the ASIair — FFmpeg trac #10419](https://trac.ffmpeg.org/ticket/10419)
- [Hacking the ZWO ASIAIR Plus Kernel（joeytroy.com）](https://joeytroy.com/hacking-the-zwo-asiair-plus-kernel/)
- [Development – ASIAIR Plus & Compute Module 4（Cloudy Nights）](https://www.cloudynights.com/forums/topic/828897-development-asiair-plus-compute-module-4/)
- [ASIAir LX200 Classic "ASIAdpater" Open Source Project?（Cloudy Nights）](https://www.cloudynights.com/forums/topic/921037-asiair-lx200-classic-asiadpater-open-source-project/)
- [ASIAIR and opensource software licences（ZWO 官方论坛）](https://bbs.zwoastro.com/d/13240-asiair-and-opensource-software-licences/12)
- [ASIAIR Plus (RK3568) — OpenAstro](https://www.openastro.net/docs/sbc-install/zwo-asiair-plus-rk3568/)
- [ASIAIR Plus (Pi CM4) — OpenAstro](https://www.openastro.net/docs/sbc-install/zwo-asiair-plus-cm4/)
- [Bricked ASIAIR Plus（ZWO 官方论坛）](https://bbs.zwoastro.com/d/15835-bricked-asiair-plus)
- [ZWO ASIAIR — APKPure（com.zwoasi.asiair）](https://m.apkpure.com/tw/asiair/com.zwoasi.asiair)
