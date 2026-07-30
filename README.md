# 海康设备发现与管理工具

这是一个跨平台的 Go 命令行工具，用于通过海康 SADP 协议和 ARP 扫描发现局域网设备，并执行基础设备查询与 SADP 管理操作。

## 功能

- SADP 多播发现：获取设备 IP、MAC、序列号、设备类型、固件版本等信息。
- ARP 网段扫描：发现不响应 SADP 的海康设备，至少返回 IP 和 MAC。
- 联合扫描：合并 ARP 与 SADP 结果并按 MAC 去重。
- XML、CSV 输出：便于导入其他系统处理。
- ISAPI 设备信息查询：支持 HTTP Digest 认证。
- SADP 命令：查询、激活、修改网络参数、重启等。
- 旧版固件密码重置码生成。

## 环境要求

- Go `1.26` 或更高版本。
- 扫描设备时，电脑应与设备处于同一局域网或 VLAN。
- SADP 使用 UDP 多播地址 `239.255.255.250:37020`。
- ARP 扫描依赖本机 ARP 表，不能跨路由发现设备。

## 编译运行

在项目目录执行：

```bash
# 查看帮助
go run ./cmd/sadp --help

# 编译当前平台
go build -buildvcs=false -o bin/sadp ./cmd/sadp
./bin/sadp --help
```

Windows PowerShell 下编译：

```powershell
go build -buildvcs=false -o bin\sadp.exe ./cmd\sadp
.\bin\sadp.exe --help
```

如果使用 Make：

```bash
make ENVIRONMENT=local build
```

## 设备发现

### SADP 自动发现

不需要提前知道设备 IP，程序会在所有已启用的 IPv4 网卡上发送探测，并绑定固定 UDP `37020` 端口，以兼容旧款设备和官方 SADP 工具行为。

```bash
./sadp discover:sadp --timeout 10s --debug
```

### ARP 网段扫描

ARP 扫描需要提供网段。例如本机地址是 `192.168.0.150`、掩码是 `255.255.255.0` 时，扫描 `192.168.0.0/24`：

```bash
./sadp discover --timeout 2s 192.168.0.0/24
```

ARP 能发现设备的 IP 和 MAC，但无法直接提供序列号、设备类型或固件版本。

### 联合扫描

联合执行 ARP 与 SADP，并按 MAC 地址合并结果：

```bash
./sadp scan --timeout 2s 192.168.0.0/24
```

输出 XML：

```bash
./sadp scan --xml 192.168.0.0/24
./sadp scan --xml --output devices.xml 192.168.0.0/24
```

`--xml` 输出为纯 XML。只通过 ARP 找到的设备会包含 IP/MAC，SADP 找到的设备会包含完整设备字段。

注意：命令行选项应放在网段参数之前，例如使用 `scan --xml 192.168.0.0/24`。

## ISAPI 查询

标准 ISAPI 设备信息接口通常需要账号密码。推荐使用环境变量，避免密码进入 Shell 历史记录：

```bash
export HIKVISION_USERNAME=admin
export HIKVISION_PASSWORD='设备密码'
./sadp isapi info 192.168.0.117
```

JSON 输出：

```bash
./sadp isapi info 192.168.0.117 --json
```

HTTPS 自签名证书：

```bash
./sadp isapi info https://192.168.0.117 --insecure
```

没有账号密码时，能否读取完整设备信息取决于固件配置。ARP 只能返回 IP/MAC；SADP 或 ONVIF 发现可能返回部分信息，但不能绕过设备认证获取受保护的序列号。

## 其他命令

```bash
# 探测常用 HTTP 端点
./sadp probe 192.168.0.117

# 查看 SADP 命令列表
./sadp send --list

# 生成旧版固件（低于 5.3.0）密码重置码
./sadp reset --serial 0123456789 --date 20231215
```

只对自己拥有或获授权管理的设备执行扫描和配置操作。

## 配置项

| 环境变量 | 默认值 | 说明 |
| --- | --- | --- |
| `DISCOVERY_WORKERS` | `100` | ARP 并发扫描数量 |
| `DISCOVERY_TIMEOUT` | `1s` | 单个地址探测超时 |
| `SADP_TIMEOUT` | `5s` | SADP 等待响应时间 |
| `HTTP_TIMEOUT` | `10s` | HTTP 请求超时 |
| `HIKVISION_USERNAME` | 空 | ISAPI 用户名 |
| `HIKVISION_PASSWORD` | 空 | ISAPI 密码 |
| `OUTPUT_DIR` | `data` | 输出目录 |
| `DEBUG` | `false` | 是否输出调试日志 |

示例：

```bash
DISCOVERY_WORKERS=50 SADP_TIMEOUT=10s ./sadp scan --debug 192.168.0.0/24
```

## Ubuntu 发布包

项目的 `dist` 目录提供：

- `sadp-linux-amd64.tar.gz`：适用于 `x86_64`。
- `sadp-linux-arm64.tar.gz`：适用于 `aarch64`。

Ubuntu 上确认架构并解压：

```bash
uname -m
tar -xzf sadp-linux-amd64.tar.gz
cd sadp-linux-amd64
chmod +x sadp
./sadp --help
```

## 开发检查

```bash
make ENVIRONMENT=local test
make ENVIRONMENT=local vet
make ENVIRONMENT=local check
```

项目结构：

```text
cmd/sadp             命令行入口
internal/cli         命令解析与业务流程
internal/sadp        SADP 协议实现
internal/network     ARP、HTTP 与网络工具
internal/isapi       ISAPI Digest 客户端
internal/crypto      密码重置码生成
internal/config      环境变量配置
```

## 许可证

本项目使用 MIT 许可证，详见 [LICENSE](LICENSE)。
