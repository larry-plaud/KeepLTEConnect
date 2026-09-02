# KeepLTEConnect

> 一款用于自动化 RF 测试场景的 Windows 桌面小工具：持续监控被测设备（DUT）在 **R&S CMW500** 上的 LTE 分组域连接状态，一旦掉线便通过 **SIM Simulator** 串口自动执行恢复流程，把设备重新拉回附着状态，无需人工值守。

[![.NET](https://img.shields.io/badge/.NET-10.0--windows-512BD4)](https://dotnet.microsoft.com/)
[![Platform](https://img.shields.io/badge/platform-Windows-0078D6)](#运行环境)
[![UI](https://img.shields.io/badge/UI-WPF-1D4ED8)](#)

---

## 目录

- [背景与用途](#背景与用途)
- [核心特性](#核心特性)
- [工作原理](#工作原理)
- [软件架构](#软件架构)
- [运行环境](#运行环境)
- [快速开始](#快速开始)
- [编译与发布](#编译与发布)
- [界面使用说明](#界面使用说明)
- [关键参数（固定，无 UI 开关）](#关键参数固定无-ui-开关)
- [通信协议说明](#通信协议说明)
- [项目结构](#项目结构)
- [注意事项](#注意事项)

---

## 背景与用途

在长时间的自动化射频测试中，被测终端与综测仪（CMW500）之间的 LTE 连接偶尔会意外掉线，导致测试用例中断、需要人工干预重连。**KeepLTEConnect** 作为一个常驻的“看门狗”程序，自动完成以下闭环：

**监控连接状态 → 检测到掉线 → 通过 SIM Simulator 下发恢复指令 → 等待重新附着 → 恢复监控**

从而保证测试可以无人值守地长时间运行。

## 核心特性

- **SCPI over TCP**：直接通过裸 TCP（默认 5025 端口）与 CMW500 通信，**无需安装 NI-VISA**，只需填写标准 VISA 资源字符串。
- **PSW 状态轮询**：周期性查询 LTE 信令的分组域（Packet-Switched）连接状态，并归一化为 `OFF / ON / ATT / CEST`。
- **防抖判定**：连续多次采样确认掉线后才触发恢复，避免瞬时抖动误触发。
- **自动恢复序列**：通过 SIM Simulator 串口下发一组恢复 shell 命令，重新激活 SIM 仿真并触发终端重新附着。
- **UART / USB 自动协商**：串口握手时自动识别 SIM Simulator 使用的是 UART 还是 USB-CDC 帧格式；COM 口也支持 `Auto` 自动扫描。
- **私有帧协议实现**：内置 CRC-16/CCITT-FALSE 校验 + 魔数分帧的收发/解析逻辑，字节级对齐参考实现。
- **启动自检**：程序启动时对协议实现做 CRC、帧结构、JSON 转义等自检，任一不符即弹窗并退出，杜绝向 SIM Simulator 发送错误帧。
- **深色主题 WPF 界面**：内置状态指示灯、实时日志窗口，操作直观。

## 工作原理

主监控循环（见 `MainWindow.xaml.cs` → `WatchLoopAsync`）：

1. **连接 CMW500**：解析 VISA 地址，建立 TCP 连接，读取 `*IDN?` 确认设备身份。
2. **轮询状态**：每 1 秒查询一次
   `FETCh:LTE:SIGNaling1:PSWitched:STATe?`，归一化结果。
   - `ATT`（已附着）或 `CEST`（已建立连接）视为 **连接正常**。
   - 其它状态（如 `OFF`）计一次“坏样本”。
3. **触发恢复**：坏样本连续达到 `DebounceSamples`（默认 2）次，进入恢复流程 `DoRecoveryUntilReattachAsync`：
   - 打开 SIM Simulator 串口（每次都重新打开，因为 `cfun` 复位可能导致 COM 口重新枚举）。
   - 依次下发恢复命令：
     ```text
     cd uart3_proxy
     send AT+ECSIMCFG="SimSimulator",1
     send at+cfun=1
     ```
   - 进入 **Wait Reattach** 状态，最长等待 `ReattachTimeoutSec`（默认 60 秒），期间持续轮询 PSW 状态。
   - 若在超时前恢复为 `ATT/CEST`，返回正常监控；否则重试整个恢复流程。
4. **异常自愈**：SCPI 连接出错时，外层循环会自动重连。

状态机对应的界面状态：`Idle → Connecting… → Monitoring → Recovering → Wait Reattach → Reconnecting → Stopped`。

## 软件架构

分层清晰，各司其职：

| 层 | 文件 | 职责 |
| --- | --- | --- |
| 应用入口 | `App.xaml` / `App.xaml.cs` | 启动、全局深色主题资源、协议自检 |
| 界面 + 编排 | `MainWindow.xaml` / `MainWindow.xaml.cs` | UI 绑定、监控循环与恢复流程编排、日志 |
| SCPI 传输 | `Scpi/ScpiTcpClient.cs` | 裸 TCP SCPI 客户端、VISA 解析、PSW 查询与归一化 |
| 私有帧协议 | `Protocol/SimFrame.cs` | CRC16、UART/USB 帧构建与解析、启动自检 |
| 串口传输 | `Protocol/SimSerialClient.cs` | SIM Simulator 串口打开、握手、shell 命令收发 |

## 运行环境

- **操作系统**：Windows（WPF，`net10.0-windows`）。
- **.NET SDK**：.NET 10 SDK（编译时需要）。发布版为**自包含单文件**，目标机器无需预装 .NET 运行时。
- **硬件**：
  - R&S CMW500 综测仪（网络可达，SCPI 5025 端口开放）。
  - SIM Simulator 设备（通过 COM 串口连接，波特率 230400）。
- **依赖包**：`System.IO.Ports` 9.0.0（NuGet 自动还原）。

## 快速开始

```bash
# 克隆后进入项目目录
cd KeepLTEConnect

# 还原依赖并运行（Debug，框架依赖模式，便于快速迭代）
dotnet run
```

运行后：

1. 在 **⚡ CMW500** 中填写 VISA 地址（下拉框内置常用预设，可编辑）。
2. 在 **🔌 SIM Simulator** 中选择 COM 口（或选 `Auto` 自动扫描），必要时点 **🔄 Refresh** 刷新串口列表。
3. 点击 **▶ Start** 开始监控，**◼ Stop** 停止。

## 编译与发布

**Debug**（框架依赖，便于开发调试）：

```bash
dotnet build
```

**Release**（自包含、单文件、ReadyToRun、压缩）：

```bash
dotnet publish -c Release -r win-x64
```

Release 配置在 `.csproj` 中已做发布加固（单文件 / 自包含 / 去符号 / 压缩），生成的 `KeepLTEConnect.exe` 可直接拷贝到目标机运行。

> 项目还内置了一个发布配置文件 `Properties/PublishProfiles/FolderProfile.pubxml`，目标为 `win-x86` 单文件自包含，输出到指定文件夹，可在 Visual Studio 中直接“发布”。

## 界面使用说明

界面自上而下分为四块：

- **⚡ CMW500** — 填写 / 选择 VISA 资源地址（格式见下）。
- **🔌 SIM Simulator** — 选择 COM 口，`Auto` 表示自动遍历所有串口尝试握手。
- **▶ Watch** — Start / Stop 按钮；右侧显示彩色状态灯、当前状态文字及 **PSW** 实时状态。
- **📝 Log** — 带时间戳的运行日志，自动滚动到底部；缓冲区超过约 200 KB 会自动截断前半段。

VISA 地址格式（大小写不敏感，端口可省略，默认 5025）：

```text
TCPIP0::<ip>[::<port>]::INSTR
例如：TCPIP0::172.29.0.3::INSTR
```

## 关键参数（固定，无 UI 开关）

以下参数在源码中以常量形式固定（`MainWindow.xaml.cs`）：

| 参数 | 值 | 含义 |
| --- | --- | --- |
| `LteInstance` | `1` | LTE 信令实例号（SCPI 查询中的 `SIGNaling1`） |
| `PollIntervalSec` | `1.0` | 状态轮询间隔（秒） |
| `DebounceSamples` | `2` | 触发恢复前需连续检测到的“坏样本”数 |
| `ReattachTimeoutSec` | `60` | 单次恢复后等待重新附着的最长时间（秒） |
| `CooldownSec` | `3` | 串口握手失败后的冷却重试间隔（秒） |

如需修改，请直接改动源码常量后重新编译。

## 通信协议说明

### SCPI（`ScpiTcpClient`）

- 传输：裸 TCP，默认端口 5025，`NoDelay`，操作超时 5 秒。
- 命令以 `\n` 结尾发送；查询按行（`\n` 结束、忽略 `\r`）读取响应。
- 主要查询：`*IDN?`、`FETCh:LTE:SIGNaling<n>:PSWitched:STATe?`。
- 状态归一化：全 / 短形式统一为 `OFF / ON / ATT / CEST`，未知值原样上报到日志。

### SIM Simulator 私有帧（`SimFrame` / `SimSerialClient`）

- 串口参数：**230400** 波特、8N1、无握手、DTR/RTS 打开。
- 帧结构：`FA FB FC FD` 魔数 + 16 字节头 + JSON 载荷，**CRC-16/CCITT-FALSE**（初值 `0xFFFF`，`crc16("123456789") == 0x29B1`）。
- 两种载体：
  - **UART 帧**：16 字节头（魔数、版本、消息长度、消息 CRC、头 CRC）+ JSON。
  - **USB-CDC 帧**：额外 16 字节 USB 头包裹 UART 帧。
- 载荷 JSON：心跳（`cmdType=0`）与 shell 命令（`cmdType=0xFFFF`）。JSON 采用宽松转义（`UnsafeRelaxedJsonEscaping`），保证 `+ = < > "` 等字符与参考实现**字节级一致**。
- 握手：打开串口后发心跳帧，收到合法响应即认为该协议（UART/USB）可用；`Auto` 模式先试 UART 再试 USB。

## 项目结构

```text
KeepLTEConnect/
├─ App.xaml / App.xaml.cs          # 应用入口、深色主题资源、启动自检
├─ MainWindow.xaml / .xaml.cs      # 主界面 + 监控/恢复编排逻辑
├─ AssemblyInfo.cs                 # 程序集主题信息
├─ Protocol/
│  ├─ SimFrame.cs                  # 私有帧协议：CRC16、帧构建/解析、自检
│  └─ SimSerialClient.cs          # SIM Simulator 串口客户端（握手、收发）
├─ Scpi/
│  └─ ScpiTcpClient.cs            # 裸 TCP SCPI 客户端
├─ Properties/PublishProfiles/    # 发布配置
├─ KeepLTEConnect.csproj          # 项目文件（含构建时间戳生成 Target）
├─ KeepLTEConnect.slnx            # 解决方案
└─ KeepLTEConnect.ico             # 应用图标
```

## 注意事项

- **构建有效期保护**：`.csproj` 会在每次编译时生成 `BuildStamp`（构建时刻的 UTC 时间戳），程序在启动监控时会据此校验构建是否过期（约 360 天）。过期后 **Start 将不再启动**，需重新编译发布最新版本。
- **恢复命令为固定序列**：当前恢复流程针对特定 DUT / SIM Simulator 场景硬编码（`cd uart3_proxy` 等），换用其它设备前请核对并按需修改 `RecoveryCommands`。
- **串口重枚举**：`cfun` 复位可能导致 SIM Simulator 的 COM 口号变化，因此每轮恢复都会重新扫描 / 打开串口；建议优先使用 `Auto` 模式。
- 本工具用于**授权范围内的实验室自动化测试**环境。
