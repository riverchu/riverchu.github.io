---
layout:     post
title:      "Linux Shell Note(2)"
subtitle:   "Shell 简单命令"
date:       2019-04-15
author:     "ChuRiver"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Linux
    - Shell
---

本篇简单介绍一下shell上的基本命令

## Index

- [cat](#cat)
- [find](#find)
    - [基础用法](#基础用法)
    - [基于文件时间搜索](#基于文件时间搜索)
    - [基于文件大小搜索](#基于文件大小搜索)
    - [删除匹配的文件](#删除匹配的文件)
    - [基于文件权限和所有权的搜索](#基于文件权限和所有权的搜索)
    - [执行命令或动作](#执行命令或动作)
    - [跳过指定目录](#跳过指定目录)
- [xargs](#xargs)
- [tr](#tr)
- [hash](#hash)
    - [md5sum](#md5sum)
    - [openssl](#openssl)
- [加密工具与散列](#加密工具与散列)
- [排序与去重](#排序与去重)
    - [排序](#sort)
    - [去重](#uniq)
- [临时文件](#临时文件)
- [分割文件](#分割文件)
- [文件名操作](#文件名操作)
    - [扩展名切分](#扩展名切分)
    - [批量重命名和移动](#批量重命名和移动)
- [拼写检查与词典操作](#拼写检查与词典操作)
- [交互输入自动化](#交互输入自动化)

## cat
cat - 查看文件命令
```shell
cat file1 file2 file3
echo 'text' | cat - file.txt
cat -s file.txt # 去掉多余的换行
cat -T # 将制表符显示为^|
cat -n # 显示行号 增加-b跳过对空白行的标号
```
录制并回放终端会话
```shell
script -t 2> timing.log -a output.session
type commands;
......
exit
scriptreplay timing.log output.session #回放命令
```

## find
find - 查找文件命令

### 基础用法
```
find . # 列出所有文件名
find . -print0 #使用\0作为定界符
搜索文件名
find . -name "*.txt" -print
find . -iname "*.txt" -print #iname忽略大小写
find . \( -name "*.txt" -o -name "*.pdf" \) -print
find . -path "*/test/*" -print #通配符匹配文件名和路径 可用-regex参数使用正则方式匹配
find . -regex ".*\(\.py\|\.sh\)$" # iregex忽略大小写
否定参数
find . ! -name "*.txt" -print #反向过滤
基于深度的搜索
find . -maxdepth 1 -name "f*" -print
根据文件类型进行搜索
find . -type d -print
```

type参数列表

文件类型|参数类型
:-:|:-:
普通文件|f  
符号链接|l
目录|d
字符设备|c
块设备|b
套接字|s
FIFO|p

### 基于文件时间搜索
单位：天    
- -atime 访问时间    
- -mtime 修改时间    
- -ctime 变化时间    

unix系统没有创建时间的概念,所以不存在相应参数。

ex.
```shell
# -7    7天内被访问的文件
# 7     正好7天前访问的文件
# +7    超过7天被访问的文件
find . -type f -atime -7 -print
```

单位:分钟
- -amin
- -mmin
- -cmin
使用方法同上

-newer参数指定参考文件      

ex.
```shell
# 找出修改时间比file.txt新的所有文件
find . -type f -newer file.txt -print
```

### 基于文件大小搜索
文件大小的描述参数：  

大小单位|参数
:-:|:-:
块(512字节)|b
字节|c  
字(2字节)|w
1024字节|k
1024K字节|M    
1024M字节|G

ex.
```shell
find . -type f -size +2k # 单位kb
```

### 删除匹配的文件
```shell
find . -type f -name "*.swp" -delete
```

### 基于文件权限和所有权的搜索
```shell
find . -type f -perm 644 -print #找出权限为644的文件
find . -type f -user root -print # 所有者为root的文件 root可用uid代替
```

### 执行命令或动作
用'+'代替';'可以将文件拼接为一个参数执行
ex.
```shell
find . -type f -user root -exec chown river {} \; # 变更文件所有者
find . -type f -name "*.txt" -exec printf "%s" {} \;
```

### 跳过指定目录
```shell
find . \( -name ".git" -prune \) -o \( -type f -print \)
```

## xargs
组合命令的输出和输入

基本形式
```shell
# 多行输入换单行
cat example.txt|xargs
# 单行换多行
cat example.txt|xargs -n 3 #指定每行的最大参数数量

echo "123x123x123" | xargs -d x #指定定界符为x，将其替换为空格
```

ex.
```shell
cat args.txt | xargs -n 1 ./test.sh # cat的输出每个参数都作为test.sh的参数执行
cat args.txt | xargs -I {} ./test.sh -p {} -l # -I指定替换的字符串{}
find . -type f -name "*.txt" -print0 | xargs -0 rm -f # -0指定定界符为\0

# 可循环执行
cat files.txt | ( while read arg; do cat $arg; done ) # 替换cat $arg可以一次执行多个命令
```

## tr

命令格式：`tr [options] set1 set2`   
ex.
```shell
echo "HELLO WORLD" | tr 'A-Z' ‘a-z' # 字符集合 'ABD-}' 'aA.,' 'a-ce-x' 'a-c0-9' 不连续的字符会被视为三个字符 \t \n也可使用
```

### 加密
```shell
echo 12345 | tr '0-9' '9876543210'
```

### 解密
```shell
echo 12345 | tr '9876543210' '0-9’
```

### ROT13加密法
```shell
echo 'tr came, tr saw, tr conquered.' | tr 'a-zA-Z' 'n-za-mN-ZA-M'
echo 'tr came, tr saw, tr conquered.' | tr 'n-za-mN-ZA-M' 'a-zA-Z'
tr '\t' ' ' < file.txt
```

### 删除
```shell
cat file.txt | tr -d '[set1]'
```

### 补集
```shell
tr -c [set1] [set2] # set2是可选的 补集是数学概念
```

### 压缩
```shell
tr -s '[set]' # 压缩重复的字符
```

### 计算
```shell
cat num.txt | echo $[ $(tr '\n' '+' ) 0 ]
```

### 字符类

字符集名称|字母和数字
:-:|:-:
alnum|字母和数字
alpha|字母
cntrl|控制符（非打印）
digit|数字
graph|图形字符
lower|小写字母
print|可打印字符
punct|标点符号
space|空白字符
upper|大写字母
xdifit|十六进制数字

ex.
```shell
tr '[:lower:]' '[:upper:]'
```

## hash

### md5sum
计算：`md5sum filename`   
ex. `md5sum file1 file2 file3`    

校验：`md5sum -c MD5_file`     
ex. `md5sum -c *.md5`

类似的还有：`shasum`、`sha1sum`、`sha256sum`

### openssl
openssl也可以实现hash计算的功能
```shell
$ openssl dgst --help
Usage: dgst [options] [file...]
  file... files to digest (default is stdin)
 -help               Display this summary
 -c                  Print the digest with separating colons
 -r                  Print the digest in coreutils format
 -out outfile        Output to filename rather than stdout
 -passin val         Input file pass phrase source
 -sign val           Sign digest using private key
 -verify val         Verify a signature using public key
 -prverify val       Verify a signature using private key
 -signature infile   File with signature to verify
 -keyform format     Key file format (PEM or ENGINE)
 -hex                Print as hex dump
 -binary             Print in binary form
 -d                  Print debug info
 -debug              Print debug info
 -fips-fingerprint   Compute HMAC with the key used in OpenSSL-FIPS fingerprint
 -hmac val           Create hashed MAC with key
 -mac val            Create MAC (not necessarily HMAC)
 -sigopt val         Signature parameter in n:v form
 -macopt val         MAC algorithm parameters in n:v form or key
 -*                  Any supported digest
 -rand val           Load the file(s) into the random number generator
 -writerand outfile  Write random data to the specified file
 -engine val         Use engine e, possibly a hardware device
 -engine_impl        Also use engine given by -engine for digest operations
```

ex.
```shell
openssl dgst -sha256 exec_file
```

## 加密工具与散列
`crypt`命令
```shell
crypt [passphrase] <input_file >output_file
crypt [passphrase] <encrypted_file >output_file
```

`gpg`命令
```shell
gpg -c filename # 加密
gpg filename # 解密
```

`base64`
```shell
base64 filename > outputfile
cat file | base64 > outputfile
```

解码
```shell
base64 -d file > outputfile
cat base64_file | base64 -d > outputfile
```

opensslpasswd
```shell
opensslpasswd -1 -salt SALT_STRING PASSWORD
```

目前openssl更简单方便，功能也更齐全。

## 排序与去重
### sort
用法如下：
```shell
sort file1.txt file2.txt > sorted.txt
sort file1.txt file2.txt -o sorted.txt

sort -n file                    # 按数字排序
sort -r file                    # 逆序
sort -M months.txt              # 按月份排
sort -m sorted1 sorted2         # 合并已排序过的文件

sort file1.txt file2.txt | uniq # 去重

sort -C filename                # 检查文件是否已经进行了排序 从返回值判断

sort -nrk 1 data.txt            # k指key 指定列数
sort -nk 2,3 data.txt           # 指定范围排序

sort -z data.txt | xargs -0     # 输出用\0作为定界符
sort -bd unsorted.txt           # -b忽略前导空白行 -d指明以字典序进行排序
```

### uniq
只作用于排过序的输入，用法如下：
```shell
sort unsorted.txt | uniq
uniq sorted.txt

uniq -u sorted.txt # 只显示唯一的行
sort unsorted.txt | uniq -u

sort unsorted.txt | uniq -c # 统计各行出现的次数
sort unsorted.txt | uniq -d # 找出文件中重复的行

# -s 指定跳过前n个字符
# -w 指定用于比较的最大字符数
# -z 指定定界符为\0
uniq -z file.txt # 定界符为\0
```

## 临时文件
用法如下：
```shell
# 创建了一个临时文件，一般在/tmp目录下
filename=`mktemp`
echo $filename

# 创建了一个临时目录
dirname=`mktemp -d`
echo $dirname

# 只生成文件名，不实际创建文件或目录
tmpfile=`mktemp -u`
echo $tmpfile

# 根据模板创建临时文件名 至少3个X
mktemp test.XXX
```

## 分割文件
基本形式
```shell
split -b 10k data.file # 将data文件分割为10k的文件
split -b 10k data.file -d -a 4 # -d数字作为后缀 -a指定后缀长度
```

一般用法
```shell
split -b 10k data.file preffix # 指定前缀为preffix
split -l number data.file # 按行数分割文件
csplit server.log /SERVER/ -n 2 -s {*} -f server -b "%02d.log" ; rm server00.log
```

其他参数
```shell
# /SERVER/ 匹配行 分割
# /[REGEX]/ 文本样式
# {*} 指定分割次数 *表示不限制
# -s 静默模式
# -n 指定后缀的长度（数字个数）
# -f 指定前缀
# -b 指定后缀格式
```

## 文件名操作

### 根据扩展名切分文件名
`%`和`#`操作符

保留文件名
```shell
${VAR%.*} # 删除$VAR中从右向左匹配的通配符
${VAR%%.*} # 贪婪模式从右向左匹配最多的字符串
```

保留后缀
```shell
${VAR#*.} # 从左向右匹配字符串并删除
${VAR##*.} # 从左向右贪婪模式匹配
```

### 批量重命名和移动
- #### 脚本
```shell
count=1;
for img in `find . -iname '*.png' -o -iname '*.jpg' -type f -maxdepth 1`
do
    new=image-$count.${img##*.}
    echo "Renaming $img to $new"
    mv "$img" "$new"
    let count++
done
```

- #### rename命令操作
```shell
rename *.JPG *.jpg
rename 's/ /_/g' * # 将空格替换为_
rename 'y/A-Z/a-z/' * #大小写转换
rename 'y/a-z/A-Z/' *
```

## 拼写检查与词典操作
`aspell`命令进行拼写检查，`/usr/share/dict/`目录下有单词文件

`look`命令查找单词:
```shell
look word # 默认在/usr/share/dict/words文件内查找单词
look word dictfile
```

## 交互输入自动化
expect命令可以进行交互操作    
ex.1
```shell
echo -e "1\nhello\n" # -e 表示会解释转义序列
echo -e \xeb\x1a
expect
spawn ./test.sh # 指定需要自动化的命令
expect "Enter Number:" # 需要等待的消息
send "1\n" # 发送
expect eof # 命令交互结束
```
利用wait命令也可实现    
ex.2
```shell
PIDARRAY=()
for file in file1.iso fil2.iso
do
    md5sum $file &
    PIDARRAY+=("$!")
done
wait ${PIDARRAY[@]}
# $! 存有最近一个后台进程的PID
# wait等待输入
```
