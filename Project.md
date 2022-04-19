# C++ Linux下开发

[toc]

## Linux基础知识

### gcc / g++

#### gcc 和  g++ 都是 GNU（组织）的编译器

- gcc 和 g++ 都可以编译 C++ 代码
  - 编译过程中，g++会调用gcc，但是gcc命令不能**自动**和C++程序使用的库**链接**，所以需要g++来实现
  - 编译可以使用 gcc/g++，链接可以使用 g++ / gcc -lstdc++

#### GCC 常用选项

| 编译选项                                  | 说明                         |
| ----------------------------------------- | ---------------------------- |
| `-E`                                      | 预处理                       |
| `-S`                                      | 汇编                         |
| `-c`                                      | 编译                         |
| `-o [file1] [file2] / [file2] -o [file1]` | 生成可执行文件 file1         |
| `-I`                                      | 包含头文件目录               |
| `-g`                                      | 生成调试信息                 |
| `-D`                                      | 编译时指定宏                 |
| `-w`                                      | 不生成warning                |
| `-Wall`                                   | 生成所有warning              |
| `-On`                                     | 优化 n=0~3 越大级别越高      |
| `-l`                                      | 指定使用的库                 |
| `-L`                                      | 编译时搜索库的路径           |
| `-fPIC/fpic`                              | 生成与位置无关代码           |
| `-shared`                                 | 生成共享目标文件，建立共享库 |
| `-std`                                    | 指定标准                     |

```c++
// test.c
...
#ifdef DEBUG
    ...
# endif
    
    
// terminal
g++ test.c -o test     	   	// 不执行DEBUG里面的内容
g++ test.c -o test -DDEBUG  // 执行
```

### 静态库

- 优点
  - 加载速度快
  - 发布程序无需提供额外库，移植方便
- 缺点
  - 消耗系统资源，浪费内存
    - 每一个进程都会加载在内存中，会加载重复代码
  - 更新、部署、发布麻烦
    - 每次更新都要重新编译

- 命名规则 `libxxx.a`

- 生成

  1. `gcc`生成 `.o`

     ```shell
     gcc -c xxx1.c xxx2.c -I inlucde/
     ```

  2. 使用`ar` 打包 `.o`

     ```shell
     ar rcs libxxx.a xxx1.o xxx2.o
     ```

     | 参数 | 含义                 |
     | ---- | -------------------- |
     | r    | 将文件插入备存文件中 |
     | c    | 建立备存文件         |
     | s    | 建立索引             |

- 使用

  - ```shell
    .
    ├── include
    │   └── head.h
    ├── lib
    │   └── libcalc.a
    ├── main.c
    └── source
        ├── add.c
        ├── div.c
        ├── mult.c
        └── sub.c
    ```

    ```shell
    gcc main.c -o main -I ./include/ -lcalc -L ./lib/
    ```

### 动态库

- 优点
  - 进程间资源共享
  - 更新、部署、发布简单
  - 可以控制何时加载
    - 使用到才会加载
- 缺点
  - 加载速度相比于静态库较慢
  - 发布程序需要提供动态库

- 命名规则 `libxxx.so`，在Linux下是**可执行文件**

- 生成

  1. `gcc` 生成 `.o`，注意要生成与位置无关的代码

     ```shell
     gcc -c -fpic/-fPIC xxx1.c xxx2.c
     ```

  2. `gcc` 生成动态库

     ```shell
     gcc -shared xxx1.o xxx2.o -o libxxx.so
     ```

- 使用与上述静态库一致

  ```shell
  .
  ├── include
  │   └── head.h
  ├── lib
  │   └── libcalc.so
  ├── main
  ├── main.c
  └── source
      ├── add.c
      ├── div.c
      ├── mult.c
      └── sub.c
      
  gcc main.c -o main -I ./include/ -lcalc -L ./lib/
  ```

  - 运行没有问题，但是在执行 `main` 的时候报错 `./main: error while loading shared libraries: libcalc.so: cannot open shared object file: No such file or directory`，加载失败

    - 程序启动之后，在使用相应API的时候 动态库**理应**被动态加载到内存中，可以通过 `ldd（list dynamic dependencies）` 检查库的依赖关系

      ```shell
      leo@ubuntu:~/Linux/lesson05/library$ ldd main
              linux-vdso.so.1 (0x00007ffd2217a000)
              libcalc.so => not found
              libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f7e07907000)
              /lib64/ld-linux-x86-64.so.2 (0x00007f7e07efa000)
      ```

    - 执行程序时，需要知道 动态库 的**绝对路径**，通过系统的动态载入器来获取路径

    - 对于 elf 格式的可执行文件，通过 `ld-linux.so` 来完成，寻找顺序

      - 首先搜索 elf 文件的 `DT_RPATH`段 （无法修改）
      - 搜索环境变量 `LD_LIBRARY_PATH`
      - `/etc/ld.so.cache`
        - 将路径粘贴到`/etc/ld.so.conf`，通过`sudo ldconfig`更新
      - `/lib/,  /usr/lib` （包含许多系统自带文件，可能重复替换，不推荐）
        - 把动态库放到目录下

      

### 环境变量

- 输入`env`查看，环境变量格式`KEY = VALUE1:VALUE2:...`

- 配置环境变量

  - `export KEY=$KEY:VALUE`， `$`取原有的值，`:`增加新值 

    *只适用于当前终端*

  - 永久配置

    - 用户级别

      `/home/.bashrc`的最后以后增加上述方法

      使用`. .bashrc`或`source .bashrc`生效

    - 系统级别

      `/etc/profile`的最后增加上述方法，需要`sudo`

### Makefile

`make`是一个命令工具，可以解释`Makefile`中的指令

- Visual C++ 是 nmake
- GNU 是 make

`Makefile`用于自动化编译，可以定义编译顺序，是否需要重新编译等

#### 规则

```shell
目标 ... : 依赖 ...
	命令 （Shell）
	...
```

- 目标：最重要生成的文件
- 依赖：生成目标所需要的文件或是目标
- 命令：通过执行命令对依赖操作生成目标（必须要Tab缩进）

#### 工作原理

- 在执行命令前要检查规则中的依赖是否存在

  - 存在，执行

  - 不存在，向下检查其他规则，检查有没有其他规则能生成依赖

    找到了在执行该规则

- 检测更新，在执行规则中的命令时，比较目标和依赖文件的时间

  - 依赖时间晚于目标，则执行重新生成目标
  - 否则不执行

*Makefile的其他规则都是为第一条规则服务的，若后续规则和第一条无关则不会执行*

#### 变量

- 自定义变量 `Name = Value`
- 预定义变量
  - `AR = ar(default)`
  - `CC = cc(default)`
  - `CXX = g++(default)`
  - `$@` 目标完整名称
  - `$<` 第一个依赖文件的名称
  - `$^` 所有依赖文件 
- 获取变量值
  - `$(Name)`

#### 模式匹配

- `%`   匹配字符串， 匹配第一条规则中的依赖

  ```makefile
  %.o : %.c
  ```

#### 函数

- `$(wildcard PATTERN)`	
  - 获取指定目录下指定类型的文件列表
  - PATTERN 某个或多个目录下的对应的某种类型的文件，使用空格间隔

- `$(patsubst <pattern>, <replacement>, <text>)`
  - 查找`<text>`中符合`<pattern>`的字符串，匹配则用`<replacement>`替换

#### 定义clean

用于删除中间生成的 .o 文件

```makefile
...

.PHONY:clean
clean:
	rm $(objs) -f
```

终端中输入`make clean` 执行

*如果当前目录下存在一个文件名为 clean 的文件，那么则不会执行，因为Makefile中的clean规则没有依赖，那么目标 clean 视为永远比依赖要新的，所以不会执行， 需要增加.PHONY将其定义为伪目标*

### 文件IO

从内存的角度考虑，输入为从文件到内存，输出为从内存到文件

#### 标准C库IO

![C_IO](img/C_IO.jpg)

为了实现跨平台 

- Java提供虚拟机

- C会调用系统API 

