实验三
======

提交截止时间为5月13日晚23:00，请注意GitHub将在截止时间到达时锁定你的仓库，不再接受变更。

请大家通过此链接在GitHub上提交[实验三](https://classroom.github.com/a/yeJkKW4S)。


实验目的
--------

本实验旨在通过编写一个基于内存的文件系统，加深对文件系统和内存管理方面的理解。

注：本实验不强制要求使用何种编程语言、使用FUSE、或使用Linux操作系统，只要完成实验要求即可，但为了叙述方便，以下文档使用C语言在Linux上通过FUSE实现。


实验内容
--------

1. 创建一个可挂载的内存文件系统，此文件系统的文件信息均存储于内存中
2. 可以进行基本的文件操作（增删改普通文件）
3. 自行实现此文件系统的内存分配


实验环境
--------

本文档默认使用64位Linux操作系统，使用gcc编译器，并使用目前大部分Linux操作系统自带的2.9.x版本的libfuse为例编写，请先安装相应的编译环境，并安装libfuse（如在Ubuntu下安装libfuse-dev软件包，或在Gentoo下安装sys-fs/fuse软件包）。


使用FUSE
--------

FUSE是用户空间文件系统，它避免了在内核模式开发文件系统，可以大幅简化工作量，但不可避免的会带来额外的内核态/用户态的开销。

请先创建一个`oshfs.c`文件，内容如下：

```c
#define FUSE_USE_VERSION 26
#include <fuse.h>
static const struct fuse_operations op;
int main(int argc, char *argv[])
{
    return fuse_main(argc, argv, &op, NULL);
}
```

使用如下指令编译：

```sh
$ gcc -D_FILE_OFFSET_BITS=64 -o oshfs oshfs.c -lfuse
```

新建挂载点：

```sh
$ mkdir mountpoint
```

之后即可使用`oshfs`挂载你的文件系统到mountpoint上：

```sh
$ ./oshfs mountpoint
```

此时可以通过`mount`命令确认你的文件系统已被挂载，但如果运行`ls mountpoint`，则会出现以下错误提示：

```sh
$ ls mountpoint
ls: cannot access 'mountpoint': Function not implemented
```

说明我们还未实现相关函数，因此此文件系统不支持列目录操作。


libfuse的示例程序
-----------------

在进行以下实验之前，请大家先了解一下FUSE的工作原理。

libfuse自身有几个简单的例子，可参见[libfuse的代码](https://github.com/libfuse/libfuse/tree/fuse_2_9_bugfix/example)。

其中fusexmp是一个简单的passthrough，可供参考。

hello则是一个简单的只读文件系统，其中只有一个hello文件，存放了"Hello World!\n"字符串。


一个朴素的实现
--------------

既然是内存文件系统，那么最简单的实现也就是直接将文件列表的链表和文件内容直接存放在malloc分配的内存中，一个简单的实现见[示例程序](oshfs.c)。

此程序可创建、修改文件，但未实现删除文件，也不支持目录操作。

你需要自行实现对文件的删除操作，对于目录操作的实现则不做强制要求。


存储管理
--------

在上述示例程序中，我们采用了malloc分配内存，在本实验中需要自行使用mmap向操作系统申请一块内存，并自行决定存储空间的分配算法。

mmap和munmap的使用方式可参考示例代码的全局mem变量和init函数中的两个demo。

全局mem变量保存了每个block的内存指针（NULL即代表尚未分配），我们应该将所有的文件系统信息均存储于mem指向的内存中。换句话说，我们在此实验中，在用户态模拟实现一个换页算法，mem指针数组的每个指针均指向一个页面的进程实际地址，其总体构成一个线性地址空间，供我们在实现文件系统时使用。

init的Demo 1展示了如何通过mmap分配一个blocksize的空间；Demo 2中展示了你可以通过一个系统调用获得一块连续的空间，然后自行切分，可单独对任意内存区域进行munmap操作。


实验要求
--------

1. 设计文件系统块大小blocksize。
2. 设计一段连续的文件系统地址空间，文件系统地址空间大小为size，则共有blocknr=size/blocksize个block。
3. 自行设计存储算法，将文件系统的所有内容（包括元数据和链表指针等）均映射到文件系统地址空间。
4. 将文件系统地址空间切分为blocknr个大小为blocksize的块，并在*需要时*将对应块通过mmap映射到进程地址空间，在不再需要时通过munmap释放对应内存。
5. （选做）选做内容包括但不限于：实现目录操作、修改文件的各种元数据、健壮的错误处理等。


注意
----

1. 在文件系统地址空间剩余可用的空间不连续时，如果此时要求创建一个大文件，只要其总大小小于剩余可用空间，则应该切分存储，而不应报错空间不足。
2. 对于你的文件系统，通过读取虚拟地址空间的信息应能完全恢复整个文件系统所有信息（包括但不限于目录结构、文件所有者、内容等）。
3. 请尽量优化算法，减少文件碎片，提高读写效率。
4. 如你选择使用其他编程语言，依旧需要通过上述方式分配内存，不可使用语言自带的堆栈分配器。
5. 如你选择在Linux（甚至Windows）上通过内核模块实现，那么你可以忽视上面所有关于内存分配的要求，即可用kmalloc等所有内核函数。


FAQ
---

1. 示例程序中的函数分别是什么含义？

请参考[libfuse的API文档](https://github.com/libfuse/libfuse.github.io/archive/3eb76850180447e128a0cd68aaec642df2ab05d2.zip)

2. 如何测试？

你可以使用基本的Linux命令测试你的文件系统，比如：

```sh
$ cd mountpoint
$ ls -al
$ echo helloworld > testfile
$ ls -l testfile # 查看是否成功创建文件
$ cat testfile # 测试写入和读取是否相同
$ dd if=/dev/zero of=testfile bs=1M count=2000
$ ls -l testfile # 测试2000MiB大文件写入
$ dd if=/dev/urandom of=testfile bs=1M count=1 seek=10
$ ls -l testfile # 此时应为11MiB
$ dd if=testfile of=/dev/null # 测试文件读取
$ rm testfile
$ ls -al # testfile是否成功删除
```

3. 性能和兼容性方面的考虑？

本实验不要求文件系统的性能，但这里提出几个问题，可以供大家思考：

首先是过多的链表操作。比如如果对于文件的随机读写请求都需要遍历以便读写位置前的链表所有节点，是否存在效率问题。

其次是对齐。如果将每个block作为链表的一个结点，那么一个block实际存储的信息便不是一个整数，比如对于4KiB的block文件系统，那么每个block会存储3.9KiB的文件内容和一个next指针，这对一些期望文件物理对齐的应用不友好，比如一个存储raw格式的虚拟磁盘，在其上还有一个文件系统的情况。

此外关于兼容性。关于大小端序的问题，由于内存文件系统一般在同一个cpu上操作，因此可以不考虑。但关于文件元信息，直接将struct stat存储在文件系统中是否是一个好的做法？

4. 关于文件系统的限制？

文件系统存在一些合理的使用限制可以方便我们开发，比如ext4限制文件名255字符、最大文件大小16TiB、最大文件数目等等。但限制最大文件4KiB、最多10个文件显然不是合理的限制。注意本实验主要考察存储管理算法，设定不合理的限制一般意味着你的算法本身不合理。对于今天来说，GiB量级的文件操作是很常见的。

5. 如何调试？

可以通过-f参数使程序在前台运行，可以直接通过stdio输出调试信息：

```sh
$ ./oshfs -f mountpoint
```


评分细则
--------

1. 程序可以正确运行(1)
2. 实现了文件到文件系统的映射算法(2)
3. 实现了存储空间的动态管理算法(1)
4. 算法及实现：思路清晰，性能、可扩展性好，以及选做功能的实现情况(2)
5. 注：满分5分，超过满分的以5分计分

*本次实验将采用人工批改与计算机自动批改相结合的方式，凡确认的抄袭行为将导致本次实验0分，请不要复制他人的代码。*
