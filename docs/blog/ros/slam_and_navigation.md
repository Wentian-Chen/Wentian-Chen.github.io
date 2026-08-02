# SLAM 与 Navigation：移动机器人的定位导航全流程

研究移动机器人避不开两个词：SLAM 和 Navigation。前者解决"我是谁、我在哪"，后者解决"我要去哪、怎么去"。这篇文章按"建图 → 定位 → 导航"的完整链路，把这两个方向的核心概念和 ROS 生态梳理一遍。内容以经典 ROS1 实现为主（gmapping、AMCL、move_base），同时覆盖 ROS2 的对应方案（slam_toolbox、Nav2）。

## 引言

移动机器人导航可以概括为三个经典问题：

- **我是谁**：机器人是什么形态，轮式、履带还是四足？运动学模型如何？（通常由底盘驱动和运动学标定回答）
- **我在哪**：定位问题。我在地图的哪个位置、朝向哪里？
- **我要去哪**：目标点在哪，怎么规划一条无碰撞的路径走过去？

其中"我在哪"是核心难点：真实环境里没有 GPS（室内），也没有人告诉我初始位置，我必须只靠自己的传感器和运动模型，在一张（可能还没建好的）地图里持续估计自己的位姿。这就是 SLAM 和定位要解决的问题。

## SLAM 基础

### 什么是 SLAM

SLAM（Simultaneous Localization and Mapping，同时定位与建图）指机器人在未知环境中一边移动，一边**用传感器数据同时估计自己的位姿（定位）和构建环境地图（建图）**。两者互相依赖：定位需要地图作为参照，建图又需要准确的位姿把传感器观测拼在一起——所以叫"同时"。

### 里程计 odom

里程计（odometry）是机器人在**自身坐标系**下估计的相对位移：轮式机器人根据轮子转速积分得到移动量，配上 IMU 的姿态融合，就得到从起点开始"走了多远、转了多大角度"。它高频（几十到几百 Hz）但不绝对——**漂移不可避免**，越走偏差越大。SLAM 和定位的作用之一，就是用传感器观测（激光/视觉）去修正里程计的累积误差。

### 激光雷达 vs 视觉

- **激光雷达（Lidar）**：直接测量距离，精度高、不受光照影响，2D 激光建图（gmapping/cartographer）成熟稳定，是轮式机器人导航的标配。缺点是贵，且 2D 激光只能给出一个平面的信息。
- **视觉（Camera）**：信息丰富、成本低，能做稠密重建和语义理解，适合三维环境（ORB-SLAM、RTAB-Map）。缺点是对光照、纹理敏感，位姿估计的鲁棒性不如激光。

!!! note "说明"
    工程上常把两者融合：激光负责定位建图，视觉负责识别与避障。学术上视觉 SLAM 是热门方向，但工业移动机器人大多仍以激光为主。

### 滤波派 vs 图优化派

实现 SLAM 的主流方法可以分成两派：

- **滤波派**：以扩展卡尔曼滤波（EKF）和粒子滤波为代表，在线地、增量式地估计位姿。gmapping 就是粒子滤波的代表，实现直观、实时性好，但误差会随时间累积，适合中小场景。
- **图优化派**：把所有历史位姿和观测看成一张图，每次新观测进来，对整个图的位姿做一次整体优化（比如 g2o、Ceres）。Cartographer 是代表，精度高、能处理大场景，代价是计算量和实现复杂度更高。

两类方法都依赖一个关键机制——**回环检测（Loop Closure）**：机器人走回曾经到过的地方时，识别出"我回来了"，从而把累积误差一次性纠正。这也是为什么建图时教程总强调"多走回头路"。

## 常用建图方案

| 方案 | 生态 | 输入 | 特点与适用场景 |
| ---- | ---- | ---- | ---- |
| gmapping | ROS1 | 2D 激光 | 基于粒子滤波的 2D 栅格建图，室内小场景经典选择，轻量、上手快 |
| Cartographer | ROS1/ROS2 | 2D/3D 激光 | 谷歌开源，基于图优化，激光里程计+回环检测，适合大场景和复杂走廊，精度高 |
| ORB-SLAM 系列 | 独立/ROS | 单目/双目/RGB-D | 基于特征点的视觉 SLAM，经典论文实现，适合视觉研究场景 |
| RTAB-Map | ROS1/ROS2 | RGB-D/双目 | 基于回环检测（词袋）的视觉 SLAM，能输出稠密 3D 地图 |
| slam_toolbox | ROS2 | 2D 激光 | ROS2 官方生态的 2D 建图与定位工具，接口现代，支持重定位 |

- **gmapping**：ROS1 里用 2D 激光建室内小图的首选，粒子滤波方法，包小、参数少，跑通最快。
- **Cartographer**：谷歌开源的图优化方案，自带激光里程计和回环检测，建大图不飘，代价是配置复杂度高不少。
- **ORB-SLAM**：视觉 SLAM 的标杆，特征点法，代码和研究资料极多，适合做视觉方向的研究。
- **RTAB-Map**：靠回环检测解决视觉 SLAM 的漂移，室内 RGB-D 建图效果直观。
- **slam_toolbox**：ROS2 里的"亲儿子"方案，可以简单理解成 ROS2 生态里 gmapping 的现代化替代。

!!! tip "技巧"
    选型建议：ROS1 + 室内小场景用 gmapping；ROS1/ROS2 + 大场景或走廊环境用 Cartographer；做视觉研究用 ORB-SLAM 或 RTAB-Map；纯 ROS2 项目直接用 slam_toolbox。

## TF 树

SLAM 和导航都建立在一个前提上：**坐标系变换关系（TF 树）必须正确**。一套典型的移动机器人 TF 树如下：

```text
map       世界/地图坐标系（SLAM 和定位的输出所在）
└── odom  里程计坐标系（以机器人启动点为原点）
    └── base_link  机器人本体坐标系（机器人模型的根）
        ├── laser  激光雷达
        └── camera_link  相机
```

各层变换的含义：

- **map → odom**：由定位模块（AMCL）发布，是"我在哪"的答案，低频但绝对。
- **odom → base_link**：由底盘里程计发布，高频但漂移。
- **base_link → laser / camera**：静态变换，传感器装在哪写死，由 `static_transform_publisher` 发布。

!!! warning "注意"
    初学者最常犯的错是把传感器坐标系挂错父系，或者静态变换漏发，结果 SLAM/导航一启动就报 TF 错误。建任何系统前，先把 TF 树想清楚，用 `rosrun tf view_frames`（ROS1）或 `ros2 run tf2_tools view_frames`（ROS2）检查一遍。

## 定位

### 建图之后为什么还要定位

建图时我们用的是 SLAM：同时维护地图和位姿。但**建好的地图是固定的**，之后机器人每次启动，面对的问题是"我在这张已知地图里的哪个位置"——这只需要定位，不需要再建图。而且地图已经存在，就可以用更专精的定位算法，比在线 SLAM 更稳定。

### AMCL

AMCL（Adaptive Monte Carlo Localization，自适应蒙特卡洛定位）是 ROS1 中基于已知地图的激光定位标准方案，原理是**粒子滤波**：

1. 用一大群粒子表示"机器人可能在哪"的分布，每个粒子是一个位姿假设。
2. 每个粒子把激光观测和地图比对，相似度高的粒子获得更高权重。
3. 每次移动后按权重重采样，粒子逐渐收敛到真实位姿附近。
4. "自适应"体现在粒子数量会随收敛程度动态调整，收敛了少用粒子，丢了对多撒粒子重新搜。

### 全局定位与局部定位

定位问题按"初始信息有多少"可以分两类：

- **局部定位（跟踪）**：已知初始位姿（比如启动时指定过），只需要跟随机器人移动持续修正。AMCL 粒子收敛后运行的就是这种模式，速度快、稳定。
- **全局定位（绑架问题）**：完全不知道自己在哪，需要在整个地图上搜索位姿。AMCL 在 `global_localization` 模式下会在全图撒粒子；"绑架问题"（kidnapped robot problem）指机器人突然被搬到另一个位置，测试的就是全局定位能力。

### 启动 AMCL

启动定位节点：

```bash
roslaunch amcl amcl.launch
```

启动时如果在 rviz 里给一个 2D Pose Estimate，粒子会从该位置附近快速收敛；不给的话 AMCL 会尝试全局定位，收敛慢一些。定位输出的是 `map → odom` 的变换：AMCL 把"我在地图的哪里"告诉整个系统，`odom` 子树里的所有坐标（包括导航目标）就都能换算到地图系下了。

## 导航

### move_base 与 Nav2

- **move_base（ROS1）**：经典导航框架，内部由全局规划器（global planner）、局部规划器（local planner）、全局代价地图（global costmap）、局部代价地图（local costmap）四个核心组件组成，配 recovery behavior（恢复行为）处理卡死。
- **Nav2（ROS2）**：move_base 的 ROS2 继承者，架构更模块化，引入了生命周期节点管理和行为树（Behavior Tree）来编排任务，规划器、控制器、代价地图都是可热插拔的独立组件。

两者的架构思想一致：**全局规划算大方向，局部规划实时避障**。

move_base 还内置了**恢复行为（recovery behaviors）**：当局部规划陷入死局（比如机器人被障碍围住），它会依次尝试原地旋转清空代价地图、清除局部代价地图、清除全局代价地图等操作，试图脱离困境；都失败才宣布导航失败。

### 全局规划器 vs 局部规划器

- **全局规划器**：在地图尺度上从起点规划到目标点的路径，典型算法是 **NavFn**（基于 Dijkstra/A* 的导航函数）和 A*。它只用静态地图信息，算出的是一条"大方向正确"的长路径。
- **局部规划器**：在机器人局部范围内，结合实时传感器数据（障碍物）对全局路径做跟踪和避障，输出速度指令。经典算法有 **DWA**（Dynamic Window Approach，动态窗口法，速度空间采样）和 **TEB**（Timed Elastic Band，时间弹性带，轨迹优化）。DWA 简单鲁棒，TEB 路径更平滑、能处理狭窄空间。

### 代价地图 costmap

规划器不看原始点云，而是看"代价地图"——一张栅格图，每个格子标注了通过代价。代价地图通常由三层叠加：

- **静态层（Static Layer）**：把已知的地图（SLAM 建好的 .pgm 地图）加载进来作为底图。
- **障碍物层（Obstacle Layer）**：根据实时传感器观测（激光/点云）把障碍物写进地图，负责动态障碍检测。
- **膨胀层（Inflation Layer）**：把障碍物按机器人半径做膨胀，让规划路径与障碍物保持安全距离。

## 完整工作流

一个典型的"建图 → 定位 → 导航"闭环，命令清单如下（以 ROS1 为例，包名为通用写法，实际 launch 由你的机器人包提供）：

整个流程中涉及的关键话题也值得记一下，方便排查：

```text
/scan       激光雷达数据（建图和定位的输入）
/odom       里程计数据（odom → base_link 变换）
/tf         所有坐标变换的汇总
/map        SLAM 建图时发布的地图（栅格）
/amcl_pose  AMCL 输出的机器人位姿估计
/cmd_vel    底盘速度指令（导航系统最终写给驱动）
```

### 1. 建图

启动 SLAM 节点（以 gmapping 为例），同时用遥控手柄或 teleop 节点控制机器人走遍整个环境：

```bash
roslaunch gmapping slam_gmapping.launch        # 启动 2D SLAM 建图
rosrun teleop_twist_keyboard teleop_twist_keyboard.py  # 遥控机器人移动
```

注意走位：慢速、多回环（走回头路触发回环闭合），地图才会干净。

### 2. 保存地图

SLAM 节点在 `/map` 话题上持续发布栅格地图，用 map_saver 保存：

```bash
rosrun map_server map_saver -f ~/maps/my_map   # 生成 my_map.pgm + my_map.yaml
```

### 3. 定位

保存地图后，不再建图，改为加载地图 + AMCL 定位：

```bash
roslaunch map_server map_server.launch          # 或用你的机器人包提供的 launch 加载地图
roslaunch amcl amcl.launch                      # 启动 AMCL 定位
```

如果定位初始分布不对，可以给 AMCL 指定初始位姿（rviz2 里 2D Pose Estimate），让它从正确位置开始收敛。

### 4. 导航

启动 move_base（具体 launch 文件由你的机器人包提供，比如 turtlebot3 系的 navigation 包）：

```bash
roslaunch move_base move_base.launch            # 实际使用你机器人包的 launch 文件
```

在 rviz 里用 2D Nav Goal 点一个目标点，机器人就会规划路径并开过去。ROS2 对应流程：`ros2 launch nav2_bringup bringup_launch.py` 启动 Nav2，配合 `slam_toolbox` 建图。

### 一个典型的导航配置目录

实操中，导航相关的一切通常收敛成这样一个包结构（各包命名不同，结构大同小异）：

```text
my_robot_navigation/
├── maps/
│   ├── my_map.pgm     # SLAM 保存的栅格地图图片
│   └── my_map.yaml    # 地图元数据（分辨率、原点、占用阈值）
├── config/
│   ├── costmap_common_params.yaml      # 通用代价地图参数（footprint、障碍层）
│   ├── global_costmap_params.yaml      # 全局代价地图参数
│   ├── local_costmap_params.yaml       # 局部代价地图参数
│   ├── base_local_planner_params.yaml  # 局部规划器（DWA 等）参数
│   └── amcl_params.yaml                # AMCL 定位参数
└── launch/
    ├── map_server.launch               # 加载地图
    ├── amcl.launch                     # 启动定位
    └── move_base.launch                # 启动导航（引用上面的 config）
```

建议把"参数文件"和"launch 文件"分开管理：调参只改 config，不碰 launch，逻辑清晰很多。

!!! tip "技巧"
    建议把每一步拆开验证：建图时盯 rviz 看地图质量，定位时看 AMCL 粒子是否收敛，最后才测导航。一次跑全流程，出问题根本不知道是哪一环的锅。

## 常见问题

### TF 缺失报错

典型报错：`Could not find transform ...` 或 `invalid frame id [xxx]`。检查顺序：

1. `rosrun tf view_frames` 生成 TF 树，看哪些 frame 没连上。
2. 确认所有静态变换（base_link → laser 等）都发布，且父系挂对。
3. 确认里程计节点在运行、`odom` 话题有数据。

### odom 漂移

里程计越走越偏是物理特性，不是 bug。缓解办法：

- 标定轮距、轮径等运动学参数（如 `calibrate_linear` / 官方标定教程）。
- 引入 IMU 做航位推算融合（`robot_localization` 包）。
- 依靠定位（AMCL）周期性修正，odom 负责短时平滑，map 负责长期纠偏。

### costmap 不更新

导航时动态障碍物没有出现在代价地图里，多半是：

- 局部代价地图的观测源（observation sources）没配置对，检查 costmap 参数里的传感器 topic。
- 传感器数据的坐标系在 TF 树里缺失（回到 TF 问题）。
- 膨胀半径或 footprint 配置错误，导致障碍写入后被立即清掉或不可见。

### AMCL 定位丢失

粒子散开、位姿跳动（即"机器人在哪"突然对不上），常见原因：

- 初始位姿给得离谱，粒子重采样无法收敛：重新用 2D Pose Estimate 指定初始位姿。
- 环境特征太少（超长走廊、空旷大厅）：粒子滤波天然难收敛，考虑加激光特征或改用 Cartographer 类方案。
- 地图质量差或与当前环境不一致（家具搬走了）：重新建图。
- 激光帧率过低或数据剧烈抖动：检查激光话题是否稳定，必要时在 AMCL 参数里调大 `laser_max_range`、`update_min_d` 等容错参数。

### 导航跑不起来 / 机器人不动

`move_base` 收到目标但机器人原地不动，优先检查：

- 是否收到恢复行为报警：多半是局部规划器认为周围全是障碍，检查 footprint 是否比实际车身大。
- `cmd_vel` 话题有没有数据：`rostopic echo /cmd_vel` 确认 move_base 在输出速度。
- TF 树的 `odom → base_link` 是否由底盘驱动正常发布：不发布的话所有规划都会认为机器人没在移动。
- AMCL 粒子是否收敛：位姿不对时，规划器按错误位姿规划，看起来就像"乱走"或"不走"。

## 结语与参考链接

SLAM 和导航是移动机器人从"会动"到"会干活"的分水岭。我的建议是先完整跑通 ROS1 的 gmapping + AMCL + move_base 链路，把 TF 树、代价地图、粒子滤波这些概念在实战中理解透，再迁移到 ROS2 的 slam_toolbox + Nav2。概念相通，迁移成本其实不高。

官方资料（按阅读顺序）：

- [ROS wiki: gmapping](http://wiki.ros.org/gmapping) — 2D SLAM 建图文档
- [ROS wiki: amcl](http://wiki.ros.org/amcl) — 粒子滤波定位文档
- [ROS wiki: move_base](http://wiki.ros.org/move_base) — ROS1 导航框架文档
- [ROS wiki: costmap_2d](http://wiki.ros.org/costmap_2d) — 代价地图层文档
- [Cartographer ROS 文档](https://google-cartographer-ros.readthedocs.io/en/latest/) — 谷歌建图方案官方文档
- [slam_toolbox](https://github.com/SteveMacenski/slam_toolbox) — ROS2 建图工具仓库
- [Nav2 官方文档](https://docs.nav2.org/) — ROS2 导航框架文档
- [tf2 教程（ROS wiki）](http://wiki.ros.org/tf2) — 坐标变换入门必读
- [ROS2 tf2 教程](https://docs.ros.org/en/rolling/Tutorials/Intermediate/Tf2.html) — ROS2 版 TF 教程
