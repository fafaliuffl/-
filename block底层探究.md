##block底层探究（一）
iOS SDK 4.0开始，Apple引入了block这一特性。趁最近比较闲，来研究一下block底层实现方式。先来看一段简单的代码
```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int a = 10;
        int b = 11;
        int (^block)(int,int) = ^(int a, int b) {
            return a+b;
        };
        block(a,b);
    }
    return 0;
}
```
在上面代码中创建了一个带有两个参数的block并调用它。将代码保存为 main.m文件使用命令
> xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m

将代码转换为底层c++。会生成一个main.cpp文件。包含大约3w行代码。直接看最后面关键部分：
```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static int __main_block_func_0(struct __main_block_impl_0 *__cself, int a, int b) {

            return a+b;
        }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        int a = 10;
        int b = 11;
        int (*block)(int,int) = ((int (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
        ((int (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, a, b);
    }
    return 0;
}
```
在main函数里，我们可以看到block变量创建方式。声明block是一个指向`int (*)(int,int)`类型的函数指针。内部的值为
```
((int (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA))
```
看起来很头疼。我们来一步一步分解。
首先发现调用了一个函数
```
__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA))
```
而 **__main_block_impl_0** 是什么东西呢，翻看main函数上面的代码我们可以发现 **__main_block_impl_0**是一个结构体，内部为
```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
也就是说 我们的**block**本质上其实是一个指向 **__main_block_impl_0**结构体的指针。
该结构体里面有两个变量：
* **struct __block_impl impl** 
* **struct __main_block_desc_0** * **Desc**

 **__block_impl** 和 **__main_block_desc_0** 类型 搜索上面代码可以找到类型定义：
```
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```
可以看到 **__block_impl** 里面有一个isa指针。大胆猜测其实block本质也是一个Objective-C对象。
**__main_block_desc_0**里面储存着**__main_block_impl_0**变量的大小也就是block变量所占的内存。

构造该结构体时传入了两个参数。第一个参数为**__main_block_func_0**再翻看上面代码我们发现** __main_block_func_0**是一个函数
```
static int __main_block_func_0(struct __main_block_impl_0 *__cself, int a, int b) {

            return a+b;
        }
```
对比一下，发现函数里面实现跟我们原本block内部实现是一样的。所以这个函数应该就是block本体。这个函数指针被传入了** __main_block_impl_0**结构体里面的 **impl** 中，被 **FuncPtr** 变量持有。而第二个参数大概是一些描述信息。
到了调用时我们的调用被转换成了：
```
((int (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, a, b);
```
又是一连串的类型转换。我们慢慢来抽：
从括号里面大致可以看出
```
((int (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)
```
是一个函数头，后面是传入参数
继续分析发现实质上是调用了 
```
block->FuncPtr
```
这个函数，而通过前面我们得知**FuncPtr**函数实质上就是我们包裹在block中的代码
这里有一个问题，我们知道**block**是一个**__main_block_impl_0**类型的对象。而这个类型里面直接拿不到**FuncPtr**的。要通过**impl**才能拿到的。那为什么这里可以这么写呢。
我们知道C语言（包括C++）结构体本质是直接按照先后顺序（整齐）排列在内存中的。而取结构体内部的本质就是内存偏移。**impl**变量是**__main_block_impl_0**类型的第一个变量。所以**impl**的初始内存地址就是**block**的初始内存地址。**FuncPtr**在**block**中偏移位置与在**impl**中偏移位置是一样的。所以可以通过类型转换
```
(__block_impl *)block)->FuncPtr
```
直接获取**FuncPtr**这个函数调用。
以上

##block底层探究（二）
[Objective-C底层探究之block（一）](https://www.jianshu.com/p/9f31bf83635b)

从前面我们知道了block调用其实就是函数的调用。block本身用结构体做了一些封装。那现在又有一个疑问。block中可以无缝使用外部的变量。加上**__block**关键字还可以在内部修改变量又是怎么实现呢
我们可以把前面的代码改一下：

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int a = 10;
        int b = 11;
        int c = 13;
        int (^block)(int,int) = ^(int a, int b) {
            return a+b+c;
        };
        block(a,b);
    }
    return 0;
}
```

同样使用命令解析得到C++代码

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int c;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _c, int flags=0) : c(_c) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static int __main_block_func_0(struct __main_block_impl_0 *__cself, int a, int b) {
  int c = __cself->c; // bound by copy

            return a+b+c;
        }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        int a = 10;
        int b = 11;
        int c = 13;
        int (*block)(int,int) = ((int (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, c));
        ((int (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, a, b);
    }
    return 0;
}
```

我们来看看与之前例子不同的部分：

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int c;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _c, int flags=0) : c(_c) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static int __main_block_func_0(struct __main_block_impl_0 *__cself, int a, int b) {
  int c = __cself->c; // bound by copy

            return a+b+c;
        }
```

发现**__main_block_impl_0**结构体里面多了一个变量**c**而在**block**创建的时候将**c**变量通过构造函数传了进来

```
int (*block)(int,int) = ((int (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, c));
```

也就是说**c**变量实质上是做了一层拷贝。而在函数调用时通过**__cself**参数拿到block的指针并把**c**拷贝一份局部变量后使用。
那加了**__block**关键字之后是怎么实现在内部改变外部值的呢，我们修改代码如下：

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int a = 10;
        int b = 11;
        __block int c = 13;
        int (^block)(int,int) = ^(int a, int b) {
            c = 14;
            return a+b+c;
        };
        block(a,b);
    }
    return 0;
}
```

同样使用命令解析：

```
struct __Block_byref_c_0 {
  void *__isa;
__Block_byref_c_0 *__forwarding;
 int __flags;
 int __size;
 int c;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_c_0 *c; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_c_0 *_c, int flags=0) : c(_c->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static int __main_block_func_0(struct __main_block_impl_0 *__cself, int a, int b) {
  __Block_byref_c_0 *c = __cself->c; // bound by ref

            (c->__forwarding->c) = 14;
            return a+b+(c->__forwarding->c);
        }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->c, (void*)src->c, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->c, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        int a = 10;
        int b = 11;
        __attribute__((__blocks__(byref))) __Block_byref_c_0 c = {(void*)0,(__Block_byref_c_0 *)&c, 0, sizeof(__Block_byref_c_0), 13};
        int (*block)(int,int) = ((int (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_c_0 *)&c, 570425344));
        ((int (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, a, b);
    }
    return 0;
}
```

可以发现block里面多了一个指针 `__Block_byref_c_0` 类型的指针。而block外面加了__block 标记的变量c被解释成 `__Block_byref_c_0` 类型的结构体。并在调用时将c的地址传入block的构造函数中。block解析的结构体也多了一个指向`__Block_byref_c_0`对象的 c 指针用于储存该地址。这样就可以在block结构体内改变c变量的值了。
