# 网络坐标 API

`nte_coordinate_api` 提供被动读取 NTE 角色世界坐标的接口。

## 导入

```python
from nte_coordinate_api import CoordinateCapture
```

模块公开内容：

```python
__all__ = ("CoordinateCapture",)
```

## `CoordinateCapture`

```python
CoordinateCapture(
    interface: str | None = None,
    packet_filter: str = "tcp port 30031 or udp",
)
```

创建一个坐标捕获实例。构造实例不会启动捕获，需要随后调用 `start()`。

### 构造参数

#### `interface`

类型：

```python
str | None
```

默认值：

```python
None
```

指定用于捕获网络流量的网卡名称。

- 传入 `None` 时使用系统抓包库的默认网卡。
- 传入字符串时使用对应网卡。
- 网卡名称无效时，`start()` 可能抛出底层抓包异常。

示例：

```python
capture = CoordinateCapture(interface="Ethernet")
```

#### `packet_filter`

类型：

```python
str
```

默认值：

```python
"tcp port 30031 or udp"
```

指定捕获数据包时使用的 BPF 过滤表达式。

过滤范围越大，传入坐标解析器的数据包越多。除非调用方明确知道游戏使用的
协议和端口，否则建议保留默认值。

示例：

```python
capture = CoordinateCapture(packet_filter="udp")
```

## 方法

### `start()`

```python
start() -> None
```

启动后台网络捕获。

调用成功后，实例会持续解析网络数据，并保存最近一次有效的角色世界坐标。

行为：

- 非阻塞；
- 可重复调用；
- 已启动时再次调用不会创建新的捕获线程；
- 捕获过程在后台运行；
- 调用 `close()` 前会持续更新最新坐标。

可能抛出的异常包括：

| 异常 | 说明 |
| --- | --- |
| `RuntimeError` | 缺少必要的 Python 抓包依赖。 |
| `OSError` | 网卡、抓包驱动或 BPF 过滤器初始化失败。 |
| 其他底层异常 | 由网络捕获实现抛出。 |

建议在业务入口捕获异常：

```python
try:
    capture.start()
except Exception as exc:
    logger.error("无法启动坐标捕获：%s", exc)
```

### `read()`

```python
read(
    max_age: float = 1.0,
) -> tuple[float, float, float] | None
```

返回最近一次有效坐标。

#### `max_age`

类型：

```python
float
```

默认值：

```python
1.0
```

允许返回的坐标最大缓存时间，单位为秒。

设最近一次有效坐标的捕获时间为 `sample_time`，当前时间为 `now`：

```python
now - sample_time <= max_age
```

时返回该坐标；超过限制时返回 `None`。

`max_age` 不会被自动修正。传入 `0` 或负数时，通常不会返回缓存坐标。

常见取值：

| 值 | 用途 |
| --- | --- |
| `0.2` | 对坐标实时性要求较高。 |
| `1.0` | 默认值，适合一般导航。 |
| `2.0` | 容忍较短的网络数据间隔。 |

#### 返回值

有有效坐标时返回：

```python
(x, y, z)
```

类型为：

```python
tuple[float, float, float]
```

字段含义：

| 索引 | 名称 | 类型 | 说明 |
| --- | --- | --- | --- |
| `0` | `x` | `float` | 原始世界坐标 X。 |
| `1` | `y` | `float` | 原始世界坐标 Y。 |
| `2` | `z` | `float` | 原始世界坐标 Z，通常表示高度。 |

这些值是游戏网络数据中的原始三维坐标。API 不会对坐标进行平移、旋转、
缩放、投影或地图标定。

以下情况返回 `None`：

- 尚未调用 `start()`；
- 启动后尚未收到有效坐标；
- 最近一次坐标超过 `max_age`；
- 角色传送或切换实例时移动数据暂时中断；
- 网络流发生变化，新的移动流尚未确认；
- 当前数据包中没有可识别的坐标。

`read()` 是非阻塞方法。返回 `None` 不代表实例已停止，也不一定代表发生错误。

示例：

```python
coordinate = capture.read(max_age=0.5)

if coordinate is None:
    return

x, y, z = coordinate
```

### `close()`

```python
close() -> None
```

停止后台捕获并释放相关资源。

行为：

- 可重复调用；
- 未启动时调用是安全的；
- 已启动时会停止并等待后台捕获结束；
- 调用后实例不再接收新的坐标。

`close()` 不会主动清除最近一次缓存坐标，因此在缓存过期前调用 `read()` 仍可能
取得关闭前的最后一个坐标。

应使用 `try/finally` 确保资源被释放：

```python
capture = CoordinateCapture()

try:
    capture.start()
    # 使用坐标
finally:
    capture.close()
```

## 完整示例

```python
import time

from nte_coordinate_api import CoordinateCapture


capture = CoordinateCapture(
    interface=None,
    packet_filter="tcp port 30031 or udp",
)

try:
    capture.start()

    while True:
        coordinate = capture.read(max_age=1.0)
        if coordinate is not None:
            x, y, z = coordinate
            print(f"x={x:.2f}, y={y:.2f}, z={z:.2f}")

        time.sleep(0.1)
finally:
    capture.close()
```

## 状态与生命周期

实例没有公开状态属性。调用方应根据方法结果判断当前状态：

| 操作结果 | 含义 |
| --- | --- |
| `start()` 正常返回 | 后台捕获已启动或此前已经启动。 |
| `start()` 抛出异常 | 捕获未能正常启动。 |
| `read()` 返回坐标 | 当前存在未过期的有效坐标。 |
| `read()` 返回 `None` | 当前没有满足实时性要求的有效坐标。 |
| `close()` 正常返回 | 捕获资源已经释放。 |

推荐生命周期：

```text
创建实例 → start() → 多次 read() → close()
```

## 线程安全

- 后台捕获线程负责更新最新坐标；
- `read()` 可以从其他线程调用；
- 对最新坐标的读取和更新具有同步保护；
- 不建议多个线程同时调用 `start()` 或 `close()`；
- 一个实例只应由一个业务组件管理生命周期。

## 坐标连续性

正常移动时，坐标会随角色位置持续更新。

以下操作可能导致短暂返回 `None`：

- 传送；
- 切换地图或实例；
- 切换角色；
- 网络重连；
- 游戏移动时间戳重置；
- 游戏切换到新的网络流。

接口会尝试重新识别有效移动流。调用方不应因为一次 `None` 立即销毁并重建实例，
而应根据业务需要等待后续坐标恢复。

## 使用限制

- 该接口只读取网络流量，不发送或修改数据包；
- 调用方需要具备访问指定网卡的系统权限；
- 系统需要提供可用的抓包驱动；
- 返回的是原始世界坐标，不保证与任何地图像素坐标系直接一致；
- 坐标转换和标定应由调用方实现。
