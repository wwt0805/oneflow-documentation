## 术语

以下术语和定义适用于本文件。

### 1.张量

张量即Tensor，为深度学习领域数据存储的基本单元，可对其进行前向和反向过程。

### 2. Device：

张量计算设备，如常见的cpu、gpu等。

### 3.算子

算子(Operator)简称Op，即深度学习框架中最小功能函数的定义，正是由这些op组合在一起才可以完成各种复杂的张量计算。

### 4.Kerenel

一个Op定义了一个功能函数，对应的函数具体实现由Kernel完成，且由于device类型不同，一个Op可以对应多个Kernel，即同样的Op在不同device上可能由不同Kernal来实现。



## 编译层接口说明

编译层接口位于用户接口和运行时接口之间，在框架正式开始执行深度学习训练/推理等任务之前，编译层将会对用户在用户层定义的各种模型结构、参数、执行任务等进行处理、资源分配、整理和编译，最终转化为由**张量**构成的计算图，从而方便运行时的执行调用。

以下是一个编译后的得到计算图图示:

![plan_illustration](https://docs.oneflow.org/arch_design/imgs/plan_illustration.svg)



编译层接口应满足以下功能或要求，包括但不限于：

- 提供通用规范的API，方便用户层接口调用
- 可以使用至少一种编程语言进行调用、具备使用多种编程语言调用的拓展性
- 具备张量计算构图模块，可将用户定义的深度学习网络结构、训练/推理等任务转换成计算图
- 具备张量编译器模块，可对计算图结构进行拓扑分析、编译运行时图结构优化的功能
- 提供第三方张量优化/编译模块(如：XLA、TensorRT等)接入接口，以实现张量编译器功能的可拓展性
- 支持在原有计算图上动态增加、删除、修改、查询子图/节点的功能
- 对机器资源、GPU资、CPU、内存等软硬件资源等做好充分的申请和运行前规划
- 需要兼容单机任务、多机分布式任务的通用编译接口API
- 提供通用规范的API，对接运行时接口



## 运行时接口说明

运行时接口是深度学习框架的核心接口，包含但不限于以下几种子模块：

- 去中心化控制器模块

- 张量任务执行模块

- 张量状态跟踪模块

- 数据存储模块

- 网络通信模块

下面，将分别对各个子模块进行说明。

### 去中心化控制器模块

传统的深度学习框架大多采用类似master-slave的中心化控制的模式，此模式通常在单节点下可以良好运行，但拓展至分布式多机时，可能会带来严重的通信开销、数据传输开销、导致单节点瓶颈会拖累整个集群，影响深度学习任务的执行效率。为此，我们需要以中心化的方式，对深度学习任务进行控制。

去中心化控制器模块，应具备但不限于如下能力：

- 不设置master主控制器，控制器模块应该去中心化
- 每个控制器独立负责一个张量子图任务，控制和指挥子图任务的执行、状态跟踪、数据存储等
- 对于两个有上下游依赖关系的张量子图任务，两个控制器之间可以流水线式接力工作
- 对于没有依赖关系的张量子图任务，控制器之间可以并行工作，互不干涉
- 控制器模块之间可通过网络通信进行交流沟通



### 张量任务执行模块

张量任务执行模块，是执行张量子图任务的核心，任务执行的具体实施者，其应满足但不限于如下能力：

- 接受去中心化控制器模块的调度和指挥

- 具备高效执行张量子图任务的能力

- 具备一定的容错性

  

### 张量状态跟踪模块

张量状态跟踪模块，负责记录和跟踪张量子图任务执行期间的各种状态变化，其应满足但不限于如下能力：

- 张量子图状态记录和更新
- 高效性、可靠性、实时性
- 有且只有一个去中心化控制器对其负责



### 数据存储模块

数据存储模块，复制存储张量子图任务执行期间产生的数据存储需求，其应满足但不限于如下能力：

- 支持基于硬盘、内存等多种存储方式的灵活切换
- 高效性、可靠性



### 网络通信模块

网络通信模块包含两个部分：数据层面通信、控制层面通信，其应满足但不限于如下能力：

- 支持基于gRPC等方式的控制层面通信

- 支持基于nccl、mpi、epoll、rdma等方式的数据层面通信

- 高效性、可靠性

  



# 框架性能评测

我们需要对深度学习框架进行性能测试，其目的是为了从速度和性能来衡量深一个深度学习框架的优劣，也能更加科学的评估框架软件架构设计与性能之间的关系。

性能评测的方式主要为横向对比，即通过对比不同深度学习框架，在相同软硬件条件下、相同的深度学习模型任务下，考察深度学习框架的吞吐率和加速比。

## 名词解释

性能指标通常不考虑模型精度，而是以速度相关的指标如吞吐率、加速比等作为性能评判的标准。

### 吞吐率

吞吐率是每秒平均处理的训练样本数量，吞吐率的单位可以根据该领域的实际任务进行灵活变动，如图像分类：images/second、NLP：words/second等。

### 加速比

加速比，是用于衡量分布式横向拓展情况下，框架性能的指标。理想情况下的加速比为线性加速比，在线性加速比时，框架的吞吐率(速度)随着机器节点数的增加而1:1地线性增加。

## 性能评测维度

性能评测需要覆盖单机、多机的情况，且至少需要覆盖如下维度：

- 单机1卡
- 单机4卡
- 单机8卡
- 2机16卡
- 4机32卡

## 性能评测要求

- 1.硬件环境一致
- 2.软件环境一致
- 3.网络结构一致
- 4.数据集一致
- 5.流程可复现
- 6.中值原则
- 7.最优化配置原则

### 1.硬件环境一致

需保证各个框架评测时所用的机器硬件条件完全一致，典型的硬件设备主要包括：CPU、GPU、内存等。典型的硬件示例：

- Tesla V100-SXM2-16GB x 8

- InfiniBand 100 Gb/sec (4X EDR)， Mellanox Technologies MT27700 Family

- Intel(R) Xeon(R) Gold 5118 CPU @ 2.30GHz

- Memory 384G

  

### 2.软件环境一致

需保证性能评测时，各个框架使用的驱动、支撑库等版本相同。典型的软件环境主要包括：Python、CUDA、cuDNN、NCCL等。

### 3.网络结构一致

即各个框架采用相同的，通用模型进行测试，如图像分类领域的ResNet50、NLP领域的BERT-Base、BERT-Large等，模型测试时需保证主要参数一致。

### 4.数据集一致

需采用相同的数据集，如图像分类领域采用ImageNet，NLP采用wiki数据集；保证数据集内容一致的情况下，可以由各个框架以自身通用的方式进行制作和读取。

### 5.流程可复现

即测试过程保留代码、脚本、日志、数据集制作格式等具体信息，方便外部对流程进行复现。

### 6.中值原则

多次测试（5~7次）后，取中值的方式进行结果判定。

### 7.最优化配置原则

保证公平的情况下，各个框架可以使用各自最优化的代码设定、环境变量、超参数配置等。