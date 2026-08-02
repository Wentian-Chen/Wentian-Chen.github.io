# Isaac Sim 使用指南与实战教程

本教程基于 Isaac Sim 5.0 API 文档及相关开发经验整理，涵盖了从环境搭建、模型管理到运动控制与 ROS2 联合仿真的全流程指南。

## 目录
1. [部署与安装](#1-部署与安装)
2. [ROS2 环境配置](#2-ros2-环境配置)
3. [仿真环境构建与资产管理](#3-仿真环境构建与资产管理)
4. [机器人设置与关节微调](#4-机器人设置与关节微调)
5. [运动生成 (Lula & cuRobo)](#5-运动生成-lula--curobo)
6. [核心编程接口 (API)](#6-核心编程接口-api)
7. [强化学习与策略部署](#7-强化学习与策略部署)
8. [ROS2 与 OmniGraph 联合仿真](#8-ros2-与-omnigraph-联合仿真)

---

## 1. 部署与安装

### 远程服务器部署
你可以通过 Docker 在远程工作站进行部署，并通过客户端推流进行操作。
执行以下指令以启动 Isaac Sim 4.5 的 Docker 容器：
```
docker run --name cwt-isaac-sim-4.5 --entrypoint bash it --runtime nvidia \
gpus all -e "ACCEPT_EULA=Y" --network host \
-e "PRIVACY_CONSENT=Y" \
-v ~/docker/isaac-sim/cache/kit:/isaac-sim/kit/cache:rw \
-v ~/docker/isaac-sim/cache/ov:/root/.cache/ov:rw \
-v ~/docker/isaac-sim/cache/pip:/root/.cache/pip:rw \
-v ~/docker/isaac-sim/cache/glcache:/root/.cache/nvidia/GLCache:rw \
-v ~/docker/isaac-sim/cache/computecache:/root/.nv/ComputeCache:rw \
-v ~/docker/isaac-sim/logs:/root/.nvidia-omniverse/logs:rw \
-v ~/docker/isaac-sim/data:/root/.local/share/ov/data:rw \
-v ~/docker/isaac-sim/documents:/root/Documents:rw \
-v /home/charles/project/download:/isaac-sim/data \
nvcr.io/nvidia/isaac-sim:4.5.0
```
- 注意：需要将 -v 挂载路径中的 /home/charles/project/download 替换为你自己的本地路径。

参考：搜搜[Isaacsim 5.0.0 Container Installation](https://docs.isaacsim.omniverse.nvidia.com/5.0.0/installation/install_container.html)
[](https://docs.isaacsim.omniverse.nvidia.com/5.0.0/installation/manual_livestream_clients.html)sda