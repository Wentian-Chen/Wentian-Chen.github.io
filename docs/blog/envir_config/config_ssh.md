# 配置 SSH

SSH 是我日常使用频率最高的工具：实验室的服务器、家里的 NAS、开发板，全靠 SSH 远程登录管理。相比直接接显示器操作，SSH 登录后在终端里就能完成几乎所有配置，还能配合 tmux 实现「断开连接、任务继续跑」的效果，这对动辄训练几十个小时的深度学习任务来说几乎是刚需。

不过 SSH 的默认配置安全性一般，直接暴露在公网上的 22 端口经常被扫描工具爆破。这篇文章在介绍安装配置的同时，也把安全要点整理了出来：用密钥代替密码登录、禁用密码登录、禁止 root 直接登录，这三件事做完，暴力破解基本就无能为力了。

## 安装和配置SSH

参考：[在linux中安装和配置SSH](https://www.cnblogs.com/huangjiabobk/p/18221683)

```bash
# 安装OpenSSH服务器（sshd）、nano编辑器、ufw
sudo apt-get install openssh-server nano ufw  -y

# 配置/etc/ssh/sshd_config
sudo nano /etc/ssh/sshd_config
# 通过添加或者删除#来屏蔽或者开启配置项
## 默认端口为22，可以通过更改Port 22来修改。
## 通过PermitRootLogin no禁用root登录
## 设置PubkeyAuthentication yes和PasswordAuthentication no 来启用密钥登录和禁止密码登录

# 手动添加公钥
sudo nano ~/.ssh/authorized_keys
# 将公钥复制进去

# ufw放行ssh端口
sudo ufw allow ssh

# 重启ssh服务 
sudo systemctl restart sshd
# 设置SSH服务开机启动
sudo systemctl enable sshd
```

配置的关键都在 `/etc/ssh/sshd_config` 里：`Port` 决定监听端口，`PermitRootLogin no` 禁止 root 直接登录，`PubkeyAuthentication yes` 配合 `PasswordAuthentication no` 实现只允许密钥登录。公钥一般来自本地电脑 `~/.ssh/id_rsa.pub`（或 `id_ed25519.pub`）文件的内容，把它追加进服务器的 `authorized_keys` 即可，每行一个公钥。

最后两步不要漏：`ufw allow ssh` 是给防火墙放行 22 端口，不放行的话即使 SSH 服务正常运行也连不进来；`systemctl enable sshd` 设置开机自启，否则服务器重启后 SSH 就断了。改完配置记得执行重启命令让配置生效，我自己就曾因为忘记重启服务，排查了半天才发现改的配置根本没加载。

!!! danger "踩坑"
    修改 `sshd_config` 时注意操作顺序：先把公钥写进 `authorized_keys`，用密钥登录确认没问题，最后再把 `PasswordAuthentication` 改为 `no`。否则一旦密钥没配置好，远程机器就再也连不进去了，只能跑去机房接显示器救援。

## Rsync 大文件断点续传

SCP不支持断点续传，使用Rsync能够解决大文件传输因网络等原因出现中断，希望继续传输的问题。

```bash
# rsync [选项] 源路径 目标路径
# -a: 归档模式（保留权限、时间戳等）
# -v: 显示进度
# -z: 压缩传输（省流量）
# -P: 显示进度条并支持断点续传

# 上传
rsync -avzP ./local_folder/ user@remote:/remote/folder/

# 下载
rsync -avzP user@remote:/remote/folder/ ./local_folder/ 
```

传输大文件（比如数据集、模型权重）我基本只用 rsync：中断之后重新执行同一条命令，它只会续传剩余部分，不用从头再来。注意源路径末尾的 `/` 表示同步目录内容而不是目录本身，这个细节容易踩坑。另外 rsync 默认走 SSH 通道，所以它复用的是 SSH 的认证和加密，只要 SSH 能连上，rsync 就能用。

## TMUX 终端复用器

用于训练等任务的后台挂机，解决因终端关闭导致训练任务中断的问题。

```bash
# 安装
sudo apt install tmux 

# tmux new -s 自定义的任务名 
# -s 代表 session (会话) 名字
tmux new -s my_training 

# 启动任务
python train_model.py

# 挂起终端
tmux detach 

# 查看后台任务
tmux ls

# 重新连接
# -t 代表 自定义的任务名
tmux attach -t my_training 

# 语法：tmux kill-session -t 自定义的任务名 
tmux kill-session -t my_training
```

tmux 的使用流程很固定：`tmux new -s 名字` 建会话、跑任务、`tmux detach` 挂起、`tmux ls` 查看、`tmux attach -t 名字` 回来。即使本地电脑关机、网络断开，服务器上会话里的训练进程也会继续运行，这是远程训练任务的标准做法。

还有一个使用细节：一个会话里同时只能有一个前台终端，如果之前 attach 过没 detach，再 attach 时会提示冲突。遇到这种情况，先用 `tmux ls` 看看状态，再决定是 attach 回去还是直接 kill 重建会话，不用慌。

## 配置代理

### 服务器使用个人电脑的本地代理

```bash
# 语法：ssh -R [服务器端口]:[本地IP]:[本地端口] user@server
# 服务器端口可以自定义，示例的服务器使用7890，个人电脑本地代理是7897
ssh -R 7890:127.0.0.1:7897 user@server_ip

# 在服务器设置环境变量，让服务器的终端通过7890端口forward到本地的7897端口
export all_proxy=http://127.0.0.1:7890

# 测试
curl google.com
# 测试成功则会返回HTML格式的信息
```

`-R` 是远程转发（remote forward）：把服务器的 7890 端口反向映射到本机的 7897 端口，服务器上的流量就能借道本地代理出去。这样即使服务器本身没有代理，也能访问 Google、GitHub 这些站点，下载代码和模型会方便很多。

### 个人电脑使用服务器的代理

```bash
# 语法：ssh -L [本地端口]:[服务器IP]:[服务器端口] user@server
ssh -L 7890:127.0.0.1:7890 user@server_ip
```

`-L` 是本地转发（local forward），反过来把本地 7890 端口映射到服务器的 7890 端口。如果服务器上有 HTTP/SOCKS 代理服务，本机就能通过服务器中转访问内网资源，比如实验室内网的 GitLab 或者私有模型仓库。中转之后的流量从服务器出口发出，对访问目标来说来源就是服务器的 IP，这个特性在访问一些有 IP 白名单限制的服务时非常有用。

## 端口转发

将服务器上的端口转发到本地，使本地能够访问到服务器上的端口。如在远程服务器上跑了一个 Jupyter Notebook (端口 8888) 或者 TensorBoard (端口 6006)，经过端口转发后，在本地就可以通过 http://localhost:本地端口 访问 Jupyter Notebook，通过 http://localhost:本地端口 访问 TensorBoard。

```bash
语法：ssh -L [本地端口]:[服务器IP]:[服务器端口] user@server
ssh -L 8080:localhost:8888 user@remote_server
```

端口转发是远程开发体验的关键一环：服务器上跑的 Jupyter、TensorBoard、Gradio 这类 Web 服务默认只监听服务器本地，加上一条 `-L` 转发，本地浏览器直接打开 `http://localhost:8080` 就能访问，不用把服务暴露到公网，安全又方便。执行转发后保持这个 SSH 会话不退出即可，断开转发也就断了。

一个小建议：如果端口转发在频繁使用，可以考虑把常用转发写进 `~/.ssh/config`，以后一条短命令就能连上；不过这一步属于锦上添花，先用好上面几条基础命令就够了。

## 写在最后

SSH 这套组合拳（密钥登录 + rsync 传数据 + tmux 挂任务 + 端口转发看界面）基本覆盖了我远程开发的所有场景，从配环境到跑训练再到看结果，全程不需要物理接触服务器。安全性方面，只要坚持「密钥登录、禁用密码、关掉 root 直连」这三个原则，服务器放在公网也不用太担心被爆破。

最后提醒一点：任何 SSH 配置改动都有把机器锁在门外的风险，改之前先确认自己还有一条后路（比如另一台机器能连、或者物理控制台可用），再动手也不迟。祝大家远程开发愉快。
