## 前言
学会调试生成的程序是一种能力，而使用好工具更是能力的体现。想必gdb不用过多介绍。本文主要介绍如何用gdb命令行远程调试目标板上的程序以及如何配合vscode可视化远程调试目标板上的程序
## 编译安装
[下载链接](http://ftp.gnu.org/gnu/gdb/)  
我下载的是7.8.1，好像说新版的不太好。
下载好之后解压进入,执行下述命令进行配置安装:
``` bash
hsq@ares:~$ cd gdb-7.8.1/
hsq@ares:~/gdb-7.8.1$ ./configure --target=arm-linux-gnueabihf --prefix=$PWD/build # target是交叉编译器的名字，prefix是准备安装的位置
hsq@ares:~/gdb-7.8.1$ make 
hsq@ares:~/gdb-7.8.1$ make install # 执行完这句就可以在刚才指定的位置$PWD/build/bin/下生成可执行文件arm-linux-gnueabihf-gdb
hsq@ares:~/gdb-7.8.1$ cd gdb/gdbserver/
hsq@ares:~/gdb-7.8.1/gdb/gdbserver/$ ./configure --target=arm-linux-gnueabihf --host=arm-linux-gnueabihf
hsq@ares:~/gdb-7.8.1/gdb/gdbserver/$ make # 执行完这句就在当前目录下生成了可执行文件gdbserver
```
经过上述命令就得到了2个可执行文件：arm-linux-gnueabihf-gdb和gdbserver，前者用在宿主机上执行，后者拷贝到目标板上执行，都放到/usr/bin目录下以便于在命令行里直接运行
## 远程debug程序(remote gdb)
假设需要调试一个编译好的可执行文件test，宿主机和目标板在一个局域网内，都能访问到该目录/test/，目标板的ip是192.168.1.100，宿主机的ip是192.168.1.105
### 使用操作
+ 目标板
    ```bash
    pi@raspberrypi:/mnt/test/$ ls # 该目录包含源码和编译好可执行文件
    test.c test
    pi@raspberrypi:/mnt/test/$ gdbserver :1234 ./test # 启动gdbserver,不指定ip只指定端口为1234，开始监听
    Process ./neon_test created; pid = 5817
    Listening on port 1234
    ```
+ 宿主机
    ```bash
    hsq@ares:~/Public/nfs/test$ ls # 同样能访问到该目录，建议采用nfs方式共享该文件夹
    test.c test
    hsq@ares:~/Public/nfs/test$ arm-linux-gnueabihf-gdb ./test -tui # 启动
    (gdb) target remote 192.168.1.105:1234 # 连接目标板上的程序，如果成功目标板会输出一句Remote debugging from host 192.168.1.105
    (gdb) b test.c:5 # 设置断点
    (gdb) c # 开始调试
    ```
    TODO：动态库加载。warning: Unable to find dynamic linker breakpoint function.
+ 疑惑
既然目标板和宿主机都是命令行，那我干嘛不ssh登陆目标板后直接在目标板上gdb呢？干嘛还要这么麻烦呢？？？迷惑行为
## gdbserver配合vscode进行远程可视化调试
利用命令行的远程gdb是最原始的，能够直接用命令调试，比较硬核，但是有时候不太方便，用vscode的话就和日常debug一样方便，所需要做的就是配置launch.json文件即可。在F5之前需要和上面操作一样在目标板端开始监听。用vscode缺点就是调试汇编程序不方便，不能查看寄存器
TODO:有没有方法用vscode查看汇编寄存器
```c
.vscode/launch.json:
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "remote gdb",
            "type": "cppdbg",
            "request": "launch",            
            "program": "${fileDirname}/neon_test",//需要调试的可执行文件
            "miDebuggerServerAddress": "192.168.1.100:1234",//目标板的ip地址和端口
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceRoot}",
            "environment": [],
            "externalConsole": true,
            "MIMode": "gdb",
            "miDebuggerPath": "/home/hsq/DeepLearning/raiberry/gdb/gdb-7.8.1/build/bin/arm-linux-gnueabihf-gdb",//交叉编译器gdb
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ],
            // "preLaunchTask": "gcc build active file",
        }
    ]
}
```
## gdb调试指令
### 启动gdb
+ gdb ./test
调试可执行文件test
+ gdb ./test -tui
带显示的调试可执行文件test
### 断点
+ info b
查看已经设置的断点
+ b file:line
根据文件名和行数设置断点
+ b fun
根据函数名字设置断点
+ delete b 1
删除序号为1的断点
### 执行
+ start
开始debug，运行到main函数的第一句停下。但是远程gdb好像不能用这个，只能用c
+ c
continue，执行到断点出
+ s
step in，下一步，会进入函数
+ n
step over，下一步，会跳过函数
+ finish
step out，运行到退出当前函数
### 查看
+ info registers \$r1
查看寄存器r1的值，传入是值则查看的也是值，传入的是指针则查看的是地址（这里加不加$都一样）
+ p /x $r1
传入的是指针则查看的是地址，传入的是值则不需要/x用默认的/d即可查看传入的值
+ p $q0
查看neon寄存器里的值
+ x /nfu addr
    + n表示要显示的内存单元的个数
    + f表示显示方式, 可取如下值
x 按十六进制格式显示变量。
d 按十进制格式显示变量。
u 按十进制格式显示无符号整型。
o 按八进制格式显示变量。
t 按二进制格式显示变量。
a 按十六进制格式显示变量。
i 指令地址格式
c 按字符格式显示变量。
f 按浮点数格式显示变量。
    + u表示一个地址单元的长度
b表示单字节，
h表示双字节，
w表示四字节，
g表示八字节
    + x/1fw $r1
查看1个浮点数，传入的必须是指针，查看的是指向的值

## 参考链接
1. [gdb+gdbserver远程调试技术](https://blog.csdn.net/zhaoxd200808501/article/details/77838933)
1. [vscode+gdbserver的配置](https://medium.com/@spe_/debugging-c-c-programs-remotely-using-visual-studio-code-and-gdbserver-559d3434fb78)