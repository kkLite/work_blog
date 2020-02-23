# 简述
AI已经侵入我们的生活，人脸识别、语言识别、智能音响等等无处不在，但很早之前的AI数据的处理、模型推理等都在云端，随着时间和需求的变化，聪明的研究者开始把AI的推理训练放到终端（手机设备、IoT设备等）来做，[MNN框架](https://github.com/alibaba/MNN)是阿里推出的`端侧推理引擎

本文主要目的在于介绍下端上计算的过程，了解什么是端上计算，为移动开发者入门学习提供思路，下面我们来一起探讨学习下吧，如果对你有用记得点赞鼓励❤

# 为什么要进行端上计算

## 传统智能计算的过程

简述下基本流程

1. 终端收集数据到服务端
2. 服务端根据收集的数据使用算法进行模型训练，最终产生可以商用的模型，然后部署
3. 这一步就是推理阶段了，终端把用户的数据传送给服务端，服务端利用模型进行推理计算，然后返回结果给终端
4. 终端根据结果做相应的操作

由上面的流程可知，存在几个问题

1. 数据隐私性，需要把数据上报给服务端，而且如果用户不授权的话则收取不到
2. 网络延迟，终端必须等待网络请求完成才能做出响应，这不适合实时操作强的任务
3. 算力集中，完全依赖服务器的算力

## 端上计算优势

端上计算其实是把推理和训练过程放在终端进行，这样做有如下优势

1. 响应速度快，不依赖云端服务的请求，可快速做出决策，目前很多应用已经在使用，如淘宝、咸鱼、抖音等
2. 数据隐私安全，政策对数据隐私的管控越来越严格，想把数据上传到服务端越来越难
3. 节省服务器资源
4. 发挥联合计算的优势，虽然一台手机/IoT设备的算力有限，但是如果把所有的设备算力集合在一起，将远远超越云服务的算力，不过这个方向还在发展中

# 安装
本文以iOS平台为例进行安装，相比于其他库，`MNN`的安装略微有点复杂，[官方文档](https://www.yuque.com/mnn/cn/build_ios)也不是特别健全(根据个人爱好可以参照官网的教程)，总之遇到问题就解决吧

## 下载`MNN`源码

```
git clone https://github.com/alibaba/MNN.git
```

## 安装操作
**第一步**：

进入仓库的主目录：`MNN`

**第二步：** 

基础依赖项的检查：`cmake`、`protobuf`、`C++编译器（gcc\clang）`，如果没有安装自行查找安装

官方对依赖的版本要求及检查命令如下表：
| 依赖项    | 建议版本                                                     | 检查是否安装的命令          |
| --------- | ------------------------------------------------------------ | --------------------------- |
| cmake     | 理论依赖3.0以上即可，但建议版本以及测试环境中的版本都为3.10+ | `cmake -v`                  |
| protobuf  | version >= 3.0 is required                                   | `protoc --version`          |
| C++编译器 | `GCC`或`Clang`皆可,version >= 4.9，对于macOS已经默认安装`Clang` | `clang -v` 或者    `gcc -v` |

**第三步：** 

编译模型转化器：`MNNConvert`，整个编译过程需要几分钟

```
1. cd MNN/
2. ./schema/generate.sh
3. mkdir build
4. cd build
5. cmake .. -DMNN_BUILD_CONVERTER=true && make -j4
```
**第四步：(可选)** 

下载demo需要的模型然后转换成`MNN`支持的模型，主要用于demo工程使用，可以跳过

特别注意： 这一步会经常遇到两个问题：

1. 忘记先编译好模型转化器：`MNNConvert`（第三步）
2. 下载demo模型时，容易下载失败，可能是被墙或其他原因，如果一直失败的话可以把模型手动下载下来，然后进行转换（大致操作就是修改`./tools/script/get_model.sh`脚本，注释掉下载逻辑，然后手动下载完成之后，再执行下面的命令，操作不成功的可以评论或者加微信交流）

在`MNN`主目录下执行如下命令

```
./tools/script/get_model.sh
```
**第五步：**

用`Xcode`打开`project/ios/MNN.xcodeproj`，点击编译即可，这样就把完整的`MNN`库编译出来了

**总结：**

其实这个安装过程做了三件事
1. 编译模型转换器`MNNConvert`，主要用来把其他框架产生的模型（如：tf、caffemodel）转换成`MNN`可用的模型
2. 下载demo模型，并利用`MNNConvert`转换成`MNN模型`，这一步是为了demo工程提供模型的，所以可选步骤
3. 编译`MNN`库

总体看来，如果你是用来开发和调试源码的话，那么`MNNConvert`肯定要编译好，MNN库也要编译好，如果只是想看下demo工程，那么上面的那些都不用做，直接下载下面的demo工程跑起来看下

## demo工程

学习一个框架直接的方式就是看它的demo如何使用，上面的安装步骤提到，由于demo用到了第三方开源的模型，所以**必须要先完成上面安装操作的第四步**， 然后找到路径`MNN/demo`下的demo工程，打开直接run即可

在demo工程可以进行源码调试、api用法等，赶快去学习吧！

# 执行推理的过程

不考虑技术细节站在更高的角度看`MNN`，其实`MNN`主要做的事情就是推理(之后可能还会增加训练)，那么猜测下需要做哪些事情如：加载模型、输入数据、输出结果等。我们以demo工程为例来研究下推理的过程

## 模型加载：Interpreter & Session

整个过程我们把它比喻成浏览器加载网页

**Interpreter：**模型解释器，就像浏览器的引擎则用来解释网页、分析网页结构等进行渲染

**Session：** 会话，网页渲染完成，总要进行会话操作的，比如请求个图片、点击button等，就是一次通信过程，而在`MNN`中，`Session`就是一次推理的过程，持有推理数据



所以`Interpreter`加载模型，`Session`负责推理数据，`Session`通过`Interpreter`创建，一个`Interpreter`可以创建多个`Session`，下面看下源码解析

### Interpreter解释器和Session对象

**声明**

```objective-c
@interface Model : NSObject {
    std::shared_ptr<MNN::Interpreter> _net; // 解释器，用来持有模型数据的
    MNN::Session *_session; // 会话，推理数据的持有者
}
```

在`Model`类中，声明了解释器对象`_net`和会话对象`_session`，这是个基类，因为不同类型的模型会创建自己的`Model`，并继承`Model`类

**创建Interpreter**

```objective-c
// 获取模型所在的路径
NSString *model = [[NSBundle mainBundle] pathForResource:@"mobilenet_v2.caffe" ofType:@"mnn"];
// 从磁盘加载模型，当然也是支持从内存读取的
_net = std::shared_ptr<MNN::Interpreter>(MNN::Interpreter::createFromFile(model.UTF8String));

```

**创建会话**

```objective-c
- (void)setType:(MNNForwardType)type threads:(NSUInteger)threads {
    MNN::ScheduleConfig config; // 一些配置，暂时忽略
    config.type      = type;
    config.numThread = (int)threads;
    if (_session) { // 释放旧的session
        _net->releaseSession(_session);
    }
  	// 创建新session
    _session = _net->createSession(config);
}
```

到此模型加载和准备工作就完成了，下面介绍下如何输入数据

## 输入和运行数据

模型加载完成，就要开始推理了，其实不要想太复杂，就几个接口而已，作为入门先看懂骨架再看细节吧

**数据容器**

输入数据，自然需要容器承载这些数据，在`MNN`中设计了`Tensor`类用来作为数据的容器，看下面的解释

```c++
/**
 * data container. 这个注释已经说明，这个类是数据的容器
 * data for host tensor is saved in `host` field. its memory is allocated malloc directly.
 * data for device tensor is saved in `deviceId` field. its memory is allocated by session's backend.
 * usually, device tensors are created by engine (like net, session).
 * meanwhile, host tensors could be created by engine or user.
 */
class MNN_PUBLIC Tensor {
public:
    struct InsideDescribe;

    /** dimension type used to create tensor */
    enum DimensionType {
        /** for tensorflow net type. uses NHWC as data format. */
        TENSORFLOW,
        /** for caffe net type. uses NCHW as data format. */
        CAFFE,
        /** for caffe net type. uses NC4HW4 as data format. */
        CAFFE_C4
    };

    /** handle type */
    enum HandleDataType {
        /** default handle type */
        HANDLE_NONE = 0,
        /** string handle type */
        HANDLE_STRING = 1
    };
  ....
}
```

**输入数据&运行**

数据输入返回的就是`Tensor`对象，核心代码就一句（多种输入方式不多介绍，本文以入门为主）

* 输入数据函数`getSessionInput` 的定义

```c++
Tensor* Interpreter::getSessionInput(const Session* session, const char* name){}
```

* demo工程的示例代码

```objective-c
// 输入数据，返回Tensor对象input
auto input = _net->getSessionInput(_session, nullptr);
MNN::Tensor tensorCache(input);
input->copyToHostTensor(&tensorCache);
for (int i = 0; i < cycles; i++) {
    input->copyFromHostTensor(&tensorCache);
  	// 开始run
    _net->runSession(_session);
}
...
```

## 获取结果

即在推理run完成之后，我们需要获取结果，返回的也是`Tensor`对象

* 获取结果的函数`getSessionOutput`定义

```c++
Tensor* Interpreter::getSessionOutput(const Session* session, const char* name) {}
```

* demo工程示例代码

```objective-c
// 获取结果对象output
MNN::Tensor *output = _net->getSessionOutput(_session, nullptr);
MNN::Tensor copy(output);
output->copyToHostTensor(&copy);
float *data = copy.host<float>();
...
```

## 小结

这个过程旨在介绍整个框架的使用过程，结合demo工程理解更快，并没有过多深入，希望能快速入门



# MNNKit

有时候会思考一些问题：其实进行智能计算，我们只需要一个合适的模型然后利用它进行推理计算，那么模型就是可以复用的，没必要每个人都要重新训练一套模型

对于这个问题，阿里就把超级成熟的模型放出来给大家用了，这些模型已经在淘宝等app上实践过，它就是[MNNKit](https://github.com/alibaba/MNNKit)， 基于端上推理引擎[MNN](https://github.com/alibaba/MNN)提供的系列应用层解决方案，目前开放了三个模型，有需要的开发者可以直接拿来玩了

| Kit SDK              | 功能  |
| -------------------- | -------- |
| FaceDetection        | 人脸识别 |
| HandGestureDetection | 手势识别 |
| PortraitSegmentation | 人像分割 |

## MNNKit的组织结构

(直接引用了GitHub库的描述)

![](https://tva1.sinaimg.cn/large/0082zybply1gc5ij1bzhkj30g105lmwz.jpg)

**MNN层：**最核心层，提供了模型加载和推理

**MNNKit Core：**主要针对MNN的C++接口的封装和抽象，转化成更高级别的OC或Java实现的API，方便调用者使用

**业务层：**主要针对具体的算法模型的封装，主要给业务使用的

# 总结

端上计算需要做什么？对于移动开发者来说初次接触AI开发很是陌生，该怎么入手AI开发呢，个人目前也在学习中，有一些深刻又吐血的经验分享：

> 如果你对数学有一定的熟悉的话，千万别one by one的从头开始学数学，这真的太低效了，建议先从开发者最熟悉的代码层面入手，了解AI的基本工作流程，如学学`MNN`怎么用的，你就会知道原来就是拿模型去推理然后得到结果反馈给用户
>
> 然后在学会怎么训练模型，建议从简单的机器学习算法入手，根据算法需要用的数学公式，然后查这些公式的用法，边学边用更好理解



整体下来介绍了什么是端上计算及优势，对MNN库有了基本的介绍，到这相信大家已经有了基本的了解。



**最后欢迎关注笔者公众号**：【码上work】，本公众号致力于浅显易懂的方式讲解移动/全栈开发、有趣算法、计算机基础知识等，帮你构建完整的知识体系，一起成为顶级开发者。公众号回复`12345`有大量学习资料：

1. iOS电子书（整理好组织结构）
2. 全栈开发视频教程（全套）
3. 机器学习视频（全套）
4. 区块链视频教程（全套）

![](https://tva1.sinaimg.cn/large/0082zybply1gc6dlpqi8wj3092050jrz.jpg)

