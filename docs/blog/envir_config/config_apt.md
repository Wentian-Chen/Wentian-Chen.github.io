# 配置国内 APT 镜像源

Ubuntu 默认使用的官方源服务器在国外，在国内网络环境下执行 `apt update` 经常超时，下载软件包的速度也慢得让人着急。把 APT 源切换为国内镜像站之后，速度和稳定性都会有明显提升。

我自己在实验室和家里维护了好几台机器，从 x86 台式机到 ARM 开发板都遇到过这个问题，所以这篇文章把 amd64 和 arm64 两种架构、以及 Ubuntu 24.04 前后的两种源文件格式都整理全了，按需复制对应段落即可。替换的核心思路是用 `sed` 把源文件里的官方域名一次性替换成镜像站域名，简单且可重复执行。

!!! note "说明"
    修改源文件之前一定要先备份，万一替换出错，或者以后想切回官方源，都能一键恢复。下面的每个段落都保留了备份步骤。

## 镜像站对比

| 镜像站 | amd64 域名 | arm64 域名 | 备注 |
| --- | --- | --- | --- |
| 中科大 | mirrors.ustc.edu.cn | mirrors.ustc.edu.cn | 教育网内速度极快 |
| 清华 | mirrors.tuna.tsinghua.edu.cn | mirrors.tuna.tsinghua.edu.cn | 同步频率高，比较稳定 |
| 阿里云 | mirrors.aliyun.com | mirrors.aliyun.com | 公网带宽大，家宽友好 |
| 华为云 | repo.huaweicloud.com | repo.huaweicloud.com | 运营商线路表现稳定 |

需要说明的是，amd64 与 arm64 的软件源域名并不相同：amd64 走 `archive.ubuntu.com`（安全更新源为 `security.ubuntu.com`），而 arm64 统一走 `ports.ubuntu.com`，这也是下面命令要区分两类架构的原因。镜像站的域名后缀（`ubuntu` / `ubuntu-ports`）也要和架构对应，否则会拉到错误架构的软件包。

如果你在配置时不确定自己系统的版本号，可以先用 `lsb_release -a` 看一下发行版本；不确定架构，用 `uname -m` 确认一下再动手，避免把 24.04 前后的命令搞混。

## For Ubuntu (amd64)

参考文档：

- [ubuntu amd64 清华源配置指南](https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/)
- [ubuntu amd64 中科大源配置指南](https://mirrors.ustc.edu.cn/help/ubuntu.html)

### Ubuntu 24.04 以下版本（不含 24.04）

24.04 之前，Ubuntu 使用 `/etc/apt/sources.list` 单文件配置软件源。这个文件里每一行是一条源记录，`sed` 替换的目标就是记录里的域名部分。下面的命令把所有镜像站都列出来了，实际使用时挑一个执行即可，不用全跑。

```bash
# 在修改配置文件之前，建议先备份原始的sources.list文件：
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak

# 使用sed命令一键替换
## 中科大源
sudo sed -i 's@//.*archive.ubuntu.com@//mirrors.ustc.edu.cn@g' /etc/apt/sources.list
sudo sed -i 's@//.*security.ubuntu.com@//mirrors.ustc.edu.cn@g' /etc/apt/sources.list
## 清华源
sudo sed -i 's@//.*archive.ubuntu.com@//mirrors.tuna.tsinghua.edu.cn@g' /etc/apt/sources.list
sudo sed -i 's@//.*security.ubuntu.com@//mirrors.tuna.tsinghua.edu.cn@g' /etc/apt/sources.list
## 阿里源
sudo sed -i 's@//.*archive.ubuntu.com@//mirrors.aliyun.com@g' /etc/apt/sources.list
sudo sed -i 's@//.*security.ubuntu.com@//mirrors.aliyun.com@g' /etc/apt/sources.list
## 华为源
sudo sed -i 's@//.*archive.ubuntu.com@//repo.huaweicloud.com@g' /etc/apt/sources.list
sudo sed -i 's@//.*security.ubuntu.com@//repo.huaweicloud.com@g' /etc/apt/sources.list

# 更新
sudo apt update
```

上面四组 `sed` 分别对应四个镜像站，只执行你想用的那一组即可。替换的原理是把 `archive.ubuntu.com` 和 `security.ubuntu.com` 两个官方域名都改成本地镜像站域名。

### Ubuntu 24.04 及以上版本

从 24.04 开始，Ubuntu 改用 `/etc/apt/sources.list.d/ubuntu.sources`（deb822 格式）管理软件源，命令的替换目标也相应变化。deb822 格式比旧格式信息更清晰，每条源都带架构、套件、组件等字段，但替换域名的思路和旧版本完全一样。

```bash
# 在修改配置文件之前，建议先备份原始的ubuntu.sources文件：
sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.bak

# 使用sed命令一键替换
## 中科大源
sudo sed -i 's@//.*archive.ubuntu.com@//mirrors.ustc.edu.cn@g' /etc/apt/sources.list.d/ubuntu.sources
sudo sed -i 's@//.*security.ubuntu.com@//mirrors.ustc.edu.cn@g' /etc/apt/sources.list.d/ubuntu.sources
## 清华源
sudo sed -i 's@//.*archive.ubuntu.com@//mirrors.tuna.tsinghua.edu.cn@g' /etc/apt/sources.list.d/ubuntu.sources
sudo sed -i 's@//.*security.ubuntu.com@//mirrors.tuna.tsinghua.edu.cn@g' /etc/apt/sources.list.d/ubuntu.sources
## 阿里源
sudo sed -i 's@//.*archive.ubuntu.com@//mirrors.aliyun.com@g' /etc/apt/sources.list.d/ubuntu.sources
sudo sed -i 's@//.*security.ubuntu.com@//mirrors.aliyun.com@g' /etc/apt/sources.list.d/ubuntu.sources
## 华为源
sudo sed -i 's@//.*archive.ubuntu.com@//repo.huaweicloud.com@g' /etc/apt/sources.list.d/ubuntu.sources
sudo sed -i 's@//.*security.ubuntu.com@//repo.huaweicloud.com@g' /etc/apt/sources.list.d/ubuntu.sources

# 更新
sudo apt update
```

!!! warning "注意"
    24.04 及以上的系统不要再往 `/etc/apt/sources.list` 里写源了，deb822 格式下该文件默认为空，改它不会生效，一定要替换 `sources.list.d/ubuntu.sources`。

## For Ubuntu (arm64)

arm64（如树莓派、Jetson 系列开发板）走的是 `ports.ubuntu.com` 源，替换规则与 amd64 不同，注意区分。

参考文档：

- [ubuntu arm64 清华源配置指南](https://mirror.tuna.tsinghua.edu.cn/help/ubuntu-ports/)
- [ubuntu arm64 中科大源配置指南](https://mirrors.ustc.edu.cn/help/ubuntu-ports.html)

### Ubuntu 24.04 以下版本（不含 24.04）

```bash
# 在修改配置文件之前，建议先备份原始的sources.list文件：
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak

# 使用sed命令一键替换
## 中科大源
sudo sed -i 's@//ports.ubuntu.com@//mirrors.ustc.edu.cn@g' /etc/apt/sources.list
## 清华源
sudo sed -i 's@//ports.ubuntu.com@//mirrors.tuna.tsinghua.edu.cn@g' /etc/apt/sources.list
## 阿里源
sudo sed -i 's@//ports.ubuntu.com@//mirrors.aliyun.com@g' /etc/apt/sources.list
## 华为源
sudo sed -i 's@//ports.ubuntu.com@//repo.huaweicloud.com@g' /etc/apt/sources.list

# 更新
sudo apt update
```

### Ubuntu 24.04 及以上版本

```bash
# 在修改配置文件之前，建议先备份原始的ubuntu.sources文件：
sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.bak

# 使用sed命令一键替换
## 中科大源
sudo sed -i 's@//ports.ubuntu.com@//mirrors.ustc.edu.cn@g' /etc/apt/sources.list.d/ubuntu.sources
## 清华源
sudo sed -i 's@//ports.ubuntu.com@//mirrors.tuna.tsinghua.edu.cn@g' /etc/apt/sources.list.d/ubuntu.sources
## 阿里源
sudo sed -i 's@//ports.ubuntu.com@//mirrors.aliyun.com@g' /etc/apt/sources.list.d/ubuntu.sources
## 华为源
sudo sed -i 's@//ports.ubuntu.com@//repo.huaweicloud.com@g' /etc/apt/sources.list.d/ubuntu.sources

# 更新
sudo apt update
```

## 常见问题

**问：为什么要区分 24.04 前后？**

因为 24.04 起 Ubuntu 把源配置文件从 `/etc/apt/sources.list` 迁移到了 deb822 格式的 `ubuntu.sources`，文件名和路径都变了，命令也要跟着改，混用会导致修改无效。

**问：amd64 和 arm64 的命令可以通用吗？**

不能。amd64 替换的是 `archive.ubuntu.com` 和 `security.ubuntu.com`，arm64 替换的是 `ports.ubuntu.com`，两者的目标镜像路径也不同（`ubuntu` / `ubuntu-ports`），混用会替换不到任何内容，或者拉取错误架构的软件包。

## 写在最后

换源是 Ubuntu 装机后必做的第一件事，也是性价比最高的优化之一。这篇文章把 amd64 / arm64 两种架构、24.04 前后两种文件格式、四个常用镜像站都覆盖了，之后换机器、重装系统，直接回来复制对应段落就行。

最后再强调一遍两个最容易踩的点：一是务必先备份，二是 24.04 及以上的系统不要再去改 `/etc/apt/sources.list`。只要这两点注意到位，换源这个操作基本不会出问题。如果 `sudo apt update` 之后报「Release file is not valid yet」或者 404 错误，通常是镜像同步延迟导致的，等一段时间或者换个镜像站即可。

## 验证配置是否生效

替换源并执行 `sudo apt update` 之后，观察输出：如果列出的是 `http://mirrors.xxx.edu.cn/ubuntu ...` 这类镜像站地址，而不是官方源域名，说明替换成功。之后再安装一个小软件包（比如 `htop`）感受一下下载速度，确认无误即可正常使用。

如果之后想恢复官方源，直接用之前备份的文件覆盖回去，再执行一次 `sudo apt update` 即可，全程无需重装系统。
