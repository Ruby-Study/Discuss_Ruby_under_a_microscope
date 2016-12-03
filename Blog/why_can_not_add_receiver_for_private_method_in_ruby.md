首发于[知乎专栏·哲学与计算](https://zhuanlan.zhihu.com/p/23684649/edit)

### 引子：

今天一位群友问了一个问题：

```
class C
  def public_method
    self.private_method
  end

  private
  def private_method; end
end


C.new.private_method #=> NoMethodError: private method 'private_method' called (...)
```
为什么都是在同一个类里的两个方法，又不是类外部，怎么还不能用self调用了呢？

其实他看的书里已经给出了答案： 「因为私有方法只能用隐式的接收者」。

这实际上是Ruby语言的天生的固有的规则，就好像是「地球是圆的」一样。

但实际上他无法理解这种规定，那么强行接受这种规则可好？有人说了，这就是Ruby语言创造者的喜好嘛，他喜欢这样设计，你接受就好了嘛，问什么问。我并不赞同这样的观点。任何事物都不是凭空产生的，这样的设计背后一定是有原因的。

### 从设计哲学入手

这个世界分成神造和人造。假如你想了解宇宙的结构，为什么地球是圆的，为什么地球要绕太阳旋转，你只需要找造物主问一问，“你是怎么想的？宇宙为什么要设计成这样？”，那么困扰人类的谜题就很容易解答了。但实际上，你找不到这样一位造物主，更别说去问这个问题了。但是，作为计算机语言，作为一个人造的东西，我们只需要找语言作者去问问就能知道他如何设计这门语言，幸好，松本行弘很勤快，写了一本《松本行弘的程序世界》一书让我们来观摩他的思想。

书里开篇就写了他设计Ruby的过程，他首先是把Ruby定位于一门面向对象语言，他确定了三个设计哲学，第一设计哲学是简洁，他把Ruby的面向对象实现变的简洁无比，简洁之中又保持的高度一致性。这里就不细说了，细说可以写一本书，事实上这样的书很多了，Avdi和Sandi就写了好几本，还有Martin Fowler。

面向对象的特性之一是封装（说到这里，今天有人说，封装只是Java和C++的特性，我无言以对）。封装，是为了隐藏信息。为什么隐藏信息？看看我从维基百科复制的例子：

```
/* 一个面向过程的程序会这样写： */
定义莱丝
莱丝.设置音调(5)
莱丝.吸气()
莱丝.吐气()

/* 而当女人高歌被封装到类中，任何人都可以简单地使用： */
定义莱丝是女人
莱丝.高歌()
```
看看，面向对象思想充满了人文气息，这很难理解吗？「高歌()」是被暴露出来的，和其他对象通信的，「高歌()」内部，你可以把吸气、吐气、设置音调等方法隐藏起来，其他对象根本不关心这些。所以，我们只需要听女人唱歌就好了，关注她什么时候吸气、什么时候吐气干嘛呢？

这就叫封装。是一种代码组织方式，它实际上是对问题领域进行了良好的界限划分，从而实现了松耦合。否则的话，拿上面的例子来说，女人要「高歌()」，还得让你帮她「吸气()」，是不是有点扯淡？「吸气」「吐气」「调音」都是她自己的事。

### 如何实现封装？private/public/protected

很多面向对象语言都使用private/public/protected来实现封装。还是刚才那人说「那xxx语言也没有private呀，所以你就当private不存在就好了」，这不是扯淡吗？现在如日中天的Golang语言（它也有面向对象特性），有private关键字吗？那么Golang就不能封装了吗？

话说回来，Ruby中的private其实不是关键字，是个方法。使用方式有两种：

```
class Girl
  attr_accessor :name
  def initialize(name)
    @name = name
  end

  def sing
    inspiration
    expiration
    train_voice
  end

  private
  def inspiration; end
  def expiration; end
  def train_voice; end
end

girl = Girl.new('你女朋友的名字') #没有女朋友的随便写吧
girl.sing
```

上面是称为scope的方式，也就是private方法，把class定义的scope从中隔开了。另外的方式是这样的：
```
class Girl
  attr_accessor :name
  def initialize(name)
    @name = name
  end

  def sing
    inspiration
    expiration
    train_voice
  end

  def inspiration; end
  def expiration; end
  def train_voice; end

  private :inspiration, :expiration, :train_voice
end

girl = Girl.new('你女朋友的名字') #没有女朋友的随便写吧
girl.sing
```

这是通过方法参数的形式来定义private方法。不管是scope还是方法参数，效果是一样的。

但是调用的时候，你必须使用隐式的接收者，而不能显式的指定，包括指定self，不服气你自己试试。

### 问题来了：为什么只能隐式调用？

在从底层的Ruby实现解释其原因之前，我们不妨从另一个角度问一下自己：

    请问，你自言自语的时候会喊你自己的名字吗？（当然，不排除有喊自己名字的，大千世界，什么奇葩都有）
    你在问自己这个问题之前，有没有喊你自己的名字？

### 从底层源码实现角度来观察为什么private方法只能是隐式接收者

好了，铺垫完了，现在我们从底层Ruby源码实现的角度来解释一下：

我们回顾一下Ruby的对象模型：object.message： 「.」前面的object称为「消息接收者」，「.」后面的message称为「消息」。比如：

```
贾那啥.听("你妈叫你吃饭了")
```

在Ruby中这些message都被称为「方法」，但是在Ruby底层，是有区分的：

    方法：显式的指定了「消息接收者」的函数调用。
    函数: 没有显式指定「消息接收者」的函数调用。

有人问了，看把你牛逼的，你咋知道的？呵呵，我翻译的《Ruby Under a Microscope》马上要上市了，中文名是《Ruby源码剖析》，大家记得买，看完你也知道了。

我们也来看看代码：

```
code =<<END
  puts 1
END

puts RubyVM::InstructionSequence.compile(code).disasm
```

输出

```
== disasm: #<ISeq:<compiled>@<compiled>>================================
0000 trace            1                                               (   1)
0002 putself          
0003 putobject_OP_INT2FIX_O_1_C_
0004 opt_send_without_block <callinfo!mid:puts, argc:1, FCALL|ARGS_SIMPLE>, <callcache>
0007 leave  
```

从YARV指令里看到puts调用没有给定显式的接收者，那么它的函数调用标识是FCALL。

看另外一段代码：

```

code =<<END
  a = A.new
  a.a
END
puts RubyVM::InstructionSequence.compile(code).disasm
```
输出

```
== disasm: #<ISeq:<compiled>@<compiled>>================================
local table (size: 2, argc: 0 [opts: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])
[ 2] a          
0000 trace            1                                               (   1)
0002 getinlinecache   9, <is:0>
0005 getconstant      :A
0007 setinlinecache   <is:0>
0009 opt_send_without_block <callinfo!mid:new, argc:0, ARGS_SIMPLE>, <callcache>
0012 setlocal_OP__WC__0 2
0014 trace            1                                               (   2)
0016 getlocal_OP__WC__0 2
0018 opt_send_without_block <callinfo!mid:a, argc:0, ARGS_SIMPLE>, <callcache>
0021 leave  
```

看到了吗，如果给定了显式的接收者，方法调用的时候没有FCALL标识。

好，接下来，我们知道private是一个方法，而不是关键字，那可以看看private的方法定义：

```
static VALUE
rb_mod_private(int argc, VALUE *argv, VALUE module)
{
    return set_visibility(argc, argv, module, METHOD_VISI_PRIVATE);
}
```

全局搜索Ruby源码，发现了set_visibility的定义：

```
static VALUE
set_visibility(int argc, const VALUE *argv, VALUE module, rb_method_visibility_t visi)
{
    if (argc == 0) {
       rb_scope_visibility_set(visi);
    }
    else {
       set_method_visibility(module, argc, argv, visi);
    }
    return module;
}
```

看得出来，如果private没有给定参数，就设置作用域的可见性，否则就只是对给定参数操作。没有给定参数的情况下，那就是private方法下面所覆盖的作用域。

那么rb_mod_private函数中这个 METHOD_VISI_PRIVATE常量是什么呢？不急，我们慢慢找。先来看看来看看rb_scope_visibility_set的实现：

```
void
rb_scope_visibility_set(rb_method_visibility_t visi)
{
    vm_cref_set_visibility(visi, FALSE);
}
```

cref是什么？ cref是跟踪词法作用域和类的继承关系的。

又有人问了，看把你牛逼的，你咋知道的？呵呵，我翻译的《Ruby Under a Microscope》马上要上市了，中文名是《Ruby源码剖析》，大家记得买，看完你也知道了。

来看看vm_cref_set_visibility的代码：

```
static void
vm_cref_set_visibility(rb_method_visibility_t method_visi, int module_func)
{
    rb_scope_visibility_t *scope_visi = (rb_scope_visibility_t *)&rb_vm_cref()->scope_visi;
    scope_visi->method_visi = method_visi;
    scope_visi->module_func = module_func;
}
```

看到这里，咱们得停下来消化一下： Visibility，这个单词有文化的人都知道，是「可见性」的意思。这里不得不说，Ruby的源码写的也是蛮不错的，你看这命名，包含了十足的语义启示，所以我看明白了，这是设置作用域的可见性。

好了，接着往下走吧。那么rb_scope_visibility_t是个什么呢？它是一个结构体：

```

typedef struct rb_scope_visi_struct {
     rb_method_visibility_t method_visi : 3;
     unsigned int module_func : 1;
} rb_scope_visibility_t;
```
这里有奇怪的数字，我们再顺藤摸瓜，找找 rb_method_visibility_t的定义，结果发现：

```
/* cref */

typedef enum {
    METHOD_VISI_UNDEF     = 0x00,
    METHOD_VISI_PUBLIC    = 0x01,
    METHOD_VISI_PRIVATE   = 0x02,
    METHOD_VISI_PROTECTED = 0x03,

    METHOD_VISI_MASK = 0x03
} rb_method_visibility_t;
```
原来Ruby内部是用16进制数字来表示作用域的可见性，

    undef的方法可见性是0
    public方法可见性是1
    private方法可见性是2
    protected可见性是3.

那么我就明白了，这个rb_scope_visi_struct结构体默认是3. 是protected。我们总算知道前面的METHOD_VISI_PRIVATE是啥意思了。


METHOD_VISI_MASK看字母意思，可能是用来做位掩码，在Ruby源码里全局搜索以后，发现确实如此：
```
rb_print_inaccessible(VALUE klass, ID id, rb_method_visibility_t visi)
{
    const int is_mod = RB_TYPE_P(klass, T_MODULE);
    VALUE mesg;
    switch (visi & METHOD_VISI_MASK) {
      case METHOD_VISI_UNDEF:
      case METHOD_VISI_PUBLIC:    mesg = inaccessible_mesg(""); break;
      case METHOD_VISI_PRIVATE:   mesg = inaccessible_mesg(" private"); break;
      case METHOD_VISI_PROTECTED: mesg = inaccessible_mesg(" protected"); break;
      default: UNREACHABLE;
    }
    rb_name_err_raise_str(mesg, klass, ID2SYM(id));
}
```

继续往下看，rb_vm_cref()是啥呢？来看看
```
rb_cref_t *
rb_vm_cref(void)
{
    rb_thread_t *th = GET_THREAD();
    rb_control_frame_t *cfp = rb_vm_get_ruby_level_next_cfp(th, th->cfp);

    if (cfp == NULL) {
     return NULL;
    }

    return rb_vm_get_cref(cfp->ep);
}
```

这里返回的是rb_vm_get_cref(cfp->ep);的求值：

```
static rb_cref_t *
rb_vm_get_cref(const VALUE *ep)
{
    rb_cref_t *cref = vm_env_cref(ep);

    if (cref != NULL) {
     return cref;
    }
    else {
     rb_bug("rb_vm_get_cref: unreachable");
    }
}
```
ep是环境指针，ep是记录上下文帮助实现闭包的。

有人问了，看把你牛逼的，你咋知道的？呵呵，我翻译的《Ruby Under a Microscope》马上要上市了，中文名是《Ruby源码剖析》，大家记得买，看完你也知道了。

这里我们分析private就不用管这个了，所以知道它只是返回已记录的cref。

而rb_cref_t 是一个结构体：

```
typedef struct rb_cref_struct {
    VALUE flags;
    const VALUE refinements;
    const VALUE klass;
    struct rb_cref_struct * const next;
    const rb_scope_visibility_t scope_visi;
} rb_cref_t;
```

该结构体跟踪了代码所属的词法作用域（next指针），当然还有其他信息，比如类的继承关系（klass指针） 。

有人问了，看把你牛逼的，你咋知道的？呵呵，我翻译的《Ruby Under a Microscope》马上要上市了，中文名是《Ruby源码剖析》，大家记得买，看完你也知道了。

然而书里没说的是，这个结构体其实也记录了词法作用域的可见性，这里定义了一个常量:scope_visi

然后我们回到 rb_mod_private函数中：

```
return set_visibility(argc, argv, module, METHOD_VISI_PRIVATE);
```

所以，上面vm_cref_set_visibility函数中rb_vm_cref()->scope_visi;的意思是：开一个新的栈帧返回一个rb_cref_t结构体，那么这个时候scope_visi 的值是3，那么其实到了这里，这样一层一层传下来，scope_visi的值就是2了。

再看vm_cref_set_visibility函数：
```
rb_scope_visibility_t *scope_visi = (rb_scope_visibility_t *)&rb_vm_cref()->scope_visi;
scope_visi->method_visi = method_visi;
scope_visi->module_func = module_func;
```

vm_cref_set_visibility函数里重新定义了一个scope_visi结构体，新增了两个属性method_visi和module_func，这俩属性是当private方法里传入参数的时候才有用，比如：

```
class A
  def a
    puts 1
  end
  private :a
end
```

就这样，private方法定义了该下方作用域的可见性，当Ruby在函数调用的时候：

```
vm_call_method(rb_thread_t *th, rb_control_frame_t *cfp, struct rb_calling_info *calling, const struct rb_call_info *ci, struct rb_call_cache *cc)
{
    VM_ASSERT(callable_method_entry_p(cc->me));

    if (cc->me != NULL) {
     switch (METHOD_ENTRY_VISI(cc->me)) {
       case METHOD_VISI_PUBLIC: /* likely */
         return vm_call_method_each_type(th, cfp, calling, ci, cc);

       case METHOD_VISI_PRIVATE:
         if (!(ci->flag & VM_CALL_FCALL)) {
          enum method_missing_reason stat = MISSING_PRIVATE;
          if (ci->flag & VM_CALL_VCALL) stat |= MISSING_VCALL;

          cc->aux.method_missing_reason = stat;
          CI_SET_FASTPATH(cc, vm_call_method_missing, 1);
          return vm_call_method_missing(th, cfp, calling, ci, cc);
         }
         return vm_call_method_each_type(th, cfp, calling, ci, cc);
...
}
```

想看完整函数定义的可以点击该[代码源码位置](https://github.com/ruby/ruby/blob/9cbd6ee09770be3d73a17ab1195a094c59c9f9ee/vm_insnhelper.c#L2264)。

当case是PRIVATE的分支的时候，会判断是否是FCALL或VCALL，这意味着什么呢？ 如果不是FCALL或VCALL，则抛出方法找不到的异常错误，也就是开篇看到的示例中的错误。

至此，我们已经相对完整的给出了文章标题所示问题的答案。

### 其他收获

```
vm_cref_new_toplevel(rb_thread_t *th)
     {
         rb_cref_t *cref = vm_cref_new(rb_cObject, METHOD_VISI_PRIVATE /* toplevel visibility is private */, FALSE, NULL, FALSE);
…     
         cref = vm_cref_new(th->top_wrapper, METHOD_VISI_PRIVATE, FALSE, cref, FALSE);
        }

         return cref;
     }
```

在搜索中还发现了Ruby默认顶级作用域是private的事实。

### 小结

读者自己总结吧。
