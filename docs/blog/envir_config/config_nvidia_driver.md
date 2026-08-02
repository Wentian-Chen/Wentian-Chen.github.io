# 配置 NVIDIA 显卡驱动

NVIDIA 显卡在 Ubuntu 上默认使用开源的 nouveau 驱动，功能残缺、性能也很差，跑深度学习、CUDA 应用之前的第一步永远是先装好官方驱动。装驱动这件事本身不难，但坑不少：驱动版本怎么选、开源版和闭源版有什么区别、开启 Secure Boot 之后为什么重启总是进不了系统……这篇文章把安装步骤和完整的排查流程都整理出来，我在多台笔记本和台式机上验证过。

需要提前说明的是：驱动装好后要重启系统才会加载内核模块，所以「安装完成后重启、然后 `nvidia-smi` 验证」是标准流程，不要装完就急着跑训练，否则大概率报 CUDA 找不到设备。

## 安装 NVIDIA 显卡驱动

参考：

- [NVIDIA Official Drivers Installation Guide](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/introduction.html)
- [Ubuntu Drivers Installation Guide](https://documentation.ubuntu.com/server/how-to/graphics/install-nvidia-drivers/index.html)

### 驱动版本怎么选

根据ubuntu官方的[Ubuntu Drivers Installation Guide](https://documentation.ubuntu.com/server/how-to/graphics/install-nvidia-drivers/index.html)指南，官方提供两种类型的NVIDIA Driver版本：

1. Unified Driver Architecture (UDA) drivers，which are recommended for the generic desktop use。（适用个人电脑，训练模型，科学计算 等场景）
2. Enterprise Ready Drivers，which are recommended on servers and for computing tasks. （适用于企业）

简单说，个人电脑、训练模型、科学计算这类场景选 UDA 版本就够了；服务器和重计算任务才需要考虑 Enterprise 版本。我的经验和官方推荐一致：普通用户直接用 UDA。

驱动还分为开源（`open`）与闭源（proprietary）两种内核模块实现。官方推荐安装开源，即`open`版本，适配笔记本，台式等。我自己用下来 open 版本稳定性没有问题，新显卡（比如 30/40 系列）优先选 open 版即可。

!!! warning "注意"
    如果电脑开启了 Secure Boot（安全启动），安装驱动后第一次重启会进入蓝色的 MOK（Machine Owner Key）管理界面，需要输入之前设置的管理员密码确认导入密钥，否则驱动内核模块不会被加载。这是正常流程，不是安装失败，耐心走完即可。如果重启后没有出现这个界面，说明 Secure Boot 未开启，跳过就好。

### 安装步骤

官方推荐使用`ubuntu-drivers tool`完成驱动安装，尤其是电脑启用了`Secure Boot`，步骤如下：

```bash
# 列出可安装的驱动版本 
sudo ubuntu-drivers list

# 方法一：自动检测最适合电脑的版本进行安装
sudo ubuntu-drivers install

# 指定想安装的版本，以535为例
sudo ubuntu-drivers install nvidia:535

# 指定想安装的版本，以535-open为例
sudo ubuntu-drivers install nvidia:535-open
```

`ubuntu-drivers install` 会自动检测显卡并安装推荐版本；想自己指定版本，就用 `nvidia:版本号` 的写法，带 `-open` 后缀的是开源内核模块版本。安装完成后重启系统，执行 `nvidia-smi` 能看到显卡型号、驱动版本和显存占用，就说明驱动正常工作了。

## 问题排除流程

如果按上面的步骤安装后 `nvidia-smi` 依然报错，按下面的流程一步步排查即可。我把排查步骤按阶段分组，每组前面加了一行说明，照着顺序走基本能定位问题。以下命令都来自我实际排障时使用的清单，直接复制执行即可。

### 排查前的准备

先把系统更新到最新，并确认编译内核模块所需的工具链已经装好：

```bash
# 前提：
sudo apt update 
sudo apt install linux-headers-$(uname -r) build-essential
```

### 检查硬件与驱动状态

先确认系统是否真的检测到了 NVIDIA 显卡，以及驱动包是否已经下载、安装：

```bash
# 是否检测到物理显卡
lspci | grep -i nvidia
# 是否已经下载了nvidia驱动包
ls /var/cache/apt/archives | grep nvidia-driver
# 检测是否已经安装了驱动
dpkg -l | grep nvidia-driver
```

如果 `lspci | grep -i nvidia` 没有任何输出，先检查显卡是否插好、BIOS 里是否禁用了独立显卡，这步过不了后面都是白搭。

### 清理旧驱动

多个驱动版本互相冲突是最常见的故障来源，先把所有 NVIDIA 相关包清干净再重新装：

```bash
# 移除已安装的驱动
sudo apt purge *nvidia*
sudo apt autoremove
sudo apt autoclean
```

### 添加 PPA 并安装指定版本

Ubuntu 官方源里的驱动版本可能偏旧，先添加 `graphics-drivers` PPA 拿到最新驱动，再安装指定版本：

```bash
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
```

!!! note "说明"
    `nvidia-driver-570` 这类 metadata 包会自动把对应的驱动、DKMS 组件等依赖一起装好，日常使用装它就够了；`nvidia-dkms-570` 是内核模块包，metadata 类型会自动带上，不需要单独安装。

### 检查内核模块

驱动安装后，确认内核模块是否正确加载、版本与驱动是否匹配：

```bash
# 检查内核模块版本
cat /proc/driver/nvidia/version
# 内核模块信息
sudo modinfo nvidia
```

### Secure Boot 与 MOK 密钥

如果系统开启了 Secure Boot，驱动内核模块无法通过签名校验，需要自己生成密钥并导入 MOK：

```bash
# 检测安全模式
mokutil --sb-state
# 生成密码
sudo apt install mokutil openssl
sudo openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -nodes -days 36500 -subj "/CN=NVidia Driver/"
sudo mokutil --import MOK.der
```

`mokutil --sb-state` 输出 `SecureBoot enabled` 就说明处于安全模式。执行导入后重启，进入 MOK 管理界面选择 Enroll MOK，输入 `mokutil --import` 时设置的密码，驱动就能通过签名校验了。

### 加载驱动并最终验证

最后检查驱动是否加载成功：

```bash
# 验证驱动加载
lsmod | grep nvidia
# 加载驱动
sudo modprobe nvidia
# 最后核验
nvidia-smi
```

`nvidia-smi` 能正常输出显卡型号、驱动版本和显存占用，说明驱动安装彻底成功。如果 `lsmod | grep nvidia` 有输出但 `nvidia-smi` 报错，多半是内核模块与驱动版本不匹配，回到「清理旧驱动」那一步重装一遍。整个排查流程走下来，绝大多数驱动问题都能定位到「没装全、版本冲突、签名没过」这三类原因之一。
