# 固件获取渠道（实测）

> 调查日期：2026-10-11。本文件记录**实际可用的下载路径**，以及哪些路走不通。

## 结论速查

| 想要的东西 | 公开可下载？ | 怎么拿 |
|---|---|---|
| **OS 整盘镜像**（含完整 rootfs + 应用层） | ✅ **官方公开** | ZWO 官方恢复手册 PDF 里的 Google Drive 链接 |
| **`asiair_armhf.deb`**（应用层更新包） | ❌ **没有公开镜像** | 只走 App 的 OTA 推送；要从设备取或抓 OTA 流量 |

**关键认知**：**OS 镜像里包含应用层**（`/home/pi/ASIAIR/bin/` 全套二进制）。所以拿 OS 镜像是**一步到位**的方案，比 deb 还全。
而且**全程只读**——不写卡、不写设备，**零变砖风险**。

---

## 一、官方 OS 镜像（Google Drive）

链接藏在 ZWO 官方文档 [How to Restore ASIAIR OS](https://astronomy-imaging-camera.com/manuals/How_to_Restore_ASIAIR_OS.pdf) 里（PDF 内嵌的 Drive 链接，网页上不直接暴露）。

| 文件 | 大小 | 来源 | Drive ID |
|---|---|---|---|
| `ASIAIR_OS_V1.2.zip` | **929 MB** | ZWO 官方 PDF | `1eS91KKysWFHW-BFIqYnPARNijL4PvTec` |
| `ASIAIR_PRO_OS_V1.4.zip` | **6.3 GB** | ZWO 官方 PDF | `19VpbElWxGSCx_aB56cdLEPP-jaJM8D1N` |
| `new_airpro.zip` | 5.5 GB | 第三方转链（manuals.plus） | `1fvWM1lenwQP29G9_Q_XCMKhm0_IB2ULH` |
| ~~（一个失效链接）~~ | — | — | `1MLd0TO7yzPisqlzVpUVn743DaEnKzox9` → Page Not Found |

**适用机型**：这两个官方镜像是 **SD 卡机型**（原版 ASIAIR = Pi 3B，ASIAIR Pro = Pi 4B）。
**ASIAIR Plus 及以后（eMMC 机型）没有官方公开镜像**——只能靠自己先备份。

### 下载方法（大文件必须走确认页）

Drive 对大文件会插一个"病毒扫描"页。两步：

```bash
ID=1eS91KKysWFHW-BFIqYnPARNijL4PvTec

# 1. 取 uuid
uuid=$(curl -sSL "https://drive.google.com/uc?export=download&id=$ID" \
       | grep -oE 'name="uuid" value="[^"]*"' | sed 's/.*value="//;s/"//')

# 2. 带 confirm=t 和 uuid 下载（注意是 drive.usercontent.google.com）
curl -L -o ASIAIR_OS_V1.2.zip \
  "https://drive.usercontent.google.com/download?id=$ID&export=download&confirm=t&uuid=$uuid"
```

**找大小的方法**（决定下不下之前）：抓确认页的正文，里面有 `文件名 (N.NG)` 字样。

---

## 二、`asiair_armhf.deb` 为什么拿不到

- 它是**设备端的更新包**，由手机 App 通过 OTA 推给设备，**ZWO 不提供下载页**。
- 已确认：**没有公开镜像**、没有 GitHub 转载、没有社区搬运。
- 设备上它存在于更新缓存里（`/home/pi/ASIAIR/` 或 OTA 暂存区）。

**三条可行路径**（都要设备在手）：

1. **从设备取**——需要设备访问权限。社区有 `run_jailbreak.py` 走未保护的 OTA socket 拿 root SSH（**固件 1.91 起已封堵**）。
2. **抓 OTA 流量**——App 更新时抓包，拿到下载 URL 和包体。
3. **从 App 里挖 URL**——APK（`com.zwoasi.asiair`，约 1.17 GB）里含 OTA 服务器地址。

---

## 三、本机网络注意事项

- **`drive.google.com`、`github.com`、`raw.githubusercontent.com` 直连都不通**。
- **本机有可用代理：`127.0.0.1:10090`**（系统级 `ProxyEnable=0` 没启用，但端口是活的）。
  用法：`curl -x http://127.0.0.1:10090 ...`
- 打开 VPN 后部分站点可直连，但 Drive 大文件仍建议走代理更稳。
- **C 盘只剩约 7.7 GB**，所有镜像一律放 D 盘。本项目放 `firmware/`（已被 `.gitignore` 排除）。

---

## 四、拿到镜像之后

镜像是个 SD 卡整盘 `.img`，分区：
- **BOOT 分区**：FAT，Windows/macOS 可直接读。含 `zwoair_license`（与芯片 ID 绑定的许可证，**分析时无用，也别弄丢**）和 `ASIAIR_Config.txt`。
- **其余分区**：ext4，含完整 rootfs，**`/home/pi/ASIAIR/bin/zwoair_*` 在这里**。

在 WSL2 里：`unzip` → `fdisk -l` 看分区偏移 → `mount -o loop,offset=...` 挂载目标分区 → 取二进制。

**全程只读，不碰任何设备。**
