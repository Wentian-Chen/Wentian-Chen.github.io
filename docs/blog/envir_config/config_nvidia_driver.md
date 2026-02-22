<!--
 * @Author: boboji11 wendychen112@qq.com
 * @Date: 2026-02-22 22:16:49
 * @LastEditors: boboji11 wendychen112@qq.com
 * @LastEditTime: 2026-02-22 22:22:12
 * @FilePath: \WentianChen.github.io\docs\blog\config_nvidia_driver.md
 * @Description: 
 * 
 * Copyright (c) 2026 by boboji11 , All Rights Reserved. 
-->
# 配置NVIDIA显卡驱动
## 安装NVIDIA显卡驱动
参考：

- [NVIDIA Official Drivers Installation Guide](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/introduction.html)
- [Ubuntu Drivers Installation Guide](https://documentation.ubuntu.com/server/how-to/graphics/install-nvidia-drivers/index.html)

根据ubuntu官方的[Ubuntu Drivers Installation Guide](https://documentation.ubuntu.com/server/how-to/graphics/install-nvidia-drivers/index.html)指南，官方提供两种类型的NVIDIA Driver版本：

1. Unified Driver Architecture (UDA) drivers，which are recommended for the generic desktop use。（适用个人电脑，训练模型，科学计算 等场景）
2. Enterprise Ready Drivers，which are recommended on servers and for computing tasks. （适用于企业）
   
官方推荐使用`ubuntu-drivers tool`完成驱动安装，尤其是电脑启用了`Secure Boot`，步骤如下：

推荐安装开源，即`open`版本，适配笔记本，台式等。
```
# 列出可安装的驱动版本 
sudo ubuntu-drivers list
# 方法一：自动检测最适合电脑的版本进行安装
sudo ubuntu-drivers install
# 指定想安装的版本，以535为例
sudo ubuntu-drivers install nvidia:535
# 指定想安装的版本，以535-open为例
sudo ubuntu-drivers install nvidia:535-open
```
## 问题排除流程
```
# 前提：
sudo apt update 
sudo apt install linux-headers-$(uname -r) build-essential
# 是否检测到物理显卡
lspci | grep -i nvidia
# 是否已经下载了nvidia驱动包
ls /var/cache/apt/archives | grep nvidia-driver
# 检测是否已经安装了驱动
dpkg -l | grep nvidia-driver
# 移除已安装的驱动
sudo apt purge *nvidia*
sudo apt autoremove
sudo apt autoclean
# 添加显卡驱动PPA
sudo add-apt-repository ppa:graphics-drivers/ppa -y
sudo apt update
# 查询可安装、推荐的nvidia驱动
ubuntu-drivers devices
# 安装指定版本的驱动
# 推荐安装metadata类型
sudo apt install nvidia-driver-570
# 安装DKMS 组件（metadatat类型不需要安装）
sudo apt install nvidia-dkms-570
# 检查内核模块版本
cat /proc/driver/nvidia/version
# 内核模块信息
sudo modinfo nvidia
# 检测安全模式
mokutil --sb-state
# 生成密码
sudo apt install mokutil openssl
sudo openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -nodes -days 36500 -subj "/CN=NVidia Driver/"
sudo mokutil --import MOK.der
# 验证驱动加载
lsmod | grep nvidia
# 加载驱动
sudo modprobe nvidia
# 最后核验
nvidia-smi
```
