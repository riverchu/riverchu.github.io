---
layout:     post
title:      "Linux Shell Note 3"
subtitle:   "Shell 文件操作"
data:       2019-04-16
author:     "ChuRiver"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Linux
    - Shell
---

本篇详细说说文件操作命令

## Index

- [生成文件](#生成文件)
- [交集与差集](#文本文件的交集与差集)
- [操作重复文件](#操作重复文件)
- [文件权限和粘滞位](#文件权限和粘滞位)
    - [创建不可修改文件](#创建不可修改文件)
- [touch](#touch)
- [符号链接](#符号链接)
- [列举文件类型统计信息](#列举文件类型统计信息)
- [环回文件](#环回文件)
- [ISO及混合型ISO](#iso及混合型iso)
- [文件差异比较](#文件差异比较)
- [文件内容查看](#文件内容查看)
    - [head](#head)
    - [tail](#tail)
- [ls](#ls)
- [目录栈](#目录栈)
- [wc](#wc)
- [tree](#tree)

## 生成文件
> Copy a file, converting and formatting according to the operands.

`dd`命令可以生成数据流，并存储至文件或其他位置

- `if`    参数指定读取位置，默认为stdin
- `of`    参数指定输出位置，默认为stdout
- `bs`    指定block size
- `count` 指定生成block的个数

```sh
dd -if=/dev/zero -of=data.txt bs=1m count=1
```

## 文本文件的交集与差集
`comm`命令
> Compare sorted files FILE1 and FILE2 line by line

- 交集: 两个文件中相同的行
- 求集: 指定文件中包含但互不相同的行
- 差集: 打印出包含在文件A中但不包含在其他文件中的行

命令要求输入必须是排过序的文件，输出默认为3列，即：

|只在A中出现的行|只在B中出现的行|A、B中相同的行|

e.g.
```shell
comm A.txt B.txt        # 输出三行
comm A.txt B.txt -1 -2  # 不显示第一列 只显示两文件中相同的行
comm A.txt B.txt -3     # 不显示第三列
comm A.txt B.txt -3 | sed 's/^\t//' # 不显示第三列 去掉每行开头的制表符
comm A.txt B.txt -1 -3  # 不显示第一列第三列
```

## 操作重复文件
用脚本实现查找、删除重复文件的操作。
```

ls -lS --time-style=long-iso | awk ' BEGIN {    
    getline; getline;
    name1=$8; size=$5
}
{
    name2=$8;
    if (size==$5)
    {
        "md5sum "name1 | getline; csum1=$1;
        "md5sum "name2 | getline; csum2=$1;
        if ( csum1==csum2 )
        {
            print name1; print name2;
        }
    };
    size=$5; name1=name2;
}' | sort -u > duplicate_files
cat duplicate_files | xargs -I {} md5sum {} | sort | uniq -w 32 | awk '{print "^"$2"$" }' | sort -u > duplicate_sample
echo Removing...
comm duplicate_files duplicate_sample -2 -3 | tee /dev/stderr | xargs rm
echo Removed duplicates files successfully.

```

## 文件权限和粘滞位
```sh
$ls -l
drwxr-xr-x  2 root root 4096 Aug 18 08:08 bak
-rwxr-xr-x  1 root root   52 Nov 23  2017 update.sh
```
第一列代表文件类型：

字符|意义
:-:|:-:
-|普通文件
d|目录
c|字符设备
b|块设备
l|符号链接
s|套接字
p|管道

### 特殊权限位
S（对应rwx的x）setuid/setgid，只能使用在Linux ELF格式的二进制文件上，代表任意用户可以文件所有者的权限执行可执行文件。

### 读/写/执行权限

r|允许读取目录中的文件和子目录的列表
w|允许在目录中创建或删除文件或目录  
x|指明是否可以访问目录中的文件和子目录    


### 粘滞位
只有创建目录的用户才有权限删除目录内的文件，对应其他用户权限中x位置。若既有x也有粘滞位 用T，否则用t。
e.g.
```sh
-------rwt
-------rwT
```

`chmod`命令设定文件权限。
> Change the mode of each FILE to MODE.

e.g.
```sh
chmod u=rwx g=rw o=r filename   # 分别制定创建者user，创建者所在组group，和其他用户others权限
chmod o+x filename              # 为others增加执行权限
chmod a+x filename              # 为所有人增加执行权限
chmod a-x filename              # 去掉所有人的执行权限

chmod 777 filename              # 赋予文件所有权限

chmod a+t filename              # 增加粘滞位权限

chmod 777 . -R                  # 递归赋予全部权限
chmod 777 "$(pwd)" -R           # 递归当前目录，并赋予全部权限

chmod +s exe_file               # 增加特殊权限
```

`chown`命令可以修改某个文件或目录的所有者和所属的组。
> Change the owner and/or group of each FILE to OWNER and/or GROUP.

chown user:group filename
e.g.
```sh
# 将文件 file1.txt 的拥有者设为river群体的使用者chu :
chown chu:river file
```

### 创建不可修改文件

`chattr`命令可以设定文件是否可以修改， `+i`设置文件不可更改，`-i`去掉文件不可更改属性
e.g.
```sh
chattr +i hello.txt # 设置为不可修改
chattr -i world.tar # 去掉不可修改属性
```

## touch

`touch`命令用来生成空白文件和修改文件时间属性。
> Update the access and modification times of each FILE to the current time.

用法为`touch filename`。

e.g.
```sh
for name in blank{1..100}.txt
do
    touch $name
done
```

若已经存在则修改所有时间戳为当前时间。（在默认情况下，ls显示出来的是该文件的mtime，即文件内容最后修改时间，如果你需要查看另外两个时间，可以加上--time参数）
```sh
touch -a file               # 修改访问时间为当前时间
touch -m file               # 修改修改为当前时间
touch -d "format time" file #修改时间戳为指定的时间

touch -r bbb.txt aaa.txt    # 用文件bbb.txt的时间记录(访问时间与修改时间)来修改aaa.txt的时间记录
```

## 符号链接
`ln`命令可创建符号链接。
```sh
ln -s target symbolic_link_name # 创建符号链接

# 列出所有的符号链接
ls -l | grep "^l"
find . -type l -print

# 打印出符号链接所指向的目标路径
readlink sym
ls -l sym
```

## 列举文件类型统计信息

`file`命令可查看文件类型信息，用法：`file filename`。

e.g.
```sh
file -b filename # 打印不包括文件名在内的文件类型信息
```

e.g. 统计各类型文件个数的脚本
```
#!/bin/bash
if [ $# -ne 1 ];
then
    echo "Usage is $0 basepath";
    exit
fi
path=$1
declare -A statarray;
while read line;
do
    ftype=`file -b "$line" | cut -d, -f1`
    let statarray["$ftype"]++;
done < <(find $path -type f -print) # <(find)作为子进程输出代替文件名
# done <<< "`find $path -type f -print`"
echo ==========================File types and count ===============================
for ftype in "${!statarray[@]}"; # ${!statarray[@]} 返回一个数组索引列表
do
    echo $ftype : ${statarray["$ftype"]}
done
```

## 环回文件
环回文件指在文件中而非物理设备中创建的文件系统(ntfs、exfat或者ext4等文件系统格式)。   
创建->挂载->卸载的全过程如下：

- 创建环回文件并挂载、卸载:

```sh
# 测试创建文件系统
dd if=/dev/zero of=loopbackfile.img bs=1G count=1

# 用mkfs命令将1gb的文件格式化为ext4文件系统 exfat-兼容mac和windows
mkfs.ext4 loopbackfile.img

# 检查文件系统
file loopbackfile.img

# 挂载
mkdir /mnt/loopback
mount -o loop loopbackfile.img /mnt/loopback #-o执行挂载环回文件系统

# 或手动操作
losetup /dev/loop1 loopbackfile.img
mount /dev/loop1 /mnt/loopback

# 卸载
umount mount_print
```

- 创建分区、将换回文件挂载至分区:

```sh
# 创建分区
losetup /dev/loop1 loopback.img
fdisk /dev/loop1

# 创建挂载第一个分区
losetup -o 32256 /dev/loop2 loopback.img # -o 指定偏移量

# 快速挂载带有分区的环回磁盘镜像
kpartx -v -a diskimage.img

mount /dev/mapper/loop0p1 /mnt/disk1
unmount ...

# 移除映射关系
kpartx -d diskimage.img
```

可将iso作为环回文件进行挂载：
```sh
mkdir /mnt/iso
mount -o loop linux.iso /mnt/iso
```

### 应用环回文件的更改
`sync`可强迫将已更改的数据写入磁盘，并更新极快。对于环回文件的操作可用此方式更新。

## ISO及混合型ISO

从cdrom创建一个ISO镜像:
```sh
cat /dev/cdrom > image.iso
dd if=/dev/cdrom of=image.iso
```

`mkisofs`创建文件系统:
```sh
mkisofs -V "Label" -o image.iso source_dir/ # -V 指定卷标 -o iso文件路径
```

混合型ISO 可写入usb并可引导：
```sh
isohybird image.iso
```

iso导入硬盘：
```sh
dd if=image.iso of=/dev/sdb1
cat image.iso >> /dev/sdb1
```

命令行刻录ISO：
```sh
# -speed SPEED 指定刻录速度
# -multi 多区段

cdrecord -v dev=/dev/cdrom image.iso
```

弹出光驱命令(基本上用不上了吧)
```sh
eject
eject -t # 合上光驱
```

## 文件差异比较
`diff`命令可比较不同文件之间的差异，是git diff命令的灵感来源。

一体化输出：
```sh
diff -u
```

根据差异存储文件恢复原文件
```sh
diff -u a.txt b.txt > version.patch
patch -p1 version1.txt < version.patch #再输入一遍可撤销修改
diff -Naur d1 d2 #对于目录进行递归
```
-N 将所有缺失文件视为空文件     
-a 将所有文件视为文本文件  
-u 生成一体化输出  
-r 遍历   

## 文件内容查看

### head
打印文件头部
```sh
head file                   # 打印前10行
cat text | head             # stdin读取数据

head -n 4 file              # 指定打印前4行
head -n -M file             # 打印除了最后m行外所有行
```

### tail
```sh
tail file                   # 打印最后10行
cat text | tail             # stdin读取数据

tail -n 5 file              # 打印最后5行
tail -n +(M+1)              # 打印前M行之外的所有行

tail -f growing_file        # 追踪打印新出现的行
tail -f filename -s time    # -s指定睡眠间隔
tail -f filename --pid PID  # 指定跟随进程pid，进程结束时，tail也结束
```

## ls
`ls`可以说是最常用的命令之一了。
```sh
# 列出当前路径下的目录
ls -d */
ls -F | grep "/$"
ls -l | grep "^d"
find . -type d -maxdepth 1 -print
```

## 目录栈

### pushd
pushd会压入当前路径 并转到指定路径:
```sh
~ $ pushed /var/www
```
执行后会压入~ 切换到/var/www；`dirs`查看栈内容，`pushd +n`切换到指定目录。

### popd
有压栈就有出栈：
```sh
popd        # 将当前目录更改为上级目录
popd +n     # 移除特定的路径
```

`cd -`命令可以回到上一个目录。

## wc

> Print newline, word, and byte counts for each FILE, and a total line if more than one FILE is specified.  A word is a non-zero-length sequence of characters delimited by white space.

`wc`统计行数：
```sh
wc -l file
cat file | wc -l
```

`wc`统计单词数：
```sh
wc -w file
cat file | wc -w
```

`wc`统计字节数：
```sh
wc -c file
cat file | wc -c
```

默认使用：
```sh
wc file # 分别打印出行数、单词数、字符数
wc file -L # 打印出文件中最长的一行的长度
```

## tree
`tree`查看文件结构
```sh
tree path -P PATTERN # 用通配符描述样式
tree path -I PATTERN # 排除符合通配符样式的文件
tree -h # 同时打印文件和目录大小
```

可将结果输出为html文件，并开启web服务
```sh
tree PATH -H http://localhost -o out.html # url可随意替换
```
