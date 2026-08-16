# TideOS

基于 FreeBSD 源码深度定制的桌面操作系统分发版。名字 TideOS，寓意版本如潮汐般规律涨落、稳定更迭。

## 产品定位

**第一阶段（当前，Alpha 之前）：最优雅、最稳定的桌面 BSD。**
先把桌面体验、更新机制、品牌做扎实，成为"开箱即用的 FreeBSD 桌面"。

**第二阶段（Alpha 之后）：兼容层副标题——"潮起处，三海汇流"。**
FreeBSD 是月亮（看不见的驱动力），托起两股潮：

- **Linux 兼容层**：基于内核 Linuxulator（与 drm-kmod 显卡驱动同源），一键安装 Ubuntu 用户态；CLI/服务场景承诺稳定，GUI 标注实验性
- **Windows 兼容层**：一键 Wine + DXVK（Vulkan 驱动由 drm-kmod 提供），面向游戏与常用应用
- **Android 明确不做子系统**（依赖 binder/ashmem 等 Linux 内核特性，FreeBSD 上无兼容层路径），仅提供 bhyve 虚拟机模板作为高级用户工具

兼容层包装为 `tideos-windows` / `tideos-linux` 一键包，等桌面主线立住后作为差异化卖点，不提前透支信任。

## 仓库说明

| 分支 | 来源 | 用途 |
|---|---|---|
| `main` | freebsd-src main（16.0-CURRENT） | 仅作文档承载（README），不用于构建 |
| `tideos/15` | 从 upstream `stable/15` 拉出（HEAD `9de55e815b0`） | 发行版定制分支，全部自定义修改收敛于此 |

## 当前进度

| 里程碑 | 状态 | 说明 |
|---|---|---|
| M1 环境搭建 | ✅ 完成 | 本地 KVM 构建机（FreeBSD 15.1-RELEASE，12 vCPU / 24GB / 128GB ZFS）；`tideos/15` 分支零定制构建通过：buildworld + buildkernel + installworld/installkernel；重启后运行自建系统（`15.1-STABLE tideos/15-n284866`） |
| M2 最小定制 ISO | 🔨 进行中 | 品牌 v1.0 定稿（极简潮线/深海蓝/月光白）；`TIDEOS` 内核配置编译通过；loader 开机画面 + 登录欢迎语已打入 bsdinstall；release ISO 构建中 |
| M3 桌面集成 | ⏳ 待开始 | Xorg/Wayland + Plasma/Xfce + 显卡 + WiFi + 音频 |
| M4 poudriere 仓库 + 安装器 | ⏳ 待开始 | BSDINSTALL_PKGS + postinstall 定制 |
| M5 发布流水线 + pkgbase | ⏳ 待开始 | 自动化发布 + base 更新机制 |
| M6 中文本地化 + 硬件测试 | ⏳ 待开始 | 首个公开 Alpha |

构建机访问：`ssh root@127.0.0.1 -p 2222`（本机 KVM 虚拟机，密钥登录）。

参考同类项目：GhostBSD（桌面、Xfce/MATE）、NomadBSD（live 桌面）、MidnightBSD（macOS 风格桌面）、helloSystem（精简桌面）、TrueNAS（深度定制但服务器方向）。建议先研究 GhostBSD 的开源仓库做法。

---

## 1. 技术选型与前提

| 项目 | 选择 | 说明 |
|---|---|---|
| 基础版本 | FreeBSD stable/15（当前稳定版） | 发行版基线必须用 stable/releng 分支，绝不能用 main（-CURRENT 开发版）；stable/14 已入维护期可作备选 |
| 构建机 | FreeBSD 15.x，amd64 | release 构建必须同架构本地构建（chroot 机制），不支持交叉构建 |
| 构建机配置 | ≥8 vCPU / 16GB 内存 / 200GB 磁盘（ZFS） | buildworld+buildkernel 数小时级，磁盘需给 /usr/obj 留 ≥50GB |
| 内核图形 | drm-kmod（Intel/AMD 开源）、nvidia-driver（N 卡闭源） | FreeBSD 桌面显卡的关键依赖 |
| 桌面环境 | KDE Plasma 6 或 Xfce 4.20（推荐） | Plasma 更现代，Xfce 在 FreeBSD 上最成熟；GNOME 亦可但移植工作略多 |
| 显示服务器 | Wayland 为主 + XWayland | 15.x 之后 drm-kmod + KDE Wayland 可用 |
| 软件包仓库 | 自建 pkg 仓库（poudriere 构建） | 深度定制必须自建，否则官方 pkg 装不了专属定制包 |
| 更新机制 | pkgbase（推荐） | 把 base 系统也打成 pkg，base+ports 统一用 pkg 更新 |

## 2. 源码仓库与分支策略

```sh
git clone https://github.com/0xaicrypto/tideos.git ~/tideos-src
cd ~/tideos-src
git remote add upstream https://git.freebsd.org/freebsd-src.git
git fetch upstream stable/15
git checkout -b tideos/15 upstream/stable/15   # 自定义修改全部在这个分支
git push -u origin tideos/15
```

> 注意：GitHub fork 默认只有 `main`（16.0-CURRENT，开发版），发行版基线从 `stable/15` 拉出 `tideos/15` 分支，`main` 不用管（可删可留）。

策略要点：

- 自定义修改**全部收敛在 tideos/15 分支**，绝不直接改 upstream 分支
- 每次跟踪上游：`git fetch upstream && git merge upstream/stable/15`
- 修改用独立 commit 打标签（`tideos-branding`、`tideos-kernel`、`tideos-installer`），方便合并时冲突定位
- 定期用 `git log upstream/stable/15..HEAD` 审查自己的改动面，越少越容易维护

## 3. 深度定制清单

### 3.1 内核层

| 文件 | 修改内容 |
|---|---|
| `sys/amd64/conf/TIDEOS` | 内核配置：`include GENERIC` 开头，用 `nooptions`/`nodevice` 裁剪，加 `options`（如 `KSTACK_PAGES`、`HWPMC`） |
| `sys/sys/param.h` | OSVERSION、FreeBSD_version 改为自有版本号（影响 uname -v、模块兼容性） |
| `sys/kern/` 相关 | 加自己的 sysctl 树（如 `kern.tideos.*`） |
| 内核模块 | 需要随发行版分发的第三方模块编译进 KERNCONF |

构建：
```sh
make -j$(sysctl -n hw.ncpu) buildkernel KERNCONF=TIDEOS
```

### 3.2 用户态与品牌

| 位置 | 内容 |
|---|---|
| `etc/defaults/rc.conf` | 默认开机服务（如默认开启 powerd、dbus、桌面相关服务） |
| `etc/rc.d/` | 新增发行版专属 rc 脚本（首次启动初始化、桌面会话引导） |
| `share/skel/` | 新用户默认 dotfiles（.profile、.xprofile 等） |
| `etc/motd` / rc.d/motd | 登录欢迎信息与品牌文案 |
| `release/` | RELEASENAME、release 脚本中的发行版名 |
| `sys/boot/` | loader 品牌：`loader_logo` 自定义 logo、splash 图（`splash_bmp_load`） |
| `usr.sbin/bsdinstall/` | 安装器（见 3.4） |
| `etc/installerconfig` 模板 | 无人值守安装配置样例 |

### 3.3 桌面系统集成（Phase 2 核心）

依赖：`xorg` + `graphics/drm-kmod`（Intel/AMD）、`graphics/nvidia-driver`（NVIDIA），`graphics/firmware-*`（GPU 固件），`misc/fbsd-nvlist`（N 卡模块自动加载）。

桌面栈选型（推荐 A 方案）：

- **A：KDE Plasma 6**：`plasma6-plasma-desktop` + `x11/sddm` + `plasma6-plasma-nm`（网络）+ `plasma6-plasma-pa`（音频）+ `x11-toolkits/qt6-wayland`
- **B：Xfce**：`x11-wm/xfce4` + `x11/lightdm` + 组件更轻，成熟度最高
- **C：GNOME**：`x11/gnome` + `x11/gdm`，FreeBSD 上适配工作量最大

周边：音频 `audio/pipewire` + OSS（snd_hda）；触控板/输入用默认 libinput；电源管理 `sysutils/powerd`；WiFi 依赖 Intel `iwlwifi`/`iwm`、高通 `ath`、MTK `mt76` 固件包；浏览器 firefox/chromium；截图、剪贴板等桌面工具。

中文本地化（如目标用户为中文用户）：`fonts/noto-cjk`、`x11-fm/...` 输入法 `textproc/fcitx5`（含 `fcitx5-qt`、`fcitx5-gtk`）、`zh_CN.UTF-8` 默认 locale 设置。

### 3.4 安装器定制（fork bsdinstall）

`usr.sbin/bsdinstall/` 是 shell + dialog 脚本结构，定制点：

1. **分区**：`partedit` 默认分区方案（如 ZFS + boot/efi 自动布局）
2. **镜像源**：`mirrorselect` 指向自建 pkg 仓库
3. **安装后钩子**：`scripts/postinstall.d/` 增加桌面初始化脚本（装显卡驱动、启用服务、建桌面用户）
4. **预装包**：bsdinstall 支持 `BSDINSTALL_PKGS` 环境变量，安装时自动从仓库拉取桌面 meta 包（furyBSD/GhostBSD 同款做法）
5. **品牌**：欢迎界面、产品文案、默认键盘/时区

### 3.5 软件包仓库（poudriere）

```sh
pkg install poudriere
sysrc poudriere_enable=YES
poudriere jail -c -j 15amd64 -v 15.1-RELEASE -a amd64
poudriere ports -c -p default
# 定制选项：/usr/local/etc/poudriere.d/15amd64-make.conf
#   OPTIONS_SET/OPTIONS_UNSET、DEFAULT_VERSIONS、本地 patch 目录
poudriere bulk -j 15amd64 -p default -f /usr/local/etc/poudriere.d/pkglist
poudriere pkgrepo -j 15amd64 -p default          # 生成仓库
pkg repo /usr/local/poudriere/data/packages/15amd64-default  # 签名
```

自建仓库后，安装器与系统 `pkg.conf` 均指向它；发布时用 `pkg audit` + vuln.xml 跟踪安全漏洞。

### 3.6 更新机制

**推荐 pkgbase**：`make packages` 把 base 系统打成 .pkg，与自建 ports 仓库合并为统一仓库。用户升级 = `pkg upgrade`，一条命令覆盖内核、base、应用。需注意：

- pkgbase 在 15 上比 14 成熟，落地前先在 dev 环境充分验证
- 备选方案：fork `usr.sbin/freebsd-update` 自建二进制更新服务（工作量大，不推荐起步阶段做）

## 4. 构建与发布流水线

### 4.1 开发环境首次构建

```sh
cd ~/tideos-src
make -j16 buildworld
make -j16 buildkernel KERNCONF=TIDEOS
```

### 4.2 Release 镜像构建

```sh
cd ~/tideos-src/release
make release RELEASENAME=TideOS-1.0-RELEASE \
     NODOC=YES NOPORTS=YES   # 产出在 /usr/obj/release/
```

- 产出：disc1/dvd1 ISO、memstick.img（U 盘）、bootonly，以及 MANIFEST + 校验和
- 自定义镜像内容：改 `release/amd64/mkisoimages.sh`（拷入品牌文件、logo、预配置）
- 签名：release 构建用 `SIGNATURE_TYPE=gpg` + `SIGNING_KEY`（官方流程在隔离机签名），或先用 `NOKEY` 关闭，品牌成熟后再启用

### 4.3 发布流水线（自动化建议）

1. 上游 merge → 全量构建（CI 机上）→ 冒烟测试（bhyve/qemu 启动 + 桌面登录）
2. 通过后 poudriere 全量构建 packages
3. 打 ISO / memstick，签名，生成校验和
4. 推送镜像站 + pkg 仓库 + 更新 manifest
5. 发布公告与安全跟踪（订阅 freebsd-security-notifications，自建 SA 页面）

## 5. 测试策略

| 层 | 手段 |
|---|---|
| 冒烟 | bhyve（FreeBSD 自带）或 qemu 启动 ISO，验证安装全流程、首次启动、桌面可达 |
| 硬件矩阵 | Intel/AMD GPU、Intel WiFi、Realtek 网卡、笔记本（触摸板/电源/挂起）逐项过 |
| 回归 | 上游 merge 后跑相同测试套件；CI 用 GitHub Actions + qemu |
| 中文验证 | 字体、输入法、locale 全链路 |

## 6. 里程碑时间线（单人全职估算）

| 阶段 | 内容 | 周期 |
|---|---|---|
| M1 | 环境搭建：构建机、git 分支、首次 buildworld | 1–2 周 |
| M2 | 最小定制 ISO：品牌 + rc.conf + loader logo，可装机 | 2–3 周 |
| M3 | 桌面集成：Xorg/Wayland + Plasma/Xfce + 显卡 + WiFi + 音频 | 3–5 周 |
| M4 | poudriere 自建仓库 + 安装器定制（BSDINSTALL_PKGS + postinstall） | 3–4 周 |
| M5 | 发布流水线 + pkgbase 更新机制 | 4–6 周 |
| M6 | 中文本地化、硬件矩阵测试、首个公开 Alpha | 3–4 周 |

合计约 4–6 个月到首个 Alpha；之后进入长期维护模式（上游跟踪 + 安全响应）。

## 7. 风险与应对

| 风险 | 应对 |
|---|---|
| 上游 merge 冲突累积 | 自定义面收敛到少量独立 commit；每月小步 merge 而非半年大 merge |
| WiFi/GPU 等硬件支持缺口 | 发布前明确硬件兼容矩阵；备选旧内核模块（如 nvidia 旧驱动） |
| pkgbase 不够成熟 | 先在 dev 环境验证；不行则先用"base 走发行版镜像更新 + 应用走 pkg"过渡 |
| 内核参数改动导致模块 ABI 破坏 | 尽量用 loader.conf/sysctl 调优而非改源码；保留 GENERIC 兼容模块集 |
| 长期维护人力 | 尽早自动化 CI；文档化构建流程；吸引社区贡献 |
| 法律 | FreeBSD 许可证允许衍生发行版，保留版权声明（share/misc/bsd-family-tree、COPYRIGHT）即可 |

## 8. 启动下一步

1. 起名 + 定仓库（git 托管 + 构建机）
2. M1：装好 FreeBSD 构建机，`git clone` + 首次 `make buildworld` 跑通
3. 以原版 FreeBSD 安装 ISO 为基线，先做出 M2 的最小定制镜像，建立版本节奏
