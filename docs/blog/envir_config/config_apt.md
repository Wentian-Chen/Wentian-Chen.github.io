# 配置 国内镜像APT 源

## For Ubuntu (amd64)
参考文档：

- [ubuntu amd64 清华源配置指南](https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/)
- [ubuntu amd64 中科大源配置指南](https://mirrors.ustc.edu.cn/help/ubuntu.html)

以下指令适用Ubuntu 24.04以下版本，不包括24.04
```
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
以下指令适用Ubuntu 24.04及以上版本
```
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
## For Ubuntu (arm64)
参考文档：

- [ubuntu arm64 清华源配置指南](https://mirror.tuna.tsinghua.edu.cn/help/ubuntu-ports/)
- [ubuntu arm64 中科大源配置指南](https://mirrors.ustc.edu.cn/help/ubuntu-ports.html)
  
以下指令适用Ubuntu 24.04以下版本，不包括24.04
```
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
以下指令适用Ubuntu 24.04及以上版本
```
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