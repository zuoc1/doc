# 一. Linux

官网：[www.gnu.org](http://www.gnu.org/)
[www.linux.org](http://www.linux.org/)
[www.kernel.org](http://www.kernel.org/)

## 1. 新建用户

```shell
# 添加账号并设置根目录，使用adduser不用useradd
useradd -m -s /bin/bash zuoc

# 设置密码
sudo passwd zuoc

# 给用户添加sudo权限，
# 方法一：默认是使用nano编辑器操作步骤如下：
# 1.先执行“Ctrl+O”快捷键进行保存，在tmp后执行回车。
# 2.通过直接“Ctrl+X”快捷键退出即可。
sudo visudo
zuoc ALL=(ALL) ALL
%zuoc ALL=(ALL) NOPASSWD: ALL
# 方法二：把用户添加到 sudo 用户组
usermod -aG sudo zuoc

# 删除用户
# -f：强制删除用户，即使用户当前已登录；
# -r：删除用户的同时，删除与用户相关的所有文件。
userdel -rf zuoc

# 查看用户 UID ，一般是 1000
id -u zuoc

# 添加别名
alias ll='ls -aFhil' # 显示隐藏文件/文件类型/易读的方式显示文件或目录大小/显示inode节点信息/详细信息
# 修改PS1提示符
PS1='[\[\e[37;40m\]\u/\[\e[32;40m\]\d/\[\e[34;40m\]\t \[\e[36;40m\]\W]\[\e[0m\]\$ '
```

## 2. 软件安装/更新

```shell
# 安装软件
apt-get install software_name
# 卸载软件及其配置
apt-get --purge remove software_name
# 卸载软件及其依赖的安装包
apt-get autoremove software_name
# 罗列已安装软件
dpkg --list

# update是更新可安装软件列表到当前运行ubuntu系统，upgrade是更新当前ubuntu系统已经安装了的软件到最新
sudo apt-get update
sudo apt-get upgrade
```

设置阿里云镜像：

```shell
sudo mv /etc/apt/sources.list /etc/apt/sourses.list.backup
sudo vi /etc/apt/sources.list

deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
```

## 4. 常用命令

### 4.1 创建/删除目录

```shell
# m自定义权限，p递归创建
# rmdir只能删除空目录，较少使用
mkdir [-mp] 目录名
rmdir [-p] 目录名
```
### 4.2 删除文件或目录

rm 命令是一个具有破坏性的命令，因为 rm 命令会永久性地删除文件或目录，这就意味着，如果没有对文件或目录进行备份，一旦使用 rm 命令将其删除，将无法恢复，因此，尤其在使用 rm 命令删除目录时，要慎之又慎。

```
rm[选项] 文件或目录

选项：
-f：强制删除（force），和 -i 选项相反，使用 -f，系统将不再询问，而是直接删除目标文件或目录。
-i：和 -f 正好相反，在删除文件或目录之前，系统会给出提示信息，使用 -i 可以有效防止不小心删除有用的文件或目录。
-r：递归删除，主要用于删除目录，可删除指定目录及包含的所有内容，包括所有的子目录和文件。
```

### 4.3 创建文件及修改文件时间戳

Linux 系统中，每个文件主要拥有 3 个时间参数（通过 stat 命令进行查看），分别是文件的访问时间、数据修改时间以及状态修改时间：

- 访问时间（Access Time，简称 atime）：只要文件的内容被读取，访问时间就会更新。例如，使用 cat 命令可以查看文件的内容，此时文件的访问时间就会发生改变。

- 数据修改时间（Modify Time，简称 mtime）：当文件的内容数据发生改变，此文件的数据修改时间就会跟着相应改变。

- 状态修改时间（Change Time，简称 ctime）：当文件的状态发生变化，就会相应改变这个时间。比如说，如果文件的权限或者属性发生改变，此时间就会相应改变。

touch 命令可以只修改文件的访问时间，也可以只修改文件的数据修改时间，但是不能只修改文件的状态修改时间。因为，不论是修改访问时间，还是修改文件的数据时间，对文件来讲，状态都会发生改变，即状态修改时间会随之改变（更新为操作当前文件的真正时间）。

```
touch [选项] 文件名

-a：只修改文件的访问时间；
-c：仅修改文件的时间参数（3 个时间参数都改变），如果文件不存在，则不建立新文件。
-d：后面可以跟欲修订的日期，而不用当前的日期，即把文件的 atime 和 mtime 时间改为指定的时间。
-m：只修改文件的数据修改时间。
-t：命令后面可以跟欲修订的时间，而不用目前的时间，时间书写格式为 YYMMDDhhmm。

# ctime不会变为设定时间，但更新为当前服务器的时间
touch -d "2017-05-04 15:44" bols
```

### 4.4 建立链接（硬链接和软链接）文件

ln 命令用于给文件创建链接，根据 Linux 系统存储文件的特点，链接的方式分为以下 2 种：

- 软链接：类似于 Windows 系统中给文件创建快捷方式，即产生一个特殊的文件，该文件用来指向另一个文件，此链接方式同样适用于目录。
- 硬链接：我们知道，文件的基本信息都存储在 inode 中，而硬链接指的就是给一个文件的 inode 分配多个文件名，通过任何一个文件名，都可以找到此文件的 inode，从而读取该文件的数据信息。

```
ln [选项] 源文件 目标文件
选项：
-s：建立软链接文件。如果不加 "-s" 选项，则建立硬链接文件；
-f：强制。如果目标文件已经存在，则删除目标文件后再建立链接文件；

# 建立硬链接文件，目标文件没有写文件名，会和原名一致，也就是/tmp/cangls 是硬链接文件
ln /root/cangls /tmp
# 这里需要注意的是，软链接文件的源文件必须写成绝对路径，而不能写成相对路径（硬链接没有这样的要求）；否则软链接文件会报错。这是初学者非常容易犯的错误。
```

总结：

- 软连接有自己的inode，删除不会影响原文件。硬链接指向同一inode，删除最后一个硬链接文件才会删除，类似于共享指针。

- 硬链接不能跨文件系统（分区）建立，因为在不同的文件系统中，inode 号是重新计算的。
- 硬链接不能链接目录，因为如果给目录建立硬链接，那么不仅目录本身需要重新建立，目录下所有的子文件，包括子目录中的所有子文件都需要建立硬链接，这对当前的 Linux 来讲过于复杂。

### 4.5 复制文件和目录

```shell
cp [选项] 源文件 目标文件

选项：
-a：相当于 -d、-p、-r 选项的集合，这几个选项我们一一介绍；
-d：如果源文件为软链接（对硬链接无效），则复制出的目标文件也为软链接；
-i：询问，如果目标文件已经存在，则会询问是否覆盖；
-l：把目标文件建立为源文件的硬链接文件，而不是复制源文件；
-s：把目标文件建立为源文件的软链接文件，而不是复制源文件；
-p：复制后目标文件保留源文件的属性（包括所有者、所属组、权限和时间）；
-r：递归复制，用于复制目录；
-u：若目标文件比源文件有差异，则使用该选项可以更新目标文件，此选项可用于对文件的升级和备用。
```

如果在复制软链接文件时不使用 "-d" 选项，则 cp 命令复制的是源文件，而不是软链接文件；只有加入了 "-d" 选项，才会复制软链接文件。请大家注意，"-d" 选项对硬链接是无效的。

而当我们执行备份、曰志备份的时候，这些文件的时间可能是一个重要的参数，这就需执行 "-p" 选项了。这个选项会保留源文件的属性，包括所有者、所属组和时间。

"-a" 选项相当于 "-d、-p、-r" 选项，这几个选项我们已经分别讲过了。所以，当我们使用 "-a" 选项时，目标文件和源文件的所有属性都一致，包括源文件的所有者，所属组、时间和软链接性。使用 "-a" 选项来取代 "-d、-p、-r" 选项更加方便。

"d" 选项要求源文件必须是软链接，目标文件才会复制为软链接；而 "-l" 和 "-s" 选项的源文件只需是普通文件，目标文件就可以直接复制为硬链接和软链接。

### 4.6 移动文件或改名

需要注意的是，同 rm 命令类似，mv 命令也是一个具有破坏性的命令，如果使用不当，很可能给系统带来灾难性的后果。也可以移动目录，和 rm、cp 不同的是，mv 移动目录不需要加入 "-r" 选项

```shell
mv [选项] 源文件 目标文件

选项：
-f：强制覆盖，如果目标文件已经存在，则不询问，直接强制覆盖；
-i：交互移动，如果目标文件已经存在，则询问用户是否覆盖（默认选项）；
-n：如果目标文件已经存在，则不会覆盖移动，而且不询问用户；
-v：显示文件或目录的移动过程；
-u：若目标文件已经存在，但两者相比，源文件更新，则会对目标文件进行升级；
```

### 4.7 通配符

tab补全文件名 ，按下 2 次 Tab 键可补全命令。

| 符号 | 作用                                                         |
| ---- | ------------------------------------------------------------ |
| *    | 匹配任意数量的字符。                                         |
| ?    | 匹配任意一个字符。                                           |
| []   | 匹配括号内的任意一个字符，甚至 [] 中还可以包含用 -（短横线）连接的字符或数字，表示一定范围内的字符或数字。 |

### 4.8 判断是内部命令还是外部命

内部命令由 Shell 自带，会随着系统启动，可以直接从内存中读取；而外部命令仅是在系统中有对应的可执行文件，执行时需要读取该文件。

```shell
$ type pwd
pwd is a shell builtin  <-- pwd是内部命令
```

### 4.9 环境变量

我们可以使用 env 命令来查看到 Linux 系统中所有的环境变量。

| 环境变量名称 | 作用                                   |
| ------------ | -------------------------------------- |
| HOME         | 用户的主目录（也称家目录）             |
| SHELL        | 用户使用的 Shell 解释器名称            |
| PATH         | 定义命令行解释器搜索用户执行命令的路径 |
| EDITOR       | 用户默认的文本解释器                   |
| RANDOM       | 生成一个随机数字                       |
| LANG         | 系统语言、语系名称                     |
| HISTSIZE     | 输出的历史命令记录条数                 |
| HISTFILESIZE | 保存的历史命令记录条数                 |
| PS1          | Bash解释器的提示符                     |
| MAIL         | 邮件保存路径                           |

```shell
# 根据PATH查找某个命令所在的绝对路径
which rm
```

### 4.10 cat命令

注意，cat 命令用于查看文件内容时，不论文件内容有多少，都会一次性显示。如果文件非常大，那么文件开头的内容就看不到了。不过 Linux 可以使用`PgUp+上箭头`组合键向上翻页，但是这种翻页是有极限的，如果文件足够长，那么还是无法看全文件的内容。因此，cat 命令适合查看不太大的文件。

```shell
cat [选项] 文件名
或者
cat 文件1 文件2 > 文件3
```

这两种格式中，前者用于显示文件的内容，常用选项及各自的含义如下所示；而后者用于连接合并文件。

| 选项 | 含义                                                     |
| ---- | -------------------------------------------------------- |
| -A   | 相当于 -vET 选项的整合，用于列出所有隐藏符号；           |
| -E   | 列出每行结尾的回车符 $；                                 |
| -n   | 对输出的所有行进行编号；                                 |
| -b   | 同 -n 不同，此选项表示只对非空行进行编号。               |
| -T   | 把 Tab 键 ^I 显示出来；                                  |
| -V   | 列出特殊字符；                                           |
| -s   | 当遇到有连续 2 行以上的空白行时，就替换为 1 行的空白行。 |

将文件 file1.txt 和 file2.txt 的内容合并后输出到文件 file3.txt 中

```shell
$ ls
file1.txt    file2.txt
$ cat file1.txt
http://c.biancheng.net(file1.txt)
$ cat file2.txt
is great(file2.txt)
$ cat file1.txt file2.txt > file3.txt
$ more file3.txt
# more 命令可查看文件中的内容
http://c.biancheng.net(file1.txt)
is great(file2.txt)
$ ls
file1.txt    file2.txt    file3.txt
```

### 4.11 more命令：分屏显示文件内容

```shell
more [选项] 文件名
```

| 选项 | 含义                                                     |
| ---- | -------------------------------------------------------- |
| -f   | 计算行数时，以实际的行数，而不是自动换行过后的行数。     |
| -p   | 不以卷动的方式显示每一页，而是先清除屏幕后再显示内容。   |
| -c   | 跟 -p 选项相似，不同的是先显示内容再清除其他旧资料。     |
| -s   | 当遇到有连续两行以上的空白行时，就替换为一行的空白行。   |
| -u   | 不显示下引号（根据环境变量 TERM 指定的终端而有所不同）。 |
| +n   | 从第 n 行开始显示文件内容，n 代表数字。                  |
| -n   | 一次显示的行数，n 代表数字。                             |

| 交互指令            | 功能                         |
| ------------------- | ---------------------------- |
| h 或 ？             | 显示 more 命令交互命令帮助。 |
| q 或 Q              | 退出 more。                  |
| v                   | 在当前行启动一个编辑器。     |
| :f                  | 显示当前文件的文件名和行号。 |
| !<命令> 或 :!<命令> | 在子Shell中执行指定命令。    |
| 回车键              | 向下移动一行。               |
| 空格键              | 向下移动一页。               |
| Ctrl+l              | 刷新屏幕。                   |
| =                   | 显示当前行的行号。           |
| '                   | 转到上一次搜索开始的地方。   |
| Ctrf+f              | 向下滚动一页。               |
| .                   | 重复上次输入的命令。         |
| / 字符串            | 搜索指定的字符串。           |
| d                   | 向下移动半页。               |
| b                   | 向上移动一页。               |

### 4.12 head命令：显示文件开头的内容

注意，如不设置显示的具体行数，则默认显示 10 行的文本数据。

```shell
head [选项] 文件名

head -n 20 anaconda-ks.cfg
或则
head -20 anaconda-ks.cfg
```

| 选项 | 含义                                                         |
| ---- | ------------------------------------------------------------ |
| -n K | 这里的 K 表示行数，该选项用来显示文件前 K 行的内容；如果使用 "-K" 作为参数，则表示除了文件最后 K 行外，显示剩余的全部内容。 |
| -c K | 这里的 K 表示字节数，该选项用来显示文件前 K 个字节的内容；如果使用 "-K"，则表示除了文件最后 K 字节的内容，显示剩余全部内容。 |
| -v   | 显示文件名；                                                 |

### 4.13 less命令：查看文件内容

less 命令的作用和 more 十分类似，都用来浏览文本文件中的内容，不同之处在于，使用 more 命令浏览文件内容时，只能不断向后翻看，而使用 less 命令浏览，既可以向后翻看，也可以向前翻看。

不仅如此，为了方面用户浏览文本内容，less 命令还提供了以下几个功能：

- 使用光标键可以在文本文件中前后（左后）滚屏；
- 用行号或百分比作为书签浏览文件；
- 提供更加友好的检索、高亮显示等操作；
- 兼容常用的字处理程序（如 Vim、Emacs）的键盘操作；
- 阅读到文件结束时，less 命令不会退出；
- 屏幕底部的信息提示更容易控制使用，而且提供了更多的信息。

```shell
less [选项] 文件名
```

| 选项            | 选项含义                                               |
| --------------- | ------------------------------------------------------ |
| -N              | 显示每行的行号。                                       |
| -S              | 行过长时将超出部分舍弃。                               |
| -e              | 当文件显示结束后，自动离开。                           |
| -g              | 只标志最后搜索到的关键同。                             |
| -Q              | 不使用警告音。                                         |
| -i              | 忽略搜索时的大小写。                                   |
| -m              | 显示类似 more 命令的百分比。                           |
| -f              | 强迫打开特殊文件，比如外围设备代号、目录和二进制文件。 |
| -s              | 显示连续空行为一行。                                   |
| -b <缓冲区大小> | 设置缓冲区的大小。                                     |
| -o <文件名>     | 将 less 输出的内容保存到指定文件中。                   |
| -x <数字>       | 将【Tab】键显示为规定的数字空格。                      |

less 交互指令及功能

| 交互指令   | 功能                                   |
| ---------- | -------------------------------------- |
| /字符串    | 向下搜索“字符串”的功能。               |
| ?字符串    | 向上搜索“字符串”的功能。               |
| n          | 重复*前一个搜索（与 / 成 ? 有关）。    |
| N          | 反向重复前一个搜索（与 / 或 ? 有关）。 |
| b          | 向上移动一页。                         |
| d          | 向下移动半页。                         |
| h 或 H     | 显示帮助界面。                         |
| q 或 Q     | 退出 less 命令。                       |
| y          | 向上移动一行。                         |
| 空格键     | 向下移动一页。                         |
| 回车键     | 向下移动一行。                         |
| 【PgDn】键 | 向下移动一页。                         |
| 【PgUp】键 | 向上移动一页。                         |
| Ctrl+f     | 向下移动一页。                         |
| Ctrl+b     | 向上移动一页。                         |
| Ctrl+d     | 向下移动一页。                         |
| Ctrl+u     | 向上移动半页。                         |
| j          | 向下移动一行。                         |
| k          | 向上移动一行。                         |
| G          | 移动至最后一行。                       |
| g          | 移动到第一行。                         |
| ZZ         | 退出 less 命令。                       |
| v          | 使用配置的编辑器编辑当前文件。         |
| [          | 移动到本文档的上一个节点。             |
| ]          | 移动到本文档的下一个节点。             |
| p          | 移动到同级的上一个节点。               |
| u          | 向上移动半页。                         |

### 4.14 tail命令：显示文件结尾的内容

```shell
tail [选项] 文件名

tail -n 3 /etc/passwd
或则
tail -3 /etc/passwd
# 这条命令会显示文件的最后 10 行内容，而且光标不会退出命令，每隔一秒会检查一下文件是否增加新的内容，如果增加就追加到原来的输出结果后面并显示。
tail -f anaconda-ks.cfg
```

| 选项 | 含义                                                         |
| ---- | ------------------------------------------------------------ |
| -n K | 这里的 K 指的是行数，该选项表示输出最后 K 行，在此基础上，如果使用 -n +K，则表示从文件的第 K 行开始输出。 |
| -c K | 这里的 K 指的是字节数，该选项表示输出文件最后 K 个字节的内容，在此基础上，使用 -c +K 则表示从文件第 K 个字节开始输出。 |
| -f   | 输出文件变化后新增加的数据。如果想终止输出，按【Ctrl+c】键中断 tail 命令即可。 |

### 4.15 输入输出重定向

文件描述符(0，1，2)分别与标准输入(stdin)，标准输出(stdout)和标准错误(stderr)对应。因此，函数 scanf() 使用 stdin，而函数 printf() 使用 stdout。

输入重定向中用到的符号及作用：

| 命令符号格式           | 作用                                                         |
| ---------------------- | ------------------------------------------------------------ |
| 命令 < 文件            | 将指定文件作为命令的输入设备                                 |
| 命令 << 分界符         | 表示从标准输入设备（键盘）中读入，直到遇到分界符才停止（读入的数据不包括分界符），这里的分界符其实就是自定义的字符串 |
| 命令 < 文件 1 > 文件 2 | 将文件 1 作为命令的输入设备，该命令的执行结果输出到文件 2 中。 |

输出重定向用到的符号及作用：

| 命令符号格式                         | 作用                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| 命令 > 文件                          | 将命令执行的标准输出结果重定向输出到指定的文件中，如果该文件已包含数据，会清空原有数据，再写入新数据。 |
| 命令 2> 文件                         | 将命令执行的错误输出结果重定向到指定的文件中，如果该文件中已包含数据，会清空原有数据，再写入新数据。 |
| 命令 >> 文件                         | 将命令执行的标准输出结果重定向输出到指定的文件中，如果该文件已包含数据，新数据将写入到原有内容的后面。 |
| 命令 2>> 文件                        | 将命令执行的错误输出结果重定向到指定的文件中，如果该文件中已包含数据，新数据将写入到原有内容的后面。 |
| 命令 >> 文件 2>&1 或者 命令 &>> 文件 | 将标准输出或者错误输出写入到指定文件，如果该文件中已包含数据，新数据将写入到原有内容的后面。注意，第一种格式中，最后的 2>&1 是一体的，可以认为是固定写法。 |

```shell
# 当指定了 0 作为分界符之后，只要不输入 0，就可以一直输入数据。
$ cat << 0
>c.biancheng.net
>Linux
>0
c.biancheng.net
Linux

# 将 /etc/passwd 文件中内容复制到 a.txt 中
$ cat < /etc/passwd > a.txt

$ cat Linux.txt > demo.txt
$ cat demo.txt
Linux
$ cat Linux.txt > demo.txt
$ cat demo.txt
Linux     <--这里的 Linux 是清空原有的 Linux 之后，写入的新的 Linux
$ cat Linux.txt >> demo.txt
$ cat demo.txt
Linux
Linux     <--以追加的方式，新数据写入到原有数据之后
$ cat b.txt > demo.txt
cat: b.txt: No such file or directory  <-- 错误输出信息依然输出到了显示器中
$ cat b.txt 2> demo.txt
$ cat demo.txt
cat: b.txt: No such file or directory  <--清空文件，再将错误输出信息写入到该文件中
$ cat b.txt 2>> demo.txt
$ cat demo.txt
cat: b.txt: No such file or directory
cat: b.txt: No such file or directory  <--追加写入错误输出信息
```

### 4.16 grep命令详解：查找文件内容

grep命令能够在一个或多个文件中，搜索某一特定的字符模式（也就是正则表达式），此模式可以是单一的字符、字符串、单词或句子。需要注意的是，在基本正则表达式中，如通配符 *、+、{、|、( 和 )等，已经失去了它们原本的含义，而若要恢复它们原本的含义，则要在之前添加反斜杠 \，如 \*、\+、\{、\|、\( 和 \)。

注意，如果是搜索多个文件，grep 命令的搜索结果只显示文件中发现匹配模式的文件名；而如果搜索单个文件，grep 命令的结果将显示每一个包含匹配模式的行。

| 通配符 | 功能                                                |
| ------ | --------------------------------------------------- |
| c*     | 将匹配 0 个（即空白）或多个字符 c（c 为任一字符）。 |
| .      | 将匹配任何一个字符，且只能是一个字符。              |
| [xyz]  | 匹配方括号中的任意一个字符。                        |
| [^xyz] | 匹配除方括号中字符外的所有字符。                    |
| ^      | 锁定行的开头。                                      |
| $      | 锁定行的结尾。                                      |

```shell
grep [选项] 模式 文件名
```

| 选项 | 含义                                                       |
| ---- | ---------------------------------------------------------- |
| -c   | 仅列出文件中包含模式的行数。                               |
| -i   | 忽略模式中的字母大小写。                                   |
| -l   | 列出带有匹配行的文件名。                                   |
| -n   | 在每一行的最前面列出行号。                                 |
| -v   | 列出没有匹配模式的行。                                     |
| -w   | 把表达式当做一个完整的单字符来搜寻，忽略那些部分匹配的行。 |

### 4.17 sed命令完全攻略

```shell
sed [选项] [脚本命令] 文件名

$ grep demo /etc/passwd
demo:x:502:502::/home/Samantha:/bin/bash
# 找到demo用户，并将bash替换为csh，此处/demo/相当于行号
$ sed '/demo/s/bash/csh/' /etc/passwd
root:x:0:0:root:/root:/bin/bash
...
demo:x:502:502::/home/demo:/bin/csh
...


$ cat test.txt
<html>
<title>First Wed</title>
<body>
h1Helloh1
h2Helloh2
h3Helloh3
</body>
</html>
# 使用正则表示式给所有第一个的h1、h2、h3添加<>，给第二个h1、h2、h3添加</>
# 首先找到h[0-9]，接着执行{}中的两个命令，去掉转义：s//<&>/1  s//</&>/2，<&>表示给匹配项添加<>。
$ cat sed.sh
/h[0-9]/{
    s//\<&\>/1
    s//\<\/&\>/2
}
$ sed -f sed.sh test.txt
<h1>Hello</h1>
<h2>Hello</h2>
<h3>Hello</h3>
```

| 选项            | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| -e 脚本命令     | 该选项会将其后跟的脚本命令添加到已有的命令中。               |
| -f 脚本命令文件 | 该选项会将其后文件中的脚本命令添加到已有的命令中。           |
| -n              | 默认情况下，sed 会在所有的脚本指定执行完毕后，会自动输出处理后的内容，而该选项会屏蔽启动输出，需使用 print 命令来完成输出。 |
| -i              | 此选项会直接修改源文件，要慎用。                             |

成功使用 sed 命令的关键在于掌握各式各样的脚本命令及格式，它能帮你定制编辑文件的规则。

#### 4.17.1 sed s 替换指定模式

其中，address 表示指定要操作的具体行，pattern 指的是需要替换的内容，replacement 指的是要替换的新内容。

```shell
[address]s/pattern/replacement/flags

# 使用数字 2 作为标记的结果就是，sed 编辑器只替换每行中第 2 次出现的匹配模式。
$ sed 's/test/trial/2' data4.txt
This is a test of the trial script.
This is the second test of the trial script.

# 用新文件替换所有匹配的字符串，可以使用 g 标记。
$ sed 's/test/trial/g' data4.txt
This is a trial of the trial script.
This is the second trial of the trial script.

# 我们知道，-n 选项会禁止 sed 输出，但 p 标记会输出修改过的行，将二者匹配使用的效果就是只输出被替换命令修改过的行，例如：
$ cat data5.txt
This is a test line.
This is a different line.
$ sed -n 's/test/trial/p' data5.txt
This is a trial line.

# w 标记会将匹配后的结果保存到指定文件中，比如：
$ sed 's/test/trial/w test.txt' data5.txt
This is a trial line.
This is a different line.
$ cat test.txt
This is a trial line.

# 在使用 s 脚本命令时，替换类似文件路径的字符串会比较麻烦，需要将路径中的正斜线进行转义，例如：
$ sed 's/\/bin\/bash/\/bin\/csh/' /etc/passwd
```

| flags 标记 | 功能                                                         |
| ---------- | ------------------------------------------------------------ |
| n          | 1~512 之间的数字，表示指定要替换的字符串出现第几次时才进行替换，例如，一行中有 3 个 A，但用户只想替换第二个 A，这是就用到这个标记； |
| g          | 对数据中所有匹配到的内容进行替换，如果没有 g，则只会在第一次匹配成功时做替换操作。例如，一行数据中有 3 个 A，则只会替换第一个 A； |
| p          | 会打印与替换命令中指定的模式匹配的行。此标记通常与 -n 选项一起使用。 |
| w file     | 将缓冲区中的内容写到指定的 file 文件中；                     |
| &          | 用正则表达式匹配的内容进行替换；                             |
| \n         | 匹配第 n 个子串，该子串之前在 pattern 中用 \(\) 指定。       |
| \          | 转义（转义替换部分包含：&、\ 等）。                          |

#### 4.17.2 sed d 删除行

如果需要删除文本中的特定行，可以用 d 脚本命令，它会删除指定行中的所有内容。但使用该命令时要特别小心，如果你忘记指定具体行的话，文件中的所有内容都会被删除。

在此强调，在默认情况下 sed 并不会修改原始文件，这里被删除的行只是从 sed 的输出中消失了，原始文件没做任何改变。

```shell
[address]d

$ cat data.txt
This is line number 1.
This is line number 2.
This is line number 3.
This is line number 4.
$ sed 'd' data.txt
#什么也不输出，证明成了空文件
$ sed '3d' data.txt
This is line number 1.
This is line number 2.
This is line number 4.
$ sed '2,3d' data.txt
This is line number 1.
This is line number 4.

# 也可以使用两个文本模式来删除某个区间内的行，但这么做时要小心，你指定的第一个模式会“打开”行删除功能，第二个模式会“关闭”行删除功能，因此，sed 会删除两个指定行之间的所有行（包括指定的行），例如：
$ sed '/1/,/3/d' data.txt
#删除第 1~3 行的文本数据
This is line number 4.

# 或者通过特殊的文件结尾字符，比如删除 data6.txt 文件内容中第 3 行开始的所有的内容：
$ sed '3,$d' data6.txt
This is line number 1.
This is line number 2.
```

#### 4.17.3 sed a 和 i 插入行

a 命令表示在指定行的后面附加一行，i 命令表示在指定行的前面插入一行。

```shell
[address]a（或 i）\新文本内容

# 如果你想将一个多行数据添加到数据流中，只需对要插入或附加的文本中的每一行末尾（除最后一行）添加反斜线即可，例如：
$ sed '1i\
> This is one line of new text.\
> This is another line of new text.' data6.txt
This is one line of new text.
This is another line of new text.
This is line number 1.
This is line number 2.
This is line number 3.
This is line number 4.
```

#### 4.17.4 sed c 替换行

```shell
[address]c\用于替换的新文本

$ sed '3c\
> This is a changed line of text.' data6.txt
This is line number 1.
This is line number 2.
This is a changed line of text.
This is line number 4.
# 在这个例子中，sed 编辑器会修改第三行中的文本，其实，下面的写法也可以实现此目的：
$ sed '/number 3/c\
> This is a changed line of text.' data6.txt
This is line number 1.
This is line number 2.
This is a changed line of text.
This is line number 4.
```

#### 4.17.5 sed y 替换字符

y 转换命令是唯一可以处理单个字符的 sed 脚本命令，转换命令会对 inchars 和 outchars 值进行一对一的映射，即 inchars 中的第一个字符会被转换为 outchars 中的第一个字符，第二个字符会被转换成 outchars 中的第二个字符...这个映射过程会一直持续到处理完指定字符。如果 inchars 和 outchars 的长度不同，则 sed 会产生一条错误消息。

```shell
[address]y/inchars/outchars/

$ sed 'y/123/789/' data8.txt
This is line number 7.
This is line number 8.
This is line number 9.
This is line number 4.
# 转换命令是一个全局命令，也就是说，它会文本行中找到的所有指定字符自动进行转换，而不会考虑它们出现的位置，再打个比方：
$ echo "This 1 is a test of 1 try." | sed 'y/123/456/'
This 4 is a test of 4 try.
```

#### 4.17.6 sed p 打印

```shell
[address]p

$ cat data6.txt
This is line number 1.
This is line number 2.
This is line number 3.
This is line number 4.
# 用 -n 选项和 p 命令配合使用，我们可以禁止输出其他行，只打印包含匹配文本模式的行。
$ sed -n '/number 3/p' data6.txt
This is line number 3.
# sed 命令会查找包含数字 3 的行，然后执行两条命令。首先，脚本用 p 命令来打印出原始行；然后它用 s 命令替换文本，并用 p 标记打印出替换结果。输出同时显示了原来的行文本和新的行文本。
$ sed -n '/3/{
> p
> s/line/test/p
> }' data6.txt
This is line number 3.
This is test number 3.
```

#### 4.17.7 sed w 写入

通过使用 w 脚本命令，sed 可以实现将包含文本模式的数据行写入目标文件。

```shell
[address]w filename

$ sed '1,2w test.txt' data6.txt
This is line number 1.
This is line number 2.
This is line number 3.
This is line number 4.
$ cat test.txt
This is line number 1.
This is line number 2.

$ cat data11.txt
Blum, R       Browncoat
McGuiness, A  Alliance
Bresnahan, C  Browncoat
Harken, C     Alliance
$ sed -n '/Browncoat/w Browncoats.txt' data11.txt
$ cat Browncoats.txt
Blum, R       Browncoat
Bresnahan, C  Browncoat
```

#### 4.17.8 sed r 读取

r 命令用于将一个独立文件的数据插入到当前数据流的指定位置，该命令的基本格式为：

```shell
[address]r filename

$ cat data12.txt
This is an added line.
This is the second added line.
$ sed '3r data12.txt' data6.txt
This is line number 1.
This is line number 2.
This is line number 3.
This is an added line.
This is the second added line.
This is line number 4.
$ sed '$r data12.txt' data6.txt
This is line number 1.
This is line number 2.
This is line number 3.
This is line number 4.
This is an added line.
This is the second added line.
```

#### 4.17.9 sed q 退出脚本命令

q 命令的作用是使 sed 命令在第一次匹配任务结束后，退出 sed 程序，不再进行对后续数据的处理。

```shell
# sed 命令在打印输出第 2 行之后，就停止了，是 q 命令造成的。
$ sed '2q' test.txt
This is line number 1.
This is line number 2.

# 在匹配到 number 1 时，将其替换成 number 0，然后直接退出。
$ sed '/number 1/{ s/number 1/number 0/;q; }' test.txt
This is line number 0.
```

### 4.18 sed 多行命令

sed 包含了三个可用来处理多行文本的特殊命令，分别是：

1. Next 命令（N）：将数据流中的下一行加进来创建一个多行组来处理。
2. Delete（D）：删除多行组中的一行。
3. Print（P）：打印多行组中的一行。

#### 4.18.1 N 多行操作命令

```shell
# N 命令会将下一行文本内容添加到缓冲区已有数据之后（之间用换行符分隔），从而使前后两个文本行同时位于缓冲区中，sed 命令会将这两行数据当成一行来处理。
$ cat data2.txt
This is the header line.
This is the first data line.
This is the second data line.
This is the last line.
# 去掉换行符
$ sed '/first/{ N ; s/\n/ / }' data2.txt
This is the header line.
This is the first data line. This is the second data line.
This is the last line.

# 在数据文件中查找一个可能会分散在两行中的文本短语
$ cat data3.txt
On Tuesday, the Linux System
Administrator's group meeting will be held.
All System Administrators should attend.
Thank you for your attendance.
$ sed 'N ; s/System Administrator/Desktop User/' data3.txt
On Tuesday, the Linux Desktop User's group meeting will be held.
All Desktop Users should attend.
Thank you for your attendance.
```

用 N 命令将发现第一个单词的那行和下一行合并后，即使短语内出现了换行，你仍然可以找到它，这是因为，替换命令在 System 和 Administrator之间用了通配符（.）来匹配空格和换行符这两种情况。但当它匹配了换行符时，它就从字符串中删掉了换行符，导致两行合并成一行。这可能不是你想要的。

要解决这个问题，可以在 sed 脚本中用两个替换命令，一个用来匹配短语出现在多行中的情况，一个用来匹配短语出现在单行中的情况，比如：

```shell
$ sed 'N
> s/System\nAdministrator/Desktop\nUser/
> s/System Administrator/Desktop User/
> ' data3.txt
On Tuesday, the Linux Desktop
User's group meeting will be held.
All Desktop Users should attend.
Thank you for your attendance.
```

但这个脚本中仍有个小问题，即它总是在执行 sed 命令前将下一行文本读入到缓冲区中，当它到了后一行文本时，就没有下一行可读了，此时 N 命令会叫 sed 程序停止，这就导致，如果要匹配的文本正好在最后一行中，sed 命令将不会发现要匹配的数据。

解决这个 bug 的方法是，将单行命令放到 N 命令前面，将多行命令放到 N 命令后面，像这样：

```shell
$ sed '
> s/System\nAdministrator/Desktop\nUser/
> N
> s/System Administrator/Desktop User/
> ' data3.txt
On Tuesday, the Linux Desktop
User's group meeting will be held.
All Desktop Users should attend.
Thank you for your attendance.
```

#### 4.18.2 D 多行删除命令

sed 不仅提供了单行删除命令（d），也提供了多行删除命令 D，其作用是只删除缓冲区中的第一行，也就是说，D 命令将缓冲区中第一个换行符（包括换行符）之前的内容删除掉。

```shell
# 文本的第二行被 N 命令加到了缓冲区，因此 sed 命令第一次匹配就是成功，而 D 命令会将缓冲区中第一个换行符之前（也就是第一行）的数据删除，所以，得到了如上所示的结果。
$ cat data4.txt
On Tuesday, the Linux System
Administrator's group meeting will be held.
All System Administrators should attend.
$ sed 'N ; /System\nAdministrator/D' data4.txt
Administrator's group meeting will be held.
All System Administrators should attend.

# sed会查找空白行，然后用 N 命令来将下一文本行添加到缓冲区。此时如果缓冲区的内容中含有单词 header，则 D 命令会删除缓冲区中的第一行。
$ cat data5.txt

This is the header line.
This is a data line.

This is the last line.
$ sed '/^$/{N ; /header/D}' data5.txt
This is the header line.
This is a data line.

This is the last line.
```

#### 4.18.3 P 多行打印命令

同 d 和 D 之间的区别一样，P（大写）命令和单行打印命令 p（小写）不同，对于具有多行数据的缓冲区来说，它只会打印缓冲区中的第一行，也就是首个换行符之前的所有内容。

```shell
$ cat test.txt
aaa
bbb
ccc
ddd

$ sed '/.*/N;P' test.txt
aaa
aaa
bbb
ccc
ccc
ddd
$ sed '/.*/N;p' test.txt
aaa
bbb
aaa
bbb
ccc
ddd
ccc
ddd
```

第一个 sed 命令，每次都使用 N 将下一行内容追加到缓冲区内容的后面（用换行符间隔），也就是说，第一次时缓冲区中的内容为 aaa\nbbb，但 P（大写） 命令的作用的打印换行符之前的内容，也就是 aaa，之后则是 sed 在自动输出功能输出 aaa 和 bbb（sed 命令会自动将 \n 输出为换行），依次类推，就输出了所看到的结果。

第二个 sed 命令，使用的是 p （小写）单行打印命令，它会将缓冲区中的所有内容全部打印出来（\n 会自动输出为换行），因此，出现了看到的结果。

#### 4.18.4 sed 保持空间

前面我们一直说，sed 命令处理的是缓冲区中的内容，其实这里的缓冲区，应称为模式空间。值得一提的是，模式空间并不是 sed 命令保存文件的唯一空间。sed 还有另一块称为保持空间的缓冲区域，它可以用来临时存储一些数据。

sed 保持空间命令：

| 命令 | 功能                             |
| ---- | -------------------------------- |
| h    | 将模式空间中的内容复制到保持空间 |
| H    | 将模式空间中的内容附加到保持空间 |
| g    | 将保持空间中的内容复制到模式空间 |
| G    | 将保持空间中的内容附加到模式空间 |
| x    | 交换模式空间和保持空间中的内容   |

通常，在使用 h 或 H 命令将字符串移动到保持空间后，最终还要用 g、G 或 x 命令将保存的字符串移回模式空间。保持空间最直接的作用是，一旦我们将模式空间中所有的文件复制到保持空间中，就可以清空模式空间来加载其他要处理的文本内容。

```shell
$ cat data2.txt
This is the header line.
This is the first data line.
This is the second data line.
This is the last line.
$ sed -n '/first/ {h ; p ; n ; p ; g ; p }' data2.txt
This is the first data line.
This is the second data line.
This is the first data line.
```

这个例子的运行过程是这样的：

- sed脚本命令用正则表达式过滤出含有单词first的行；
- 当含有单词 first 的行出现时，h 命令将该行放到保持空间；
- p 命令打印模式空间也就是第一个数据行的内容；
- n 命令提取数据流中的下一行（This is the second data line），并将它放到模式空间；
- p 命令打印模式空间的内容，现在是第二个数据行；
- g 命令将保持空间的内容（This is the first data line）放回模式空间，替换当前文本；
- p 命令打印模式空间的当前内容，现在变回第一个数据行了。

#### 4.18.5 sed改变指定流程

通常，sed 程序的执行过程会从第一个脚本命令开始，一直执行到最后一个脚本命令（D 命令是个例外，它会强制 sed 返回到脚本的顶部，而不读取新的行）。sed 提供了 b 分支命令来改变命令脚本的执行流程，其结果与结构化编程类似。

其中，address 参数决定了哪些行的数据会触发分支命令，label 参数定义了要跳转到的位置。需要注意的是，如果没有加 label 参数，跳转命令会跳转到脚本的结尾。如果我们不想直接跳到脚本的结尾，可以为 b 命令指定一个标签（也就是格式中的 label，最多为 7 个字符长度）。在使用此该标签时，要以冒号开始（比如 :label2），并将其放到要跳过的脚本命令之后。这样，当 sed 命令匹配并处理该行文本时，会跳过标签之前所有的脚本命令，但会执行标签之后的脚本命令。

```shell
[address]b [label]

# 因为 b 命令未指定 label 参数，因此数据流中的第2行和第3行并没有执行那两个替换命令。
$ cat data2.txt
This is the header line.
This is the first data line.
This is the second data line.
This is the last line.
$ sed '{2,3b ; s/This is/Is this/ ; s/line./test?/}' data2.txt
Is this the header test?
This is the first data line.
This is the second data line.
Is this the last test?

# 如果文本行中出现了 first，程序的执行会直接跳到 jump1 标签之后的脚本行。如果分支命令的模式没有匹配，sed 会继续执行所有的脚本命令。
$ sed '{/first/b jump1 ; s/This is the/No jump on/
> :jump1
> s/This is the/Jump here on/}' data2.txt
No jump on header line
Jump here on first data line
No jump on second data line
No jump on last line

# b 分支命令除了可以向后跳转，还可以向前跳转。
# 当缓冲区中的行内容中有逗号时，脚本命令就会一直循环执行，每次迭代都会删除文本中的第一个逗号，并打印字符串，直至内容中没有逗号。
$ echo "This, is, a, test, to, remove, commas." | sed -n '{
> :start
> s/,//1p
> /,/b start
> }'
This is, a, test, to, remove, commas.
This is a, test, to, remove, commas.
This is a test, to, remove, commas.
This is a test to, remove, commas.
This is a test to remove, commas.
This is a test to remove commas.
```

#### 4.18.6 t 测试命令

类似于 b 分支命令，t 命令也可以用来改变 sed 脚本的执行流程。t 测试命令会根据 s 替换命令的结果，如果匹配并替换成功，则脚本的执行会跳转到指定的标签；反之，t 命令无效。跟分支命令一样，在没有指定标签的情况下，如果 s 命令替换成功，sed 会跳转到脚本的结尾（相当于不执行任何脚本命令）。

```shell
[address]t [label]

# 此例中，第一个替换命令会查找模式文本 first，如果匹配并替换成功，命令会直接跳过后面的替换命令；反之，如果第一个替换命令未能匹配成功，第二个替换命令就会被执行。
$ sed '{
> s/first/matched/
> t
> s/This is the/No match on/
> }' data2.txt
No match on header line
This is the matched data line
No match on second data line
No match on last line

$  echo "This, is, a, test, to, remove, commas. " | sed -n '{
> :start
> s/,//1p
> t start
> }'
This is, a, test, to, remove, commas.
This is a, test, to, remove, commas.
This is a test, to, remove, commas.
This is a test to, remove, commas.
This is a test to remove, commas.
This is a test to remove commas.
```

### 4.19 awk命令详解

和 sed 命令类似，awk 命令也是逐行扫描文件（从第 1 行到最后一行），寻找含有目标文本的行，如果匹配成功，则会在该行上执行用户想要的操作；反之，则不对行做任何处理。

这里的匹配规则，和 sed 命令中的 address 部分作用相同，用来指定脚本命令可以作用到文本内容中的具体行，可以使用字符串（比如 /demo/，表示查看含有 demo 字符串的行）或者正则表达式指定。

awk 的主要特性之一是其处理文本文件中数据的能力，它会自动给一行中的每个数据元素分配一个变量。

默认情况下，awk 会将如下变量分配给它在文本行中发现的数据字段：

- $0 代表整个文本行；
- $1 代表文本行中的第 1 个数据字段；
- $2 代表文本行中的第 2 个数据字段；
- $n 代表文本行中的第 n 个数据字段。


前面说过，在 awk 中，默认的字段分隔符是任意的空白字符（例如空格或制表符）。 在文本行中，每个数据字段都是通过字段分隔符划分的。awk 在读取一行文本时，会用预定义的字段分隔符划分每个数据字段。

```shell
awk [选项] '脚本命令' 文件名
# '脚本命令'由以下两部分组成
'匹配规则{执行命令}'

# 在此命令中，/^$/ 是一个正则表达式，功能是匹配文本中的空白行，同时可以看到，执行命令使用的是 print 命令，此命令经常会使用，它的作用很简单，就是将指定的文本进行输出。因此，整个命令的功能是，如果 test.txt 有 N 个空白行，那么执行此命令会输出 N 个 Blank line。
$ awk '/^$/ {print "Blank line"}' test.txt

$ cat data2.txt
One line of test text.
Two lines of test text.
Three lines of test text.
$ awk '{print $1}' data2.txt
One
Two
Three

# 要在命令行上的程序脚本中使用多条命令，只要在命令之间放个分号即可。
$ echo "My name is Rich" | awk '{$4="Christine"; print $0}'
My name is Christine
# 除此之外，也可以一次一行地输入程序脚本命令。
$ awk '{
> $4="Christine"
> print $0}'
My name is Rich <--命令行输入
My name is Christine <--命令行输出

# 在程序文件中，也可以指定多条命令，只要一条命令放一行即可，之间不需要用分号。
$ cat awk.sh
{print $1 "'s home directory is " $6}
$ awk -F: -f awk.sh /etc/passwd
root's home directory is /root
bin's home directory is /bin
daemon's home directory is /sbin
adm's home directory is /var/adm
lp's home directory is /var/spool/lpd
...

# BEGIN 会强制 awk 在读取数据前执行该关键字后指定的脚本命令。END 关键字允许我们指定一些脚本命令，awk 会在读完数据后执行它们。
# 这里的脚本命令中分为 2 部分，BEGIN 部分的脚本指令会在 awk 命令处理数据前运行，而真正用来处理数据的是第二段脚本命令。
$ cat data3.txt
Line 1
Line 2
Line 3
$ awk 'BEGIN {print "The data3 File Contents:"}
> {print $0}
> END {print "End of File"}' data3.txt
The data3 File Contents:
Line 1
Line 2
Line 3
End of File
```

| 选项       | 含义                                                         |
| ---------- | ------------------------------------------------------------ |
| -F fs      | 指定以 fs 作为输入行的分隔符，awk 命令默认分隔符为空格或制表符。 |
| -f file    | 从脚本文件中读取 awk 脚本指令，以取代直接在命令行中输入指令。 |
| -v var=val | 在执行处理过程之前，设置一个变量 var，并给其设备初始值为 val。 |

### 4.20 awk命令的高级玩法

注意，awk 脚本程序中输出函数还可以使用 C 语言中的 printf 函数。

#### 4.20.1 awk 使用变量

在 awk 的脚本程序中，支持使用变量来存取值。awk 支持两种不同类型的变量：

- 内建变量：awk 本身就创建好，用户可以直接拿来用的变量，这些变量用来存放处理数据文件中的某些字段和记录的信息。
- 自定义变量：awk 支持用户自己创建变量。

**内建变量**

awk 程序使用内建变量来引用程序数据里的一些特殊功能。常见的一些内建变量，包括上一节介绍的数据字段变量（$0、$1、$2...$n）以及下表所示的这些变量。

| 变量        | 功能                                                 |
| ----------- | ---------------------------------------------------- |
| FIELDWIDTHS | 由空格分隔的一列数字，定义了每个数据字段的确切宽度。 |
| FNR         | 当前输入文档的记录编号，常在有多个输入文档时使用。   |
| NR          | 输入流的当前记录编号。                               |
| FS          | 输入字段分隔符                                       |
| RS          | 输入记录分隔符，默认为换行符 \n。                    |
| OFS         | 输出字段分隔符，默认为空格。                         |
| ORS         | 输出记录分隔符，默认为换行符 \n。                    |

环境信息变量

| 变量名     | 功能                                                     |
| ---------- | -------------------------------------------------------- |
| ARGC       | 命令行参数个数。                                         |
| ARGIND     | 当前文件在 ARGC 中的位置。                               |
| ARGV       | 包含命令行参数的数组。                                   |
| CONVFMT    | 数字的转换格式，默认值为 %.6g。                          |
| ENVIRON    | 当前 shell 环境变量及其值组成的关联数组。                |
| ERRNO      | 当读取或关闭输入文件发生错误时的系统错误号。             |
| FILENAME   | 当前输入文档的名称。                                     |
| FNR        | 当前数据文件中的数据行数。                               |
| IGNORECASE | 设成非 0 值时，忽略 awk 命令中出现的字符串的字符大小写。 |
| NF         | 数据文件中的字段总数。                                   |
| NR         | 已处理的输入记录数。                                     |
| OFMT       | 数字的输出格式，默认值为 %.6g。                          |
| RLENGTH    | 由 match 函数所匹配的子字符串的长度。                    |
| TSTART     | 由 match 函数所匹配的子字符串的起始位置。                |

```shell
$ cat data1
data11,data12,data13,data14,data15
data21,data22,data23,data24,data25
data31,data32,data33,data34,data35
$ awk 'BEGIN{FS=","; OFS="-"} {print $1,$2,$3}' data1
data11-data12-data13
data21-data22-data23
data31-data32-data33

# 一旦设置了 FIELDWIDTH 变量，awk 就会忽略 FS 变量，并根据提供的字段宽度来计算字段，一旦设置就不能再改变了，因此，这种方法并不适用于变长的字段
$ cat data1b
1005.3247596.37
115-2.349194.00
05810.1298100.1
$ awk 'BEGIN{FIELDWIDTHS="3 5 2 5"}{print $1,$2,$3,$4}' data1b
100 5.324 75 96.37
115 -2.34 91 94.00
058 10.12 98 100.1

$ cat data2
Riley Mullen
123 Main Street
Chicago, IL  60601
(312)555-1234

Frank Williams
456 Oak Street
Indianapolis, IN  46201
(317)555-9876
# 字段分隔符为换行，记录分隔符为空行
$ awk 'BEGIN{FS="\n"; RS=""} {print $1,$4}' data2
Riley Mullen (312)555-1234
Frank Williams (317)555-9876

$ cat data1
data11,data12,data13,data14,data15
data21,data22,data23,data24,data25
data31,data32,data33,data34,data35
# FNR 变量含有当前数据文件中已处理过的记录数，NR 变量则含有已处理过的记录总数。
# 当只使用一个数据文件作为输入时，FNR 和 NR 的值是相同的；如果使用多个数据文件作为输入，FNR 的值会在处理每个数据文件时被重置，而 NR 的值则会继续计数直到处理完所有的数据文件。
$ awk '
> BEGIN {FS=","}
> {print $1,"FNR="FNR,"NR="NR}
> END{print "There were",NR,"records processed"}' data1 data1
data11 FNR=1 NR=1
data21 FNR=2 NR=2
data31 FNR=3 NR=3
data11 FNR=1 NR=4
data21 FNR=2 NR=5
data31 FNR=3 NR=6
There were 6 records processed
```

**自定义变量**

和其他典型的编程语言一样，awk 允许用户定义自己的变量在脚本程序中使用。awk 自定义变量名可以是任意数目的字母、数字和下划线，但不能以数字开头。更重要的是，awk 变量名区分大小写

```shell
$ awk '
> BEGIN{
> testing="This is a test"
> print testing
> testing=45
> print testing
> }'
This is a test
45

$ cat script1
BEGIN{FS=","} {print $n}
$ awk -f script1 n=2 data1
data12
data22
data32
$ awk -f script1 n=3 data1
data13
data23
data33

# 需要注意的是，使用命令行参数来定义变量值会有一个问题，即设置了变量后，这个值在代码的 BEGIN 部分不可用。
$ cat script2
BEGIN{print "The starting value is",n; FS=","}
{print $n}
$ awk -f script2 n=3 data1
The starting value is
data13
data23
data33

# 解决这个问题，可以用 -v 命令行参数，它可以实现在 BEGIN 代码之前设定变量。在命令行上，-v 命令行参数必须放在脚本代码之前。
$ awk -v n=3 -f script2 data1
The starting value is 3
data13
data23
data33
```

#### 4.20.2 awk 使用数组

```shell
var[index]=element

$ awk 'BEGIN{
> capital["Illinois"] = "Springfield"
> print capital["Illinois"]
> }'
Springfield

# 算术运算
$ awk 'BEGIN{
> var[1] = 34
> var[2] = 3
> total = var[1] + var[2]
> print total
> }'
37

# 索引值不会按任何特定顺序返回
$ awk 'BEGIN{
> var["a"] = 1
> var["g"] = 2
> var["m"] = 3
> var["u"] = 4
> for (test in var)
> {
>    print "Index:",test," - Value:",var[test]
> }
> delete var["g"]
> print "---"
> for (test in var)
> {
>    print "Index:",test," - Value:",var[test]
> }
> }'
Index: u  - Value: 4
Index: m  - Value: 3
Index: a  - Value: 1
Index: g  - Value: 2
---
Index: m  - Value: 3
Index: u  - Value: 4
Index: a  - Value: 1
```

#### 4.20.3 awk使用分支结构

```shell
if (condition)
    statement1
else
    statements
或则
if (condition) statement1; else statement2

$ cat data4
10
5
13
50
34
$ awk '{if ($1 > 20) print $1 * 2; else print $1 / 2}' data4
5
2.5
6.5
100
68
```

#### 4.20.4 awk使用循环结构

awk 支持使用的循环结构的用法和 C 语言完全一样，除此之外，awk 还支持使用 break（跳出循环）、continue（终止当前循环）关键字，其用法和 C 语言中也完全相同。

```shell
while (条件) {
   运行代码；
}

$ cat data5
130 120 135
160 113 140
145 170 215
$ awk '{
> total = 0
> i = 1
> while (i < 4)
> {
>    total += $i
>    i++
> }
> avg = total / 3
> print "Average:",avg
> }' data5
Average: 128.333
Average: 137.667
Average: 176.667
```

```shell
do
{
运行代码；
}while(条件)

$ awk '{
> total = 0
> i = 1
> do
> {
>    total += $i
>    i++
> } while (total < 150)
> print total }' data5
250
160
315
```

```shell
for(变量；条件；计数器)
{
    运行代码；
}

$ awk '{
> total = 0
> for (i = 1; i < 4; i++)
> {
>    total += $i
> }
> avg = total / 3
> print "Average:",avg
> }' data5
Average: 128.333
Average: 137.667
Average: 176.667
```

#### 4.20.5 awk使用函数

**内建函数**

和内建变量类似，awk 也提供了不少内建函数，可进行一些常见的数学、字符串以及时间函数运算，如下 所示。

| 函数分类                      | 函数原型                                                     | 函数功能                                                     |
| ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 数学函数                      | atan2(x, y)                                                  | x/y 的反正切，x 和 y 以弧度为单位。                          |
| cos(x)                        | x 的余弦，x 以弧度为单位。                                   |                                                              |
| exp(x)                        | x 的指数函数。                                               |                                                              |
| int(x)                        | x 的整数部分，取靠近零一侧的值。                             |                                                              |
| log(x)                        | x 的自然对数。                                               |                                                              |
| srand(x)                      | 为计算随机数指定一个种子值。                                 |                                                              |
| rand()                        | 比 0 大比 1 小的随机浮点值。                                 |                                                              |
| sin(x)                        | x 的正弦，x 以弧度为单位。                                   |                                                              |
| sqrt(x)                       | x 的平方根。                                                 |                                                              |
| 位运算函数                    | and(v1, v2)                                                  | 执行值 v1 和 v2 的按位与运算。                               |
| compl(val)                    | 执行 val 的补运算。                                          |                                                              |
| lshift(val, count)            | 将值 val 左移 count 位。                                     |                                                              |
| or(v1, v2)                    | 执行值 v1 和 v2 的按位或运算。                               |                                                              |
| rshift(val, count)            | 将值 val 右移 count 位。                                     |                                                              |
| xor(v1, v2)                   | 执行值 v1 和 v2 的按位异或运算。                             |                                                              |
| 字符串函数                    | asort(s [,d])                                                | 将数组 s 按数据元素值排序。索引值会被替换成表示新的排序顺序的连续数字。另外，如果指定了 d，则排序后的数组会存储在数组 d 中。 |
| asorti(s [,d])                | 将数组 s 按索引值排序。生成的数组会将索引值作为数据元素值，用连续数字索引来表明排序顺序。另外如果指定了 d，排序后的数组会存储在数组 d 中。 |                                                              |
| gensub(r, s, h [, t])         | 查找变量 $0 或目标字符串 t（如果提供了的话）来匹配正则表达式 r。如果 h 是一个以 g 或 G 开头的字符串，就用 s 替换掉匹配的文本。如果 h 是一个数字，它表示要替换掉第 h 处 r 匹配的地方。 |                                                              |
| gsub(r, s [,t])               | 查找变量 $0 或目标字符串 t（如果提供了的话）来匹配正则表达式 r。如果找到了，就全部替换成字符串 s。 |                                                              |
| index(s, t)                   | 返回字符串 t 在字符串 s 中的索引值，如果没找到的话返回 0。   |                                                              |
| length([s])                   | 返回字符串 s 的长度；如果没有指定的话，返回 $0 的长度。      |                                                              |
| match(s, r [,a])              | 返回字符串 s 中正则表达式 r 出现位置的索引。如果指定了数组 a，它会存储 s 中匹配正则表达式的那部分。 |                                                              |
| split(s, a [,r])              | 将 s 用 FS 字符或正则表达式 r（如果指定了的话）分开放到数组 a 中，并返回字段的总数。 |                                                              |
| sprintf(format, variables)    | 用提供的 format 和 variables 返回一个类似于 printf 输出的字符串。 |                                                              |
| sub(r, s [,t])                | 在变量 $0 或目标字符串 t 中查找正则表达式 r 的匹配。如果找到了，就用字符串 s 替换掉第一处匹配。 |                                                              |
| substr(s, i [,n])             | 返回 s 中从索引值 i 开始的 n 个字符组成的子字符串。如果未提供 n，则返回 s 剩下的部分。 |                                                              |
| tolower(s)                    | 将 s 中的所有字符转换成小写。                                |                                                              |
| toupper(s)                    | 将 s 中的所有字符转换成大写。                                |                                                              |
| 时间函数                      | mktime(datespec)                                             | 将一个按 YYYY MM DD HH MM SS [DST] 格式指定的日期转换成时间戳值。 |
| strftime(format [,timestamp]) | 将当前时间的时间戳或 timestamp（如果提供了的话）转化格式化日期（采用 shell 函数 date() 的格式）。 |                                                              |
| systime()                     | 返回当前时间的时间戳。                                       |                                                              |

时间戳指的是格林威治时间，即从 1970年1月1日8时1起到现在的总秒数。

**自定义函数**

除了awk 中的内建函数，还可以在 awk 脚本程序中自定义函数，需要注意的是，在定义函数时，它必须出现在所有代码块之前（包括 BEGIN 和 END代码块）。

注意，自定义函数的函数名必须能够唯一标识此函数，换句话说，在同一个 awk 脚本程序中，多个函数的函数名不能相同。同时，函数的参数可以有多个（0 个、1 个或多个）。

```shell
function 函数名(参数1, 参数2, ...)
{
  运行代码;
}

# 此函数会打印记录中的第三个数据字段
function printthird()
{
  print $3
}

# 函数还能用 return 语句返回值
function myrand(limit) {
  return int(limit * rand())
}
```

**创建函数库**

awk 提供了一种途径来将多个函数放到一个库文件中，这样用户就能在所有的 awk 脚本程序中使用了。为了方便大家理解，下面给大家举个实例。

首先，我们需要创建一个存储所有 awk 函数的文件：

```shell
$ cat funclib
function myprint() {
  printf "%-16s - %s\n", $1, $4
}
function myrand(limit)
{
  return int(limit * rand())
}
function printthird()
{
  print $3
}
```

要想让 awk 成功读取 funclib 函数库文件，就需要使用 -f 选项，但此选项无法和 awk 脚本程序同时放到命令行中一起使用。因此，要使用库函数文件，只能再创建一个脚本程序文件，例如：

```shell
$ cat script4
BEGIN{ FS="\n"; RS=""}
{
   myprint()
}
$ awk -f funclib -f script4 data2
Riley Mullen   - (312)555-1234
Frank Williams  - (317)555-9876
Haley Snell   - (313)555-4938
```

### 4.21 查询网络状态

```shell
# 查询端口
netstat -ap | grep 9999
# 杀掉进程
kill pid
```
### 4.22 查询哈希值

作用：可以对比两个文件是否完全一致

```shell
# linux系统下查看某个文件哈希值：XXX为文件名
sha1sum XXX
# Windows系统下查看某个文件哈希值：XXX为文件名
certutil -hashfile XXX SHA1
```

### 4.23 file

可以查询是动态库还是静态库，是64位还是32位

```shell
$file aaa.so
aaa.so: ELF 32-bit LSB shared object,ARM,EABI5 version 1(SYSV),dynamically linked, not stripped

$file aaa.a
aaa.a: current ar archive
```

类似于objdump -a

```shell
$objdump -a aaa.so
aaa.so:  file format elf32-little
aaa.so

$objdump -a aaa.a
In archive aaa.a:
```

其它工具，有时间补充用法：nm strings address2line objdump readelf

## 5. Linux软件二进制包安装

二进制包，也就是源码包经过成功编译之后产生的包。由于二进制包在发布之前就已经完成了编译的工作，因此用户安装软件的速度较快（同 Windows下安装软件速度相当），且安装过程报错几率大大减小。

二进制包是 Linux 下默认的软件安装包，因此二进制包又被称为默认安装软件包。目前主要有以下 2 大主流的二进制包管理系统：

- RPM 包管理系统：功能强大，安装、升级、査询和卸载非常简单方便，因此很多 Linux 发行版都默认使用此机制作为软件安装的管理方式，例如 Fedora、CentOS、SuSE 等。
- DPKG 包管理系统：由 Debian Linux 所开发的包管理机制，通过 DPKG 包，Debian Linux 就可以进行软件包管理，主要应用在 Debian 和 Ubuntu 中。

RPM 包管理系统和 DPKG 管理系统的原理和形式大同小异，可以触类旁通。

### 5.1 RPM包统一命名规则

```shell
包名-版本号-发布次数-发行商-Linux平台-适合的硬件平台-包扩展名
```

例如，RPM 包的名称是`httpd-2.2.15-15.el6.centos.1.i686.rpm`，其中：

- httped：软件包名。这里需要注意，httped 是包名，而 httpd-2.2.15-15.el6.centos.1.i686.rpm 通常称为包全名，包名和包全名是不同的，在某些 Linux 命令中，有些命令（如包的安装和升级）使用的是包全名，而有些命令（包的查询和卸载）使用的是包名，一不小心就会弄错。
- 2.2.15：包的版本号，版本号的格式通常为`主版本号.次版本号.修正号`。
- 15：二进制包发布的次数，表示此 RPM 包是第几次编程生成的。
- el*：软件发行商，el6 表示此包是由 Red Hat 公司发布，适合在 RHEL 6.x (Red Hat Enterprise Unux) 和 CentOS 6.x 上使用。
- centos：表示此包适用于 CentOS 系统。
- i686：表示此包使用的硬件平台，目前的 RPM 包支持的平台如下。
- rpm：RPM 包的扩展名，表明这是编译好的二进制包，可以使用 rpm 命令直接安装。此外，还有以 src.rpm 作为扩展名的 RPM 包，这表明是源代码包，需要安装生成源码，然后对其编译并生成 rpm 格式的包，最后才能使用 rpm 命令进行安装。

| 平台名称 | 适用平台信息                                                 |
| -------- | ------------------------------------------------------------ |
| i386     | 386 以上的计算机都可以安装                                   |
| i586     | 586 以上的计算机都可以安装                                   |
| i686     | 奔腾 II 以上的计算机都可以安装，目前所有的 CPU 是奔腾 II 以上的，所以这个软件版本居多 |
| x86_64   | 64 位 CPU 可以安装                                           |
| noarch   | 没有硬件限制                                                 |

### 5.2 RPM包安装、卸载和升级（rpm命令）详解

通常情况下，RPM 包采用系统默认的安装路径，所有安装文件会按照类别分散安装到如下所示的目录中。

| 安装路径        | 含 义                      |
| --------------- | -------------------------- |
| /etc/           | 配置文件安装目录           |
| /usr/bin/       | 可执行的命令安装目录       |
| /usr/lib/       | 程序所使用的函数库保存位置 |
| /usr/share/doc/ | 基本的软件使用手册保存位置 |
| /usr/share/man/ | 帮助文件保存位置           |

RPM 包的默认安装路径是可以通过命令查询的。

除此之外，RPM 包也支持手动指定安装路径，但此方式并不推荐。因为一旦手动指定安装路径，所有的安装文件会集中安装到指定位置，且系统中用来查询安装路径的命令也无法使用（需要进行手工配置才能被系统识别），得不偿失。

与 RPM 包不同，源码包的安装通常采用手动指定安装路径（习惯安装到 /usr/local/ 中）的方式。既然安装路径不同，同一 apache 程序的源码包和 RPM 包就可以安装到一台 Linux 服务器上（但同一时间只能开启一个，因为它们需要占用同一个 80 端口）。

实际情况中，一台服务器几乎不会同时包含两个 apache 程序，管理员不好管理，还会占用过多的服务器磁盘空间。

#### 5.2.1 RPM 包的安装

注意一定是包全名。涉及到包全名的命令，一定要注意路径，可能软件包在光盘中，因此需提前做好设备的挂载工作。

```shell
$ rpm -ivh 包全名
# -i：安装（install）;
# -v：显示更详细的信息（verbose）;
# -h：打印 #，显示安装进度（hash）;

# 注意，直到出现两个 100% 才是真正的安装成功，第一个 100% 仅表示完成了安装准备工作。
$ rpm -ivh \
/mnt/cdrom/Packages/httpd-2.2.15-15.el6.centos.1.i686.rpm
Preparing...
####################
[100%]
1:httpd
####################
[100%]

# 一次性安装多个软件包，仅需将包全名用空格分开即可。
$ rpm -ivh a.rpm b.rpm c.rpm
```

如果还有其他安装要求（比如强制安装某软件而不管它是否有依赖性），可以通过以下选项进行调整：

- -nodeps：不检测依赖性安装。软件安装时会检测依赖性，确定所需的底层软件是否安装，如果没有安装则会报错。如果不管依赖性，想强制安装，则可以使用这个选项。注意，这样不检测依赖性安装的软件基本上是不能使用的，所以不建议这样做。
- -replacefiles：替换文件安装。如果要安装软件包，但是包中的部分文件已经存在，那么在正常安装时会报"某个文件已经存在"的错误，从而导致软件无法安装。使用这个选项可以忽略这个报错而覆盖安装。
- -replacepkgs：替换软件包安装。如果软件包已经安装，那么此选项可以把软件包重复安装一遍。
- -force：强制安装。不管是否已经安装，都重新安装。也就是 -replacefiles 和 -replacepkgs 的综合。
- -test：测试安装。不会实际安装，只是检测一下依赖性。
- -prefix：指定安装路径。为安装软件指定安装路径，而不使用默认安装路径。


apache 服务安装完成后，可以尝试启动：

```shell
$ service 服务名 start|stop|restart|status
# start：启动服务；
# stop：停止服务；
# restart：重启服务；
# status: 查看服务状态；

# 启动apache服务
$ service httpd start
# 服务启动后，可以查看端口号 80 是否出现。
$ netstat -tlun | grep 80
tcp 0 0 :::80:::* LISTEN
```

#### 5.2.2 RPM包的升级

```shell
$ rpm -Uvh 包全名
# -U（大写）选项的含义是：如果该软件没安装过则直接安装；若没安装则升级至最新版本。

$ rpm -Fvh 包全名
# -F（大写）选项的含义是：如果该软件没有安装，则不会安装，必须安装有较低版本才能升级。
```

#### 5.2.3 RPM包的卸载

RPM 软件包的卸载要考虑包之间的依赖性。例如，我们先安装的 httpd 软件包，后安装 httpd 的功能模块 mod_ssl 包，那么在卸载时，就必须先卸载 mod_ssl，然后卸载 httpd，否则会报错。

```shell
$ rpm -e 包名
# -e 选项表示卸载，也就是 erase 的首字母。
# RPM 软件包的卸载命令支持使用“-nocteps”选项，即可以不检测依赖性直接卸载，但此方式不推荐大家使用，因为此操作很可能导致其他软件也无法征程使用。

# 如果卸载 RPM 软件不考虑依赖性，执行卸载命令会包依赖性错误。
$ rpm -e httpd
error: Failed dependencies:
httpd-mmn = 20051115 is needed by (installed) mod_wsgi-3.2-1.el6.i686
httpd-mmn = 20051115 is needed by (installed) php-5.3.3-3.el6_2.8.i686
httpd-mmn = 20051115 is needed by (installed) mod_ssl-1:2.2.15-15.el6.
centos.1.i686
httpd-mmn = 20051115 is needed by (installed) mod_perl-2.0.4-10.el6.i686
httpd = 2.2.15-15.el6.centos.1 is needed by (installed) httpd-manual-2.2.
15-15.el6.centos.1 .noarch
httpd is needed by (installed) webalizer-2.21_02-3.3.el6.i686
httpd is needed by (installed) mod_ssl-1:2.2.15-15.el6.centos.1.i686
httpd=0:2.2.15-15.el6.centos.1 is needed by(installed)mod_ssl-1:2.2.15-15.el6.centos.1.i686
```

####  5.2.4 rpm命令查询软件包

```shell
rpm 选项 查询对象
# -q：查询软件包是否安装(query)
# -qa：查询系统中所有安装的软件包(query all)
# -qi：查询软件包的详细信息(query information)
# -qip：查询未安装软件包的详细信息(query information package)
# -ql：查询软件包的文件列表(query list)
# -qlp：查询未安装软件包中包含的所有文件以及打算安装的路径(query list package)，注意，由于软件包还未安装，因此需要使用“绝对路径+包全名”的方式才能确定包。
# -qf：命令查询系统文件属于哪个RPM包(query file)，注意，只有使用 RPM 包安装的文件才能使用该命令，手动方式建立的文件无法使用此命令。
# -qR[p]：查询软件包的依赖关系(query requires)

# 注意这里使用的是包名，而不是包全名。因为已安装的软件包只需给出包名，系统就可以成功识别（使用包全名反而无法识别）。
$ rpm -q httpd
httpd-2.2.15-15.el6.centos.1.i686

$ rpm -qa | grep httpd
httpd-devel-2.2.15-15.el6.centos.1.i686
httpd-tools-2.2.15-15.el6.centos.1.i686
httpd-manual-2.2.15-15.el6.centos.1.noarch
httpd-2.2.15-15.el6.centos.1.i686

$ rpm -qi httpd
Name : httpd Relocations:(not relocatable)
#包名
Version : 2.2.15 Vendor:CentOS
#版本和厂商
Release : 15.el6.centos.1 Build Date: 2012年02月14日星期二 06时27分1秒
#发行版本和建立时间
Install Date: 2013年01月07日星期一19时22分43秒
Build Host:
c6b18n2.bsys.dev.centos.org
#安装时间
Group : System Environment/Daemons Source RPM:
httpd-2.2.15-15.el6.centos.1.src.rpm
#组和源RPM包文件名
Size : 2896132 License: ASL 2.0
#软件包大小和许可协议
Signature :RSA/SHA1,2012年02月14日星期二 19时11分00秒，Key ID
0946fca2c105b9de
#数字签名
Packager：CentOS BuildSystem <http://bugs.centos.org>
URL : http://httpd.apache.org/
#厂商网址
Summary : Apache HTTP Server
#软件包说明
Description:
The Apache HTTP Server is a powerful, efficient, and extensible web server.
#描述

$ rpm -ql httpd
/etc/httpd
/etc/httpd/conf
/etc/httpd/conf.d
/etc/httpd/conf.d/README
/etc/httpd/conf.d/welcome.conf
/etc/httpd/conf/httpd.conf
/etc/httpd/conf/magic
…省略部分输出…

$ rpm -qlp /mnt/cdrom/Packages/bind-9.8.2-0.10.rc1.el6.i686.rpm
/etc/NetworkManager/dispatcher.d/13-named
/etc/logrotate.d/named
/etc/named
/etc/named.conf
/etc/named.iscdlv.key
/etc/named.rfc1912.zones
…省略部分输出…

$ rpm -qf /bin/ls
coreutils-8.4-19.el6.i686

$ rpm -qR httpd
/bin/bash
/bin/sh
/etc/mime.types
/usr/sbin/useradd
apr-util-ldap
chkconfig
config(httpd) = 2.2.15-15.el6.centos.1
httpd-tods = 2.2.15-15.el6.centos.1
initscripts >= 8.36
…省略部分输出…

$ rpm -qRp /mnt/cdrom/Packages/bind-9.8.2-0.10.rc1.el6.i686.rpm
/bin/bash
/bin/sh
bind-libs = 32:9.8.2-0.10.rc1.el6
chkconfig
chkconfig
config(bind) = 32:9.8.2-0.10.rc1.el6
grep
libbind9.so.80
libc.so.6
libc.so.6(GLIBC_2.0)
libc.so.6(GLIBC_2.1)
…省略部分输出…
```

#### 5.2.5 RPM包验证和数字证书（数字签名）

执行 `rpm -qa` 命令可以看到，Linux 系统中装有大量的 RPM 包，且每个包都含有大量的安装文件。因此，为了能够及时发现文件误删、误修改文件数据、恶意篡改文件内容等问题，Linux 提供了以下两种监控（检测）方式：

- RPM 包校验：其实就是将已安装文件和 /var/lib/rpm/ 目录下的数据库内容进行比较，确定文件内容是否被修改。
- RPM 包数字证书校验：用来校验 RPM 包本身是否被修改。

**Linux RPM 包校验**

RPM 包校验可用来判断已安装的软件包（或文件）是否被修改，此方式可使用的命令格式分为以下 3 种。

```shell
$ rpm -Va
# -Va 选项表示校验系统中已安装的所有软件包。
$ rpm -V 已安装的包名
# -V 选项表示校验指定 RPM 包中的文件，是 verity 的首字母。
$ rpm -Vf 系统文件名
# -Vf 选项表示校验某个系统文件是否被修改。

# 校验 apache 软件包中所有的安装文件是否被修改
# 可以看到，执行后无任何提示信息，表明所有用 apache 软件包安装的文件均未改动过，还和从原软件包安装的文件一样。
$ rpm -V httpd

# 修改 apache 的配置文件 /etc/httpd/conf/httpd.conf
$ vim /etc/httpd/conf/httpd.conf
...省略部分内容...
Directorylndex index.html index.html.var index.php
\#这句话是定义apache可以识别的默认网页文件名。在后面加入了index.php
\#这句话大概有400行左右
…省略部分内容...

$ rpm -V httpd
S.5....T. c /etc/httpd/conf/httpd.conf
```

可以看到，结果显示了文件被修改的信息。该信息可分为以下 3 部分：

1. 最前面的 8 个字符（S.5....T）都属于验证信息，各字符的具体含义如下：
   - S：文件大小是否改变。
   - M：文件的类型或文件的权限（rwx）是否改变。
   - 5：文件MD5校验和是否改变（可以看成文件内容是否改变）。
   - D：设备的主从代码是否改变。
   - L：文件路径是否改变。
   - U：文件的属主（所有者）是否改变。
   - G：文件的属组是否改变。
   - T：文件的修改时间是否改变。
   - .：若相关项没发生改变，用 . 表示。
2. 被修改文件类型，大致可分为以下几类：
   - c：配置文件（configuration file）。
   - d：普通文档（documentation）。
   - g："鬼"文件（ghost file），很少见，就是该文件不应该被这个 RPM 包包含。
   - l：授权文件（license file）。
   - r：描述文件（read me）。
3. 被修改文件所在绝对路径（包含文件名）。


由此，S.5....T. c S.5....T. c /etc/httpd/conf/httpd.conf 表达的完整含义是：配置文件 httpd.conf 的大小、内容、修改时间被人为修改过。

注意，并非所有对文件做修改的行为都是恶意的。通常情况下，对配置文件做修改是正常的，比如说配置 apache 就要修改其配置文件，而如果验证信息提示对二进制文件做了修改，这就需要小心，除非是自己故意修改的。

**Linux RPM数字证书验证**

RPM 包校验方法只能用来校验已安装的 RPM 包及其安装文件，如果 RPM 包本身就被动过手脚，此方法将无法解决问题，需要使用 RPM 数字证书验证方法。

简单的理解，RPM 包校验其实就是将现有安装文件与最初使用 RPM 包安装时的初始文件进行对比，如果有改动则提示给用户，因此这种方式无法验证 RPM 包本身被修改的情况。

数字证书，又称数字签名，由软件开发商直接发布。Linux 系统安装数字证书后，若 RPM 包做了修改，此包携带的数字证书也会改变，将无法与系统成功匹配，软件无法安装。

可以将数字证书想象成自己的签名，是不能被模仿的（厂商的数字证书是唯一的），只有我认可的文件才会签名（只要是厂商发布的软件，都符合数字证书验证）；如果我的文件被人修改了，那么我的签名就会变得不同（如果软件改变，数字证书就会改变，从而通不过验证。当然，现实中人的手工签名不会直接改变，所以数字证书比手工签名还要可靠）。

使用数字证书验证 RPM 包的方法具有如下 2 个特点：

1. 必须找到原厂的公钥文件，然后才能进行安装。
2. 安装 RPM 包会提取 RPM 包中的证书信息，然后和本机安装的原厂证书进行验证。如果验证通过，则允许安装；如果验证不通过，则不允许安装并发出警告。

```shell
# 数字证书默认会放到系统中`/etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6`位置处，通过以下命令也可验证：
$ ll /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6
-rw-r--r--.1 root root 1706 6 月 26 17:29 /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6

# 安装数字证书的命令如下，--import表示导入数字证书：
$ rpm --import /efc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6

# 数字证书安装完成后，可使用如下命令进行验证：
$ rpm -qa | grep gpg-pubkey
gpg-pubkey-c105b9de-4e0fd3a3

# 数字证书本身也是一个 RPM 包，因此可以用 rpm 命令查询数字证书的详细信息，也可以将其卸载。查询数字证书详细信息的命令如下：
$ rpm -qi gpg-pubkey-c105b9de-4e0fd3a3
Name : gpg-pubkey
Relocations: (not relocatable)
Version : c105b9de Vendor: (none)
Release : 4e0fd3a3 Build Date: 2012年11月12日 星期一 23时05分20秒
Install Date: 2012年11月12日星期一23时05分20秒 Build Host: local host
Group : Public Keys
Source RPM: (none)
Size : 0
License: pubkey
…省略部分输出…
-----END PGP PUBLIC KEY BLOCK----

# 卸载数字证书可以使用 -e 选项，命令如下：
$ rpm -e gpg-pubkey-c105b9de-4ead3a3
```

可以看到，数字证书已成功安装。在装有数字证书的系统上安装 RPM 包时，系统会自动验证包的数字证书，验证通过则可以安装，反之将无法安装（系统会报错）。

虽然数字证书可以手动卸载，但不推荐大家将其卸载。

#### 5.2.6 提取RPM包文件(cpio命令)详解

使用 cpio 命令备份或恢复数据，需注意以下几点：

- 使用 cpio 备份数据时如果使用的是绝对路径，那么还原数据时会自动恢复到绝对路径下；同理，如果备份数据使用的是相对路径，那么数据会还原到相对路径下。
- cpio 命令无法自行指定备份（或还原）的文件，需要目标文件（或目录）的完整路径才能成功读取，因此此命令常与 find 命令配合使用。
- cpio 命令恢复数据时不会自动覆盖同名文件，也不会创建目录（直接解压到当前文件夹）。

```shell
### cpio 命令主要有以下 3 种基本模式：
## 1. "-o" 模式：指的是 copy-out 模式，就是把数据备份到文件库中，命令格式如下：
$ cpio -o[vcB] > [文件丨设备]
# -o：copy-out模式，备份；
# -v：显示备份/还原过程；
# -c：使用较新的portable format存储方式；
# -B：设定输入/输出块为 5120Bytes，而不是模式的 512Bytes；
## 2. "-i" 模式：指的是 copy-in 模式，就是把数据从文件库中恢复，命令格式如下：
$ cpio -i[vcdu] < [文件|设备]
# -i：copy-in 模式，还原；
# -d：还原时自动新建目录；
# -u：自动使用较新的文件覆盖较旧的文件；
## 3. "-p" 模式：指的是复制模式，使用 -p 模式可以从某个目录读取所有文件，但并不将其备份到 cpio 库中，而是直接复制为其他文件。

# 利用find命令指定要备份/etc/目录，使用>导出到etc.cpio文件
$ find /etc -print | cpio -ocvB > /root/etc.cpio
$ ll -h etc.cpio
-rw--r--r--.1 root root 21M 6月5 12:29 etc.cpio

# 使用 cpio 恢复之前备份的数据，如果大家査看一下当前目录/root/，就会发现没有生成/etc/目录。这是因为备份时/etc/目录使用的是绝对路径，所以数据直接恢复到/etc/系统目录中，而没有生成在/root/etc/目录中
$ cpio -idvcu < /root/etc.cpio

# 使用 -p 将 /boot/ 复制到 /test/boot 目录中可以执行如下命令：
[root@localhost ~]$ cd /tmp/
[root@localhost tmp]$ rm -rf*
[root@localhost tmp]$ mkdir test
[root@localhost tmp]$ find /boot/ -print | cpio -p /tmp/test
[root@localhost tmp]$ ls test/boot
```

**使用 cpio 命令提取 RPM 包中指定文件**

在服务器使用过程，如果系统文件被误修改或误删除，可以考虑使用 cpio 命令提取出原 RPM 包中所需的系统文件，从而修复被误操作的源文件。

```shell
# RPM 包允许逐个提取包中文件，rpm2cpio 就是将 RPM 包转换为 cpio 格式的命令，通过 cpio 命令即可从 cpio 文件库中提取出指定文件。
[root@localhost ~]$ rpm2cpio 包全名 | cpio -idv .文件绝对路径
```

举个例子，假设我们不小心把 /bin/ls 命令删除了，通常有以下 2 种方式修复：

1. 将 coreutils-8.4-19.el6.i686 包（包含 ls 命令的 RPM 包）通过 -force 选项再安装一遍；
2. 使用 cpio 命令从 coreutils-8.4-19.el6.i686 包中提取出 /bin/ls 文件，然后将其复制到相应位置；

```shell
# 这里我们选择第 2 种方式
# 查看ls文件属于哪个软件包
[root@localhost ~]$ rpm -qf /bin/ls
coreutils-8.4-19.el6.i686

# 把/bin/ls命令移动到/root/目录下，造成误删除的假象
[root@localhost ~]$ mv /bin/ls /root/
[root@localhost ~]$ ls
-bash: ls: command not found
# 提取ls命令文件到当前目录下
[root@localhost ~]$ rpm2cpio /mnt/cdrom/Packages/coreutils-8.4-19.el6.i686.rpm | cpio -idv ./bin/ls
# 把提取出来的ls命令文件复制到/bin/目录下
[root@localhost ~]$ cp /root/bin/ls /bin/
# 可以看到，ls命令又可以正常使用了
[root@localhost ~]$ ls
anaconda-ks.cfg bin inittab install.log install.log.syslog ls
```

#### 5.2.7 SRPM源码包安装（两种方式）

| 文件格式 | 文件名格式  | 直接安装与否 | 内含程序类型   | 可否修改参数并编译 |
| -------- | ----------- | ------------ | -------------- | ------------------ |
| RPM      | xxx.rpm     | 可           | 已编译         | 不可               |
| SRPM     | xxx.src.rpm | 不可         | 未编译的源代码 | 可                 |

SRPM 包是未经编译的源码包，无法直接用来安装软件，需要经过以下 2 步：

1. 将 SRPM 包编译成二进制的 RPM 包；
2. 使用编译完成的 RPM 包安装软件；

使用 SRPM 包安装软件（编译 SRPM 包）的方式有以下 2 种：

1. 利用 rpmbuild 命令可以直接使用 SRPM 包安装软件，也可以先将 SRPM 包编译成 RPM 包，再使用 RPM 包安装软件；
2. 利用 *.spec 文件可实现将 SRPM 包编译成 RPM 包，再使用 RPM 包安装软件；

```shell
# rpmbuild 命令也是一个程序，但是这个程序不会默认安装，所以要想使用 rpmbuild 命令就必须提前安装。这里我们使用 rpm 命令来安装 rpmbuild 命令。
[root@localhost~]$ rpm -ivh /mnt/cdroin/Packages/rpm-build-4.8.0-27.el6.i686.rpm
Preparing...
###################
[100%]
1:rpm-build
###################
[100%] -->出现两个 100% 才证明 rpmbuild 安装成功。
```

需要注意的是，SRPM 本质上仍属于 RPM 包，所以安装时仍需考虑包之间的依赖性，要先安装它的依赖包，才能正确安装。

```shell
# 如果我们只想安装 SRPM 包，而不用修改源代码，那么直接使用 rpmbuild 命令即可。使用 rpmbuild 安装 SRPM 包的命令格式如下 
[root@localhost ~]$ rpmbuild [选项] 包全名
# -rebuild：编译 SRPM 包生成 RPM 二进制包；
# -recompile：编译 SRPM 包，同时安装。
  
# 这里选择使用 -rebuild 选项先将 SRPM 包编译成 RPM 二进制包。exit 0 是编译成功的标志，此编译过程产生的临时文件会自动删除。SRPM 包编译完成后，会在当前目录生成 rpmbuild 目录，整个编译过程生成的文件（软件包）都存在这里。
[root@localhost ~]$ rpmbuild -rebuild httpd-2.2.15-5.el6.src.rpm
warning: InstallSourcePackage at: psm.c:244: Header V3 RSA/SHA256 Signature, key
ID fd431d51: NOKEY
warning: user mockbuild does not exist - using root
warning: group mockbuild does not exist - using root
# 警告为mockbuild用户不存在，使用root代替。这里不是报错，不用紧张
…省略部分输出…
Wrote: /root/rpmbuild/RPMS/i386/ httpd-2.2.15-5.el6.i386.rpm
Wrote: /root/rpmbuild/RPMS/i386/httpd-devel-2.2.15-5.el6.i386.rpm
Wrote: /root/rpmbuild/RPMS/noarch/httpd-manual-2.2.15-5.el6.noarch.rpm
Wrote: /root/rpmbuild/RPMS/i386/httpd-tools-2.2.15-5.el6.i386.rpm
Wrote: /root/rpmbuild/RPMS/i386/ mod_ssl-2.2.15-5.el6.i386.rpm
# 写入RPM包的位置，只要看到，就说明编译成功
Executing(%clean): /bin/sh -e/var/tmp/rpm-tmp.Wb8TKa
\+ umask 022
\+ cd/root/rpmbuild/BUILD
\+ cd httpd-2.2.15
\+ rm -rf /root/rpmbuild/BUILDROOT/httpd-2.2.15-5.el6.i386
\+ exit 0
Executing(-clean): /bin/sh -e/var/tmp/rpm-tmp.3UBWql
\+ umask 022
\+ cd/root/rpmbuild/BUILD
\+ rm-rf httpd-2.2.15
\+ exit 0

[root@localhost ~]$ ls /root/rpmbuild/
BUILD RPMS SOURCES SPECS SRPMS

# 编译好的 RPM 包保存在 /root/rpmbuild/RPMS/ 目录下
[root@localhost ~]$ ll /root/rpmbuild/RPMS/i386/
-rw--r--r-- 1 root root 3039035 11月19 06:30 httpd-2.2.15-5.el6.i386.rpm
-rw--r--r-- 1 root root 154371 11月19 06:30 httpd-devel-2.2.15-5.el6.i386.rpm
-rw--r--r-- 1 root root 124403 11月19 06:30 httpd-tools-2.2.15-5.el6.i386.rpm
-rw--r--r-- 1 root root 383539 11月19 06:30 mod_ssl-2.2.15-5.el6.i386.rpm
```

| 文件名  | 文件内容                                                     |
| ------- | ------------------------------------------------------------ |
| BUILD   | 编译过程中产生的数据保存位置                                 |
| RPMS    | 编译成功后，生成的 RPM 包保存位置                            |
| SOURCES | 从 SRPM 包中解压出来的源码包（*.tar.gz）保存位置             |
| SPECS   | 生成的设置文件的安装位置。第二种安装方法就是利用这个文件进行安装的 |
| SRPMS   | 放置 SRPM 包的位置                                           |

如此，我们就得到可直接安装软件的 RPM 包。实际上，使用 rpmbuild命令编译 SRPM 包经历了以下 3 个过程：

1. 先把 SRPM 包解开，得到源码包；
2. 对源码包进行编译，生成二进制文件；
3. 把二进制文件重新打包生成 RPM 包。

**利用 *.spec 文件安装**

```shell
# 想利用 .spec 文件安装软件，需先将 SRPM 包解开。当然，我们可以使用 rpmbuild 命令解开 SRPM 包，但这里选择另一种方式。
[root@localhost ~]$ rpm -i httpd-2.2.15-5.el6.src.rpm
# -i 选项用于安装 rpm 包时表示安装，但对于 SRPM 包的安装来说，这里只会将 .src.rpm 包解开后将个文件放置在当前目录下的 rpmbuild 目录中，并不涉及安装操作。
# 通过此命令，也可以在当前目录下生成 rpmbuild 目录，此 rpmbuild 目录中仅有 SOURCES 和 SPECS 两个子目录。其中，SOURCES 目录中放置的是源码，SPECS 目录中放置的是设置文件。

# 接下来使用 SPECS 目录中的设置文件生成 RPM 包。
[root@localhost ~]$ rpmbuild -ba /root/rpmbuild/SPECS/httpd.spec
# 其中，-ba 选项的含义是编译，会同时生成 RPM 二进制包和 SRPM 源码包。这里还可以使用 -bb 选项用来仅生成 RPM 二进制包。
# 命令执行完成，会在 /root/rpmbuild/ 目录下生成 BUILD、RPMS、SOURCES、SPECS 和 SRPMS 目录，RPM 包放在 RPMS 目录中，SRPM 包生成在 SRPMS 目录中。
```

以上两种方式都可实现将 SRPM 包编译为 RPM 二进制包，剩下的工作就是使用 RPM 包安装软件。

#### 5.2.8 重建RPM数据库（修复损坏的RPM数据库）

我们知道，RPM 包是很多 Linux 发行版（Fefora、RedHat、SuSE 等）采用的软件包管理方式，安装到系统中的各 RPM 包，其必要信息都会保存到 RPM 数据库中，以便用户使用 rpm 命令对软件包执行查询、安装和卸载等操作。

但并非所有的用户操作都“按常理出牌”，例如 RPM 包在升级过程被强行退出、RPM 包安装意外中断等误操作，都可能使 RPM 数据库出现故障，后果是当安装、删除、査询软件包时，请求无法执行，如图所示：![RPM数据库出现故障](http://c.biancheng.net/uploads/allimg/190327/2-1Z32G523055J.gif)

这时就需要重建 RPM 数据库，执行如下 2 步操作：

```shell
# 1. 删除当前系统中已损坏的RPM数据库。
[root@localhost ~]$ rm -f /var/lib/rpm/_db.*
# 2. 重建 RPM 数据库，这一步需花费一定时间才能完成。
[root@localhost -]$ rpm -rebuilddb
```

 除了用户误操作导致 RPM 数据库崩溃，有些黑客入侵系统后，为避免系统管理员通过 RPM 包校验功能检测出问题，会更改 RPM 数据库。理论上，系统一旦被黑客“光顾”，则做的任何操作都将不可信。

对于这种情况，我们可以按照以下步骤对文件进行检测：

```shell
# 1. 对于要校验的文件或命令，找到它属于哪个软件包。
[root@localhost ~]$ rpm -qf/etc/rc.d/init.d/smb
samba-3.0.23c-2
   
# 2. 使用 -dump 选项查看每个文件的信息，使用 grep 命令提取对应文件信息：
[root@localhost ~]$ rpm -ql -dump samba|grep /etc/rc.d/init.d/smb
/etc/rc.d/init.d/smb 2087 1157165946 b1c26e5292157a83cadabe851bf9b2f9 0100755 root root 1 0 0X
# 此信息中，“2087”表示 smb 文件最初的字符数，“b1c26e5292157a83cadabe851bf9b2f9”表示 smb 文件的 MD5 校验值，“0755 root root”表示文件权限及所有者、所属组。
   
# 3. 查看实际的文件，通过对比文件大小，所有人、所属组、权限、MD5 校验值等数据，判断文件是否被改动过：
[root@localhost ~]$ ls -l /etc/rc.d/init.d/smb
-rwxr-xr-x 1 root root 2087 Sep 2 2006/etc/rc.d/init.d/smb
[root@localhost ~]$ md5sum /etc/rc.d/init.d/smb
b1c26e5292157a83cadabe851bf9b2f9 /etc/rc.d/init.d/smb
# 以上校验结果显示，系统的 /etc/rc.d/init.d/smb 文件的信息和通过 rpm-ql-dump Samba 命令获取的信息一致，因此可以断定此文件没有被入侵或更改。
```

注意，如果确信 RPM 数据库遭到了修改，就要基于从光盘或者其他值得信赖的来源处获得的 Samba RPM 文件进行检査。

```shell
[root@localhost~]$ rpm -ql --dump -p /mnt/cdrom/Fedora/RPMS/samba-3.0.23c-2.i386.rpm | grep /etc/rc.d/init.d/smb
warning: samba-3.0.23c-2.i386.rpm: Header V3 DSA signature: NOKEY, key ID 412a&62
/etc/rc.d/init.d/smb 2087 1157165946 b1c26e5292157a83cadabe851 bf9b2f9 0100755 root root 1 0 0 X
```

得到的结果如果和基于 RPM 数据库运行的命令结果不同，说明 RPM 数据库已被更改，就需要修正文件错误和系统漏洞，重建 RPM 数据库。

## 6. vim

![](http://c.biancheng.net/uploads/allimg/181008/2-1Q00Q41T01J.jpg)

### 6.1 Vim 打开文件

| Vi 使用的选项          | 说 明                                             |
| ---------------------- | ------------------------------------------------- |
| vim filename           | 打开或新建一个文件，并将光标置于第一行的首部      |
| vim -r filename        | 恢复上次 vim 打开时崩溃的文件                     |
| vim -R filename        | 把指定的文件以只读方式放入 Vim 编辑器中           |
| vim + filename         | 打开文件，并将光标置于最后一行的首部              |
| vi +n filename         | 打开文件，并将光标置于第 n 行的首部               |
| vi +/pattern filename  | 打幵文件，并将光标置于第一个与 pattern 匹配的位置 |
| vi -c command filename | 在对文件进行编辑前，先执行指定的命令              |

### 6.2 Vim 插入文本

从命令模式进入输入模式进行编辑，可以按下 I、i、O、o、A、a 等键来完成，使用不同的键，光标所处的位置不同。

| 快捷键    | 功能描述                                                     |
| --------- | ------------------------------------------------------------ |
| i         | 在当前光标所在位置插入随后输入的文本，光标后的文本相应向右移动 |
| I         | 在光标所在行的行首插入随后输入的文本，行首是该行的第一个非空白字符，相当于光标移动到行首执行 i 命令 |
| o         | 在光标所在行的下面插入新的一行。光标停在空行首，等待输入文本 |
| O（大写） | 在光标所在行的上面插入新的一行。光标停在空行的行首，等待输入文本 |
| a         | 在当前光标所在位置之后插入随后输入的文本                     |
| A         | 在光标所在行的行尾插入随后输入的文本，相当于光标移动到行尾再执行 a 命令 |

### 6.3 Vim 查找文本

如果想忽略大小写，则输入命令 ":set ic"；调整回来输入":set noic"。

如果在字符串中出现特殊符号，则需要加上转义字符 "\"。常见的特殊符号有 \、*、?、$ 等。如果出现这些字符，例如，要查找字符串 "10$"，则需要在命令模式中输入 "/10\$"。

| 快捷键 | 功能描述                         |
| ------ | -------------------------------- |
| /abc   | 从光标所在位置向前查找字符串 abc |
| /^abc  | 查找以 abc 为行首的行            |
| /abc$  | 查找以 abc 为行尾的行            |
| ?abc   | 从光标所在为主向后查找字符串 abc |
| n      | 向同一方向重复上次的查找指令     |
| N      | 向相反方向重复上次的查找指定     |

### 6.4 Vim 替换文本

| 快捷键          | 功能描述                                                     |
| --------------- | ------------------------------------------------------------ |
| r               | 替换光标所在位置的字符                                       |
| R               | 从光标所在位置开始替换字符，其输入内容会覆盖掉后面等长的文本内容，按“Esc”可以结束 |
| :s/a1/a2/g      | 将当前光标所在行中的所有 a1 用 a2 替换                       |
| :n1,n2s/a1/a2/g | 将文件中 n1 到 n2 行中所有 a1 都用 a2 替换                   |
| :g/a1/a2/g      | 将文件中所有的 a1 都用 a2 替换                               |

```shell
# 将所有的 "root" 替换为 "liudehua"
:1, $s/root/liudehua/g
或
:%s/root/liudehua/g
# 只替换从第 10 行到第 20 行的 "root"
:10,20 s/root/liudehua/g
```

### 6.5 Vim删除文本

注意，被删除的内容并没有真正删除，都放在了剪贴板中。将光标移动到指定位置处，按下 "p" 键，就可以将刚才删除的内容又粘贴到此处。

| 快捷键  | 功能描述                               |
| ------- | -------------------------------------- |
| x       | 删除光标所在位置的字符                 |
| dd      | 删除光标所在行                         |
| ndd     | 删除当前行（包括此行）后 n 行文本      |
| dG      | 删除光标所在行一直到文件末尾的所有内容 |
| D       | 删除光标位置到行尾的内容               |
| :a1,a2d | 函数从 a1 行到 a2 行的文本内容         |

### 6.6 Vim复制和粘贴文本

| 快捷键    | 功能描述                                                   |
| --------- | ---------------------------------------------------------- |
| p         | 将剪贴板中的内容粘贴到光标后                               |
| P（大写） | 将剪贴板中的内容粘贴到光标前                               |
| y         | 复制已选中的文本到剪贴板                                   |
| yy        | 将光标所在行复制到剪贴板，此命令前可以加数字 n，可复制多行 |
| yw        | 将光标位置的单词复制到剪贴板                               |

### 6.7 Vim其他常用快捷键

某些情况下，可能需要把两行进行连接。比如说，下面的文件中有两行文本，现在需要将其合并成一行（实际上就是将两行间的换行符去掉）。可以直接在命令模式中按下 "J" 键。

如果不小心误删除了文件内容，则可以通过 "u" 键来撤销刚才执行的命令。如果要撤销刚才的多次操作，可以多按几次 "u" 键。

### 6.8 Vim 保存退出文本

需要注意的是，"w!" 和 "wq!" 等类似的指令，通常用于对文件没有写权限的时候（显示 readonly），但如果你是文件的所有者或者 root 用户，就可以强制执行。

| 命令        | 功能描述                                           |
| ----------- | -------------------------------------------------- |
| :wq         | 保存并退出 Vim 编辑器                              |
| :wq!        | 保存并强制退出 Vim 编辑器                          |
| :q          | 不保存就退出 Vim 编辑器                            |
| :q!         | 不保存，且强制退出 Vim 编辑器                      |
| :w          | 保存但是不退出 Vim 编辑器                          |
| :w!         | 强制保存文本                                       |
| :w filename | 另存到 filename 文件                               |
| x！         | 保存文本，并退出 Vim 编辑器，更通用的一个 vim 命令 |
| ZZ          | 直接退出 Vim 编辑器                                |

### 6.9 Vim移动光标快捷键汇总

**Vim光标以单词为单位移动**

| 快捷键   | 功能描述                            |
| -------- | ----------------------------------- |
| w 或 W   | 光标移动至下一个单词的单词首        |
| b 或 B   | 光标移动至上一个单词的单词首        |
| e 或 E   | 光标移动至下一个单词的单词尾        |
| nw 或 nW | n 为数字，表示光标向右移动 n 个单词 |
| nb 或 nB | n 为数字，表示光标向左移动 n 个单词 |

**Vim光标移动至行首或行尾**

| 快捷键 | 功能描述                                 |
| ------ | ---------------------------------------- |
| 0 或 ^ | 光标移动至当前行的行首                   |
| $      | 光标移动至当前行的行尾                   |
| n$     | 光标移动至当前行只有 n 行的行尾，n为数字 |

**Vim光标移动至指定字符**

| 快捷键 | 功能描述                          |
| ------ | --------------------------------- |
| fx     | 光标移动至当前行中下一个 x 字符处 |
| Fx     | 光标移动至当前行中下一个 x 字符处 |

**Vim光标移动到指定行**

| 快捷键 | 功能描述                                                 |
| ------ | -------------------------------------------------------- |
| gg     | 光标移动到文件开头                                       |
| G      | 光标移动至文件末尾                                       |
| nG     | 光标移动到第 n 行，n 为数字                              |
| :n     | 编辑模式下使用的快捷键，可以将光标快速定义到指定行的行首 |

**Vim光标移动到匹配的括号处**

程序员在编辑程序时，经常会为将光标移动到与一个 "(" 匹配的 ")" （对于 [] 和 {} 也是一样的）处而感到头疼。Vim 里面提供了一个非常方便地査找匹配括号的命令，这就是 "%"。

可以将光标先定位在 "{" 处，然后再使用 "％" 命令，使之定位在 "}" 处。

### 6.10 Vim 撤销和恢复撤销快捷键

| 快捷键    | 功能                                                         |
| --------- | ------------------------------------------------------------ |
| u（小写） | undo 的第 1 个字母，功能是撤销最近一次对文本做的修改操作。   |
| Ctrl+R    | Redo 的第 1 个字母，功能是恢复最近一次所做的撤销操作。       |
| U（大写） | 第一次会撤销对一行文本（光标所在行）做过的全部操作，第二次使用该命令会恢复对该行文本做过的所有操作。 |

例子：

```shell
http://c.biancheng.net
# yy后移动到下一行，执行两次p
http://c.biancheng.net
http://c.biancheng.net
http://c.biancheng.net
# u
http://c.biancheng.net
http://c.biancheng.net
# u
http://c.biancheng.net
# Ctrl+R
http://c.biancheng.net
http://c.biancheng.net
# Ctrl+R
http://c.biancheng.net
http://c.biancheng.net
http://c.biancheng.net
# 修改
http://c.biancheng.net
http://c.biancheng.net
Linux教程 http://c.biancheng.net/linux_tutorial/
# U
http://c.biancheng.net
http://c.biancheng.net
http://c.biancheng.net
# U
http://c.biancheng.net
http://c.biancheng.net
Linux教程 http://c.biancheng.net/linux_tutorial/
```

### 6.11 Vim可视化模式及其用法

通过在 Vim 命令模式下键入不同的键，可以进入不同的可视化模式。

| 命令             | 功能                                                         |
| ---------------- | ------------------------------------------------------------ |
| v（小写）        | 又称字符可视化模式，此模式下目标文本的选择是以字符为单位的，也就是说，该模式下要一个字符一个字符的选中要操作的文本。 |
| V（大写）        | 又称行可视化模式，此模式化目标文本的选择是以行为单位的，也就是说，该模式化可以一行一行的选中要操作的文本。 |
| Ctrl+v（组合键） | 又称块可视化模式，该模式下可以选中文本中的一个矩形区域作为目标文本，以按下 Ctrl+v 位置作为矩形的一角，光标移动的终点位置作为它的对角。 |

在 Vim 命令模式下编辑文本的很多命令，在可视化模式下仍然可以使用。常用命令如下。

| 命令      | 功能                                                         |
| --------- | ------------------------------------------------------------ |
| d         | 删除选中的部分文本。                                         |
| D         | 删除选中部分所在的行，和 d 不同之处在于，即使选中文本中有些字符所在的行没有都选中，删除时也会一并删除。 |
| y         | 将选中部分复制到剪贴板中。                                   |
| p（小写） | 将剪贴板中的内容粘贴到光标之后。                             |
| P（大写） | 将剪贴板中的内容粘贴到光标之前。                             |
| u（小写） | 将选中部分中的大写字符全部改为小写字符。                     |
| U（大写） | 将选中部分中的小写字符全部改为大写字符。                     |
| >         | 将选中部分右移（缩进）一个 tab 键规定的长度（CentOS 6.x 中，一个tab键默认相当于 8 个空白字符的长度）。 |
| <         | 将选中部分左移一个 tab 键规定的长度（CentOS 6.x 中，一个tab键默认相当于 8 个空白字符的长度）。 |

### 6.12 Vim多窗口编辑模式

在编辑文件时，有时需要参考另一个文件，如果在两个文件之间进行切换则比较麻烦。可以使用 Vim 同时打开两个文件，每个文件分别占用一个窗口。

例如，在査看 /etc/passwd 时需要参考 /etc/shadow，有两种办法可以实现：

1. 先使用 Vim 打开第一个文件，接着输入命 令 ":sp/etc/shadow" 水平切分窗口，然后按回车键；如果想垂直切分窗口则可以输入 ":vs/etc/shadow";
2. 可以直接执行命令"vim -o 第一个文件名 第二个文件名"，也就是 "vim -o /etc/passwd /etc/shadow"。
切换到另一个文件窗口，可以按 "Ctrl+WW" 快捷键。

如果想将一个文件的内容全部复制到另一个文件中，则可以输入命令 ":r 被复制的文件名"，即可将导入文件的全部内容复制到当前光标所在行下面。

### 6.13 Vim批量注释和自定义注释快捷键

使用 Vim 编辑 Shell 脚本，在进行调试时，需要进行多行的注释，每次都要先切换到输入模式，在行首输入注释符"#"再退回命令模式，非常麻烦。

连续行的注释其实可以用替换命令来完成。换句话说，在指定范围行加"#"注释，可以使用 ":起始行，终止行 s/^/#/g"，例如：

```shell
:1,10s/^/#/g
```

表示在第 1~10 行行首加"#"注释。"^"意为行首；"g"表示执行替换时不询问确认。如果希望每行交互询问是否执行，则可将 "g" 改为 "c"。

取消连续行注释，则可以使用 ":起始行，终止行s/^#//g"，例如：

```shell
:1,10s/^#//g
```

意为将行首的"#"替换为空，即删除。

当然，使用语言不同，注释符号或想替换的内容不同，都可以采用此方法，灵活运用即可。

添加"//"注释要稍微麻烦一些，命令格式为 ":起始行，终止行 s/^/\/\//g"。例如：

```shell
:1,5s/^/\/\//g
```

表示在第 1~5 行行首加"//"注释，因为 "/" 前面需要加转义字符 "\\"，所以写出来比较奇特。

以上方法可以解决连续行的注释问题，如果是非连续的多行就不灵了，这时我们可以定义快捷键简化操作。格式如下：

```shell
:map 快捷键 执行命令
```

如定义快捷键 "Ctrl+P" 为在行首添加 "#" 注释，可以执行 ":map^P l#<Esc>"。其中 "^P" 为定义快捷键 "Ctrl+P"。注意：必须同时按 "Ctrl+V+P" 快捷键生成 "^P" 方可有效，或先按 "Ctrl+V" 再按 "Ctrl+P" 也可以，直接输入 "^P" 是无效的。

"l#<Esc>" 就是此快捷键要触发的动作，"l" 为在光标所在行行首插入，"#" 为要输入的字符，"<Esc>" 表示退回命令模式。"<Esc>" 要逐个字符输入，不可直接按键盘上的 Esc 键。

设置成功后，直接在任意需要注释的行上按 "Ctrl+P" 快捷键，就会自动在行首加上 "#" 注释。取消此快捷键定义，输入 ":unmap^P" 即可。

我们可以延伸一下，如果想取消文件行首的快捷键，则可以设置 ":map^B 0x"，快捷键为 "Ctrl+B", "0" 表示跳到行首，"x" 表示删除光标所在处字符。

再如，有时我们写完脚本等文件，需要在末尾注释中加入自己的邮箱，则可以直接定义每次按快捷键 "Ctrl+E" 实现插入邮箱，定义方法为 ":map^E asamlee@itxdl.net<Esc>"。其中 "a" 表示在当前字符后插入，"samlee@itxdl.net" 为插入的邮箱，"<Esc>" 表示插入后返回命令模式。

所以，通过定义快捷键，我们可以把前面讲到的命令组合起来使用。

将快捷键对应的命令保存在 .vimrc 文件中，即可在每次使用 Vim 时自动调用，非常方便。

### 6.14 Vim配置文件（.vimrc）详解

Vim 配置文件分为系统配置文件和用户配置文件：

- 系统配置文件位于 Vim 的安装目录（默认路径为 /etc/.vimrc）；

- 用户配置文件位于主目录 ~/.vimrc，即通过执行 `vim ~/.vimrc` 命令即可对此配置文件进行合理修改。通常情况下，Vim 用户配置文件需要自己手动创建。

Vim 提供的环境配置参数有很多，可以在 Vim 中输入“：set all”指令来查询。

常见的可以写入.vimrc文件中的设置参数

| 设置参数                                                     | 含 义                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| :set nu<br>:set nonu                                         | 设置与取消行号。                                             |
| :syn on<br/>:syn off                                         | 是否依据语法显示相关的颜色帮助。在Vim中修改相关的配置文件或Shell脚本文件 时（如前面示例的脚本/etc/init.d/sshd)，默认会显示相应的颜色，用来帮助排错。如果觉得颜色产生了干扰，则可以取消此设置 |
| set hlsearch<br/>set nohlsearch                              | 设置是否将査找的字符串高亮显示。默认是hlsearch高亮显示       |
| set nobackup<br/>set backup                                  | 是否保存自动备份文件。默认是nobackup不自动备份。如果设定了:set backup，则会产生“文件名〜”作为备份文件 |
| set ruler<br/>set noruler                                    | 设置是否显示右下角的状态栏。默认是ruler显示                  |
| set showmode<br/>set noshowmode                              | 设置是否在左下角显示如“一INSERT--”之类的状态栏。默认是showmode显示 |
| set fileencodings=utf-8,ucs-bom,gb18030,gbk,gb2312,cp936<br/>set termencoding=utf-8<br/>set encoding=utf-8 | 设置编码格式，encoding 选项用于缓存的文本、寄存器、Vim 脚本文件等；fileencoding 选项是 Vim 写入文件时采用的编码类型；termencoding 选项表示输出到终端时采用的编码类型。 |
| set cursorline                                               | 突出显示当前行。                                             |
| set mouse=a<br/>set selection=exclusive<br/>set selectmode=mouse,key | Vim 编辑器里默认是不启用鼠标的，通过此设置即可启动鼠标。     |
| set autoindent                                               | 设置自动缩进，即每行的缩进同上一节相同。                     |
| set tabstop=4                                                | 设置 Tab 键宽度为 4 个空格。                                 |

### 6.15 在Vim中执行Linux命令

由于 Vim 编辑器中支持直接执行 Linux 命令，从而可以方便快捷地对文件完成以下操作：

1. 将一个命令的输出结果存入正在编辑的文件；
2. 将正在编辑的文件中的一些数据作为某个指定 Linux 命令的输入。

```shell
# 新建一个 demo.txt 文件，并手动输入如下内容，并将光标移动至下一行开头
http://c.biancheng.net
# 按 Esc 令 Vim 返回到命令模式，再按下!!，这时在窗口的左下角会出现:.!的提示信息，这就表明我们可以输入 Linux 命令了。例如，我们输入 date 命令，按 Enter（回车）键。窗口左下角的:.!表示操作文本的范围，其中 . 表示从光标所在行开始，! 表示后续会执行 Linux 命令，整体表示命令的执行结果将插入到光标所在行的位置，因此，如果光标所在位置处有数据，就会被命令的执行结果直接覆盖掉。
http://c.biancheng.net
Tue Sep 13 19:34:17 CST 2022
# 在此基础上，再向该文件中手动输入以下数据：
http://c.biancheng.net
Tue Sep 13 19:34:17 CST 2022
1 C语言中文网
3 c.biancheng.net
2 Linux教程
# 输出完成之后，将光标调整至第 3 行第 1 个字符的位置，然后按 Esc 使 Vim 进行命令模式，并按下!}组合键，你会看到窗口的左下角出现:.,$!的提示信息。其中 . 表示光标所在的当前行，$ 表示文件最后一行，因此和之前不同，这次选取的是文件中第 3 行及之后的所有内容。
# 我们使用 sort 命令对选中文本按照第 1 列进行降序排序，执行命令如下：:.,$!sort -nr -k1，按 Enter（回车）键。
http://c.biancheng.net
Tue Nov 12 07:20:49 PST 2017
3 c.biancheng.net
2 Linux教程
1 C语言中文网
```

Vim执行Linux命令的方式

| 格式        | 功能                                                         |
| ----------- | ------------------------------------------------------------ |
| :!命令      | 直接运行一个 Linux 命令，运行完毕之后，即可返回到 Vim 中。   |
| :w!命令     | 将 Vim 中所有的文本内容作为指定命令的输入。但命令的执行结果不会写入到当前文件中。 |
| :r!命令     | 将命令执行的结果写入到当前 Vim 中，例如 :!ls 表示将 ls 的执行结果输入到 Vim 中。 |
| :nr!命令    | 其中 n 为数字，表示将命令的执行结果写入到 Vim 第 n 行的位置。例如，:3r!date 表示将 date 命令的执行结果写入到第 3 行文本处。 |
| :n,m!命令   | 其中 n 表示起始行号，m为结束行号，功能是将 Vim 中指定的部分文本作为某个命令的输入，同时将命令的输出也插入到当前指定的位置。 |
| :n,m w!命令 | 其中 n 表示起始行号，m为结束行号，其功能是 Vim 中指定的部分文本作为某个命令的输入，但命令的执行结果不会写入到文件中。 |
| !!date      | 向 Vim 中插入当前时间。                                      |

## 7. 解压缩

### 7.1 基本命令

这五个是独立命令，需要用到其中一个，可以和别的命令连用但只能用其中一个。

```shell
tar
-c:建立压缩档案
-x:解压
-t:查看内容
-r:向压缩归档文件末尾追加文件
-A:向压缩归档文件追加文件
-u:更新原压缩包中的文件
```

下面的参数是根据需要在压缩或解压档案时可选的。

```shell
-z: 有gzip属性的
-J: 有xz属性的
-j: 有bz2属性的
-Z: 有compress属性的

-v: 显示所有过程
-O: 将文件解开到标准输出
```

下面的参数-f是必须的

```shell
-f: 使用档案名字，切记，这个参数是最后一个参数，后面只能接档案名。
```
例子解析：

```shell
# 这条命令是将所有.jpg的文件打成一个名为all.tar的包。-c是表示产生新的包，-f指定包的文件名。
tar -cf all.tar *.jpg

# 这条命令是将所有.gif的文件增加到all.tar的包里面去。-r是表示增加文件的意思。
tar -rf all.tar *.gif

# 这条命令是更新原来tar包all.tar中logo.gif文件，-r是表示更新文件的意思。
tar -uf all.tar logo.gif

# 这条命令是列出all.tar包中所有文件，-t是列出文件的意思。
tar -tf all.tar *.gif

# 这条命令是解出all.tar包中所有文件，-x是解开的意思。
tar -xf all.tar
```
### 7.2 分卷压缩和解压

```shell
split [--help][--version][-d][-<行数>][-b <字节>][-C <字节>][-l <行数>][要切割的文件][输出文件名]
参数说明：
-<行数> : 指定每多少行切成一个小文件
-b <字节> : 指定每多少字节切成一个小文件
--help : 在线帮助
--version : 显示版本信息
-d : 使用数字后缀而不是字母
-C <字节> : 与参数"-b"相似，但是在切 割时将尽量维持每行的完整性
[输出文件名] : 设置切割后文件的前置文件名， split会自动在前置文件名后再加上编号
```

例子：

```shell
# 分两步，弄清楚原理
tar -czvf cmake.tar.gz cmake/
split -b 100m cmake.tar.gz cmake.tar.gz
# 一步，实用
tar -czvf - cmake/ | split -b 100m - cmake.tar.gz
# 解压到cmake2文件夹
cat cmake.tar.gza* | tar -xzv -C cmake2/
```

### 7.3 各种压缩格式总结

#### 7.3.1 打包

```shell
# .tar只打包不压缩
tar -cvf jpg.tar *.jpg
tar -xvf jpg.tar
```

#### 7.3.2 gzip（.tar.gz/.tgz/.gz）

```shell
# 分两步，弄清楚原理
tar -cvf cmake.tar cmake/
gzip cmake.tar # cmake.tar被删除，生成cmake.tar.gz
# 一步，实用
tar -czvf jpg.tar.gz *.jpg
# 解压
tar -xzvf jpg.tar.gz -C /home/aaa/
```

gzip 命令只能用来压缩文件，不能压缩目录，即便指定了目录，也只能压缩目录内的所有文件。在 Linux 中，打包和压缩是分开处理的。而 gzip 命令只会压缩，不能打包，所以才会出现没有打包目录，而只把目录下的文件进行压缩的情况。

```shell
gzip [选项] 源文件
-c	将压缩数据输出到标准输出中，并保留源文件。
-d	对压缩文件进行解压缩。
-r	递归压缩指定目录下以及子目录下的所有文件。
-v	对于每个压缩和解压缩的文件，显示相应的文件名和压缩比。
-l	对每一个压缩文件，显示以下字段：
压缩文件的大小；
未压缩文件的大小；
压缩比；
未压缩文件的名称。
-数字	用于指定压缩等级，-1 压缩等级最低，压缩比最差；-9 压缩比最高。默认压缩比是 -6。
gunzip [选项] 文件
gzip -d [选项] 文件
-r	递归处理，解压缩指定目录下以及子目录下的所有文件。
-c	把解压缩后的文件输出到标准输出设备。
-f	强制解压缩文件，不理会文件是否已存在等情况。
-l	列出压缩文件内容。
-v	显示命令执行过程。
-t	测试压缩文件是否正常，但不对其做解压缩操作。

# 会删除源文件，一个文件生成一个压缩包而不是压缩到一个包
gzip -v *.jpg
# 不会删除源文件，压缩到一个包
gzip -cv *.jpg > jpg.gz
# 解压，用gzip -d或者gunzip
gunzip jpg.gz
gunzip -c jpg.gz > filename
# 纯文本文件，zcat可在不解压的情况下查看内容
zcat jpg.gz
```

#### 7.3.3 xz（.tar.xz）

```shell
# 分两步，弄清楚原理
tar -cvf cmake.tar cmake/
xz -z cmake.tar
# 一步，实用
tar -cJvf cmake.tar.xz cmake/

# 分两步，弄清楚原理
xz -d cmake.tar.xz
tar -xvf cmake.tar -C cmake2/
# 一步，实用
tar -xJvf cmake.tar.xz
```

#### 7.3.4 bz2（.tar.bz2）

```shell
tar -cjvf jpg.tar.bz2 *.jpg
tar -xjvf jpg.tar.bz2
```

bzip2 命令同 gzip 命令类似，只能对文件进行压缩（或解压缩），对于目录只能压缩（或解压缩）该目录及子目录下的所有文件。注意，gzip 只是不会打包目录，但是如果使用“-r”选项，则可以分别压缩目录下的每个文件；而 bzip2 命令则根本不支持压缩目录，也没有“-r”选项。

从理论上来讲，".bz2"格式的算法更先进、压缩比更好；而 ".gz"格式相对来讲的时间更快。

```shell
bzip2 [选项] 源文件
-d	执行解压缩，此时该选项后的源文件应为标记有 .bz2 后缀的压缩包文件。
-k	bzip2 在压缩或解压缩任务完成后，会删除原始文件，若要保留原始文件，可使用此选项。
-f	bzip2 在压缩或解压缩时，若输出文件与现有文件同名，默认不会覆盖现有文件，若使用此选项，则会强制覆盖现有文件。
-t	测试压缩包文件的完整性。
-v	压缩或解压缩文件时，显示详细信息。
-数字	这个参数和 gzip 命令的作用一样，用于指定压缩等级，-1 压缩等级最低，压缩比最差；-9 压缩比最高
bzip2 -d [选项] 源文件
bunzip2 [选项] 源文件
-k	解压缩后，默认会删除原来的压缩文件。若要保留压缩文件，需使用此参数。
-f	解压缩时，若输出的文件与现有文件同名时，默认不会覆盖现有的文件。若要覆盖，可使用此选项。
-v	显示命令执行过程。
-L	列出压缩文件内容。

# 此压缩命令会在压缩的同时删除源文件。
bzip2 anaconda-ks.cfg
# 压缩的同时保留源文件。
bzip2 -k install.log.syslog
# 纯文本文件，bzcat可在不解压的情况下查看内容
bzcat jpg.gz
```

#### 7.3.5 compress（.Z）

```shell
tar -cZvf jpg.tar.Z *.jpg
tar -xZvf jpg.tar.Z

# *.Z 用uncompress解压
```

#### 7.3.6 rar（.rar）

```shell
# rar格式的压缩，需要先下载rar for Linux，与windows通用
rar a jpg.rar *.jpg
unrar e jpg.rar
```

#### 7.3.7 zip（.zip）

```shell
# zip格式的压缩，需要先下载zip for Linux，与windows通用
zip -qr jpg.zip *.jpg
unzip jpg.zip

zip [选项] 压缩包名 源文件或源目录列表
-r	递归压缩目录，及将制定目录下的所有文件以及子目录全部压缩。
-m	将文件压缩之后，删除原始文件，相当于把文件移到压缩文件中。
-v	显示详细的压缩过程信息。
-q	在压缩的时候不显示命令的执行过程。
-压缩级别	压缩级别是从 1~9 的数字，-1 代表压缩速度更快，-9 代表压缩效果更好。
-u	更新压缩文件，即往压缩文件中添加新文件。
unzip [选项] 压缩包名
-d 目录名	将压缩文件解压到指定目录下。
-n	解压时并不覆盖已经存在的文件。
-o	解压时覆盖已经存在的文件，并且无需用户确认。
-v	查看压缩文件的详细信息，包括压缩文件中包含的文件大小、文件名以及压缩比等，但并不做解压操作。
-t	测试压缩文件有无损坏，但并不解压。
-x 文件列表	解压文件，但不包含文件列表中指定的文件。
```

## 8.  查询系统信息

```shell
# 查询发行版本
$ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 20.04.5 LTS
Release:	20.04
Codename:	focal
```



```shell
# 查询所有
$ uname -a
Linux zuoc 5.10.102.1-microsoft-standard-WSL2 #1 SMP Wed Mar 2 00:30:59 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux

# 查看内核名称
# 当前系统使用的是Linux内核，内核可以分为四大类：单内核、微内核、混合内核、外内核，Linux属于单内核。
uname -s[--sysname]
Linux

# 查看内核发行版本
uname -r[--kernel-release]
4.4.0-62-generic

# 查看内核发行版本
# SMP：对称多处理机，表示内核支持多核、多处理器
# Wed Jan 18 14:10:15 UTC 2017：内核的编译时间（build date）为（2017/01/18 14:10:15）
uname -v[----kernel-version]
#83-Ubuntu SMP Wed Jan 18 14:10:15 UTC 2017

# 查看硬件名称，可以通过此属性判断操作系统的架构
# x86-64：64位系统
# ix86：32位系统（x表示3、4、5、6）
uname -m[--machine]
x86_64

# 查看硬件平台，表示构建内核的架构
uname -i[--hardware-platform]
x86_64

# 查看处理器类型，表示该机器处理器的类型（CPU）
uname -p
x86_64

# 查看操作系统类型
uname -o[--operating-system]
GUN/Linux

# 查看主机名
uname -n[--nodename]
zuoc
```

## 9. man

在线手册：http://man.he.net/     http://www.tin.org/bin/man.cgi    https://man7.org/linux/man-pages/man2/eventfd.2.html

man 命令是 Linux 下的帮助指令，通过 man 指令可以查看 Linux 中的指令帮助、配置文件帮助和编程帮助等信息。

```shell
man [-adfhktwW] [section] [-M path] [-P pager] [-S list] [-m system] [-p string] title..
# 数字：指定从哪本 man 手册中搜索帮助；
# 关键字：指定要搜索帮助的关键字。

# 查询自己
man man
```

1. 是普通的命令
2. 是系统调用，如open、write之类的
3. 是库函数，如printf、fread
4. 是特殊文件，也就是/dev下的各种设备文件
5. 是指文件的格式，比如passwd，就会说明这个文件中各个字段的含义
6. 是给游戏留的，由各个游戏自己定义
7. 是附件还有一些变量，比如向environ这种全局变量在这里就有说明
8. 是系统管理用的命令,这些命令只能由root使用,如ifconfig
9. 是非标准

man 命令中常用按键以及用途

| 按键      | 用途                               |
| :-------- | :--------------------------------- |
| 空格键    | 向下翻一页                         |
| PaGe down | 向下翻一页                         |
| PaGe up   | 向上翻一页                         |
| home      | 直接前往首页                       |
| end       | 直接前往尾页                       |
| /         | 从上至下搜索某个关键词，如“/linux” |
| ？        | 从下至上搜索某个关键词，如“?linux” |
| n         | 定位到下一个搜索到的关键词         |
| N         | 定位到上一个搜索到的关键词         |
| q         | 退出帮助文档                       |

### 9.1 参数

| **man命令常用参数** |                                                              |
| ------------------- | ------------------------------------------------------------ |
| -a                  | 显示所有匹配项                                               |
| -d                  | 显示man查照手册文件时候，搜索路径信息,不显示手册页内容       |
| -D                  | 同-d,显示手册页内容                                          |
| -f                  | 同命令whatis ，将在whatis数据库查找以关键字开同的帮助索引信息 |
| -h                  | 显示帮助信息                                                 |
| -k                  | 同命令apropos 将搜索whatis数据库，模糊查找关键字             |
| -S list             | 指定搜索的领域及顺序 如：-S 1:1p httpd 将搜索man1然后 man1p目录 |
| -t                  | 使用troff 命令格式化输出手册页 默认：groff输出格式页         |
| -w                  | 不带搜索title 打印manpath变量 带title关键字 打印找到手册文件路径,默认搜索一个文件后停止 |
| -W                  | 同-w                                                         |
| section             | 搜索领域【限定手册类型】默认查找所有手册                     |

| **man命令其它参数** |                                                       |
| ------------------- | ----------------------------------------------------- |
| -c                  | 显示使用 cat 命令的手册信息                           |
| -C                  | 指定man 命令搜索配置文件 默认是man.config             |
| -K                  | 搜索一个字符串在所有手册页中,速度很慢                 |
| -M                  | 指定搜索手册的路径                                    |
| -P pro              | 使用程序pro显示手册页面 默认是less                    |
| -B pro              | 使用pro程序显示HTML手册页 默认是less                  |
| -H pro              | 使用pro程序读取HTML手册，用txt格式显示，默认是cat     |
| -p str              | 指定通过groff格式化手册之前，先通过其它程序格式化手册 |

### 9.2 例子

一般遇到一个不是很熟悉命令可以先通过：

man -k command1 查询所有类似帮助文件信息，这样输出最多也可以用：

man -f command1 查询以command1开头所有相关帮助信息列表 如果发现有类似：command1 (5)

man 5 command1 通过直接定位5获得帮助信息

```shell
# 显示passwd帮助文件路径，passwd.1通过名称知道这个是passwd命令帮助手册，那它的其它命令的呢？
man -w passwd
/usr/share/man/man1/passwd.1.gz

# 加入-a获得所有帮助手册文件地址,默认只会查找一个
man -aw passwd
/usr/share/man/man1/passwd.1.gz
/usr/share/man/man1/passwd.1ssl.gz
/usr/share/man/man5/passwd.5.gz

# 查看man5的passwd
man 5 passwd

# 在领域类型是：1:2 范围内查找手册，对应目录分别是man1、man2
man -S 1:2 passwd

# 在whatis数据库（有所有网站man帮助以及cat,doc帮助信息索引）中查询，文件标题以：http开头信息的文档
# 中间的(8) 对应我们可以用：man 8 httpd 调用，对于显示(rpm)实际上显示有个httpd帮助信息，是属于一个httpd rpm安装包，通过man rpm httpd查看不了。可以通过rpm -ql httpd 查找安装包
man -f httpd
httpd                (8)  - Apache Hypertext Transfer Protocol Server
httpd               (rpm) - Apache HTTP Server
httpd-devel         (rpm) - Development tools for the Apache HTTP server.

# 在whatis数据库中，查询包含httpd所有帮助手册，以及安装包. 可以通过：rpm -ql lighttpd
man -k httpd
CGI::Carp            (3pm)  - CGI routines for writing to the HTTPD (or other) error log
httpd                (8)  - Apache Hypertext Transfer Protocol Server
httpd               (rpm) - Apache HTTP Server
httpd-devel         (rpm) - Development tools for the Apache HTTP server.
httpd_selinux        (8)  - Security Enhanced Linux Policy for the httpd daemon
lighttpd             (1)  - a fast, secure and flexible webserver
lighttpd            (rpm) - Lightning fast webserver with light system requirements
lighttpd-fastcgi    (rpm) - FastCGI module and spawning helper for lighttpd and PHP configuration
ncsa_auth            (8)  - NCSA httpd-style password file authentication helper for Squid

# 其实这个包刚好是：lighttpd             (1)  - a fast, secure and flexible webserver 帮助手册
rpm -ql lighttpd | grep gz
/usr/share/man/man1/lighttpd.1.gz

# 显示man 命令查找手册的路径
man -w
/usr/kerberos/man:/usr/local/share/man:/usr/share/man/zh_CN:/usr/share/man:/usr/local/man
```

# 二. SHELL

```shell
# 后续所有的bash命令的返回code如果不是0，那么脚本立即退出，后续的脚本将不会得到执行的机会;
set -e
# 这个是默认的状态，表示就算后续的命令如果返回值不是0，那么脚本依然向下执行;
set +e
```

