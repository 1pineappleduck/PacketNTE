<div align="center">
  <img alt="LOGO" src="logo_packetnte.png" width="256" height="256" />
</div>

# PacketNTE

PacketNTE 是一个用于被动读取 NTE 角色原始世界坐标的 Python 扩展模块。

本项目仅发布编译处理后的二进制扩展文件，
核心解析逻辑不随仓库开源，仓库只提供运行产物、接口文档和调用示例。

## 功能简介

PacketNTE 提供 `nte_coordinate_api` 模块，用于在本机环境中被动读取角色世界坐标。

主要能力：

* 被动读取本机网络流量；
* 提取最近一次有效的角色世界坐标；
* 提供简单的 Python 调用接口；
* 支持指定网卡；
* 支持自定义 BPF 抓包过滤表达式；
* 返回原始三维世界坐标 `(x, y, z)`；
* 核心实现以 `.pyd` 二进制扩展形式发布。

## 非开源说明

PacketNTE 不是完整源码开源项目。

本仓库不会公开以下内容：

* 网络数据解析逻辑；
* 坐标字段识别规则；
* 内部偏移、结构验证或候选筛选逻辑；
* 混淆和加密前的源码；
* 私有构建流程。

仓库中发布的 `.pyd` 文件仅用于通过公开 API 调用。

请勿对二进制模块进行反编译、逆向分析、破解、二次打包或未经授权的再分发。

## 项目结构

推荐目录结构：

```text
PacketNTE/
├─ README.md
├─ docs/
│  └─ coordinate-api.md
├─ examples/
│  └─ basic.py
└─ thirdparty/
   └─ nte_coordinate_api.cp312-win_amd64.pyd
```

其中：

* `docs/coordinate-api.md`：网络坐标 API 详细文档；
* `examples/`：调用示例；
* `thirdparty/`：存放已编译的 `.pyd` 扩展模块。

如果你的 API 文档路径不同，请将 README 中的链接改成实际路径。

## API 文档

完整 API 说明请查看：

[网络坐标 API](coordinate-capture-api.md)

该文档包含：

* `CoordinateCapture` 的构造参数；
* `start()` 的行为说明；
* `read()` 的返回值与缓存时间说明；
* `close()` 的资源释放说明；
* 线程安全说明；
* 坐标连续性说明；
* 完整调用示例。

## 环境要求

### Python 版本

`.pyd` 文件与 Python 版本强绑定。

例如：

```text
nte_coordinate_api.cp312-win_amd64.pyd
```

表示该文件适用于：

```text
Python 3.12 x64
```

如果 Python 版本或系统架构不匹配，模块可能无法导入。

可以使用以下命令检查当前环境：

```bash
python --version
python -c "import platform; print(platform.architecture())"
```

### 运行依赖

PacketNTE 运行时需要可用的 Python 抓包环境。

安装 Python 依赖：

```bash
pip install scapy
```

Windows 用户通常还需要安装 Npcap：

```text
https://npcap.com/
```

安装 Npcap 时建议启用 WinPcap 兼容模式。

部分系统需要使用管理员权限运行程序，否则可能无法正常捕获网络流量。

## 快速开始

将对应 Python 版本的 `.pyd` 文件放入 `thirdparty` 目录。

示例：

```text
thirdparty/
└─ nte_coordinate_api.cp312-win_amd64.pyd
```

然后在 Python 中导入：

```python
import os
import sys
import time

ROOT = os.path.dirname(os.path.abspath(__file__))
THIRDPARTY = os.path.join(ROOT, "thirdparty")

if THIRDPARTY not in sys.path:
    sys.path.insert(0, THIRDPARTY)

from nte_coordinate_api import CoordinateCapture


capture = CoordinateCapture()

try:
    capture.start()

    while True:
        coordinate = capture.read(max_age=1.0)

        if coordinate is not None:
            x, y, z = coordinate
            print(f"\rx={x:.2f}, y={y:.2f}, z={z:.2f}", end="", flush=True)
        else:
            print("\rwaiting for coordinate...", end="", flush=True)

        time.sleep(0.1)

finally:
    capture.close()
```

## 基本用法

PacketNTE 对外提供的核心类是：

```python
from nte_coordinate_api import CoordinateCapture
```

典型生命周期：

```text
创建实例 → start() → 多次 read() → close()
```

最小示例：

```python
from nte_coordinate_api import CoordinateCapture

capture = CoordinateCapture()

try:
    capture.start()

    coordinate = capture.read(max_age=1.0)
    if coordinate is not None:
        x, y, z = coordinate
        print(x, y, z)

finally:
    capture.close()
```

更多接口细节请查看：

[网络坐标 API](coordinate-capture-api.md)

## 指定网卡

默认情况下，PacketNTE 使用系统抓包库的默认网卡。

如果需要指定网卡，可以传入 `interface`：

```python
capture = CoordinateCapture(interface="Ethernet")
```

不同系统、不同抓包驱动下的网卡名称格式可能不同。

如果默认网卡无法读取坐标，可以先枚举本机网卡，再选择实际承载游戏流量的网卡。

## 自定义过滤规则

默认过滤规则为：

```text
tcp port 30031 or udp
```

可以在创建实例时传入自定义 BPF 过滤表达式：

```python
capture = CoordinateCapture(packet_filter="udp")
```

如果调用方已经明确知道目标协议、端口或服务器地址，可以使用更精确的过滤规则，以减少无关数据包。

## 返回坐标说明

`read()` 返回的坐标格式为：

```python
(x, y, z)
```

类型为：

```python
tuple[float, float, float]
```

这些值是原始世界坐标。

PacketNTE 不负责：

* 世界坐标到地图坐标的转换；
* 地图缩放；
* 地图旋转；
* 坐标投影；
* 坐标平滑；
* 路径规划；
* UI 显示。

如果需要在地图或导航系统中使用坐标，请在上层业务中自行完成坐标标定和转换。

## 常见问题

### 1. 无法导入 `nte_coordinate_api`

请检查：

* `.pyd` 文件是否存在；
* `.pyd` 文件所在目录是否已经加入 `sys.path`；
* Python 版本是否匹配；
* 系统架构是否匹配；
* 是否缺少运行库或依赖 DLL。

### 2. `read()` 一直返回 `None`

可能原因包括：

* 尚未调用 `start()`；
* 游戏当前没有产生可识别的坐标数据；
* 抓包网卡选择错误；
* 没有足够的系统权限；
* Npcap 或抓包驱动不可用；
* 过滤规则过窄；
* 网络流正在切换；
* 角色处于加载、传送、切图或切换实例过程中。

`read()` 返回 `None` 不一定代表发生错误。
调用方应允许短时间内没有坐标，并等待后续数据恢复。

### 3. 是否需要管理员权限？

在 Windows 环境下，抓包通常需要管理员权限。

如果无法读取数据，建议先尝试使用管理员权限运行终端或程序。

### 4. 切图、传送或切换角色后坐标短暂中断怎么办？

这是正常情况。

PacketNTE 会尝试重新识别有效移动流。
调用方不应因为一次 `None` 就立即销毁实例，而应在业务层等待坐标恢复。

推荐逻辑：

```python
coordinate = capture.read(max_age=1.0)

if coordinate is None:
    # 暂时没有有效坐标，等待下一次读取
    return

x, y, z = coordinate
```

## 使用限制

PacketNTE 只提供被动坐标读取能力。

它不会：

* 主动发送网络数据；
* 修改网络数据包；
* 注入游戏进程；
* 修改游戏内存；
* 提供自动操作逻辑；
* 提供路径规划逻辑；
* 提供地图数据。

使用者应自行确认相关软件、平台或服务的用户协议。
请勿将 PacketNTE 用于违反规则、破坏公平性、商业作弊或其他不当用途。

## License

PacketNTE 核心模块为私有闭源组件。

未经授权，禁止：

* 反编译；
* 逆向分析；
* 修改后再发布；
* 二次分发；
* 用于违反平台规则的用途。

具体授权方式以发布页面或作者说明为准。
