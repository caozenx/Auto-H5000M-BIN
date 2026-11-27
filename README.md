# ImmortalWrt H5000M 自动编译

[![Build](https://github.com/caozenx/Auto-H5000M-BIN/actions/workflows/build-test.yml/badge.svg)](https://github.com/caozenx/Auto-H5000M-BIN/actions/workflows/build-test.yml)

为 **Hiveton H5000M**（SoC MT7987A + Wi-Fi MT7992，5G CPE）自动编译 ImmortalWrt 固件。fork 自 [existyay/Auto-H5000M-BIN](https://github.com/existyay/Auto-H5000M-BIN)。

| 项 | 值 |
| --- | --- |
| 源码 | [`padavanonly/immortalwrt-mt798x-24.10`](https://github.com/padavanonly/immortalwrt-mt798x-24.10) @ `mt798x-mt799x-6.6-mtwifi` |
| 基础配置 | 上游 `defconfig/mt7987_mt7992.config` + 本仓库 `h5000m.extra.config` + `scripts/local-build.sh` 里的功能开关 |
| 内核 | Linux 6.6 |
| Wi-Fi 驱动 | **MediaTek 闭源 mtwifi**（`kmod-mt_wifi7` / `kmod-mt_hwifi` / `kmod-mt7992`），不是开源 mt76 |
| 默认地址 | `192.168.100.1`（可用 `LAN_IP` 修改） |
| 默认账号 | `root` / `admin`（上游默认，刷机后请立即修改） |
| 自动构建 | 每周一 00:00（北京时间）；也可在 Actions 页面手动触发 |

固件下载：[Releases](https://github.com/caozenx/Auto-H5000M-BIN/releases)

---

## 刷哪个文件

| 文件 | 用途 |
| --- | --- |
| `immortalwrt-mediatek-filogic-hiveton-h5000m-squashfs-sysupgrade.bin` | **日常升级**：在 LuCI「备份/升级」或 `sysupgrade` 里刷。勾选"保留配置"则现有网络、Wi-Fi 等设置不变 |
| `immortalwrt-mediatek-filogic-hiveton-h5000m-initramfs-kernel.bin` | **U-Boot 恢复 / 首次刷入**：从内存启动的临时系统，进去后再刷 sysupgrade |
| `build.config` / `enabled-packages.txt` / `openwrt-image.manifest` | 本次构建的完整配置、启用的包、固件内包清单，用于核对 |

> **不要刷其他 `.bin`。** 上游 filogic 平台编译时会顺带生成其他参考板的引导程序（`mt7981-ram-ddr3/ddr4-bl2.bin`、`mt7986-ram-ddr3/ddr4-bl2.bin`），它们**不属于 H5000M（MT7987A）**。原仓库会把它们一起发布；本 fork 已改为只发布文件名含 `hiveton-h5000m` 的产物。

> 默认 LAN 地址只在**全新刷机或恢复出厂**时生效；保留配置升级会沿用路由器上现有的 `/etc/config/network`。

---

## 硬件适配

这些都来自上游设备树 `target/linux/mediatek/dts/mt7987a-hiveton-h5000m.dts` 和 `board.d`，不是本仓库额外添加的：

| 硬件 | 内容 |
| --- | --- |
| Wi-Fi 指示灯 | 两颗白色 LED（GPIO 4 / 3），默认跟随 `ra0`（2.4G）/ `rai0`（5G）收发闪烁 |
| WAN 指示灯 | 2.5G PHY 上的白色 LED，默认跟随 `eth1` |
| 按键 | reset（GPIO 1）、wps（GPIO 0） |
| 风扇 | `pwm-fan`，25 档，由 `luci-app-Airpifanctrl` 控制 |
| Wi-Fi 校准 | 读 eMMC 里的 EEPROM（`MTK_WIFI7_FIRST_IF_EEPROM_FLASH`），SKU `BE5040` |
| 5G 模组 | USB 接口，QModem 管理（见下文） |

---

## 功能开关

所有开关都是环境变量；GitHub Actions 手动触发时对应同名的小写输入项。定时构建使用下表默认值。

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `ENABLE_WED` | `true` | MTK Wi-Fi 硬件转发加速（编译期，决定是否编入 `kmod-mtk_wed`），见下文 [WED](#wed-是什么开还是关) |
| `ENABLE_HNAT` | `false` | 刷机后 TurboACC 是否默认开 MediaTek HNAT。`false` = 打包 `zz-h5000m-turboacc`，首启设 `fastpath=none`、`fullcone=0`（包仍在，LuCI 可手动打开），见 [HNAT 已知问题](#hnat--wed-已知问题连接中途卡死) |
| `ENABLE_CLOUDFLARED` | `true` | Cloudflare Tunnel：`cloudflared` + `luci-app-cloudflared` + 中文包（immortalwrt 官方 feed）。feed 里的 `cloudflared` 是 `2025.5.0`（`openwrt-24.10` 分支，2026-11-27 查），比官方发布旧；`/etc/config/cloudflared`（含 token）随保留配置升级保留 |
| `LAN_IP` | `192.168.100.1` | 默认 LAN 地址 |
| `ENABLE_QMODEM` | `true` | [FUjr/QModem](https://github.com/FUjr/QModem) 5G 模组管理（Quectel / Fibocom / MEIG / SIMCom） |
| `ENABLE_QMODEM_NEXT` | `true` | QModem 新版 JS 界面 `luci-app-qmodem-next`（与 Lua 版互斥） |
| `ENABLE_QMODEM_LUA` | `false` | QModem 经典 Lua 界面 `luci-app-qmodem` |
| `ENABLE_ORIGINAL_MODEM` | `false` | 上游 `luci-app-modem` + ModemManager；与 QModem 互斥，同时打开会自动关掉它 |
| `ENABLE_UPNP` | `true` | UPnP / NAT-PMP（`miniupnpd-nftables`）。WAN 是公网 IP 时，局域网设备可自行开端口 |
| `ENABLE_MOSDNS` | `true` | MosDNS + v2ray-geodata |
| `ENABLE_ADBLOCK` | `true` | DNS 层广告 / 恶意域名过滤 |
| `ENABLE_NIKKI` | `false` | Nikki（mihomo）代理 |
| `ENABLE_VLMCSD` | `false` | KMS 激活服务器 |
| `ENABLE_ADGUARDHOME` | `false` | AdGuardHome |
| `ENABLE_OPENCLASH` | `false` | OpenClash |
| `ENABLE_HOMEPROXY` | `false` | HomeProxy |
| `ENABLE_DOCKERMAN` | `false` | DockerMan + dockerd |

下载加速（国内网络可按需覆盖）：

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `GOPROXY` / `GOSUMDB` | `goproxy.cn` / `sum.golang.google.cn` | Go 依赖（MosDNS / Nikki / HomeProxy） |
| `DOWNLOAD_MIRROR` | 清华 / 中科大 / 北外 | OpenWrt 源码包镜像 |
| `GITHUB_PROXY_PREFIXES` | `ghfast` / `gh-proxy` / `gh.llkk` | GitHub 直连失败后的代理前缀，原始地址优先 |

---

## WED 是什么，开还是关

**WED（Wireless Ethernet Dispatch）** 是 MediaTek NETSYS 里的一块硬件：开启后，有线 ↔ Wi-Fi 之间的转发由硬件（配合 HNAT / PPE）直接完成，不再经过 CPU。

| | 开启（默认） | 关闭 |
| --- | --- | --- |
| Wi-Fi 吞吐 | 最高 | 受 CPU（4 × Cortex-A53）限制，高带宽时下降 |
| CPU 占用 | 低 | Wi-Fi 大流量时明显升高 |
| 数据通路 | 硬件转发，问题较难排查 | 全走内核，便于抓包、排查 |
| 风险 | 原仓库提到个别环境重启后进入死循环（未在本机复现） | 性能损失 |

**建议保持开启**，只在遇到 Wi-Fi 稳定性问题、需要对照排查时临时用 `ENABLE_WED=false` 编一版测试。

> `ENABLE_WED` 是**编译期**开关，只决定固件里有没有 WED；真正让流量走硬件的是**运行时**的 TurboACC（`fastpath=mediatek_hnat`）。WED 只在 HNAT 开启时起作用，`ENABLE_HNAT=false`（默认）时 WED 编进去也不会被用上。由于下面的问题，本 fork 默认关 HNAT。

## HNAT / WED 已知问题：连接中途卡死

2026-11-27 在 H5000M（ImmortalWrt 24.10-SNAPSHOT r33418，`kmod-mtk_wed 6.6.94.TEST-r1`，TurboACC `fastpath=mediatek_hnat`、`bind_rate=30`、`ppenum=2`）上实测。

**现象**：iPhone / iPad / Mac 等 Wi-Fi 终端打开网页或 App 页面，第一次要等很久，甚至请求失败，刷新就好。Steam 商店首次打开时，`shared.fastly.steamstatic.com` 的 30 张图全部卡住（约 5.9 s 后报错），刷新后 157 张全部成功。

**根因**：连接的收包速率超过 `bind_rate`（30 包/秒）后，HNAT 把它交给硬件 PPE 转发。**部分 WAN→LAN（下行）条目写错了**，之后的下行包全被硬件丢掉，内核和 tcpdump 都看不到。客户端表现为连接建立、收了约 32 KB 后不再有数据，要等应用超时重连。

`/sys/kernel/debug/hnat/hnat_entry` 里 PPE1 的条目：

| 条目 | 改写结果 |
| --- | --- |
| 正确 | `213.156.140.136:443->WAN:49808 => 213.156.140.136:443->192.168.100.150:49808` |
| 坏 ① | `199.232.211.52:443->WAN:57215 => 36.3.72.0:57540->63.5.14.59:56831`（乱码地址，多条坏条目是同一串） |
| 坏 ② | `199.232.215.52:443->WAN:57185 => 199.232.215.52:443->WAN:57185`（没改写回内网） |

**证据**（8 并发、每次新建连接，下载 Fastly 上一张 1.3 MB 的图）：

| 测试 | 结果 |
| --- | --- |
| 有线 PVE，HNAT 开，60 次 | 0 次卡死，最长 2.2 s |
| Wi-Fi Mac，HNAT 开，4 轮 × 60 次 | 4 / 2 / 2 / 5 次卡死（都停在约 32–49 KB，12 s 超时） |
| Wi-Fi Mac 40 次，同时读 `hnat_entry` | 2 次卡死（本地端口 57185 / 57215），**正好对应 2 条坏条目** |
| Wi-Fi Mac，HNAT 关（先 `hook_toggle=0` 3 轮，再持久关闭后 4 轮，共 7 × 60 次） | **0 次 12 s 卡死**，平均从 2.4–2.8 s 降到 1.7–1.9 s（重启后约 1 分钟那轮有 4 次 5–8.7 s 的慢请求） |
| 有线 PVE 4 路下载 15 s | HNAT 开 538 Mbit/s、CPU 2%；HNAT 关 539 Mbit/s、CPU 平均 16%（单核最高 21%） |

**结论**：关掉 HNAT 后卡死消失，吞吐不变（这条 1G 宽带上 4 核 A53 只用到约 16% CPU）。有线 0 次卡死，坏条目只出现在 Wi-Fi 方向，**推测**问题在 WED 路径，未证实；`ENABLE_WED=false` + HNAT 开是否就正常，**未测试**。

**规避**：

| 方式 | 做法 |
| --- | --- |
| 本 fork 默认 | `ENABLE_HNAT=false`：首启 `fastpath=none`、`fullcone=0` |
| 现有固件 | LuCI → 网络 → TurboACC → Fastpath engine 选 `Disable`，保存应用（**有人在场**） |
| 临时验证 | `echo 0 > /sys/kernel/debug/hnat/hook_toggle`（不持久，重启恢复） |

> ⚠️ **不要在 HNAT 已关时执行 `/etc/init.d/turboacc restart`**。旧版 turboacc 的 `start_service` 会先写 `hook_toggle=0` 再写 `1`、最后再写 `0`；中间那次重新打开 HNAT 导致驱动报 `hnat_set_wdma_pse_port_state() is null`，**Wi-Fi 消失、全网断网，只能断电恢复**（2026-11-27 实测）。上游新版 turboacc 已经去掉了这段"先开再关"的逻辑，但仍建议只通过 LuCI 修改。

> 上游 `uci-defaults/turboacc` 在每次刷机首启时都会设 `fastpath=mediatek_hnat`，并按内核模块开 fullcone（`1` 或 `2`）。`uci-defaults` 在保留配置升级时**也会执行**，所以不改的话，每次刷完 HNAT 都会被重新打开。本 fork 的 `zz-h5000m-turboacc` 按文件名排在它后面执行，把这两项改回关闭。

> 原仓库的"关闭 WED"写在 `h5000m.extra.config` 里，但会被后面的驱动配置重新打开，并且后面的校验还强制要求 WED 存在，所以**从未生效**（发布的固件里 `kmod-mtk_wed=y`）。本 fork 改为由 `ENABLE_WED` 统一控制：在驱动配置之后执行 `apply_wed_setting()`，并在 `make defconfig` 后、固件清单里双向校验（要开必须有，要关必须没有）。

---

## 5G 模组：QModem 还是 ModemManager

两者都是"模组管理器"：会探测模组、占用它的 AT 口和 `/dev/cdc-wdm*` 控制口、负责拨号。**同一台设备上只能留一个**，否则会争抢端口，表现为拨号失败、AT 命令无响应、信号信息时有时无。

| | QModem（默认） | ModemManager（`ENABLE_ORIGINAL_MODEM`） |
| --- | --- | --- |
| 定位 | 面向路由器 5G 模组的 LuCI 面板 | 通用 Linux 模组管理服务（桌面发行版常用） |
| 拨号 | 自带 `quectel-CM-5G-M` / `tom_modem` | 通过 netifd `modemmanager` 协议 |
| 功能 | 信号详情、锁频段 / 锁小区、短信收发转发、AT 终端、多模组 | 标准 QMI / MBIM，功能较基础 |
| 适合 | Quectel RM520N-GL 等 5G 模组 | 标准 QMI 模组、需要 ModemManager 生态时 |

**建议用 QModem。**

> 原仓库在启用 QModem 时也把 `modemmanager` / `modemmanager-rpcd` / `luci-proto-modemmanager` 一起装上并开机自启，理由是"QModem 的 mbim/qmi 辅助脚本需要它"。但 QModem 自己的 Makefile 并不依赖 ModemManager。本 fork 已去掉，并在 defconfig 后和固件清单里校验：只要没开 `ENABLE_ORIGINAL_MODEM`，固件里就不能有 ModemManager。

---

## 本仓库对上游的修改

| 修改 | 说明 |
| --- | --- |
| `patches/mtwifi-apcli-active-only.patch` | `mtwifi_cfg` 按实际启用的 VIF 计算 `BssidNum`（上游固定为 4）并跳过禁用的接口，避免禁用部分 VIF 后 AP 起不来 |
| `patches/mtk-hnat-local-dest.patch` | `mtk_hnat` 不卸载目的地址为本机的流量，修复开启 HNAT 后 Wi-Fi 客户端访问路由器后台异常 |
| `set_default_lan_ip()` | 把上游 `config_generate` 里的默认 `192.168.6.1` 改为 `LAN_IP`，并校验地址格式 |
| `apply_wed_setting()` | 见上文 WED |
| `apply_hnat_default()` | `ENABLE_HNAT=false` 时生成 `files/etc/uci-defaults/zz-h5000m-turboacc`（`fastpath=none`、`fullcone=0`），见 [HNAT 已知问题](#hnat--wed-已知问题连接中途卡死) |
| Cloudflared | `ENABLE_CLOUDFLARED` 控制 `cloudflared` / `luci-app-cloudflared` / 中文包；defconfig 后和固件清单都做双向校验 |
| ModemManager | 见上文 5G 模组 |
| 发布产物 | 只发布 `*hiveton-h5000m*`，不再附带 MT7981 / MT7986 的 bl2 |
| `show_features()` | 修复 heredoc 未闭合：原来会把 `resolve_modem_stack()` 整个吞进打印文本，导致"QModem 与原版 Modem 互斥"的检查从未执行 |
| 其他 | QMI WWAN 驱动适配 Linux 6.6；Go feed 固定 `sbwml/packages_lang_golang -b 24.x`；`mihomo-meta` 冲突剥离；补齐 IPv6 netfilter（ip6tables / NAT6）；UPnP 显式选 `miniupnpd-nftables`；Adblock 运行时依赖显式声明 |

---

## GitHub Actions

- 触发：每周日 16:00 UTC（北京时间周一 00:00）定时构建；或在 Actions 页面手动 `Run workflow`。
- 手动触发时所有开关（含 `enable_wed`、`lan_ip`）都可单独设置；不勾 `publish_release` 则只上传 Artifact、不更新 Release。
- 勾选 `publish_release` 时覆盖发布到固定的 `latest` tag。
- 可选 `run_config_coverage`（`quick` / `full`）先跑覆盖性配置测试，与编译并行，不阻塞发布。
- feeds 更新失败直接中止，避免产出缺包固件。
- 免费额度不够时可选 `runner_type=self-hosted`，见下文。

### Self-hosted runner

1. 仓库 **Settings → Actions → Runners → New self-hosted runner**，选 Linux x64，标签至少包含 `linux`、`x64`、`h5000m`。
2. 准备 Ubuntu 22.04 / 24.04，安装依赖（可直接运行 `bash scripts/local-build.sh --install-deps`），预留 30 GB 以上磁盘（ccache 满载约 50 GB）。
3. 以普通用户运行 runner：`./config.sh --labels linux,x64,h5000m && ./svc.sh install && ./svc.sh start`。
4. 手动触发时 `runner_type` 选 `self-hosted`。自托管时脚本会跳过 `apt-get` 和磁盘清理。

---

## 本地编译

### Linux / WSL2

```bash
bash scripts/local-build.sh --install-deps   # 仅首次
THREADS=8 bash scripts/local-build.sh
```

```bash
ENABLE_WED=false LAN_IP=192.168.1.1 bash scripts/local-build.sh
```

只验证配置（拉源码、feeds、补丁、defconfig 和包校验，不编译）：

```bash
bash scripts/local-build.sh --config-only
```

### Windows + WSL2

```powershell
.\scripts\local-build.ps1 -InstallDeps   # 仅首次
.\scripts\local-build.ps1
```

```powershell
$env:ENABLE_WED = 'false'; $env:LAN_IP = '192.168.1.1'
.\scripts\local-build.ps1
```

`local-build.ps1` 会把 `ENABLE_*`、`LAN_IP`、`THREADS` 及下载镜像等变量转发进 WSL，并同步到 WSL 原生路径 `~/Auto-H5000M-BIN-localbuild` 编译（比 Docker / 跨文件系统快很多）。

| PowerShell 开关 | bash 选项 | 说明 |
| --- | --- | --- |
| `-InstallDeps` | `--install-deps` | apt-get 安装编译依赖 |
| `-PrepareOnly` | `--prepare-only` | 拉源码 + feeds + 补丁后停止 |
| `-ConfigOnly` | `--config-only` | 再执行 defconfig 和包校验后停止 |
| `-SkipToolchain` | `--skip-toolchain` | 跳过显式 `make toolchain/install` |
| `-SkipDownload` | `--skip-download` | 跳过 `make download` |
| `-SkipFeedsUpdate` | `--skip-feeds-update` | 跳过 `./scripts/feeds update -a` |

### 覆盖性配置测试

不完整编译，只跑到补丁、feeds、defconfig 和关键包校验：

```bash
bash scripts/coverage-test.sh quick
```

```bash
PROFILE_SET=full bash scripts/coverage-test.sh
```

`quick` = 默认配置 + 代理栈；`full` 额外覆盖最小系统、HomeProxy / MosDNS / Nikki 单独、原版 modem、QModem（next / lua）、**WED 关闭**、**Cloudflared 关 + HNAT 默认开**（`cloudflared-off`）、Adblock、可选服务、全兼容插件、DockerMan。覆盖测试能降低回归风险，但不能代替刷机后的实机验证。

---

## 已知问题

| 问题 | 说明 / 规避 |
| --- | --- |
| **Wi-Fi 必须用纯 WPA2（`psk2`）** | 该闭源驱动在 WPA2/WPA3 混合模式（`sae-mixed`）下，发往 Wi-Fi 终端的组播 / 广播全部丢失：IPv6 邻居发现失败、网页首次打开慢、mDNS / AirPlay / HomeKit 发现异常。改成 `psk2` 后恢复。改加密后 Apple 设备需"忽略此网络"再重连 |
| **改 Wi-Fi 不要用无参数 `wifi`** | 两个频段整体重启时切换加密方式，曾导致驱动卡死、全网断网，只能断电恢复。请在 LuCI 里改，或 `uci commit wireless; wifi reload`（只重载改动的射频） |
| Apple Mac 收不到 WMM UP 2 帧 | DSCP 16–23（CS2 / AF21–23）发往 Mac 的帧全部丢失；dropbear 的 SSH 用 AF21，所以 Mac 直连路由器 SSH 会超时。原因在闭源驱动内，未解决。规避：经有线主机跳板，或在路由器上把发往 Wi-Fi 的 DSCP 16–23 改写为 0 |
| **MediaTek HNAT 让 Wi-Fi 连接中途卡死** | 部分下行硬件转发条目写错，表现为"首次打开慢 / 失败，刷新就好"。默认 `ENABLE_HNAT=false`，详见 [HNAT 已知问题](#hnat--wed-已知问题连接中途卡死) |
| `h5000m.extra.config` 里的死锁自动重启配置无效 | `CONFIG_KERNEL_*_PANIC=y`、`CONFIG_CMDLINE="panic=10"` 在 OpenWrt 的配置系统里不存在，`make defconfig` 会静默丢弃；实际只编进了软 / 硬死锁和 hung task **检测器**，不会自动重启（panic 后的重启由 `kernel.panic=3` 负责）。需要的话应改为运行时 sysctl |
| 闭源驱动 | 驱动与芯片固件随上游 padavanonly 更新；Wi-Fi 相关问题只能等上游或换开源 mt76（主线 OpenWrt 已支持 H5000M，但目前有 Wi-Fi NAT 吞吐偏低的报告） |
| 默认 UPnP 开启 | WAN 有公网 IP 时注意；不需要可用 `ENABLE_UPNP=false` |

---

## 仓库结构

```
.
├── .github/workflows/build-test.yml   # GitHub Actions（调用 scripts/local-build.sh）
├── feeds.conf.default                 # feeds：immortalwrt packages/luci、openwrt routing/telephony、nikki、QModem
├── h5000m.extra.config                # 追加到 .config 的本机配置
├── patches/
│   ├── mtwifi-apcli-active-only.patch
│   └── mtk-hnat-local-dest.patch
└── scripts/
    ├── local-build.sh                 # 本地与 CI 共用的唯一构建入口
    ├── local-build.ps1                # Windows + WSL2 包装
    ├── coverage-test.sh               # 覆盖性配置测试
    └── check-deps.sh
```

---

## 许可证

继承上游 ImmortalWrt 项目许可证。
