---
title: Linux文件系统：inode&block，文件&目录，硬链&软链
tags:
  - Linux
  - 面试
  - 原创
date: 2018-03-10 00:10:13
---

`inode`与`block`是理解Linux文件系统的基础概念。在此基础上实现了`文件`、`目录`、`硬链`、`软链`。

<!--more-->

>操作系统：CentOS 6.5（Linux 2.6）。

# inode & block

## block和inode的意义

理解inode，要从文件储存说起。

文件储存在硬盘上，**硬盘的最小存储单位叫做"`扇区`"**（Sector，常见512B），**多个扇区组成的一个"`块`"**（block，常见4096B，即连续8个sector组成一个block）。操作系统一次性读取一个块（8个连续8个扇区），以提高磁盘IO效率。因此，**块是磁盘读写的最小单位**。

同时，**块是文件存取的最小单位**，多个块组成`文件内容`。显然，还需要一个结构存储`文件元信息`（文件的创建者、创建日期、内容大小等），也就引出了`索引节点`（inode，Index Node）。**索引节点用于索引一个文件的所有块**。

## inode的内容

inode维护文件的元信息。可以用`stat <path>`命令查看某个文件的inode信息：

```bash
➜  fs_test stat example.txt
  File: "example.txt"
  Size: 4847      	Blocks: 16         IO Block: 4096   普通文件
Device: fd00h/64768d	Inode: 261774      Links: 1
Access: (0664/-rw-rw-r--)  Uid: (  500/    lauo)   Gid: (  500/    lauo)
Access: 2018-03-09 22:55:01.306013501 +0800
Modify: 2018-03-09 22:55:01.307013501 +0800
Change: 2018-03-09 22:55:31.796013501 +0800
```

**除了文件名**以外的所有文件信息，都存在inode之中。

>至于为什么没有文件名，下面会有详细解释。

## inode大小与inode数量上限

inode也会消耗硬盘空间，所以_**硬盘格式化的时候**，操作系统自动将硬盘分成两个区域：一个是`数据区`（data area），存放block；另一个是`inode区`（也交inode表，inode table），存放inode_。实际上还有其他区，暂时不考虑。

每个inode节点的大小，一般是128B或256B。可以用如下`dumpe2fs -h <disk_path>`命令查看：

```bash
➜  fs_test sudo dumpe2fs -h /dev/sda1 | grep "Inode size"
dumpe2fs 1.41.12 (17-May-2010)
Inode size:	          128
```

>`/dev/sda1`指挂载的SATA盘。至于SATA盘是什么，猴子暂时木有关心。

**inode节点的最大数量，在格式化时就给定**。在初始化文件系统时，inode table初始大小为0，然后，操作系统每分配1024B或2048B给data area，就分配inode大小的空间（128B或256B）给inode table，保持2048 : 256 = 8 : 1的分配比例。注意，此处仅仅是**保持data area与inode table的分配比例（如8:1）**，与未来一个inode指向多少block无关。

假定在一块2GB的硬盘中，data area与inode table的分配比例为8 : 1，则inode table占用11.11%的磁盘空间，即227.56MB；inode size 256B，则inode数量的上限为932067。

使用如下`df -i`命令查看每个硬盘分区的inode数量上限、已用数量、剩余数量等：

```bash
➜  fs_test df -i
Filesystem                   Inodes IUsed  IFree IUse% Mounted on
/dev/mapper/VolGroup-lv_root 912128 33668 878460    4% /
tmpfs                         63627     1  63626    1% /dev/shm
/dev/sda1                    128016    38 127978    1% /boot
```

既然inode节点的数量存在上限，而每个文件又必须有一个inode，因此**有可能发生inode已经用光，但是硬盘还未存满的情况**（如每个文件的大小都小于2048B）。这时，就无法在硬盘上创建新文件了。

## inode号

Linux系统**使用inode号来识别文件**，文件名仅仅是文件的别名。

表面上，用户通过文件名打开文件——实际上，系统内部分三步完成这一过程：

1. 找到文件名对应的inode号（内存）。
2. 通过inode号码，获取inode内容（磁盘）。
3. 最后，根据inode内容，找到文件数据存储的所有block，读出数据（磁盘）。

>因此，如果只获取文件名与inode号码，是不需要读取磁盘的。

使用`ls -i <path>`命令查看文件名对应的inode号码：

```bash
➜  fs_test ls -i example.txt
261774 example.txt
```

## inode到block的映射

><font color="red">// TODO: 操作系统讲过，待考证</font>。

# 文件 & 目录

## 文件

介绍inode时，已经说明了文件的实现——每个文件都对应一个inode，文件名是文件的别名，inode是文件的本体；操作系统维护文件名到inode的映射。

Linux系统中，**一切皆文件**，包括后面要解释的目录、硬链接、软链接，包括socket、设备、内存等。理解这一点非常重要。

## 目录文件

Linux系统中，`目录`（directory）也是一种文件。打开目录，实际上就是打开`目录文件`。

目录文件的结构非常简单，就是一系列`目录项`（dirent）的列表。**目录项是一个二元组`<文件名, inode号码>`**。

`ls <path>`命令只列出目录文件中的所有文件名：

```bash
➜  fs_test ls exp_dir
example_1.txt  ex_dir_1
```

`ls -i <path>`命令列出整个目录文件，即文件名和inode号码：

```bash
➜  fs_test ls -i exp_dir
261780 example_1.txt  261781 ex_dir_1
```

>前面`ls -i <file>`的时候只需要查询文件名到inode的map；而此处`ls -i <dir>`则多一次读取目录文件的开销。

如果要查看文件的详细信息，就必须根据inode号码，读取inode内容（磁盘）。`ls -l <path>`命令列出文件的详细信息：

```bash
➜  fs_test ls -l exp_dir
总用量 12
-rw-rw-r--. 1 msh msh 4847 3月   9 23:15 example_1.txt
drwxrwxr-x. 2 msh msh 4096 3月   9 23:16 ex_dir_1
```

理解了上面这些知识，就能理解目录的权限：

* 目录文件的读权限（r）和写权限（w）针对目录文件本身
* 由于目录文件内只有文件名和inode号码，**如果想读取目录项的inode内容，Linux规定需要用户拥有目录的可执行权限（x）**。

# 硬链 & 软链

## 硬链接

一般情况下，文件名和inode号码是"一一对应"的。但是，Linux系统允许**多个文件名指向同一个inode号码**，也就是`硬链接`（hard link）。

一旦建立硬链，就不再区分则源文件与硬链文件的概念，二者是指向相同inode的“平等的两个文件”（如果以文件名区分）。由于指向相同的inode，硬链接的行为就很好理解了：

* 如果修改文件内容，则通过其他文件名访问的内容也随之修改。
* 如果删除文件，则系统检查是否还有其他文件名指向inode。如果没有，则删除inode与block；否则，仅删除当前文件名到inode的映射。

使用`ln <src> <hard_link>`命令创建硬链接：

```bash
➜  fs_test ln example.txt example_2.txt
➜  fs_test ls -li
总用量 20
261774 -rw-rw-r--. 2 msh msh 4847 3月   9 22:55 example_2.txt
261774 -rw-rw-r--. 2 msh msh 4847 3月   9 22:55 example.txt
261779 drwxrwxr-x. 3 msh msh 4096 3月   9 23:16 exp_dir
```

第一列是inode号，第3列是指向该inode的文件名的数量，即`链接数`。可见，`example.txt`、`example_2.txt`的inode号都为261774；公有2个文件名指向inode 261774。

顺便说一下，**"`.`"和"`..`"也是目录，通过硬链分别指向当前目录和父目录**。因此，所有目录的链接数必然大于等于2，即当前目录和"."目录。如上，`exp_dir`目录的链接数为3。

## 软链接

硬链使用起来不是那么方便，管理起来也比较麻烦。于是使用另一种更简单的方式实现了`软链`（soft link，或称符号链接symbolic link）：**如果文件A软链指向文件B，则标记文件A为软链文件，并在文件A中记录文件B的路径**。

此时，文件A与文件B使用不同的inode。那么，如何通过文件A访问到文件B的内容呢？_操作系统会在我们访问文件A时，发现文件A是软链文件，则自动将访问者导向文件B_。因此，无论打开哪一个文件，最终读取的都是文件B。

对于读、修改操作，硬链接与软连接实现的效果相同。但删除（包括rename）操作上的表现不同：

* 如果删除文件A，则文件B无影响。
* 如果删除文件B，则文件A依然存在，但访问文件A时会报错"没有那个文件或目录"。

使用`ln -s <src> <soft_link>`命令创建软链接：

```bash
➜  fs_test ln -s example.txt example_3.txt
➜  fs_test ls -li
总用量 20
261774 -rw-rw-r--. 2 msh msh 4847 3月   9 22:55 example_2.txt
261782 lrwxrwxrwx. 1 msh msh   11 3月  10 00:05 example_3.txt -> example.txt
261774 -rw-rw-r--. 2 msh msh 4847 3月   9 22:55 example.txt
261779 drwxrwxr-x. 3 msh msh 4096 3月   9 23:16 exp_dir
```

可见， `example_3.txt`是软链接文件，指向`example.txt`，二者的inode号码不同，链接数也没有关系。

---

>参考：
>
>* [理解inode](http://www.ruanyifeng.com/blog/2011/12/inode.html)
>* [硬盘类型和Linux分区](http://blog.csdn.net/zollty/article/details/7001950)
>* [linux文件系统—inode及相关概念 inode大小的最佳设置](http://blog.csdn.net/lidan3959/article/details/16981137)
>
