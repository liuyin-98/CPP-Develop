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

从内存的角度考虑，输入为从文件到磁盘，输出为从磁盘到文件

#### 标准C库IO

![C_IO](img/C_IO.jpg)

为了实现跨平台 

- Java提供虚拟机

- C会调用系统API 

C的IO速度 **高于** Linux的IO速度，C里面有缓冲区

- 缓冲区的存在使得读写磁盘的次数变少了，因此效率增加

![C_Linux_IO](img/C_Linux_IO.jpg)

#### 虚拟地址空间

- 存在的问题
  - 内存小无法加载多个进程
  - 进程释放后导致的内存不连续

- 系统会给每一个进程分配一个虚拟地址空间
  - 32位系统会分配$2^{32}=4 \ GB$的空间
  - 64位系统分配$2^{48} $的空间

MMU（内存管理单元）进行虚拟地址和物理地址的转换

![VAS](img/VAS.jpg)

#### 文件描述符

![File_Description](img/File_Description.jpg)

进程控制块可以看成一个复杂的结构体，其中包含文件描述符表，是一个数组，默认大小为1024

标准输入输出错误都指向文件 当前终端 /dev/tty（一切皆文件）

### Linux IO

查看Linux函数 `man 2 LinuxfuncitonName`， C函数 `man 3 CfunctionName`

- `open() / close()`

  ```c++
      #include <sys/types.h>  // 定义了宏
      #include <sys/stat.h>   // 定义了宏
      #include <fcntl.h>      // open 在这里面声明
      
      // 打开已经存在的文件
      int open(const char *pathname, int flags);
      	pathname	: 文件路径
      	flags		: 访问权限以及其他操作
          			  O_RDONLY, O_WRONLY, O_RDWR 互斥
          返回新的文件描述符，失败则返回-1
          
      errno 属于Linux系统函数库中的全局变量，记录最近的错误号
     	perror 属于标准C库，打印errno对应的错误描述
  
  	#include <stdio.h>
         void perror(const char *s);
        s : 用户描述
        最终输出 s:xxx(实际错误)
      
      // 创建一个文件
      int open(const char *pathname, int flags, mode_t mode);
      	flags 	: 上述参数必选而且互斥，可选参数还有O_CREAT
      	mode 	: 八进制数，表示用户对新文件的操作权限 0777 （八进制数0开头）
      			  最终权限是 mode & ~umask（root 0022）
      			  umask 用于抹去某些权限
      			  每个文件的权限用 10 个字符表示 第一个是文件类型
      			  后续总共三组依次代表 当前用户 当前用户组 其他
      			  每一位 代表 r w x
  
  ```

- `read() / write()`

  都是相对于内存而言的

  ```c++
  
  #include <unistd.h>
  
  ssize_t read(int fd, void *buf, size_t count);
  	fd		: 文件描述符
  	buf		: 从文件读取数据存放的地方
  	count	: 数组的大小
  	
  	返回 读取到的字节数量，0说明文件已经读取完毕，失败则返回-1并设置errno
  
  ssize_t write(int fd, const void *buf, size_t count);
  	buf		: 带写入数据
  	count	: 写入数据大小
  	
  	返回 写入的字节数量， 0说明没有写入内容， 失败则返回-1并设置error
  
  
  ```

- `lseek()`

  ```c++
  #include <sys/types.h>
  #include <unistd.h>
  
  off_t lseek(int fd, off_t offset, int whence);
  	offset 	: 偏移值
  	whence	: 分别相对于 起始 / 当前 / 结尾 的偏移量
  			  SEEK_SET 
                The file offset is set to offset bytes.
  
         		  SEEK_CUR 
                The file offset is set to its current location plus offset bytes.
  
  		      SEEK_END 
                The file offset is set to the size of the file plus offset bytes.
     返回 文件指针位置
  ```

  - 用法
    - 移动文件到起始
    - 获取文件指针当前位置
    - 获取文件大小
    - 拓展文件长度

- `stat() / lstat()`

  ```c++
  #include <sys/types.h>
  #include <sys/stat.h>
  #include <unistd.h>
  
  int stat(const char *pathname, struct stat *statbuf);
  	获取一个文件相关的信息
      File: text.txt
    	Size: 1453            Blocks: 8          IO Block: 4096   regular file
  	Device: 801h/2049d      Inode: 1077085     Links: 1
  	Access: (0664/-rw-rw-r--)  Uid: ( 1000/     leo)   Gid: ( 1000/     leo)
  	Access: 2022-04-19 20:11:09.587940906 +0800
  	Modify: 2022-04-19 20:36:59.151587663 +0800
  	Change: 2022-04-19 20:36:59.151587663 +0800
  	Birth: -    
          
      statbuf : 传出参数
      成功则返回 0， 否则 -1并设置errno 
              
  int lstat(const char *pathname, struct stat *statbuf);
  	可以获取 软链接 文件信息
  
  struct stat {
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
  ```

  - `mode_t`

  ![mode_t](img/mode_t.jpg)