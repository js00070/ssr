# 代码阅读记录
## 代码仓库
fork了一份到https://github.com/js00070/ssr，目前在dev分支上做修改

## 安装依赖
在我的deepin上, 由于已经装了很多库, 所以无法确定完整的依赖是什么, 只能确定, 依赖中必然包括以下的库
```bash
sudo apt install libsndfile1-dev
sudo apt install libjack-jackd2-dev
sudo apt install libfftw3-dev
sudo apt install portaudio19-dev
sudo apt install jackd
```
为了编译doc文档,需要安装如下依赖
```bash
sudo apt install doxygen
sudo apt install sphinx-common
pip install sphinx_rtd_theme
```

## 编译
### 编译文档
```bash
cd ~/ssr/apf/doc
doxygen

cd ~/ssr/doc/manual
make html
```

### 编译官方示例
为了在linux上编译通过，对示例代码和makefile稍微做了些修改，但jack_query_thread.cpp代码可能有问题，始终无法编译通过，待解决
```bash
cd ~/ssr/apf/examples
make
```

## 代码阅读

### apf/apf/misc.h

#### class CRTP<Derived>
奇异递归模板模式(Curiously Recurring Template Pattern), 参考https://zhuanlan.zhihu.com/p/54945314

CRTP的好处是不用调用虚函数也能实现多态, 省去了虚函数开销

成员函数Derived& derived(), 顾名思义, 相当于是返回子类的this

官方页面: https://audioprocessingframework.github.io/classapf_1_1CRTP.html

#### class NonCopyable
禁用了拷贝构造函数和拷贝=, 可以移动构造和移动=

官方页面: https://audioprocessingframework.github.io/classapf_1_1NonCopyable.html

#### class BlockParameter<T>
内部保存了某个类型T变量的当前值(_current)和备份值(_old)

_current是调用移动构造函数得到的,而_old是调用拷贝构造函数得到的

get()方法和()操作符可以得到_current的引用

old()方法可以的领导_old的引用

官方页面: https://audioprocessingframework.github.io/classapf_1_1BlockParameter.html

### apf/apf/commandqueue.h
#### class CommandQueue
从非实时线程往实时线程发送命令信息,具体见论文4.2节

内部有两个LockFreeFifo<Command*>, 分别是_in_fifo和_out_fifo, _in_fifo里存着实时线程中要执行的Command, _out_fifo里存着已经执行过的Command的引用, 等待着在非实时线程中释放这些Command的内存

成员函数process_commands的流程就是, 从_in_fifo里取出Command, 执行Command的execute虚函数, 然后把这个Command指针加入到_out_fifo中

成员函数cleanup_commands就是把_out_fifo中的Command都释放

成员函数deactivate和reactivate就是在操作内部的_active: bool 变量, 前者是cleanup之后令_active为false, 后者是cleanup之后令_active为true, 具体这么做是为啥, 意义不明

官方页面: https://audioprocessingframework.github.io/classapf_1_1CommandQueue.html

### apf/apf/lockfreefifo.h
#### class LockFreeFifo<T*>
线程安全的队列(单读者单写者),具体见论文4.1节

官方页面: https://audioprocessingframework.github.io/classapf_1_1LockFreeFifo_3_01T_01_5_01_4.html

### apf/apf/rtlist.h
#### class RtList<T*>

一个RtList<T*>实例中包含一个用来实际存放元素的_the_actual_list(使用的是std::list),以及一个用来缓存对_the_actual_list进行操作的各种指令的CommandQueue _fifo, 常见的对list进行的操作都在内部实现了相应的CommandQueue::Command的子类(AddCommand, RemCommand, ClearCommand). RtList 没有锁, 若要多线程同时对RtList进行操作, 需要手动加mutex

官方页面: https://audioprocessingframework.github.io/classapf_1_1RtList_3_01T_01_5_01_4.html

### class MimoProcessor
MimoProcessor<Derived, interface_policy, thread_policy, query_policy>

关于policy-based designed, 参考https://www.cnblogs.com/mthoutai/p/6871667.html

由于这个类过于复杂, 先看MimoProcessor内部定义的类

ProcessItem: 参考dummy_example.cpp的第13行开始的使用例子. 继承了ProcessItem<X>的类X的实例是放入RtList的元素

宏APF_PROCESS(A, B)相当于是定义一个process类(继承了B::Process), 构造函数相当于是执行A的成员函数APF_PROCESS_internal, 函数内容由用户编写

ScopedLock: 接收thread_policy里定义的Lock, ScopedLock实例的生命周期开始时锁住, 结束时释放锁

CleanupFunction:　接收CommandQueue fifo, 括号运算符执行cleanup_commands

QueryThread: 继承了thread_policy里的ScopedThread<CleanupFunction>, 这个ScopedThread接收的是CommandQueue&和thread_policy里定义的usleeptime. QueryThread是在MimoProcessor的成员函数make_query_thread中初始化的, 初始化了一个unique_ptr<QueryThread>

QueryCommand: 继承了CommandQueue::Command, 疑问: Derived& _parent是干啥的? cleanup函数里的F.update和_parent.new_query是干啥的? QueryCommand的功能是啥?

成员函数activate与deactivate的作用, 迷

成员函数add, 添加input/output通道

MimoProcessor的构造函数

MimoProcessor的成员函数_process_list, 参数是一个RtList, 顾名思义, 就是对RtList进行处理, 将RtList地址赋值给MimoProcessor的成员变量_current_list之后, 内部调用了_process_current_list_in_main_thread()这个函数, 

在_process_current_list_in_main_thread函数中, 先唤醒了_thread_data里的所有线程, 然后调用_process_selected_items_in_current_list(0), 0是主线程的线程号, 之后等待线程完成,

在_process_selected_items_in_current_list函数中, 会执行之前的那个RtList里每一个item的process()方法

另外, _process_selected_items_in_current_list函数还在WorkThreadFunction类的()操作符中使用, 实际上就是在对应的WorkerThread中执行process

MimoProcessor::Xput 是Input和OutPut的共同基类, 为了代码复用

### pointer_policy.h

MimoProcessor的第二个模版参数interface_policy就是pointer_policy类型的, MimoProcessor里的Input/Output不仅继承了Xput, 还继承了interface_policy里的Input/Output类, MimoProcessor::Input/Output的process()里执行了interface_policy里的fetch_buffer()函数

另外很重要的是, pointer_policy::Input/Output中包括了buffer_type buffer成员, simpleprocessor.h中的Input::APF_PROCESS里用的buffer就是这里的buffer

### combine_channels.h
似乎是和通道间结合处理相关的代码

### simpleprocessor.h

31行, 注释里说input buffer和output buffer的地址是复用的, 避免了多余的拷贝

构造函数, 接收一个parameter_map, 确定输入通道数和输出通道数, 调用add成员函数添加通道(通道编号)

SimpleProcessor::Input::APF_PROCESS 实际上是把interface_policy::Input里的buffer拷贝到自己的_buffer里



### dummy_example.cpp

APF_PROCESS展开后的写法

```cpp
// APF_PROCESS(MyIntermediateThing, ProcessItem<MyIntermediateThing>)
        // 展开的写法:
        struct Process : ProcessItem<MyIntermediateThing>::Process {
          explicit Process(MyIntermediateThing& ctor_arg) : ProcessItem<MyIntermediateThing>::Process(ctor_arg) {
            ctor_arg.APF_PROCESS_internal();
          } 
        };
        void APF_PROCESS_internal()
        {
          // do your processing here!
        }
```
