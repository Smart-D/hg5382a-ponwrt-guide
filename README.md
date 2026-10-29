# 烽火 HG5382A / HG5582A / HG5585F 刷 PonWrt 完整指南

> Airoha AN7581 · 平行 NAND · XG-PON / XGS-PON
>
> 本文基于一台 **HG5382A**（Winbond W29N02KVSIAF 颗粒）的实机刷写全过程整理，
> 包含**官方文档里没写、但会让你刷完上不了网甚至变砖**的关键细节。

---

## ⚠️ 免责声明

- 刷机会**失去运营商保修**，且可能违反入网协议。
- **原厂固件无法回退**（原厂 UBI apps 卷一旦清空就拿不回来）。
- 本文不提供任何固件/校准数据，**不要**把别人的 factory 分区刷进你的机器。
- 操作风险自负。

---

## 一、适用机型

| 机型 | SoC | 存储 | 说明 |
|---|---|---|---|
| FiberHome **HG5382A / HG5382A3** | AN7581 | 平行 NAND | 本文实测机型 |
| FiberHome **HG5585F** CT/CU | AN7581 | 平行 NAND | 同平台 |
| FiberHome HG5582A | AN7581 | 平行 NAND | 同系列 |

> **必须确认光模块型号**：HG5382A 用 `UX3361`，HG5382A3 用 `UX3363`。
> 刷好后可以在 LuCI 或 `/sys/kernel/debug/airoha-xpon-pon0/frontend` 里看到。
> 校准数据**逐机唯一，两种型号不能混用**。

### ❗ 本平台只能跑 XG-PON / XGS-PON

GPON 与 XG-PON 波长不同（GPON 1490/1310 nm，XG-PON 1577/1270 nm），
AN7581 现有设备的 BOSA 无法工作在 GPON 模式。

> 这点由 PonWrt 作者 [pbs05](https://github.com/pbs05) 在
> [ponwrt#22](https://github.com/pbs05/ponwrt/issues/22) 明确说明。
> 如果你的线路是纯 GPON，**本方案不适用**。

---

## 二、刷完能到什么状态（实测）

```
PON:      onu_state O5 / ONU-ID 分配成功 / lifecycle operational
认证:     OMCI CTC LOID accepted
校准:     calibration ready
光功率:   rx -18.63 dBm / tx +3.58 dBm
PPP oE:   up，正常拿到运营商内网地址
延迟:     ping 223.5.5.5 ≈ 36 ms
吞吐:     ~590 Mbps（24 线程，CPU 99% 打满）
```

**注意吞吐**：PonWrt 的 PPPoE/NAT 走**软件转发**，本机实测约 590 Mbps，
约为原厂固件（923 Mbps）的 60%。想跑满千兆需改桥接让路由器拨号。

---

## 三、工具与素材

| 工具 | 用途 | 来源 |
|---|---|---|
| `an758x-stock2ubi` | 免拆刷入工具（在光猫上跑，提供 Web UI） | [pbs05/an758x-stock2ubi](https://github.com/pbs05/an758x-stock2ubi/releases) |
| `uboot-an758x`（官方引导） | 重建 UBI 后刷回的 preloader + bl31-u-boot.fip | [pbs05/uboot-an758x](https://github.com/pbs05/uboot-an758x/releases) |
| `uboot-an758x-winbond`（**华邦迁移版**） | Winbond 颗粒专用，8bit 读 / 4bit 写 | [xe5700/uboot-an758x-winbond](https://github.com/xe5700/uboot-an758x-winbond/releases) |
| PonWrt 固件 | `ponwrt-airoha-an7581-*-squashfs-sysupgrade.itb` | [pbs05/ponwrt](https://github.com/pbs05/ponwrt) 或 [huladabang/ponwrt](https://github.com/huladabang/ponwrt/tags) |
| `hg5382a_factory_tools.zip` | JFFS2 提取 + factory.bin 拼装 | 见[致谢](#致谢)里 7a3daf 的帖子附件 |

---

## 四、⚠️ 刷机前必须做的备份（否则必废）

### 4.1 开 telnet

老固件可通过 `/fh_tool/api` 接口开启（AES 加密的私有 API）。

```
telnet <ONT_IP>
用户名：telnetadmin
口令：  FH-nE7jA%5m + MAC 后 6 位（大写）
su 口令：Fh@ + MAC 后 6 位
```

### 4.2 备份 factory 分区（**最重要，没有之一**）

```sh
cat /proc/mtd                     # 找到 name 为 factory 的分区号（各机型可能不同）
dd if=/dev/mtdblockN of=/tmp/factory_backup.bin bs=128k
sha256sum /tmp/factory_backup.bin
```

把文件取回电脑并**至少存三份**。里面是：

- 光模块 **PON 校准表**（`table`，2048 字节）
- 基础 MAC、PON SN、整机 SN

> 官方固件刷写工具会把这个分区转成 UBI 里的 `factory` 卷。
> **丢了它，PON 物理层直接锁不住，设备停在 O1。**

### 4.3 同时备份 boot / 其它关键分区（可选但建议）

```sh
for p in boot bootflag KernelA RootfsA; do
  idx=$(awk -v n="\"$p\"" '$0 ~ n {print $1}' /proc/mtd | tr -d :)
  [ -n "$idx" ] && dd if=/dev/${idx}ro of=/tmp/${p}.bin bs=128k
done
```

> **不要 `dd` 正在挂载的 UBI 分区**（如 apps）—— 部分机型会直接用户态死锁，
> 表现为 httpd/telnetd 只接受连接但不回数据，只能断电重启。

### 4.4 记录业务参数

LOID、宽带账号密码、VLAN ID、802.1p、MTU —— 刷完要用。

---

## 五、刷机流程

### Step 1 · 上传 stock2ubi 到光猫

光猫只有 `wget`（只能下载、不能上传）。两种把工具送进去的办法：

**方式 A（推荐，不依赖 PC 防火墙）**：在光猫起 FTP，PC 主动推送

```sh
# 光猫上（本机 busybox 的 ftpd 是 inetd 风格，没有 -p，必须用 tcpsvd 包一层）
tcpsvd -E 0.0.0.0 2121 ftpd -w -A /tmp 2>/tmp/f.err &
```

```python
# PC 上
ftp.connect("<ONT_IP>", 2121); ftp.login("anonymous", "")
ftp.storbinary("STOR an758x-stock2ubi", open(path, "rb"), blocksize=32768)
```

**方式 B**：PC 起 HTTP 服务，光猫 `wget`（需要 Windows 防火墙放行入站）

```sh
cd /tmp
wget http://<PC_IP>:8000/an758x-stock2ubi
chmod +x /tmp/an758x-stock2ubi
```

> 💡 **后台运行**：`/tmp/an758x-stock2ubi &`
> 避免 telnet 超时把刷机过程打断（来自网友 *lotusswan* 的经验）。

启动后浏览器访问 `http://<ONT_IP>:3333`。

---

### Step 2 · 判断你的 NAND 是东芝还是华邦

这是**决定会不会变砖**的关键分叉。

| 颗粒 | 用什么引导 |
|---|---|
| Toshiba TC58NVG1S3HTAI0 等 | 官方版 `preloader.bin` / `bl31-u-boot.fip` |
| **Winbond W29N02KVSIAF** | **必须先用迁移版** `*-tmp-8bit-read-4bit-write.bin/.fip` |

**根因**（由网友 *lyq1996* 定位，见 [uboot-an758x#21](https://github.com/pbs05/uboot-an758x/issues/21)）：

```
原厂固件：整片闪存按 8-bit BCH 编码读写
uboot-an758x v2026.07-3 起：按 ONFI 标称值改成 4-bit
        ↓
Stock2UBI 写入的 BL2/FIP 是原厂内核写的 → 8-bit 编码
新 BL2 只按 4-bit 解码 → 大量块 ECC 不可纠正
        ↓
ERROR: Stored BL31 + U-Boot FIP failed (-2)
Press x to load BL31 + U-Boot FIP via XMODEM   →  全灯灭
```

**没有拆机手段的话，华邦颗粒千万别直接刷官方版。**

---

### Step 3 · 写入迁移版引导（华邦机型）

在 stock2ubi 的 Web UI 里：

| 项 | 选择 |
|---|---|
| BL2 / Preloader | `*-preloader-tmp-8bit-read-4bit-write.bin` |
| BL31+U-Boot FIP | `*-bl31-u-boot-tmp-8bit-read-4bit-write.fip` |

写入成功会提示 `BL2 written ... readback verified`，随后自动重启。

---

### Step 4 · 进 U-Boot 恢复页

1. PC 网卡设静态 **`192.168.0.2/24`**，网关留空
2. 上电约 1 秒后**按住 Reset**，浏览器打开 `http://192.168.0.1`

迁移版固件会提示当前处于 bridge 模式（`"bridge": true`）。

---

### Step 5 · 重建 UBI（不可逆）

点「**重建 UBI**」。

这一步会：
- 把闪存从原厂 **8-bit 布局切换到 4-bit 布局**
- **清空整个 UBI 分区（含 fip 卷）**

> ⚠️ 从此以后**必须立刻按顺序刷回**，否则卡 BL2 只能 TTL 救砖。

---

### Step 6 · 刷回官方引导 + 系统

**顺序不能错**：

1. `flash-bl2` → **官方** `*-preloader.bin`
2. `flash-uboot` → **官方** `*-bl31-u-boot.fip`
3. `flash` → `ponwrt-*-squashfs-sysupgrade.itb`
4. 重启

用 HTTP API 驱动（脚本化）时注意：**上传必须是文件原始字节**，不是 multipart：

```python
req = urllib.request.Request(BASE + "/api/upload", data=raw_bytes,
                            headers={"Content-Type": "application/octet-stream"})
```

> 用 multipart 会让固件的魔数校验（`0xaa640001`）失败，报
> `Expected a BL2 preloader FIP or a full main-area firstblock...` —— **这个报错很有误导性**。

---

### Step 7 · 🔴 恢复 factory 校准数据（本文最重要的部分）

**不做这一步，PON 永远停在 O1。**

#### 症状对照

| 指标（debugfs） | 无校准 | 有校准 |
|---|---|---|
| `calibration` | `invalid` | **`ready`** |
| `rx_power_8472` | `0x0000` 恒零 | `0x0089` |
| `tx_bias_8472` | `0x0000` | `0x2612` |
| `freqdet_locked` | **0**（锁不住） | **1** |
| `frequency_level` | 偏离目标 | **等于** `frequency_target` |
| `xgtc_sync` | `hunt` | **`in-sync`** |
| `onu_state` | **O1** | **O5** |
| 模块温度 | 66~74 ℃ 空转发热 | **48 ℃** |

#### 7.1 确认 factory 卷

```sh
for v in /sys/class/ubi/ubi0_*; do echo "$(basename $v) $(cat $v/name) $(cat $v/data_bytes)"; done
# 期望看到： ubi0_3 factory 0        ← 卷存在但为空（1 MiB，static）
```

#### 7.2 从备份重建 1 MiB `factory.bin`

PonWrt 设备树期望的布局：

| 偏移 | 大小 | 内容 |
|---|---|---|
| `0x0000` | `0x800` | PON 校准（APONCAL，魔数 ASCII `APONCAL`）|
| `0x1000` | `0x1000` | WiFi EEPROM（无 WiFi 机型可全 FF）|
| `0x2000` | 6 | 基础 MAC |
| `0x2010` | 8 | PON SN |
| `0x2020` | `0x20` | 整机 SN（ASCII，0 填充）|

用 `hg5382a_factory_tools.zip` 里的脚本（见致谢）：

```bash
# 先看备份里有什么文件
python3 jffs2_extract.py factory_backup.bin --list
#   ... table  ino=11 size=2048 compr=NONE,ZLIB    ← 校准表

# 拼装（会自动从 JFFS2 提取 table 并转 APONCAL，带自检）
python3 build_factory.py \
    --mac AA:BB:CC:DD:EE:FF \
    --ponsn <16位hex> \
    --serial <整机SN ASCII> \
    --mtd factory_backup.bin \
    --out factory.bin
```

#### 7.3 写入 UBI

```sh
scp factory.bin root@<ONT_IP>:/tmp/
sha256sum /tmp/factory.bin                 # 与本地比对
ubiupdatevol /dev/ubi0_3 /tmp/factory.bin  # ubi0_3 = factory
cat /sys/class/ubi/ubi0_3/data_bytes       # 应为 1048576
reboot
```

重启后 `dmesg` 应出现：

```
airoha-paged-bosa 0-0051: UX3363 frontend registered, ID ..., calibration ready
```

> 若 `factory` 卷不存在，可在 U-Boot 恢复页用 `flash-board-data` 写入
> （要求文件恰好 1 MiB，会校验大小）。

---

### Step 8 · PON 认证配置

```sh
uci set pon.line0.mode='xgpon'                      # 只能 xgpon / xgspon
uci set pon.line0.serial_number='<16位hex>'          # 本机 PON SN
uci set pon.line0_omci.loid='<你的LOID>'
uci set pon.line0_omci.loid_password=''
uci set pon.line0_oam.loid='<你的LOID>'
uci set pon.line0_oam.loid_password=''
# 联通 ZTEG OLT 会校验版本号格式，必须 VX.0 / VX.X.X
uci set pon.line0_omci.vendor_id='FHTT'
uci set pon.line0_omci.equipment_id='HG5382A'
uci set pon.line0_omci.hardware_version='V1.0'
uci set pon.line0_omci.software_version='V1.0.0'
uci commit pon

/etc/init.d/airoha-pond restart      # 必须重启 pond，只 reload pon 不生效
```

验证：

```sh
ponctl --device pon0 status | head -20
logread | grep -i loid          # 期望 status=1 meaning=accepted
```

---

### Step 9 · PPPoE + VLAN

**这一步是"收不到 PADO"的元凶**：必须打 VLAN 标签，而且 802.1p 要按运营商的要求设。

```sh
# 父接口 1500，PPPoE 自身 1492
uci set network.pon0mtu=device
uci set network.pon0mtu.name='pon0'
uci set network.pon0mtu.mtu='1500'

# VLAN 子接口
uci set network.pon41=device
uci set network.pon41.type='8021q'
uci set network.pon41.ifname='pon0'
uci set network.pon41.vid='<VLAN_ID>'
uci set network.pon41.name='pon0.<VLAN_ID>'
uci set network.pon41.mtu='1500'

# WAN
uci set network.wan=device='pon0.<VLAN_ID>'
uci set network.wan.proto='pppoe'
uci set network.wan.username='<宽带账号>'
uci set network.wan.password='<宽带密码>'
uci set network.wan.mtu='1492'
uci set network.wan6.disabled='1'      # 不需要 v6 时关掉，避免刷日志
uci commit network
```

**802.1p 需要单独设置**（uci 没有对应选项，且重启会丢）：

```sh
ip link set dev pon0.<VLAN_ID> type vlan egress-qos-map 0:<PCP>
```

持久化用开机脚本：

```sh
cat > /etc/init.d/pon-vlan-qos <<'EOF'
#!/bin/sh /etc/rc.common
START=99
apply() {
    for i in $(seq 1 30); do [ -d /sys/class/net/pon0.41 ] && break; sleep 1; done
    ip link set dev pon0 mtu 1500 2>/dev/null
    ip link set dev pon0.41 mtu 1500 2>/dev/null
    ip link set dev pon0.41 type vlan egress-qos-map 0:1 2>/dev/null
}
start() { apply; }
boot() { apply; }
EOF
chmod +x /etc/init.d/pon-vlan-qos
/etc/init.d/pon-vlan-qos enable
```

---

## 六、踩坑速查表

| # | 现象 | 原因 | 解决 |
|---|---|---|---|
| 1 | PON 停 O1、`freqdet_locked=0`、光功率恒 0、模块 74℃ | **缺 factory 校准** | 恢复 factory 卷（Step 7）|
| 2 | PPPoE `Timeout waiting for PADO` | 没打 VLAN 或 802.1p 不对 | 见 Step 9 |
| 3 | 注册到 O5 后被踢、O3/O5 摆动 | OLT 校验 OMCI 版本号格式 | 硬件 `VX.0`、软件 `VX.X.X` |
| 4 | `loid_configured=false` | 只 reload 了 `pon` 服务 | `/etc/init.d/airoha-pond restart` |
| 5 | `Interface pon0 has MTU of 1492 -- should be at least 1500` | 父接口 MTU 被改小 | pon0 与 VLAN 口都设 1500 |
| 6 | `failed to queue PON line start: -95`，之后所有模式都报 -95 | 运行中切 GPON 把驱动搞坏 | 本机只能 xgpon，**不要试**；已坏则重启 |
| 7 | 刷机命令只回显不执行 | 复杂长命令在 telnet 上不可靠 | 把逻辑写成 `/tmp/xxx.sh` 再 `sh` 调用 |
| 8 | telnet 只连不回数据 | 短时间开了太多会话 | 单会话 + 重试；不行就断电重启 |
| 9 | 读 `mtd15`（apps）后整个用户态死锁 | raw 读正在挂载的 UBI 分区 | 不要读；只能断电重启 |
| 10 | 上传的引导被拒绝：`Expected a BL2 preloader FIP...` | `/api/upload` 传了 multipart | 改传**原始字节** |

---

## 七、性能说明

| 场景 | 速率 |
|---|---|
| 原厂固件（硬件加速） | ~923 Mbps（1GbE 上限）|
| PonWrt 自拨号（软件转发） | **~590 Mbps**（CPU 99%） |
| PonWrt 桥接 + 路由器拨号 | 接近线速（取决于路由器） |

瓶颈是 PonWrt 的 PPPoE/NAT 未走硬件卸载（NPU/PPE 存在但未接入该路径）。
追求满速请改桥接。

---

## 八、致谢

这个指南能写出来，靠的是一群素不相识的网友把踩过的坑公开出来。**排名不分先后**：

- **[@pbs05](https://github.com/pbs05)** —— PonWrt / `uboot-an758x` / `an758x-stock2ubi` 作者。
  没有这套开源工具，AN7581 光猫刷机根本无从谈起；还在
  [ponwrt#22](https://github.com/pbs05/ponwrt/issues/22) 里明确回答了"AN7581 不能跑 GPON"这个关键问题。

- **[@xe5700](https://github.com/xe5700)** —— 提供了
  [uboot-an758x-winbond](https://github.com/xe5700/uboot-an758x-winbond)
  华邦颗粒专用迁移版引导（8-bit 读 / 4-bit 写）。**没有它，华邦机型刷机必砖。**

- **[@lyq1996](https://github.com/Loong1996)** —— 在
  [uboot-an758x#21](https://github.com/pbs05/uboot-an758x/issues/21)
  里**定位了变砖的根因**（原厂 8-bit BCH vs 工程 4-bit ECC），
  并维护了 [ImmortalWrt-Airoha](https://github.com/Loong1996/ImmortalWrt-Airoha)
  与恢复指南，其 `backup-and-restore.md` 里"逐机唯一的出厂数据丢了就再也拿不回来"的警告救了很多人。

- **[@huladabang（呼啦大棒）](https://github.com/huladabang)** —— 恩山《AN7581 光猫：免拆机刷入 PonWrt 固件教程》作者，
  并编译了各机型的 [PonWrt 预编译包](https://github.com/huladabang/ponwrt/tags)。
  **本文所用固件正是从他的 an7581 target 构建产物中提取的**（整包 tar 里包含所有设备的 sysupgrade）。

- **恩山网友 `7a3daf`** —— 《HG5382A 刷 PonWrt：factory 备份、PPPoE 与 OLT 兼容》
  （[tid-8493012](https://www.right.com.cn/forum/thread-8493012-1-1.html)）。
  **本文最有价值的三点全部来自这篇帖子**：
  1. 「刷机前必须备份 factory 分区，否则 PON 口无法使用」
  2. `hg5382a_factory_tools.zip` 工具集（`jffs2_extract.py` / `build_factory.py`），
     没有它就无法从 JFFS2 备份里还原校准数据
  3. 联通 ZTEG OLT 会校验 OMCI 版本号格式，需改成 `VX.0` / `VX.X.X`

- **网友 `lostz`** —— 在 issue 里给出"这批机器砖不了，随便玩"的判断，
  以及**光模块校准数据丢失不可恢复**的直白提醒。

- **网友 `lotusswan`** —— 提示 `/tmp/an758x-stock2ubi &` 后台运行，避免 telnet 超时中断刷机。

- **[@naoki66](https://github.com/naoki66)** —— 维护 Gemtek XR1710G / XG2010G 的 ImmortalWrt 构建，
  为同平台机型提供了另一条可选路线。

- **`mao.fan`** —— 《免拆刷入 OpenWrt 保姆级教程》图文作者，降低了入门门槛。

- **恩山无线论坛 / OpenWrt 社区** —— 各种零散但关键的实测经验。

> 如果本文有帮到你，请把上面的原帖和仓库点个 star —— 功劳是他们的。

---

## 九、参考链接

| 内容 | 链接 |
|---|---|
| PonWrt 主仓库 | https://github.com/pbs05/ponwrt |
| 免拆刷入工具 | https://github.com/pbs05/an758x-stock2ubi |
| 官方 U-Boot | https://github.com/pbs05/uboot-an758x |
| **华邦迁移版 U-Boot** | https://github.com/xe5700/uboot-an758x-winbond |
| 变砖根因 / 救砖讨论 | https://github.com/pbs05/uboot-an758x/issues/21 |
| "能否支持 GPON"官方答复 | https://github.com/pbs05/ponwrt/issues/22 |
| 预编译固件（各机型） | https://github.com/huladabang/ponwrt/tags |
| 备份/恢复文档 | https://github.com/Loong1996/ImmortalWrt-Airoha/blob/master/docs/backup-and-restore.md |
| 恩山 HG5382A 刷机帖（factory/PPPoE/OLT） | https://www.right.com.cn/forum/thread-8493012-1-1.html |
| 恩山免拆刷入教程 | https://www.right.com.cn/forum/thread-8491946-1-1.html |
| 保姆级图文教程 | https://mao.fan/article/576 |

---

## License

本文档以 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 发布。
文中提到的所有软件/固件版权归各自作者所有。

**再次提醒：不要把本文当成"照抄就能成功"的教程 —— 先确认你的光模块型号、NAND 颗粒、线路制式，并且一定先备份 factory。**
