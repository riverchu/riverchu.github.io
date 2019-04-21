---
layout:     post
title:      "Linux Shell Note 4"
subtitle:   "Shell 文本操作"
date:       2019-04-21
author:     "ChuRiver"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Linux
    - Shell
---

本篇讲文本操作命令

## index
- [正则表达式](#正则表达式)
    - [非打印字符](#非打印字符)
    - [特殊字符](#特殊字符)
    - [限定符](#限定符)
    - [定位符](#定位符)
- [grep](#grep)
- [sed](#sed)
- [awk](#awk)
- [实用技巧](#实用技巧)

## 正则表达式

### 普通字符

普通字符包括没有显式指定为元字符的所有可打印和不可打印字符。这包括所有大写和小写字母、所有数字、所有标点符号和一些其他符号。

### 非打印字符

非打印字符也可以是正则表达式的组成部分。下表列出了表示非打印字符的转义序列：

字符|描述
:-:|:-
`\cx`|匹配由x指明的控制字符。例如， `\cM`匹配一个`Control-M`或回车符。
`\f`|匹配一个换页符。等价于`\x0c`和`\cL`。
`\n`|匹配一个换行符。等价于`\x0a`和`\cJ`。
`\r`|匹配一个回车符。等价于`\x0d`和`\cM`。
`\s`|匹配任何空白字符，包括空格、制表符、换页符等等。等价于`[\f\n\r\t\v]`。注意 Unicode 正则表达式会匹配全角空格符。
`\S`|匹配任何非空白字符。等价于`[^\f\n\r\t\v]`。
`\t`|匹配一个制表符。等价于`\x09`和`\cI`。
`\v`|匹配一个垂直制表符。等价于`\x0b`和`\cK`。

### 特殊字符

特殊字符指会被解析为特殊含义的字符，如`runoo*b`中的`*`，简单的说就是表示任何字符串的意思。如果要查找字符串中的`*`符号本身，则需要对`*`进行转义，即`runo\*ob`匹配`runo*ob`。

许多元字符要求在试图匹配它们时特别对待。若要匹配这些特殊字符，必须首先使字符"转义"，即，将反斜杠字符`\`放在它们前面。下表列出了正则表达式中的特殊字符：

特别字符|描述
:-:|:-
$|匹配输入字符串的结尾位置。如果设置了RegExp对象的Multiline属性，则`$`也匹配'\n'或'\r'。要匹配`$`字符本身，用`\$`。
()|标记一个子表达式的开始和结束位置。子表达式可以获取供以后使用。要匹配这些字符，用`\(`和`\)`。
*|匹配前面的子表达式零次或多次。要匹配`*`字符，用`\*`。
+|匹配前面的子表达式一次或多次。要匹配`+`字符，用`\+`。
.|匹配除换行符`\n`之外的任何单字符。要匹配`.`，用`\.`。
[|标记一个中括号表达式的开始。要匹配`[`，用`\[`。
?|匹配前面的子表达式零次或一次，或指明一个非贪婪限定符。要匹配`?`字符，用`\?`。
`\`|将下一个字符标记为或特殊字符、或原义字符、或向后引用、或八进制转义符。
^|匹配输入字符串的开始位置，除非在方括号表达式中使用，此时它表示不接受该字符集合。要匹配`^`字符本身，用`\^`。
{|标记限定符表达式的开始。要匹配`{`，用`\{`。
\||指明两项之间的一个选择。要匹配`|`，用`\|`。

### 限定符

限定符用来指定正则表达式的一个给定组件必须要出现多少次才能满足匹配。有`*`或`+`或`?`或`{n}`或`{n,}`或`{n,m}`共6种。

字符|描述
:-:|:-
*|匹配前面的子表达式零次或多次。例如，`zo*`能匹配 "z" 以及 "zoo"。* 等价于{0,}。
+|匹配前面的子表达式一次或多次。例如，`zo+`能匹配 "zo" 以及 "zoo"，但不能匹配 "z"。`+`等价于`{1,}`。
?|匹配前面的子表达式零次或一次。例如，`do(es)?`可以匹配 "do"、"does"中的 "does"、"doxy"中的"do"。`?`等价于`{0,1}`。
{n}|n是一个非负整数。匹配确定的n次。例如，`o{2}`不能匹配"Bob"中的'o'，但是能匹配"food"中的两个o。
{n,}|n是一个非负整数。至少匹配n次。例如，`o{2,}`不能匹配"Bob"中的'o'，但能匹配"foooood"中的所有o。`o{1,}`等价于`o+`。`o{0,}`则等价于`o*`。
{n,m}|m和n均为非负整数，其中n<=m。最少匹配n次且最多匹配m次。例如，`o{1,3}`将匹配"fooooood"中的前三个o。`o{0,1}`等价于 `o?`。请注意在逗号和两个数之间不能有空格。

### 定位符

定位符使您能够将正则表达式固定到行首或行尾。它们还使您能够创建这样的正则表达式，这些正则表达式出现在一个单词内、在一个单词的开头或者一个单词的结尾。

定位符用来描述字符串或单词的边界，`^`和`$`分别指字符串的开始与结束，`\b`描述单词的前或后边界，`\B`表示非单词边界。

字符|描述
:-:|:-
^|匹配输入字符串开始的位置。如果设置了RegExp对象的Multiline属性，`^`还会与`\n`或`\r`之后的位置匹配。
$|匹配输入字符串结尾的位置。如果设置了RegExp对象的Multiline属性，`$`还会与`\n`或`\r`之前的位置匹配。
\b|匹配一个单词边界，即字与空格间的位置。
\B|非单词边界匹配。

注意：
- 不能将限定符与定位符一起使用。由于在紧靠换行或者单词边界的前面或后面不能有一个以上位置，因此不允许诸如 ^* 之类的表达式。
- 若要匹配一行文本开始处的文本，请在正则表达式的开始使用 ^ 字符。
- 若要匹配一行文本的结束处的文本，请在正则表达式的结束处使用 $ 字符。

## grep

> Search for PATTERN in each FILE or standard input.
PATTERN is, by default, a basic regular expression (BRE).

用法：`grep pattern filename [--color=auto]`，filename可省略从stdin输入。

常规使用
```sh
echo -e "this is a word\nnext line." | grep word    # 从stdin接收输入
grep match_text file1 file2 file3                   # 匹配多个文件
```

正则匹配
```sh
grep -E "pattern" filename
egrep "pattern" filename
```

匹配
```sh
echo ehic is a line. | egrep -o ""          # 只输出匹配部分
grep -v match_pattern file                  # 输出不匹配部分

grep -c "" file                             # 统计匹配行数
echo -e "" | egrep -o "" | wc -l            # 统计匹配次数
grep "" -n file                             # 打印匹配行的行号 可打印多个文件匹配结果
cat file | grep "" -n

grep -b -o ""                               # 打印字节偏移量 bo不分家
grep -l "" file1 file2                      # 返回包含匹配字符串的文件名
grep -L "" file1 file2                      # 返回不包含的

grep 'text' . -R -n                         # 递归查到文件中字符串
grep -i "hello"                             # 忽略大小写
grep -e 'pattern1'  -e 'pattern2'           # 同时匹配两个样式
grep -f                                     # 从文件中读取相应的模式，可同时匹配多个模式
```

指定或排除一些文件
```sh
grep "" . -r --include *.{c,cpp}
grep "" . -r --exclude "README"
grep "" . -r --exclude-dir ""
```

-Z代表输出以\0为终结符
```sh
grep 'test' file* -lZ | xargs -0 rm
```

静默输出
```sh
# 若有匹配则返回0,无匹配则返回非0值
grep -q "" filename
grep '' -A 3 #匹配之后的三行
grep '' -B 3 #匹配行之前的三行
grep '' -C 3 #匹配行的前三行和后三行
```

## cut
> Print selected parts of lines from each FILE to standard output.

用法：
```sh
cut -s                          # 不打印不包含定界符的行

cut -f 2,4 filename             # 显示2，4列
cut -f3 --complement filename   # 做补集 不显示第3列
cut -f2 -d ':' filename         # 指定定界符

cut -c1-5 filename              # 打印1-5个字符
cut filename -c -2              # 打印前两个字符
cut filename -c1-3,6-9 --output-delimiter ',' # 指定输出定界符
```

## sed
> The sed utility reads the specified files, or the standard input if no files are specified, modifying the input as specified by a list of commands. The input is then written to the standard output.

用法：
```sh
sed 's/pattern/replace_string/' file        # 替换文本
cat file | sed 's/pattern/repalce_string/'  # stdin作为输入

sed -i 's/text/replace/' file               # 替换文件text部分为replace,并保存至文件

sed 's/pattern/replace/g' file              # 全局替换
sed 's/pattern/replace/2g' file             # 从第二个匹配的字符串开始替换
```

定界符除`/`外还可使用`:`和`|`
```sh
sed 's:text:replace:g'
sed 's|text|replace|g'

sed 's|te\xt|replace|g'                     # \x会被转义
```

连续使用多个字符处理操作
```sh
sed 'expression' | sed 'expression'
sed 'expression; expression'
sed -e 'expression' -e 'expression'
```

e.g.
```sh
sed '/^$/d' file                            # 移除空白行
sed -i 's/\d[0-9]\{3\}\b/NUMBER/g' filename # 正则表达式匹配3位数字

echo hello world | sed 's/\w\+/[&]/g'       # &为匹配的字符
sed 's/digit \([0-9]\)/\1/'                 # ()内位匹配内容 \1为第一个匹配的字符
sed 's/\([a-z]\+\) \([A-Z]\+\)/\2 \1/'

$text=hello
$echo hello world | sed "s/$text/HELLO/"    # 可使用环境变量
```

## awk

`awk`是一种文本分析工具。

用法：`awk "BEGIN{ statements } { statements } END{ end statements } " file`

e.g.1
```sh
awk ' BEGIN{ print "start" } pattern { commands } END{ print "end" } ' file
awk 'BEGIN{ i=0 } { i++ } END{ print i}' filename

echo -e "line1\nline2" | awk 'BEGIN{ print "Start" } { print } END{ print "END" } '
echo | awk '{ var1="v1"; var2="v2"; var3="v3"; print var1,var2,var3 ; } '
echo | awk '{ var1="v1"; var2="v2"; var3="v3"; print var1 "-" var2 "-" var3 ; } '
echo | awk '{ "grep root /etc/passwd" | getline cmdout; print cmdout }'
```

e.g.2
```sh
echo -e "line1 f2 f3\nline2 f4 f5\nline3 f6 f7" | awk '{
print "Line no:"NR",No of fileds:"NF, "$0="$0, "$1="$1, "$2="$2, "$3="$3
}'
```

参数|意义
:-:|:-
NR|记录数量-当前行号
NF|字段数量-当前字段数
$0|当前行的文本内容
$1|第一个字段
$2|第二个字段

其他用法:
```sh
awk '{ print $3,$2 }' file      # 改变输出顺序
awk 'END{ print NR }' file      # 改变输出信息

# 求和
seq 5 | awk 'BEGIN{ sum=0; print "Summation:" }
{ print $1"+"; sum+=$1 } END{ print "=="; print sum }'

# 定义变量
VAR=1000
echo | awk -v VARIABLE=$VAR '{ print VARIABLE }'

var1="Variable1"; var2="Variable2"
echo | awk '{ print v1,v2 }' v1=$var1 v2=$var2

awk '{ print v1,v2 }' v1=$var1 v2=$var2 filename

seq 5 | awk 'BEGIN{ getline; print "Read ahead first line", $0 } { print $0} '  # getline var 可将某行内容存入var

# 小技巧
awk 'NR < 5' # 行号小于5的行
awk 'NR==1,NR==4' #行号在1到5之间的行
awk '/linux/' # 包含linux的行
awk '!/linux/' # 不包含Linux的行

#指定定界符
awk -F: '{ print $NF }' /etc/passwd
awk 'BEGIN{ FS=":" }{ print $NF }' /etc/passwd
```

内建字符串控制函数:
```sh
length(string)                                  # 返回字符串长度
index(string, search_string)                    # 返回 search_string在字符串中出现的位置
split(string, array, delimiter)                 # 用定界符生成一个字符串列表并存入数组
substr(string, start-position, end-positon)     # 在字符串中用字符起止偏移量截取字符
sub(regex, replacement_str, string)             # 正则表达式替换第一处匹配
gsub(regex, replacement_str, string)            # 替换所有匹配
match(regex, string)                            # 检测是否有匹配结果 有返回非0，没有返回0 RSTART和RLENGTH---匹配内容起始位置和长度
```

## 实用技巧

- 统计文件中词频
```sh
#!/bin/bash
if [ $# -ne 1 ];
then
    echo "Usage: $0 filename"
    exit -1
fi
filename=$1
egrep -o "\b[[:alpha:]]+\b" $filename | \
awk '{ count[$0]++ }
END{ printf("%-14s%s\n", "Word", "Count") ;
for(ind in count)
{ printf("%-14s%d\n",ind,count[ind]); }
}'
```

- 压缩和解压缩javascript
sample.js
```js
function sign_out()
{
    $("#loading").show();
    $.get("log_in",{logout:"True"},
        function(){
            window.location="";
    });
}
```

    压缩与解压缩命令：
```sh
cat sample.js| tr -d '\n\t' | tr -s ' ' | sed 's:/\*.*\*/::g' | sed 's/ \?\([{}();,:]\) \?/\1/g'
cat sample.js| sed 's/;/;\n/g; s/{/{\n\n/g; s/}/\n\n}/g'
```

- 按列合并多个文件
```sh
paste file1 file2 file3
paste file1 file2 file3 -d "," #指定定界符
```

- 打印文件或行中的第n个单词或列
```sh
awk '{ print $5 } ' filename
ls -l | awk '{ print $1 " :   " $8 }'
```

- 打印行或样式之间的文本
```sh
# 打印M行到N行的所有文本
awk 'NR==M, NR==N' filename
sed 100 | awk 'NR==4, NR==6'
# 打印两个匹配项之间的内容，匹配项由正则表达式表示
awk '/start_pattern/, /end_pattern/' filename
```

- 逆序打印行     
```sh
# tac 逆序cat
tac file1 file2
# 默认分隔符为\n 可用-s 指定
sed 5 | tac
```
    可用awk实现：
```sh
seq 9 | awk '{ lifo[NR]=$0 }
END{ for(lno=NR;lno>-1;lno--){ print lifo[lno] }
}'
```

- 解析电子邮件地址和URL  
    正则表达式
```sh
# 匹配电子邮件
[A-Za-z0-9._]+@[A-Za-z0-9.]+\.[a-zA-Z]{2,4}
# 匹配http url
http://[a-zA-Z0-9\-\.]+\.[a-zA-Z]{2,4}
```
    在文件内移除包含某个单词的句子
```sh
# 移除包含mobile phones的句子
sed -i 's/ [^.]*mobile phones[^.]*\.//g' filename
```

- 对目录中所有文件进行文本替换
```sh
find . -name *.cpp -print0 | xargs -I {} -0 sed -i 's/Copyright/Copyleft/g' {}
find . -name *.cpp -exec sed -i 's/Copyright/Copyleft/g' \{\} \;
find . -name *.cpp -exec sed -i 's/Copyright/Copyleft/g' \{\} \+
```

- 文本切片及参数操作
```sh
echo ${var/line/REPLACE}    # 将var中的line替换为replace
echo ${variable_name:start_position:length}
echo ${var:4}               # 打印4开始的字符（从0开始计位置）
echo ${var:4:8}             # 打印4开始的8个字符
echo ${var:(-1)}            # 打印倒数第一个字符
echo ${var:(-2):2}          # 打印倒数第二个字符开始的两个字符
echo ${var:(-2):(-1)}       # 打印倒数第一个字符到倒数第二个字符
```
