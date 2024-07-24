# Linux Learning

### 1 Command

+ less/more：通常配合其他显示命令使用

  ```shell
  ps -aux | less -s
  ```

+ split：分割文件，- 表示从标准输出流中读取，-b表示按多少字节分割，-6表示按照每六行进行分割。- 有时候表示使用标准输入或输出

  ```shell
  tar cvzf - filedir | split -d -b 50m - filename
  ```

+ sort： 默认将文本文件的第一列以ASCII码的次序排出，默认是递增顺序，-r表示相反顺序，-b表示忽略前导空格，-u代表去重，-t ":"代表排序时分隔符，-n代表数值大小，-k 代表选择第几列，-h代表排序时按数值大小带单位，-o 代表输出文件，-c 代表检查是否已经排序，-C可以通过查看$?状态码来进行判断

  ```shell
  sort -b -r -u -k 5  testfile
  ```

+ cut：

  + -b m-n代表截取第m个到第n个字节，没有就代表到最前或最后，也可以截取单个字节-b 1,3,4，-n代表不要拆分多字节字符
  + -c 3 代表截取三个字符
  + -d “ ” -f 1- 代表以空格为分隔符选取所有字段，-s 代表只打印包含分隔符的字段，--output-delimiter "-"，--complement 代表打印补集

  ```shell
  cat /etc/passwd | grep test | cut -d ":" -f 1
  ```

+ sed：逐行读取并处理文件，常用脚本表达式如下

  + `sed -e '1i\a new line' xxx.txt`，演示效果，可以有多个-e脚本

  + `sed -i '${lines[0]} i\a new line' xxx.txt`，生成新文件并覆盖，-ie会自动生成源文件备份
  + `sed -ne '1p' -e'2p' xxx.txt`，代表只打印改变的内容，常与p一起使用
  + `sed -f test.sh xxx.txt`，代表脚本都在test.sh中
  + i代表行前插入，a代表行后插入，d代表删除，s代表局部替换，c代表整行覆盖，p参数代表打印

  ```shell
  sed -e 's/new/old/g'
  ```

+ awk：-v代表传递参数给脚本表达式，OFS是输出Field分隔符，RS是记录分隔符，FS是输入Field分隔符，NR表示当前已经处理的记录行数(行号)，FNR表示当前处理文件的记录数(处理每个文件时重新从1开始计数)，NF表示该行的记录数 

  ```shell
  awk -参数 '模式 {脚本}'
  awk '{print ($NF-1)}' test.log #NF代表最后一列，可以做加减，也可以直接写数字,0代表整行
  awk '{OFS="#";$1=$1;print $0}' test.log #修改输出分隔符后需要刷新数据
  awk -v OFS="#" '{$1=$1;print $0}' test.log | awk -v FS="#" 'print $NF' #-v代表传递变量进脚本
  awk '{printf "%-3s %2d\n", $1, $2}'  test.log
  awk '{if(NR==3) {print $0} else {print "No"}}' test.log #if判断,NR代表行数
  awk '/Car|Rat/ {print $0}' test.log #正则表达式/T\/V/，匹配T/V
  awk 'BEGIN{printf "%2s %2s %2s %2s\n", "姓名", "语文", "数学", "英语"} {printf "%-4s %-4d %-4d %-4d\n", $1, $2, $3, $4} END{printf "%3s:%2d%s\n", "一共有",NR,"行"}' test.log
  awk -f test.sh test.log#-f代表读取脚本文件
  ```

  + `awk 'BEGIN{OFS="#"} NR==2 {$1=$1; print $0}' test.log`

  内置函数：

  + index：查找给定的字符串的起始位置，不存在则返回0

    ```shell
    awk '/xm/ {print index($1, "m")}' test.log
    ```

  + length：返回指定字串的长度

    ```shell
    awk '/xm/ {print length($1)}' test.log
    ```

+ grep: -i代表忽略大小写，-w代表精确匹配，-e代表查找条件取多个并集，-n打印所处行数，-v代表取补集，-r代表递归查找，-l代表只打印包含的文件名，-E启用正则

  ```shell
  grep -e hello -e today testfile1.txt
  grep -E 'hello|today' testfile1.txt
  ```

+ tail/head：tail -n 5代表倒数5行，-n +5代表从正数第5行打印到最后一行，-f代表持续输出文本新增内容，-F 区别在于会持续监测日志文件是否存在。

+ uniq：去除重复行，区分大小写，-c代表打印出现次数，-d代表只打印重复的行，-u打印只出现了一次的行，-i代表忽略大小写，-f 1代表跳过1个字段，-s 1代表跳过1个字符，-w 1代表按照第一个字符去重

  ```shell
  uniq test1.txt test2.txt		
  ```

+ xargs：标准输入转化为参数，例如echo,rm,touch等，-n代表一次传递多少个参数，默认是能传递多少传递多少，按照空白字符拆分参数。-d代表分隔符，-p代表打印执行的命令并问你是否确认执行命令，-t只打印执行的命令，-I {}表示占位符，-r 代表传递参数为空就不执行

  ```shell
  echo -n (不打印换行)"hello world" | xargs -n 1 echo
  echo -n "hello#world" | xargs -I {} echo {}
  #多行合并：cat *.txt | xargs echo，\想要打印需要用单引号或双引号或加上反斜杠
  ```

+ Regex Expression

| 符号   | 含义                                                       |
| ------ | ---------------------------------------------------------- |
| ^      | ^old匹配以old单词开头的行                                  |
| \$     | old\$匹配以old单词结尾的行                                 |
| ^\$    | ^\$匹配空行                                                |
| .      | 匹配任意一个且只有一个字符，不能匹配空行                   |
| \      | 用于显示特殊字符，例如\\.                                  |
| *      | 匹配0次或者多次                                            |
| .*     | 匹配所有内容                                               |
| ^.*    | 匹配任意多个字符开头的内容                                 |
| .*$    | 匹配任意多个字符结尾的内容                                 |
| [abc]  | 匹配[]集合内的任意一个字符                                 |
| [^abc] | 匹配除了^后面的任意字符                                    |
| +      | 匹配1次或者多次                                            |
| [:/]+  | 匹配括号内的:或者\\字符一次或多次                          |
| ？     | 匹配前一个字符0次或1次                                     |
| \|     | 表示同时过滤多个字符串                                     |
| ()     | 代表分组过滤                                               |
| a{n,m} | 匹配前一个字符最少n次最多m次，a{n,}和a{,m}分别是最少和最多 |
| a{n}   | 匹配前一个字符正好n次                                      |

### 2 基本知识

**Linux中的进程运行状态**

`ps`命令用于查看进程，其有以下常用参数:

+ `ps -aux`: 显示所有用户的进程，并包括没有控制终端的进程
+ `ps -ux $(whoami) --forest`：显示特定用户的所有进程，并以树状形式展示
+ `ps -e --sort=-%mem`：显示以内存使用率排序的所有进程：
+ `ps -p 1234`：显示特定进程ID的进程信息

进程有如下几种运行状态标志:

+ `R`: 正在运行或者在运行队列中的进程
+ `S`: 正在等待某个事件（例如 I/O 操作完成）发生的进程
+ `D`: 不可中断睡眠状态的进程，通常是在等待磁盘 I/O 等关键操作完成。
+ `Z`: 表示进程已经终止，但其父进程尚未调用 wait() 系列系统调用来获取其终止状态信息，不占用系统资源
+ `T`: 已停止的进程
+ `l`：多线程（使用 CLONE_THREAD 标志的）
+ `Ss+`: 表示该进程是会话领导者，并且是前台进程组的一部分
+ `<`: 高优先级进程
+ `N`: 低优先级进程

### 3 命令实践

+ `wget`后台运行并将输出导入到文件并实时查看：
  
```sh
nohup wget https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip > wget.log 2>&1 &
tail -f 10 wget.log
```

### 4 疑难杂症

+ `source/. xxx.sh`和`sh xxx.sh`的区别：

两者都是在Linux系统中运行脚本的不同方式，但是使用时有一点最大的区别：`source/.`是在当前 shell 会话中执行脚本，影响当前环境变量，但是`bash`是启动一个新的子shell执行脚本，不会影响当前环境
