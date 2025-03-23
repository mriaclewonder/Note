

# Linux系统编程

## Linux命令

**1.1pwd**

> pwd:打印出当前目录的名称

**1.2cat**

>echo "hello world">filname
>
>cat filename

都可以将文本输入到指定文件中

**1.3 vim快捷键**

>读取指定文件的前十行
>
>esc -> shift+:  ->  r !head -10 filename

**1.4 man**

>man 2 stat  是系统调用
>
>man 3 printf 是库函数

### 1.tar和gzip

| **参数** | **含义**                                                  |
| :------- | :-------------------------------------------------------- |
| -c       | 生成档案文件，创建打包文件                                |
| -v       | 列出归档解档的详细过程，显示进度                          |
| -f       | 指定档案文件名称，f后面一定是.tar文件，所以必须放选项最后 |
| -t       | 列出档案中包含的文件                                      |
| -x       | 解开档案文件                                              |

```makefile
# 解除归档
tar -xvf test.tar
tar -cvf 创建归档文件
tar -xvf 解除归档文件(还原)
tar -tvf 查看归档文件内容
# 压缩
gzip test.tar
# 先打包后压缩
tar zcvf test.tar.gz 1.c 2.c 3.c 4.c把 1.c 2.c 3.c 4.c 压缩成 test.tar.gz
```

## vim-gcc-动态库

### 1.静态库和动态库简介

所谓“程序库”，简单说，就是包含了数据和执行码的文件。其不能单独执行，可以作为其它执行程序的一部分来完成某些功能。

库的存在可以使得程序模块化，可以加快程序的再编译，可以实现代码重用,可以使得程序便于升级。

程序库可分**静态库(static library)**和**共享库(shared library)**。

 ### 2.静态库制作和使用

**静态库可以认为是一些目标代码的集合，是在可执行程序运行前就已经加入到执行码中，成为执行程序的一部分。**

按照习惯,一般以“.a”做为文件后缀名。静态库的命名一般分为三个部分：

- 前缀：lib
- 库名称：自己定义即可
- 后缀：.a

所以最终的静态库的名字应该为：**libxxx.a**

**1） 静态库制作**

步骤1：将c源文件生成对应的.o文件

> deng@itcast:~/test/3static_lib$ gcc -c add.c -o add.o
> deng@itcast:~/test/3static_lib$ gcc -c sub.c -o sub.o 
>
> deng@tcast:~/test/3static_lib$ gcc -c mul.c -o mul.o 
>
> deng@itcast:~/test/3static_lib$ gcc -c div.c -o div.o

步骤2：使用打包工具ar将准备好的.o文件打包为.a文件 libtest.a

> deng@itcast:~/test/3static_lib$ ar -rcs libtest.a add.o sub.o mul.o div.o

**在使用ar工具是时候需要添加参数：rcs**

- r更新
- c创建
- s建立索引

**2）静态库使用**

静态库制作完成之后，需要将.a文件和头文件一起发布给用户。

假设测试文件为main.c，静态库文件为libtest.a头文件为head.h

编译命令：

> deng@itcast:~/test/4static_test$ gcc test.c -L./ -I./ -ltest -o test

参数说明：

- -L：表示要连接的库所在目录
- -I./: I(大写i) 表示指定头文件的目录为当前目录
- -l(小写L)：指定链接时需要的库，去掉前缀和后缀

![image-20240327190957353](C:\Users\jin\AppData\Roaming\Typora\typora-user-images\image-20240327190957353.png)

### 3.动态库制作和使用

**共享库在程序编译时并不会被连接到目标代码中，而是在程序运行是才被载入**。不同的应用程序如果调用相同的库，那么在内存里只需要有一份该共享库的实例，规避了空间浪费问题。

动态库在程序运行是才被载入，也解决了静态库对程序的更新、部署和发布页会带来麻烦。用户只需要更新动态库即可，增量更新。

按照习惯,一般以“.so”做为文件后缀名。共享库的命名一般分为三个部分：

- 前缀：lib
- 库名称：自己定义即可
- 后缀：.so

所以最终的动态库的名字应该为：libxxx.so

**1）动态库制作**

步骤一：生成目标文件，此时要加编译选项：-fPIC（fpic）

> deng@itcast:~/test/5share_lib$ gcc -fPIC -c add.c
> deng@itcast:~/test/5share_lib$ gcc -fPIC -c sub.c 
>
> deng@itcast:~/test/5share_lib$ gcc -fPIC -c mul.c 
>
> deng@itcast:~/test/5share_lib$ gcc -fPIC -c div.c

参数：-fPIC 创建与地址无关的编译程序（pic，position independent code），是为了能够在多个应用程序间共享。

步骤二：生成共享库，此时要加链接器选项: -shared（指定生成动态链接库）

> deng@itcast:~/test/5share_lib$ gcc -shared add.o sub.o mul.o div.o -o libtest.so

步骤三: 通过nm命令查看对应的函数

> deng@itcast:~/test/5share_lib$ nm libtest.so | grep add
>
>  00000000000006b0 T add 
>
> deng@itcast:~/test/5share_lib$ nm libtest.so | grep sub 
>
> 00000000000006c4 T sub

ldd查看可执行文件的依赖的动态库

> deng@itcast:~/share/3rd/2share_test$ ldd test linux-vdso.so.1 => (0x00007ffcf89d4000) libtest.so => /lib/libtest.so (0x00007f81b5612000) libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f81b5248000) /lib64/ld-linux-x86-64.so.2 (0x00005562d0cff000)

**3）如何让系统找到动态库**

- 拷贝自己制作的共享库到/lib或者/usr/lib(不能是/lib64目录)
- 临时设置LD_LIBRARY_PATH：

> export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:库路径

- 永久设置,把export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:库路径，设置到~/.bashrc或者 /etc/profile文件中

  > deng@itcast:~/share/3rd/2share_test$ vim ~/.bashrc
  >
  > 最后一行添加如下内容:
  >
  > export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/deng/share/3rd/2share_test

  使环境变量生效

  > deng@itcast:~/share/3rd/2share_test$ source ~/.bashrc deng@itcast:~/share/3rd/2share_test$ ./test
  > a + b = 20 a - b = 10

- 将其添加到 /etc/ld.so.conf文件中

  编辑/etc/ld.so.conf文件，加入库文件所在目录的路径

  运行sudo ldconfig -v，该命令会重建/etc/ld.so.cache文件

  > deng@itcast:~/share/3rd/2share_test$ sudo vim /etc/ld.so.conf
  >
  > 文件最后添加动态库路径(绝对路径)

  ![image-20240229170558798](C:\Users\jin\AppData\Roaming\Typora\typora-user-images\image-20240229170558798.png)

  > 使生效
  >
  > deng@itcast:~/share/3rd/2share_test$ sudo ldconfig -v

- 使用符号链接， 但是一定要使用绝对路径

  deng@itcast:~/test/6share_test$ sudo ln -s /home/deng/test/6share_test/libtest.so /lib/libtest.so

### 4.命令模式下的操作

**1）切换到编辑模式**

| 按键    | 功能                                   |
| :------ | :------------------------------------- |
| i       | 光标位置当前处插入文字                 |
| I       | 光标所在行首插入文字                   |
| o(字母) | 光标下一行插入文字（新行）             |
| O(字母) | 光标上一行插入文字（新行）             |
| a       | 光标位置右边插入文字                   |
| A       | 光标所在行尾插入文字                   |
| s       | 删除光标后边的字符，从光标当前位置插入 |
| S       | 删除光标所在当前行，从行首插入         |

 

**2) 光标移动**

| **按键** | **功能**                             |
| :------- | :----------------------------------- |
| Ctrl + f | 向前滚动一个屏幕                     |
| Ctrl + b | 向后滚动一个屏幕                     |
| gg       | 到文件第一行行首                     |
| G(大写)  | 到文件最后一行行首，G必须为大写      |
| mG或mgg  | 到指定行，m为目标行数                |
| 0(数字)  | 光标移到到行首（第一个字符位置）     |
| $        | 光标移到到行尾                       |
| l(小写L) | 向右移动光标                         |
| h        | 向左移动光标                         |
| k        | 向上移动光标                         |
| j        | 向下移动光标                         |
| ^        | 光标移到到行首（第一个有效字符位置） |

 

**3）复制粘贴**

| **按键** | **功能**                     |
| :------- | :--------------------------- |
| [n]yy    | 复制从当前行开始的 n 行      |
| p        | 把粘贴板上的内容插入到当前行 |

 

**4）删除**

| **按键**    | **功能**                                                     |
| :---------- | :----------------------------------------------------------- |
| [n]x        | 删除光标后 n 个字符                                          |
| [n]X        | 删除光标前 n 个字符                                          |
| D           | 删除光标所在开始到此行尾的字符                               |
| [n]dd       | 删除从当前行开始的 n 行（准确来讲，是剪切，剪切不粘贴即为删除） |
| dG          | 删除光标所在开始到文件尾的所有字符                           |
| dw          | 删除光标开始位置的字,包含光标所在字符                        |
| d0(0为数字) | 删除光标前本行所有内容,不包含光标所在字符                    |
| dgg         | 删除光标所在开始到文件首行第一个字符开始的所有字符           |

 

**5）撤销恢复**

| **按键**  | **功能**            |
| :-------- | :------------------ |
| **.**(点) | 执行上一次操作      |
| u         | 撤销前一个命令      |
| ctrl+r    | 反撤销              |
| 100 + .   | 执行上一次操作100次 |

 

**6）保存退出**

| **按键**      | **功能** |
| :------------ | :------- |
| ZZ(shift+z+z) | 保存退出 |

 

**7）查找**

| **按键** | **功能**                                   |
| :------- | :----------------------------------------- |
| /字符串  | 从当前光标位置向下查找（n，N查找内容切换） |
| ?字符串  | 从当前光标位置向上查找（n，N查找内容切换） |

 

**8）替换**

| **按键** | **功能**                                |
| :------- | :-------------------------------------- |
| r        | 替换当前字符                            |
| R        | 替换当前行光标后的字符(ESC退出替换模式) |

 

**9）可视模式**

| **按键**  | **功能**                                                     |
| :-------- | :----------------------------------------------------------- |
| v         | 按字符移动，选中文本，可配合h、j、k、l选择内容，使用d删除，使用y复制 |
| Shift + v | 行选（以行为单位）选中文本，可配合h、j、k、l选择内容，使用d删除，使用y复制 |
| Ctrl + v  | 列选 选中文本，可配合h、j、k、l选择内容，使用d删除，使用y复制 |

 

### 5.末行模式下的操作

**1）保存退出**

| **按键**    | **功能**                                     |
| :---------- | :------------------------------------------- |
| :wq         | 保存退出                                     |
| :x(小写)    | 保存退出                                     |
| :w filename | 保存到指定文件                               |
| :q          | 退出，如果文件修改但没有保存，会提示无法退出 |
| :q!         | 退出，不保存                                 |
|             |                                              |

all 表示所有

**2）替换**

| **按键**         | **功能**                               |
| :--------------- | :------------------------------------- |
| :s/abc/123/      | 光标所在行的第一个abc替换为123         |
| :s/abc/123/g     | 光标所在行的所有abc替换为123           |
| :1,10s/abc/123/g | 将第一行至第10行之间的abc全部替换成123 |
| :%s/abc/123/g    | 当前文件的所有abc替换为123             |
| :%s/abc/123/gc   | 同上，但是每次替换需要用户确认         |
| :1,$s/abc/123/g  | 当前文件的所有abc替换为123             |

 

**3）分屏**

| **按键**           | **功能**                       |
| :----------------- | :----------------------------- |
| :sp                | 当前文件水平分屏               |
| :vsp               | 当前文件垂直分屏               |
| : sp 文件名        | 当前文件和另一个文件水平分屏   |
| : vsp 文件名       | 当前文件和另一个文件垂直分屏   |
| ctrl-w-w           | 在多个窗口切换光标             |
| :wall/:wqall/:qall | 保存/保存退出/退出所有分屏窗口 |
| vim -O a.c b.c     | 垂直分屏                       |
| vim -o a.c b.c     | 水平分屏                       |

 

**4) 其它用法(扩展)**

| 按键                         | 功能                                             |
| :--------------------------- | :----------------------------------------------- |
| :!man 3 printf               | 在vim中执行命令 （q退出）                        |
| :r !ls -l                    | 将ls -l执行的结果写入当前文件中                  |
| :r /etc/passwd               | 将/etc/passwd文件中的内容写入到当前文件中        |
| :w /tmp/txt                  | 将当前文件内容写入到/tmp/txt文件中               |
| :w! /tmp/txt                 | 强制将当前文件内容写入到/tmp/txt文件中           |
| :1,10s/^/\/\//g              | 将第1行到10行行首添加// (^表示行首) /\/\转移字符 |
| :1,10s#^#//#g                | 将第1行到10行行首添加// (#可以临时代替/ 分隔)    |
| :%s/;/\r{\r\treturn0;\r}\r/g | 将;替换成{ return 0; }                           |
| :1,10s#//##g                 | 将第1行到10行行首去掉// (#可以临时代替/ 分隔)    |

 

### 6. GDB调试器

#### 6.1 GDB简介

GNU工具集中的调试器是GDB（GNU Debugger），该程序是一个交互式工具，工作在字符模式。

除gdb外，linux下比较有名的调试器还有xxgdb, ddd, kgdb, ups。



GDB主要帮忙你完成下面四个方面的功能：

1. 启动程序，可以按照你的自定义的要求随心所欲的运行程序。
2. 可让被调试的程序在你所指定的调置的断点处停住。（断点可以是条件表达式）
3. 当程序被停住时，可以检查此时你的程序中所发生的事。
4. 动态的改变你程序的执行环境。

 

#### 6.2 生成调试信息

一般来说GDB主要调试的是C/C++的程序。要调试C/C++的程序，首先在编译时，我们必须要把调试信息加到可执行文件中。使用编译器（cc/gcc/g++）的 -g 参数可以做到这一点。如：

> gcc -g hello.c -o hello
>
> g++ -g hello.cpp -o hello

如果没有-g，你将看不见程序的函数名、变量名，所代替的全是运行时的内存地址。

 

#### 6.3启动GDB

- 启动gdb：gdb program

  program 也就是你的执行文件，一般在当前目录下。



- 设置运行参数

  set args 可指定运行时参数。（如：set args 10 20 30 40 50 ）

  show args 命令可以查看设置好的运行参数。

  

- 启动程序

  run： 程序开始执行，如果有断点，停在第一个断点处

  start： 程序向下执行一行。

#### 6.5显示源代码

用list命令来打印程序的源代码。默认打印10行。

Ø list linenum： 打印第linenm行的上下文内容.

Ø list function： 显示函数名为function的函数的源程序。

Ø list： 显示当前行后面的源程序。

Ø list -： 显示当前行前面的源程序。



一般是打印当前行的上5行和下5行，如果显示函数是是上2行下8行，默认是10行，当然，你也可以定制显示的范围，使用下面命令可以设置一次显示源程序的行数。

Ø set listsize count：设置一次显示源代码的行数。

Ø show listsize： 查看当前listsize的设置。

#### 6.6 断点操作

**1）简单断点**

break 设置断点，可以简写为b

Ø b 10 设置断点，在源程序第10行

Ø b func 设置断点，在func函数入口处

 

**2）多文件设置断点**

C++中可以使用class::function或function(type,type)格式来指定函数名。

如果有名称空间，可以使用namespace::class::function或者function(type,type)格式来指定函数名。

Ø break filename:linenum -- 在源文件filename的linenum行处停住

Ø break filename:function -- 在源文件filename的function函数的入口处停住

Ø break class::function或function(type,type) -- 在类class的function函数的入口处停住

Ø break namespace::class::function -- 在名称空间为namespace的类class的function函数的入口处停住

**3）查询所有断点**

- info b
- info break
- i break
- i b

#### 6.7 条件断点

一般来说，为断点设置一个条件，我们使用if关键词，后面跟其断点条件。

设置一个条件断点：

> b test.c:8 if Value == 5

 

#### 6.8维护断点

1）delete [range...] 删除指定的断点，其简写命令为d。

- 如果不指定断点号，则表示删除所有的断点。range表示断点号的范围（如：3-7）。
- 比删除更好的一种方法是disable停止点，disable了的停止点，GDB不会删除，当你还需要时，enable即可，就好像回收站一样。

2） disable [range...] 使指定断点无效，简写命令是dis。

如果什么都不指定，表示disable所有的停止点。

3） enable [range...] 使无效断点生效，简写命令是ena。

如果什么都不指定，表示enable所有的停止点。

 

#### 6.9调试代码

- run 运行程序，可简写为r
- next 单步跟踪，函数调用当作一条简单语句执行，可简写为n
- step 单步跟踪，函数调进入被调用函数体内，可简写为s
- finish 退出进入的函数
- until 在一个循环体内单步跟踪时，这个命令可以运行程序直到退出循环体,可简写为u。
- continue 继续运行程序，停在下一个断点的位置，可简写为c
- quit 退出gdb，可简写为q

 

#### 6.10数据查看

1）查看运行时数据

print 打印变量、字符串、表达式等的值，可简写为p

p count 打印count的值

 

#### 6.11自动显示

你可以**设置一些自动显示的变量**，当程序停住时，或是在你单步跟踪时，这些变量会自动显示。相关的GDB命令是display。

- display 变量名
- info display -- 查看display设置的自动显示的信息。
- undisplay num（info display时显示的编号）
- delete display dnums… -- 删除自动显示，dnums意为所设置好了的自动显式的编号。如果要同时删除几个，编号可以用空格分隔，如果要删除一个范围内的编号，可以用减号表示（如：2-5）
- disable display dnums…
- enable display dnums…
- disable和enalbe不删除自动显示的设置，而只是让其失效和恢复。

#### 6.12 查看修改变量的值

1）ptype width -- 查看变量width的类型

type = double

2）p width -- 打印变量width 的值

$4 = 13

你可以使用set var命令来告诉GDB，width不是你GDB的参数，而是程序的变量名，如：

set var width=47 // 将变量var值设置为47

**在你改变程序变量取值时，最好都使用set var格式的GDB命令**

## Makefile

### 1. Makefile语法规则

**一条规则：**

> 目标：依赖文件列表
>
> 命令列表

Makefile基本规则三要素：

1）目标：

- 通常是要产生的文件名称，目标可以是可执行文件或其它obj文件，也可是一个动作的名称

2）依赖文件：

- 用来输入从而产生目标的文件
- 一个目标通常有几个依赖文件（可以没有）

3）命令：

- make执行的动作，一个规则可以含几个命令（可以没有）
- 有多个命令时，每个命令占一行

**举例说明：**

测试代码：

```
test:
	echo "hello world"
test:test1 test2  
	echo "test"	
test1:
	echo "test1"
test2:
	echo "test2"
```

### 2. make命令格式

make是一个命令工具，它解释Makefile 中的指令（应该说是规则）。

make命令格式：

make [ -f file ][ options ][ targets ]

1.[ -f file ]：

- make默认在工作目录中寻找名为GNUmakefile、makefile、Makefile的文件作为makefile输入文件
- -f 可以指定以上名字以外的文件作为makefile输入文件

l

2.[ options ]

- -v： 显示make工具的版本信息
- -w： 在处理makefile之前和之后显示工作路径
- -C dir：读取makefile之前改变工作路径至dir目录
- -n：只打印要执行的命令但不执行
- -s：执行但不显示执行的命令

3.[ targets ]：

- 若使用make命令时没有指定目标，则make工具默认会实现makefile文件内的第一个目标，然后退出
- 指定了make工具要实现的目标，目标可以是一个或多个（多个目标间用空格隔开）

### 3. makefile事例

### 4. 最简单的makefile

```makefile
test:test.c add.c sub.c mul.c 
    gcc test.c add.c sub.c mul.c -o test
```

缺点：效率低，修改一个文件，所有文件会被全部编译

![image-20240301205422738](C:\Users\jin\AppData\Roaming\Typora\typora-user-images\image-20240301205422738.png)

### 5. 第二个版本的makefile

```makefile
test:test.o add.o sub.o mul.o
	gcc test.o add.o sub.o mul.o -o test
test.o:test.c
	gcc -c test.c
add.o:add.c
	gcc -c add.c
sub.o:sub.c
	gcc -c sub.c
mul.o:mul.c
	gcc -c mul.c
```

![image-20240301205926408](C:\Users\jin\AppData\Roaming\Typora\typora-user-images\image-20240301205926408.png)

### 6. makefile的变量

在Makefile中使用变量有点类似于C语言中的宏定义，使用该变量相当于内容替换，使用变量可以使Makefile易于维护,修改内容变得简单变量定义及使用。

**7.1 自定义变量**

1）定义变量方法：

变量名=变量值

2）引用变量：

$(变量名)或${变量名}

3）makefile的变量名：

- makefile变量名可以以数字开头
- 变量是大小写敏感的
- 变量一般都在makefile的头部定义
- 变量几乎可在makefile的任何地方使用

```makefile
OBJS=test.o add.o sub.o mul.o
TARGET=test
${TARGET}:${OBJS}
	gcc ${OBJS} -o ${TARGET}

test.o:test.c
	gcc -c test.c
add.o:add.c
	gcc -c add.c
sub.o:sub.c
	gcc -c sub.c
mul.o:mul.c
	gcc -c mul.c
```

![image-20240301212605292](C:\Users\jin\AppData\Roaming\Typora\typora-user-images\image-20240301212605292.png)

除了使用用户自定义变量，makefile中也提供了一些变量（变量名大写）供用户直接使用，我们可以直接对其进行赋值。

>CC = gcc #arm-linux-gcc
>
>CPPFLAGS : C预处理的选项 如:-I
>
>CFLAGS: C编译器的选项 -Wall -g -c
>
>LDFLAGS : 链接器选项 -L -l

### 7. 自动变量

- $@: 表示规则中的目标
- $<: 表示规则中的第一个条件
- $^: 表示规则中的所有条件, 组成一个列表, 以空格隔开,如果这个列表中有重复的项则消除重复项。

**注意：自动变量只能在规则的命令中中使用**

```makefile
#变量
OBJS=add.o sub.o mul.o test.o add.o
TARGET=test
CC=gcc

#$@: 表示目标
#$<: 表示第一个依赖
#$^: 表示所有的依赖

${TARGET}:${OBJS}
	#${CC} ${OBJS} -o ${TARGET}
	${CC} $^ -o $@

add.o:add.c
	${CC} -c $< -o $@ 

sub.o:sub.c
	${CC} -c $< -o $@ 

mul.o:mul.c
	${CC} -c $< -o $@ 

test.o:test.c
	${CC} -c $< -o $@
	
clean:
	rm -rf ${OBJS} ${TARGET}

```

![image-20240301214246791](C:\Users\jin\AppData\Roaming\Typora\typora-user-images\image-20240301214246791.png)

### 8. 模式规则

模式规则示例:

> %.o:%.c
>
> $(CC) -c $(CFLAGS) $(CPPFLAGS) $< -o $@

 

Makefile第三个版本：

```makefile
OBJS=test.o add.o sub.o mul.o div.o
TARGET=test
$(TARGET):$(OBJS)
    gcc $(OBJS) -o $(TARGET)
%.o:%.c
    gcc -c $< -o $@
```

### 9. Makefile中的函数

makefile中的函数有很多，在这里给大家介绍两个最常用的。

> 1. wildcard – 查找指定目录下的指定类型的文件
>
> src = $(wildcard *.c) //找到当前目录下所有后缀为.c的文件,赋值给src
>
> 1. patsubst – 匹配替换
>
> obj = $(patsubst %.c,%.o, $(src)) //把src变量里所有后缀为.c的文件替换成.o

在makefile中所有的函数都是有返回值的。

Makefile第四个版本：

```makefile
SRC=$(wildcard *.c)
OBJS=$(patsubst %.c, %.o, $(SRC))
TARGET=test
$(TARGET):$(OBJS)
    gcc $(OBJS) -o $(TARGET) 

%.o:%.c
    gcc -c $< -o $@
```

### 10. Makefile中的伪目标

clean用途: 清除编译生成的中间.o文件和最终目标文件

make clean 如果当前目录下有同名clean文件，则不执行clean对应的命令，解决方案：

Ø **伪目标声明:** **.PHONY:clean**

声明目标为伪目标之后，makefile将不会该判断目标是否存在或者该目标是否需要更新

**clean命令中的殊符号：**

- “-”此条命令出错，make也会继续执行后续的命令。如:“-rm main.o”
- “@”不显示命令本身,只显示结果。如:“@echo clean done”

Makefile第五个版本：

```makefile
SRC=$(wildcard *.c)
OBJS=$(patsubst %.c, %.o, $(SRC))
TARGET=test
$(TARGET):$(OBJS)
    gcc $(OBJS) -o $(TARGET) 

%.o:%.c
    gcc -c $< -o $@
.PHONY:clean
clean:
    rm -rf $(OBJS) $(TARGET)
```

## 文件IO

### 1. 系统调用简介和实现

**1.1 什么是系统调用**

系统调用，顾名思义，说的是操作系统提供给用户程序调用的一组“特殊”接口。用户程序可以通过这组“特殊”接口来获得操作系统内核提供的服务，比如用户可以通过文件系统相关的调用请求系统打开文件、关闭文件或读写文件，可以通过时钟相关的系统调用获得系统时间或设置定时器等。

从逻辑上来说，系统调用可被看成是一个内核与用户空间程序交互的接口——它好比一个中间人，把用户进程的请求传达给内核，待内核把请求处理完毕后再将处理结果送回给用户空间。

系统服务之所以需要通过系统调用来提供给用户空间的根本原因是为了对系统进行“保护”，因为我们知道 Linux 的运行空间分为内核空间与用户空间，它们各自运行在不同的级别中，逻辑上相互隔离。

所以用户进程在通常情况下不允许访问内核数据，也无法使用内核函数，它们只能在用户空间操作用户数据，调用用户空间函数。

比如我们熟悉的“hello world”程序（执行时）就是标准的用户空间进程，它使用的打印函数 printf 就属于用户空间函数，打印的字符“hello word”字符串也属于用户空间数据。

但是很多情况下，用户进程需要获得系统服务（调用系统程序），这时就必须利用系统提供给用户的“特殊接口”——系统调用了，它的特殊性主要在于规定了用户进程进入内核的具体位置。

换句话说，用户访问内核的路径是事先规定好的，只能从规定位置进入内核，而不准许肆意跳入内核。有了这样的陷入内核的统一访问路径限制才能保证内核安全无误。我们可以形象地描述这种机制：作为一个游客，你可以买票要求进入野生动物园，但你必须老老实实地坐在观光车上，按照规定的路线观光游览。当然，不准下车，因为那样太危险，不是让你丢掉小命，就是让你吓坏了野生动物。

**1.2 系统调用的实现**

系统调用是属于操作系统内核的一部分的，必须以某种方式提供给进程让它们去调用。CPU 可以在不同的特权级别下运行，而相应的操作系统也有不同的运行级别，**用户态和内核态**。运行在内核态的进程可以毫无限制的访问各种资源，而在用户态下的用户进程的各种操作都有着限制，比如不能随意的访问内存、不能开闭中断以及切换运行的特权级别。显然，属于内核的系统调用一定是运行在内核态下，但是如何切换到内核态呢？

答案是软件中断。软件中断和我们常说的中断（硬件中断）不同之处在于，它是通过软件指令触发而并非外设引发的中断，也就是说，又是编程人员开发出的一种异常（该异常为正常的异常）。**操作系统一般是通过软件中断从用户态切换到内核态。**

### 2. 系统调用和库函数的区别

Linux 下对文件操作有两种方式：**系统调用（system call）**和**库函数调用（Library functions）**。

库函数由两类函数组成：

1）不需要调用系统调用

不需要切换到内核空间即可完成函数全部功能，并且将结果反馈给应用程序，如strcpy、bzero 等字符串操作函数。

2）需要调用系统调用

需要切换到内核空间，这类函数通过封装系统调用去实现相应功能，如 printf、fread等。

系统调用是需要时间的，程序中频繁的使用系统调用会降低程序的运行效率。当运行内核代码时，CPU工作在内核态，在系统调用发生前需要保存用户态的栈和内存环境，然后转入内核态工作。系统调用结束后，又要切换回用户态。这种环境的切换会消耗掉许多时间 。

### 3.  C库中IO函数工作流程

库函数访问文件的时候根据需要，设置不同类型的缓冲区，从而减少了直接调用 IO 系统调用的次数，提高了访问效率。

这个过程类似于快递员给某个区域（内核空间）送快递一样，快递员有两种方式送：

1）来一件快递就马上送到目的地，来一件送一件，这样导致来回走比较频繁（系统调用）

2）等快递攒着差不多后（缓冲区），才一次性送到目的地（库函数调用）

### 4.  错误处理函数

errno 是记录系统的最后一次错误代码。代码是一个int型的值，在errno.h中定义。查看错误代码errno是调试程序的一个重要方法。

当Linux C api函数发生异常时，一般会将errno全局变量赋一个整数值，不同的值表示不同的含义，可以通过查看该值推测出错的原因。

```c
#include <stdio.h>  //fopen
#include <errno.h>  //errno
#include <string.h> //strerror(errno)

int main()
{
    FILE *fp = fopen("xxxx", "r");
    if (NULL == fp)
    {
        printf("%d\n", errno);  //打印错误码
        printf("%s\n", strerror(errno)); //把errno的数字转换成相应的文字
        perror("fopen err");    //打印错误原因的字符串
    }
    return 0;
}
```

### 5.  虚拟地址空间

每个进程都会分配虚拟地址空间，在32位机器上，该地址空间为4G 。

在进程里平时所说的指针变量，保存的就是虚拟地址。当应用程序使用虚拟地址访问内存时，处理器（CPU）会将其转化成物理地址（MMU）。

MMU：将虚拟的地址转化为物理地址。

这样做的好处在于：

- 进程隔离，更好的保护系统安全运行
- 屏蔽物理差异带来的麻烦，方便操作系统和编译器安排进程地址

### 6. 文件描述符

在 Linux 的世界里，一切设备皆文件。我们可以系统调用中 I/O 的函数（I：input，输入；O：output，输出），对文件进行相应的操作（ open()、close()、write() 、read() 等）。

打开现存文件或新建文件时，系统（内核）会返回一个文件描述符，文件描述符用来指定已打开的文件。这个文件描述符相当于这个已打开文件的标号，文件描述符是非负整数，是文件的标识，操作这个文件描述符相当于操作这个描述符所指定的文件。

程序运行起来后（每个进程）都有一张文件描述符的表，标准输入、标准输出、标准错误输出设备文件被打开，对应的文件描述符 0、1、2 记录在表中。程序运行起来后这三个文件描述符是默认打开的。

>```
>#define STDIN_FILENO  0 //标准输入的文件描述符
>```
>
>```
>#define STDOUT_FILENO 1 //标准输出的文件描述符
>```
>
>```
>#define STDERR_FILENO 2 //标准错误的文件描述符
>```

### 7.  常用文件IO函数

**7.1 open**

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
功能：
    打开文件，如果文件不存在则可以选择创建。
参数：
    pathname：文件的路径及文件名
    flags：打开文件的行为标志，必选项 O_RDONLY, O_WRONLY, O_RDWR
    mode：这个参数，只有在文件不存在时有效，指新建文件时指定文件的权限
返回值：
    成功：成功返回打开的文件描述符
    失败：-1
```

必选项：

| **取值** | **含义**               |
| :------- | :--------------------- |
| O_RDONLY | 以只读的方式打开       |
| O_WRONLY | 以只写的方式打开       |
| O_RDWR   | 以可读、可写的方式打开 |

可选项，和必选项按位或起来

| **取值**   | **含义**                                                   |
| :--------- | :--------------------------------------------------------- |
| O_CREAT    | 文件不存在则创建文件，使用此选项时需使用mode说明文件的权限 |
| O_EXCL     | 如果同时指定了O_CREAT，且文件已经存在，则出错              |
| O_TRUNC    | 如果文件存在，则清空文件内容                               |
| O_APPEND   | 写文件时，数据添加到文件末尾                               |
| O_NONBLOCK | 对于设备文件, 以O_NONBLOCK方式打开可以做非阻塞I/O          |

**7.2 close**

```c
#include <unistd.h>
int close(int fd);
功能：
    关闭已打开的文件
参数：
    fd : 文件描述符，open()的返回值
返回值：
    成功：0
    失败： -1, 并设置errno
```

**7.3 read**

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
功能：
    把指定数目的数据读到内存（缓冲区）
参数：
    fd : 文件描述符
    buf : 内存首地址
    count : 读取的字节个数
返回值：
    成功：实际读取到的字节个数
    失败： - 1
```

**7.4 write**

```c
#include <unistd.h>
ssize_t write(int fd, const void *buf, size_t count);
功能：
    把指定数目的数据写到文件（fd）
参数：
    fd :  文件描述符
    buf : 数据首地址
    count : 写入数据的长度（字节）
返回值：
    成功：实际写入数据的字节个数
    失败： - 1
```

**阻塞和非阻塞的概念**

读常规文件是不会阻塞的，不管读多少字节，read一定会在有限的时间内返回。

从终端设备或网络读则不一定，如果从终端输入的数据没有换行符，调用read读终端设备就会阻塞，如果网络上没有接收到数据包，调用read从网络读就会阻塞，至于会阻塞多长时间也是不确定的，如果一直没有数据到达就一直阻塞在那里。

同样，写常规文件是不会阻塞的，而向终端设备或网络写则不一定。

【注意】阻塞与非阻塞是对于文件而言的，而不是指read、write等的属性。

**7.5 lseek**

```c
#include <sys/types.h>
#include <unistd.h>

off_t lseek(int fd, off_t offset, int whence);
功能：
    改变文件的偏移量
参数：
    fd：文件描述符
    offset：根据whence来移动的位移数（偏移量），可以是正数，也可以负数，如果正数，则相对于whence往右移动，如果是负数，则相对于whence往左移动。如果向前移动的字节数超过了文件开头则出错返回，如果向后移动的字节数超过了文件末尾，再次写入时将增大文件尺寸。

    whence：其取值如下：
        SEEK_SET：从文件开头移动offset个字节
        SEEK_CUR：从当前位置移动offset个字节
        SEEK_END：从文件末尾移动offset个字节
返回值：
    若lseek成功执行, 则返回新的偏移量
    如果失败， 返回-1
```

所有打开的文件都有一个当前文件偏移量(current file offset)，以下简称为 cfo。cfo 通常是一个非负整数，用于表明文件开始处到文件当前位置的字节数。

读写操作通常开始于 cfo，并且使 cfo 增大，增量为读写的字节数。文件被打开时，cfo 会被初始化为 0，除非使用了 O_APPEND 。

### 8.  文件操作相关函数

### 8.1 stat函数(重点)

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

int stat(const char *path, struct stat *buf);
int lstat(const char *pathname, struct stat *buf);
功能：
    获取文件状态信息
    stat和lstat的区别：
        当文件是一个符号链接时，lstat返回的是该符号链接本身的信息；
        而stat返回的是该链接指向的文件的信息。
参数：
    path：文件名
    buf：保存文件信息的结构体
返回值：
    成功： 0
    失败: -1
```

**测试程序**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

#if（条件编译）
#if的使用和if else的使用非常相似，一般使用格式如下

#if 整型常量表达式1
程序段1
#elif 整型常量表达式2
程序段2
#else
程序段3
#endif

执行起来就是，如果整形常量表达式为真，则执行程序段1，否则继续往后判断依次类推（注意是整形常量表达式），最后#endif是#if的结束标志
    
#if 0
struct stat
{
               dev_t     st_dev;         /* ID of device containing file */
               ino_t     st_ino;         /* Inode number */
               mode_t    st_mode;        /* File type and mode */
               nlink_t   st_nlink;       /* Number of hard links */
               uid_t     st_uid;         /* User ID of owner */
               gid_t     st_gid;         /* Group ID of owner */
               dev_t     st_rdev;        /* Device ID (if special file) */
               off_t     st_size;        /* Total size, in bytes */
               blksize_t st_blksize;     /* Block size for filesystem I/O */
               blkcnt_t  st_blocks;      /* Number of 512B blocks allocated */

               /* Since Linux 2.6, the kernel supports nanosecond
                  precision for the following timestamp fields.
                  For the details before Linux 2.6, see NOTES. */

               struct timespec st_atim;  /* Time of last access */
               struct timespec st_mtim;  /* Time of last modification */
               struct timespec st_ctim;  /* Time of last status change */

           #define st_atime st_atim.tv_sec      /* Backward compatibility */
           #define st_mtime st_mtim.tv_sec
           #define st_ctime st_ctim.tv_sec
};

#endif


int main(void)
{
	int ret = -1;
	struct stat s;

	// Obtain the specified file information
	ret = stat("txt",&s);
	if(-1 == ret)
	{
		perror("stat");
		return 1;
		
	}

	// File attribute information
	printf("st_dev: %lu\n",s.st_dev);
	printf("st_ino: %ld\n",s.st_ino);
	printf("st_nlink: %lu\n",s.st_nlink);
	printf("st_uid: %d\n",s.st_uid);
	printf("st_gid: %d\n",s.st_gid);
	printf("st_size: %ld\n ",s.st_size);


	return 0;
}

```

![image-20240303142142654](C:\Users\jin\AppData\Roaming\Typora\typora-user-images\image-20240303142142654.png)

**用stat来实现获取文件类型**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>


int show_file_type(struct stat *s)
{
	switch(s->st_mode & S_IFMT)
	{

		case S_IFREG:
			printf("This is normal document\n");
			break;
		case S_IFDIR:
			printf("Directory\n");
			break;
		case S_IFIFO:
			printf("FIFO/pipe\n");
			break;
		case S_IFLNK:
			printf("Symlink\n");
			break;
		case S_IFSOCK:
			printf("Socket\n");
			break;
		case S_IFCHR:
			printf("Character device\n");
			break;
		case S_IFBLK:
			printf("Block device\n");
			break;
		default:
			printf("unknown?\n");
			break;
	
	}
		return 0;
}


int main(int argc,char **argv)
{
	int ret = -1;
	struct stat s;
	
	// Fault-tolerant judgment
	if(2 != argc)
	{
		printf("usage: ./a.out filename\n");
		return 1;
	}

	// Obtain the specifide file information
	ret = stat(argv[1],&s);
	if(-1 == ret)
	{
		perror("stat");
		return 1;
	}

	show_file_type(&s);

	return 0;
}




```



![image-20240303152736545](C:\Users\jin\AppData\Roaming\Typora\typora-user-images\image-20240303152736545.png)

### 8.2文件描述符复制(重点)

dup() 和 dup2() 是两个非常有用的系统调用，都是用来复制一个文件的描述符，使新的文件描述符也标识旧的文件描述符所标识的文件。

这个过程类似于现实生活中的配钥匙，钥匙相当于文件描述符，锁相当于文件，本来一个钥匙开一把锁，相当于，一个文件描述符对应一个文件，现在，我们去配钥匙，通过旧的钥匙复制了一把新的钥匙，这样的话，旧的钥匙和新的钥匙都能开启这把锁。

对比于 dup(), dup2() 也一样，通过原来的文件描述符复制出一个新的文件描述符，这样的话，原来的文件描述符和新的文件描述符都指向同一个文件，我们操作这两个文件描述符的任何一个，都能操作它所对应的文件。

```c

#include <unistd.h>

int dup(int oldfd);
功能：
    通过 oldfd 复制出一个新的文件描述符，新的文件描述符是调用进程文件描述符表中最小可用的文件描述符，最终 oldfd 和新的文件描述符都指向同一个文件。
参数：
    oldfd : 需要复制的文件描述符 oldfd
返回值：
        成功：新文件描述符
        失败： -1
#include <unistd.h>

int dup2(int oldfd, int newfd);
功能：
    通过 oldfd 复制出一个新的文件描述符 newfd，如果成功，newfd 和函数返回值是同一个返回值，最终 oldfd 和新的文件描述符 newfd 都指向同一个文件。
参数：
    oldfd : 需要复制的文件描述符
    newfd : 新的文件描述符，这个描述符可以人为指定一个合法数字（0 - 1023），如果指定的数字已经被占用（和某个文件有关联），此函数会自动关闭 close() 断开这个数字和某个文件的关联，再来使用这个合法数字。
返回值：
    成功：返回 newfd
    失败：返回 -1
```

**事例**

```c
#include <stdio.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <fcntl.h>

int main(void)
{
	int fd = -1;
	int newfd = -1;

	fd=open("txt",O_RDWR | O_CREAT , 0664);
	if(-1 == fd)
	{
		perror("open");
		return 1;
	}
	printf("fd=%d\n",fd);

	// 1. newfd=dup(fd);
    // 2
	newfd = 2;
	dup2(fd,newfd);

	if(-1 == newfd)
	{
		perror("dup");
		return 1;
	}
	printf("newfd=%d\n",newfd);


	// write
	write(fd,"ABCDEFG",7);
	write(newfd,"1234567",7);

	// close
	close(fd);
	close(newfd);

	return 0;
}
```

### 8.3fcntl

```c
#include <unistd.h>
#include <fcntl.h>

int fcntl(int fd, int cmd, ... /* arg */);
功能：改变已打开的文件性质，fcntl针对描述符提供控制。
参数：
    fd：操作的文件描述符
    cmd：操作方式
    arg：针对cmd的值，fcntl能够接受第三个参数int arg。
返回值：
    成功：返回某个其他值
    失败：-1
fcntl函数有5种功能：

1) 复制一个现有的描述符（cmd=F_DUPFD）

2) 获得／设置文件描述符标记(cmd=F_GETFD或F_SETFD)

3) 获得／设置文件状态标记(cmd=F_GETFL或F_SETFL)

4) 获得／设置异步I/O所有权(cmd=F_GETOWN或F_SETOWN)

5) 获得／设置记录锁(cmd=F_GETLK, F_SETLK或F_SETLKW)
```



```c
#include <stdio.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <fcntl.h>

int main(void)
{
	int fd = -1;
	int newfd = -1;
	int ret = -1;

	fd = open("txt", O_WRONLY | O_CREAT , 0664);
	if(-1 == fd)
	{
		perror("open");
		return 1;
	}
	printf("fd=%d\n",fd);

	newfd = fcntl(fd, F_DUPFD ,0);
	if(-1 == newfd)
	{
		perror("fcntl");
		return 1;
	}
	printf("newfd=%d\n",newfd);

	// write
	write(fd,"123456789",9);
	write(newfd,"AAAAAA",6);

	// close
	close(fd);
	close(newfd);

	return 0;
}
```

```c

#include <stdio.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <fcntl.h>

int main(void)
{
	int fd = -1;
	int newfd = -1;
	int ret = -1;

	fd = open("txt",O_WRONLY | O_CREAT ,0064);
	if(-1 == fd)
	{
		perror("open");
		return 1;
	}
	printf("fd=%d\n",fd);

	// 获取文件标记
	ret = fcntl(fd,F_GETFL);
    	if(-1 == ret) {
		perror("fcntl");
		return 1;
	}

	if(ret & O_APPEND)
	{
		printf("before\n");
	}
	else
	{
		printf("after\n");
	}
	
	// 设置文件标记
	ret = ret | O_APPEND;
	fcntl(fd,F_SETFL,ret);

	// 获取文件标记
	
	
	ret = fcntl(fd,F_GETFL);
    	if(-1 == ret) {
		perror("fcntl");
		return 1;
	}

	if(ret & O_APPEND)
	{
		printf("before1\n");
	}
	else
	{
		printf("after2\n");
	}
	return 0;
}
```

### 8.4getcwd

```c
5.1 getcwd函数
#include <unistd.h>
​
char *getcwd(char *buf, size_t size);
功能：获取当前进程的工作目录
参数：
    buf : 缓冲区，存储当前的工作目录
    size : 缓冲区大小
返回值：
    成功：buf中保存当前进程工作目录位置
    失败：NULL
 

5.2 chdir函数
#include <unistd.h>
​
int chdir(const char *path);
功能：修改当前进程(应用程序)的路径
参数：
    path：切换的路径
返回值：
    成功：0
    失败：-1
 

5.3 opendir函数
#include <sys/types.h>
#include <dirent.h>
​
DIR *opendir(const char *name);
功能：打开一个目录
参数：
    name：目录名
返回值：
    成功：返回指向该目录结构体指针
    失败：NULL
​
 

5.4 closedir函数
#include <sys/types.h>
#include <dirent.h>
​
int closedir(DIR *dirp);
功能：关闭目录
参数：
    dirp：opendir返回的指针
返回值：
    成功：0
    失败：-1
​
 

5.5 readdir函数
#include <dirent.h>
​
struct dirent *readdir(DIR *dirp);
功能：读取目录
参数：
    dirp：opendir的返回值
返回值：
    成功：目录结构体指针
    失败：NULL
​
相关结构体说明：

struct dirent
{
    ino_t d_ino;                  // 此目录进入点的inode
    off_t d_off;                    // 目录文件开头至此目录进入点的位移
    signed short int d_reclen;      // d_name 的长度, 不包含NULL 字符
    unsigned char d_type;           // d_type 所指的文件类型 
    char d_name[256];               // 文件名
};
```

d_type文件类型说明：

| **取值**   | **含义** |
| :--------- | :------- |
| DT_BLK     | 块设备   |
| DT_CHR     | 字符设备 |
| DT_DIR     | 目录     |
| DT_LNK     | 软链接   |
| DT_FIFO    | 管道     |
| DT_REG     | 普通文件 |
| DT_SOCK    | 套接字   |
| DT_UNKNOWN | 未知     |

```c

#include <stdio.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <fcntl.h>

int main(void)
{
	int ret = -1;
	char buf[128];

	// 获取当前工作目录
	memset(buf,0,128);
	if(NULL == getcwd(buf,128))
	{
		perror("getcwd");
		return 1;
	}
	printf("buf:%s\n",buf);

	// 修改当前目录
	ret = chdir("/home/jin");
	if(-1 == ret)
	{
		perror("chdir");
		return 1;
	}

	// 获取当前目录
	memset(buf,0,128);
	if(NULL == getcwd(buf,128))
	{
		perror("getcwd");
		return 1;
	}
	printf("buf:%s\n",buf);

	return 0;
}
```

### 8.5readdir

```c
#include <stdio.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <fcntl.h>
#include <dirent.h>


int main(void)
{
	// open dir
	DIR *dir = NULL;
	struct dirent *d = NULL;

	dir = opendir("/home/jin/Desktop/System programming");

	if(dir == NULL)
	{
		perror("opendir");
		return 1;
	}
	
	printf("打开目录成功.......\n");

	while(1)
	{
		d = readdir(dir);
		if(d == NULL)
		{
			break;
		}
		printf("d_type:%hu  d_name:%s\n",d->d_type,d->d_name);
	
	}
    
	// close dir
	closedir(dir);
	return 0;
}
```

## 进程控制

### 1.进程和程序

代码通过编译器编译之后就变成了 **可执行程序**，在可执行程序运行起来之后（**没有结束之前**），他成为了一个进程

**程序是存放在存储介质上的一个可执行文件，而进程是程序执行的过程**。进程的状态是变化的，其包括进程的创建、调度和消亡。**程序是静态的，进程是动态的**。

![1527992375886](file:///C:/Users/jin/Desktop/%E7%AC%AC4%E9%98%B6%E6%AE%B5-Linux%E9%AB%98%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E5%8F%91/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%AC%AC05%E5%A4%A9%EF%BC%88%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%EF%BC%89/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/assets/1527992375886.png)

在 Linux 系统中，操作系统是通过进程去完成一个一个的任务，**进程是管理事务的基本单元。**

进程拥有自己独立的**处理环境**（如：当前需要用到哪些环境变量，程序运行的目录在哪，当前是哪个用户在运行此程序等）和**系统资源**（如：处理器 CPU 占用率、存储器、I/O设备、数据、程序）。

### 2.单道、多道程序设计(了解)

**单道程序设计**

所有进程一个一个排队执行。若A阻塞，B只能等待，即使CPU处于空闲状态。而在人机交互时阻塞的出现是必然的。所有这种模型在系统资源利用上及其不合理，在计算机发展历史上存在不久，大部分便被淘汰了。

**多道程序设计**

在计算机内存中同时存放几道相互独立的程序，它们在管理程序控制之下，相互穿插的运行。多道程序设计必须有硬件基础作为保证。

在计算机中**时钟中断**即为多道程序设计模型的理论基础。并发时，任意进程在执行期间都不希望放弃cpu。因此系统需要一种强制让进程让出cpu资源的手段。时钟中断有硬件基础作为保障，对进程而言不可抗拒。 操作系统中的中断处理函数，来负责调度程序执行。

在多道程序设计模型中，多个进程轮流使用CPU (分时复用CPU资源)。而当下常见CPU为纳秒级，1秒可以执行大约10亿条指令。由于人眼的反应速度是毫秒级，所以看似同时在运行。

> 1s = 1000ms
>
> 1ms = 1000us
>
> 1us = 1000ns
>
> 1s = 1000000000ns

### 3.并发和并行

**并行(parallel)：**指在同一时刻，有多条指令在多个处理器上同时执行。

**并发(concurrency)：**指在同一时刻只能有一条指令执行，但多个进程指令被快速的轮换执行，使得在宏观上具有多个进程同时执行的效果，但在微观上并不是同时执行的，只是把时间分成若干段，使多个进程快速交替的执行。

举例说明：

- 并行是两个队列**同时**使用两台咖啡机
- 并发是两个队列**交替**使用一台咖啡机

### 4.MMU(了解)

MMU是Memory Management Unit的缩写，中文名是[内存管理](https://baike.baidu.com/item/内存管理)单元，它是[中央处理器](https://baike.baidu.com/item/中央处理器)（CPU）中用来管理[虚拟存储器](https://baike.baidu.com/item/虚拟存储器)、物理存储器的控制线路，同时也负责[虚拟地址](https://baike.baidu.com/item/虚拟地址)映射为[物理地址](https://baike.baidu.com/item/物理地址)，以及提供硬件机制的内存访问授权，多用户多进程操作系统。

![1527907116958](file:///C:/Users/jin/Desktop/%E7%AC%AC4%E9%98%B6%E6%AE%B5-Linux%E9%AB%98%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E5%8F%91/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%AC%AC05%E5%A4%A9%EF%BC%88%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%EF%BC%89/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/assets/1527907116958.png)

### 5.PCB(进程控制块)

进程运行时，内核为进程每个进程分配一个PCB（进程控制块），维护进程相关的信息，Linux内核的进程控制块是task_struct结构体。

在 /usr/src/linux-headers-xxx/include/linux/sched.h 文件中可以查看struct task_struct 结构体定义：

> deng@itcast:~/share$ vim /usr/src/linux-headers-4.10.0-28/include/linux/sched.h

![1527907587827](file:///C:/Users/jin/Desktop/%E7%AC%AC4%E9%98%B6%E6%AE%B5-Linux%E9%AB%98%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E5%8F%91/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%AC%AC05%E5%A4%A9%EF%BC%88%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%EF%BC%89/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/assets/1527907587827.png)

其内部成员有很多，我们掌握以下部分即可：

- 进程id。系统中每个进程有唯一的id，在C语言中用pid_t类型表示，其实就是一个非负整数。
- 进程的状态，有就绪、运行、挂起、停止等状态。
- 进程切换时需要保存和恢复的一些CPU寄存器。
- 描述虚拟地址空间的信息。
- 描述控制终端的信息。
- 当前工作目录（Current Working Directory）。
- umask掩码。
- 文件描述符表，包含很多指向file结构体的指针。
- 和信号相关的信息。
- 用户id和组id。
- 会话（Session）和进程组。
- 进程可以使用的资源上限（Resource Limit）。

### 6.进程的状态（重点）

进程状态反映进程执行过程的变化。这些状态随着进程的执行和外界条件的变化而转换。

在三态模型中，进程状态分为三个基本状态，即**运行态，就绪态，阻塞态**。

在五态模型中，进程分为**新建态、终止态，运行态，就绪态，阻塞态**。

![1527908066890](file:///C:/Users/jin/Desktop/%E7%AC%AC4%E9%98%B6%E6%AE%B5-Linux%E9%AB%98%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E5%8F%91/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%AC%AC05%E5%A4%A9%EF%BC%88%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%EF%BC%89/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/assets/1527908066890.png)

**①TASK_RUNNING：**进程正在被CPU执行。当一个进程刚被创建时会处于TASK_RUNNABLE，表示己经准备就绪，正等待被调度。

**②TASK_INTERRUPTIBLE（可中断）：**进程正在睡眠（也就是说它被阻塞）等待某些条件的达成。一旦这些条件达成，内核就会把进程状态设置为运行。处于**此状态的进程也会因为接收到信号而提前被唤醒**，**比如给一个TASK_INTERRUPTIBLE状态的进程发送SIGKILL信号，这个进程将先被唤醒（进入TASK_RUNNABLE状态），然后再响应SIGKILL信号而退出**（变为TASK_ZOMBIE状态），并不会从TASK_INTERRUPTIBLE状态直接退出。

**③TASK_UNINTERRUPTIBLE（不可中断）：**处于等待中的进程，待资源满足时被唤醒，**但不可以由其它进程通过信号或中断唤醒**。由于不接受外来的任何信号，**因此无法用kill杀掉这些处于该状态的进程**。而**TASK_UNINTERRUPTIBLE状态存在的意义就在于**，**内核的某些处理流程是不能被打断的**。如果响应异步信号，程序的执行流程中就会被插入一段用于处理异步信号的流程，于是原有的流程就被中断了，这可能使某些设备陷入不可控的状态。处于TASK_UNINTERRUPTIBLE状态一般总是非常短暂的，通过ps命令基本上不可能捕捉到。　

**④TASK_ZOMBIE（僵死）：**表示进程已经结束了，**但是其父进程还没有调用wait4或waitpid()来释放进程描述符**。为了父进程能够获知它的消息，子进程的进程描述符仍然被保留着。一旦父进程调用了wait4()，进程描述符就会被释放。

**⑤TASK_STOPPED（停止）：**进程停止执行。当进程接收到SIGSTOP，SIGTSTP，SIGTTIN，SIGTTOU等信号的时候。此外，**在调试期间接收到任何信号**，都会使进程进入这种状态。**当接收到SIGCONT信号，会重新回到TASK_RUNNABLE**。

### 7.PS 查看进程状态

进程是一个具有一定独立功能的程序，它是操作系统动态执行的基本单元。

ps命令可以查看进程的详细状况，常用选项(选项可以不加“-”)如下：

| **选项** | **含义**                                 |
| :------- | :--------------------------------------- |
| -a       | 显示终端上的所有进程，包括其他用户的进程 |
| -u       | 显示进程的详细状态                       |
| -x       | 显示没有控制终端的进程                   |
| -w       | 显示加宽，以便显示更多的信息             |
| -r       | 只显示正在运行的进程                     |

ps aux

ps ef

ps -a

![2016-05-29_230200](file:///C:/Users/jin/Desktop/%E7%AC%AC4%E9%98%B6%E6%AE%B5-Linux%E9%AB%98%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E5%8F%91/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%AC%AC01%E5%A4%A9%EF%BC%88%E5%9F%BA%E6%9C%AC%E5%91%BD%E4%BB%A4%EF%BC%89/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/assets/clip_image002-1527436564167.jpg)

stat中的参数意义如下：

| **参数** | **含义**                               |
| :------- | :------------------------------------- |
| D        | 不可中断 Uninterruptible（usually IO） |
| R        | 正在运行，或在队列中的进程             |
| S(大写)  | 处于休眠状态                           |
| T        | 停止或被追踪                           |
| Z        | 僵尸进程                               |
| W        | 进入内存交换（从内核2.6开始无效）      |
| X        | 死掉的进程                             |
| <        | 高优先级                               |
| N        | 低优先级                               |
| s        | 包含子进程                             |
| +        | 位于前台的进程组                       |

### 8.top(动态显示运行中的进程)

top命令能够在运行后，在指定的时间间隔更新显示信息。可以在使用top命令时加上-d 来指定显示信息更新的时间间隔。

在top命令执行后，可以按下按键得到对显示的结果进行排序：

| **按键** | **含义**                           |
| :------- | :--------------------------------- |
| M        | 根据内存使用量来排序               |
| P        | 根据CPU占有率来排序                |
| T        | 根据进程运行时间的长短来排序       |
| U        | 可以根据后面输入的用户名来筛选进程 |
| K        | 可以根据后面输入的PID来杀死进程。  |
| q        | 退出                               |
| h        | 获得帮助                           |

### 9. kill

kill命令指定进程号的进程，需要配合 ps 使用。

使用格式：

kill [-signal] pid

信号值从0到15，其中9为绝对终止，可以处理一般信号无法终止的进程。

**kill 9133** ：9133 为应用程序所对应的进程号

![20150318112100659](file:///C:/Users/jin/Desktop/%E7%AC%AC4%E9%98%B6%E6%AE%B5-Linux%E9%AB%98%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E5%8F%91/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%AC%AC01%E5%A4%A9%EF%BC%88%E5%9F%BA%E6%9C%AC%E5%91%BD%E4%BB%A4%EF%BC%89/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/assets/clip_image002-1527436647473.jpg)

有些进程不能直接杀死，这时候我们需要加一个参数“ -9 ”，“ -9 ” 代表强制结束：

![20150318112401325](file:///C:/Users/jin/Desktop/%E7%AC%AC4%E9%98%B6%E6%AE%B5-Linux%E9%AB%98%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E5%8F%91/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%AC%AC01%E5%A4%A9%EF%BC%88%E5%9F%BA%E6%9C%AC%E5%91%BD%E4%BB%A4%EF%BC%89/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/assets/clip_image002-1527436664857.jpg)

### 10.killall

通过进程的名字来杀死进程

>  killall -9 sleep

### 11.进程号和相关函数

每个进程都由一个进程号来标识，其类型为 pid_t（整型），进程号的范围：0～32767。进程号总是唯一的，但进程号可以重用。当一个进程终止后，其进程号就可以再次使用。

![1527994838155](file:///C:/Users/jin/Desktop/%E7%AC%AC4%E9%98%B6%E6%AE%B5-Linux%E9%AB%98%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E5%8F%91/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%AC%AC05%E5%A4%A9%EF%BC%88%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%EF%BC%89/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/assets/1527994838155.png)

**进程号（PID）**：

标识进程的一个非负整型数。

**父进程号（PPID）**：

任何进程（ 除 init 进程）都是由另一个进程创建，该进程称为被创建进程的父进程，对应的进程号称为父进程号（PPID）。如，A 进程创建了 B 进程，A 的进程号就是 B 进程的父进程号。

**进程组号（PGID）**：

进程组是一个或多个进程的集合。他们之间相互关联，进程组可以接收同一终端的各种信号，关联的进程有一个进程组号（PGID） 。这个过程有点类似于 QQ 群，组相当于 QQ 群，各个进程相当于各个好友，把各个好友都拉入这个 QQ 群里，主要是方便管理，特别是通知某些事时，只要在群里吼一声，所有人都收到，简单粗暴。但是，这个进程组号和 QQ 群号是有点区别的，默认的情况下，当前的进程号会当做当前的进程组号。

### 12.getpid getppid getgpid

```c
#include <sys/types.h>
#include <unistd.h>
pid_t getpid(void);获取本进程号（PID）

pid_t getppid(void);获取调用此函数的进程的父进程号（PPID）

pid_t getpgid(pid_t pid);获取进程组号（PGID）
```

### 13.进程的创建（重点）

系统允许一个进程创建新进程，新进程即为子进程，子进程还可以创建新的子进程，形成进程树结构模型。

```c
#include <sys/types.h>
#include <unistd.h>

pid_t fork(void);
功能：
    用于从一个已存在的进程中创建一个新进程，新进程称为子进程，原进程称为父进程。
返回值：
    成功：子进程中返回 0，父进程中返回子进程 ID。pid_t，为整型。
    失败：返回-1。
    失败的两个主要原因是：
        1）当前的进程数已经达到了系统规定的上限，这时 errno 的值被设置为 EAGAIN。
        2）系统内存不足，这时 errno 的值被设置为 ENOMEM。
```

fork() 之后创建了一个新的进程，新进程为子进程，原来的进程为父进程。

### 14. 父子进程关系

使用 fork() 函数得到的子进程是父进程的一个复制品，它从父进程处继承了整个进程的地址空间：包括进程上下文（进程执行活动全过程的静态描述）、进程堆栈、打开的文件描述符、信号控制设定、进程优先级、进程组号等。

子进程所独有的只有它的进程号，计时器等（只有小量信息）。因此，使用 fork() 函数的代价是很大的。

![1527909236123](file:///C:/Users/jin/Desktop/%E7%AC%AC4%E9%98%B6%E6%AE%B5-Linux%E9%AB%98%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E5%8F%91/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%AC%AC05%E5%A4%A9%EF%BC%88%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%EF%BC%89/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/assets/1527909236123.png)

简单来说， 一个进程调用 fork() 函数后，系统先给新的进程分配资源，例如存储数据和代码的空间。然后把原来的进程的所有值都复制到新的新进程中，只有少数值与原来的进程的值不同。相当于克隆了一个自己。

实际上，更准确来说，Linux 的 fork() 使用是通过**写时拷贝** (copy- on-write) 实现。写时拷贝是一种可以推迟甚至避免拷贝数据的技术。内核此时并不复制整个进程的地址空间，而是让父子进程共享同一个地址空间。只用在需要写入的时候才会复制地址空间，从而使各个进行拥有各自的地址空间。也就是说，资源的复制是在需要写入的时候才会进行，在此之前，只有以只读方式共享。

注意：fork之后父子进程共享文件，fork产生的子进程与父进程相同的文件文件描述符指向相同的文件表，引用计数增加，共享文件文件偏移指针。

### 15. 区分父子进程

子进程是父进程的一个复制品，可以简单认为父子进程的代码一样的。那大家想过没有，这样的话，父进程做了什么事情，子进程也做什么事情（如上面的例子），是不是不能实现满足我们实现多任务的要求呀，那我们是不是要想个办法区别父子进程呀，这就通过 fork() 的返回值。

fork() 函数被调用一次，但返回两次。两次返回的区别是：子进程的返回值是 0，而父进程的返回值则是新子进程的进程 ID。

```c
int main()
{
    pid_t pid;
    pid = fork();
    if (pid < 0)
    {   // 没有创建成功  
        perror("fork");
        return 0;
    }

    if (0 == pid)
    { // 子进程  
        while (1)
        {
            printf("I am son\n");
            sleep(1);
        }
    }
    else if (pid > 0)
    { // 父进程  
        while (1)
        {
            printf("I am father\n");
            sleep(1);
        }
    }

    return 0;
}
```

父子进程各做一件事（各自打印一句话）。这里，我们只是看到只有一份代码，实际上，fork() 以后，有两个地址空间在独立运行着，有点类似于有两个独立的程序（父子进程）在运行着。

**一般来说，在 fork() 之后是父进程先执行还是子进程先执行是不确定的。这取决于内核所使用的调度算法。**

需要注意的是，在子进程的地址空间里，子进程是从 fork() 这个函数后才开始执行代码。

### 16.父子进程地址空间

**父子进程各自的地址空间是独立的**

```c
int a = 10;     // 全局变量

int main()
{
    int b = 20; //局部变量
    pid_t pid;
    pid = fork();
    if (pid < 0)
    {   // 没有创建成功
        perror("fork");
    }

    if (0 == pid)
    { // 子进程
        a = 111;
        b = 222;    // 子进程修改其值
        printf("son: a = %d, b = %d\n", a, b);
    }
    else if (pid > 0)
    { // 父进程
        sleep(1);   // 保证子进程先运行
        printf("father: a = %d, b = %d\n", a, b);
    }

    return 0;
}

```

通过得知，在子进程修改变量 a，b 的值，并不影响到父进程 a，b 的值。

### 17.GDB调试多进程

使用gdb调试的时候，gdb只能跟踪一个进程。可以在fork函数调用之前，通过指令设置gdb调试工具跟踪父进程或者是跟踪子进程。默认跟踪父进程。

- set follow-fork-mode child 设置gdb在fork之后跟踪子进程。
- set follow-fork-mode parent 设置跟踪父进程（默认）。

注意，一定要在fork函数调用之前设置才有效。

### 18.进程退出函数

```c
#include <stdlib.h>
void exit(int status);

#include <unistd.h>
void _exit(int status);
功能：
    结束调用此函数的进程。
参数：
    status：返回给父进程的参数（低 8 位有效），至于这个参数是多少根据需要来填写。
返回值：
    无
```

1exit() 和 *exit() 函数功能和用法是一样的，无非时所包含的头文件不一样，还有的区别就是：**exit()属于标准库函数**，***exit()属于系统调用**函数。

![1527910170715](file:///C:/Users/jin/Desktop/%E7%AC%AC4%E9%98%B6%E6%AE%B5-Linux%E9%AB%98%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E5%8F%91/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%AC%AC05%E5%A4%A9%EF%BC%88%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%EF%BC%89/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/assets/1527910170715.png)

### 19.等待子进程退出函数

#### 19.1 概述

```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *status);
功能：
    等待任意一个子进程结束，如果任意一个子进程结束了，此函数会回收该子进程的资源。
参数：
    status : 进程退出时的状态信息。
返回值：
    成功：已经结束子进程的进程号
    失败： -1
```

在每个进程退出的时候，内核释放该进程所有的资源、包括打开的文件、占用的内存等。但是仍然为其保留一定的信息，这些信息主要主要指进程控制块PCB的信息（包括进程号、退出状态、运行时间等）。

父进程可以通过调用wait或waitpid得到它的退出状态同时彻底清除掉这个进程。

wait() 和 waitpid() 函数的功能一样，区别在于，

​		**wait() 函数会阻塞，**

​		**waitpid() 可以设置不阻塞**，waitpid() 还可以指定等待哪个子进程结束。

注意：一次wait或waitpid调用只能清理一个子进程，清理多个子进程应使用循环。

调用 wait() 函数的进程会挂起（阻塞），直到它的一个子进程退出或收到一个不能被忽视的信号时才被唤醒（相当于继续往下执行）。

若调用进程没有子进程，该函数立即返回；若它的子进程已经结束，该函数同样会立即返回，并且会回收那个早已结束进程的资源。

所以，wait()函数的主要功能为回收已经结束子进程的资源。

如果参数 status 的值不是 NULL，wait() 就会把子进程退出时的状态取出并存入其中，这是一个整数值（int），指出了子进程是正常退出还是被非正常结束的。

这个退出信息在一个 int 中包含了多个字段，直接使用这个值是没有意义的，我们需要用宏定义取出其中的每个字段。

 

**宏函数可分为如下三组：**

\1) WIFEXITED(status)

为非0 → 进程正常结束

WEXITSTATUS(status)

如上宏为真，使用此宏 → 获取进程退出状态 (exit的参数)

\2) WIFSIGNALED(status)

为非0 → 进程异常终止

WTERMSIG(status)

如上宏为真，使用此宏 → 取得使进程终止的那个信号的编号。

\3) WIFSTOPPED(status)

为非0 → 进程处于暂停状态

WSTOPSIG(status)

如上宏为真，使用此宏 → 取得使进程暂停的那个信号的编号。

WIFCONTINUED(status)

为真 → 进程暂停后已经继续运行

#### 19.2waitpid函数

```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t waitpid(pid_t pid, int *status, int options);
功能：
    等待子进程终止，如果子进程终止了，此函数会回收子进程的资源。

参数：
    pid : 参数 pid 的值有以下几种类型：
      pid > 0  等待进程 ID 等于 pid 的子进程。
      pid = 0  等待同一个进程组中的任何子进程，如果子进程已经加入了别的进程组，waitpid 不会等待它。
      pid = -1 等待任一子进程，此时 waitpid 和 wait 作用一样。
      pid < -1 等待指定进程组中的任何子进程，这个进程组的 ID 等于 pid 的绝对值。

    status : 进程退出时的状态信息。和 wait() 用法一样。

    options : options 提供了一些额外的选项来控制 waitpid()。
            0：同 wait()，阻塞父进程，等待子进程退出。
            WNOHANG：没有任何已经结束的子进程，则立即返回。
            WUNTRACED：如果子进程暂停了则此函数马上返回，并且不予以理会子进程的结束状态。（由于涉及到一些跟踪调试方面的知识，加之极少用到）
                 
返回值：
    waitpid() 的返回值比 wait() 稍微复杂一些，一共有 3 种情况：
        1) 当正常返回的时候，waitpid() 返回收集到的已经回收子进程的进程号；
        2) 如果设置了选项 WNOHANG，而调用中 waitpid() 发现没有已退出的子进程可等待，则返回 0；
        3) 如果调用中出错，则返回-1，这时 errno 会被设置成相应的值以指示错误所在，如：当 pid 所对应的子进程不存在，或此进程存在，但不是调用进程的子进程，waitpid() 就会出错返回，这时 errno 被设置为 ECHILD；
```

### 20.孤儿进程

父进程运行结束，但子进程还在运行（未运行结束）的子进程就称为孤儿进程（Orphan Process）。

每当出现一个孤儿进程的时候，内核就把孤儿进程的父进程设置为 init ，而 init 进程会循环地 wait() 它的已经退出的子进程。这样，当一个孤儿进程凄凉地结束了其生命周期的时候，init 进程就会代表党和政府出面处理它的一切善后工作。

因此孤儿进程并不会有什么危害。

### 21.僵尸进程

进程终止，父进程尚未回收，子进程残留资源（PCB）存放于内核中，变成僵尸（Zombie）进程。

这样就会导致一个问题，如果进程不调用wait() 或 waitpid() 的话， 那么保留的那段信息就不会释放，其进程号就会一直被占用，但是系统所能使用的进程号是有限的，如果大量的产生僵尸进程，将因为没有可用的进程号而导致系统不能产生新的进程，此即为僵尸进程的危害，应当避免。

### 22.进程替换

**概述**

在 Windows 平台下，我们可以通过双击运行可执行程序，让这个可执行程序成为一个进程；而在 Linux 平台，我们可以通过 ./ 运行，让一个可执行程序成为一个进程。

但是，如果我们本来就运行着一个程序（进程），我们如何在这个进程内部启动一个外部程序，由内核将这个外部程序读入内存，使其执行起来成为一个进程呢？这里我们通过 exec 函数族实现。

exec 函数族，顾名思义，就是一簇函数，在 Linux 中，并不存在 exec() 函数，exec 指的是一组函数，一共有 6 个：

```c
#include <unistd.h>
extern char **environ;

int execl(const char *path, const char *arg, .../* (char  *) NULL */);
int execlp(const char *file, const char *arg, ... /* (char  *) NULL */);
int execle(const char *path, const char *arg, .../*, (char *) NULL, char * const envp[] */);
int execv(const char *path, char *const argv[]);
int execvp(const char *file, char *const argv[]);
int execvpe(const char *file, char *const argv[], char *const envp[]);

int execve(const char *filename, char *const argv[], char *const envp[]);
```

## 进程间通信

进程是一个独立的资源分配单元，不同进程（这里所说的进程通常指的是用户进程）之间的资源是独立的，没有关联，不能在一个进程中直接访问另一个进程的资源。但是，进程不是孤立的，**不同的进程需要进行信息的交互和状态的传递等，因此需要进程间通信(** IPC：Inter Processes Communication )。

进程间通信的目的：

- 数据传输：一个进程需要将它的数据发送给另一个进程。
- 通知事件：一个进程需要向另一个或一组进程发送消息，通知它（它们）发生了某种事件（如进程终止时要通知父进程）。
- 资源共享：多个进程之间共享同样的资源。为了做到这一点，需要内核提供互斥和同步机制。
- 进程控制：有些进程希望完全控制另一个进程的执行（如 Debug 进程），此时控制进程希望能够拦截另一个进程的所有陷入和异常，并能够及时知道它的状态改变。

**Linux 操作系统支持的主要进程间通信的通信机制：**

同一主机间通信：无名/有名管道，信号   消息队列，共享内存，信号量

不同：socket

### 1.无名管道

#### 1.1概述

当涉及管道时，有几个关键概念需要注意：

1. **无名管道**：也称为匿名管道，没有在文件系统中有明确的名称或路径。它通常用于在相关进程之间进行通信。
2. **半双工**：数据在同一时刻只能在一个方向上流动，因此管道是半双工的，而不是全双工的（可以双向传输）。
3. **单向数据流**：数据只能从管道的一端写入，从另一端读出。如果需要双向通信，则需要创建两个管道。
4. **先入先出（FIFO）**：写入管道中的数据遵循先入先出的规则。也就是说，先写入的数据会被先读取出来。
5. **无格式数据传输**：管道传输的数据是无格式的，这意味着管道本身不负责数据的解析或分隔，需要通信的进程必须事先约定好数据的格式。
6. **存在于内存中（重点）**：管道不是普通的文件，而是存在于内存中。它们通常有一个缓冲区，不同的系统可能有不同大小的缓冲区。
7. **一次性读取**：从管道读取数据是一次性操作，一旦数据被读取，就会从管道中被抛弃，释放空间以接收更多的数据。
8. **父子进程或兄弟进程之间的通信**：管道只能在具有公共祖先的进程之间使用，如父进程与子进程，或者两个兄弟进程。这是因为管道是由一个进程创建并共享给其派生的子进程。

总的来说，管道提供了一种简单而有效的进程间通信机制，适用于需要低延迟、基于字节流的通信场景。

#### 1.2pipe函数

```c
#include <unistd.h>
int pipe(int pipefd[2]);
功能：创建无名管道。
参数：
    pipefd : 为 int 型数组的首地址，其存放了管道的文件描述符 pipefd[0]、pipefd[1]。
    当一个管道建立时，它会创建两个文件描述符 fd[0] 和 fd[1]。其中 fd[0] 固定用于读管道，而 fd[1] 固定用于写管道。一般文件 I/O的函数都可以用来操作管道(lseek() 除外)。
返回值：
    成功：0
    失败：-1
// 子进程通过无名管道给父进程传递一个字符串数据：
int main()
{
    int fd_pipe[2] = { 0 };
    pid_t pid;
    if (pipe(fd_pipe) < 0)
    {// 创建管道
        perror("pipe");
    }
    pid = fork(); // 创建进程
    if (pid == 0)
    { // 子进程
        char buf[] = "I am mike";
        // 往管道写端写数据
        write(fd_pipe[1], buf, strlen(buf));

        _exit(0);
    }
    else if (pid > 0)
    {// 父进程
        wait(NULL); // 等待子进程结束，回收其资源
        char str[50] = { 0 };
        // 从管道里读数据
        read(fd_pipe[0], str, sizeof(str));
        printf("str=[%s]\n", str); // 打印数据
    }
    return 0;
}
```

#### 1.3管道读写的特点

**读管道：**

Ø 管道中有数据，read返回实际读到的字节数。

Ø 管道中无数据：

u 管道写端被全部关闭，read返回0 (相当于读到文件结尾)

u 写端没有全部被关闭，read阻塞等待(不久的将来可能有数据递达，此时会让出cpu)

**写管道：**

Ø 管道读端全部被关闭， 进程异常终止(也可使用捕捉SIGPIPE信号，使进程终止)

Ø 管道读端没有全部关闭：

u 管道已满，write阻塞。

u 管道未满，write将数据写入，并返回实际写入的字节数。

####  1.4设置非阻塞的方法

```c
//获取原来的flags
int flags = fcntl(fd[0], F_GETFL);
// 设置新的flags
flag |= O_NONBLOCK;
// flags = flags | O_NONBLOCK;
fcntl(fd[0], F_SETFL, flags);
```

#### 1.5查看缓冲区大小 

可以使用**ulimit -a** 命令来查看当前系统中创建管道文件所对应的内核缓冲区大小。

#### 1.6查看缓冲区函数fpathconf

```c
#include <unistd.h>
long fpathconf(int fd, int name);
功能：该函数可以通过name参数查看不同的属性值
参数：
    fd：文件描述符
    name：
        _PC_PIPE_BUF，查看管道缓冲区大小
        _PC_NAME_MAX，文件名字字节数的上限
返回值：
    成功：根据name返回的值的意义也不同。
    失败： -1
    
int main()
{
    int fd[2];
    int ret = pipe(fd);
    if (ret == -1)
    {
        perror("pipe error");
        exit(1);
    }
    long num = fpathconf(fd[0], _PC_PIPE_BUF);
    printf("num = %ld\n", num);
    return 0;
}
```

#### 1.7事例

```c
// 子进程读，父进程写
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#define SIZE 64

int main(void)
{
	pid_t pid;
	char buf[SIZE];
	int ret = -1;
	int fd[2];
	ret = pipe(fd);
	if(ret < 0)
	{
		perror("pipe");
		return 1;
	}

	pid = fork();
	// 判断进程
	if(pid < 0)
	{
		perror("fork");
		return 1;
	}

	if(pid == 0)
	{
		// 子进程来读
		// 关闭写端
		close(fd[1]);
		
		memset(buf,0,SIZE);
		printf("子进程读取管道内容...\n");
		

        
		
		ret = read(fd[0],buf,SIZE);
		
		if(ret < 0)
		{
			perror("read");
			exit(-1);
		}

		printf("child processing buf:%s\n",buf);
		// 关闭读端
		close(fd[1]);

		exit(0);
	}
	
	// 管道缓冲区函数
	printf("read size:%ld\n",fpathconf(fd[0],_PC_PIPE_BUF));
	printf("write size:%ld\n",fpathconf(fd[1],_PC_PIPE_BUF));
	// 关闭读端
	close(fd[0]);
    
	// 用来测试非阻塞
	sleep(1);

	ret = write(fd[1],"ABCDEFGH",8);
	if(ret < 0)
	{
		perror("write");
		return 1;
	}
	
	printf("parent processing write len:%d\n",ret);
	//关闭写端
	close(fd[1]);

	return 0;
}
```



### 2.有名管道

命名管道（FIFO）和无名管道（pipe）是用于进程间通信的两种方式，它们有许多相似之处，但也有一些显著的区别：

相同之处：

1. **数据流动方式**：无论是 FIFO 还是无名管道，数据在管道中的流动都是单向的，并且遵循先入先出的原则。

2. **半双工特性**：管道都是半双工的，只能在一个方向上进行数据传输。

3. **基于文件描述符**：在应用程序中，管道都通过文件描述符来操作。

不同之处：

1. **存在形式**：
   - 无名管道是在内存中存在的，没有文件系统中的实体，因此只能在相关联的进程之间使用。
   - 命名管道作为文件系统中的一个特殊文件存在，有自己的路径名，因此不相关的进程也可以通过打开这个路径名来进行通信。
2. **持久性**：
   - 当使用无名管道的进程退出后，相关的管道就会被销毁，不再存在。
   - 使用命名管道时，即使创建命名管道的进程退出，FIFO 文件仍然存在于文件系统中，其他进程可以继续使用它进行通信。
3. **关联方式**：
   - 无名管道只能在具有亲缘关系的进程之间使用，通常是父子进程之间或者兄弟进程之间。
   - 命名管道可以被不相关的进程使用，只要它们有权限访问命名管道的路径

#### 2.1创建管道命令mkfifo

mkfifo fifo(pipe name)

#### 2.2函数创建管道

```c
#include <sys/types.h>
#include <sys/stat.h>
int mkfifo(const char *pathname, mode_t mode);
功能：
    命名管道的创建。
参数：
    pathname : 普通的路径名，也就是创建后 FIFO 的名字。
    mode : 文件的权限，与打开普通文件的 open() 函数中的 mode 参数相同。(0666)
返回值：
    成功：0   状态码
    失败：如果文件已经存在，则会出错且返回 -1。
```

#### 2.3有名管道读写操作

一旦使用mkfifo创建了一个FIFO，就可以使用open打开它，常见的文件I/O函数都可用于fifo。如：close、read、write、unlink等。

FIFO严格遵循先进先出（first in first out），对管道及FIFO的读总是从开始处返回数据，对它们的写则把数据添加到末尾。**它们不支持诸如lseek()等文件定位操作。**

```c
//进程1，写操作
int fd = open("my_fifo", O_WRONLY);  
char send[100] = "Hello Mike";
write(fd, send, strlen(send));
//进程2，读操作
int fd = open("my_fifo", O_RDONLY);//等着只写  
char recv[100] = { 0 };
//读数据，命名管道没数据时会阻塞，有数据时就取出来  
read(fd, recv, sizeof(recv));
printf("read from my_fifo buf=[%s]\n", recv);
```

#### 2.4 有名管道注意事项

\1) 一个为只读而打开一个管道的进程会阻塞直到另外一个进程为只写打开该管道

2）一个为只写而打开一个管道的进程会阻塞直到另外一个进程为只读打开该管道

**读管道：**

Ø 管道中有数据，read返回实际读到的字节数。

Ø 管道中无数据：

u 管道写端被全部关闭，read返回0 (相当于读到文件结尾)

u 写端没有全部被关闭，read阻塞等待

**写管道：**

Ø 管道读端全部被关闭， 进程异常终止(也可使用捕捉SIGPIPE信号，使进程终止)

Ø 管道读端没有全部关闭：

u 管道已满，write阻塞。

u 管道未满，write将数据写入，并返回实际写入的字节数

#### 2.5单管道实现通信

```c
// write.c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#include <unistd.h>
#define SIZE 128

int main(void)
{
	int i = 0;
	int fd = -1;
	int ret = -1;
	char buf[SIZE];

	// 以只写的模式打开一个管道
	fd = open("fifo",O_WRONLY);
	if(fd == -1)
	{
		perror("open");
		return 1;
	}
	printf("以只写的模式打开一个管道ok...\n");;

	while(1)
	{
		memset(buf,0,SIZE);
		sprintf(buf,"hello itcast %d",i++);
		ret = write(fd,buf,strlen(buf));

		if(ret <= 0)
		{
			perror("write");
			break;
		}

		printf("write fifo:%d\n",ret);
		sleep(1);
		
	}
	// 关闭
	close(fd);
	
	return 0;
}


// read.c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#include <unistd.h>
#define SIZE 128

int main(void)
{
	int i = 0;
	int fd = -1;
	int ret = -1;
	char buf[SIZE];

	// 以只写的模式打开一个管道
	fd = open("fifo",O_RDONLY);
	if(fd == -1)
	{
		perror("open");
		return 1;
	}
	printf("以只读的模式打开一个管道ok...\n");;

	while(1)
	{
		memset(buf,0,SIZE);
		
		ret = read(fd,buf,SIZE);

		if(ret <= 0)
		{
			perror("read");
			break;
		}

		printf("read fifo:%s\n",buf);
	}


	// 关闭
	close(fd);
	
	return 0;
}


```

#### 2.6管道实现简单聊天



```c
//re.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
#define SIZE 128

int main(void)
{
	int fdr = -1;
	int fdw = -1;
	int ret = -1;
	char buf[SIZE];


	fdr = open("fifo1",O_RDONLY);
	if(fdr == -1)
	{
		perror("fifo1 open");
		return 1;
	}
	printf("以只读的方式打开管道1....\n");
	
	fdw = open("fifo2",O_WRONLY);
	if(fdw == -1)
	{
		perror("fifo2 open");
		return 1;
	}	
	printf("以只写的方式打开管道2....\n");


	// 循环读写
	while(1)
	{
		// 读管道1
		memset(buf,0,SIZE);
		ret = read(fdr,buf,SIZE);
		if(ret <= 0)
		{
			perror("read");
			break;
		}
		printf("read:%s\n",buf);

			
		
		memset(buf,0,SIZE);
		fgets(buf,SIZE,stdin);
		if('\n' == buf[strlen(buf) - 1])
			buf[strlen(buf) - 1] = '\0';
		
		//写管道2
		ret = write(fdw,buf,strlen(buf));
		if(ret <= 0)
		{
			perror("write");
			break;
		}
		printf("write ret:%d\n",ret);
	
	}
	
	close(fdr);
	close(fdw);

	return 0;
}

//wr.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>

#define SIZE 128


int main(void)
{
	int fdr = -1;
	int fdw = -1;
	int ret = -1;
	char buf[SIZE];


	fdw = open("fifo1",O_WRONLY);
	if(fdw == -1)
	{
		perror("fifo1 open");
		return 1;
	}	
	printf("以只写的方式打开管道1....\n");

	fdr = open("fifo2",O_RDONLY);
	if(fdr == -1)
	{
		perror("fifo2 open");
		return 1;
	}
	printf("以只读的方式打开管道2....\n");
	



	// 循环读写
	while(1)
	{
		memset(buf,0,SIZE);
		fgets(buf,SIZE,stdin);
		if('\n' == buf[strlen(buf) - 1])
			buf[strlen(buf) - 1] = '\0';
			
		//写管道1
		ret = write(fdw,buf,strlen(buf));
		if(ret <= 0)
		{
			perror("write");
			break;
		}
		printf("write ret:%d\n",ret);
		
		// 读管道2
		memset(buf,0,SIZE);
		ret = read(fdr,buf,SIZE);
		if(ret <= 0)
		{
			perror("read");
			break;
		}
		printf("read:%s\n",buf);	
	}
	
	close(fdr);
	close(fdw);

	return 0;
}
```

### 3.共享内存存储映射

存储映射I/O (Memory-mapped I/O) 使一个磁盘文件与存储空间中的一个缓冲区相映射。

![1527925077595](file:///C:/Users/jin/Desktop/%E6%96%87%E4%BB%B6%E6%8E%A5%E6%94%B6%E6%9F%9C/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%AC%AC06%E5%A4%A9%EF%BC%88%E8%BF%9B%E7%A8%8B%E9%97%B4%E9%80%9A%E4%BF%A1%EF%BC%89/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/assets/1527925077595.png)

于是当从缓冲区中取数据，就相当于读文件中的相应字节。于此类似，将数据存入缓冲区，则相应的字节就自动写入文件。这样，就可在不适用read和write函数的情况下，使用地址（指针）完成I/O操作。

共享内存可以说是最有用的进程间通信方式，也是最快的IPC形式, 因为进程可以直接读写内存，而不需要任何数据的拷贝。

#### 3.1mmap函数

```c
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
功能:
    一个文件或者其它对象映射进内存
参数：
    addr :  指定映射的起始地址, 通常设为NULL, 由系统指定
    length：映射到内存的文件长度
    prot：  映射区的保护方式, 最常用的 :
        a) 读：PROT_READ
        b) 写：PROT_WRITE
        c) 读写：PROT_READ | PROT_WRITE
    flags：  映射区的特性, 可以是
        a) MAP_SHARED : 写入映射区的数据会复制回文件, 且允许其他映射该文件的进程共享。
        b) MAP_PRIVATE : 对映射区的写入操作会产生一个映射区的复制(copy - on - write), 对此区域所做的修改不会写回原文件。
    fd：由open返回的文件描述符, 代表要映射的文件。
    offset：以文件开始处的偏移量, 必须是4k的整数倍, 通常为0, 表示从文件头开始映射
返回值：
    成功：返回创建的映射区首地址
    失败：MAP_FAILED宏
```

**关于mmap函数的使用总结：**

\1) 第一个参数写成NULL

\2) 第二个参数要映射的文件大小 > 0

\3) 第三个参数：PROT_READ 、PROT_WRITE

\4) 第四个参数：MAP_SHARED 或者 MAP_PRIVATE

\5) 第五个参数：打开的文件对应的文件描述符

\6) 第六个参数：4k的整数倍，通常为0

#### 3.2munmap函数

```c
#include <sys/mman.h>
int munmap(void *addr, size_t length);
功能：
    释放内存映射区
参数：
    addr：使用mmap函数创建的映射区的首地址
    length：映射区的大小
返回值：
    成功：0
    失败：-1
```

这些注意事项对于使用 `mmap()` 函数来创建内存映射区是非常重要的：

1. **隐含的读操作**：在创建映射区的过程中，会隐含进行一次对映射文件的读操作。这意味着在创建映射区时，会涉及到对文件的访问。

2. **权限匹配**：
   - 当使用 `MAP_SHARED` 时，映射区的权限应该小于或等于文件打开的权限，以确保对映射区的保护。
   - 对于 `MAP_PRIVATE`，由于权限是针对内存的，因此文件打开的权限不影响。

3. **映射区的释放**：映射区的释放与文件的关闭无关，即使文件已经关闭，映射区仍然存在。

4. **映射文件大小的限制**：映射文件的大小不能为0，否则无法创建映射区。因此，用于映射的文件必须要有实际大小。

5. **munmap的使用**：`munmap()` 函数用于解除映射区，传入的地址必须是 `mmap()` 的返回地址。此外，应避免对指针进行 `++` 操作，以确保操作的安全性。

6. **文件偏移量的限制**：文件偏移量通常需要是4K的整数倍，具体要求取决于系统。

7. **检查返回值**：`mmap()` 函数创建映射区的成功概率不高，因此在使用后应该检查其返回值，以确保映射区建立成功后再进行后续操作，避免出现错误。

#### 3.3事例

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/mman.h>

int main(void)
{
    int fd = -1;
    void *addr = NULL;
    
    fd = open("txt", O_RDWR);
    if (fd == -1)
    {
        perror("open");
        return 1;
    }
    
    // 将文件映射到内存
    addr = mmap(NULL, 1024, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED)
    {
        perror("mmap");
        return 1;
    }
    printf("文件映射成功...\n");
    
    close(fd);
    
    memcpy(addr, "12345678910", 10); 
    
    // 断开映射
    munmap(addr, 1024);    
    
    return 0;
}

```

#### 3.4实现父子通信

```c
 int fd = open("xxx.txt", O_RDWR);// 打开一个文件
 int len = lseek(fd, 0, SEEK_END);//获取文件大小

    // 创建内存映射区
    void *ptr = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (ptr == MAP_FAILED)
    {
        perror("mmap error");
        exit(1);
    }
    close(fd); //关闭文件

    // 创建子进程
    pid_t pid = fork();
    if (pid == 0) //子进程
    {
        sleep(1); //演示，保证父进程先执行

        // 读数据
        printf("%s\n", (char*)ptr);
    }
    else if (pid > 0) //父进程
    {
        // 写数据
        strcpy((char*)ptr, "i am u father!!");

        // 回收子进程资源
        wait(NULL);
    }

    // 释放内存映射区
    int ret = munmap(ptr, len);
    if (ret == -1)
    {
        perror("munmap error");
        exit(1);
```

