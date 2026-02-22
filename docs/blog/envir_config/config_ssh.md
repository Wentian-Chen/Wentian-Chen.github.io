# 配置SSH
## 安装和配置SSH
参考：[在linux中安装和配置SSH](https://www.cnblogs.com/huangjiabobk/p/18221683)
```
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

## Rsync 大文件断点续传
SCP不支持断点续传，使用Rsync能够解决大文件传输因网络等原因出现中断，希望继续传输的问题。
```
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
## TMUX 终端复用器
用于训练等任务的后台挂机，解决因终端关闭导致训练任务中断的问题。
```
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
## 配置代理
### 服务器使用个人电脑的本地代理
```
# 语法：ssh -R [服务器端口]:[本地IP]:[本地端口] user@server
# 服务器端口可以自定义，示例的服务器使用7890，个人电脑本地代理是7897
ssh -R 7890:127.0.0.1:7897 user@server_ip

# 在服务器设置环境变量，让服务器的终端通过7890端口forward到本地的7897端口
export all_proxy=http://127.0.0.1:7890

# 测试
curl google.com
# 测试成功则会返回HTML格式的信息
```
### 个人电脑使用服务器的代理
```
# 语法：ssh -L [本地端口]:[服务器IP]:[服务器端口] user@server
ssh -L 7890:127.0.0.1:7890 user@server_ip
```
## 端口转发
将服务器上的端口转发到本地，使本地能够访问到服务器上的端口。如在远程服务器上跑了一个 Jupyter Notebook (端口 8888) 或者 TensorBoard (端口 6006)，经过端口转发后，在本地就可以通过 http://localhost:本地端口 访问 Jupyter Notebook，通过 http://localhost:本地端口 访问 TensorBoard。

```
语法：ssh -L [本地端口]:[服务器IP]:[服务器端口] user@server
ssh -L 8080:localhost:8888 user@remote_server
```