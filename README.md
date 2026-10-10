# cups-cnijfilter

基于 Docker 的 CUPS 打印服务，内置佳能（Canon）喷墨打印机官方驱动（`cnijfilter`），旨在为老旧打印机赋予现代网络打印及 AirPrint 局域网共享能力。

当前默认预装驱动型号：**Canon PIXMA MP280 系列 (MP280 / MP288)**。

---

## 🎯 项目背景与特点

许多老旧的佳能打印机（如 MP280 系列）虽然硬件耐用，但在较新的 Linux 发行版或 NAS 系统（如群晖、Unraid、Armbian 等）上安装官方驱动极其繁琐，核心痛点在于：
- 官方驱动（`cnijfilter`）发布年份较早，强依赖已废弃的旧版本动态链接库（如 `libtiff.so.4`、`libpng12.so.0`）。
- 现代 Linux 系统升级后缺失对应依赖，导致打印筛选器（Filter: `pstocanonij`、`cifmp280`）静默崩溃或提示 `Filter failed`。

本项目基于成熟的 [olbat/cupsd](https://github.com/olbat/dockerfiles/tree/master/cupsd) 镜像构建，在容器内自动补全旧版动态链接库软链接，实现开箱即用的网络打印服务。

### ✨ 特性

- 🖨️ **驱动集成**：预装 Canon 官方 Linux 驱动（`cnijfilter-common` 与 `cnijfilter-mp280series`）以及专属过滤程序。
- 🔗 **依赖修复**：Dockerfile 中自动检索并动态软链接 `libtiff` 与 `libpng`，彻底解决 `Filter failed` 报错。
- 🌐 **局域网免驱共享**：支持通过 CUPS 协议将 USB 打印机共享至局域网，支持 Windows、macOS、iOS (AirPrint) 与 Android。
- 📦 **轻量灵活**：提供 Dockerfile 与 Docker Compose 配置文件，便于快速部署与迁移。

---

## 📂 项目结构

```text
.
├── build/
│   ├── Dockerfile                                 # 镜像构建定义
│   └── cnijfilter-mp280series-3.40-1-deb.tar.gz   # 佳能官方驱动安装包
├── docker-compose.yml                             # 容器编排配置
├── LICENSE                                        # 开源许可证
└── README.md                                      # 说明文档
```

---

## 🚀 快速上手

### 1. 前置准备

- 宿主机系统：Linux（Ubuntu / Debian / 群晖 / OpenWrt / PVE / Raspberry Pi OS 等）。
- 已安装 **Docker** 与 **Docker Compose**。
- 将打印机通过 USB 线连接至宿主机，并开机。
- （可选）在宿主机上运行 `lsusb`，确认系统已识别到 Canon 打印设备：
  ```bash
  lsusb | grep -i canon
  ```

### 2. 构建镜像

克隆本仓库后，进入项目根目录执行构建：

```bash
docker build -t cups-mp280:latest ./build
```

### 3. 准备持久化目录与配置

在启动容器前，建议先创建数据卷目录，避免 Docker 将文件挂载误创建为文件夹：

```bash
mkdir -p spool
touch cupsd.conf
```

> **提示**：如果需要自定义 CUPS 配置，可以在容器首次启动后将默认配置导出：
> ```bash
> docker cp cupsd:/etc/cups/cupsd.conf ./cupsd.conf
> ```

### 4. 启动服务

使用 Docker Compose 启动：

```bash
docker compose up -d
```

也可以使用直接运行的 Docker 命令：

```bash
docker run -d \
  --name cupsd \
  --restart unless-stopped \
  --net host \
  --privileged \
  -v /var/run/dbus:/var/run/dbus \
  -v /dev/bus/usb:/dev/bus/usb \
  -v $(pwd)/spool:/var/spool/cups \
  cups-mp280:latest
```

---

## 🖨️ 打印机配置与管理

### 1. 登录 CUPS Web 管理后台

打开浏览器，访问：
```text
http://<宿主机IP>:631
```

- 默认管理账号：`print`
- 默认密码：`print`

### 2. 添加打印机

1. 进入导航栏的 **Administration**（管理）页面。
2. 点击 **Add Printer**（添加打印机）。
3. 在 **Local Printers**（本地打印机）列表中选择识别出的 USB 佳能打印机（例如 `Canon MP280 series`）。
4. 设置打印机名称（Name）和描述（Description），并务必勾选 **Share This Printer**（共享此打印机）。
5. 在驱动（Make / Model）选择界面：
   - Make 选择 **Canon**。
   - Model 列表中选择预装的 **`Canon MP280 series ver.3.40`**（或带 cnijfilter 字样的对应型号）。
6. 点击 **Add Printer** 保存配置。
7. 在打印机管理页面中点击 **Maintenance** -> **Print Test Page**，验证测试页能否正常打印。

---

## 📱 客户端连接

| 客户端系统 | 连接方式 |
| :--- | :--- |
| **Windows** | 打开「设置」->「蓝牙和其他设备」->「打印机和扫描仪」->「添加设备」。若未自动发现，可选择「按名称选择共享打印机」，输入 `http://<宿主机IP>:631/printers/<打印机名称>`。 |
| **macOS / iOS** | 由于启用了 `--net host` 与宿主机 D-Bus/Avahi，Apple 设备通常可通过 **AirPrint / Bonjour** 自动发现并直接打印。 |
| **Android** | 可使用系统自带的「默认打印服务」或下载 CUPS / Mopria 兼容打印插件，添加服务器 IP 即可。 |

---

## ❓ 常见问题排查 (FAQ)

### 1. 打印任务提示 `Filter failed`？
佳能官方驱动依赖 32/64 位旧版动态库。本项目在 `build/Dockerfile` 中已自动处理软链接：
- `libtiff.so.4` -> 宿主环境现存 `libtiff.so.*`
- `libpng12.so.0` -> 宿主环境现存 `libpng*.so.*`

如果更改基础镜像或跨架构构建，可在容器内执行 `ldd /usr/lib/cups/filter/cifmp280` 查看是否仍有 `not found` 的依赖库。

### 2. CUPS 后台无法发现 USB 打印机？
- 确认 `docker-compose.yml` 中配置了 `privileged: true` 以及挂载了 `/dev/bus/usb`。
- 在宿主机终端检查打印机连接：`lsusb`。
- 重启打印机电源或重新插拔 USB 线。

### 3. 局域网设备无法发现打印机？
- 容器必须使用 `network_mode: host` 模式，确保 mDNS / Bonjour 广播包能直接在局域网内传输。
- 确认宿主机防火墙放行了 TCP `631` 端口及 UDP `5353` 端口（mDNS）。

---

## 🛠️ 扩展支持其他佳能型号

如需支持其他型号的佳能打印机（例如 MP230、MP250、iP2700、E510 等）：

1. 从佳能官方支持网站下载对应型号的 Linux `.deb` 驱动压缩包（通常命名格式如 `cnijfilter-<model>series-x.xx-x-deb.tar.gz`）。
2. 将驱动包放入 `build/` 目录。
3. 修改 [build/Dockerfile](build/Dockerfile)：
   - 更新 `COPY` 语句中的文件名。
   - 调整 `tar` 解包及 `dpkg -i` 的包名模式。
   - 调整对应过滤程序（如 `/usr/lib/cups/filter/cif<model>`）的执行权限。
4. 重新构建镜像即可。

---

## 📄 开源许可证

本项目基于 [MIT License](LICENSE) 开源。
佳能官方驱动包版权归 Canon Inc. 所有，使用时请遵循厂商相应软件许可协议。