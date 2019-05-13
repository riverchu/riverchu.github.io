---
layout:     post
title:      "Linux Shell Note 5"
subtitle:   "Shell web相关命令"
date:       2019-05-13
author:     "ChuRiver"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Linux
    - Shell
---

本篇介绍web相关的一些命令

## index
- [wget](#wget)
- [lynx](#lynx)
- [curl](#curl)
- [解析网站数据](#解析网站数据)
- [图片抓取器及下载工具](#图片抓取器及下载工具)
- [网页相册生成器](#网页相册生成器)
- [Twitter命令行客户端](#twitter命令行客户端)
- [基于Web后端的定义查询工具](#基于web后端的定义查询工具)
- [查找网站中的无效链接](#查找网站中的无效链接)
- [跟踪网站变动](#跟踪网站变动)
- [以POST方式发送网页并读取响应](#以post方式发送网页并读取响应)

## wget
命令格式：`wget URL`     
参数：  
    `-O` 指定输出文件名  
    `-o` 指定输出日志信息保存的文件名  
e.g.    
```bash
wget URL -O output.file -o log.file
```

尝试次数 e.g.
```bash
wget -t 5 URL # 指定尝试次数为5
wget -t 0 URL # 无限重试
```

下载速度限制
```bash
--limit-rate 20k # 单位为k 或 m
```

下载配额限制
```bash
--quote 100m # 指定下载最大配额，防止占用过多磁盘空间
-Q # 同上
```

断点续传
```bash
wget -c URL
```

复制整个网站（镜像）
```bash
wget --mirror --convert-links URL
wget -r -N -l -k DEPTH URL # 效果同上
```
参数解释：  
    `-l depth` 指定下载页面最深层级  
    `-r` 配合-l遍历所有界面  
    `-N` 使用文件的时间戳  
    `-k` 或 `--convert-links` 表示将页面内地址转为本地地址  

提供认证信息
```bash
wget --user user --password pass URL
# 或使用 --ask-password 交互手动输入密码
```

## lynx
lynx是基于命令行的浏览器
```bash
lynx URL -dump > webpage_as_text.txt
```

# curl
命令常用格式：`curl URL`  
```bash
curl URL -o file.txt # 将下载数据写入文件中，但不可指定文件名，使用-o替换则可以指定下载文件名
curl URL --progress # 用‘#’显示进度信息
curl URL --silent
curl URL -s
```

断点续传
```bash
curl URL/file -C offset # 指定偏移量继续下载
curl -C - URL # 由curl自主推断正确的续传位置
```

指定参照页字符串
```bash
curl --referer Referer_URL target_URL
```

设置cookie
```bash
curl URL --cookie "user=slynux;pass=hack"
curl URL --cookie-jar cookie_file # 从文件中读取cookie
```

设置代理字符串
```bash
curl URL --user-agent "Mozilla/5.0"
curl URL -A "Mozilla/5.0"
curl -H "Host: abc.com" -H "Accept-language: en" URL
```

限定带宽，即下载速率
```bash
curl URL --limit-rate 20k
```

指定最大下载量
```bash
curl URL --max-filesize bytes
```

认证
```bash
curl -u user:pass URL
curl -u user URL # 手动输入密码
curl --user user[:password] URL
```

只打印响应头部信息
```bash
curl -I URL
curl --head URL
```

命令行访问Gmail
```sh
#!/bin/bash
username="username"
password="password"

SHOW_COUNT=5 # 需要显示的未读邮件数量

echo
curl -u $username:$password --silent "https://mail.google.com/mail/feed/atom" | tr -d '\n' | sed 's:<entry>: \n:g' | sed -n 's/.*<title>\(.*\)<\/title.*<author><name>\([^<]*\)<\/name><email>\([^<]*\).*/From: \2 [\3] \nSubject: \1\n/p' | head -n $(( $SHOW_COUNT * 3 ))

tr -d '\n' # 移除所有换行符
sed 's:</entry>: \n:g' 将</entry>换成换行符
sed 再将title、author、email匹配取出
```

## 解析网站数据
可以由`sed`配合`awk`实现
```bash
lynx -dump -nolist http://www.johntorres.net/BoxOfficefemaleList.html |
grep -o "Rank-.*" |
sed -e 's/ *Rank-\([0-9]*\) *\(.*\)/\1\t\2/' |
sort -nk -1 > actresslist.txt
```

## 图片抓取器及下载工具
```bash
#!/bin/bash

if [ $# -ne 3];
then
    echo "Usage: $0 URL -d DIRECTORY"
    exit -1
fi #

# 遍历1-4个参数
for i in {1..4}
do
    case $1 in
    -d) shift; directory=$1; shift ;; # shift左移参数 即 $1=$2
     *) url=${url:-$1}; shift;; # 如果url不为空返回URL值，否则返回$1
    esac
done

mkdir -p $directory;
baseurl=$(echo $url | egrep -o "https?://[a-z.]+")

echo Downloading $url
curl -s $url | egrep -o "<img src=[^>]*>" | sed 's/<img src=\"\([^"]*\).*/\1/g' > /tmp/$$.list

sed -i "s|^/|$baseurl/|" /tmp/$$.list

cd $directory;

while read filename;
do
    echo Downloading $filename
    curl -s -O "$filename" --silent
done < /tmp/$$.list
```

## 网页相册生成器
```sh
#!/bin/bash
echo "Creating album.."
mkdir -p thumbs
cat <<EOF1 > index.html
<html>
<head>
<style>

body
{
    width:470px;
    margin:auto;
    border: 1px dashed grey;
    padding:10px;
}

img
{
    margin: 5px;
    border: 1px solid black;
}

</style>
</head>

<body>
<center><h1> #Album title </h1></center>
<p>

EOF1  

for img in *.jpg;
do
    convert "$img" -resize "100x" "thumbs/$img"
    echo "<a href=\"$img\" ><img src=\"thumbs/$img\" title=\"$img\" /></a>" >> index.html
done

cat <<EOF2 >> index.html

</p>
</body>
</html>
EOF2

echo Album generated to index.html
```

## Twitter命令行客户端
需下载base-oauth库
```bash
#!/bin/bash
oauth_consumer_key=consumer_key
oauth_consumer_secret=consuemr_secret
config_file=~/.$oauth_consumer_key-$oauth_consumer_secret-rc

if [[ "$1" != "read" ]] && [[ "$1" != "tweet" ]];
then
    echo -e "Usage: $0 tweet status_message\n OR\n $0 read\n"
    exit -1;
fi #

source TwitterOAuth.sh # 使用库
TO_init # 库初始化

if [ ! -e $config_file ]; then
    TO_access_token_helper
    if (( $? == 0 )); then
        echo oauth_token=${TO_ret[0]} > $config_file
        echo oauth_token_secret=${TO_ret[1]} >> $config_file
    fi #
fi #

source $config_file
if [[ "$1" = "read" ]];
then
    TO_statuses_home_timeline '' 'shantanutushar' '10'
    echo $TO_ret | sed 's/<\([a-z]\)/\n<\1/g' | grep -e '^<text>' -e '^<name>' | sed 's/<name>/\ -by /g' | sed 's$</*[a-z]*>$$g'

elif [[ "$1" = "tweet" ]];
then
    shift
    TO_statuses_update '' "$@"
    echo 'Tweeted :)'
fi
```

## 基于Web后端的定义查询工具
```bash
#!/bin/bash

apikey=API_KEY

if [ $# -ne 2 ];
then
    echo -e "Usage: $0 WORD NUMBER"
    exit -1;
fi #

curl -s URL/$1?key=$apikey | grep -o \<dt\>.*\</dt\> | sed 's$</*[a-z]*>$$g' | head -n $2 | nl
```

## 查找网站中的无效链接
```bash
#!/bin/bash

if [ $# -ne 1 ];
then
    echo -e "$Usage: $0 URL\n"
    exit 1;
fi

echo Broken links:

mkdir /tmp/$$.lynx
cd /tmp/$$.lynx

lynx -traversal $1 > /dev/null
count=0

sort -u reject.dat > links.txt

while read link;
do
    output=`curl -I $link -s | grep "HTTP/.*OK"`;
    if [[ -z $output ]];
    then
        echo $link;
        let count++
    fi #
done < links.txt

[ $count -eq 0 ] && echo No broken links found.
```

## 跟踪网站变动
```bash
#!/bin/bash
if [ $# -ne 1];
then
    echo -e "Usage: $0 URL\n"
    exit 1;
fi #

first_time=0

if [ ! -e "last.html" ];
then
    first_time=1
fi #

curl --silent $1 -o recent.html

if [ $first_time -ne 1 ];
then
    changes=$(diff -u last.html recent.html)
    if [ -n "$changes" ];
    then
        echo -e "Changes:\n"
        echo "$changes"
    else
        echo -e "\nWebsite has no changes"
    fi #
else
    echo "[First run] Archiving.."
fi #

cp recent.html last.html
```

## 以POST方式发送网页并读取响应
```bash
curl URL -d "postvar=postdata2&postvar2=postdata2"
wget URL --post-data "key1=value1&key2=value2"
```
