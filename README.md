# 在 Linux / Kubuntu 上启用 LHDC v5 蓝牙音频编码指南

本指南记录了在 Linux（以 Kubuntu / Ubuntu 为例）环境下，通过 PipeWire 上游主线合并的 **Merge Request !2886** 为系统添加 **LHDC v5** 高清蓝牙音频支持的完整流程。

---

## 目录

- [在 Linux / Kubuntu 上启用 LHDC v5 蓝牙音频编码指南](#在-linux--kubuntu-上启用-lhdc-v5-蓝牙音频编码指南)
  - [目录](#目录)
  - [背景说明](#背景说明)
    - [LDAC vs LHDC 现状](#ldac-vs-lhdc-现状)
    - [与 AOSP 的关系](#与-aosp-的关系)
    - [PipeWire MR !2886](#pipewire-mr-2886)
  - [方案设计：独立覆盖层（Overlay）](#方案设计独立覆盖层overlay)
  - [完整操作步骤](#完整操作步骤)
    - [步骤 1：安装基础依赖与编译工具](#步骤-1安装基础依赖与编译工具)
    - [步骤 2：编译安装底层库 liblhdcv5](#步骤-2编译安装底层库-liblhdcv5)
    - [步骤 3：编译 PipeWire 主线 BlueZ 5 插件](#步骤-3编译-pipewire-主线-bluez-5-插件)
    - [步骤 4：部署独立 SPA 插件目录](#步骤-4部署独立-spa-插件目录)
    - [步骤 5：配置 WirePlumber 优先加载路径](#步骤-5配置-wireplumber-优先加载路径)
    - [步骤 6：重启服务并验证](#步骤-6重启服务并验证)
  - [连接与使用](#连接与使用)
  - [注意事项与限制](#注意事项与限制)
  - [卸载与恢复默认](#卸载与恢复默认)

---

## 背景说明

### LDAC vs LHDC 现状

- **LDAC**：在 modern Linux（如 Kubuntu 24.04+ / 26.10）中，官方仓库已内置 `libldacbt-enc2` 及 `libspa-codec-bluez5-ldac.so`，**开箱即用原生支持**，无需自行编译。
- **LHDC**：因专利与商业授权原因，官方发行版并未收录 LHDC 编码库。若耳机仅支持 LHDC 或需使用 LHDC v5 协议，必须手动补全底层算法库与 PipeWire 插件。

### 与 AOSP 的关系

LHDC 专利方（Savitech）未向 Linux 发行版提供官方开源驱动。Linux 社区能够实现该编码，是因为 Google 与厂商在较新的 **Android 开源项目（AOSP）** 中以 **Apache-2.0 协议** 开源了 LHDC v5 的 Rust 算法实现。

Linux 社区通过封装项目（如 `liblhdcv5`）提取该算法并包装为标准的 Linux 原生动态库（`.so`）与 `pkg-config` 接口，使桌面端 PipeWire 能够完成调用。

### PipeWire MR !2886

PipeWire 上游主线（Gitlab Freedesktop）已正式合并了由开发者提交的 **Merge Request !2886**（_bluez5: add LHDC V5 codec using liblhdcv5_），为 BlueZ 5 模块增加了对 `lhdcv5` 的原生 A2DP 编解码支持。

---

## 方案设计：独立覆盖层（Overlay）

为了防止直接将编译产物安装到 `/usr` 或覆盖系统已有的 PipeWire 文件引起后续 `apt upgrade` 冲突，我们采用 **Overlay（隔离覆盖层）** 方案：

- 将自行编译的 BlueZ 5 插件全套安装至专用目录：`/usr/lib/spa-0.2-lhdc/bluez5`。
- 通过 Systemd 环境变量覆盖（`SPA_PLUGIN_DIR`），让 WirePlumber 优先读取该专用目录，未命中时回退到系统的 `/usr/lib/x86_64-linux-gnu/spa-0.2`。
- 保证系统包管理器不受任何污染，可随时一键干净还原。

---

## 完整操作步骤

### 步骤 1：安装基础依赖与编译工具

更新软件源并安装构建所需的工具及库头文件：

```bash
sudo apt update
sudo apt install -y git meson ninja-build cargo rustc pkg-config \
    libbluetooth-dev libglib2.0-dev libdbus-1-dev libasound2-dev \
    libsbc-dev libfreeaptx-dev libldacbt-enc-dev libldacbt-abr-dev \
    liblc3-dev libopus-dev
```

---

### 步骤 2：编译安装底层库 liblhdcv5

拉取 AOSP 封装库并编译安装到系统：

```bash
git clone https://github.com/DBeidachazi/liblhdcv5.git /tmp/liblhdcv5
cd /tmp/liblhdcv5

# 编译
make

# 安装动态库与 pkg-config 描述文件
sudo make install
sudo ldconfig

# 验证安装（正常应返回版本号如 0.1.0）
pkg-config --modversion lhdcv5
```

---

### 步骤 3：编译 PipeWire 主线 BlueZ 5 插件

拉取包含 MR !2886 的主线 PipeWire 源码，仅构建 BlueZ 5 插件组（跳过无关组件以提升编译速度）：

```bash
git clone --depth=1 https://gitlab.freedesktop.org/pipewire/pipewire.git /tmp/pipewire
cd /tmp/pipewire

# 配置构建选项
meson setup build \
    --prefix=/usr \
    --buildtype=release \
    -Dauto_features=disabled \
    -Dbluez5=enabled \
    -Dbluez5-codec-lhdc=enabled \
    -Dbluez5-codec-ldac=enabled \
    -Dbluez5-codec-aptx=enabled \
    -Dbluez5-codec-lc3=enabled

# 编译所有配置的插件（约 10~20 秒完成）
ninja -C build
```

> **注意**：
> 如果之前已配置过 `build` 目录导致提示 `Directory already configured`，可直接执行 `ninja -C build`；如需修改编译参数，请执行 `meson setup --reconfigure build <选项>` 或删除 `build` 目录重建。

---

### 步骤 4：部署独立 SPA 插件目录

创建隔离目录并将编译出的所有 BlueZ 共享库拷贝至该目录下：

```bash
sudo mkdir -p /usr/lib/spa-0.2-lhdc/bluez5

# 拷贝包含 libspa-codec-bluez5-lhdc.so 等全部 bluez5 插件
sudo cp build/spa/plugins/bluez5/*.so /usr/lib/spa-0.2-lhdc/bluez5/
```

---

### 步骤 5：配置 WirePlumber 优先加载路径

为当前用户的 WirePlumber 服务创建配置覆盖：

```bash
mkdir -p ~/.config/systemd/user/wireplumber.service.d/

cat << 'EOF' > ~/.config/systemd/user/wireplumber.service.d/override.conf
[Service]
Environment=SPA_PLUGIN_DIR=/usr/lib/spa-0.2-lhdc:/usr/lib/x86_64-linux-gnu/spa-0.2
EOF
```

---

### 步骤 6：重启服务并验证

重载 Systemd 用户服务并重启音频管理服务：

```bash
systemctl --user daemon-reload
systemctl --user restart wireplumber pipewire pipewire-pulse
```

检查 PipeWire 是否已成功注册 LHDC endpoint：

```bash
pw-dump | grep -i "lhdc"
```

如果输出中包含类似 `api.bluez5.a2dp.lhdc` 的字样，说明插件加载成功。

---

## 连接与使用

1. 打开蓝牙并连接耳机。
2. **KDE 桌面环境**：
   - 打开 **系统设置 -> 声音 (System Settings -> Audio)**。
   - 选择已连接的蓝牙耳机。
   - 在 **音频配置 (Audio Profile / Codec)** 下拉菜单中选择 **LHDC v5**。
3. **命令行查看与切换**：

   ```bash
   # 查看当前卡片 profile 列表
   pactl list cards

   # 手动指定为 LHDC v5 播放模式
   pactl set-card-profile <bluez_card名称> a2dp-sink-lhdc_v5
   ```

---

## 注意事项与限制

1. **协议版本匹配**：
   - 本实现为 **LHDC v5** 规范。
   - 若耳机仅支持较早期的 **LHDC v3 / v4**（如初代小米降噪耳机、一加 Buds Pro 等），两者握手将无法成功，系统会自动回退至 LDAC、aptX 或 SBC。
2. **耳机端配置**：
   - 若耳机配套 App（手机端）中开启了“双设备连接/多点连接”，固件通常会强制禁用高规格传输协议。如遇无法切换，请在配套 App 中关闭多设备连接。
   - 确保耳机在手机端 App 中已开启“音质优先”模式。

---

## 卸载与恢复默认

如遇任何兼容性问题或希望完全恢复为系统原生状态，仅需两步：

```bash
# 1. 移除 WirePlumber 配置覆盖
rm -rf ~/.config/systemd/user/wireplumber.service.d/override.conf

# 2. 删除自定义插件目录
sudo rm -rf /usr/lib/spa-0.2-lhdc

# 3. 重启音频服务
systemctl --user daemon-reload
systemctl --user restart wireplumber pipewire pipewire-pulse
```
