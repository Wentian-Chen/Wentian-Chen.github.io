# 配置ROS

我刚开始接触机器人开发的时候，第一个劝退我的不是机器人的运动学，而是 ROS 的安装。apt 源、rosdep、环境变量……每一步都可能在网络这一步卡住。这篇文章整理了我自己完整跑通一遍的安装流程，ROS1 以 Noetic 为例、ROS2 以 Humble 为例，包含国内镜像源和 rosdep 激活的几种方案。

## 先想清楚：ROS1 还是 ROS2？

简单粗暴的结论：**如果项目必须依赖 ROS1 时代的成熟生态（比如 move_base、gmapping、SLAM 工具箱里的老包），或者要和旧代码、老机器人对接，选 ROS1 Noetic**；**如果是从零开始学习、或者做新项目，优先选 ROS2 Humble**。

几点客观差异供参考：

- ROS1 是中心化的架构，节点之间靠唯一的 `roscore` 做路由，`roscore` 挂了整个系统就瘫了；ROS2 去掉了 `roscore`，节点通过 DDS 直接互相发现和通信。
- ROS2 支持多机通信、实时性更好，也引入了 QoS 机制，通信策略更灵活。
- ROS1 已经停止新增功能，主流支持停留在 Noetic（Ubuntu 20.04）；ROS2 是目前官方主推的版本，新功能和新教程都在往 ROS2 迁移。
- 对于做移动机器人和具身智能的同学，学术界的很多新代码（比如机械臂控制、视觉语言导航）已经全面转向 ROS2，但传统导航栈（move_base + AMCL + gmapping）依然是 ROS1 最成熟。

我自己的建议：**如果你只是要跑通一个传统轮式机器人导航 demo，ROS1 Noetic 会非常顺利；如果你打算长期做研究、写新代码，直接 ROS2 Humble，长痛不如短痛。**

## ROS1

本部分以为`noetic`为例。

### 安装ROS1

参考：[Ubuntu install of ROS Noetic](https://wiki.ros.org/noetic/Installation/Ubuntu)

!!! note "说明"
    安装前请确认系统是 **Ubuntu 20.04**。Noetic 官方只支持 20.04，其他版本强行安装很容易出现依赖地狱。另外，`roscore` 时代已经过去了，后续所有命令都不再需要额外启动 roscore，`roslaunch` 会自动拉起它。

```bash
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

!!! tip "技巧"
    如果 `curl https://raw.githubusercontent.com/...` 下载 `ros.asc` 失败，说明你访问不了 GitHub 的 raw 域名。可以先用 `apt install curl`，然后到 [ros/rosdistro](https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc) 这个链接用浏览器或者镜像站下载密钥文件，再手动 `sudo apt-key add` 导入。

### 激活rosdep

`rosdep` 是 ROS 的**依赖管理工具**：它会读取你工作空间中各个包的 `package.xml`，解析出它们声明的系统依赖（比如某个库），然后自动调用 apt 等包管理器帮你装齐。`catkin_make` / `catkin build` 编译到一半报"找不到某某头文件"，八成就是 `rosdep install` 这一步没做好。

`rosdep` 的使用分两步：

- `rosdep init`：把 `rosdep` 的索引源（`sources.list.d/20-default.list`）写入 `/etc/ros/`，相当于告诉 rosdep"去哪里查依赖信息"。
- `rosdep update`：从索引源拉取最新的依赖数据库，之后 `rosdep install` 才能正常工作。

在国内，这两步**几乎必然失败**：`rosdep init` 要访问 `raw.githubusercontent.com`（被墙），`rosdep update` 要访问 `raw.githubusercontent.com` 和 `github.com`，同样大概率超时。所以国内环境的本质思路就两条：**要么给 rosdep 换一个国内镜像源（清华源），要么用工具/代理绕过墙**。下面五个方法我按推荐程度排序，任选其一即可。

!!! warning "注意"
    `rosdep init` 成功过一次后就不能再执行了（会报 already exists）。如果你反复执行导致报错，可以先 `rm -rf /etc/ros/rosdep/` 清掉再重来。

#### 方法一：使用清华源的方法（推荐）

参考：

- [清华源 ROS Distro](https://mirrors.tuna.tsinghua.edu.cn/help/rosdistro/)
- [清华源 手动模拟rosdep init](https://mirrors.tuna.tsinghua.edu.cn/help/github-raw/)

```bash
# 手动模拟rosdep init
# 手动模拟 rosdep init
mkdir -p /etc/ros/rosdep/sources.list.d/
curl -o /etc/ros/rosdep/sources.list.d/20-default.list -L https://mirrors.tuna.tsinghua.edu.cn/github-raw/ros/rosdistro/master/rosdep/sources.list.d/20-default.list

# 为 rosdep update 换源
export ROSDISTRO_INDEX_URL=https://mirrors.tuna.tsinghua.edu.cn/rosdistro/index-v4.yaml
rosdep update
```

!!! note "说明"
    清华源把 GitHub 的 raw 文件做了镜像，`curl` 这一行就是在模拟 `rosdep init` 写入索引源文件的过程；`ROSDISTRO_INDEX_URL` 则把 `rosdep update` 的数据源指向了清华镜像。整个方案不需要科学上网，也是我最推荐的一个。如果想一劳永逸，可以把最后两行 export 也写进 `~/.bashrc`。

#### 方法二：鱼香ros-rosdepc

```bash
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

!!! note "说明"
    `rosdepc` 是鱼香ROS维护的 `rosdep` 国内替代品，本质就是把源自动替换成国内镜像。如果你的项目里别人写的是 `rosdep install ...`，装了 `rosdepc` 后也可以用，只是命令名不同。这个方案在写 `config_ros.md` 的当时是我实际用过的，遇到报错就多试几次。

#### 方法三：科学上网法

```bash
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
echo "185.199.110.133 raw.githubusercontent.com " >> /etc/hosts
# 激活，失败的话，多试几次。
rosdep
```

!!! danger "踩坑"
    第一行 `export ALL_PROXY` 里的 `7897` 是我本地代理的端口，**请改成你自己的代理端口**，否则所有命令都会超时。另外 `echo ... >> /etc/hosts` 需要 root 权限，如果提示 `Permission denied`，先 `sudo -i` 或者改用 `sudo tee -a`。改 hosts 属于"绕过"手段，IP 失效是常事，失效了重新查 IP 再改即可。

#### 方法四：鱼香ros一键脚本

```bash
wget http://fishros.com/install -O fishros && . fishros
```

!!! note "说明"
    这是一键配置脚本，运行后会进入交互菜单，选择配置 ROS 环境相关的选项即可。适合不想手动折腾、想省事的同学，但脚本本身来自第三方，建议在干净的虚拟机上先试一次。

#### 方法五：6-rosdep

```bash
sudo apt-get install python3-pip
sudo pip3 install 6-rosdep
sudo 6-rosdep
sudo apt install python3-rosdep
sudo rosdep init
rosdep update
```

!!! tip "技巧"
    这个方案的思路是先用 `6-rosdep` 把国内源配置好（包括 hosts 和索引源），再正常执行 `rosdep init` 和 `rosdep update`，此时它们会走镜像。如果 `rosdep update` 仍有报错，多半是某个 yaml 文件拉不下来，重试几次通常能过。

## ROS2

本部分以为`Humble`为例。

### 安装ROS2

参考：[鱼香ros中文版ROS2 Humble](https://fishros.org/doc/ros2/humble/Tutorials.html)

!!! note "说明"
    Humble 对应 **Ubuntu 22.04**。ROS2 不再需要 `roscore`，节点靠 DDS 自动发现，这也是下面命令里没有 roscore 的原因。安装完成后 `export ROS_DOMAIN_ID=0` 这行建议保留在 `~/.bashrc` 里，它决定了你的节点在哪个"域"内通信，多机协作时很有用。

```bash
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

!!! danger "踩坑"
    `curl -sSL ... ros.key` 访问的是 `raw.githubusercontent.com`，在国内大概率超时。如果失败，可以用清华源镜像的 `ros.key`（参考上面 rosdep 的清华源方案），或者直接用科学上网后再执行这一行。

### 激活rosdep

ROS2 的 rosdep 使用方式和 ROS1 完全一致（`rosdep init` + `rosdep update`），同样会被墙，同样用清华源解决。

参考：

- [清华源 ROS Distro](https://mirrors.tuna.tsinghua.edu.cn/help/rosdistro/)
- [清华源 手动模拟rosdep init](https://mirrors.tuna.tsinghua.edu.cn/help/github-raw/)

```bash
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

!!! tip "技巧"
    `tee ... <<EOF` 这种方式适合"主机完全没网"的情况，把索引源内容手动写进文件里。其实两个方案殊途同归：只要 `/etc/ros/rosdep/sources.list.d/20-default.list` 存在且 `ROSDISTRO_INDEX_URL` 指向清华镜像，`rosdep update` 就能在国内网络下顺利跑通。之后编译你的 ROS2 工作空间时遇到依赖报错，就可以用 `rosdep install --from-paths src -y --ignore-src` 来一键补装依赖了。

到这里，ROS1 Noetic 和 ROS2 Humble 就都装好了。安装完成只是一个开始，下一步建议先跑一遍 `roscore`（ROS1）或者 `ros2 run demo_nodes_cpp talker`（ROS2），确认环境真的通了，再进入学习阶段。
