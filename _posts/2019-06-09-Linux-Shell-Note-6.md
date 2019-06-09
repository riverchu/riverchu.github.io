---
layout:     post
title:      "Linux Shell Note 6"
subtitle:   "Plan B"
date:       2019-06-09
author:     ChuRiver
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Linux
    - Shell
---

本篇讲文本备份命令

## index
- [tar归档](#tar归档)
- [cpio归档](#cpio归档)
- [gzip压缩](#gzip压缩)
- [zip归档与压缩](#zip归档与压缩)
- [pbzip2归档工具](#pbzip2归档工具)
- [创建压缩文件系统](#创建压缩文件系统)
- [rsync备份快照系统](#rsync备份快照系统)
- [利用git备份](#利用git备份)
- [fsarchiver全盘镜像创建](#fsarchiver全盘镜像创建)
- [dd全盘备份](#dd全盘备份)


## tar归档

`-f` 必须是tar命令的最后一个参数，用于指定文件，可一次指定多个目标文件归档为一个文件存储。

压缩指定文件夹：
```bash
tar -cf output.tar [SOURCES]
```

列出归档文件中所包含的文件
```bash
tar -tf archive.tar
```

`-v`表示冗长模式，可以打印细节参数
```bash
tar -tvf archive.tar
tar -tvvf archive.tar
```

向归档文件中追加文件（对于同名文件也可添加，会包含两个一样的文件）
```bash
tar -rvf original.tar new_file
```

从归档文件中提取文件或文件夹, 类似于解压
```bash
tar -xf archive.tar                             # 解压到当前目录
tar -xf archive.tar -C /path/to/dir             # 指定解压目录
tar -xvf archive.tar archive.tar file1 file2    # 指定文件名，只提取指定文件名的文件或文件夹
```

在tar中使用stdin和stdout,由“-”指定
```bash
tar cvf - files/ | ssh user@example.com "tar xv -C Documents/" # 将tar结果用管道传递至ssh另一端
```

拼接合并归档文件
```bash
tar -Af file1.tar file2.tar # 将file1和file2内容合并至file1.tar
```

通过检查时间戳来更新归档文件中的内容
```bash
tar -uf archive.tar file # 时间戳比原文件更新时才会更新，否则无变化
```

比较归档文件与文件系统中的内容
```bash
tar -df archive.tar # 会将归档文件内文件与外部同名文件进行差别比较
```

归档文件中删除文件
```bash
tar -f archive.tar --delete file1 file2 ...
tar --delete --file archive.tar [FILE LIST]
```

压缩归档文件(不同的压缩方式)
- `-j` bunzip2格式 文件后缀：~.tar.bz2
- `-z` gzip格式 文件后缀：~.tar.gz
- `--lzma` 格式 文件后缀：~.tar.lzma   

追加文件时，`-a`或`--auto-compress`可以自动根据归档文件后缀使用对应压缩方式

归档文件中排除部分文件, `--exclude [PATTERN]`或`-X`
```bash
tar -cf arch.tar * --exclude "*.txt"
tar -cf arch.tar * -X list.txt
```

排除版本控制目录
```bash
tar --exclude-vcs -czvvf archive.tar.gz dir
```

输出总字节数
```bash
tar -cf arch.tar * --exclude "*.txt" --totals

FILE_LIST="f1 f2 f3 f4 f5"
for f in $FILE_LIST;
do
tar -rvf archive.tar $f
done
```


## cpio归档
多用于RPM软件包和initramfs文件，使用stdin获取文件名、并将文件用stdout输出至文件
```bash
echo file1 file2 file3 | cipo -ov > archive.cipo
```

列出cip归档文件中的内容
```bash
cpio -it < archive.cipo
```

提取文件
```bash
cipo -id < archive.cipo
```

- `-o` 指定了输出  
- `-v` 用来打印归档文件列表     
- `-i` 用于指定输入     
- `-t` 列出归档文件的内容      
- `-d` 表示提取   


## gzip压缩
gzip只能压缩***单个文件或数据流***，不能批量操作

压缩
```bash
gzip filename
```

解压缩
```bash
gunzip filename.gz
```

列出压缩文件属性信息
```bash
gzip -l test.txt.gz
```

从stdin中读入，将压缩文件写出到stdout
```bash
cat file | gzip -c > file.gz # -c 指定
```

gzip压缩级别包括`--fast`或`--best`

zcat无需压缩，直接读取gzip格式文件
```bash
zcat test.gz # 列出归档文件内文件
```

压缩率 (共9级)
```bash
gzip -5 test.img # 1-9
```

bzip2更高的压缩率，但较慢
```bash
bzip2 filename # 压缩
bunzip2 filename.bz2 # 解压缩
```

lzma压缩率最高
```bash
lzma filename # 压缩
lzma filename.lzma # 解压缩
tar -cvvf --lzma archive.tar.lzma [FILES]
```

## zip归档与压缩
压缩
```bash
zip archive.zip [SOURCE FILES/DIRS]
```

递归压缩
```bash
zip -r archive.zip folder1 folder2
```

解压缩
```bash
unzip file.zip # 解压缩后不会删除file.zip
```

更新压缩文件内容
```bash
zip file.zip -u newfile
```

删除压缩文件中内容
```bash
zip -d arch.zip file.txt
```

列出压缩文件中内容
```bash
unzip -l archive.zip
```

## pbzip2归档工具
pbzip2是可利用多个CPU核心的归档工具

压缩文件
```bash
pbzip2 filename # 压缩结果为filename.bz2
```

压缩和归档多个文件 (配合tar实现)
```bash
tar cf myfile.tar.bz2 --use-compress-prog=pbzip2 dir/
tar -c dir/ | pbzip2 -c > myfile.tar.bz2
```

提取
```bash
pbzip2 -dc myfile.tar.gz | tar x
pbzip2 -d myfile.tar.bz2
```

指定处理器数量
```bash
pbzip2 -p4 myfile.tar # -p 指定了使用4个处理器核心
```

指定压缩比`1-9`


## 创建压缩文件系统
`squashfs`是一种具有超高压缩率的只读型文件系统，这种文件系统能够将2GB~3GB的数据压缩成一个700M的文件，如linux的liveCD，以环回模式挂载
添加源目录和文件

1. 创建一个`squashfs`文件：
```bash
mksquashfs SOURCES compressedfs.squashfs
```

2. 挂载
```bash
mkdir /mnt/squash
mount -o loop compressedfs.squashfs /mnt/squash
```

3. 排除部分文件
```bash
mksuqashfs /etc test.squashfs -e /etc/passwd /etc/shadow
mksuqashfs /etc test.squashfs -ef excludelist
```


## rsync备份快照系统
`-a` 表示归档
`-v` 冗长模式

复制
```bash
rsync -av source_path destination_path
```

将数据备份到远程服务器或主机
```bash
rsync -av source_dir username@host:PATH
resync -av /home/slynux/data slynux@192.168.0.6:/home/backups/data
```

从远程主机恢复数据到本地主机
```bash
rsync -av username@host:PATH destination
```

压缩传输
```bash
rsync -avz source destination
```

同步目录
```bash
rsync -av /home/test/ /home/backups
```

连带目录一起同步
```bash
rsync -av /home/test /home/backups
```

若source路径末尾带"/"会将source文件夹内文件全部复制到destination位置，若不带则会将source文件夹本身完整同步.  
在source结尾不带"/"时，若destination路径末尾带"/"，会将来源文件全部复制到destination内部，否则会在destination内部新建同名目录。

排除部分文件
- `--exclude PATTERN`
- `--exclude-from FILE_PATH`

```bash
rsync -avz /home/code/some_code /mnt/disk/backup/code --exlude "*.txt"
```

更新rsync时，删除源端已经不存在，但目的端仍有的文件
```bash
rsync -avz SOURCE DESTINATION --delete
```

定期备份可使用`crontab`实现

## 利用git备份
1. 创建备份位置git仓库
```bash
mkdir -p /home/backups/backup.git
cd /home/backups/backup.git
git init --bare
```

2. 在要备份的文件处初始化，并创建远程仓库
```bash
git init
git commit --allow-empty -am "Init"
git remote add origin user@remotehost:/home/backups/backup.git
```

## fsarchiver全盘镜像创建
备份
```bash
fsarchiver savefs backup.fsa /dev/sda1 #将/dev/sda1备份至backup.fsa
```

同时备份多个分区
```bash
fsarchiver savefs backup.fsa /dev/sda1 /dev/sda2
```

从备份归档中恢复分区
```bash
fsarchiver restfs backup.fsa id=0,dest=/dev/sda1 # id指定归档文件中的第一个分区，dest指定恢复位置
```

从备份文档中恢复多个分区
```bash
fsarchiver restfs backup.fsa id=0,dest=/dev/sda1 id=1,dest=/dev/sdb1
```

## dd全盘备份

备份
```bash
dd -if=/dev/sda -of=/dev/sdb
dd -if=/dev/sda1 -of=~/sda1.img
```

恢复
```bash
dd -if=/dev/sdb -of=/dev/sda
dd -if=~/sda1.img -of=/dev/sda1
```
