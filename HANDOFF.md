# HANDOFF — 这个项目后续需要一台真机

> 本仓库由一位**没有 ASIAIR 设备**的研究者发起（2026-10-11）。
> 调研阶段能在纯静态条件下做的部分已经做完，见 `docs/`。
> **再往下推进，需要有人手上有实机。** 这份文档就是给那个人的。

---

## 一、现在到哪了

已完成（纯静态，无需设备）：

| 文档 | 内容 |
|---|---|
| `docs/findings-01-架构底细.md` | ASIAIR 技术栈逐条取证（OS / 应用层 / 协议 / 硬件分代） |
| `docs/findings-02-跨代共性.md` | 应用层全型号共用、OS 层按 SoC 分叉 |
| `docs/prior-art-前人工作综述.md` | 社区已做工作的全量梳理（**协议层已有文档**） |
| `docs/sources-固件获取渠道.md` | 官方镜像下载方式 + 本机工具链 |

**核心结论**：ASIAIR = 魔改 Raspbian + 魔改 INDI + 闭源 ZWO 库。协议是明文 JSON-RPC，端口拓扑与信封格式**已被社区完整逆向**（见 prior-art 文档）。

---

## 二、需要一台真机才能做的事（按优先级）

### P0 — 卡住整个项目的一步

- [ ] **取 `asiair_armhf.deb`**。这是应用层更新包，只走 App 的 OTA 推送，**没有任何公开镜像**。
  - 路径一：从设备更新缓存里取（`/home/pi/ASIAIR/` 附近）
  - 路径二：App 更新时抓 OTA 流量，拿到下载 URL
  - 路径三：APK（`com.zwoasi.asiair`）里挖 OTA 服务器地址
- [ ] **确认自己的机型**（Pro / Plus CM4 / Plus 256GB / Mini）。这决定后续走哪条路。

### P1 — 补充协议文档的空白

`asiair-dashboard/docs/PROTOCOL.md` 明确列了"未文档化"的部分：

- [ ] **写/动作命令**：`set_control_value`、开始/停止曝光、goto/slew、导星启停、解析板、跑序列
- [ ] **端口 4800 的二进制帧格式**（已知 header 有 `03 c3` magic，内嵌 JPEG `ff d8 ff`）
- [ ] **四个侧通道**：4400 / 4500 / 4801 / 4900
- [ ] **事件推送**的完整清单（目前只有 Exposure / SaveImage / Sequence 三类）

### P2 — 验证与取证

- [ ] **验证跨代二进制是否一致**：对比两台不同代设备的 deb 哈希与文件清单（`findings-02` 的核心推断）
- [ ] **验证 4350 洞在最新固件是否仍在**（社区文档实测的是 42.97 / seestar-3.2；已知 ZWO 改过端口号 4361→4360）
- [ ] **新固件的认证机制**：Seestar 7.18+ 出现 `get_verify_str` / `verify_client` 挑战应答，ASIAIR 侧是否同步？
- [ ] **许可证取证**：对照公开的 INDI / PHD2 / Siril / FFmpeg 源码，坐实 GPL/LGPL 侵权（Bill Nash 的诉求）

### P3 — 硬件与 OS 层（仅 Rockchip 机型需要）

- [ ] **开机第一件事：整盘备份 eMMC**。ZWO 不提供官方镜像，这是回到原厂的唯一退路
- [ ] 提取分区表与分区布局
- [ ] 验证 `findings-02` 里"包内同时有 `/home/pi/` 与 `.rknn`"的物证

---

## 三、不需要设备也能继续做的

- 用官方 OS 镜像做静态摸底（拆 `.deb` / `.img`，`readelf` / `strings` 二进制）
- 读 `smart-underworld/seestar_alp`（296★，Python 参考实现）补全协议理解
- 对照公开 INDI 源码做许可证取证
- 完善本仓库的文档

---

## 四、工具与渠道

**本机已备好的工具链**（Windows 原生，不需要 WSL2）见 `docs/sources-固件获取渠道.md`。
要点：7-Zip 26.02 读 ext4 / `.deb`；`arm-none-eabi-*` binutils 在 `D:\STM32Toolchain\bin`。

**社区渠道**（找人、找人问）：

- [Cloudy Nights — ASIAIR Jailbreak 帖](https://www.cloudynights.com/forums/topic/900861-seestar-s50asiair-jailbreak-ssh/)
- [indilib 论坛 — ASIAIR 开源许可证讨论](https://www.indilib.org/forum/development/10380-asiair-and-opensource-software-licences.html)
- [StefanDorresteijn/asiair-dashboard](https://github.com/StefanDorresteijn/asiair-dashboard) — 有 ASIAIR Plus，已逆向协议
- [basraayman/zwo-asiair-jailbreak](https://github.com/basraayman/zwo-asiair-jailbreak) — 有设备，做过安全研究
- [smart-underworld/seestar_alp](https://github.com/smart-underworld/seestar_alp) — 最活跃的生态

---

## 五、边界（请接手方也遵守）

**本仓库不存放 ZWO 原厂固件、镜像或提取出的二进制。** `.gitignore` 已按此设置。

- **逆向分析 ≠ 再分发。** 前者是研究，后者是明确的版权问题。
- 想拿镜像，请**自己**从 `docs/sources-固件获取渠道.md` 里的官方链接下载——那是 ZWO 自己公开的。
- 涉及设备操作（挂载 eMMC、maskrom 刷写、jailbreak）**有变砖风险**，先做整盘备份。

---

## 六、怎么接手

任选其一（仓库当前 owner 是 `Jiang-sustech`）：

1. **加协作者**（推荐起步）—— 向 owner 要 write 权限，双方都保留能力
2. **Transfer ownership** —— 完全移交，适合 owner 确定不再参与
3. **Fork + PR** —— 最轻量，继续以 PR 方式贡献

有问题请开 issue。
