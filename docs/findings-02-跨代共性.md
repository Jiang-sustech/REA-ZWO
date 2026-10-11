# 02 · 各代核心功能是否一样？

> 调查日期：2026-10-11。问题：ASIAIR Pro / Plus(CM4) / Plus 256GB / Mini 是否共用同一套核心？

## 结论：要分两层看，答案正好相反

| 层 | 是否跨代共用 | 说明 |
|---|---|---|
| **应用层**（ZWO 自己的软件栈） | ✅ **全型号共用，单一架构 armhf** | `asiair_armhf.deb` + `indi-y_armhf.deb` + libgphoto2，一套包喂所有机型 |
| **OS 层**（bootloader / kernel / 根文件系统 / 驱动） | ❌ **按 SoC 分叉** | Pi 系（Pro、AA+ CM4）与 Rockchip 系（Mini、AA+ 256GB）是两套完全不同的底座 |

**而且不止跨代——连 Seestar 都共用同一套。** ZWO 的 Larry Ji 在 NEAF 当面确认 Seestar App 「就是同一个软件」；社区那个 `run_jailbreak.py` 一把工具同时打通 ASIAIR 和 Seestar，走的是同一套 OTA 机制。

### 实际含义

**逆向目标锁定 `asiair_armhf.deb` 就对了，跟你手上是哪台无关。**
之前说的「型号决定路线」，指的是**怎么把固件取出来**（拆卡 / rpiboot / maskrom），**不是指分析对象**。分析对象是同一个。

所以最优路径变了：**先用零风险的拆包路线拿下应用层，把 `zwoair_imager` 啃下来**。型号只决定你要不要额外去碰硬件拿 OS 层。

---

## 证据

### 1. 固件更新 = 一组 Debian 包，全是 armhf

ZWO 官方论坛《ASIAIR and opensource software licences》帖中列出的实际包名：

```
asiair_armhf.deb                  ← ZWO 自己的应用层
indi-y_armhf.deb                  ← ZWO 魔改版 INDI
libgphoto2-6_2.5.27-1_armhf.deb   ← 开源，未改
libgphoto2-port12_2.5.27-1_armhf.deb
```

帖中同时指出：除上述 deb 外，还静态编入了 **Astrometry.net、PHD2、indi core、indi 3rdparty** 的代码（GPL3 / LGPL2.1 / BSD3）。

**证据强度：强**（原文包名）。

### 2. 更新包内部目录按架构分，只观察到 armhf

FFmpeg ticket #10419 给出的解包路径：

```
~/Downloads/ASIAIR_V2.1/assets/patch/deb/asiair_armhf/home/pi/ASIAIR/bin
```

更新包 `ASIAIR_V2.1` → `assets/patch/deb/<arch>/...`。**只观察到 armhf 一个架构目录。**

`/home/pi/` 这个路径本身就是 Pi 时代的遗留——说明 ZWO 迁到 Rockchip 后**沿用了同一套目录约定**，没有为 Rockchip 重写布局。

**证据强度：强**（原文路径）。「单一 armhf 架构」为**推断**，见待验证项。

### 3. 一个版本号覆盖所有机型

- v2.4.4 发布说明条目：*"Added guide loss beep alert (**ASIAIR Plus/Mini/PRO**)"* —— 一条更新说明同时点名三个型号。
- ZWO 论坛「Unannounced ASIAIR v2.4.4」帖中，**Mini 用户和 Plus 用户都在讨论同一个 v2.4.4**。
- 官方口径：*"The App on your device and the firmware on the ASIAIR are a **matched set**."* —— App 与设备固件成套更新，同一个 App 版本推给不同硬件。

**证据强度：强。**

### 4. 跨产品线共用

| 证据 | 内容 |
|---|---|
| ZWO 代表确认 | Seestar App 与 ASIAIR App「是同一个软件」 |
| 同一把 jailbreak | `run_jailbreak.py` 同时打 ASIAIR 与 Seestar，走同一 OTA 机制 |
| 协议同源 | JSON-RPC；ASIAIR 端口 4040 / 4360 / 4700，Seestar 为 UDP 4720 初始化 + TCP 4700 JSON-RPC 2.0 |
| 共用演进 | 新固件加了 `get_verify_str` / `verify_client` 挑战应答（Seestar firmware 7.18+），说明两边的协议在同步加固 |

**证据强度：中到强**（多方独立，但无直接代码比对）。

### 5. 但 OS 层确实分叉

- Pi 系：Raspberry Pi OS 血统，自编 kernel 4.19.x，ZWO 改过 `.elf` 启动文件。
- Rockchip 系：Rockchip BSP，且 `model/` 目录里是 `.rknn` 模型（**只有 Rockchip NPU 能跑**，Pi 没有 NPU）。
- 同一份 deb 里同时出现 `/home/pi/`（Pi 遗留）和 `.rknn`（Rockchip 专属）——**这正是"应用层共用、OS 层分叉"的直接物证**：包里带着 Rockchip 的模型文件，却装在 Pi 式布局里。

**证据强度：强。**

---

## 待验证

- [ ] 「单一 armhf 架构」是否成立：RK3568 是 Cortex-A55（ARMv8），**理论上跑 AArch32 用户态是可行的**，这能解释为什么没有 arm64 包——但**无直接证据**。验证方法：比对两台不同代设备的 `asiair_armhf.deb` 哈希与文件清单。
- [ ] 构建号 `firmware 1208/4170`（同一 App 版本下出现两个固件编号）是否对应不同机型 —— 这是**唯一指向"存在按机型区分的固件"的线索**，值得优先查清。
- [ ] `zwoair_imager` 在不同代设备上是否二进制一致（对比 `sha256`）。
- [ ] OTA 服务器是否按机型返回不同的 deb 集合。

---

## 来源

- [ASIAIR and opensource software licences（ZWO 官方论坛）](https://bbs.zwoastro.com/d/13240-asiair-and-opensource-software-licences/12) — **包名清单出自此帖**
- [Use of ffmpeg in the ASIair — FFmpeg trac #10419](https://trac.ffmpeg.org/ticket/10419) — 解包路径
- [Unannounced ASIAIR v2.4.4（ZWO 官方论坛）](https://bbs.zwoastro.com/d/25293-unannounced-asiair-v244/22)
- [Seestar S50 / ASIAIR Jailbreak + SSH（Cloudy Nights）](https://www.cloudynights.com/forums/topic/900861-seestar-s50asiair-jailbreak-ssh/)
- [Seestar-MCP — 协议逆向](https://taco-ops.github.io/taco-ops/projects/seestar-mcp/)
- [ZWO Seestar S50（Cloudy Nights）](https://www.cloudynights.com/forums/topic/880365-zwo-seestar-s50/) — ZWO 代表确认同源
