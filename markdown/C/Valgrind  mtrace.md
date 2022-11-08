[toc]

官网：https://valgrind.org/downloads/current.html

# 一. 快速入门

安装

```shell
# 简易安装
sudo apt install valgrind

# 自编译最新版，官网下载后解压
tar -xjvf valgrind-3.19.0.tar.bz2
cd valgrind-3.19.0

# 配置生成makefile
./configure

# 编译和安装
make -j4
sudo make install

# 查看版本号
valgrind --version
```

用-g编译程序以包含调试信息，这样Memcheck的错误消息就会包含精确的行号。使用-O0也是一个好主意，如果您可以忍受编译太慢的话。使用-O1错误消息中的行号可能不准确，尽管一般来说在-O1编译的代码上运行Memcheck工作得相当好，而且与运行-O0相比，速度提高相当显著。不建议使用-O2或以上参数，因为Memcheck偶尔会报告不存在的未初始化值错误。

您的程序将比正常运行慢得多（例如20到30次），并且使用更多的内存。Memcheck 将发出有关它检测到的内存错误和泄漏的消息。

包含内存错误和内存泄漏的例子：

```c
#include <stdlib.h>

void f(void)
{
    int* x = malloc(10 * sizeof(int));
    x[10] = 0;       // problem 1: heap block overrun
}                    // problem 2: memory leak -- x not freed

int main(void)
{
    f();
    return 0;
}
```

```shell
gcc -g -O0 example.c -o example

# Memcheck是默认工具，--leak-check选项打开详细的内存泄漏检测器。
valgrind --leak-check=yes ./example
==13871== Memcheck, a memory error detector
==13871== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.
==13871== Using Valgrind-3.19.0 and LibVEX; rerun with -h for copyright info
==13871== Command: ./example
==13871== 
test valgrind
==13871== Invalid write of size 4
==13871==    at 0x10918B: f (example.c:7)
==13871==    by 0x1091AC: main (example.c:13)
==13871==  Address 0x4a4d4a8 is 0 bytes after a block of size 40 alloc'd
==13871==    at 0x483C855: malloc (vg_replace_malloc.c:381)
==13871==    by 0x10917E: f (example.c:6)
==13871==    by 0x1091AC: main (example.c:13)
==13871== 
==13871== 
==13871== HEAP SUMMARY:
==13871==     in use at exit: 40 bytes in 1 blocks
==13871==   total heap usage: 2 allocs, 1 frees, 1,064 bytes allocated
==13871== 
==13871== 40 bytes in 1 blocks are definitely lost in loss record 1 of 1
==13871==    at 0x483C855: malloc (vg_replace_malloc.c:381)
==13871==    by 0x10917E: f (example.c:6)
==13871==    by 0x1091AC: main (example.c:13)
==13871== 
==13871== LEAK SUMMARY:
==13871==    definitely lost: 40 bytes in 1 blocks
==13871==    indirectly lost: 0 bytes in 0 blocks
==13871==      possibly lost: 0 bytes in 0 blocks
==13871==    still reachable: 0 bytes in 0 blocks
==13871==         suppressed: 0 bytes in 0 blocks
==13871== 
==13871== For lists of detected and suppressed errors, rerun with: -s
==13871== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```

- 19182 是进程 ID;
- 第一行（"Invalid write..."）告诉您它是什么类型的错误。在这里，由于堆块溢出，程序写入了一些它不应该拥有的内存。
- 第一行下面是一个堆栈跟踪，告诉您问题发生在哪里。堆栈跟踪可能变得相当大，并且令人困惑，特别是如果您正在使用c++ STL。从下往上读会有帮助。如果堆栈跟踪不够大，则使用--num-callers选项使其更大。
- 代码地址（例如0x483C855）通常并不重要，但偶尔对于追踪更奇怪的错误至关重要。
- 有些错误消息有第二个组件，用于描述所涉及的内存地址。这个例子表明写内存刚刚超过example.c的第6行malloc()分配的块的末尾。
- 按报告的顺序修复错误是值得的，因为以后的错误可能是由较早的错误引起的。如果不这样做，Memcheck查找原因变得困难。
- 堆栈跟踪会告诉您泄漏内存的分配位置。不幸的是，记忆检查无法告诉您内存泄漏的原因。

内存泄露的类型

```tex
definitely lost(绝对丢失了):你的程序正在泄漏内存——修复它!
probably lost(可能丢失了):你的程序正在泄漏内存，除非你对指针做了一些有趣的事情(比如将它们移动到堆内存的中间)。
```

Memcheck还报告了未初始化值的使用，最常见的消息是“Conditional jump or move depends on uninitialised value(s)”。很难确定这些错误的根本原因。尝试使用--track-origins=yes来获取额外的信息。这使得Memcheck运行速度变慢，但你得到的额外信息通常节省了很多时间来找出未初始化值的来源。

Memcheck并不完美，它偶尔会误报，有抑制误报的机制。但是，它在99%的情况下通常是正确的，因此您应该注意不要忽略它的错误消息。毕竟，您不会忽略编译器产生的警告消息，对吗？如果Memcheck报告了库代码中无法更改的错误，抑制机制也很有用。默认的抑制集隐藏了许多此类信息，但您可能会遇到更多。

Memcheck无法检测程序的所有内存错误。例如，它不能检测对静态分配的数组或栈上的数组的超出范围的读写。但是它应该检测到许多可能使你的程序崩溃的错误（例如段错误）。

尽量使你的程序非常干净，使Memcheck报告没有错误。一旦你达到了这种状态，就更容易看出什么时候对程序的修改会导致Memcheck报告新的错误。使用Memcheck几年的经验表明，即使是大型程序也可以运行Memcheck-clean。例如，KDE、OpenOffice.org和Firefox的很大一部分都是Memcheck-clean的，或者非常接近Memcheck-clean。

valgrind报如下错误解决办法：

因为libc或ld.so库进行过strip操作。直接安装一个debug版本的库就可以了：`sudo apt-get install libc6-dbg`。

```
valgrind:  Fatal error at startup: a function redirection
valgrind:  which is mandatory for this platform-tool combination
valgrind:  cannot be set up.  Details of the redirection are:
valgrind:  
valgrind:  A must-be-redirected function
valgrind:  whose name matches the pattern:      strlen
valgrind:  in an object with soname matching:   ld-linux-x86-64.so.2
valgrind:  was not found whilst processing
valgrind:  symbols from the object with soname: ld-linux-x86-64.so.2
valgrind:  
valgrind:  Possible fixes: (1, short term): install glibc's debuginfo
valgrind:  package on this machine.  (2, longer term): ask the packagers
valgrind:  for your Linux distribution to please in future ship a non-
valgrind:  stripped ld.so (or whatever the dynamic linker .so is called)
valgrind:  that exports the above-named function using the standard
valgrind:  calling conventions for this platform.  The package you need
valgrind:  to install for fix (1) is called
valgrind:  
valgrind:    On Debian, Ubuntu:                 libc6-dbg
valgrind:    On SuSE, openSuSE, Fedora, RHEL:   glibc-debuginfo
valgrind:  
valgrind:  Note that if you are debugging a 32 bit process on a
valgrind:  64 bit system, you will need a corresponding 32 bit debuginfo
valgrind:  package (e.g. libc6-dbg:i386).
valgrind:  
valgrind:  Cannot continue -- exiting now.  Sorry.
```

# 二. 用户手册

## 1. 引言

1. ****Memcheck****是一个内存错误检测器。它可以帮助您使程序（尤其是用 C 语言编写的程序和C++程序）更加正确。
2. ****Cachegrind****是一个缓存和分支预测探查器。它可以帮助您使程序运行得更快。
3. ****Callgrind****是一个调用图生成缓存分析器。它与Cachegrind有一些重叠，但也收集了一些Cachegrind没有的信息。
4. ****Helgrind****是一个线程检测器。它可以帮助您使多线程程序更加正确。
5. **DRD** 也是一个线程错误检测器。它与Helgrind相似，但使用不同的分析技术，因此可能会发现不同的问题。
6. ****Massif****是一个堆分析器。它可以帮助您使程序使用更少的内存。
7. **DHAT** 是一种不同类型的堆分析器。它可以帮助您了解块生存期、块利用率和布局效率低下的问题。
8. **BBV**是一个实验性的SimPoint基本块向量生成器。它对从事计算机体系结构研究和开发的人很有用。

## 2. 使用和理解valgrind核心

### 2.1 valgrind如何处理你的程序

valgrind被设计为尽可能不侵入。它直接与现有的可执行文件一起使用。您无需重新编译、重新链接或以其他方式修改要检查的程序。

无论使用哪种工具，Valgrind都会在程序启动之前控制程序。从可执行文件和关联的库中读取调试信息，以便在适当的时候，可以根据源代码位置来表述错误消息和其他输出。

然后，您的程序将在Valgrind内核提供的synthetic CPU上运行。当新代码首次执行时，核心会将代码传递给所选工具。该工具将自己的检测代码添加到此代码中，并将结果交还给核心，核心负责协调此检测代码的持续执行。

添加的检测代码量因工具而异。最大规模，Memcheck添加代码来检查每个内存访问和每个值计算，使其运行速度比本机慢10-50倍。最小规模，称为Nullgind的最小工具根本没有添加任何仪器，并且总共“仅”导致大约4倍的减速。

瓦尔格林模拟程序执行的每一条指令。因此，活动工具不仅会检查或分析应用程序中的代码，还会检查所有支持的动态链接库（包括 C 库、图形库等）中的代码。

如果您正在使用错误检测工具，Valgrind可能会检测系统库中的错误，例如您必须使用的GNU C或X11库。您可能对这些错误不感兴趣，因为您可能无法控制这些代码。因此，Valgrind允许您选择性地抑制错误，通过在Valgrind启动时读取的抑制文件中记录它们。构建机制选择默认的抑制，为在您的机器上检测到的操作系统和库提供合理的行为。为了更容易地编写抑制，可以使用--gen-suppressions=yes选项。这告诉Valgrind为每个报告的错误打印一个抑制，然后您可以将其复制到一个抑制文件中。

不同的错误检查工具报告不同类型的错误。因此，抑制机制允许您说明每种抑制应用于哪个或哪些工具。

```shell
valgrind [valgrind-options] your-prog [your-prog-options]

# memcheck是默认的，可以忽略
valgrind --tool=memcheck ls -l
```

### 2.2 入门

如果使用c++，您可能会考虑的另一个选项是-fno-inline。这样可以更容易地看到函数调用链，这有助于减少在大型c++应用程序中导航时的混乱。例如，使用这个选项时，使用Memcheck调试OpenOffice.org会更容易一些。您不必这样做，但这样做可以帮助Valgrind生成更准确、更少混淆的错误报告。如果您打算用GNU GDB或其他调试器调试您的程序，那么您很可能已经这样设置了。或者，Valgrind选项--read-inline-info=yes指示Valgrind读取描述内联信息的调试信息。这样，函数调用链就会正确显示出来，即使你的应用程序是用内联编译的。

如果你打算使用Memcheck:在很少的情况下，编译器优化(在-O2或以上，有时-O1)已经观察到生成的代码会欺骗Memcheck错误报告未初始化值错误，或遗漏未初始化值错误。我们已经仔细研究了修复这个问题的细节，不幸的是，这样做的结果是会进一步显著地降低这个已经很慢的工具。因此，最好的解决方案是完全关闭优化。由于这常常使程序变得难以管理地慢，一个合理的折衷方法是使用-O。这可以让你获得更高优化级别的大部分好处，同时保持相对较小的几率从Memcheck误报或误报。此外，你应该用-Wall编译你的代码，通常使用-Wall也是一个好主意，因为它可以识别一些或所有的问题，在更高的优化级别Valgrind可能未检测到错误。所有其他工具(据我们所知)都不受优化级别的影响，对于像Cachegrind这样的分析工具，最好在它正常的优化级别编译你的程序。

当您准备好滚动时，按上面所述运行Valgrind。注意，您应该在这里运行真正的(机器代码)可执行文件。如果您的应用程序是由shell或Perl脚本启动的，那么您需要修改它，以便在真正的可执行文件上调用Valgrind。直接在Valgrind下运行这样的脚本将导致您得到与/bin/sh、/usr/bin/perl或任何您正在使用的解释器相关的错误报告。这可能不是你想要的，可能会让人困惑。您可以通过提供选项--trace-children=yes来强制解决这个问题，但仍然可能造成混淆。

### 2.3 valgrind输出commentary

```tex
==12345== some-message-from-Valgrind
```

默认情况下，Valgrind工具只将必要的消息写入commentary，以避免您被次要的信息淹没。如果你想要更多关于正在发生的事情的信息，重新运行，将-v选项传递给Valgrind。第二个-v给出了更多的细节。

您可以将commentary定向到三个不同的位置：

1. 默认值：将其发送到文件描述符，默认情况下为 2 （stderr）。因此，如果您不给核心任何选项，它将为标准错误流编写commentary。如果要将其发送到其他文件描述符（例如数字 9），可以指定 `--log-fd=9`。

   这是最简单和最常见的安排，但是当整个流程树都期望特定的文件描述符（特别是 stdin/stdout/stderr）可供自己使用时，可能会导致问题。

2. 一个侵入性较小的选项是将commentary写入文件，您可以通过--log-file=filename指定该文件。有一些特殊的格式说明符，可用于在日志文件名中使用进程ID或环境变量名。如果您的程序调用多个进程(特别是MPI程序)，这些是有用的/必要的。

3. 干扰最少的选项是将commentary发送到网络套接字。套接字被指定为IP地址和端口号对，就像这样:--log-socket=192.168.0.1:12345，如果您想将输出发送到主机IP 192.168.0.1端口12345(注意:我们不知道12345是否是一个预先存在的重要端口)。您也可以省略端口号:--log-socket=192.168.0.1，在这种情况下使用默认端口号1500。这个默认值是由源文件中的常量VG_CLO_DEFAULT_LOGPORT定义的。

   请注意，不幸的是，您必须在此处使用IP地址，而不是主机名。

   如果在另一端没有监听设备，那么写入网络套接字是没有意义的。我们提供了一个简单的侦听程序valgrind-listener，它接受指定端口上的连接并复制发送到stdout的任何内容。也许有人会告诉我们这是一个可怕的安全风险。随着时间的推移，人们可能会写出更复杂的监听器。

   `valgrind-listener`可以接受来自多达 50 个进程的同时连接。在每行输出的前面，它以圆括号的形式打印当前活动连接数。

   `valgrind-listener`接受三个命令行选项：

   - `-e --exit-at-zero`

     当连接的进程数回落到零时，退出。没有这个，它将永远运行，也就是说，直到你发送它 Control-C。

   - `--max-connect=INTEGER`

     默认情况下，侦听器最多可以连接到 50 个进程。有时，这个数字太小了。使用此选项可提供不同的限制，例如：`--max-connect=100`

   - `portnumber`

     更改它监听的默认端口(1500)。端口的取值范围为1024 ~ 65535。同样的限制也适用于由--log-socket指定到Valgrind本身的端口号。

   如果Valgrinded进程由于任何原因（侦听器未运行，无效或无法访问的主机或端口等）无法连接到侦听器，则Valgrind切换回将commentary写入stderr。这同样适用于任何失去与侦听器的已建立连接的进程。换句话说，杀死侦听器不会杀死向其发送数据的进程。

以下是关于commentary与工具分析输出之间关系的重要一点。commentary包含Valgrind核心和所选工具的信息。如果该工具报告错误，它将向commentary报告错误。但是，如果该工具执行分析，则配置文件数据将写入某种类型的文件中，具体取决于工具，并且与--log-*选项的生效无关。commentary旨在成为一个低带宽，人类可读的频道。另一方面，分析数据通常是大量的，如果没有进一步的处理就没有意义，这就是我们选择这种安排的原因。

### 2.4 错误报告

```tex
==25832== Invalid read of size 4
==25832==    at 0x8048724: BandMatrix::ReSize(int, int, int) (bogon.cpp:45)
==25832==    by 0x80487AF: main (bogon.cpp:66)
==25832==  Address 0xBFFFF74C is not stack'd, malloc'd or free'd
```

这条消息说程序对地址0xBFFFF74C进行了非法的4字节读取，据Memcheck所知，这不是一个有效的堆栈地址，也不对应任何当前堆块或最近释放的堆块。读取发生在bogon.cpp的第45行，从相同文件的第66行调用，等等。对于与已识别(当前或已释放)堆块相关的错误，例如读取已释放的内存，Valgrind不仅会报告错误发生的位置，还会报告相关堆块的分配/释放位置。

Valgrind记得所有的错误报告。当检测到错误时，将其与旧报告进行比较，以确定它是否重复。如果是，则记录错误，但不会发出进一步的注释。这避免了您被无数重复的错误报告淹没。

如果想知道每个错误发生了多少次，可以使用-v选项运行。当执行完成时，将打印出所有报告，并根据它们的出现次数进行排序。这样就很容易看出哪些错误发生得最频繁。

在相关操作实际发生之前报告错误。例如，如果您正在使用Memcheck，并且您的程序试图从地址0读取，Memcheck将发出这样的消息，然后您的程序可能会因分段错误而死亡。

通常，您应该尝试按照错误报告的顺序修复错误。不这样做可能会让人困惑。例如，一个程序将未初始化的值复制到多个内存位置，然后使用它们，当在Memcheck上运行时，将生成一些错误消息。第一个这样的错误消息很可能为问题的根本原因提供最直接的线索。

检测重复错误的过程是非常昂贵的，如果程序产生大量的错误，则会造成很大的性能开销。为了避免严重的问题，Valgrind将在看到1000个不同的错误或总共看到1000万个错误后停止收集错误。在这种情况下，你最好停止你的程序并修复它，因为Valgrind在此之后不会告诉你任何其他有用的东西。注意，1,000/10,000,000的限制适用于删除抑制错误之后。这些限制在m_errormgr.c中定义，如果需要可以增加。

为了避免这种限制，你可以使用--error-limit=no选项。那么Valgrind总是会显示错误，不管有多少。谨慎使用此选项，因为它可能会对性能产生不良影响。

### 2.5 抑制错误

错误检查工具在系统库中检测大量的问题，例如在操作系统中预装的C库。您无法轻松地修复这些错误，但您不希望看到这些错误(是的，有很多错误!)所以Valgrind在启动时读取要抑制的错误列表。在构建系统时，./configure脚本会创建一个默认的抑制文件。

您可以在空闲时修改和添加到抑制文件中，或者更好的方法是编写自己的文件。支持多个抑制文件。如果你的部分项目包含了你不能或不想修复的错误，但你又不想不断被提醒这些错误，那么这个方法就很有用了。

注意:到目前为止，添加抑制的最简单的方法是使用核心命令行选项中描述的--gen-suppressions=yes选项。这将自动生成抑制。但是，为了获得最好的结果，您可能需要手动编辑--gen-suppressions=yes的输出，在这种情况下，最好阅读本节。

要抑制的每个错误都进行了非常具体的描述，以最大程度地减少抑制指令无意中抑制一堆您确实希望看到的类似错误的可能性。抑制机制旨在允许精确而灵活地指定要抑制的错误。

如果使用-v选项，在执行结束时，Valgrind将为每个使用的抑制输出一行，给出它被使用的次数、名称和定义抑制的文件名和行号。根据抑制类型的不同，文件名和行号后面可选地跟着附加信息(例如Memcheck泄漏抑制的块数和字节数)。下面是`valgrind -v --tool=memcheck ls -l`所使用的抑制:

```tex
--1610-- used_suppression:      2 dl-hack3-cond-1 /usr/lib/valgrind/default.supp:1234
--1610-- used_suppression:      2 glibc-2.5.x-on-SUSE-10.2-(PPC)-2a /usr/lib/valgrind/default.supp:1234
```

允许多个抑制文件。Valgrind从$PREFIX/lib/ Valgrind /default加载抑制模式，除非--default-suppressions=no已指定。通过指定--suppressions=/path/to/file，可以要求从其他文件添加抑制。供给一次或多次。

如果您想了解更多关于抑制的信息，请在阅读以下文档的同时查看一个现有的抑制文件。源码发行版中的glibc-2.3.supp文件提供了一些很好的例子。

屏蔽文件中的空白行和注释行将被忽略。注释行由0个或多个空格后接#字符，再接一些文本组成。

每种抑制都有以下组成部分:

- 第一行：它的名字。这仅仅为抑制提供了一个方便的名称，在程序完成时打印出来的已用抑制的摘要中引用了它。名称是什么并不重要;任何标识字符串都可以。

- 第二行：抑制所针对的工具的名称（如果有多个，则以逗号分隔），以及抑制本身的名称，用冒号分隔（注意：不允许有空格），例如：`tool_name1,tool_name2:suppression_name`

  回想一下，Valgrind是一个模块化系统，其中不同的仪器工具可以在程序运行时观察程序。由于不同的工具检测不同类型的错误，因此有必要说明抑制对哪个（些）工具有意义。

  如果工具不理解指向它的任何抑制，工具将在启动时抱怨。工具会忽略不针对它们的抑制。因此，将所有工具的抑制放入同一个抑制文件中是非常实用的。

- 下一行:少数抑制类型在第二行之后有额外的信息，例如Memcheck的' Param '抑制。

- 剩余的行:这是错误的调用上下文——导致错误的函数调用链。这一行最多可以有24条。

  位置可以是共享对象、函数或源行的名称。它们分别以' obj: '、' fun: '或' src: '开头。要匹配的函数、对象和文件名可以使用通配符' * '和' ? '。源行使用' filename[:lineNumber] '的形式指定。

  **重要提示:** c++函数名必须是**乱码**。如果你手写抑制，使用'--demangle=no '选项来获取错误消息中被篡改的名称。c++名称被破坏的一个例子是' _ZN9QListView4showEv '。这是GNU c++编译器内部使用的格式，也是必须在抑制文件中使用的格式。等效的请求名称' QListView::show() '，是你在c++源代码级别看到的。

  位置线也可以是简单的“‘…’”(三个点)。这是一个帧级通配符，它匹配零个或多个帧。帧级通配符很有用，因为它们可以很容易地忽略感兴趣的帧之间无兴趣的帧的数量变化。当编写抑制时，这通常是很重要的，抑制的目的是对编译器进行的函数内联数量的变化具有健壮性。

- 最后，整个抑制必须在花括号之间。每个大括号必须是该行的第一个字符。

只有当错误匹配抑制中的所有细节时，抑制才会抑制错误。这里有一个例子:

```tex
{
  __gconv_transform_ascii_internal/__mbrtowc/mbtowc
  Memcheck:Value4
  fun:__gconv_transform_ascii_internal
  fun:__mbr*toc
  fun:mbtowc
}
```

它的意思是:仅对于Memcheck，抑制使用未初始化值的错误：当数据大小为4时，当它出现在\_\_gconv_transform_ascii_internal函数中，当从任何名称匹配的\_\_mbr*toc函数调用时，当它从mbtowc调用时。它在其他任何情况下都不适用。向用户标识此抑制的字符串是\_\_gconv_transform_ascii_internal/__mbrtowc/mbtowc。

再举一个Memcheck工具的例子:

```tex
{
  libX11.so.6.2/libX11.so.6.2/libXaw.so.7.0
  Memcheck:Value4
  obj:/usr/X11R6/lib/libX11.so.6.2
  obj:/usr/X11R6/lib/libX11.so.6.2
  obj:/usr/X11R6/lib/libXaw.so.7.0
}
```

这抑制了libX11.so.6.2中出现的任何大小为4的非初始化值错误，当从同一库中的任何位置调用时，当从libXaw.so.7.0中的任何地方调用时。位置的不精确说明令人遗憾，但这也是您所能期望的，因为本示例所使用的Linux发行版上的X11库已经删除了它们的符号表。

src的一个例子，同样作用于Memcheck工具:

```tex
{
  libX11.so.6.2/libX11.so.6.2/libXaw.so.7.0
  Memcheck:Value4
  src:valid.c:321
}
```

这抑制了在validate .c中第321行出现的任何大小为4的未初始化值错误。

尽管上面的两个例子没有说明这一点，但是您可以在抑制中自由地混合使用obj:、fun:和src:行。

最后，这里有一个使用`three frame-level` 通配符的例子:

```tex
{
   a-contrived-example
   Memcheck:Leak
   fun:malloc
   ...
   fun:ddd
   ...
   fun:ccc
   ...
   fun:main
}
```

这抑制了Memcheck内存泄漏错误，在" main "调用" ccc "完成分配的情况下（中间包涵0次或多次间接调用），通过“ddd”向前调用，最终到“malloc”。

### 2.6 调试信息

Valgrind支持通过debuginfod下载debuginfo文件，debuginfod是一个分发ELF/DWARF调试信息的HTTP服务器。当一个debuginfo文件在本地找不到时，Valgrind能够使用它的构建id查询debuginfo服务器。

为了使用这个特性，必须安装`debuginfod-find`并且$DEBUGINFOD_URLS必须包含debuginfod服务器的url。Valgrind不支持通常通过$DEBUGINFOD_PROGRESS和$DEBUGINFOD_VERBOSE来启用的`debuginfod-find`详细输出。这些环境变量将被忽略。

有关debuginfod的更多信息，请参见https://sourceware.org/elfutils/Debuginfod.html

### 2.7 核心命令行选项

如上所述，Valgrind的核心接受一组通用选项。这些工具还接受特定于工具的选项，这些选项针对每个工具单独记录。

在大多数情况下，Valgrind的默认设置成功地给出了合理的行为。我们按粗略类别对可用选项进行分组。

#### 2.7.1 工具选择选项

最重要的选择。运行名为“toolname”的Valgrind工具，例如memcheck, cachegrind, callgrind, helgrind, drd, massif, dhat, lackey, none, expbbv等。

`--tool=<toolname> [default: memcheck]`

#### 2.7.2 基本选项

这些选项适用于所有工具。

| 选项                                              | 说明                                                         |
| ------------------------------------------------- | ------------------------------------------------------------ |
| -h --help                                         | 显示所有选项的帮助，包括核心和所选工具。如果选项重复，则相当于给出--help-debug。 |
| --help-debug                                      | 与--help相同，但也列出了通常只对Valgrind的开发人员有用的调试选项。 |
| --version                                         | 显示Valgrind的核心版本号。工具可以有自己的版本号。有一个方案可以确保工具仅在核心版本是已知它们可以使用的版本时执行。这样做是为了尽量减少因工具与核心版本不兼容而引起的奇怪问题的机会。 |
| -q --quiet                                        | 以静默方式运行，并且仅打印错误消息。如果您正在运行回归测试或具有其他一些自动化测试机制，则非常有用。 |
| -v --verbose                                      | 更详细。提供有关程序各个方面的额外信息，例如：加载的共享对象、使用的抑制、检测和执行引擎的进度以及有关异常行为的警告。重复该选项会增加详细程度。 |
| --trace-children=<yes\|no> [default: no]          | 当启用时，Valgrind将跟踪到通过exec系统调用启动的子进程。这对于多进程程序是必要的。<br/>注意，Valgrind确实会跟踪到fork的子进程(很难不跟踪，因为fork会生成进程的相同副本)，所以这个选项的命名可以说很糟糕。然而，大多数fork调用的子函数都会立即调用exec。 |
| --trace-children-skip=patt1,patt2,...             | 此选项仅在指定--trace-children=yes时生效。它允许跳过一些子元素。该选项接受一个逗号分隔的模式列表，用于Valgrind不应跟踪的子可执行程序的名称。模式可能包括元字符?和*，它们有通常的含义。<br>这对于从Valgrind上运行的进程树中剔除无趣的分支非常有用。但是你在使用它的时候要小心。当Valgrind跳过对可执行文件的跟踪时，它不仅跳过了跟踪该可执行文件，还跳过了跟踪该可执行文件的任何子进程。换句话说，该标志不仅仅导致跟踪在指定的可执行文件处停止 -- 它跳过了对植根于任何指定的可执行文件的整个进程子树的跟踪。 |
| --trace-children-skip-by-arg=patt1,patt2,...      | 这与--trace-children-skip相同，唯一的区别是:是否跟踪到子进程是通过检查子进程的参数来决定的，而不是检查其可执行文件的名称。 |
| --child-silent-after-fork=<yes\|no> [default: no] | 启用后，Valgrind将不会显示由fork调用产生的子进程的任何调试或日志输出。这可以在处理创建子进程时减少输出的混乱(尽管更容易误导)。它在与`--trace-children=`一起使用时特别有用。如果您请求XML输出(--XML =yes)，也强烈建议使用此选项，子XML和父XML可能会混淆，这通常会使其无用。 |
| --vgdb=<no\|yes\|full> [default: yes]             | 当指定--vgdb=yes或--vgdb=full时，Valgrind将提供"gdbserver"功能。这允许一个外部GNU GDB调试器在Valgrind上运行时控制和调试你的程序。--vgdb=full会产生很大的性能开销，但会提供更精确的断点和观察点。<br/>如果嵌入式gdbserver已经启用，但目前没有使用gdb, vgdb命令行实用程序可以从shell向Valgrind发送“监视命令”。Valgrind核心提供了一组Valgrind监视命令。工具可以提供特定于工具的监视命令，这些命令在特定于工具的章节中有文档记录。 |
| --vgdb-error=<number> [default: 999999999]        | 当Valgrind gdbserver使用--vgdb=yes或--vgdb=full启用时，使用此选项。报告错误的工具将等待报告“number”错误，然后冻结程序并等待您连接到GDB。因此，值为0将导致gdbserver在执行程序之前启动。这通常用于在执行前插入GDB断点，也可用于不报告错误的工具，如Massif。 |
| --vgdb-stop-at=<set> [default: none]              | 当Valgrind gdbserver使用--vgdb=yes或--vgdb=full启用时，使用此选项。在--vgdb-error被报告后，Valgrind gdbserver将被调用。你还可以要求其他事件调用Valgrind gdbserver，具体方法如下:<br/>●一个或多个`startup exit valgrindabexit`的逗号分隔列表。<br/>startup exit valgrindabexit分别表示在你的程序执行之前，在你的程序的最后一条指令之后，在Valgrind异常退出(例如内部错误，内存不足，…)时调用gdbserver。<br/>注意:startup和--vgdb-error=0都将导致在你的程序执行之前调用Valgrind gdbserver。--vgdb-error=0还会导致程序在所有后续错误时停止。<br/>●所有指定的完整集合。它等价于--vgdb-stop-at=startup,exit,valgrindabexit。<br/>空集为None。 |
| --track-fds=<yes\|no\|all> [default: no]          | 当启用时，Valgrind将在退出或请求时打印`打开的文件描述符列表`，通过gdbserver监视命令`v.info open_fds`。与每个文件描述符一起打印的还有文件打开位置的堆栈反向跟踪，以及与文件描述符相关的任何详细信息，如文件名或套接字详细信息。使用all来包含对stdin、stdout和stderr的报告。 |
| --time-stamp=<yes\|no> [default: no]              | 启用后，每条消息前面都会显示自启动以来经过的wallclock时间，以天、小时、分钟、秒和毫秒表示。 |
| --log-fd=<number> [default: 2, stderr]            | 指定 Valgrind 应将其所有消息发送到指定的文件描述符。默认值 2 是标准误差通道 （stderr）。请注意，这可能会干扰客户端自己对 stderr 的使用，因为 Valgrind 的输出将与客户端发送到 stderr 的任何输出交错。 |
| --log-file=<filename>                             | 指定Valgrind应将其所有消息发送到指定的文件。如果文件名为空，则会导致中止。可以在文件名中使用三个特殊的格式说明符。<br/>●%p被替换为当前进程ID。这对于调用多个进程的程序非常有用。警告:如果你使用--trace-children=yes，并且你的程序调用多个进程或者你的程序fork而没有调用之后的exec，并且你没有使用这个说明符(或者下面的%q说明符)，来自所有这些进程的Valgrind输出将进入一个文件，可能是混乱的，可能是不完整的。注意:如果程序随后fork并调用exec，在fork和exec之间的时间段内子程序的Valgrind输出将丢失。幸运的是，对于大多数程序来说，这个差距真的很小;现代程序仍然使用posix_spawn。<br/>●%n被替换为此进程唯一的文件序列号。这对于从相同的文件名模板生成多个文件的进程非常有用。<br/>●%q{FOO}被替换为环境变量FOO的内容。如果{FOO}部分格式不正确，则会导致中止。这个说明符很少需要，但在某些情况下非常有用（例如运行MPI程序时)。其思想是指定一个变量，该变量将对作业中的每个进程进行不同的设置，例如BPROC_RANK或任何适用于MPI设置的变量。如果未设置命名的环境变量，则会导致中止。注意，在某些shell中，{和}字符可能需要用反斜杠转义。<br/>●%%替换为%。<br/>如果%后跟任何其他字符，则会导致中止。<br/>如果文件名指定了一个相对文件名，则将它放在程序的初始工作目录中:这是程序在fork之后或exec之后开始执行时的当前目录。如果它指定一个绝对文件名(例如。以'/'开头)，然后放在那里。 |
| --log-socket=\<ip-address:port-number>            | 指定Valgrind应将其所有消息发送到指定IP地址的指定端口。端口可以省略，在这种情况下使用端口1500。如果无法连接到指定的套接字，Valgrind返回将输出写入标准错误(stderr)。该选项旨在与valgrind-listener程序一起使用。 |

#### 2.7.3 与错误相关的选项

所有可以报告错误的工具都使用这些选项，例如Memcheck，但不包括Cachegrind。

| 选项                          | 说明              |
| ----------------------------- | ----------------- |
| --xml=<yes\|no> [default: no] | no> [default: no] |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |
|                               |                   |

#### 2.7.4 malloc-related选项

#### 2.7.5 不常见的选项

#### 2.7.6 调试选项

#### 2.7.7 设置默认选项

#### 2.7.8 动态变化的选项

### 2.8 对线程的支持