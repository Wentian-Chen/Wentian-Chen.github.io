# 深入理解 ROS2

ROS2 在 2022 年发布 LTS 版本 Humble（对应 Ubuntu 22.04）之后，生态已经相当成熟。作为一个研究具身智能的学生，我明显感觉到：学术界新的开源项目（机械臂抓取、视觉语言导航、仿真训练）都在往 ROS2 迁移。这篇笔记是我从 ROS1 转到 ROS2 时整理的核心知识框架，从概念到代码一次讲清。

## 引言

### ROS2 是什么

ROS2 是 ROS（Robot Operating System）的下一代版本，它保留了 ROS1"节点-话题-服务"的编程心智模型，但通信底层完全重写：**ROS2 用 DDS（Data Distribution Service）取代了 ROS1 的 TCP/UDP 直连**。DDS 是一个工业级的分布式通信标准，它自带服务发现机制——节点不需要中央服务器（ROS1 的 roscore），彼此之间可以自动发现、自动组网。

### 为什么是 DDS

DDS 带来的最直观好处有三个：**去中心化**——没有 roscore 单点故障，节点之间通过服务发现自动组网，任何节点挂掉不影响整个系统；**QoS 可配置**——原生支持"可靠/尽力而为"等通信策略，按场景选择；**多机即天然**——同一局域网内只需 ROS_DOMAIN_ID 一致即可直接通信。

### ROS2 与 ROS1 的关键差异

| 维度 | ROS1 (Noetic) | ROS2 (Humble) |
| ---- | ---- | ---- |
| 通信架构 | 中心化，依赖 roscore | 去中心化，DDS 自动发现 |
| 构建工具 | catkin_make / catkin build | colcon |
| Python | Python2/3 混用 | 仅 Python3 |
| 消息接口 | msgs/srv 文件 | 统一为 .msg/.srv/.action，用 rosidl 生成 |
| 生命周期 | 无统一概念 | 引入生命周期节点（未配置/未激活/激活/销毁） |
| 实时性 | 不保证 | 支持（executor、DDS 实时配置） |
| 参数服务器 | 集中式 rosparam | 分布式，每个节点持有自己的参数 |

!!! note "说明"
    表格里的"生命周期"指 ROS2 的 Lifecycle Node：节点可在未配置/未激活/激活等状态之间显式切换，方便系统做受控启停（导航中的规划器、控制器就是生命周期节点），这是 ROS1 完全没有的机制。

## 核心概念

ROS2 的所有编程都围绕下面五个概念展开，理解它们就理解了 ROS2 的 90%。

### 节点 Node

节点是可独立运行的最小执行单元，一个机器人的程序通常由几十个节点组成，每个节点负责一件事。节点之间通过话题、服务、动作通信，彼此不直接调用。

```bash
ros2 node list          # 查看当前所有节点
ros2 node info /talker  # 查看某个节点的详细信息（话题、服务、动作、参数）
```

### 话题 Topic

话题实现**异步、单向、多对多**的通信：发布者（publisher）把消息写到话题上，订阅者（subscriber）按自己的频率去读。适合传感器数据流（激光、图像、里程计）这种持续高频的通信。

```bash
ros2 topic list              # 查看所有话题
ros2 topic echo /chatter     # 实时打印某个话题的消息
ros2 topic info /chatter     # 查看话题的消息类型和收发双方
```

### 服务 Service

服务实现**同步、请求-响应**的通信：客户端发起请求并阻塞等待服务器的应答。适合"问一句、答一句"的场景，比如查询机器人状态、设置参数。

```bash
ros2 service list            # 查看所有服务
ros2 service type /xxx       # 查看服务类型
ros2 service call /xxx 接口名 "{参数}"  # 直接调用服务
```

### 动作 Action

动作是**异步、带反馈**的长任务通信：客户端请求一个目标，服务器一边执行一边周期性地发布反馈（feedback），最终返回结果（result）。适合导航、机械臂运动这类耗时任务——"去某个点"既要能持续汇报进度，又要能随时取消。

```bash
ros2 action list        # 查看所有动作
ros2 action info /xxx   # 查看动作类型
```

### 参数 Parameter

参数是节点的配置项，本质是一个键值对，每个节点拥有自己的一组参数。运行时可以通过命令行修改，也可以写成 yaml 文件在启动时加载。

```bash
ros2 param list              # 查看所有节点的参数
ros2 param get /node 参数名   # 读取参数
ros2 param set /node 参数名 值 # 修改参数（运行时可改）
```

!!! tip "技巧"
    用 `ros2 node info <node>` 查看节点是理解一个 ROS2 系统最快的方法，它会一次性列出这个节点的所有话题、服务、动作和参数。

## 通信与 QoS

### 两种通信模型

ROS2 通信分两大类：**发布-订阅（pub/sub）**，即话题，消息单向流动、发布者与订阅者完全解耦，适合传感器数据流；**请求-响应（req/resp）**，即服务，一问一答、同步阻塞，适合查询与指令类一次性交互。需要持续反馈、可取消的长任务则用动作（Action）。

### QoS（服务质量）

QoS 决定一次通信的可靠性承诺，最常用的两个策略是 **RELIABLE** 和 **BEST_EFFORT**：RELIABLE 保证消息不丢（发送方重传直到确认），适合控制指令、关键状态；BEST_EFFORT 尽力而为、延迟更低，适合高频传感器数据（比如点云），因为下一帧马上就到。

!!! warning "注意"
    QoS 不匹配的双方是**无法通信**的（话题建立了但收不到数据）。最常见的坑：点云传感器的 BEST_EFFORT 与订阅方的默认 RELIABLE 不匹配。如果 `ros2 topic echo` 没数据但 `ros2 topic list` 能看到话题，优先检查 QoS。

## 工程结构

### 工作空间

ROS2 的工作空间（workspace）根目录下固定有四个目录：

```text
my_ws/
├── src/      # 源码：所有功能包都放在这里
├── build/    # 构建中间产物（CMake 缓存等），可删除重建
├── install/  # 编译产物和安装环境，编译后需要 source 这里
└── log/      # 编译日志
```

### colcon build

ROS2 用 `colcon` 构建工作空间，常用命令：

```bash
colcon build                                   # 构建 src 下所有包
colcon build --packages-select my_pkg         # 只构建指定包
colcon build --packages-select my_pkg --symlink-install  # Python 包推荐，改代码免重编译
source install/setup.bash                     # 编译后必须 source 才能使用新包
```

!!! danger "踩坑"
    `colcon build` 之后**一定要** `source install/setup.bash`（最好写入 ~/.bashrc）。很多"明明编译了却 ros2 run 不到"的报错，都是没 source 新工作空间导致的。另外注意：ROS2 环境里 ROS1 和 ROS2 的 setup 不能同时 source，否则环境变量会互相污染。

### 功能包结构

一个功能包至少包含 `package.xml`（元信息）和构建文件。按语言分两种：

- **C++ 包**：构建文件是 `CMakeLists.txt`，用 `ament_cmake` 声明依赖和可执行文件。
- **Python 包**：构建文件是 `setup.py`，通过 `entry_points` 注册入口函数，不需要 `setup.py` 里的 install_requires 额外写 ROS 依赖（写在 package.xml）。

```text
my_pkg/
├── package.xml   # 包名、版本、依赖声明（两类包都需要）
├── CMakeLists.txt  # C++ 包：ament_cmake 构建脚本
├── setup.py      # Python 包：注册可执行入口
├── src/          # C++ 源码目录
└── my_pkg/       # Python 源码目录
```

package.xml 里 `<depend>rclpy</depend>` 这样的标签声明依赖，编译系统会自动处理。

## 示例代码

下面用 rclpy 写一个完整的发布-订阅例子：talker 节点每秒在 `chatter` 话题上发一条消息，listener 节点接收并打印。这是 ROS2 的 hello world，代码严格按官方教程的标准写法。

### 发布者 talker.py

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class Talker(Node):
    def __init__(self):
        super().__init__('talker')
        # 创建一个发布者：话题 chatter，消息类型 String，队列长度 10
        self.publisher_ = self.create_publisher(String, 'chatter', 10)
        # 创建一个定时器：每 1.0 秒回调一次
        self.timer = self.create_timer(1.0, self.timer_callback)
        self.count = 0

    def timer_callback(self):
        msg = String()
        msg.data = f'Hello ROS2: {self.count}'
        self.publisher_.publish(msg)
        self.get_logger().info(f'Publishing: {msg.data}')
        self.count += 1


def main(args=None):
    rclpy.init(args=args)      # 初始化 rclpy
    talker = Talker()          # 创建节点
    rclpy.spin(talker)         # 阻塞式运行，持续处理回调
    talker.destroy_node()      # 退出时销毁节点
    rclpy.shutdown()           # 关闭 rclpy
```

### 订阅者 listener.py

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class Listener(Node):
    def __init__(self):
        super().__init__('listener')
        # 创建订阅者：话题 chatter，收到消息时调用回调
        self.subscription = self.create_subscription(
            String, 'chatter', self.listener_callback, 10)

    def listener_callback(self, msg):
        self.get_logger().info(f'I heard: {msg.data}')


def main(args=None):
    rclpy.init(args=args)
    listener = Listener()
    rclpy.spin(listener)
    listener.destroy_node()
    rclpy.shutdown()
```

!!! tip "技巧"
    `rclpy.spin()` 是 rclpy 的事件循环：节点收到的所有回调（订阅、定时器、服务）都由它调度。回调里**不要做耗时操作**，否则会阻塞其他回调，这是 rclpy 编程的第一条铁律。

运行方式：两个终端各开一个（确保都已 source），分别执行 `python3 talker.py` 和 `python3 listener.py`，就能看到订阅者实时打印发布者发出的消息。

## Launch 文件

ROS2 的 launch 文件是 Python 脚本，用来一键启动一组节点并配置参数，替代 ROS1 的 xml launch。基本结构是：定义 `generate_launch_description()` 函数，返回 `LaunchDescription`，里面是一个个 `Node()` 声明。

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(
            package='my_pkg',          # 功能包名
            executable='talker',       # 可执行文件名（Python 包对应 entry point）
            name='talker',             # 节点名（覆盖代码里的名字）
            output='screen',           # 日志输出到终端
            parameters=[{'count_max': 100}],  # 传入参数
        ),
        Node(
            package='my_pkg',
            executable='listener',
            output='screen',
        ),
    ])
```

启动方式：

```bash
ros2 launch my_pkg my_launch.py
```

!!! note "说明"
    launch 文件放在包的 `launch/` 目录下，并要在 `CMakeLists.txt`（C++）或 `setup.py`（Python）里把 launch 目录安装出去，`ros2 launch` 才能找到它。

## tf2 坐标变换

### 基本概念

机器人身上有无数个坐标系：地图系 map、机器人本体 base_link、激光雷达 laser、机械臂末端 ee……tf2 就是管理这些坐标系之间变换关系的库。每个变换都是**父坐标系（parent frame）到子坐标系（child frame）**的平移+旋转，所有变换构成一棵严格单根的 TF 树（每个 frame 只能有一个父）。

典型的一棵 TF 树：

```text
map (世界/地图系)
└── odom (里程计系)
    └── base_link (机器人本体)
        └── laser (激光雷达)
        └── camera_link (相机)
```

### 静态变换与命令行示例

机器人本体和传感器之间的变换是固定的（传感器装在机器人上不动），用静态变换发布：

```bash
# 参数：x y z yaw pitch roll frame_id child_frame_id
ros2 run tf2_ros static_transform_publisher 0.1 0 0.2 0 0 0 base_link laser
```

这个命令发布"laser 在 base_link 坐标系下 (0.1, 0, 0.2) 处、无旋转"的静态变换。其他传感器（相机、imu）同理。

调试 TF 树：

```bash
ros2 run tf2_ros tf2_echo base_link laser   # 查看两个 frame 之间的实时变换
ros2 run tf2_tools view_frames              # 生成 TF 树的 PDF 报告
```

!!! warning "注意"
    发布 TF 前要先想清楚谁是谁的父系：**父子关系决定了变换树的结构**，只能按"世界→车体→传感器"这种层级来发，不能乱挂。传感器挂错父系是 TF 报错的常见原因之一。

## 常用调试命令

把下面这些命令记熟，排查 ROS2 问题基本够用：

```bash
ros2 node list                     # 所有节点
ros2 node info /node_name          # 单个节点的完整信息

ros2 topic list                    # 所有话题
ros2 topic echo /topic             # 实时查看话题消息
ros2 topic info /topic             # 话题类型与参与者
ros2 topic pub /topic 类型 "{...}" # 手动发布一条消息（测试很有用）

ros2 service list                  # 所有服务
ros2 service call /srv 类型 "{...}" # 手动调用服务

ros2 action list                   # 所有动作
ros2 action info /action           # 动作类型与参与者

ros2 param list                    # 所有参数
ros2 param get /node 参数名         # 读参数
ros2 param set /node 参数名 值      # 写参数

rqt_graph                          # 可视化话题/节点通信图，图形化 top 级调试工具
rviz2                              # 3D 可视化，机器人调试标配
```

!!! tip "技巧"
    一切异常从 `ros2 node list` 和 `ros2 topic list` 开始排查：先确认节点到底起来没有、话题到底存在没有，再往下查 QoS、TF、参数。很多问题在第一步就现形了。

## 常见问题

### 找不到节点/话题

`ros2 node list` 里什么都没有，或者 `ros2 run` 提示包不存在，按顺序检查：

1. 是否 source 了正确的环境：`source /opt/ros/humble/setup.bash`，以及你自己的 `source ~/my_ws/install/setup.bash`。
2. 是否编译过且编译成功：`colcon build --packages-select 你的包`。
3. 三个终端是否都在**同一个 ROS_DOMAIN_ID** 下（见下条）。

### DDS 多机通信需要相同 ROS_DOMAIN_ID

ROS_DOMAIN_ID 相当于通信的"频道号"，不同 ID 的节点互相看不见。多机协作时，所有机器必须：

- 设置相同的 `ROS_DOMAIN_ID`（比如都设 0，或都设 42）。
- 处于同一局域网，且能互相访问（检查防火墙，DDS 需要 UDP 多播）。

排查命令：`echo $ROS_DOMAIN_ID`，两端不一致时 `export ROS_DOMAIN_ID=0` 统一即可。

### colcon 未 source

这是出现频率最高的一类问题：编译完新包后，`ros2 run` 报错找不到。原因几乎总是**没有 source install/setup.bash**，或者 source 了但终端不是当前 shell。建议把 source 写进 `~/.bashrc`：

```bash
source /opt/ros/humble/setup.bash
source ~/my_ws/install/setup.bash   # 你的工作空间要放这一行，且不能是 ROS1 的
```

### 其他高频坑

- **QoS 不匹配**：话题看得到、echo 没数据，检查 Reliability 策略（见上文 QoS 部分）。
- **环境污染**：ROS1 和 ROS2 的 setup.bash 不要同时 source，终端里的 `ROS_DISTRO` 要一致。

ROS2 的核心知识到这里就覆盖全了。接下来最好的学习方式是动手：写一个包、跑通 talker/listener、加上 launch 文件、接上 rviz2 看坐标变换，把这篇笔记里的概念全部过一遍。
