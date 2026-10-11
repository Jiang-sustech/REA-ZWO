# 03 · 镜像解剖：从 30 GiB 到 1.8 MB

> 2026-10-11。素材：官方 `ASIAIR_OS_V1.2.zip`（929 MiB）→ `ASIAIR_OS_V1.2.img`（29.8 GiB）。
> **本文所有结论都来自实机镜像，不是论坛转述。**

---

## 一、镜像布局（来自 `/etc/fstab`，一手）

```
PARTUUID=b52378d8-01  /boot      vfat  defaults,uid=1000,gid=1000  0 2
PARTUUID=b52378d8-02  /          ext4  defaults,noatime,ro          0 1   ← 只读
PARTUUID=b52378d8-03  /home/pi   ext4  defaults,noatime,rw          0 1   ← 可写
tmpfs /var/log tmpfs nodev,nosuid 0 0
tmpfs /var/tmp tmpfs nodev,nosuid 0 0
tmpfs /tmp     tmpfs nodev,nosuid 0 0
```

**纯 MBR，无 GPT**，4 个分区：

| 分区 | 类型 | 卷标 | 字节偏移 | 大小 | 挂载点 |
|---|---|---|---|---|---|
| 1 | `0x0c` FAT32 | **BOOT** | 1,048,576 | 25,599 MiB | `/boot` |
| 2 | `0x83` ext4 | — | 27,799,846,912 | 2,938 MiB | `/`（**ro**） |
| 3 | `0x83` ext4 | — | 30,880,563,200 | 214 MiB | **`/home/pi`**（rw） |
| 4 | `0x82` swap | — | 26,844,594,176 | 911 MiB | swap |

### 三个直接推论

1. **根分区出厂只读挂载**（`ro`）——**独立证实了 `billndotnet/asiair-modkit` 里的 `mount-rw.sh`/`mount-ro.sh`**。这是设计上的防损措施（防意外断电损坏）。
2. **应用装在 `/home/pi` 这个可写分区上**（214 MiB），不在根分区里。**所以 `/home/pi/ASIAIR/` 在根文件系统里"查无此目录"是正常的**——一开始差点误判。
3. **BOOT 分区被撑到 25 GiB**——Raspbian 的 `/boot` 通常只有 256 MB。因为 `config` 里写着 `image_path=/boot/Image/`，**拍摄的图像全存在 BOOT 分区**。

> `asiair.sh` 里有 `sudo swapon /dev/mmcblk0p4` —— **`mmcblk0` 说明这是 SD 卡机型**（Pi 3B/4），不是 eMMC。

---

## 二、`/home/pi/ASIAIR/` 清单（实际提取，与前人描述逐一吻合）

```
asiair.sh                        764 B    ← rc.local 调用的入口
config                           366 B    ← 主配置（版本、路径）
bin/zwoair_imager          1,791,568 B    ← 主程序
bin/zwoair_guider          1,168,320 B    ← 导星
bin/zwoair_updater           261,892 B    ← OTA 更新器
bin/zwoair_daemon.sh           2,077 B    ← 守护/看门狗
bin/start_INDI.sh                515 B    ← 启动 indiserver
bin/zwoair_ap_led.sh           4,256 B
bin/run_update_pack.sh           330 B
bin/polar_align.py             6,127 B    ← 极轴校准（Python）
bin/get_ap_ssid.sh / set_timezone.sh / auto_shutdown.sh
lib/libASICamera2.so       1,339,855 B    ← ZWO 相机 SDK（闭源）
lib/libEAFFocuser.so          40,915 B    ← 电调 SDK（闭源）
lib/libEFWFilter.so           26,527 B    ← 滤镜轮 SDK（闭源）
```

同分区还找到 **`.indi/` 目录**，内含真实的 INDI 赤道仪配置：

```
.indi/ZEQ25_config.xml                  ← iOptron ZEQ25
.indi/EQMod Mount_config.xml            ← EQMod（信达）
.indi/Telescope Simulator_config.xml    ← 模拟器
.indi/ScopeConfig.xml / ParkData.xml
```

**这是"赤道仪控制走 INDI"的直接物证**——不是推断。

---

## 三、二进制侦查（一手）

### 架构：确认 armhf

```
bin/zwoair_imager:  ELF 32-bit LSB executable, ARM, EABI5 version 1 (GNU/Linux),
                    dynamically linked, interpreter /lib/ld-linux-armhf.so.3,
                    for GNU/Linux 3.2.0, BuildID[sha1]=2783f710..., stripped
```

```sh
arm-none-eabi-readelf -h 显示：
  Class:   ELF32
  Machine: ARM
  Flags:   0x5000400, Version5 EABI, hard-float ABI    ← armhf，实锤
```

**三个主程序全部 `stripped`；三个 `.so` 库都 `not stripped`**（有符号，是意外收获）。

### 启动链

```
/etc/rc.local
  └─ sudo -u pi /home/pi/ASIAIR/asiair.sh
       ├─ source config                       # 读版本号、路径
       ├─ systemctl restart hostapd / dnsmasq   # WiFi AP
       ├─ /home/pi/bridgeAP
       ├─ $updater_path$updater_exec &        # zwoair_updater
       ├─ $daemon_path$daemon_exec &          # zwoair_daemon.sh（看门狗）
       ├─ samba (nmbd/smbd)                   # 图像共享
       └─ swapon /dev/mmcblk0p4               # 开 swap
```

`zwoair_daemon.sh` 是个**崩溃自愈守护**：每 2 秒检查 guider/imager 存活，挂了就重启；同时用 `vcgencmd` 读 SoC 温度并**检测欠压**（`get_throttled` → 写 `undervolted=yes/no`）。崩溃日志用密码 **`zwoptical`** 打 zip 包。

### 依赖表告诉我们的架构

`zwoair_imager` 的 `DT_NEEDED`：

| 库 | 含义 |
|---|---|
| `libASICamera2` / `libEFWFilter` / `libEAFFocuser` | ZWO 自家三件套闭源 SDK |
| **`libwiringPi.so`** | **树莓派 GPIO 直控** |
| **`libpython2.7.so.1.0`** | **内嵌 Python 2.7 解释器**（所以才有 `.py`） |
| **`libnova-0.16.so.0`** | 天文计算（坐标、历元、恒星时） |
| **`libgphoto2.so.6` + `_port.so.12`** | 单反控制 |
| **`libxml2.so.2`** | **INDI 的 XML 协议** |
| `libopencv_{core,imgproc,highgui}.so.2.4` | OpenCV 2.4 图像处理 |
| `libcfitsio.so.5` | FITS 读写 |
| `libusb-1.0` / `libudev` | USB / 设备枚举 |
| `libexif` / `libjpeg` / `libz` / `libcrypto(1.1)` | 常规 |

`zwoair_guider` 多一个 `libcurl-gnutls.so.4`。

---

## 四、RPC 方法面（从二进制 strings 穷举）

**比抓包完整**——方法名就明文躺在二进制里。

| 命名空间 | V1.2 (2019) 数量 |
|---|---|
| `get_*` | 39 |
| `set_*` | 17 |
| `pi_*` | 12 |
| `start_*` | 2 |
| `stop_*` | 6 |
| **`scope_*`** | **0** |

`zwoair_imager` 73 个 / `zwoair_guider` 73 个 / `zwoair_updater` **仅 6 个**。

全量清单：`docs/assets/rpc-methods-v1.2.txt`

### 二进制校验值（供接手人比对）

```
3ccfc4cb26dc9338bc201dc5c3b125dc0bb2266c15bdb376e31f319e38785d6e  bin/zwoair_imager
707d04301061330facc1920c7c9b3bb8cfe165bfe619aa3cd6a4b1588e4edafb  bin/zwoair_guider
f55d90ccc663420937ce2e4f33916817eb84b23609ba835419902b6d57269cc9  bin/zwoair_updater
14789dd0ca49059475ac1141abadab6247b5a1d01e11c07af952138724137c82  bin/polar_align.py
9ea0e6f584e8525546f0690a3a0137920b421c4866bf90ab4f6a28290068ec52  lib/libASICamera2.so
9cd0de08958d74febabf8cba7f62d3973e74135293561d52b5d19381ab5d38cf  lib/libEAFFocuser.so
bb035d77f6a450cdb7aac2aebdc7086cd7ccfb694dc46c36653ac96b012b2c78  lib/libEFWFilter.so
```

（以上来自 2019 年的 V1.2 镜像；接手人若有实机，可与自己设备的同名文件比对，直接验证「跨代是否同一二进制」）

### 与前人文档的三处对照

| 项 | 前人（现代固件） | 本次（V1.2, 2019） | 说明 |
|---|---|---|---|
| `scope_*` 命名空间 | 有（`scope_get_info` 等） | **完全没有** | **该命名空间是后期引入的**——首次拿到的时间演进证据 |
| 端口 4400 | 列为"未文档化侧通道" | **是 INDI server**（`netstat` 那行） | **前人此条不准确** |
| 加密 | Bill Nash：ASIAIR"无加密" | 存在 **`pi_encrypt` / `pi_is_verified`** | **有出入**——至少存在验证/加密路径，需澄清范围 |

---

## 五、对"要不要上 REA"的更新结论

1. **架构判断正确**：armhf / ARMv7 / EABI5 / hard-float，与 issue #1745 里的预判完全一致。
2. **三个主程序 stripped** → Ghidra 反编译出来**没有函数名**，只有地址。这抬高了纯反编译的成本。
3. **但 `.so` 库没 stripped** → `libASICamera2.so` / `libEAFFocuser.so` / `libEFWFilter.so` **有符号**。ZWO 自家相机/电调/滤镜轮的 API 可以直接从符号读出来。
4. **RPC 方法名明文可穷举** → 行为层面的 API 面**不需要反编译**就能拿到。

**结论：REA + Ghidra 对这个目标的价值进一步下降。**
真要反编译，优先目标是那三个**没 stripped 的 `.so`**，而不是 stripped 的主程序。

---

## 六、待办

- [ ] 补全方法清单（当前只导出 `get/set/scope/pi/start/stop` 前缀，其他命名待扫）
- [ ] 对比 `ASIAIR_PRO_OS_V1.4.zip`（6.3 GB）看 2019 → 2020 的演进
- [ ] 分析三个未 stripped 的 `.so`，读出相机/电调/滤镜轮的完整 API
- [ ] 澄清 `pi_encrypt` / `pi_is_verified` 与"无加密"说法的矛盾
- [ ] 校验三个二进制的 sha256 并入库（供接手人比对）
