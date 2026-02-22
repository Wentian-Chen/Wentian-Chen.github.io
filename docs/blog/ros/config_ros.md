<!--
 * @Author: boboji11 wendychen112@qq.com
 * @Date: 2026-02-22 22:25:11
 * @LastEditors: boboji11 wendychen112@qq.com
 * @LastEditTime: 2026-02-22 22:41:11
 * @FilePath: \WentianChen.github.io\docs\blog\config_ros.md
 * @Description: 
 * 
 * Copyright (c) 2026 by boboji11 , All Rights Reserved. 
-->
# 安装ROS
## ROS1
### 安装ROS1
本部分以为`noetic`为例。

参考：[Ubuntu install of ROS Noetic](https://wiki.ros.org/noetic/Installation/Ubuntu)
```
# 配置ROS的Apt国内镜像源
# 以下指令选择其一执行即可，无需全部执行
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.ustc.edu.cn/ros/ubuntu/ `lsb_release -cs` main" > /etc/apt/sources.list.d/ros-latest.list'
sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.tuna.tsinghua.edu.cn/ros/ubuntu/ `lsb_release -cs` main" > /etc/apt/sources.list.d/ros-latest.list'
sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.bfsu.edu.cn/ros/ubuntu/ `lsb_release -cs` main" > /etc/apt/sources.list.d/ros-latest.list'
sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.sjtug.sjtu.edu.cn/ros/ubuntu/ `lsb_release -cs` main" > /etc/apt/sources.list.d/ros-latest.list'
sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.zju.edu.cn/ros/ubuntu/ `lsb_release -cs` main" > /etc/apt/sources.list.d/ros-latest.list'

# 安装 ros noetic
sudo apt install curl
curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo apt-key add -
sudo apt update
sudo apt install ros-noetic-desktop-full
echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc
source ~/.bashrc
sudo apt install python3-rosdep python3-rosinstall python3-rosinstall-generator python3-wstool build-essential
sudo apt-get install python3-pip
```
### 激活rosdep
#### 方法一：使用清华源的方法（推荐）

参考：

- [清华源 ROS Distro](https://mirrors.tuna.tsinghua.edu.cn/help/rosdistro/)
- [清华源 手动模拟rosdep init](https://mirrors.tuna.tsinghua.edu.cn/help/github-raw/)

```
# 手动模拟rosdep init
# 手动模拟 rosdep init
mkdir -p /etc/ros/rosdep/sources.list.d/
curl -o /etc/ros/rosdep/sources.list.d/20-default.list -L https://mirrors.tuna.tsinghua.edu.cn/github-raw/ros/rosdistro/master/rosdep/sources.list.d/20-default.list

# 为 rosdep update 换源
export ROSDISTRO_INDEX_URL=https://mirrors.tuna.tsinghua.edu.cn/rosdistro/index-v4.yaml
rosdep update
```
#### 方法二：鱼香ros-rosdepc
```
# 安装rosdepc
sudo pip install rosdepc
# 如果显示没有pip可以试试pip3。
sudo pip3 install rosdepc
# 如果pip3还没有
sudo apt-get install python3-pip 
sudo pip install rosdepc
# 使用
sudo rosdepc init
rosdepc update
```
#### 方法三：科学上网法
```
# 配置本地端口代理
export ALL_PROXY="http://127.0.0.1:7897"
# 初始化
rosdep init
# 查询ip地址
## 查询 raw.githubusercontent.com 的IP
## 查询到其中一个ip是 185.199.110.133
## 尝试ping，检查是否成功
ping 185.199.110.133
# 成功，则添加ip和域名，更改hosts
echo "185.199.110.133 raw.githubusercontent.com " >> /etc/hosts
# 激活，失败的话，多试几次。
rosdep
```
#### 方法四：鱼香ros一键脚本
```
wget http://fishros.com/install -O fishros && . fishros
```
#### 方法五：
```
sudo apt-get install python3-pip
sudo pip3 install 6-rosdep
sudo 6-rosdep
sudo apt install python3-rosdep
sudo rosdep init
rosdep update
```
## ROS2
### 安装ROS2
本部分以为`Humble`为例。

参考：[鱼香ros中文版ROS2 Humble](https://fishros.org/doc/ros2/humble/Tutorials.html)
```
apt update && apt install locales
locale-gen en_US en_US.UTF-8
update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
echo "export LANG=en_US.UTF-8" >> ~/.bashrc
locale

apt install software-properties-common
add-apt-repository universe

apt update && apt install curl -y
curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | tee /etc/apt/sources.list.d/ros2.list > /dev/null
apt update
apt install ros-humble-desktop -y
source /opt/ros/humble/setup.bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
export ROS_DOMAIN_ID=0
echo "export ROS_DOMAIN_ID=0" >> ~/.bashrc
```
### 激活rosdep
参考：

- [清华源 ROS Distro](https://mirrors.tuna.tsinghua.edu.cn/help/rosdistro/)
- [清华源 手动模拟rosdep init](https://mirrors.tuna.tsinghua.edu.cn/help/github-raw/)
```
apt update
apt install python3-pip -y
pip3 install -U rosdep # rosdepc
mkdir -p /etc/ros/rosdep/sources.list.d/
# 主机能上网时，可以通过curl下载
curl -o /etc/ros/rosdep/sources.list.d/20-default.list -L https://mirrors.tuna.tsinghua.edu.cn/github-raw/ros/rosdistro/master/rosdep/sources.list.d/20-default.list
# 没网时，可以手动添加
tee /etc/ros/rosdep/sources.list.d/20-default.list <<EOF
# os-specific listings first
yaml https://mirrors.tuna.tsinghua.edu.cn/github-raw/ros/rosdistro/master/rosdep/osx-homebrew.yaml osx

# generic
yaml https://mirrors.tuna.tsinghua.edu.cn/github-raw/ros/rosdistro/master/rosdep/base.yaml
yaml https://mirrors.tuna.tsinghua.edu.cn/github-raw/ros/rosdistro/master/rosdep/python.yaml
yaml https://mirrors.tuna.tsinghua.edu.cn/github-raw/ros/rosdistro/master/rosdep/ruby.yaml

# newer distributions (Groovy, Hydro, ...) must not be listed anymore, they are
EOF
# 单次添加，只更新一次
export ROSDISTRO_INDEX_URL=https://mirrors.tuna.tsinghua.edu.cn/rosdistro/index-v4.yaml
rosdep update
```
