# 前人工作综述（开源社区对 ZWO 的研究）

> 调查日期：2026-10-11。方法：GitHub API 检索 + 直接抓取各仓库 README/文档。
> **结论：协议层已被基本完整逆向，安全洞已公开，OS 层结构已知。反编译的定位需要重新想。**

---

## 零、最重要的结论

**前人已经做到的程度，远超"起步阶段"：**

| 层面 | 状态 |
|---|---|
| 网络协议 | ✅ 端口拓扑、信封格式、方法命名、方法清单**都有文档** |
| 安全 | ✅ **预认证 root 等价漏洞已公开**（有 advisory 级文档） |
| OS 层 | ✅ 只读分区、apt 源、Samba、SSH 都已知 |
| 配件协议 | ✅ EAF 电调可被完整模拟，已有可用实现 |
| 跨产品共用 | ✅ 被**独立证实**（见下文 C 节） |

**所以：如果目标只是"理解核心功能做什么"，不需要反编译——读文档和开源实现就够了。**

---

## A. 协议层：`StefanDorresteijn/asiair-dashboard`

MIT 许可，Node.js 写的 ASIAIR Plus 监控面板。**关键价值是 `docs/PROTOCOL.md`**——作者把抓包逆向的结果完整写出来了。

### 服务拓扑（比我们之前掌握的准）

App 会开**六条并行 TCP 连接**：

| 端口 | 作用 |
|---|---|
| **4700** | **主 JSON-RPC 控制 + 事件推送** |
| **4800** | 图像/帧数据流（二进制；每张 sub 一帧） |
| 4801 | 与 4800 配对，图像元数据/控制（未文档化） |
| 4900 / 4500 / 4400 | 侧通道（未文档化） |

另开：**22（SSH，Raspbian）**、445（Samba / 图像存储）、4350 / 4360（OTA）

> **注意**：我们之前从论坛拿到的"4040/4360/4700"是不准的。以这份为准。

### 传输格式

**4700 上是换行分隔的 JSON**——每条消息一个 JSON 对象，`\r\n` 结尾，**无长度前缀**。用整数 `id` 关联请求/响应。每条连接周期性发 `test_connection` 保活。

```json
// 请求
{"id": 2676, "method": "get_control_value", "params": ["CoolerOn", true]}

// 响应
{"jsonrpc": "2.0", "Timestamp": "5202.638790431", "method": "get_control_value",
 "result": {"name": "CoolerOn", "type": "number", "value": 1}, "code": 0, "id": 2676}
```

`params` 可为位置参数数组，或单元素数组包一个对象（`[{"guide_camera_name": "..."}]`）。

### 方法命名约定

`get_*` / `set_*` 设置 · `scope_*` 赤道仪 · `pi_*` 树莓派主机 · `get_img_*` 文件浏览

已观察方法（部分）：`get_connected_cameras`、`get_camera_info`、`get_camera_state`、`get_controls`、`get_control_value`、`get_app_state`、`set_app_setting`、`test_connection`、`get_target_sequences`、`get_power_supply`、`get_image_save_path`、`get_disk_volume`、`scope_get_info`、`scope_get_cap`、`get_mount_list`、`mount_scan_port`、`get_focuser_setting`、`get_dither`、`pi_get_info`、`pi_eth0_state`、`pi_station_state`

**未文档化**：写/动作命令（`set_control_value`、开始曝光、goto、导星启停、解析板、跑序列）、4800 二进制流、四个侧通道。

### 图像存储

SMB（445）**guest 只读**，共享 `EMMC Images` / `TF Images` / `Udisk Images`。帧为标准 **16-bit FITS**（`BITPIX=16`, `BZERO=32768`, `BAYERPAT`），落盘路径 `/boot/Image/Autorun/<Type>/`。

---

## B. 安全研究：`basraayman/zwo-asiair-jailbreak`

**advisory 级别**的文档，技术密度最高的一份。

- **漏洞**：`zwoair_updater` 的 `begin_recv` JSON-RPC 方法（TCP/**4350**）接收 `file_name`，未转义就拼进 `/bin/sh`。**唯一过滤是字面 `/`**，`$(...)` 和其他 shell 元字符全部通过，在 md5 与 RSA-SHA1 签名校验**之前**就被 shell 求值。
- **后果**：以 `pi` 执行，而 `pi` 在出厂固件上**有无密码 sudo** → **预认证、root 等价、来自局域网任意主机**。
- **端口**：命令 4350，文件传输 **4360**（ASIAIR / ASI2600MC Air）或 **4361**（Seestar S50）。**ZWO 唯一的修补就是把 4361 改成 4360**，注入点本身没动。
- **实测固件**：ASI2600MC Air `42.97`（`version_int=4297`）、Seestar S50 `seestar-3.2`（`version_int=2775`）

> 对照：2024 年那个 `run_jailbreak.py` 打的是同一套协议。**固件 1.91 封的是别的东西**，这个洞在 4350 上一直开着。

---

## C. 跨产品共用 —— 被独立证实

上述安全文档里有一条直接证据：

> **`zwoair_updater` 是三个产品线（Seestar 全家、ASIAIR 全家、内置 ASIAIR 的 ASI 相机）共用的同一个二进制**，"dispatches per-model behaviour off a **single model-string switch**"。

从固件里提取出的 model string：

```
ZWO SeeStar Board              ← 在售的 S50
ZWO SeeStar S30 Board
ZWO SeeStar S30P Board
ZWO SeeStar S50P Board         ← 未发布
ZWO SeeStar S50v2 Board        ← 内部代号，无公开 SKU
ZWO ASI2600MC Air Board
```

并且："field reports 表明 ASIAIR Mini / Plus / Pro **共用同样的 daemon 二进制**（同样的 RPC 方法名、同样的 JSON-RPC banner）"

**这独立佐证了 `docs/findings-02-跨代共性.md` 的"应用层共用"结论**，而且从"共用"升级为"同一个二进制 + 模型串分派"。

---

## D. 越狱工具链

### `billndotnet/asiair-modkit` —— Bill Nash 的工具集

不是越狱本身，是**越狱后的标准管理工具**。README 里全是 OS 层硬事实：

| 脚本 | 揭示的信息 |
|---|---|
| `mount-rw.sh` / `mount-ro.sh` | **核心 OS 分区出厂是只读挂载**，可重挂为读写 |
| `enable-ssh-auth.sh` | SSH 服务在，但默认禁密码/密钥认证 |
| `install-sftp-server.sh` | SFTP 默认没装 |
| `setup-package-install.sh` | 换 `resolv.conf` + `sources.list`——**设备默认用中国 apt 镜像**，作者换成美国源降延迟 |
| `tools/usbreset.py` | 枚举/重置 USB 设备 |

`DMCA-README`（名字本身说明问题）列出的动机：

1. **恢复互操作性**——ZWO 的魔改 INDI 服务器**故意屏蔽非 ZWO 配件驱动**，与 INDI 的 LGPLv2 及开放驱动结构相抵触
2. **换硬件**——有"校验阻止 CPU 更换"（restrict CPU swaps）
3. **安全**——Samba 4.9.5 过旧，需自行升级
4. 提到 indilib 官方论坛的讨论帖作为依据

### 其他

- `Oxofrimbl/asiair` —— 走 RPC 的越狱
- `open-astro/rk-flashtool` —— Rockchip USB 刷机（含 `jailbreak/` 目录），支持 ASIAIR Plus RK3568

---

## E. 配件协议（不用碰固件就能知道怎么模拟）

**`gmorain/ZWO-EAF-Proxy`** —— 让非 ZWO 电调在 ASIAIR 眼里表现为真 EAF。

- 做法：单片机在 ASIAIR 的 **USB 总线上枚举时冒用 EAF 的身份和协议**，翻译命令后驱动真电调。必须是 USB **device**（原生的 USB 外设控制器，不能用 USB-serial 桥）
- ESP32-S3 已验证（真 ASIAIR 接受并驱动了它）；RP2040 计划中
- **关键细节**：ASIAIR 用**电调温度做自动对焦补偿**，所以代理必须提供一个真实的温度探头
- 不支持的第三方配件（Moonlite、myFocuserPro2）正是被 ZWO 的闭源 SDK 挡住的

赤道仪侧：INDI 的 `lx200driver.cpp` / `lx200basic`（命令如 `:Mg` / `:RG` / `:Q`，# 结尾），社区做过脉冲导星兼容。

---

## F. Seestar 侧（同平台，生态更大）

| 项目 | 星 | 内容 |
|---|---|---|
| `smart-underworld/seestar_alp` | **296★** | Python，完整控制/自动化，暴露 **ASCOM Alpaca 服务器**（端口 5555）。含 `emulator/` 和 `docs/superpowers`（firmware param-shape 研究），甚至做了**序列号/license 字段脱敏** |
| `smart-underworld/seestar_run` | 65★ | 同组织的精简版 |
| `Arseneceefe/S50lib` | 18★ | Python 封装 |
| `bguthro/seestar-tool` | 14★ | **固件管理**桌面程序 |
| `astronomyk/seestarpy` | 13★ | Python |
| `apolloscharge/seestar-s50` | 8★ | 替换 img |
| `scopinator_seestar`（Rust crate） | — | **固件 7.18+ 的挑战应答认证**（`get_verify_str` / `verify_client`） |
| `seestar-mcp` | — | MCP 服务器，含 raw firmware passthrough |

**`seestar_alp` 是最有阅读价值的**——Python 写的完整协议实现，可以当"参考实现"对照。

---

## G. 对本题的重新评估

### 反编译的定位变了

既然**行为层已被文档化**，静态逆向的价值收窄到：

1. **具体算法**——对焦、导星、叠加、解析板、跟踪（`.rknn` nanotrack）的内部实现
2. **未文档化部分**——写命令、4800 二进制流的帧格式、四个侧通道
3. **许可证取证**——GPL/LGPL 侵权的具体证据（Bill Nash 的诉求）
4. **更深的漏洞**——4350 那个洞之外还有什么

### 更省事的路（如果有设备）

有预认证 root 洞 + 协议已公开 → **根本不用静态逆向**，直接跑起来观察即可。但**本项目无设备**，静态分析仍是唯一路径。

### 仍然值得做的事

- 用 `seestar_alp` 当参考实现，**对照**我们从镜像里提取的二进制
- 直接调用协议（把 ASIAIR 的端口当接口用）——不需要理解固件内部

---

## 来源

- [StefanDorresteijn/asiair-dashboard](https://github.com/StefanDorresteijn/asiair-dashboard) — `docs/PROTOCOL.md`
- [basraayman/zwo-asiair-jailbreak](https://github.com/basraayman/zwo-asiair-jailbreak)
- [billndotnet/asiair-modkit](https://github.com/billndotnet/asiair-modkit)
- [open-astro/rk-flashtool](https://github.com/open-astro/rk-flashtool)
- [gmorain/ZWO-EAF-Proxy](https://github.com/gmorain/ZWO-EAF-Proxy)
- [smart-underworld/seestar_alp](https://github.com/smart-underworld/seestar_alp)
- [indilib 论坛：ASIAIR and opensource software licences](https://www.indilib.org/forum/development/10380-asiair-and-opensource-software-licences.html)
- [Cloudy Nights: Seestar S50/ASIAIR Jailbreak + SSH](https://www.cloudynights.com/forums/topic/900861-seestar-s50asiair-jailbreak-ssh/)
