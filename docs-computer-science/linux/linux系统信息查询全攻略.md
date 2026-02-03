# Linux系统信息查询全攻略

## 🖥️ 一、操作系统基本信息

| 命令 | 作用 |
|------|------|
| `uname -a` | 显示内核版本、主机名、架构等综合信息 |
| `cat /proc/version` | 查看内核版本及编译信息（含 GCC 版本） |
| `cat /etc/issue` 或 `cat /etc/os-release` | 查看发行版欢迎信息或标准 OS 信息（推荐后者） |
| `cat /etc/redhat-release`（仅 RHEL/CentOS） | 查看 Red Hat 系发行版具体版本 |
| `hostname` | 查看当前主机名 |

---

## ⚙️ 二、CPU 信息

### 1. 基础信息
| 命令 | 说明 |
|------|------|
| `lscpu` | **推荐**：汇总 CPU 架构、核心数、线程数、型号等 |
| `cat /proc/cpuinfo` | 详细列出每个逻辑 CPU 的参数 |

### 2. 关键指标提取
- **物理 CPU 个数**：
  ```bash
  cat /proc/cpuinfo | grep "physical id" | sort -u | wc -l
  ```
- **每颗物理 CPU 的核心数**：
  ```bash
  cat /proc/cpuinfo | grep "cpu cores" | uniq
  ```
- **逻辑 CPU 总数**：
  ```bash
  cat /proc/cpuinfo | grep "processor" | wc -l
  ```
- **CPU 型号**：
  ```bash
  cat /proc/cpuinfo | grep "model name" | cut -d: -f2 | uniq -c
  ```

> 💡 公式：  
> **总逻辑 CPU = 物理 CPU 数 × 每颗核心数 ×（1 或 2，若开启超线程）**

- **系统位数（运行模式）**：
  ```bash
  getconf LONG_BIT   # 显示当前运行在 32 位还是 64 位模式
  uname -m           # 显示架构（如 x86_64 表示 64 位）
  ```

---

## 💾 三、内存信息

| 命令 | 作用 |
|------|------|
| `free -m` 或 `free -h` | 查看内存与交换分区使用情况（MB/人类可读） |
| `cat /proc/meminfo` | 查看详细内存信息（总量、空闲、缓存等） |
| `grep MemTotal /proc/meminfo` | 仅显示总内存 |
| `grep MemFree /proc/meminfo` | 仅显示空闲内存 |

---

## 💿 四、磁盘与分区

| 命令 | 作用 |
|------|------|
| `lsblk` | 列出所有块设备（磁盘、分区、挂载点），含依赖关系 |
| `fdisk -l` | 查看磁盘分区表（需 root 权限） |
| `df -h` | 查看各挂载分区的使用情况（人类可读） |
| `du -sh <目录>` | 查看指定目录占用空间 |
| `swapon -s` | 查看启用的交换分区 |
| `mount | column -t` | 查看当前挂载的文件系统 |
| `hdparm -i /dev/sda` | 查看 IDE/SATA 磁盘参数（旧设备） |
| `cat /proc/partitions` | 查看内核识别的分区列表 |

---

## 🌐 五、网络信息

| 命令 | 作用 |
|------|------|
| `ifconfig` 或 `ip a` | 查看网络接口配置（IP、MAC 等） |
| `ifconfig -a` | 显示所有网络接口（包括未启用的） |
| `ethtool eth0` | 查看网卡速率、双工模式等详细参数 |
| `lspci | grep -i eth` | 查看网卡硬件型号 |
| `route -n` | 查看路由表 |
| `netstat -lntp` | 查看监听的 TCP 端口及对应进程 |
| `netstat -antp` | 查看所有 TCP 连接 |
| `netstat -s` | 查看网络协议统计信息 |
| `iptables -L` | 查看防火墙规则（需权限） |
| `cat /proc/net/dev` | 查看网卡收发数据统计 |

---

## 🧠 六、系统运行状态

| 命令 | 作用 |
|------|------|
| `uptime` | 查看系统运行时间、负载、用户数 |
| `cat /proc/uptime` | 获取系统运行秒数（用于计算启动时间） |
  - **计算启动时间**：
    ```bash
    date -d "$(awk -F. '{print $1}' /proc/uptime) second ago" +"%Y-%m-%d %H:%M:%S"
    ```
  - **格式化运行时长**：
    ```bash
    cat /proc/uptime| awk -F. '{run_days=$1 / 86400;run_hour=($1 % 86400)/3600;run_minute=($1 % 3600)/60;run_second=$1 % 60;printf("系统已运行：%d天%d时%d分%d秒",run_days,run_hour,run_minute,run_second)}'
    ```
| `cat /proc/loadavg` | 查看系统平均负载（1/5/15 分钟） |

---

## 📦 七、进程与服务

| 命令 | 作用 |
|------|------|
| `ps -ef` | 列出所有进程 |
| `top` / `htop` | 实时监控进程资源占用 |
| `w` | 查看当前登录用户及活动 |
| `last` | 查看历史登录记录 |
| `id <user>` | 查看用户 UID/GID 所属组 |
| `cut -d: -f1 /etc/passwd` | 列出所有用户 |
| `cut -d: -f1 /etc/group` | 列出所有用户组 |
| `crontab -l` | 查看当前用户的定时任务 |
| `systemctl list-units --type=service --state=running` | （现代系统）查看运行中的服务 |
| `chkconfig --list` | （旧版 SysV）查看服务启停状态 |
| `chkconfig --list | grep on` | 仅查看开机自启服务 |

---

## 📦 八、软件包信息

| 命令 | 作用 |
|------|------|
| `rpm -qa` | （RHEL/CentOS）列出所有已安装 RPM 包 |
| `dpkg -l` | （Debian/Ubuntu）列出所有已安装 deb 包 |
| `lsmod` | 查看已加载的内核模块 |
| `lspci -tv` | 以树状图列出 PCI 设备 |
| `lsusb -tv` | 以树状图列出 USB 设备 |
| `lspci -v` 或 `-vv` | 查看详细 PCI 设备信息 |

---

## 📂 九、/proc 目录关键文件速查

| 文件 | 说明 |
|------|------|
| `/proc/cpuinfo` | CPU 详细信息 |
| `/proc/meminfo` | 内存使用详情 |
| `/proc/version` | 内核与 GCC 版本 |
| `/proc/uptime` | 系统运行时间（秒） |
| `/proc/loadavg` | 系统负载 |
| `/proc/swaps` | 交换分区使用情况 |
| `/proc/mounts` | 当前挂载的文件系统 |
| `/proc/partitions` | 磁盘分区信息 |
| `/proc/net/dev` | 网络接口流量统计 |
| `/proc/[pid]/` | 各进程的运行时信息（如 cmdline, status, fd 等） |
| `/proc/self` | 指向当前进程的符号链接 |

---

## ✅ 总结建议

- **日常快速查看系统版本**：`cat /etc/os-release`
- **全面了解 CPU**：优先用 `lscpu` + `cat /proc/cpuinfo`
- **排查性能问题**：结合 `top`、`free`、`df`、`uptime`、`netstat`
- **自动化脚本中**：优先读取 `/proc` 下的原始文件（如 `/proc/meminfo`），避免依赖外部命令输出格式变动

---

这份总结可作为 **Linux 系统信息查询速查手册**，建议收藏或打印备用。


---

> **声明**：本文部分内容由人工智能（AI）辅助生成，经人工整理与校对，旨在提供准确、实用的技术参考。如有疑问，欢迎交流指正。

