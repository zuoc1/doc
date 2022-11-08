# 1. 安装python

```shell
# Ubuntu安装python
sudo apt-get update
apt-cache showpkg python
sudo apt-get install python3
sudo apt-get install python3-pip
pip install --upgrade pip

# 以上使用默认版本，指定版本使用以下
sudo apt-get install python3.10.2

# CentOS安装python
sudo yum -y groupinstall "Development tools"
# 预装的依赖，上面可能已包含
sudo yum -y install zlib zlib-devel
sudo yum -y install bzip2 bzip2-devel
sudo yum -y install ncurses ncurses-devel
sudo yum -y install readline readline-devel
sudo yum -y install openssl openssl-devel
sudo yum -y install openssl-static
sudo yum -y install xz lzma xz-devel
sudo yum -y install sqlite sqlite-devel
sudo yum -y install gdbm gdbm-devel
sudo yum -y install tk tk-devel
sudo yum -y install libffi libffi-devel
# 下载或官网手动下载
wget https://www.python.org/ftp/python/3.9.9/Python-3.9.9.tar.xz
tar -xvf Python-3.8.6.tar.xz
# 安装
cd Python-3.8.6
./configure --prefix=/usr/soft/python3
make && make install
# 创建软链接
ln -s /usr/soft/python3/bin/python3.8 /usr/bin/python3
ln -s /usr/soft/python3/bin/pip3 /usr/bin/pip3
```

# 2. pip 使用代理

```bash
# 安装需求文件中的依赖
pip install -i https://mirrors.aliyun.com/pypi/simple/ --trusted-host=mirrors.aliyun.com -r requirements.txt
# 豆瓣源
pip install pyside2 -i https://pypi.douban.com/simple/
# 直接安装
pip install -i https://mirrors.aliyun.com/pypi/simple/ --trusted-host=mirrors.aliyun.com pygame
# 用户安装
pip install --user -i https://mirrors.aliyun.com/pypi/simple/ --trusted-host=mirrors.aliyun.com --upgrade pip

# pip升级过程中丢失，重新安装pip
python -m ensurepip
```

# 3. python画svg

```bash
# jpg转svg，图片太大画不了
https://www.visioncortex.org/vtracer/

# github下载
git clone https://github.com/qhduan/svgToTurtle

# 安装依赖
pip install -i https://mirrors.aliyun.com/pypi/simple/ --trusted-host=mirrors.aliyun.com -r requirements.txt
# 执行
python main.py ../c.svg --speed=5000

# 其它方法
pip install svg2turtle
svg2turtle ../c.svg
```

