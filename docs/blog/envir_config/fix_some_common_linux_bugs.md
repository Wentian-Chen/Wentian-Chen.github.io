# 常见Linux小bugs的解决方法
## ubuntu掉AX101蓝牙
参考：[Ubuntu 20.04 掉线AX101蓝牙](https://blog.csdn.net/qq_40397392/article/details/140637799)
问题情况：
```
[    2.587972] kernel: Bluetooth: hci0: Failed to load Intel firmware file intel/ibt-1040-1050.sfi (-2)
[    2.589252] kernel: Bluetooth: hci0: Failed to read MSFT supported features (-56)
```
## ubuntu没有以太网连接和图标
```
# 使用下面命令查看ubuntu的网络状态
sudo cat /var/lib/NetworkManager/NetworkManager.state 

# 查询结果如下：
[main] NetworkingEnabled=false WirelessEnabled=true WWANEnabled=true 

# 由于NetworkingEnabled是false状态，所以ubuntu上不了网
# 解决办法就是将其修改true 先关闭服务
service network-manager stop 

# 打开文件
sudo vim /var/lib/NetworkManager/NetworkManager.state
# 修改为NetworkingEnabled=true 

# 再次启动服务
service network-manager start
```
## 修改开机引导项
```
# 开始
sudo vim /etc/default/grub
# 修改 GRUB_DEFAULT=0
# 修改成引导界面，对应的选项。
# 第一行的序号是0，以此类推
# 修改完后保存退出
sudo update-grub
# 结束
```
## 双系统时间不同步
参考：[ubuntu与windows双系统时间同步问题——简明指南](https://zhuanlan.zhihu.com/p/492885761)
```
#在Ubuntu的终端执行 即可
timedatectl set-local-rtc 1
```