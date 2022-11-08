# 1. wsl

## 1.1 设置

```powershell
# 更新wsl
wsl --update
# 此命令将启用所需的可选组件，下载最新的 Linux 内核，将 WSL 2 设置为默认值，并安装 Linux 发行版（默认安装 Ubuntu）。
wsl --install
# 设置Ubuntu-20.04的wsl版本为2
wsl --set-version Ubuntu-20.04 2
# 设置wsl默认版本为2
wsl --set-default-version 2
# 安装发行版本 -d表示设为默认版本
wsl --install -d Ubuntu-20.04
# 查看可以安装的发行版本
wsl --list --online 或 wsl -l -o
# 设置与 wsl 命令一起使用的默认 Linux 发行版
wsl -s Ubuntu-20.04 或 wsl --setdefault Ubuntu-20.04
# 查看虚拟机
wsl -l -v
# 启动虚拟机
wsl
# 关闭虚拟机
wsl -t Ubuntu-20.04 或 wsl --shutdown
```

## 1.2 LxRunOffline

版本报错使用：LxRunOffline-v3.5.0-11-gfdab71a-msvc.zip

```shell
# 仓
https://github.com/DDoSolitary/LxRunOffline/releases
# 教程
https://p3terx.com/archives/manage-wsl-with-lxrunoffline.html


# 安装虚拟机
# 下载 WSL 官方离线包，改后缀名为.zip，解压后可得到名为 install.tar.gz 的文件
# 能处理.Appx但不能处理.AppxBundle
# 加入-s参数可在桌面创建快捷方式
lxrunoffline i -n <WSL名称> -d <安装路径> -f <安装包路径>.tar.gz -s

# 赋权限并迁移
icacls E:\VMware\Ubuntu-20.04 /grant "zuoch:(OI)(CI)(F)"
LxRunOffline move -n Ubuntu-20.04 -d E:\VMware\Ubuntu-20.04

# 设置默认登录名，1000为用户ID
lxrunoffline su -n <WSL名称> -v 1000

# 对 WSL 的目录进行移动
lxrunoffline m -n <WSL名称> -d <路径>

# 查看已安装 wsl
lxrunoffline l
# 查看默认 WSL
lxrunoffline gd

# 查看路径
lxrunoffline di -n <WSL名称>

# 备份 WSL，不支持WSL2
# 类似但不等同于wsl --export <WSL名称> <压缩包路径>.tar。LxRunOffline 备份完会生成一个.xml后缀的同名配置文件，比如WSL.tar.gz.xml。
lxrunoffline e -n <WSL名称> -f <压缩包路径>.tar.gz

# 恢复 WSL
# 类似但不等同于wsl --import <WSL名称> <安装路径> <压缩包路径>.tar。LxRunOf­fline 会读取备份时生成的配置文件并写入配置，前提是同目录且同名。否则你需要加入-c参数指定配置文件。
lxrunoffline i -n <WSL名称> -d <安装路径> -f <压缩包路径>.tar.gz

# 创建快捷方式
lxrunoffline s -n <WSL名称> -f <快捷方式路径>.lnk

# 使用命令运行指定 WSL，在有多个 WSL 的情况下，可以指定运行某个发行版，等同于wsl -d <WSL名称>。
lxrunoffline r -n <WSL名称>

# 查看 WSL 安装目录。
lxrunoffline di -n <WSL名称>

# 导出指定的 WSL 配置文件到目标路径。
lxrunoffline ec -n <WSL名称> -f <配置文件路径>.xml

# 查看配置信息
lxrunoffline sm -n <WSL名称>查看

# 取消注册（这个操作不会删除目录）
lxrunoffline ur -n <WSL名称>

# 使用新名称注册
lxrunoffline rg -n <WSL名称> -d <WSL路径> -c <配置文件路径>.xml
```

## 1.3 xshell ssh 连接 wsl

此后就可以使用 xshell 连接 wsl 了，连接ip：127.0.0.1，port：36022

```shell
# 卸载自带的 ssh
sudo apt-get remove openssh-server
# 再次安装
sudo apt-get install openssh-server
# 删配置文件，让ssh服务自己想办法链接
sudo rm /etc/ssh/ssh_config
sudo service ssh --full-restart
# 修改配置
# 默认的是22，但是windows有自己的ssh服务用的也是22端口，修改一下
sudo vim /etc/ssh/sshd_config
Port 36022
PasswordAuthentication yes
# 重启ssh服务，设置成服务开机自启动比较好：sudo systemctl enable ssh
sudo service ssh --full-restart
# 可直接进入root用户
sudo -i
#生成ssh秘钥和公钥：一路回车，也可使用xshell工具生成后粘贴到.ssh下，不用放root用户下也行
ssh-keygen
sudo cp /home/zuoc/.ssh/id_rsa.pub  /home/zuoc/.ssh/authorized_keys
```

## 1.4 vscode

wecode安装Remote Development。除了 Remote - SSH 和 Remote - Containers 扩展之外，此扩展包还包含 Remote - WSL 扩展，使你能够打开容器中、远程计算机上或 WSL 中的任何文件夹。

