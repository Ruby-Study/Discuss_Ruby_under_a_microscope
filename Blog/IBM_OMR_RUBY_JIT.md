# Ruby+OMR JIT简介

首发于[知乎专栏·时光与精神小屋](https://zhuanlan.zhihu.com/time-and-spirit-hut)： [Ruby+OMR JIT简介](https://zhuanlan.zhihu.com/p/24113974?refer=time-and-spirit-hut)

此文为翻译，[原文](https://developer.ibm.com/open/2016/11/18/introducing-ruby-jit/)在此

---

前不久，我开源了[ Ruby+OMR JIT 编译器](https://github.com/rubyomr-preview/rubyomr-preview)这篇文章中提到的[ Ruby JIT glue ](https://github.com/rubyomr-preview/rbjitglue)，为此我深感荣幸。

该代码（[适度修改过的CRuby VM版本](https://github.com/rubyomr-preview/ruby)和[Eclipse OMR](https://github.com/eclipse/omr/)的组合）展示了我们如何向CRuby添加一个简单的JIT编译器。

本文会涉及我在2015年12月RubyKaigi  ( [Video](https://www.youtube.com/watch?v=EDxoaEdR-_M) ,  [Slides](http://www.slideshare.net/MatthewGaudet/experiments-in-sharing-java-vm-technology-with-cruby) )的[演讲](http://rubykaigi.org/2015/presentations/MattStudies?cm_mc_uid=90162356470714696264613&cm_mc_sid_50200000=1480639232)内容，但是这次我将展示源码。

该文专注于JIT技术，然而我们在Github上的VM已经被修改为使用OMR的垃圾收集技术。 有关更多的内容，请参阅Craig Lehman和Robert Young在RubyKaigi上所做的演讲，[It's dangerous to GC alone. Take this!](http://rubykaigi.org/2015/presentations/youngrw_CraigLehmann?cm_mc_uid=90162356470714696264613&cm_mc_sid_50200000=1480639232)。

### 哲学

OMR项目基于一个前提：即许多语言运行时实现基本上是非常相似的，哪怕是完全无法类比的语言。

该理念决定了OMR被构造为语言独立的大核心，通过依赖于语言的“胶水（glue）”，链接到实际的语言实现中。

对于Ruby JIT来说，我们的“胶水”分为两个部分：

1. 修改Ruby VM，以便于加载JIT编译对象，并导出某些例程（译注: 让VM可以对外提供服务的接口），以便于JIT可以生成对VM的调用和使用VM的功能
2.  适配器代码———— 即GitHub上rbjitglue项目的内容——是专门用于Ruby的语言独立的OMR编译器代码（称为Testarossa）。

我们尽最大的努力尽量最小化第一部分，因为我们想展示一个相对低触摸（low-touch）的JIT添加。

我们一直希望Ruby社区最终可以采用OMR JIT 作为Ruby的JIT 技术，所以尽量使VM保持不变很重要，没有人喜欢维护这些像异形一样的补丁。这个决定限制了编译器技术的某些部分，但是我们可以跳过它（有些东西我们需要独立开发！）

### 修改 CRuby VM

在运行JIT代码之前，我们需要在VM和JIT编译器之间提供一个接口。

在我们的实现中，我们在Init_BareVM中增加了vm_jit_init调用。该函数（[实现于vm_jit.c中](https://github.com/rubyomr-preview/ruby/blob/ad3b419cf7cb56e60195d81a7f411359aeb0484d/vm_jit.c#L35)）加载JIT DLL并且填充一组函数指针的结构。通过这些函数指针来建立JIT和VM的通信，VM可以使用这些函数指针（[init_f, terminate_f, compile_f 和 dispatch_f](https://github.com/rubyomr-preview/ruby/blob/ad3b419cf7cb56e60195d81a7f411359aeb0484d/vm_jit.c#L73)）来调用JIT方法。同样，JIT也能调用VM方法，或者通过使用[存储于JIT回调结构中的函数指针](https://github.com/rubyomr-preview/ruby/blob/ad3b419cf7cb56e60195d81a7f411359aeb0484d/vm_jit.c#L97)来生成对VM方法的调用。稍后我们会再看这个回调结构。

另一个重要的连接是Ruby VM使用的全局变量。我们需要将这些地址传递给Ruby JIT，这份工作是由[全局结构体](https://github.com/rubyomr-preview/ruby/blob/ad3b419cf7cb56e60195d81a7f411359aeb0484d/jit.h#L100)来完成的，该结构体被填充以后就传递给vm_jit_init。

#### 编译控制

何时编译方法，即为编译控制。

生产JIT编译器有非常复杂的试探式方法来管理启动速度，提升和峰值吞吐量之间的权衡。

在Ruby JIT中，我们做了一个更为简单的选择：计算编译（counted compilation）。每个方法都有与之相关的计数器。方法运行时，计数器递减，到达0值时，该方法被编译完成。

编译控制的注入与VM调用JIT编译方法方式紧密的结合了起来。

#### JIT 编译方法调度

OMR Ruby JIT 对JIT方法采用了一个非常简单的调用约定。JIT生成的方法[只需要一个参数](https://github.com/rubyomr-preview/ruby/blob/ad3b419cf7cb56e60195d81a7f411359aeb0484d/jit.h#L117)：

```
typedef VALUE (*jit_method_t)(rb_thread_t*);
```

方法的所有其他信息都从编译代码的线程参数中去检索。

Ruby VM的核心解释器循环有一些轻微修改，以便支持调用这些编译体。

一般来说，解释器通过调用[vm_exec_core](https://github.com/rubyomr-preview/ruby/blob/ad3b419cf7cb56e60195d81a7f411359aeb0484d/vm_exec.c#L51)来启动核心解释器循环调用。

为了让我们检查是否可以使用JIT编译体，我们把该函数[聪（中）明（二）](http://martinfowler.com/bliki/TwoHardThings.html)的命名替换为了[vm_exec2](https://github.com/rubyomr-preview/ruby/blob/ad3b419cf7cb56e60195d81a7f411359aeb0484d/vm.c#L1486)。

vm_exec2[非常简单](https://github.com/rubyomr-preview/ruby/blob/ad3b419cf7cb56e60195d81a7f411359aeb0484d/vm.c#L1442):

```
static inline VALUE
vm_exec2(rb_thread_t *th, VALUE initial)
{
    VALUE result;
    if (VM_FRAME_TYPE(th->cfp) != VM_FRAME_MAGIC_RESCUE &&
        VM_FRAME_TYPE_FINISH_P(th->cfp) &&
        vm_jitted_p(th, th->cfp->iseq) == Qtrue) {
	  result = vm_exec_jitted(th);
    } else {
      result = vm_exec_core(th, initial);
    }
    return result;
}
```


该代码简单归结为两点：

1. 只要帧是正确的类型（代码里给出的JIT方法勿须理解），并且有一个JIT体，就可调用它获取结果。
2. 否则回到vm_exec_core继续解释该帧。

### 使编译器适配Ruby VM

要想把OMR JIT编译器添加到语言中，最基本的部分是如何将该语言的源码转换为JIT编译器的中间表示（IR, Intermediate Representation）。

Testarossa的中间表示叫做树（虽然它们实际上叫Directed-Acyclic-Graphs【有向无环图】，可惜历史的能量太强）。该树中的程序展示包含了三个主要元素：

1. 节点表示一个值。节点是可以产生值的原始操作（比如 lconst 10 或 lload valueInMemory），或者是复合操作，比如add操作，可以消费子节点来产生结果。节点有数据类型，操作将会需要并产生类型。例如，lconst产生一个‘long’值，对Testarossa来说，是一个64位整数。aladd会从地址的子节点和long类型产生一个’address’类型值。
2. TreeTops用于树的计算。它们是双向链表，并用于控制程序的顺序，节点树内的操作顺序无关紧要，只作用于TreeTops之间。在树的节点之间评估顺序很重要，我们可以通过“锚定”树节点下的节点来强制排序。
3. TreeTops在BasicBlocks中互相连接，定义了程序的控制流。

所以，为了JIT编译Ruby，我们不得不把下面的代码:

```
def foo(a, b)
  a + b
end
```

转换到树中。在Testarossa中，我们把这个过程称作IL生成。

#### Ruby中的IL生成

幸运的是，我们不必处理Ruby代码的解析。CRuby解释器已经为我们做了这部分工作。自Ruby1.9开始，Ruby就一直使用基于字节码的解释器，因此Ruby代码是被解析为字节码再进行解释。你可以使用RubyVM :: InstructionSequence.dump方法查看字节码：

```
$cat test.rb
def foo(a, b)
    a + b
end
$ ruby -e "puts RubyVM::InstructionSequence.compile_file('test.rb').disasm"
# <snipped iseq for main>  
<RubyVM::InstructionSequence:foo@t.rb>=======================
local table (size: 3, argc: 2 [opts: 0, rest: -1, post: 0, block: -1] s1)
[ 3] a<Arg>     [ 2] b<Arg>     
0000 trace            8                                               (   1)
0002 trace            1                                               (   2)
0004 getlocal_OP__WC__0 3
0006 getlocal_OP__WC__0 2
0008 opt_plus         <callinfo!mid:+, argc:1, ARGS_SKIP>
0010 trace            16                                              (   3)
0012 leave                                                            (   2)
```

所以，要生成Ruby的树，我们可以解析方法产生的字节码，而不是直接解析源码。在rbjitglue中，这部分工作由[RubyIlGenerator.cpp](https://github.com/rubyomr-preview/rbjitglue/blob/master/ruby/ilgen/RubyIlGenerator.cpp)文件来完成。

Ruby所遵循的Testarossa的基本字节码IL生成方案假定每个字节码对应至多一个基本块（block）。因此，基本的块数组会被创建，然后沿着字节码经历“抽象解释过程”，生成树，再放到基本块中。抽象解释的意思是，把IL生成器生成代码的过程比作解释器。

当检测到字节码级别的跳转（jump）时，相应的目的地字节码的树级跳转的基本块也会被生成。

大部分工作最终是在[RubyIlGenerator :: indexedWalker](https://github.com/rubyomr-preview/rbjitglue/blob/eb802a5/ruby/ilgen/RubyIlGenerator.cpp#L732)内部完成的。 其核心，是一个巨大的[switch语句](https://github.com/rubyomr-preview/rbjitglue/blob/eb802a5/ruby/ilgen/RubyIlGenerator.cpp#L762)：

```
switch (insn)
   {
   case BIN(nop):                  /*nothing */ _bcIndex += len; break;
   // elided
   case BIN(getlocal):             push(getlocal(getOperand(1)/*idx*/, getOperand(2)/*level*/)); _bcIndex += len; break;
   // elided
   case BIN(leave):                _bcIndex = genReturn(pop()); break;
```

对于每个YARV字节码，我们将在抽象操作中再现执行时发生的操作。例如，YARV的操作码getlocal把局部区域的值压到堆栈上，因此，这个抽象版生成的代码将从局部区域加载一个值，然后将其压到抽象的堆栈上。其他操作数将消耗此抽象堆栈上的值，比如该例中的leave。

最后，我们得到了树！让我们看看上面foo示例中getlocal 和 opt_plus 操作得到的树。

```
n23n      treetop                                                                    
n22n        lloadi  a[#612  a -24]                             
n21n          aloadi  ep[#599  ep +48]                        
n20n            aloadi  cfp[#600  cfp +48]                       
n19n              aload  <parm 0 Lrb_thread_t;>[#598  Parm]
n28n      treetop                                                                    
n27n        lloadi  b[#613  b -16]                               
n26n          aloadi  ep[#599  ep +48]                        
n25n            aloadi  cfp[#600  cfp +48]                 
n24n              aload  <parm 0 Lrb_thread_t;>[#598  Parm]  
n35n      astorei  pc[#603  pc]                                
n34n        aloadi  cfp[#600  cfp +48]                          
n33n          aload  <parm 0 Lrb_thread_t;>[#598  Parm]       
n32n        aconst 0x5606fbdad740                                                    
n36n      treetop                                                                    
n31n        lcall  vm_opt_plus[#290  helper Method]           
n30n          aload  <parm 0 Lrb_thread_t;>[#598  Parm]       
n29n          aconst 0x5606fbdad7a0                                                  
n22n          ==>lloadi
n27n          ==>lloadi
```

为了讲清楚，我将去除一些标志和节点元数据来简化。你可以通过使用traceFull,log=LOGFILENAME生成log来查看完整输出。

纵使被简化内容也不少！

1. 左边的那一列是节点号，用于标识节点，使得我们可以明确的讨论特定的节点。
2.  n23n 是treetop节点。它用于挂接子节点并保证顺序。这里指的是相对顺序，节点23下面的节点一定是在节点28下面的节点前被执行的。
3. 以n22n为根的树，是加载传递给foo的参数。该树的形状由Ruby代码决定。顺序基本上是C中的 thread->cfp->ep[indexOf(a)]
4. n35n为根的树是存储YARV程序计数器。该树的存在是为了确保在Ruby JIT中解释器可以随时接管，只要JIT代码可能失控就恢复所有的解释器状态（比如，在回调期间发生了异常）。
5. n31n为根的树对应于一个YARV 指令opt_plus的helper调用。这是一个生成的回调。

### 回调生成

前面说过，我会讨论[jit_callback_struct](https://github.com/rubyomr-preview/ruby/blob/ad3b419cf7cb56e60195d81a7f411359aeb0484d/jit.h#L29)更多的细节。它的基本形式，看起来就像这样：


```
struct jit_callbacks_struct {
   /* Include generated callbacks. */
   #include "vm_jit_callbacks.inc"
   /* ... */
   const char *         (*rb_id2name_f)                 (ID id);
  // ...
}
```

之所以用#include是因为，我们的回调的大部分实际上是在构建期间被生成的，而非手工创建。我们能这样做是得益于YARV的聪明设计。

YARV使用名为[insns.def](https://github.com/rubyomr-preview/ruby/blob/ad3b419cf7cb56e60195d81a7f411359aeb0484d/insns.def)的“指令定义文件”，用于生成核心解释器循环。但是，我们也可以利用该定义文件自动创建回调。在我们修改的RubyVM构建过程中，我们使用相同的处理代码创建解释器循环，并自动发出回调和函数指针声明。

所以，一个指令定义看起来像[这篇博客](https://github.com/rubyomr-preview/ruby/blob/ad3b419cf7cb56e60195d81a7f411359aeb0484d/insns.def#L380)定义的这样：
```
/**
  @c put
  @e to_str
  @j to_str の結果をスタックにプッシュする。
 */
DEFINE_INSN
tostring
() 				/* Instruction operands -- these are in the instruction seqeuence */
(VALUE val)     /* Values popped from the operand stack                           */
(VALUE val)     /* Return value                                                   */
{
    val = rb_obj_as_string(val);
}
```

我们生成的回调（至于vm_jit.inc）看起来像这样：

```
VALUE
vm_tostring_jit(VALUE val)
{
    val = rb_obj_as_string(val);
    return val;
}
```

生成回调允许我们支持大量的Ruby操作码，而不必手工编写大量的ILGen代码。但是有一个权衡：就像Evan Phoenix在他2015年主题演讲RubyKaigi中提到的，这种风格的JIT有优化限制，然而，它非常符合我们的Ruby JIT哲学。

### 代码生成

本文不会涉及太多代码生成。因为它并非是Ruby特定的，而且也会牵扯其他方面。但是编译的最终产品是一个startPC，它对应于为方法生成代码的开始。它将会在vm_exec_jitted被调用，如上所述。

### 总结

啧啧啧——你竟然看完了！非常感谢阅读！本文是对Ruby+OMR JIT 不太简短的总结。现在，它只适用于Ruby2.1.5，然而，我们将逐步支持到Ruby2.4。

这里是一份仍然需要为Ruby JIT构建的任务列表：

1. 运行时假设：能够推测性的假设和虚拟机状态有关的某些事情，并可以在虚拟机状态改变的时候更新代码。比如假设+操作没有被重新定义，但是当它被重新定义的时候能够丢弃编译的方法，或者为编译代码打补丁。
2. 异步编译：目前，当方法被编译的时候，它会阻止线程执行Ruby代码，直到编译完成。这其实是可以避免的，但是为了简单起见，我们选择了不实现异步编译。然而，如果我们实现了异步编译，就可以增加多个编译线程。
3. 更高级的优化：目前，Ruby JIT 使用的[优化列表比较少](https://github.com/rubyomr-preview/rbjitglue/blob/eb802a5/ruby/optimizer/Optimizer.cpp#L59)。OMR支持更多优化，我们需要启用它们。
4. 重新编译：我们有初步的重新编译支持。然而我们用了相同的编译策略，我们需要启用更多的优化。

我很想听听你对我们迄今为止进展的看法。请在本（原）文下面发表评论，通过[邮件列表](https://dev.eclipse.org/mailman/listinfo/omr-dev)发送一条见解，或者在GitHub上提交一个[issue](https://github.com/rubyomr-preview/ruby/issues)。
