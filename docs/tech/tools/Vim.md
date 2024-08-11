# VIM Notes

## Session 1

#### Move

+ 单字母：hjkl，
+ 查询移动f, F, t, T, 
+ 单词移动：w, b, e, ge， 
+ 水平移动：0，^, $, g\_, 
+ 垂直移动：{，}，ctrl+D, ctrl+u
+ 搜索：/, ?, enter, n, N
+ 数字倍增：2w, 3g, 2/baby
+ gd跳转定义，gf跳转到import的文件
+ gg, G, 2gg

#### Operator

+ operator count motion
  + Operators: c(change, delete + insert), d, y, p, =, g~, >, <, =
  + ggdG, df', dt', d5w, d5j(删除接下来的五行), ggyG, gUw
  + cc, dd, yy
+ Text objects:
  + Built-in text-objects: 
    + word, sentence, paragraph, block by(), Block by {}, <, [, t for tag
  + cit, ci", cib
+ `.` commands： Repeat the last change, can be used with search command

#### Base op

+ 选中v, 复制y, 删除d, 粘贴p
+ back:u 
+ Rel: ctrl+R

## Session 2

#### Inserting Text

+ **i** for insert and **a** for append
  + i: before the cursor I:  at the beginning of line
  + a: after th ecursor A: at the end of line
+ **o**: insert a new line below
  + **O**: insert a new line above the current one
+ **gi**: puts you into insert mode at the last place you made a change
+ **CTRL-h, w, u**：delete the last character, word, line you typed

#### Selecting Text

+ Visual mode:
  + **v**: character-wise 
  + **V**: line-wise
  + **ctrl+v**: block-wise

#### Search Text

+ **/xxx**： 搜索xxx，用**.**退出
+ **gn, gN: ** 将与gn一起运行的命令执行在next match上，和**.**一起使用

#### Copy and Paste

+ `y{motion}`

  + yl(letter), yaw(word), yas(sentence), yi( (粘贴()内的所有内容)
  + yy

+ `p, P`：前者在cursor后粘贴，后者在cursor前粘贴

+ `gp, gP`： 粘贴后把cursor移到粘贴内容后

+ 常见搭配:

  + yyp, yyP,
  + ddp, ddP, dlp

+ Registers:

  + ", a-z, 0, 1-9四类register，分别存储了不同的内容
  + `"`：unnamed reg默认寄存器存储了你的copy和cut内容
  + `a-z`：named reg可以显示的指明使用来存储copy和cut
  + `0`：yank reg存储了最近一次的复制内容
  + `1-9`：cut reg存储了最近9次你进行cut的内容，如delete or change
  + 使用：
    + `"a`开头来显示指明所用的寄存器

+ In insert mode:

  + `CTRL-R{register}:`粘贴某个寄存器中的内容

+ 显示行号: `set number`
  
