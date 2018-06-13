# 第二章Blocks

## 2.1 Blocks概要

## 前言

### 需要先知道的

#### Objective-C 转 C++的方法

因为需要看Block操作的C++源码，所以需要知道转换的方法，自己转过来看一看：
1. 在OC源文件block.m写好代码。
2. 打开终端，cd到block.m所在文件夹。
3. 输入clang -rewrite-objc block.m，就会在当前文件夹内自动生成对应的block.cpp文件。

### 关于几种变量的特点

#### c语言的函数中可能使用的变量：

* 函数的参数
* 自动变量（局部变量）
* 静态变量（静态局部变量）
* 静态全局变量
* 全局变量

而且，由于存储区域特殊，这其中有三种变量是可以在任何时候以任何状态调用的：

* 静态变量
* 静态全局变量
* 全局变量

而其他两种，则是有各自相应的作用域，超过作用域后，会被销毁。

### 2.2.1 什么是Blocks
* Blocks是C语言的扩充功能——“带有自动变量（即局部变量）的匿名函数”。
* Blocks提供了类似由C++和oc类生成实例或对象来保持变量值得方法，其代码量与编写C语言函数差不多；
* 使用Blocks可以不声明C++和oc类，也没有使用静态变量、静态全局变量或全局变量时的问题，仅用编写C语言函数的源代码量即可使用带有自动变量值的匿名函数。


## 2.2 Blocks模式

### 2.2.1 Block 语法
* 与一般的C语言函数定义相比，完整形式的Block语法有两点不同：
    - 没有函数名称（因为是匿名函数）
    - 带有“^”（便于查找）
* Block表达式完整语法

![image](https://upload-images.jianshu.io/upload_images/1197643-a6928e8e14fb839b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/678)

* 省略返回值类型的Block语法

![image](https://upload-images.jianshu.io/upload_images/1197643-5cafe896853264fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

*省略返回值类型时，如果表达式中有return语句，就使用该返回值的类型，如果表达式中没有return语句，就使用void类型。表达式中含有多个return语句时，所以return语句的返回值类型必须相同

* 省略返回值类型和参数列表的Block语法 

![image](https://upload-images.jianshu.io/upload_images/1197643-f806a6842d5fa31d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

### 2.2.2 Block类型变量（即Block变量）

在Block语法下，可将Block语法赋值给声明为Block类型的变量中（即源代码中一旦使用Block语法就相当于生成了可赋值给Block类型变量的“值”）。“Block”即指源代码中的Block语法，也指由Block语法所生成的值。
* 使用Block语法将Block赋值为Block类型变量。

```
int(^blk)(int) = ^(int count){return count + 1;};
    
```

* 由Block类型变量向Block类型变量赋值。

```
int (^blk1)(int) = blk;
    int (^blk2)(int);
    blk2 = blk1;
```

* 在函数参数中使用Block类型变量可以向函数传递Block。

```
void func(int (^blk)(int))
    {
        
    }
```

* 在函数返回值中指定Block类型，可以将Block作为函数的返回值返回。

```
int (^func()(int))
    {
        return ^(int count) {return count + 1;};
    }
```

* 在函数参数和返回值中使用Block类型变量时，可以通过typedef为Block类型提供别名，从而起到简化块类型变量名的作用。

```
tupedef int (^blk_t) (int);
```

* 通过Block类型变量调用Block与C语言通常的函数调用没有区别（例如，在函数和方法中可以将Block类型变量作为参数）。

```
- (int) methodUsingBlock:(blk_t )blk rate:(int )rate
    {
        return blk (rate);
    }
```

* Block类型变量可完全像通常的C语言变量一样使用（例如，可以使用指向Block类型变量的指针，即Block的指针类型变量）。

```
typedef int (^blk_t) (int);
    blk_t blk = ^(int count ) {return count + 1;};
    blk_t * blkptr = &blk;
    (*blkptr)(10);
```

### 2.2.3 截获自动变量值

Blocks中，Block常量表达式会截获所使用的自动变量的值（即保存该自动变量的瞬间值），从而在执行块时使用。


```
int main()
    {
        int dmy = 256;
        int val = 10;
        const char * fmt = "val = %d\n";
        void (^blk )(void ) = ^{printf(fmt ,val );};
        val = 2;
        fmt = "These valuse were changed. val = %d\n";
        blk();
        return 0;
    }
    
    输出结果：
    val = 10;
    
```
分析：Blocks中，Block表达式截获所使用的自动变量的值，即保存该自动变量的瞬间值，因为block表达式保存了自动变量的值，所以在执行block语法后，即使改写block中使用的自动变量的值也不会影响Block执行时自动变量的值。

### 2.2.4 __block说明符（即存储类型修改符）

使用附有__block说明符的自动变量可在Block中赋值，该变量称为__block变量。

### 2.2.5 截获的自动变量

* 如果将值赋值给Block中截获的自动变量，就会产生编译错误。这种情况下，需要给截获的自动变量附加__block说明符。
* 截获Objective-C对象，调用变更该对象的方法并不会产生编译错误，但是，向截获的自动变量（即所截获的Objective-C对象）赋值则会产生错误。总之，赋值给截获的自动变量会产生编译错误，但使用截获的值却不会有任何问题。
* 在现在的Block中，截获自动变量的方法并没有实现对C语言数组的截获，但是，使用指针可以解决该问题。

```
const char *text = "hello";
void (^blk)(void) = ^{
  printf("%c\n", text[2]);
};
blk();
```

## 2.3 Blocks的实现

### 2.3.1 Block的实质
* 通过支持Block的编译器，含有Block语法的源代码转换为一般C语言编译器能够处理的源代码，并作为极为普通的C语言源代码被编译。
* 这不过是概念上的问题，在实际编译时无法转换成我们能够理解的源代码，Clang（LLVM编译器）具有将含有Block语法的源代码转换为我们可读源代码的功能。通过“-rewrite-objc”选项就能将含有Block语法的源代码变换为C++的源代码（本质是使用了struct结构的C语言源代码）。

```
int main()
{
  void (^blk)(void) = ^{printf("Block\n");};
  blk();
  return 0;
}
```
通过clang转换为以下形式：

```
// 结构体 __block_impl
struct __block_impl {
    void *isa;
    int Flags;      // 标志
    int Reserved;   // 今后版本升级所需的区域
    void *FuncPtr;  // 函数指针
};


// 结构体 __main_block_impl_0
struct __main_block_impl_0 {
    // 成员变量
    struct __block_impl impl; 
    struct __main_block_desc_0* Desc;
    
    // 该结构体的构造函数
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags = 0){
        // _NSConcreteStackBlock用于初始化__block_impl结构体的isa成员
        // （将Block指针赋值给Block的结构体成员变量isa）
        impl.isa = &_NSConcreteStackBlock; 
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};


// 最初的源代码中的Block语法经clang变换，被处理成简单的C语言函数（该函数以Block语法所属的函数名——main和该Block语法在该函数出现的顺序值——0来命名）。
// __ceself为指向Block值的变量。
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    printf("Block\n");
}

// 静态结构体 __main_block_desc_0
static struct __main_block_desc_0{
    unsigned long reserved;     // 今后版本升级所需的区域
    unsigned long Block_size;   // Block的大小
} __mian_block_desc_0_DATA = { // 该结构体实例的初始化部分
    0,
    sizeof(struct __main_block_impl_0) // 使用Block（即__main_block_impl_0结构体实例）的大小进行初始化
};

// main函数，从这里开始阅读源代码
int main()
{
    // 调用结构体__main_block_impl_0的构造函数__main_block_impl_0
    void (*blk)(void) =
        (void (*)(void)) & __main_block_impl_0(
            (void *)__main_block_func_0, &__mian_block_desc_0_DATA);
    /*
    去掉转换部分，如下：
    struct __main_block_impl_0 tmp = __main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA);
    struct __main_block_impl_0 *blk = &tmp;
    这段代码对应:
    void (^blk)(void) = ^{printf("Block\n");};
    理解：
    1. 将__main_block_impl_0结构体类型的自动变量（即栈上生成的__main_block_impl_0结构体实例的指针）赋值给__main_block_impl_0结构体指针类型的变量blk
    2. __main_block_func_0是由Block语法转换的C语言函数指针。
    3. __main_block_desc_0_DATA作为静态全局变量初始化的__main_block_desc_0结构体实例指针
    4. 将__main_block_impl_0的__block_impl进行展开，__main_block_impl_0结构体根据构造函数会像下面进行初始化：
    isa = &_NSConcreteStackBlock; 
    Flags = 0;
    Reserved = 0;
    FuncPtr = __main_block_func_0;
    Desc = &__main_block_desc_0_DATA;
    */ 
    
    
    ((void (*)(struct __block_impl *))(
        (struct __block_impl *)blk)->FuncPtr) ((struct __block_impl *)blk);
    /*
    去掉转换部分，如下：
    (*blk->impl.FuncPtr)(blk);
    这段代码对应：
    blk();
    理解：
    1. 使用函数指针调用函数
    2. 由Block语法转换的__main_block_func_0函数的指针被赋值成员变量FuncPtr中
    3. __main_block_func_0函数的参数__cself指向Block值,在调用该函数的源代码中可以看出Block正是作为参数进行了传递
    */
    
    return 0;
}

```
### OC类和对象的实质（Block就是oc对象）
* 在弄清楚Block就是Objective-C对象前，要先理解objc_object结构体和objc_class结构体。
* id类型是objc_object结构体的指针类型。

```
typedef struct objc_object {
        Class isa;
 } *id;
```
* Class是objc_class结构体的指针类型。

```
typedef struct objc_class *Class;
  struct objc_class {
         Class isa;
  } ;
```
* objc_object结构体和objc_class结构体归根到底是各个对象和类的实现中最基本的结构体。
* 如下,通过一个简单的MyObject类来说明Objective-C类与对象的实质：

```
@interface MyObject : NSObject
{
  int val0;
  int val1;
}
```
* 基于objc_object结构体，该类的对象的结构体如下：

```
struct MyObject {
  Class isa; // 成员变量isa持有该类的结构体实例指针
  int val0;  // 原先MyObject类的实例变量val0和val1被直接声明为成员变量
  int val1;
}
```
理解：

* MyObject类的实例变量val0和val1被直接声明为对象的成员变量。
* “Objective-C中由类生成对象”意味着，像该结构体这样“生成由该类生成的对象的结构体实例”。
* 生成的各个对象（即由该类生成的对象的各个结构体实例），通过成员变量isa保持该类的结构体实例指针。

![image](https://upload-images.jianshu.io/upload_images/1197643-029e91aad6ffe8c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

各类的结构体是基于objc_class结构体的class_t结构体：

```
struct class_t {
  struct class_t *isa;
  struct class_t *superclass;
  Cache cache;
  IMP *vtable;
  uintptr_t data_NEVER_USE;
}
```
* 理解：
    - 在Objective-C中，比如NSObject的class_t结构体实例以及NSMutableArray的class_t结构体实例等，均生成并保持各个类的class_t结构体实例。
    - 该实例持有声明的成员变量、方法的名称、方法的实现（即函数指针）、属性以及父类的指针，并被Objective-C运行时库所使用。
* 回到正题——“Block就是Objective-C对象”，*** 先看Block结构体：***

```
struct __main_block_impl_0 {
    void *isa;
    int Flags;      // 标志
    int Reserved;   // 今后版本升级所需的区域
    void *FuncPtr;  // 函数指针
    struct __main_block_desc_0* Desc;
};

```
理解：
* 此__main_block_impl_0结构体相当于基于objc_object结构体的Objective-C类的对象的结构体。
* 对其中的isa进行初始化，如

```
isa = &_NSConcreteStackBlock;
```
即_NSConcreteStackBlock相当于class_t结构体实例
* 在将Block作为Objective-C的对象处理时，关于该类的信息放置于_NSConcreteStackBlock中。

### 2.3.2 获取自动变量值
截获自动变量值的源代码经clang转换，如下：

```
// 结构体 __block_impl
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};

// 结构体 __main_block_impl_0
struct __main_block_impl_0 {
    // 成员变量
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    const char *fmt; // Block语法表达式“使用的自动变量”被追加到该结构体
    int val;
    
    // 构造函数
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, const char *_fmt, int _val, int flags = 0) : fmt(_fmt), val(_val) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

// 静态函数 __main_block_func_0
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    const char *fmt = __cself->fmt;
    int val = __cself->val;
    /*
    理解：
    1. __main_block_impl_0结构体实例（即Block）所截获的自动变量在Block语法表达式执行之前就被声明定义，所以，在Objective-C的源代码中，执行Block语法表达式时无需改动便可使用截获的自动变量值。
    2. "截获自动变量值"意味着在执行Block语法时，Block语法表达式所使用的自动变量值被保存到Block的结构体实例（即Block自身）中。
    3. Block不能直接使用“C语言数组类型的自动变量”，所以，截获自动变量时，会将其值传递给结构体的构造函数进行保存
    */
    
    printf(fmt, val);
}

// 静态结构体 __main_block_desc_0
static struct __main_block_desc_0{
    unsigned long reserved;
    unsigned long Block_size;
} __mian_block_desc_0_DATA = {
    0,
    sizeof(struct __main_block_impl_0)
};

// 主函数，从这里开始阅读源代码
int main()
{
    int dmy = 256;
    int val = 10;
    const char *fmt = "val = %d\n";
    // 调用结构体__main_block_impl_0的构造函数初始化该结构体实例
    void (*blk)(void) = &__main_block_impl_0(__main_block_func_0, &__mian_block_desc_0_DATA, fmt, val);
    /*
    理解：
    1. 在初始化结构体实例时，会根据传递给构造函数的参数对由自动变量追加的成员变量进行初始化（即执行Block语法使用的自动变量fmt和val会初始化结构体实例）
    2. __main_block_impl_0结构体实例的初始化如下：
      impl.isa = &_NSConcreteStackBlock;
      impl.Flags = flags;
      impl.FuncPtr = fp;
      Desc = desc;
      fmt = "val = %d\n";
      val = 10;
    3. 由上可知，在__main_block_impl_0结构体实例（即Block）中，自动变量被截获。
    */
    
    return 0;
}

```

### 2.3.3 __block说明符

截获自动变量值的源代码经clang转换，如下：

```
// 结构体 __block_impl
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};

// 结构体 __main_block_impl_0
struct __main_block_impl_0 {
    // 成员变量
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    const char *fmt; // Block语法表达式“使用的自动变量”被追加到该结构体
    int val;
    
    // 构造函数
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, const char *_fmt, int _val, int flags = 0) : fmt(_fmt), val(_val) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

// 静态函数 __main_block_func_0
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    const char *fmt = __cself->fmt;
    int val = __cself->val;
    /*
    理解：
    1. __main_block_impl_0结构体实例（即Block）所截获的自动变量在Block语法表达式执行之前就被声明定义，所以，在Objective-C的源代码中，执行Block语法表达式时无需改动便可使用截获的自动变量值。
    2. "截获自动变量值"意味着在执行Block语法时，Block语法表达式所使用的自动变量值被保存到Block的结构体实例（即Block自身）中。
    3. Block不能直接使用“C语言数组类型的自动变量”，所以，截获自动变量时，会将其值传递给结构体的构造函数进行保存
    */
    
    printf(fmt, val);
}

// 静态结构体 __main_block_desc_0
static struct __main_block_desc_0{
    unsigned long reserved;
    unsigned long Block_size;
} __mian_block_desc_0_DATA = {
    0,
    sizeof(struct __main_block_impl_0)
};

// 主函数，从这里开始阅读源代码
int main()
{
    int dmy = 256;
    int val = 10;
    const char *fmt = "val = %d\n";
    // 调用结构体__main_block_impl_0的构造函数初始化该结构体实例
    void (*blk)(void) = &__main_block_impl_0(__main_block_func_0, &__mian_block_desc_0_DATA, fmt, val);
    /*
    理解：
    1. 在初始化结构体实例时，会根据传递给构造函数的参数对由自动变量追加的成员变量进行初始化（即执行Block语法使用的自动变量fmt和val会初始化结构体实例）
    2. __main_block_impl_0结构体实例的初始化如下：
      impl.isa = &_NSConcreteStackBlock;
      impl.Flags = flags;
      impl.FuncPtr = fp;
      Desc = desc;
      fmt = "val = %d\n";
      val = 10;
    3. 由上可知，在__main_block_impl_0结构体实例（即Block）中，自动变量被截获。
    */
    
    return 0;
}

```

### 2.3.4 Block存储域

从“__block说明符”一节中可知，*** Block转换为Block的结构体类型的自动变量,___block变量转换为__block的结构体类型的自动变量。所谓结构体类型的自动变量，即栈上生成的该结构体的实例。 ***

表 Block与__block变量的实质

名称 | 实质
---|---
Block | 栈上block的结构体实例
_block变量 | 栈上_block变量的结构体实例

* 从“Block的实质”一节中可知，Block是Objective-C对象，并且该Block的类为_NSConcreteStackBlock。此外，与之类似的还有两个类：

* _NSConcreteStackBlock —— *** 该类对象设置在栈上 ***
* _NSConcreteGlobalBlock —— *** 该类对象设置在程序的数据区域（.data区）中 ***
* _NSConcreteMallocBlock —— *** 该类对象设置在由malloc函数分配的内存块（即堆中） ***

在内存中的位置如下图：

![image](https://upload-images.jianshu.io/upload_images/1197643-ee6950148454e42e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

#### 注意
* 由于_NSConcreteGlobalBlock类生成的Block对象设置在程序的数据区域中（该类会运用在“全局变量”的声明中），由此该类的结构体实例的内容不依赖于执行时的状态，所以整个程序只需一个实例。
* 只在截获自动变量时，Block的结构体实例截获的值才会根据执行时的状态变化，而在不截获自动变量时，Block的结构体实例每次截获的值都相同。也就是说，即时在函数内而不在记述广域变量的地方使用Block语法时，只要Block不截获自动变量，就可以将Block的结构体实例设置在程序的数据区域。

#### 总结
* Block为_NSConcreteGlobalBlock类对象（即Block配置在程序的数据区域中）的情况有两种：

    - 记述全局变量的地方有Block语法时
    - Block语法表达式中不使用“应截获的自动变量”时
* 除此之外的Block语法生成的Block为_NSConcreteStackBlock类对象（即类对象设置在栈上）。
* 而，_NSConcreteMallocBlock类是Block超出变量作用域可存在的原因。

#### 遗留问题
“__block说明符”一节中遗留的问题：

* Block超出变量作用域可存在的原因
* __block变量的结构体成员变量__forwarding存在的原因

配置在全局变量的Block，从变量作用域外也可以通过指针安全的使用，而配置在栈上的Block，如果其所属的变量作用域结束，该Block就被废弃。如图：

![image](https://upload-images.jianshu.io/upload_images/1197643-efa816793fab3e9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

为此，Blocks提供了将Block和__block变量从栈上复制到堆上的方法来解决这个问题。

如图：

![image](https://upload-images.jianshu.io/upload_images/1197643-a0a084ddf38f45c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

实现机制：
* 复制到堆上的Block将_NSConcreteMallocBlock类对象写入Block的结构体实例的成员变量isa。

```
impl.isa = &_NSConcreteMallocBlock;
```
而__block变量的结构体成员变量__forwarding可以实现无论__变量配置在栈上还是堆上时都能够正确地访问__block变量。

#### ARC有效时，大多数情况下Block从栈上复制到堆上的代码由编译器实现

实际上，ARC有效时，大多数编译器会恰当地进行判断，自动生成将Block从栈上复制到堆上的代码。


```
typedef int (^blk_t)(int);
blk_t func(int rate)
{
  return ^(int count){return rate * count;};
}
```
该源代码中的函数会返回配置在栈上的Block。即当程序执行从该函数返回函数调用方时，变量作用域结束，因此栈上的Block也被废弃。虽然有这样的问题，但该源代码通过对应ARC的编译器可转换如下：

```
blk_t func(int rate)
{
  blk_t tmp = &__func_block_impl_0(__func_block_func_0, &__func_block_desc_0_DATA, rate);
  
  tmp = objc_retainBlock(tmp);
  
  return objc_autoreleaseReturnValue(tmp);
}


```
理解：

* ARC有效时，blk_t tmp 相当于blk_t __strong tmp.
* objc_retainBlock实际上是Block_copy函数。
详细的注释：

```
blk_t func(int rate)
{
  blk_t tmp = &__func_block_impl_0(__func_block_func_0, &__func_block_desc_0_DATA, rate);
  
  /*
   * 将通过Block语法生成的Block（即配置在栈上的Block结构体实例）
   * 赋值给相当于Block类型的变量tmp
   */
  
  tmp = _Block_copy(tmp);
  
  /*
   * _Block_copy函数
   * 将栈上的Block复制到堆上
   * 复制后，将堆上的地址作为指针赋值给变量tmp
   */
  
  return objc_autoreleaseReturnValue(tmp);
  
  /*
   * 将堆上的Block作为Objective-C对象
   * 注册到autoreleasepool中，然后返回该对象
   */
}

```
*** 将Block作为函数返回值返回时，编译器会自动生成复制到堆上的代码。 ***

#### 在少数情况下，Block从栈上复制到堆上的代码的手动实现

如果*** 向方法或函数的参数中传递Block时 ***，编译器将不能进行判断，需要使用“copy实例方法”手动复制。
但是，以下方法或函数不需要手动复制：

* Cocoa框架的方法且方法名中含有usingBlock等时
* Grand Central Dispathc 的API


*** 对Block语法调用copy方法 ***


```
- (id)getBlockArray
{
  int val = 10;
  return [[NSArray alloc] initWithObjects: ^{NSLog(@"blk:%d", val);},
                                           ^{NSLog(@"blk:%d", val);}, nil];
}

id obj = getBlockArray();

typedef void (^blk_t)(void);

blk_t blk = (blk_t)[obj objectAtIndex:0];

blk();

```
该源代码的blk()，即Block在执行时发生异常，应用程序强制结束。这是由于*** getBlockArray函数执行结束时，栈上的Block被废弃的缘故。 *** 此时，编译器不能判断是否需要复制。也可以不让编译器进行判断，而使其在所有情况下都能复制。但将Block从栈上复制到堆上是相当消耗CPU的。当Block设置在栈上也能够使用时，将Block从栈上复制到堆上只是在浪费CPU资源。因此只在此情形下让编程人员手动进行复制。

对源代码修改一下，便可正常运行：


```
- (id)getBlockArray
{
  int val = 10;
  return [[NSArray alloc] initWithObjects: [^{NSLog(@"blk:%d", val);} copy],
                                           [^{NSLog(@"blk:%d", val);} copy], nil];
}

id obj = getBlockArray();

typedef void (^blk_t)(void);

blk_t blk = (blk_t)[obj objectAtIndex:0];

blk();

```
*** 对Block类型变量调用copy方法 ***

#### 按配置Block的存储域，使用copy方法产生的复制效果

表 Block的副本


Block的类 | 副本源的配置存储域 | 复制效果
---|---|---
_NSConcreteStackBlock | 栈 | 从栈复制到堆
_NSConcreteGlobalBlock | 程序的数据区域 | 什么也不做
_NSConcreteMallocBlock | 堆 | 引用计数增加

不管Block配置在何处，用copy方法复制都不会引起任何问题。在不确定时调用copy方法即可。

#### 在ARC有效时，多次调用copy方法完全没有问题

```
blk = [[[[blk copy] copy] copy] copy];
// 经过多次复制，变量blk仍然持有Block的强引用，该Block不会被废弃。
```

### 2.3.5 __block变量存储域

从“Block存储域”一节可知，*** 使用__block变量的Block从栈复制到堆上时，__block变量也会受到影响。 ***

表 Block从栈复制到堆时对__block变量产生的影响

__block变量的配置存储域 | Block从栈复制到堆时的影响
---|---
栈 | 从栈复制到堆并被Block持
堆 | 被Block持有

*** 在一个Block中使用__block变量 ***

![image](https://upload-images.jianshu.io/upload_images/1197643-24b327fd0a21849c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

*** 在多个Block中使用__block变量 ***

![image](https://upload-images.jianshu.io/upload_images/1197643-54610824aaa5a365.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

*** Block的废弃和__block变量的释放 ***

![image](https://upload-images.jianshu.io/upload_images/1197643-9e2b317d74bc5670.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

#### 遗留的问题

“Block存储域”一节中遗留的问题：

* 使用__block变量的结构体成员变量__forwarding的原因

> 不管__block变量配置在栈上还是在堆上，都能够正确地访问该变量 

正如这句话所诉，通过Block的复制，__block变量也会从栈复制到堆上。此时可同时访问栈上的__block变量和堆上的__block变量。

```
__block int val = 0;  // __block变量

void (^blk)(void) = [^{++val;} copy]; // Block

++val;

blk();

NSLog(@"%d", val);
```
利用copy方法复制使用了__block变量的Block语法。此时，Block和__block变量均从栈复制到堆。

*** 在Block语法表达式中，使用初始化后的__block变量 ***

```
^{++val;}
```

*** 在Block语法表达式之后，使用与Block无关的__block变量 ***

```
++val;
```

然而，以上两种源代码都可以转换为：

```
++(val.__forwaring->val);
```
在变化Block语法的函数中，该变量val为 *** 复制到堆上的__block变量的结构体实例 ，而使用与Block无关的变量val，为 复制前栈上的__block变量的结构体实例 ***。

但是，栈上的__block变量的结构体实例（即变量val）在__block变量从栈复制到堆上时，会将成员变量__forwarding的值替换为复制目标堆上的__block变量的结构体实例的地址。

![image](https://upload-images.jianshu.io/upload_images/1197643-5d505a00bdd66313.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

###  2.3.6 截获对象

```
{
  id array = [[NSMutableArray alloc] init];
}
```
该源代码生成并持有NSMutableArray类的对象，但是附有__strong修饰符的赋值目标（变量array）变量作用域立即就会结束，因此对象被立即释放并废弃。

```
blk_t blk;

{
  id array = [[NSMutableArray alloc] init];
  blk = [^(id obj){
  
      [array addObject:obj];
      
      NSLog(@"array count = %ld", [array count]);
  } copy]; // 调用copy方法（Block从栈复制到堆）
}

blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);


```
变量作用域结束的同时，变量array被废弃，其对NSMutableArray类的对象的强引用失效，因此NSMutableArray类的对象被释放并废弃（此处我不确定是否会被废弃）。但是，该源代码运行正常，执行结果如下：

```
array count = 1
array count = 2
array count = 3
```
这意味着赋值给变量array的NSMutableArray类的对象在Block的执行部分超出其变量作用域而存在。

经clang转换：

```
/* Block的结构体 / 函数部分  */

// 结构体 __main_block_impl_0
struct __main_block_impl_0 {
    // 成员变量
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    id __strong array; 
    /*
    理解：
    1. 被NSMutableArray类对象并被截获的自动变量array，是附有__strong修饰符的成员变量。在Objective-C中，C语言结构体不能含有附有__strong修饰符的变量。因为编译器不知道何时进行C语言结构体的初始化和废弃操作，不能很好地管理内存。
    2. 但是，Objective-C的运行时库能准确把握Block从栈复制到堆以及堆上的Block被废弃的时机，因此Block的结构体即时含有附有__stong修饰符或__weak修饰符的变量，也可以恰当地进行初始化和废弃。
    */
    
    // 构造函数
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, id __strong_array, int flags =0) : array(_array){
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

// 静态函数 __main_block_func_0
static void __main_block_func_0(struct __main_block_impl_0 *__cself, id obj)
{
    id __strong array = __cself->array;
    
    [array addObject:obj];
    
    NSLog(@"array count = %ld", [array count]);
}

// 静态函数 __main_block_copy_0
static void __main_block_copy_0(struct __main_block_impl_0 *dst, struct __main_block_impl_0 *src){
    _Block_object_assign(&dst->array, src->array, BLOCK_FIELD_IS_OBJECT);
    /*
    理解：
    1. __main_block_copy_0函数使用_Block_object_assign函数将“对象类型对象”赋值给Block的结构体成员变量array中并持有该对象
    2. _Block_object_assign函数调用“相当于ratain实例方法的函数”，将“对象”赋值在对象类型的结构体成员变量中。
    */
}

// 静态函数 __main_block_dispose_0
static void __main_block_dispose_0(struct __main_block_impl_0 *src){
    _Block_object_dispose(src->array, BLOCK_FIELD_IS_OBJECT);
    /*
    理解：
    1. __main_block_dispose_0函数使用_Block_object_dispose函数，释放赋值在Block的结构体成员变量array中的对象。
    2. _Block_object_dispose函数调用相当于release实例方法的函数，释放赋值在对象类型的结构体成员变量中的对象。
    */
}

// 静态结构体 __main_block_desc_0
static struct __main_block_desc_0{
    unsigned long reserved;
    unsigned long Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
    void (*dispose)(struct __main_block_impl_0*);
} __mian_block_desc_0_DATA = {
    0,
    sizeof(struct __main_block_impl_0),
    __main_block_copy_0,
    __main_block_dispose_0
};
/*
理解：
1. __main_block_copy_0函数（copy函数）和__main_block_dispose_0函数（dispose函数）指针被赋值__main_block_desc_0结构体成员变量copy和dispose中，但是在转换后的源代码中，这些函数包括使用指针全都没有被调用。
2. 而是，在Block从栈复制到堆时以及堆上的Block被废弃时会调用这些函数。
*/

```

```
/* Block语法，使用Block部分 */

blk_t blk;

{
    id __strong array = [[NSMutableArray alloc] init];
    
    blk = &__main_block_impl_0(__main_block_func_0, &__mian_block_desc_0_DATA, array, 0x22000000);
    blk = [blk copy];
}

(*blk->impl.FuncPtr)(blk, [[NSObject alloc] init]);
(*blk->impl.FuncPtr)(blk, [[NSObject alloc] init]);
(*blk->impl.FuncPtr)(blk, [[NSObject alloc] init]);

```
表 调用copy函数和dispose函数的时机

函数 | 调用时机
---|---
copy函数 | 栈上的Block复制到堆时
dispose函数 | 堆上的Block被废弃时

*** 何时栈上的Block会复制到堆 ***

* 调用Block的copy实例方法时
* Block作为函数返回值返回时
* 将Block赋值给附有__strong修饰符id类型的类或Block类型成员变量时
* 在方法名中含有usingBlock的Cocoa框架方法或GCD的API中传递Block时

在栈上的Block被复制到堆时，copy函数被调用，而在释放复制到堆上的Block后，谁都不持有Block而被废弃时，dispose函数被调用。正因为这种构造，通过使用附有__strong修饰符的自动变量，Block中截获的对象才能给超出其变量作用域而存在。

*** 如何区分copy函数和dispose函数的对象类型 ***

表 截获对象时和使用__block变量时的不同

对象 | BLOCK_FIELD_IS_OBJECT
---|---
__block变量 | BLOCK_FIELD_IS_BYREF

* 通过BLOCK_FIELD_IS_OBJECT和BLOCK_FIELD_IS_BYREF参数，区分copy函数和dispose函数的对象类型是对象还是__block变量。

* 但是，与copy函数持有被截获的对象，dispose函数释放截获的对象相同，copy函数持有所使用的__block变量，dispose函数释放所使用的__block。

* 由此可知，Block中使用的赋值给附有__stong修饰符的自动变量的对象和复制到堆上的__block变量，由于被堆上的Block所持有，因而可超出其变量作用域而存在。

*** 何种情形下，不调用Block的copy实例方法 ***
在Block使用对象类型自动变量是，除以下情形外，推荐调用Block的copy实例方法：

* Block作为函数返回值返回时
* 将Block赋值给附有__strong修饰符id类型的类或Block类型成员变量时
* 在方法名中含有usingBlock的Cocoa框架方法或GCD的API中传递Block时

### 2.3.7 __block变量和对象

*** 附加__strong修饰符的对象类型__block变量和自动变量***

__block说明符可指定“任何类型”的自动变量。

下面指定用于赋值Objective-C对象的id类型自动变量：

```
__block id obj = [[NSObject alloc] init];
```
ARC有效时，id类型以及对象类型变量默认附加__strong修饰符。
所以，该代码等同于：

```
__block id __strong obj = [[NSObject alloc] init];
```
经clang转换：

```
/* __block变量的结构体部分 */

// 结构体 __Block_byref_obj_0
struct __Block_byref_obj_0 {
    void *__isa;
    __Block_byref_obj_0 *__forwarding;
    int __flags;
    int __size;
    void (*__Block_byref_id_object_copy)(void*, void*);
    void (*__Block_byref_id_object_dispose_)(void*);
    __strong id obj; // __block变量被追加为成员变量
};

// 静态函数 __Block_byref_id_object_copy_131
static void __Block_byref_id_object_copy_131(void *dst, void *src){
    _Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}

// 静态函数 __Block_byref_id_object_dispose_131
static void __Block_byref_id_object_dispose_131(void *src){
    _Block_object_dispose(*(void * *) ((char*)src + 40), 131);
}


```

```
/* __block变量声明部分 */

__Block_byref_obj_0 obj = {
    0,
    &obj,
    0x20000000,
    sizeof(__Block_byref_obj_0),
    __Block_byref_id_object_copy_131,
    __Block_byref_id_object_dispose_131,
    [[NSObject alloc] init]
};

```
在Block中使用“附有__strong修饰符的id类型或对象类型自动变量”的情况下，当Block从栈复制到堆时，使用_Block_object_copy函数，持有Block截获的对象。当堆上的Block被废弃时，使用_Block_object_dispose函数，释放Block截获的对象。

在__block变量为“附有__strong修饰符的id类型或对象类型自动变量”的情形下会发生同样的过程。当__block变量从栈复制到堆时，使用_Block_object_copy函数，持有赋值给__block变量的对象。当堆上的__block变量被废弃时，使用_Block_object_dispose函数，释放赋值给__block变量的对象。

由此可知，即时对象赋值给“复制到堆上的附有__strong修饰符的对象类型__block变量”中，只要__block变量在堆上继续存在，那么该对象就会继续处于被持有的状态。这与在Block中对象赋值给“附有__strong修饰符的对象类型自动变量”相同。

*** 附有__weak修饰符的id类型自动变量 ***
在Block中使用附有__weak修饰符的id类型自动变量：


```
blk_t blk;

{
    id array = [[NSMutableArray alloc] init];
    id __weak weakArray = array;
    
    blk = [^(id obj){
        
        [weakArray addObject:obj];
        
        NSLog(@"weakArray count = %ld", [weakArray count]);
    } copy];
}

blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
```
执行结果：

```
weakArray count = 0
weakArray count = 0
weakArray count = 0
```
该段代码能够正常运行。这是因为在变量作用域结束时，附有__strong修饰符的自动变量array所持有的NSMutableArray类对象会被释放被废弃，而附有__weak修饰符的自动变量weakArray由于对NSMutableArray类对象持有弱引用，此时nil赋值在自动变量weakArray上。

*** 附有__weak修饰符的__block变量 ***


```
blk_t blk;

{
    id array = [[NSMutableArray alloc] init];
    __block id __weak blockWeakArray = array;
    
    blk = [^(id obj){
        
        [blockWeakArray addObject:obj];
        
        NSLog(@"blockWeakArray count = %ld", [blockWeakArray count]);
    } copy];
}

blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);

```
执行结果：

```
blockWeakArray count = 0
blockWeakArray count = 0
blockWeakArray count = 0
```
这段代码也能正常运行。这是因为即时附加了__block说明符，在变量作用域结束时，附有__strong修饰符的自动变量array所持有的NSMutableArray类对象会被释放被废弃，而附有__weak修饰符的自动变量blockWeakArray由于对NSMutableArray类对象持有弱引用，此时nil赋值在自动变量blockWeakArray上。

#### 注意

* 由于附有__unsafe_unretained修饰符的变量只不过与指针相同，所以不管在Block中使用还是附加到__block变量中，也不会像__strong修饰符或__weak修饰符那样进行处理。在使用附有__unsafe_unretained修饰符的变量时，注意不要通过悬挂指针访问已被废弃的对象，否则程序可能会崩溃！
* 没有设定__autoreleasing修饰符与Block同时使用。
* __autoreleasing修饰符与__block说明符同时使用会产生编译错误。

### 2.3.8 Block循环引用

*** 如果在Block中使用附有__strong修饰符的对象类型自动变量，那么当Block从栈复制到堆时，该对象为Block所持有。 *** 这样容易引起*** 循环引用 ***。

*** 使用Block类型成员变量和附有__strong修饰符的self出现循环引用 ***


```
typedeft void (^blk_t)(void);

@interface MyObject : NSObject
{
    blk_t blk_;
}
@end

@implementation MyObject
- (id)init
{
    self = [super init];
    
    blk_ = ^{NSLog(@"self = %@", self);};
    
    return self;
}

- (void)dealloc
{
    NSLog(@"dealloc");
}
@end

int main()
{
    id o = [[MyObject alloc] init];
    
    NSLog(@"%@", o);
    
    return 0;
}

```
编译该源代码时，编译器会发出警告，这是因为出现了循环引用，从而导致dealloc实例方法没有被调用。

![image](https://upload-images.jianshu.io/upload_images/1197643-19a06f28c713341a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

*** 使用Block类型成员变量和附有__weak修饰符的self避免循环引用 ***
为避免此循环引用，可声明附有__weak修饰符的变量，并将self赋值给该变量，然后在Block语法中使用该变量。


```
- (id)init
{
    self = [super init];
    
    id __weak weakSelf = self;
    
    blk_ = ^{NSLog(@"self = %@", weakSelf);};
    
    return self;
}

```

*** 面向iOS4，使用__unsafe_unretained修饰符 ***

```
- (id)init
{
    self = [super init];
    
    id __unsafe_unretained weakSelf = self;
    
    blk_ = ^{NSLog(@"self = %@", weakSelf);};
    
    return self;
}

```
面向iOS4，使用__unsafe_unretained修饰符替代__weak修饰符，并且不用担心悬挂指针问题。

*** Block中没有使用self，但是截获了self ***

```
@interface MyObject : NSObject
{
    blk_t blk_;
    id obj_;
}
@end

@implementation MyObject
- (id)init
{
    self = [super init];
    
    blk_ = ^{NSLog(@"obj_ = %@", obj_);};
    
    return self;
}


```
该源代码，如果编译，编译器会发出警告（出现循环引用）。这是因为Block语法中使用的obj_实际上截获了self，而对编译器来说，obj_只不过是对象的结构体的成员变量。

```
blk_ = ^{NSLog(@"obj_ = %@", self->obj_);};
```
为避免循环引用，解决方法参考前面。

*** 使用__block变量避免循环引用 ***

```
typedeft void (^blk_t)(void);

@interface MyObject : NSObject
{
    blk_t blk_;
}
@end

@implementation MyObject
- (id)init
{
    self = [super init];
    
    __block id blockSelf = self;
    
    blk_ = ^{
          NSLog(@"self = %@", blockSelf);
          blockSelf = nil; // 记得清零
        };
    
    return self;
}

- (void)execBlock
{
    blk_();
}

- (void)dealloc
{
    NSLog(@"dealloc");
}
@end

int main()
{
    id o = [[MyObject alloc] init];
    
    [o execBlock];
    
    return 0;
}


```
该源代码没有引起循环引用。但是，如果不调用execBlock实例方法（即不执行赋值给成员变量blk_的Block），便会循环引用并引起内存泄露。该种循环引用可参看下面。

*** 使用__block变量不恰当会出现循环引用 ***
在生成并持有MyObject类对象的状态下会引起以下循环引用，如下图：

* MyObject类对象持有Block
* Block持有__block变量
* __block变量持有MyObject类对象

![image](https://upload-images.jianshu.io/upload_images/1197643-61aa9db32cdfc1be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

如果不执行execBlock实例方法，就会持续该循环引用从而造成内存泄露。

如果只想execBlock实例方法，Block被执行，nil被赋值在__block变量blockSelf中。


```
blk_ = ^{
    NSLog(@"self = %@", blockSelf);
    blockSelf = nil; // 记得清零
};
```
因此，__block变量blockSelf对MyObject类对象的强引用失效，从而避免了循环引用，如下图：

* MyObject类对象持有Block
* Block持有__block变量

![image](https://upload-images.jianshu.io/upload_images/1197643-7c3c11ebeff6d5e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

*** 使用__block变量避免循环引用的优缺点 ***

#### 优点

通过__block变量可控制对象的持有期间
在不能使用__weak修饰符的环境中不使用__unsafe_unretained修饰符即可（不必担心悬挂指针）
在执行Block时可动态地决定是否将nil或其他对象赋值在__block变量中，从而避免出现循环引用。

#### 缺点

为避免循环引用必须执行Block
存在执行了Block语法，却不执行Block的路径时，无法避免循环引用。

#### 总结

若由于Block引发了循环引用时，根据Block的用途选择使用__block变量、__weak修饰符或__unsafe_unretained修饰符来避免循环引用。

### 2.3.9 copy/release
*** ARC无效时，需要用copy实例方法手动将Blocl从栈复制到堆，用release实例方法来释放复制的Block。 ***

```
void (^blk_on_heap)(void) = [blk_on_stack copy];

[blk_on_heap release];
```
*** 只要Block有一次复制并配置在堆上，就可通过retain实例方法持有。***

```
[blk_on_heap retain];
```
*** 但是，对于配置在栈上的Block调用retain实例方法则不起任何作用。 ***

```
[blk_on_stack retain];
```
该源代码中，虽然对赋值给blk_on_stack的栈上的Block调用了retain实例方法，但实际上对此源代码不起任何作用。因此，推荐使用copy实例方法来持有Block（不用retain实例方法）。

由于Block是C语言的扩展，所以在C语言中也可以使用Block语法。此时使用“Block_copy函数”和“Block_release函数”代替copy/release实例方法。


```
void (^blk_on_heap)(void) = Block_copy(blk_on_stack);

Block_release(blk_on_heap);
```
*** ARC无效时，__block说明符被用来避免Block中的循环引用。***
这是由于当Block从栈复制到堆时，若Block使用的变量为附有__block说明符的id类型或对象类型的自动变量，不会被retain；若Block使用的变量为没有__block说明符的id类型或对象类型的自动变量，则被retain。


```
typedeft void (^blk_t)(void);

@interface MyObject : NSObject
{
    blk_t blk_;
}
@end

@implementation MyObject
- (id)init
{
    self = [super init];
    
    blk_ = ^{NSLog(@"self = %@", self);};
    
    return self;
}

- (void)dealloc
{
    NSLog(@"dealloc");
}
@end

int main()
{
    id o = [[MyObject alloc] init];
    
    NSLog(@"%@", o);
    
    return 0;
}

```
该源代码无论ARC有效还是无效都会引起循环引用，Block持有self，self持有Block。

可使用__block变量来避免出现该问题。

```
- (id)init
{
    self = [super init];
    
    __block id blockSelf = self; 
    
    blk_ = ^{NSLog(@"self = %@", blockSelf);};
    
    return self;
}


```
这时，由于Block使用__block变量，所以不会被retain。

#### 注意

ARC有效时，__block说明符和__unsafe_unretained修饰符一起使用，来解决附有__unsafe_unretained修饰符的自动变量不能retain的问题。

__block说明符在ARC有效无效时的用途有很大的区别，所以，在使用__block说明符必须清楚源代码是在ARC有效还是无效的情况下编译。
