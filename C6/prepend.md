# Ruby的prepend实现有关的问题


---

第6章中讲了Ruby prepend 模块的实现方式，我产生一个疑问： 当Ruby在把目标类副本设置为前置模块的超类，把原始的目标类方法移动到目标类副本中之后，原始目标类中那些方法还存在吗？


根据prepend造成的行为，原始目标类中那些方法应该是不存在了。但是这只是猜测，Pat在书中也未明说，我需要看到证据。

让我们看看编译的YARV指令：


```ruby
 code = <<-CODE
   module A
     def tt
      puts 'aa'
     end
   end

   class B
     prepend A
     def tt
      puts "bb"
     end
   end

   b = B.new
   b.tt
 CODE

puts RubyVM::InstructionSequence.compile(code).disasm

```

从输出中我们发现下面这句指令：

```ruby
0012 opt_send_simple  <callinfo!mid:prepend, argc:1, FCALL|ARGS_SKIP>
```

只执行了prepend函数调用，再没有发现删除类中方法之类的指令。

那么我们继续从Ruby源码中挖掘其中的秘密。

按老方法(参考：[问题1: 关于obj.class方法的疑问](../C5/about_obj_class.md))，我们发现了prepend的[主要功能函数](https://github.com/ruby/ruby/blob/2e2bd1c26b21ab3298b32f881bccebc14c7cac3d/class.c#L935)： 

```c
void
rb_prepend_module(VALUE klass, VALUE module)
{
    VALUE origin;
    int changed = 0;

    rb_frozen_class_p(klass);

    Check_Type(module, T_MODULE);

    OBJ_INFECT(klass, module);

    origin = RCLASS_ORIGIN(klass);
    if (origin == klass) {
    origin = class_alloc(T_ICLASS, klass);
    OBJ_WB_UNPROTECT(origin); /* TODO: conservative shading. Need more survey. */
    RCLASS_SET_SUPER(origin, RCLASS_SUPER(klass));
    RCLASS_SET_SUPER(klass, origin);
    RCLASS_SET_ORIGIN(klass, origin);
    RCLASS_M_TBL(origin) = RCLASS_M_TBL(klass);
    RCLASS_M_TBL_INIT(klass);
    rb_id_table_foreach(RCLASS_M_TBL(origin), move_refined_method, (void *)RCLASS_M_TBL(klass));
    }
    changed = include_modules_at(klass, klass, module, FALSE);
    if (changed < 0)
    rb_raise(rb_eArgError, "cyclic prepend detected");
    if (changed) {
    rb_vm_check_redefinition_by_prepend(klass);
    }
}
```

klass，就是原始类，而origin是原生类，也就是原始类的副本。 我们发现    RCLASS_M_TBL(origin) = RCLASS_M_TBL(klass); 这句代码，是设置M_TBL，也就是类的方法表，从原始类转移给原生类。 紧接着下面的那句：     RCLASS_M_TBL_INIT(klass);，非常可疑。 因为整个函数没有发现DEL之类的跟删除有关系的函数调用。

RCLASS_M_TBL_INIT也是定义在class.c文件中，只有一行代码：

```c
static void
RCLASS_M_TBL_INIT(VALUE c)
{
    RCLASS_M_TBL(c) = rb_id_table_create(0);
}
```

我们继续搜索rb_id_table_create的定义，在id_table.h文件中发现：

```c
struct rb_id_table *rb_id_table_create(size_t size);
```

看到这里大概明白了， rb_id_table_create(0); 这句代码，就是把方法表初始化了，归零了。


答案显而易见了。
