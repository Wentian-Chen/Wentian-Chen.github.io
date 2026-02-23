# LeoRobot
## 背景
本项目是为由Seeed & Nvidia & HuggingFace联合举办的[2025 具身智能黑客松-深圳站](https://www.seeedstudio.com.cn/post/%E6%8E%A2%E7%B4%A2%E5%85%B7%E8%BA%AB%E6%99%BA%E8%83%BD%E7%9A%84%E6%9C%AA%E6%9D%A5%EF%BC%9A2025-seeed-embodied-ai-%E9%BB%91%E5%AE%A2%E6%9D%BE-%E6%B7%B1%E5%9C%B3%E7%AB%99)而准备的桌面清理机器人。详情可参考项目仓库：[LeoRobot](https://github.com/Lee-LEO-H/Lerobot-GR00TN1.5)
## 项目细节
我们采用在仿真（Isaac Sim and Mujoco）+现实的遥操作采集数据，共300份数据。使用采集到的数据在GR00T N1.5-3B模型上进行微调，启用Diffusion+ 35k Steps在A6000上进行训练，大约花费9小时。在RTX 4090和Jetson Thor上完成部署进行推理。在4090上的单次平均推理耗时是55.67ms，在Jetson Thor的单次推理耗时是118.12ms。
## 项目特点
项目特点:本项目荣获本次黑客松活动第一名。获奖理由是泛化性强（能够抓取训练集之外的笔、螺丝刀等）、机械臂运行流畅、抓取准确率高等。
## 效果演示

## Thor部署教程
参考：[黑客松-Thor部署](../blog/envir_config/config_thor.md)