# REA-ZWO

用 [REA](https://github.com/morluto/rea)（Reverse Engineer Anything — 给 AI agent 接上 Ghidra / IDA / Hopper 的 MCP 工具链）来拆 **ZWO ASIAIR 天文盒子**的固件。

本仓库只存放**调研结论、脚本、笔记和抓包**。
**不存放 ZWO 的原厂固件、镜像或提取出的二进制**——原因见文末《法律边界》。

---

## 已知结论

> 截至 2026-10-11 的调查结果。逐条证据见 [`docs/findings-01-架构底细.md`](docs/findings-01-架构底细.md)。

ASIAIR **不是自研系统**，是三层拼装：

| 层 | 实际是什么 |
|---|---|
| 应用层 | **魔改 INDI**（C++，XML 协议）为核心。赤道仪走 INDI，单反走 gphoto2，ZWO 自家相机 / 电调 EAF / 滤镜轮 EFW 走**闭源 SDK**（`libASICamera2.so`、libasi） |
| 二进制 | 设备 `/home/pi/ASIAIR/bin/`：`zwoair_imager`、`zwoair_guider`、`zwoair_updater`、`zwoair_daemon.sh`。C / C++，**未见 Go 痕迹** |
| 内嵌开源 | FFmpeg **4.2.2-Kodi**、libyuv、OpenCV、IJKPlayer、SQLCipher / MMKV、TUTK P2P、RKNN 模型（nanotrack 目标跟踪） |
| OS | 魔改 **Raspberry Pi OS / Raspbian**（Debian，32-bit armhf），自编 kernel 4.19.x |
| 通信 | **明文 JSON-RPC，无加密**，端口 4040 / 4360 / 4700 |
| 手机 App | Android 原生 `com.zwoasi.asiair`，v3.0.0，约 1.17 GB |

### 硬件分三代，逆向路线完全不同

| 型号 | 取固件方式 | 风险 |
|---|---|---|
| ASIAIR **Pro** | 拔 SD 卡 → `dd` | 最低 |
| ASIAIR **Plus**（CM4，早期 32 GB） | USB-C + `rpiboot` 挂 eMMC（reset 键接 `nRPI_BOOT`） | 低 |
| ASIAIR **Plus 256GB** / **Mini**（Rockchip RK3568B2 / RV1126） | maskrom + `rkdeveloptool` | **高**——ZWO 不提供任何官方镜像，进不了 maskrom 就是砖 |

### 各代核心功能是否一样？—— 应用层共用，OS 层分叉

| 层 | 跨代共用？ | 说明 |
|---|---|---|
| **应用层** | ✅ **全型号共用，单一 armhf** | 一套 `asiair_armhf.deb` + `indi-y_armhf.deb` + libgphoto2 喂所有机型；连 Seestar 都共用 |
| **OS 层** | ❌ 按 SoC 分叉 | Pi 系（Pro、AA+ CM4）与 Rockchip 系（Mini、AA+ 256GB）两套底座 |

**所以逆向目标锁定 `asiair_armhf.deb` 就对了，跟手上是哪台无关。**「型号决定路线」指的是*怎么把固件取出来*，不是分析对象。
逐条证据见 [`docs/findings-02-跨代共性.md`](docs/findings-02-跨代共性.md)。

### 最低风险入口

**先拆 `.deb`，不要先拆机器。** 官方更新包是 Debian 包 `asiair_armhf.deb`：

```bash
ar x asiair_armhf.deb        # → control.tar.xz / data.tar.xz / debian-binary
unxz data.tar.xz
tar -xvf data.tar            # → home/pi/ASIAIR/{bin,lib,config,model}
```

拆出来的 `bin/zwoair_*` 就是主目标，全程不碰硬件、零变砖风险。
包名 `armhf` = 32 位 ARM，本身即印证了它照 Raspberry Pi OS（armhf）构建。

---

## 路线图

- [ ] 搭环境：WSL2 + Debian，装 `binwalk` / `file` / `readelf`
- [ ] 拆 `asiair_armhf.deb`，对 `zwoair_imager` 做静态摸底（架构、依赖、符号、硬编码协议）
- [ ] 确定设备具体型号，决定是否/如何做整盘备份
- [ ] 接 REA + Ghidra，反编译 `zwoair_imager` / `zwoair_guider`
- [ ] 梳理 App ↔ 设备 的 JSON-RPC 协议（4040 / 4360 / 4700）

---

## 环境备忘

- 本机 **Windows 11**，`node v24.15.0`，npm registry 可达
- **`github.com` / `raw.githubusercontent.com` 从本机直连被重置**，`api.github.com` 正常。下载 Ghidra、OpenAstro 等 GitHub 资源需要代理
- REA 在 Windows 上只有**实验性** Ghidra 支持（只读、仅原生 x86-64 PE），而目标是 **ARM Linux 二进制**——正经分析走 WSL2

---

## 法律边界

- **逆向工程本身**：为互操作性做逆向，在多数量辖区属合法范畴；但可能违反 ZWO 的最终用户许可协议（EULA）。
- **这个仓库的自我约束**：**不提交原厂固件、镜像、提取出的二进制或反编译产物。** 逆向分析和**再分发**是两回事——后者是明确的版权问题。`.gitignore` 已按此设置了排除规则。
- 涉及设备操作（挂载 eMMC、maskrom 刷写）**有变砖风险**，且先做整盘备份是回到官方固件的唯一退路。

---

## 相关

- [REA](https://github.com/morluto/rea) — 本项目的工具链
- [OpenAstro](https://www.openastro.net/) — 社区在 ASIAIR 硬件上跑 Debian 的项目
