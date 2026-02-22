<!--
 * @Author: boboji11 wendychen112@qq.com
 * @Date: 2026-02-22 20:23:30
 * @LastEditors: boboji11 wendychen112@qq.com
 * @LastEditTime: 2026-02-22 20:38:45
 * @FilePath: \WentianChen.github.io\docs\blog\config_pip.md
 * @Description: 
 * 
 * Copyright (c) 2026 by boboji11 , All Rights Reserved. 
-->
# 配置 Pip
## 配置国内镜像源
### 常用国内镜像源列表

在使用 `pip` 下载 Python 包时，由于默认源在国外，下载速度通常较慢。可以切换为以下国内镜像源：

* **清华大学:** `https://pypi.tuna.tsinghua.edu.cn/simple`
* **阿里云:** `https://mirrors.aliyun.com/pypi/simple`
* **腾讯云:** `https://mirrors.cloud.tencent.com/pypi/simple`
* **中科大:** `https://pypi.mirrors.ustc.edu.cn/simple`
* **豆瓣:** `https://pypi.douban.com/simple`

### 临时换源 
如果只需要在某次安装时临时使用镜像源，可以在 `pip install` 命令后加上 `-i` 参数。

例如，使用清华源安装某个包（将 `package_name` 替换为实际包名）：
```
pip install package_name -i https://pypi.tuna.tsinghua.edu.cn/simple
```
### 永久换源 For linux
方法一：使用`pip config`配置
```
# 清华源
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
# 腾讯云源
pip config set global.index-url https://mirrors.cloud.tencent.com/pypi/simple
```
方法二：手动配置`.pip/pip.conf`
```
# 创建文件
mkdir -p ~/.pip
vim .pip/pip.conf

# 在文件中添加以下内容
[global]
index-url = https://mirrors.cloud.tencent.com/pypi/simple
[install]
trusted-host = mirrors.cloud.tencent.com
```
### 永久换源 For Windows
首先，文件管理器文件路径地址栏敲：`%APPDATA%` 回车，快速进入 `C:\Users\你的用户名\AppData\Roaming` 文件夹中 

接着，新建 `pip` 文件夹并在文件夹中新建 `pip.ini` 配置文件 

最后，需要在`pip.ini` 配置文件内容，我们可以选择使用记事本打开，输入以下内容，并按下ctrl+s保存
```
[global]
index-url=https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host=pypi.tuna.tsinghua.edu.cn
```
## 配置代理
pip可以直接使用系统的全局代理环境变量（HTTP_PROXY 和 HTTPS_PROXY），因此可以直接在系统环境变量中设置代理环境变量。

将下方的IP和Port替换为你的代理的IP和Port。
```
# for linux
export HTTP_PROXY="http://127.0.0.1:7890"
export HTTPS_PROXY="http://127.0.0.1:7890"

# for windows (PowerShell)
$env:HTTP_PROXY="http://127.0.0.1:7890"
$env:HTTPS_PROXY="http://127.0.0.1:7890"
```