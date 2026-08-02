# 配置 Docker

Docker 和虚拟机不同：虚拟机模拟了一整台硬件（CPU、内存、网卡都要虚拟化），资源开销大、启动慢；而 Docker 容器直接共享宿主机的内核，只隔离进程的文件系统、网络、用户等命名空间，启动只要秒级，资源占用也小得多。

对于做机器人、深度学习的人来说，Docker 最大的好处是环境一致性：把 CUDA、PyTorch、ROS 这些动不动就互相冲突的环境打包进镜像，换机器、换显卡都不用重新配环境，队友之间分享环境也只需要分享一个镜像。这篇文章记录的是我在 Ubuntu 上从零配置 Docker 的完整流程，包括免 sudo、国内镜像源、X11 图形转发、GPU 支持、arm64 模拟器这些我实际用到的功能。

## 安装

参考：[Ubuntu上安装和使用Docker](https://blog.csdn.net/heyl163_/article/details/131503469?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522a85fa38e1378b0d13d689d1d4624c900%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=a85fa38e1378b0d13d689d1d4624c900&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-131503469-null-null.142^v102^pc_search_result_base2&utm_term=ubuntu%20docker%20&spm=1018.2226.3001.4187)

```bash
# 更新apt源
sudo apt update

# 安装必要工具
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common lrzsz -y

# 添加Docker源
sudo curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

# 添加Docker源（for amd64）
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

# 添加Docker源（for arm64）
sudo add-apt-repository "deb [arch=arm64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

# 安装Docker
sudo apt update
sudo apt-get install docker-ce -y

# 检查Docker是否安装成功
sudo docker version
```

amd64 和 arm64 的源地址只有 `[arch=...]` 一处不同，按自己机器的架构执行对应那一行即可，不确定架构的话两行都执行也没有影响。`$(lsb_release -cs)` 会自动展开为当前系统的版本代号（如 `noble`），无需手动修改。

## 免输入sudo使用docker

默认情况下 docker 命令需要 sudo 权限，把当前用户加入 docker 用户组后即可免 sudo 使用：

```bash
# 查询docker用户组是否存在
getent group docker

# 添加docker用户组，一般已存在，不需要执行
sudo groupadd docker

# 将登陆用户加入到docker用户组中
sudo gpasswd -a $USER docker

# 更新用户组
newgrp docker

# 测试docker命令是否可以使用sudo正常使用
docker version
# 注：此时只在当前终端生效。需要注销用户登录或者重启才能全局生效。
```

!!! note "说明"
    加入 docker 用户组等同于获得了 docker 的完整权限（等价于 root），只建议在个人电脑或自己管理的服务器上这样配置；多用户共享的机器请谨慎。

## 配置镜像源

当前国内没有稳定的免费镜像源，各大公益服务商因为不可抗力的原因，都关闭了镜像源。

目前可以使用`docker.xuanyuan.me`的免费镜像源（非广告），免费版不保证质量。同时轩辕也提供付费方案，自行参考。

```bash
# 编辑docker配置文件
sudo nano /etc/docker/daemon.json

# 在文件中添加如下内容
"registry-mirrors": [
        "https://docker.xuanyuan.me",
    ],

# 重启docker服务    
systemctl daemon-reload
systemctl restart docker
```

`daemon.json` 是 JSON 格式，`registry-mirrors` 要放在最外层的 `{}` 里，注意补全逗号和引号，否则 Docker 服务会启动失败。配置镜像源之后拉取镜像的速度会有明显提升，尤其是经常拉 Ubuntu、PyTorch 这类大镜像的场景。

## Docker配置X11转发

由于Docker是headless模式，需要配置X11转发才能转发在Docker中运行GUI程序到本地桌面。

启动x11转发的顺序是先启动x11服务，再启动docker容器。

```bash
# 确认主机的DISPLAY属性
echo $DISPLAY
## 如果没有输出，则
echo "export DISPLAY=:0" >> ~/.bashrc
source ~/.bashrc

# 开启xhost
xhost +local:docker  # 临时允许 Docker 容器连接

# 启动Docker容器
docker run ...
```

Docker 容器默认没有显示环境，想让容器里的 GUI 程序（比如 rviz、gazebo）显示到本地桌面，就要做 X11 转发。核心就是两条：主机设置好 `DISPLAY` 环境变量，并执行 `xhost +local:docker` 临时放行容器连接；`docker run` 时还要挂载 X11 socket 和临时目录（不同镜像参数略有差异），容器里才能正常打开窗口。

## 安装NVIDIA Container Toolkit

Docker 本身不能直接访问 GPU，需要 NVIDIA Container Toolkit（以前称为 nvidia-docker2）来桥接 Docker 和 NVIDIA GPU。

```bash
# 添加 NVIDIA GPG 密钥和仓库
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -sL https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 更新 apt 缓存并安装 NVIDIA Container Toolkit
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# 配置 Docker 以使用 NVIDIA 运行时
sudo nvidia-ctk runtime configure --runtime=docker

# 重启 Docker
sudo systemctl restart docker
```

装好之后，`docker run` 时加上 `--gpus all` 参数（或用 `--runtime=nvidia`）就能把 GPU 传入容器。这是深度学习容器化的基础，我的 PyTorch 训练环境都是基于这个方案搭的。

## 在AMD64上启动ARM64容器

在AMD64上可以通过QEMU模拟器启动和配置ARM64容器，但是无法访问AMD64本地的GPU。

完成以下指令后，即可拉取和运行ARM64容器。

```bash
# 安装 QEMU 用户态模拟
sudo apt-get update
sudo apt-get install qemu-user-static qemu-user binfmt-support

# 注册 ARM64 架构支持
docker run --rm --privileged multiarch/qemu-user-static --reset
```

!!! warning "注意"
    QEMU 用户态模拟只是指令集层面的翻译，性能损耗明显，而且容器内无法访问宿主机的 GPU，所以只适合编译、测试这类轻量场景，指望靠它跑深度学习训练是不现实的。

## 解决使用docker build 时的下载速度慢问题

参考：[Use a proxy server with the Docker CLI](https://docs.docker.com/engine/cli/proxy/)

方法一：使用`--build-arg`

```bash
docker build \
--build-arg HTTP_PROXY="http://proxy.example.com:3128" \
--build-arg HTTPS_PROXY="https://proxy.example.com:3129" \
.
```

这些参数只在build期间生效，不会保存在镜像中。Docker build会自动根据`--build-args`设置`HTTP_PROXY`, `HTTPS_PROXY`,`http_proxy`, `https_proxy`环境变量。这种方法不会保留环境变量到image里。

如果需要在Dockerfile的指令里显式使用`HTTP_PROXY`, `HTTPS_PROXY`，则需要确保Dockerfile文件里使用

```dockerfile
ARG HTTP_PROXY
ARG HTTPS_PROXY
ARG NO_PROXY

ENV HTTP_PROXY=$HTTP_PROXY
ENV HTTPS_PROXY=$HTTPS_PROXY
ENV NO_PROXY=$NO_PROXY
```

这种方式会在image写入环境变量，这是不推荐的。

方法二：使用`~/.docker/config.json`

```json
# 编辑文件
sudo nano ~/.docker/config.json 

# 在~/.docker/config.json添加以下内容
{
  "proxies": {
    "default": {
      "httpProxy": "你的代理服务器地址和端口",
      "httpsProxy": "你的代理服务器地址和端口",
      "noProxy": "*.test.example.com,.example.org,127.0.0.0/8"
    }
  }
}
```

`docker build` 时经常需要在容器里拉取依赖（比如 apt、pip 下载），国内网络下速度往往很慢。方法一用 `--build-arg` 传入代理，只在构建期间生效、不污染镜像，是我最常用的方式；方法二配置在 `~/.docker/config.json` 里，对后续所有容器生效，适合代理长期固定不变的场景。

## 修改Docker的默认存储路径

Docker 默认会将镜像和容器存储在`/var/lib/docker`目录下，这个目录通常在根挂载点上。

```bash
# 先停Docker服务
sudo systemctl stop docker

# 在 /home 挂载点上创建一个新的目录，例如 docker_data。
sudo mkdir -p /home/docker_data

# 将 /var/lib/docker 下的所有内容复制到新创建的目录。
sudo rsync -aqxP /var/lib/docker/ /home/docker_data/

# 修改 Docker 的配置文件 daemon.json。
sudo nano /etc/docker/daemon.json

# 添加或修改 data-root 字段，指向新的路径：
"data-root": "/home/docker_data"

# 启动 Docker 服务：
sudo systemctl start docker

# 确认 Docker 运行正常且所有镜像和容器都已迁移后，可以删除旧的 /var/lib/docker 目录以释放空间。务必在此步骤前确认 Docker 运行无误！
sudo rm -rf /var/lib/docker
```

!!! danger "踩坑"
    删除 `/var/lib/docker` 是不可逆操作。执行最后一步之前，务必确认 Docker 启动正常、所有镜像和容器都能正常列出和使用，否则数据就找不回来了。如果迁移后一切正常，再执行 `rm -rf` 释放根分区空间。
