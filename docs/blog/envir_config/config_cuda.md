<!--
 * @Author: boboji11 wendychen112@qq.com
 * @Date: 2026-02-22 17:48:01
 * @LastEditors: boboji11 wendychen112@qq.com
 * @LastEditTime: 2026-02-22 22:49:46
 * @FilePath: \WentianChen.github.io\docs\blog\envir_config\config_cuda.md
 * @Description: 
 * 
 * Copyright (c) 2026 by boboji11 , All Rights Reserved. 
-->
# 配置cuda 
## For linux 
### 安装前的检查
本部分参考[Pre-installation Actions](https://docs.nvidia.com/cuda/archive/12.8.0/cuda-installation-guide-linux/#pre-installation-actions)
```
# 验证是否装有支持CUDA的GPU
lspci | grep -i nvidia
# 预期输出：[有输出即可]

# 验证linux系统是否支持
# 注：常用的系统都支持，可跳过检查
uname -m && cat /etc/*release
# 预期输出：
# x86_64 
# Red Hat Enterprise Linux Workstation release 6.0 (Santiago)

# 验证gcc是否安装
gcc --version
```
### 安装CUDA
在[CUDA版本列表](https://developer.nvidia.com/cuda-toolkit-archive)查找需要安装的版本，并根据指示选择版本和执行页面内的指令进行安装进行安装。其中`Installer Type`选择为`deb(local)`。

参考案例如：![参考案例](../..//assets/pictures/config_cuda_choose_cuda.png)

### 安装后的配置
本部分参考[Post-installation Actions](https://docs.nvidia.com/cuda/archive/12.8.0/cuda-installation-guide-linux/#post-installation-actions)

安装常见的依赖
```
sudo apt-get install g++ freeglut3-dev build-essential libx11-dev \
    libxmu-dev libxi-dev libglu1-mesa-dev libfreeimage-dev libglfw3-dev
```
配置路径

- 方法一：通过PATH环境变量指定CUDA版本
```
export PATH=/usr/local/cuda-12.6/bin${PATH:+:${PATH}}
```
- 方法二：通过update-alternatives管理（推荐）
```
# 1. 显示当前活动的 CUDA 版本和所有可用版本
update-alternatives --display cuda

# 2. 显示特定主版本的所有可用小版本,如CUDA-12.*
update-alternatives --display cuda-12

# 3. 交互式切换 CUDA 版本
sudo update-alternatives --config cuda

# 再次确认是否切换成功
update-alternatives --display cuda
# 检查当前激活的 CUDA 版本
nvcc --version
# 检查符号链接指向
ls -l /usr/local/cuda

# 如果所安装的CUDA未被update-alternatives识别
# 则需要将 CUDA 版本注册到 alternatives 系统
# 例如，注册 CUDA 12.6，
# /usr/local/cuda：符号链接的位置（系统实际使用的路径）
# cuda：alternatives 系统中的名称
# /usr/local/cuda-11.8：实际的 CUDA 安装路径
# 110：优先级（数字越大优先级越高）
sudo update-alternatives --install /usr/local/cuda cuda /usr/local/cuda-12.6 120
```
清理冗余的安装包
```
sudo apt-get remove --purge "cuda-repo-<distro>-X-Y-local*"
```
查询当前运行的CUDA版本
```
nvcc --version
```
