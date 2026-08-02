# 配置 pip

Python 生态里最常用的包管理工具就是 `pip`。默认情况下，pip 从 PyPI 官方源（pypi.org）下载包，服务器在国外，在国内网络环境下下载一个几十 MB 的包经常要等很久，还容易超时中断。这篇文章整理了我常用的几种换源方式和代理配置，从「临时换源」到「永久换源」都有，Linux 和 Windows 都给了对应的配置方法。

另外提一句，我自己在实验室配环境时，经常需要在多个 conda 环境、多台机器之间保持一致的 pip 源配置，永久换源的方法对 conda 环境里的 pip 同样生效（源配置写在用户目录下，不随环境变化），这一点很实用。换源不会改变包的内容和校验值，只是下载来源变了，可以放心使用。

## 配置国内镜像源

### 常用国内镜像源列表

在使用 `pip` 下载 Python 包时，由于默认源在国外，下载速度通常较慢。可以切换为以下国内镜像源：

| 镜像站 | 地址 |
| --- | --- |
| 清华大学 | `https://pypi.tuna.tsinghua.edu.cn/simple` |
| 阿里云 | `https://mirrors.aliyun.com/pypi/simple` |
| 腾讯云 | `https://mirrors.cloud.tencent.com/pypi/simple` |
| 中科大 | `https://pypi.mirrors.ustc.edu.cn/simple` |
| 豆瓣 | `https://pypi.douban.com/simple` |

这些镜像站都在同步 PyPI 官方包，日常使用没有问题。我个人习惯是下载速度优先选清华源，稳定优先选阿里云。需要注意的是，镜像站偶尔会有同步延迟或临时不可用，多备几个源随时切换比较省心。

如果你的网络环境特殊（比如教育网），也可以先分别用 `-i` 参数测一下每个源的实际下载速度，再决定永久换哪个源，这样能少走弯路。

### 临时换源

如果只需要在某次安装时临时使用镜像源，可以在 `pip install` 命令后加上 `-i` 参数。

例如，使用清华源安装某个包（将 `package_name` 替换为实际包名）：

```bash
pip install package_name -i https://pypi.tuna.tsinghua.edu.cn/simple
```

!!! tip "技巧"
    临时换源只在当前这一条命令生效，不改动任何配置文件，适合只装一两个包的场景。装包出问题想排除镜像因素时，也可以用这个方式对比验证。

### 永久换源（Linux）

方法一：使用 `pip config` 配置

```bash
# 清华源
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
# 腾讯云源
pip config set global.index-url https://mirrors.cloud.tencent.com/pypi/simple
```

方法二：手动配置 `.pip/pip.conf`

```bash
# 创建文件
mkdir -p ~/.pip
vim .pip/pip.conf

# 在文件中添加以下内容
[global]
index-url = https://mirrors.cloud.tencent.com/pypi/simple
[install]
trusted-host = mirrors.cloud.tencent.com
```

两种方法效果相同：`pip config set` 命令更省事，适合快速切换；手动写配置文件则更直观，方便同时管理 `index-url`、`trusted-host` 等多个选项。其中 `trusted-host` 用来解决部分镜像站 HTTPS 证书校验失败的问题，如果换源后 pip 报证书相关错误，把镜像域名加进 `trusted-host` 即可。

需要注意 `pip config set` 写入的是当前用户级的配置文件（Linux 下是 `~/.config/pip/pip.conf`），换系统用户或换机器后需要重新配置；配置文件放在用户目录下，不需要 sudo 权限，这一点比改系统级配置更安全。

### 永久换源（Windows）

首先，文件管理器文件路径地址栏敲：`%APPDATA%` 回车，快速进入 `C:\Users\你的用户名\AppData\Roaming` 文件夹中

接着，新建 `pip` 文件夹并在文件夹中新建 `pip.ini` 配置文件

最后，需要在`pip.ini` 配置文件内容，我们可以选择使用记事本打开，输入以下内容，并按下ctrl+s保存

```ini
[global]
index-url=https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host=pypi.tuna.tsinghua.edu.cn
```

Windows 上的 `pip.ini` 和 Linux 上的 `pip.conf` 内容格式完全一样，只是文件名和存放目录不同，配置完不用重启，新开的终端里立即生效。

## 配置代理

pip可以直接使用系统的全局代理环境变量（HTTP_PROXY 和 HTTPS_PROXY），因此可以直接在系统环境变量中设置代理环境变量。

将下方的IP和Port替换为你的代理的IP和Port。

```bash
# for linux
export HTTP_PROXY="http://127.0.0.1:7890"
export HTTPS_PROXY="http://127.0.0.1:7890"

# for windows (PowerShell)
$env:HTTP_PROXY="http://127.0.0.1:7890"
$env:HTTPS_PROXY="http://127.0.0.1:7890"
```

!!! note "说明"
    Linux 下 `export` 设置的环境变量只在当前终端会话生效，希望永久生效的话，把这两行追加到 `~/.bashrc` 末尾再 `source ~/.bashrc` 即可。Windows 下 `$env:` 同样只对当前 PowerShell 会话生效，需要写入系统环境变量才能全局生效。

## 换源后的验证

换源是否生效，最直接的验证方式就是随便装一个没装过的包（比如 `pip install requests` 这样的小包），观察终端里的下载地址：如果显示的是 `https://pypi.tuna.tsinghua.edu.cn/...` 这类镜像站域名，说明配置已经生效；如果还是 `https://pypi.org/...`，说明配置没被读到，检查一下配置文件路径和内容格式是否正确。

常见的配置不生效原因有三个：一是 Linux 下配置文件放错了路径（要放在 `~/.pip/pip.conf`，而不是项目目录）；二是 Windows 下文件扩展名写错（必须是 `pip.ini`，不是 `pip.ini.txt`）；三是修改配置后没有开新终端，旧终端的缓存里还是老配置。

## 常见问题

**问：豆瓣源现在还推荐吗？**

豆瓣源目前仍可访问，但更新频率较低，历史包比较全、新包偶尔缺同步，建议作为备选，日常优先清华、阿里云、腾讯云。

**问：换源后下载的包和官方源有什么区别吗？**

没有本质区别，国内镜像站都是对 PyPI 官方源的完整同步，包的内容和哈希一致。唯一的差异是同步时间，个别新发布的包可能晚几个小时出现。

**问：公司内网/离线环境装不了包怎么办？**

如果机器能访问内网源，可以把 `index-url` 指向内网 PyPI 服务；完全离线的机器则建议在有网的机器上 `pip download` 下好 wheel 包，再拷过去离线安装。

**问：镜像源和官方源的包能混装吗？**

可以。pip 每次下载都会校验包的哈希，无论是从哪个源下载，只要校验通过就都能正常安装。混装唯一的风险是镜像同步延迟导致极个别新包找不到，遇到这种情况切回官方源装那一个包即可。

**问：`trusted-host` 是什么？什么时候需要配置？**

`trusted-host` 用于声明「信任的镜像域名」。个别镜像站的 HTTPS 证书不被 pip 的证书库信任时，加装会直接失败，把它加进 `trusted-host` 可以跳过对该域名的证书校验。主流镜像站的证书都是正常的，一般用不到这个配置，遇到证书报错时再添加即可。

## 写在最后

换源和配置代理是 pip 在国内网络环境下最常用的两个优化手段，两者不冲突、可以共存：换源解决的是下载慢，代理解决的是访问不了（比如需要拉取 GitHub 上的依赖时）。我的习惯是日常开发用镜像源，涉及 GitHub 或特殊依赖时临时开代理，配合 `-i` 参数随时切换，灵活性最高。

这篇文章里的配置方法我在多台 Linux 机器和 Windows 机器上都验证过，无论你是刚入门还是老手，照着复制都能一次成功。如果遇到换源后仍然很慢的情况，先确认是不是走了代理或者 DNS 有问题，再回头看配置文件是否生效。
