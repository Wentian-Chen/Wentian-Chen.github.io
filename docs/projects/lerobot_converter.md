<!--
 * @Author: boboji11 wendychen112@qq.com
 * @Date: 2026-02-26 22:15:30
 * @LastEditors: boboji11 wendychen112@qq.com
 * @LastEditTime: 2026-02-26 22:29:26
 * @FilePath: \WentianChen.github.io\docs\projects\lerobot_converter.md
 * @Description: 
 * 
 * Copyright (c) 2026 by boboji11 , All Rights Reserved. 
-->
# lerobot_convertor
`lerobot_converter` 是一个任意数据集转换至 LeRobot 格式的轻量级框架：

在机器人控制与 Vision-Language-Action (VLA) 模型的开发过程中，数据准备往往是最折磨人的一环。将各种来源的数据（如自定义 HDF5、RLDS）转换为 Hugging Face 官方的 LeRobotDataset 格式，常常伴随着内存爆炸和满天飞的硬编码规则。

官方或现有的转换工具往往将源数据规则硬编码在框架里，缺乏灵活性。为了让数据集的二次开发和转换变得像搭积木一样轻松，我开发并开源了 lerobot_convertor。lerobot_convertor 致力于打造一个可扩展的数据集转换器框架，它的核心设计理念是：把脏活累活封装好，把最大的自由度留给用户。

核心特点：

- 输入包容性强： 支持任意来源（HDF5、RLDS、自定义格式）。

- 职责分明： 用户只需专注于“读取 + 映射 + feature 对齐”，框架负责生成官方标准 LeRobotDataset。

- 极致轻量： 采用迭代式读取机制进行数据转换，按 episode 甚至 frame 级别处理数据，彻底告别一次性读入造成的内存爆炸问题。

- VLA 任务友好： 内置任务指令增强（Instruction Augmentation）机制，非常适合训练视觉-语言-动作模型。

框架结构如下：

- 输入：任意来源（HDF5、RLDS、自定义格式）
- 过程：用户做“读取 + 映射 + feature 对齐”
- 输出：官方 `LeRobotDataset`

更多详细细节请移步项目仓库地址：[Github-lerobot_converter](https://github.com/Wentian-Chen/lerobot_converter)