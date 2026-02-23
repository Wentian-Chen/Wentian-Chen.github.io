<!--
 * @Author: boboji11 wendychen112@qq.com
 * @Date: 2026-02-24 00:33:12
 * @LastEditors: boboji11 wendychen112@qq.com
 * @LastEditTime: 2026-02-24 01:00:05
 * @FilePath: \WentianChen.github.io\docs\blog\envir_config\config_thor.md
 * @Description: 
 * 
 * Copyright (c) 2026 by boboji11 , All Rights Reserved. 
-->
# 黑客松 - Jetson Thor 真机部署指南 

本文介绍如何在 **Jetson AGX Thor** 上部署 **GR00T N1.5** 模型，并为 **LeRobot SO-101**机械臂微调模型。 

## 系统架构说明
系统分为两个主要终端进行通讯：

* **终端1（推理服务器）：** 运行 `scripts/inference_service.py` 脚本，加载微调后的模型并启动 GR00T N1.5 策略服务 (Server)。该服务端负责接收推理请求（图像+指令），并返回动作指令。
* **终端2（机器人客户端）：** 运行 `getting_started/examples/eval_lerobot.py` 脚本，用于采集实时图像、执行动作，并连接物理的 SO-101 机械臂。

## 基础环境配置
参考：[Jetson Thor 权威指南：从开箱到大模型部署与性能优化](https://zhuanlan.zhihu.com/p/1959665010105119558)

通过 BSP 安装的 Jetson 系统默认已安装 Docker 和 NVIDIA runtime，可直接使用 Docker 部署模型。

### 配置 WIFI、SSH、Nomachine
* **连接 WIFI**：
```
# 查询当前可用的WIFI列表
sudo nmcli device wifi list
# 通过SSID连接
sudo nmcli device wifi connect SSID password <PASSWORD>
# 通过BSSID连接
sudo nmcli device wifi connect BSSID password <PASSWORD>
# 查询WIFI连接状态
nmcli device wifi
```
* **开启 SSH**：
```
sudo apt update
sudo apt install openssh-server ufw -y
# 编辑/etc/ssh/sshd_config开放以下部分
# PubkeyAuthentication yes

# 手动添加公钥
sudo nano ~/.ssh/authorized_keys

sudo systemctl restart sshd
sudo systemctl enable ssh
sudo ufw allow ssh
```
* **安装 Nomachine**：
参考：[在Ubuntu上配置NoMachine](https://www.zeeklog.com/zai-ubuntushang-an-zhuang-he-bu-shu-nomachine/)
### 设置性能模式
* **性能模式设置**：
```
sudo nvpmodel -m 0
# 0: 最高性能模式，总功耗 130W，用于释放所有性能限制。
# 1: 受限的性能模式，总功耗限制在 120W 左右。（默认）
查看当前的功率模式
nvpmodel -q
```

### 更换国内镜像源
Jetson AGX Thor是arm64架构的板子，系统为ubuntu24.04，配置流程可参考：[清华源--arm64配置教程](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu-ports/)

先备份原源文件后，将 Ubuntu 源替换为清华源（适用 arm64） 
```
cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.backup
sudo nano /etc/apt/sources.list.d/ubuntu.sources
# 用以下内容覆盖源文件
Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
# Types: deb-src
# URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports
# Suites: noble noble-updates noble-backports
# Components: main restricted universe multiverse
# Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

# 以下安全更新软件源包含了官方源与镜像站配置，如有需要可自行修改注释切换
Types: deb
URIs: http://ports.ubuntu.com/ubuntu-ports/
Suites: noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

# Types: deb-src
# URIs: http://ports.ubuntu.com/ubuntu-ports/
# Suites: noble-security
# Components: main restricted universe multiverse
# Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

# 预发布软件源，不建议启用

# Types: deb
# URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports
# Suites: noble-proposed
# Components: main restricted universe multiverse
# Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

# # Types: deb-src
# # URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports
# # Suites: noble-proposed
# # Components: main restricted universe multiverse
# # Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```
### 配置Thor本地开发环境
主要是安装JetPack SDK 与基础工具，参考：[Thor 上的基本开发环境设置](https://wiki.seeedstudio.com/cn/fine_tune_gr00t_n1.5_for_lerobot_so_arm_and_deploy_on_jetson_thor/#thor-%E4%B8%8A%E7%9A%84%E5%9F%BA%E6%9C%AC%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83%E8%AE%BE%E7%BD%AE)
```
sudo apt update
sudo apt install nvidia-jetpack firefox python3 python3-pip -y

sudo pip3 install -U pip
sudo pip3 install jetson-stats
```
### 配置好Docker
由于Jetson烧录的系统自带了docker，因此接下来检测docker是否正常。

首先检查Docker的安装:
```
docker --version
# 添加进用户组（可选）
sudo gpasswd -a $USER docker
newgrp docker
docker version 
```
接着，检查是否配置好`Nvidia Container ToolKit`:
```
cat /etc/docker/daemon.json
# 配置成功的输入如下
{
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
            }
        },
    "registry-mirrors": ["https://docker.xuanyuan.me"]
}
# 更改配置后需要重启
sudo systemctl daemon-reload
sudo systemctl restart docker
```
### 配置miniconda
下载miniconda for aarch64
```
# wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-aarch64.sh
# 安装miniconda
chmod +x Miniconda3-latest-Linux-aarch64.sh
./Miniconda3-latest-Linux-aarch64.sh
source ~/.bashrc
conda --version
```
### 配置Lerobot和gr00t的运行环境
由于Thor的系统过于“前沿”，且架构特殊。因此需要做一下优化才能正常配置Lerobot和gr00t的运行环境。

首先，核心Python Package需要从NVIDIA提供[预编译包列表](https://pypi.jetson-ai-lab.io/sbsa/cu130)里下载。

其次，lerobot是一个大型项目，因此它的`pyproject.toml`文件包含了很多依赖项，其中`pytorch3d`是不需要且NVIDIA没有提供预编译的。因此，需要将`pytorch3d`从依赖项中移除或屏蔽。

否则会出现以下报错：
```
ERROR: Ignored the following versions that require a different python version: 1.21.2 Requires-Python >=3.7,<3.11; 1.21.3 Requires-Python >=3.7,<3.11; 1.21.4 Requires-Python >=3.7,<3.11; 1.21.5 Requires-Python >=3.7,<3.11; 1.21.6 Requires-Python >=3.7,<3.11
ERROR: Could not find a version that satisfies the requirement pytorch3d==0.7.8; extra == "thor" (from gr00t[thor]) (from versions: none)
ERROR: No matching distribution found for pytorch3d==0.7.8; extra == "thor"
(gr00t-server) root@qiz:/workspace/Isaac-GR00T-main#
```
除此，Thor只能支持CUDA 13.0+版本的torch，然后通用源里没有提供arm64架构的cuda13.00版本的预编译包，我们依旧需要从[预编译包列表](https://pypi.jetson-ai-lab.io/sbsa/cu130)里下载并安装。

否则会出现以下报错：
```
(gr00t-server-cp12) root@qiz:/workspace/Isaac-GR00T-main# python deployment_scripts/gr00t_inference.py --inference_mode=tensorrt
Traceback (most recent call last):
  File "/workspace/Isaac-GR00T-main/deployment_scripts/gr00t_inference.py", line 20, in <module>
    import torch
  File "/workspace/miniconda3/envs/gr00t-server-cp12/lib/python3.12/site-packages/torch/__init__.py", line 2149, in <module>
    from torch import _VF as _VF, functional as functional  # usort: skip
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/workspace/miniconda3/envs/gr00t-server-cp12/lib/python3.12/site-packages/torch/functional.py", line 8, in <module>
    import torch.nn.functional as F
  File "/workspace/miniconda3/envs/gr00t-server-cp12/lib/python3.12/site-packages/torch/nn/__init__.py", line 8, in <module>
    from torch.nn.modules import *  # usort: skip # noqa: F403
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/workspace/miniconda3/envs/gr00t-server-cp12/lib/python3.12/site-packages/torch/nn/modules/__init__.py", line 1, in <module>
    from .module import Module  # usort: skip
    ^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/workspace/miniconda3/envs/gr00t-server-cp12/lib/python3.12/site-packages/torch/nn/modules/module.py", line 17, in <module>
    from torch.utils._python_dispatch import is_traceable_wrapper_subclass
  File "/workspace/miniconda3/envs/gr00t-server-cp12/lib/python3.12/site-packages/torch/utils/__init__.py", line 8, in <module>
    from torch.utils import (
  File "/workspace/miniconda3/envs/gr00t-server-cp12/lib/python3.12/site-packages/torch/utils/data/__init__.py", line 1, in <module>
    from torch.utils.data.dataloader import (
  File "/workspace/miniconda3/envs/gr00t-server-cp12/lib/python3.12/site-packages/torch/utils/data/dataloader.py", line 23, in <module>
    import torch.distributed as dist
  File "/workspace/miniconda3/envs/gr00t-server-cp12/lib/python3.12/site-packages/torch/distributed/__init__.py", line 33, in <module>
    from torch.distributed._distributed_c10d import (
  File "/workspace/miniconda3/envs/gr00t-server-cp12/lib/python3.12/site-packages/torch/distributed/_distributed_c10d.py", line 19, in <module>
    from torch._C._distributed_c10d import (
ImportError: cannot import name '_current_process_group' from 'torch._C._distributed_c10d' (unknown location). Did you mean: '_resolve_process_group'?
```