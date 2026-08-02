# 项目展示

<p class="km-lead">这里收录了我的一些机器人、仿真与数据相关的项目，从全自动数据采集到 VLA 微调部署，再到数据集工具链。</p>

## SimPickGen

针对 VLA 领域的数据稀缺和实机采集成本高昂等问题，设计并实现的基于 IsaacSim 的全自动机械臂随机位姿抓取及数据采集系统。

- [项目详情](simpickgen.md)

## LeoRobot

为 [2025 具身智能黑客松-深圳站](https://www.seeedstudio.com.cn/post/%E6%8E%A2%E7%B4%A2%E5%85%B7%E8%BA%AB%E6%99%BA%E8%83%BD%E7%9A%84%E6%9C%AA%E6%9D%A5%EF%BC%9A2025-seeed-embodied-ai-%E9%BB%91%E5%AE%A2%E6%9D%BE-%E6%B7%B1%E5%9C%B3%E7%AB%99) 而准备的桌面清理机器人，荣获本次黑客松活动第一名。使用 300 份仿真与实物遥操作数据，在 GR00T N1.5-3B 模型上微调，并在 RTX 4090 与 Jetson Thor 上完成部署推理。

- [项目详情](leorobot.md)
- [项目仓库](https://github.com/Lee-LEO-H/Lerobot-GR00TN1.5)

## lerobot_converter

`lerobot_converter` 是一个任意数据集转换至 LeRobot 格式的轻量级框架。输入支持任意来源（HDF5、RLDS、自定义格式），用户只需专注「读取 + 映射 + feature 对齐」，框架负责生成官方标准 LeRobotDataset，并内置任务指令增强机制，非常适合训练视觉-语言-动作模型。

- [项目详情](lerobot_converter.md)
- [项目仓库](https://github.com/Wentian-Chen/lerobot_converter)
