---
layout:     post
title:      "Linux Shell Note 1"
subtitle:   "Shell 入门"
date:       2019-04-09
author:     "ChuRiver"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Linux
    - Shell
---

> 全篇碎碎念，个人的备忘录

## Index

- [基础](#基础)     
- [数学运算](#数学运算)   
- [文件重定向](#文件重定向)     
    - [自定义文件描述符](#自定义文件描述符)   
- [数组](#数组)   
    - [关联数组](#关联数组)   
- [别名](#别名)   
- [tput](#tput)   
- [stty](#stty)   
- [日期和时延](#日期和时延)   
- [循环脚本](#循环脚本)   
- [调试信息](#调试信息)  
    - [自定义调试](#自定义调试)  
- [函数和参数](#函数和参数)  
- [存储输出](#将命令的输出存入变量)  
- [子shell](#子shell)  
- [read命令](#read命令)  
- [循环执行命令](#循环执行命令)  
- [字段分隔符和迭代器](#字段分隔符和迭代器)  
- [逻辑流程](#逻辑流程)  
- [判断](#判断)  

## 基础：
基础字符
```
#   表示root
$   表示普通用户
$#  表示返回所有脚本参数的个数。
$*  以一个单字符串显示所有向脚本传递的参数
$$  脚本运行的当前进程ID号
$!  后台运行的最后一个进程的ID号
$@  与$#相同，但是使用时加引号，并在引号中返回每个参数。
$-  显示Shell使用的当前选项，与set命令功能相同。
$?  显示最后（即上一个）命令的退出状态。0表示没有错误，其他任何值表明有错误。
```

指定shell脚本文件的解释程序:
```shell
#！shebang （#--sharp、hash、mesh ！--bang）
```

bash对应的文件：

- .bashrc 一般配置（图形界面也生效）
- .bash_profile 登录shell的配置文件（ssh等）
- .bash_history 命令历史记录文件

分隔执行多个命令（不在意是否执行成功）：
```shell
cmd1 ; cmd2
```

Tips:
! 不能用在“”之间 除非使用\转义
单引号内变量替换无效

格式化输出：
```shell
printf "%-5s %-10s %-4s \n" No Name Work    # 末尾手动加换行
# %s-字符串
# %c-字符
# %d-数字
# %f-浮点数
# %-5s 左对齐 宽度为5
```

echo命令
```shell
-n # 忽略换行符
-e # 转义字符串中字符("1\t2" 转义\t)
```     
字符颜色：  重置0 黑色30 红色31 绿色32 黄色33 蓝色34 洋红35 青色36 白色37  
背景颜色：重置0 黑色40 红色41 绿色42 黄色43 蓝色44 洋红45 青色46 白色47  
eg.   
`\e[1;31m ABC \e[0m` 字符设置为红色    
`\e[1;42m ABC \e[0m` 背景设置为绿色

查看与获取变量
```shell
bash    # 只有字符串类型变量
env     # 查看环境变量
cat /proc/$PID/environ # 查看PID进程运行时环境变量
pgrep processName # 通过进程名字获取对应PID
```

变量操作
```shell
var=value   # 赋值操作
var = value # 判断相等操作
$var ${var} # 该形式可在echo和printf的双引号内输出变量的值，但在单引号内不转换
```

环境变量(从父进程中继承而来的变量)
```shell
$PATH           # 其中内容即环境变量，目录间用：分隔
export PATH="$PATH:/home/usrt/bin" # 增加环境变量
length=${#var}  # 获取字符串长度 #var返回var长度
echo $SHELL     # 或 $0 效果相同 获取当前shell类型
$UID            # 用户UID root的UID为0
$PS1            # 命令行提示字符形式（#、$等） 包括颜色、背景

export PATH=/opt/myapp/bin:$PATH

# 自定义修改PATH变量的函数 prepend
prepend() { [ -d "$2" ] && eval $1=\"$2':'\$$1\" && export $1; }
# 若path变量为空则会在变量末尾增加':'
# 格式： ${parameter:+expression}
prepend() { [ -d "$2" ] && eval $1=\"$2\$\{$1:+':'\$$1\}\" && export $1;}
```

## 数学运算
原生操作
```shell
#!/bin/bash
no1=4;
no2=5;

# let命令（不支持浮点数）     
let result=no1+no2
echo $result

# 操作
let no1++
let no1--
let no1+=6
let no1-=6

# 赋值操作
result=$[ no1 + no2]
result=$[ $no1 + 5] # no1前可不带$
result=$(( no1 + 50 ))

# expr 命令
result=`expr 3+4`
result=$(expr $no1 + 5)
```

**bc命令**    
```shell
# 管道输入即可计算
echo "4*0.56" | bc

# 构成语句计算后存入变量
no=4;
result=`echo "$no * 1.5" | bc `
echo $result

# 设定精度
echo  "scale=2;3/8 | bc"

# 进制转换
no=100
echo "obase=2;$no" | bc
no=1100100
echo "obase=10;ibase=2;$no" | bc

# 平方和平方根计算
echo "sqrt(100)" | bc
echo "10^10" | bc
```


## 文件重定向
**文件描述符**
```
STDIN  (0): 标准输入，位置 /dev/stdin, 缺省为键盘，也可以是文件或其他命令的输出
STDOUT (1): 标准输出，位置 /dev/stdout, 缺省为 Terminal，也可以是文件
STDERR (2): 标准错误，位置 /dev/stderr, 缺省为 Terminal，也可以是文件  
```

**重定向符 `>` 与 `>>`**
```shell
echo 'test' > temp.txt
echo 'another test' >> temp.txt

# >  代表重定向到stdout    
# 2> 代表重定向到stderr  

# 可将stderr和stdout合并输出，eg:
cmd &> output.txt
cmd 2>&1 output.txt
```
**tee命令**       
可从stdin中读取数据并将数据写入out.txt的同时输出到stdout，但对于stderr不起作用。
```shell
cat 1.txt | tee out.txt | cat -n
tee -a  # 追加模式
tee -   # 使用stdin作为输入
```

**重定向符 `<` 与 `<<`**   
1. 从文件中读取作为输入: `cmd < file`

2. 脚本文件内部进行重定向
```
# 可将cat的EOF两行间内容输出到log.txt文件内部   
cat<<EOF>log.txt
LOG FILE HEADER
This is a test log file
Function:System statistics
EOF
```

### 自定义文件描述符        
- **读取文件描述符**
```shell
echo this is a test line > input.txt
exec 3<input.txt    # 将3定义为文件描述符
cat <&3              # 使用3进行文件读取,使用后需要重新定义文件描述符3
```

- **写入（截断模式）**
```shell
exec 4>output.txt
echo newline >&4
cat output.txt
```

- **写入（追加模式）**
```shell
exec 5>>output.txt
echo appended line >&5
cat output.txt
```

## 数组
```shell
array_var=(1 2 3 4 5 6)
# 或
array_var[0]="test1"

echo ${array_var[0]}

index=5
echo ${array_var[$index]}

# 全部打印
echo ${array_var[*]}
echo ${array_var[@]}

# 打印数组长度
echo ${#array_var[*]}

# 遍历
for {i=0;i<10;i++} { print $i; }
for {i in array} { print array[i]; }
```

### 关联数组
```shell
#声明为关联数组
declare -A ass_array

ass_array=([index1]=val1 [index2]=val2)
ass_array[index1]=val1

# 列出索引
echo ${!array_var[*]}
echo ${!array_var[@]}
```

## 别名
设置别名：`alias new_command='command sequence'`  
取消别名：`unalias`  
忽略别名：`\command`

## tput
获取终端信息
```shell
# 打印行数列数
tput cols   
tput lines

# 打印当前终端名
tput longname

# 移动光标到坐标(100,100)处
tput cup 100 100

# 设置终端背景颜色 n ~ [0,7]
tputserb n

# 设置样式文本为粗体
tput bold

# 设置下划线的起止
tput smu1
tput rmu1

# 记录光标位置
tput sc
# 恢复光标位置
tput rc
# 删除从当前光标到行尾的所有内容
tput ed
```

## stty
禁止回显密码内容
```shell
#!/bin/sh
# Filename: password.sh
echo -e "Enter password: "
stty -echo
read password
stty echo
echo
echo Password read.
```

## 日期和时延
```shell
# 日期
date
# 纪元时
date +%s

# 转换为纪元时
date --date "Thu Nov 18 08:07:21 IST 2010" +%s

# 转换为星期
date --date "Jan 20 2001" +%A

# 按指定格式打印日期
date "+%d %B %Y"
20 May 2010

# 按指定格式打印时间
date -s "21 June 2009 11:01:22"

# 计算命令花费的时间
start=$(date +%s)
commands;
statements;
end=$(date +%s)
difference=$(( end - start ))
echo Time taken to execute commands is $difference seconds.
time<scriptpath>
```
<style>
table th{
	width: 300px;
}
</style>
日期内容|格式
:--:|:--
星期|%a(Sat)<br>%A(Saturday)
月|%b(Nov)<br>%B(November)
日|%d(31)
固定日期格式(mm/dd/yy)|%D(10/18/10)
年|%y(10)<br>%Y(2010)
小时|%I或%H(08)
分钟|%M(33)
秒|%S(10)
纳秒|%N(695208515)
纪元时|%s

## 循环脚本
```shell
echo -n Count:
tput sc
count=0;

while True;
do
    if [$count -lt 40];
    then
        let count++;
        sleep 1;
        tput rc
        tput ed
        echo -n $count;
    else exit 0;
    fi # if done
done;
```

## 调试信息
```shell
bash -x script.sh #启用跟踪调试功能

#set +-x 对部分脚本进行调试
#!/bin/bash
for i in {1..6};
do
    set -x
    echo $i
    set +x
done;
```

### 自定义调试
```bash
function DEBUG()
{
    [ "$_DEBUG" == "on" ] && $@ ||:
}
for i in {1..10}
do
    DEBUG echo $i
done


# 原理
set -x  #在执行时显示参数和命令
set +x  #禁止调试
set -v  #当命令进行读取时显示输入
set +v  #禁止打印输入

#!/bin/bash -xv
```

## 函数和参数
```shell
# 定义函数
function fname()
{
    statements;
}

fname()
{
    statements;
}

# 调用
fname ;
fname arg1 arg2 ;

# eg.
fname ()
{
    echo $1, $2;
    echo "$@"; # 列表方式
    echo "$*"; # 单个列出所有元素 每个元素都被单独作为对象或字符串
    return 0;
}
# 参数可传递给脚本并通过 script:$0 (脚本名)访问
```

**递归函数**
```shell
F() { echo $1; F hello; sleep 1;}

:(){:|:&};: # Fork 炸弹，可设置 /etc/security/limits.conf 阻止

# 导出函数
export -f fname

# 读取命令返回值
cmd;
echo $?;

# 判断命令是否执行成功
CMD="command"
$CMD
if [ $? -eq 0];
then
    echo "$CMD executed successfully"
else
    echo "$CMD executed unsuccessfully"
fi

# 向命令传递参数
command -p -v -k 1 file
command -pv -k 1 file
command -vpk 1 file
command file -pvk 1
```

## 将命令的输出存入变量
```bash
ls | cat -n > out.txt

cmd_output=$(ls | cat -n) # 或 `ls | cat -n`
echo $cmd_output
```

## 子shell
1. 子shell
```shell
    pwd;
    (cd /bin; ls);
    pwd;
    # ()内的子shell执行不会对当前shell产生影响  
```
2. 子shell保留空格和换行符    
```
out = "$(cat text.txt)"
echo $out
```

## read命令
```shell
read -n number var      # 读取number个字符存储进var
read -s var             # 无回显方式读入
read -p "Enter input:" var # 提示信息
read -t timeout var     # 在限时timeout内读取输入 单位s
read -d delim_char var  # 用定界符作为输入的结束
# eg.
read -d ":" var
```

## 循环执行命令
```shell
repeat() { while true; do $@ && return; done }
```

true为一个二进制文件，每次调用会开启进程 使用：代替会较快     
eg: `while :; do ; done`

## 字段分隔符和迭代器
```shell
# DIY分隔符
data="name,sex,rollon,location"
oldIFS=$IFS
IFS=,
for item in $data;
do
    echo item: $item
done;
IFS=$oldIFS

# 迭代产生数组
echo {1..50}
echo {a..z}
echo {A..Z}

# 迭代执行
for i in {a..z}; do actions; done;

for ((i=0;i<10;i++))
{
    commands;
}

while condition
do
    commands;
done;

x=0;
until [ $x -eq 9 ]; # 在满足条件前循环
do
    let x++;echo $x;
done;
```

## 逻辑流程
```shell
if condition;
then   
    commands;
fi

# 或
if condition;  
then  
    commands;  
else if condition;
then
    commands;  
else  
    commands;  
fi
```     
技巧：
- `[condition] && action; `condition为真则 执行action
- `[condition] || action; `condition为假，则执行action

## 判断
**变量比较**
```shell
# ! 表否定 (和变量之间无空格有其他意义)   
[ $var -eq 0 ]
[ $var -ne 0 ]
# 其他判断符：  
# -gt 大于  
# -lt 小于  
# -ge 大于等于    
# -le 小于等于    

# 逻辑与 -a:
[ $var1 -ne 0 -a $var2 -gt 2 ]
#逻辑或 -o:
[ $var1 -ne 0 -o $var2 -gt 2 ]
```

**文件相关判断**      
```shell
[ -f $file_var ]    # 若变量包含正常的文件路径或文件名，返回真   
[ -x $var ]         # 包含的文件可执行，返回真    
[ -d $var ]         # 包含的是目录，返回真  
[ -e $var ]         # 包含的文件存在，返回真     
[ -c $var ]         # 包含的是一个字符设备文件的路径，返回真     
[ -b $var ]         # 包含的是一个块设备文件的路径，返回真      
[ -w $var ]         # 包含的文件可写，返回真     
[ -r $var ]         # 包含的文件可读，返回真     
[ -L $var ]         # 包含的是一个符号链接，返回真  
```
**字符串比较**   
*最好使用双中括号*  
```shell
[[ $str1 = $str2 ]]
[[ $str1 == $str2 ]]
[[ $str1 != $str2 ]]
[[ $str1 > $str2 ]]
[[ $str1 < $str2 ]]
[[ -z $str1 ]] str1是空字符串，返回真
[[ -n $str2 ]] str2是非空字符串，返回真
if [[ -n $str1 ]] && [[ -z $str2 ]] ;
then
    commands;
fi
```

可用test命令代替      
```shell
if test -n $str1 -a -z $str2 ; then echo 'True' ; fi
```
