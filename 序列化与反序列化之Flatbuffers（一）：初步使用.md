# 序列化与反序列化之Flatbuffers（一）：基础使用说明
## 一: 前言
在MNN中, 一个训练好的静态模型是经过Flatbuffers序列化之后保存在硬盘中的. 这带来两个问题: 1.为什么模型信息要序列化不能直接保存 2.其他框架如caffe和onnx都是用Protobuf序列化, 为什么MNN要用Flatbuffers, 有啥优势? 在解答这两个问题之前, 我们先要了解什么是序列化和反序列化.
## 二: 什么是序列化和反序列化
什么是序列化和反序列化:
>序列化是指把一个实例对象变成二进制内容，本质上就是一个byte[]数组。 为什么要把实例对象序列化呢？因为序列化后可以把byte[]保存到文件中，或者把byte[]通过网络传输到远程，这样，就相当于把实例对象存储到文件或者通过网络传输出去了。 有序列化，就有反序列化，即把一个二进制内容（也就是byte[]数组）变回实例对象。有了反序列化，保存到文件中的byte[]数组又可以“变回”实例对象，或者从网络上读取byte[]并把它“变回”实例对象
+ 序列化：把对象转换为字节序列的过程。
+ 反序列化：把字节序列恢复为对象的过程。

对象的序列化主要有两种用途:
>+ 把对象的字节序列永久地保存到硬盘上，通常存放在一个文件中；（持久化对象）
>+ 在网络上传送对象的字节序列。（网络传输对象）

举例来说, 比如我用C++训练好一个模型, 然后在代码里是用一个类来描述这个模型的:
```c
class net{
private:
    string name;
    vector<layer> layers;
    ...
}
```
我可以把这个类的指针指向的内存块整个保存到硬盘中么? 要恢复的时候直接load到内存中, 这样不是最快的嘛? 但是这样会引入几个问题  
+ 我从32位机器保存的文件用64位机器打开就还原不了, 因为里面一些类型的sizeof不一样
+ 直接保存文件大小会比较大, 有些信息其实是可以进行压缩减小所需占用的硬盘空间
+ TODO:  

总之没有人会这样做, 大家都是通过某种序列化协议, 将要保存的对象经过"某种转换"变成包含同样信息的不同形式保存到硬盘中或者进行传送. 
## 三: 为什么要使用Flatbuffers
明白了保存文件时序列化的必要性后, 我们在选择序列化协议的时候主要考虑以下几点:
+ 协议是否支持跨平台
+ 序列化的速度
+ 序列化出来的大小

而Flatbuffers[官网](https://google.github.io/flatbuffers/index.html#flatbuffers_overview)对自己的介绍是这样的:
+ Access to serialized data without parsing/unpacking
+ Memory efficiency and speed
+ Flexible
+ Tiny code footprint
+ Strongly typed
+ Convenient to use
+ Cross platform code with no dependencies

其实Flatbuffers与Protobuf相比有以下几个优势:
+ 最大的一个特点是序列化和反序列化速度更快. 这是由于Flatbuffers将数据序列化成二进制buffer, 之后的数据反序列化的时候直接根据一些偏移信息读取这个buffer即可, 就是完善版的"我可以把这个类的指针指向的内存块整个保存到硬盘中么? 要恢复的时候直接load到内存中,这样不是最快的嘛?". 因此Flatbuffers经常用于游戏中与服务器频繁的通信, 但是感觉用于保存加载神经网络模型时, 相比于Protobuf应该优势不明显, 因为模型的load过程weight的访问占据了主要时间, 而反序列化模型的结构的时间减少应该对load过程加速不明显. 下次有空弄几个模型测一下.mark一下
+ 占用空间小, 使用简单, 适合移动端使用. Protobuf的头文件和库文件加起来有十几兆, 而Flatbuffers使用的时候只需要include一个头文件即可, 更加省空间. 同时简易程度简直是新手福音
+ 再加一个两者都有的优点. 代码的自动化生成. 编写一个fbs或者proto文件来描述需要管理的对象结构, 就可以一行命令生成所有对应的cpp类代码, 十分易于管理和修改. 可以节省很多头发. 我觉得这点才是这些开源神经网络框架使用Protobuf和Flatbuffers的最重要的原因吧.

## 四: 如何使用Flatbuffers
下面的重点是用于个人记录如何使用Flatbuffers来描述一个神经网络模型(其实就是MNN的方案)以及如何用C++代码进行序列化保存和反序列化读取. 本文的相关代码已经上传到[仓库](https://github.com/HuangShiqing/LearnAndTry/tree/main/flatbuffers)中, 欢迎使用和star  
 详细内容强烈推荐阅读[官方文档](https://google.github.io/flatbuffers/)
### 4.1 安装
```bash
git clone https://github.com/google/flatbuffers
cd flatbuffers
mkdir build
cd build
cmake .. && cmake --build . --target flatc -- -j4
```
最终在目录flatbuffers/build下得到一个flatc的可执行文件即可. 安装过程一气呵成, 对比Protobuf的版本问题和一堆依赖库问题, 简直不要太舒服
### 4.2 编写fbs
使用Flatbuffers和Protobuf很相似, 都会用到先定义一个schema文件, 用于定义我们要序列化的数据结构的组织关系. 下面我们以描述一个极简神经网络模型PiNet为例(没错, 就是浓缩版的MNN), 介绍Flatbuffers常用结构int, string, enum, union, vector, table的使用方法
```c
namespace PiNet;//命名空间千万不能与内部的对象重名!!!!!!

table Pool {
    padX:int;
    padY:int;
    // ...
}
table Conv {
    kernelX:int = 1;
    kernelY:int = 1;
    // ...
}
union OpParameter {
    Conv,
    Pool,
}
enum OpType : int {
    Conv,
    Pool,
}
table Op {
    type: OpType;
    parameter: OpParameter;
    name: string;
    inputIndexes: [int];
    outputIndexes: [int];
}
table Net {
    oplists: [Op];
    tensorName: [string];
}
root_type Net;
```
我们的根类型是Net代表一个神经网络模型, 是net是table类型, 该类型应该是最常用的类型, 类似Python中的字典, 冒号左边是名key, 右边是数据类型value. []代表是数组vector. 由此可见一个Net包含多个层Op, 多个tensor. 然后我们重点看Op, 用一个enum来代表Op的类型, 以及一个union来表示该Op的parameter. enum的概念在C/C++中也有, 这里也是一样的, 就是内存空间复用. 而union其实就是非int的enum的, 这里是一个table的enum. 对每种Op的parameter都定义了一个table来描述, Conv层这里只包含了两个参数kernelX和kernelY. 其他都好理解, 这里稍微需要注意的是这个union概念的理解.

### 4.3 产生generated.h文件
```bash
/flatbuffers/build/flatc net.fbs --cpp --binary --reflect-names --gen-object-api --gen-compare
```
将编写好的fbs文件转换成可用的.h文件. 命令中--gen-object-api表示.h文件中会产生方便使用的xxT类. --gen-compare表示.h文件中每个类都会产生Operator==方法, 用于比较各对象是否相等.
我们来大致看一下产生的.h文件中有哪些内容.  
```c
enum OpType : int32_t {
  OpType_Conv = 0,
  OpType_Pool = 1,
  OpType_MIN = OpType_Conv,
  OpType_MAX = OpType_Pool
};
```
fbs中的enum就转换成了C/C++的enum, 这个好理解不用多说

```c
struct Op;
struct OpBuilder;
struct OpT;
```
fbs中定义的table结构都变成了struct, 以Op为例, 产生了3个结构, 其中Op是用于描述序列化后的Op对象, OpT是用于描述未序列化的Op对象, 这个XXT是需要我们在编译的时候加上选项--gen-object-api才会产生
```c
struct OpT : public flatbuffers::NativeTable {
  typedef Op TableType;
  PiNet::OpType type = PiNet::OpType_Conv;
  PiNet::OpParameterUnion parameter{};
  std::string name{};
  std::vector<int32_t> inputIndexes{};
  std::vector<int32_t> outputIndexes{};
};
```
OpT的结构其实就是我们想要描述Op的样子. 由此可见, 使用Flatbuffers的一大好处就是方便, 只需要用fbs几行描述好即可自动产生对应的类对象代码. 抛开序列化不说, 光这代码自动生成就够让人心动了, 简直是懒人福音
```c
struct Op FLATBUFFERS_FINAL_CLASS : private flatbuffers::Table {
  enum FlatBuffersVTableOffset FLATBUFFERS_VTABLE_UNDERLYING_TYPE {
    VT_TYPE = 4,
    VT_PARAMETER_TYPE = 6,
    VT_PARAMETER = 8,
    VT_NAME = 10,
    VT_INPUTINDEXES = 12,
    VT_OUTPUTINDEXES = 14
  };
  PiNet::OpType type() const ;
  PiNet::OpParameter parameter_type() const ;
  const void *parameter() const ;
  template<typename T> const T *parameter_as() const;
  const PiNet::Conv *parameter_as_Conv() const;
  const PiNet::Pool *parameter_as_Pool() const;
  const flatbuffers::String *name() const ;
  const flatbuffers::Vector<int32_t> *inputIndexes() const ;
  const flatbuffers::Vector<int32_t> *outputIndexes() const ;
  bool Verify(flatbuffers::Verifier &verifier) const ;
  OpT *UnPack() const;
  void UnPackTo() const;
  static flatbuffers::Offset<Op> Pack();
};
```
再看Op这个结构, Op描述的是序列化后的对象, 成员函数中主要是包含了从Op直接访问成员变量的方法(其实就是反序列化), 还有一个VTableOffset的enum, 这个我们下篇详解的时候会用到, 这里暂且不表. 另外还包含了两个Pack和UnPack方法, 顾名思义, 这就是序列化和反序列化方法. UnPack可以将序列化后的对象Op转换成未序列化的对象OpT, Pack可以将OpT转换成Op.
```c
struct OpParameterUnion {
  OpParameter type;
  void *value;

  static void *UnPack();
  flatbuffers::Offset<void> Pack() const;

  PiNet::ConvT *AsConv() {
    return type == OpParameter_Conv ?
      reinterpret_cast<PiNet::ConvT *>(value) : nullptr;
  }
  const PiNet::ConvT *AsConv() const {
    return type == OpParameter_Conv ?
      reinterpret_cast<const PiNet::ConvT *>(value) : nullptr;
  }
  PiNet::PoolT *AsPool() {
    return type == OpParameter_Pool ?
      reinterpret_cast<PiNet::PoolT *>(value) : nullptr;
  }
  const PiNet::PoolT *AsPool() const {
    return type == OpParameter_Pool ?
      reinterpret_cast<const PiNet::PoolT *>(value) : nullptr;
  }
};
```
再看Union, 包含了两个成员变量, 一个描述类型, 一个存放数据指针. 对于一个实例化的Union, 想要得到其代表的数据, 需要根据类型手动执行相应的AsXXX函数进行cast. 刚才Op的结构里包含的几个函数parameter_type(),  parameter(), parameter_as_Conv(),  parameter_as_Pool()就是封装了这个Union的成员函数
```c
bool operator==(const PoolT &lhs, const PoolT &rhs);
bool operator!=(const PoolT &lhs, const PoolT &rhs);
bool operator==(const ConvT &lhs, const ConvT &rhs);
bool operator!=(const ConvT &lhs, const ConvT &rhs);
bool operator==(const OpT &lhs, const OpT &rhs);
bool operator!=(const OpT &lhs, const OpT &rhs);
bool operator==(const NetT &lhs, const NetT &rhs);
bool operator!=(const NetT &lhs, const NetT &rhs);
```
编译选项里开了--gen-compare后就会自动产生这些比较操作符重载代码, 这里再一次展示出了Flatbuffers的方便特性, 试想一下, 如果我有100种Op参数都要手动去写比较操作符重载, 那岂不是得累死. 
### 4.4 序列化代码
```c
#include <fstream>
#include <iostream>

#include "net_generated.h"
using namespace PiNet;

int main() {
    flatbuffers::FlatBufferBuilder builder(1024);

    // table ConvT
    auto ConvT = new PiNet::ConvT;
    ConvT->kernelX = 3;
    ConvT->kernelY = 3;
    // union ConvUnionOpParameter
    OpParameterUnion ConvUnionOpParameter;
    ConvUnionOpParameter.type = OpParameter_Conv;
    ConvUnionOpParameter.value = ConvT;
    // table OpT
    auto ConvTableOpt = new PiNet::OpT;
    ConvTableOpt->name = "Conv";
    ConvTableOpt->inputIndexes = {0};
    ConvTableOpt->outputIndexes = {1};
    ConvTableOpt->type = OpType_Conv;
    ConvTableOpt->parameter = ConvUnionOpParameter;

    // table PoolT
    auto PoolT = new PiNet::PoolT;
    PoolT->padX = 3;
    PoolT->padY = 3;
    // union OpParameterUnion
    OpParameterUnion PoolUnionOpParameter;
    PoolUnionOpParameter.type = OpParameter_Pool;
    PoolUnionOpParameter.value = PoolT;
    // table Opt
    auto PoolTableOpt = new PiNet::OpT;
    PoolTableOpt->name = "Pool";
    PoolTableOpt->inputIndexes = {1};
    PoolTableOpt->outputIndexes = {2};
    PoolTableOpt->type = OpType_Pool;
    PoolTableOpt->parameter = PoolUnionOpParameter;

    // table NetT
    auto netT = new PiNet::NetT;
    netT->oplists.emplace_back(ConvTableOpt);
    netT->oplists.emplace_back(PoolTableOpt);
    netT->tensorName = {"conv_in", "conv_out", "pool_out"};
    netT->outputName = {"pool_out"};
    // table Net
    auto net = CreateNet(builder, netT);
    builder.Finish(net);

    // This must be called after `Finish()`.
    uint8_t* buf = builder.GetBufferPointer();
    int size = builder.GetSize();  // Returns the size of the buffer that
                                   //`GetBufferPointer()` points to.
    std::ofstream output("net.mnn", std::ofstream::binary);
    output.write((const char*)buf, size);

    return 0;
}
```
由于我们开启了--gen-object-api选项会产生XXT的结构, 我们只需要对各个层次的数据结构进行赋值即可, 最后只要对根节点进行一次Create即完成序列化, 很简单方便. 相比于官网的创建monster那样每个层次的数据都要Create序列化一下, 代码结构能精简不少.
### 4.5 反序列化
```c
#include <fstream>
#include <iostream>
#include <vector>

#include "net_generated.h"
using namespace PiNet;

int main() {
    std::ifstream infile;
    infile.open("net.mnn", std::ios::binary | std::ios::in);
    infile.seekg(0, std::ios::end);
    int length = infile.tellg();
    infile.seekg(0, std::ios::beg);
    char* buffer_pointer = new char[length];
    infile.read(buffer_pointer, length);
    infile.close();

    auto net = GetNet(buffer_pointer);

    auto ConvOp = net->oplists()->Get(0);
    auto ConvOpT = ConvOp->UnPack();

    auto PoolOp = net->oplists()->Get(1);
    auto PoolOpT = PoolOp->UnPack();

    auto inputIndexes = ConvOpT->inputIndexes;
    auto outputIndexes = ConvOpT->outputIndexes;
    auto type = ConvOpT->type;
    std::cout << "inputIndexes: " << inputIndexes[0] << std::endl;
    std::cout << "outputIndexes: " << outputIndexes[0] << std::endl;

    PiNet::OpParameterUnion OpParameterUnion = ConvOpT->parameter;
    switch (OpParameterUnion.type) {
        case OpParameter_Conv: {
            auto ConvOpParameterUnion = OpParameterUnion.AsConv();
            auto k = ConvOpParameterUnion->kernelX;
            std::cout << "ConvOpParameterUnion, k: " << k << std::endl;
            break;
        }
        case OpParameter_Pool: {
            auto PoolOpParameterUnion = OpParameterUnion.AsPool();
            auto k = PoolOpParameterUnion->padX;
            std::cout << "PoolOpParameterUnion, k: " << k << std::endl;
            break;
        }
        default:
            break;
    }
    return 0;
}
```
## 五: 总结
Flatbuffers的使用还是挺简单的, 理解常见的几种数据类型, 再把官网那个monster的例子看几遍就还挺好懂的. 本篇只是初步使用, 下一篇我们再做深入剖析.