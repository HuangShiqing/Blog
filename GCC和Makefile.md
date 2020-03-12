# 前言
大型项目往往会用到Makefile或者CMake。对于个人而言除了只会make之外，平时用Makefile来管理不大的项目确实很方便，可以让我们告别各种IDE的配置。该文仅作为个人学习记录
# 常用GCC编译选项

在使用gcc、g++编译C/C++程序的时候会涉，下面列出常用选项  
[参考链接](https://www.runoob.com/w3cnote/gcc-parameter-detail.html)
+ -o 指定文件名
    ```bash
    gcc test.c -o test#指定生成的可执行文件的文件名
    ```
+ -W 指定警告信息
    ```bash
    gcc test.c -Wall -o test#指定产生所有警告
    gcc test.c -w -o test#小写w指定产生不生成警告，等同于不加该参数
    ```
+ -I 大写的i，指定头文件搜索目录
    ```bash
    gcc test.c -Iinclude -o test#指定./include为头文件搜索目录
    ```
+ -L 指定额外的库文件搜索位置
    ```bash
    gcc test.c -Llib -o test#指定./lib为库文件搜索目录
    ```
+ -l 小写的L，指定库文件
    ```bash
    gcc test.c -ldynamic_lib.so -o test#在系统默认、-L额外指定的库文件搜索目录中查找名为libdynamic_lib.so的库文件（编译选项中省略了开头的`lib`）
    ```
+ -O 指定优化等级
    ```bash
    gcc test.c -O0 -o test#O0优化，等于优化，常用于debug
    gcc test.c -O3 -o test#O3优化，常用于release
    ```
+ -g 指定生成调试信息。GNU 调试器可利用该信息。一般和-O0一起使用用作debug模式
    ```bash
    gcc test.c -O0 -g -o test#debug模式
    ```
+ -shared -fPIC 用于生产动态库
    ```bash
    gcc dynamic_lib.c -shared -fPIC -o libdynamic_lib.so
    ```
+ -D 宏定义
    ```bash
    gcc test.c -DGPU -o test#相当于c文件中#define GPU
    gcc test.c -DGPU=4 -o test#相当于c文件中#define GPU 4
    ```
+ \`pkg-config --cflags --libs\` 指定.pc文件  
[pkg-config的参考链接](https://www.cnblogs.com/rainsoul/p/10567390.html)
    ```bash
    gcc test.c `pkg-config --cflags --libs opencv` -o test#读取opencv.pc文件里的头文件和库文件加载目录
    ```
+ -c 用于生成.o文件
    ```bash
    gcc -c test.c#生成test.o文件
    ```


# Makefile示例文件
Makefile基本规则不赘述，推荐这个[参考链接](https://blog.csdn.net/haoel/article/details/2886)，写的挺好的一系列文字，强烈建议仔细看完，人家04年写的东西，现在都2020年了我仍然学的津津有味，五味杂陈  
[关于通配符%](https://www.cnblogs.com/warren-wong/p/3979270.html)  
[关于模式规则](https://blog.csdn.net/haoel/article/details/2898?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)
## 示例项目
项目根路径为example,源代码位于example/src内，main.c会调用a.c和b.c包含的函数，其中a.c需要动态库gstreamer-1.0。编译过程中会在example中先生成obj文件夹，然后生成的.o文件都会放到该文件夹内，最后在example下生成可执行文件
```bash
hsq@ubuntu:~/code/example$ tree
.
├── src
│   ├── a.c
│   ├── a.h
│   ├── b.c
│   ├── b.h
│   └── main.c
└── Makefile

1 directories, 6 files
```


Makefile样本
```makefile
CROSS_COMPILE =#指定交叉编译器
DEBUG = 1#指定当前为debug模式
CC = $(CROSS_COMPILE)gcc#指定编译器
CFLAGS = -I/usr/include/gstreamer-1.0/ -Wall#指定头文件目录
LDFLAGS = -L/usr/lib/x86_64-linux-gnu/#指定库文件目录
LIBS = -lgstreamer-1.0#指定库文件名称
TARGET = test#最终生成的可执行文件名

VPATH = ./src/#告诉makefile去哪里找依赖文件和目标文件

#选择debug还是release
ifeq ($(DEBUG), 1)
CFLAGS+=-O0 -g
else
CFLAGS+=-Ofast
endif

OBJ = main.o a.o b.o#中间过程所涉及的.o文件
OBJDIR = ./obj/#存放.o文件的文件夹
OBJS = $(addprefix $(OBJDIR), $(OBJ))#添加路径

#指定需要完成的编译的对象
all: obj $(TARGET)

#将所有的.o文件链接成最终的可执行文件，需要库目录和库名，注意，OBJS要在LIBS之前。另外，如果要指定.o的生成路径，需要保证TARGET的依赖项是含路径的
$(TARGET):$(OBJS)
        $(CC) $(OBJS) $(CFLAGS) $(LDFLAGS) $(LIBS) -o $(TARGET)
#这个不是静态模式，而是通配符，第一个%类似bash中的*。
$(OBJDIR)%.o: %.c
        $(CC) -c $(CFLAGS) $< -o $@

#用于生成存放.o文件的文件夹
obj:
        mkdir obj
.PHONY : clean
clean :#删除生成的文件夹
         -rm -r obj
```

# 参考链接
[1][gcc的学习链接](http://c.biancheng.net/gcc/)  
[2][Makefile学习链接](https://blog.csdn.net/haoel/article/details/2886)