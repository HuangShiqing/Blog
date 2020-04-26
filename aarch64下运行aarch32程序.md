# 前言
能否在arm芯片的aarch64模式下运行aarch32的程序呢？
# 查看
我们先查看两个分别用gcc-arm-linux-gnueabihf和aarch64-linux-gnu编译出来两个程序文件appv7和appv8的文件信息
``` bash
hsq@ares:~$ file appv7#查看appv7的文件信息
appv7: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-, for GNU/Linux 3.2.0, BuildID[sha1]=f2e4fcbe37df253c60b2ac2de888558e9916dbdb, not stripped#32位arm文件，有依赖的动态库，还有一个解释器/lib/ld-我们稍后再分析

hsq@ares:~$ objdump -x appv7 | grep NEEDED#查看appv7的依赖包
NEEDED  libm.so.6#基础包
NEEDED  libgcc_s.so.1
NEEDED  libc.so.6#基础包
NEEDED  ld-linux-armhf.so.3#armhf的一个库，上面file出来的信息里/lib/ld-就是指这个

hsq@ares:~$ file appv8#查看appv8的文件信息
appv8: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-, for GNU/Linux 3.7.0, BuildID[sha1]=f2a550a34a3266b3fbc92f96b8704c7ffae71032, not stripped#aarch64的文件，有依赖的动态库，也需要解释器/lib/ld-

hsq@ares:~$ objdump -x appv8 | grep NEEDED#查看appv8的依赖包
NEEDED               libm.so.6
NEEDED               libc.so.6
NEEDED               ld-linux-aarch64.so.1#也有需要的解释器
```
[什么是ld-linux-]()  
# 运行
我的环境是aarch64，直接可以运行appv8没问题，但是直接运行appv7会报错，不是appv7找不到，其实就是解释器找不到
```bash
hsq@ares:~$ ./appv7
-sh: appv7: not found
```
那么就把gcc-arm-linux-gnueabihf里lib文件夹里ld-linux-armhf.so.3复制到/lib里，这里我试过用LD_LIBRARY_PATH和ld.so.conf.d里添加ld-linux-armhf.so.3所在路径，但是不行，貌似只能复制到/lib里才行

再试试行不行
```bash
hsq@ares:~$ ./appv7
error while loading shared libraries: libm.so.6: wrong ELF class: ELFCLASS64
```
还是不行，但是很明显了，虽然两个appv7和appv8都依赖同名库文件libm.so.6和libc.so.6，但是还是要注意库文件类别不同，因此直接引入gcc-arm-linux-gnueabihf的lib文件路径，建议命令行里用LD_LIBRARY_PATH暂时加入
```bash
hsq@ares:~$ export LD_LIBRARY_PATH=path
hsq@ares:~$ ./appv7
```
运行，ok！