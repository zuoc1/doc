```shell
# 安装
sudo apt-get install samba
# 修改配置文件，在smb.conf最后添加
sudo vi /etc/samba/smb.conf

[global]
  ntlm auth = yes
[exist_user]
  comment = share for work
  path = /home/exist_user/
  available = yes
  browseable = yes
  writable = yes
  public = yes
  valid users = exist_user
  create mode  = 0644
  directory mode = 0755
# 设置密码，exist_user为linux用户
smbpasswd -a exist_user
# 重启
service smbd restart
# 注意1：如果遇到无法访问的情况，请尝试注释掉sab.conf文件中nt pipe support = 0
# 注意2：如果遇到存在现有连接，请尝试使用net use * /del命令来切断已有连接
```