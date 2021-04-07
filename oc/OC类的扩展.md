#OC类的扩展和注意事项

## 什么是对于OC类的扩展？
### OC类是什么？
学习过Objective-C(简称OC)语言的同学都知道，OC中绝大部分类都是继承自`NSObject`，而什么是`NSObject`呢？
我们看`NSObject.h`，会发现`NSObject`实际上是一个遵循了`NSObject Protocol`，包含一个`Class`属性并提供了若干方法的类。
- 首先它遵循了`NSObject`协议，这使得一个`NSObject`对象可以响应`NSObject Protocol`方法的调用
- 其次它包含了一个类型为`Class`，变量名为`isa`的成员变量，但是这个`isa`被`OBJC_ISA_AVAILABILITY`修饰了
- 然后它定义了一些基础的方法，有些可以供开发者调用，有些只在特定编译环境下可以调用

### OBJC_ISA_AVAILABILITY
先来看看`isa`，这个`OBJC_ISA_AVAILABILITY`是啥意思？通过一番查找，我们在`objc-api.h`中发现了它的定义：
```c
/* OBJC_ISA_AVAILABILITY: `isa` will be deprecated or unavailable 
 * in the future */
#if !defined(OBJC_ISA_AVAILABILITY)
#   if __OBJC2__
#       define OBJC_ISA_AVAILABILITY  __attribute__((deprecated))
#   else
#       define OBJC_ISA_AVAILABILITY  /* still available */
#   endif
#endif
```
宏的意思是，如果没有定义`OBJC_ISA_AVAILABILITY`，那么就定义一个`OBJC_ISA_AVAILABILITY`的宏，但是这个宏的实现是根据`__OBJC2__`来确定的.
在`__OBJC2__`时，`OBJC_ISA_AVAILABILITY`代表`__attribute__((deprecated))`，也就是我们熟知的弃用标记；而在非`__OBJC2__`时，是空定义（`/* still available */`）。

那么再看`OBJC_ISA_AVAILABILITY`，按照它的定义，现在实际上是`deprecated`（被弃用的），我们可以试一下用这个宏来修饰成员变量，编译器确实会报警告。也就是说`isa`这个属性已经是弃用状态了，但是我们知道至少它目前还是在使用的···

###	`Class`又是什么
来到`runtime.h`，可以看到`Class`的定义。
```c
struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;
    const char * _Nonnull name                               OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;
    struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;
    struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */
```
所谓的`Class`类型，实际上是一个`objc_class`的结构体，这个结构体包含了一些成员，其中有一个`OBJC_ISA_AVAILABILITY`宏修饰的`Class`类型的`isa`，还有一些被`!__OBJC2__`包围的成员。但是，这个结构体被标记了`OBJC2_UNAVAILABLE`，什么意思，不能用了么？

#### OBJC2_UNAVAILABLE
我们在`objc-api.h`里，可以找到这个宏的定义
```c
/* OBJC2_UNAVAILABLE: unavailable in objc 2.0, deprecated in Leopard */
#if !defined(OBJC2_UNAVAILABLE)
#   if __OBJC2__
#       define OBJC2_UNAVAILABLE UNAVAILABLE_ATTRIBUTE
#   else
        /* plain C code also falls here, but this is close enough */
#       define OBJC2_UNAVAILABLE                                       \
            __OSX_DEPRECATED(10.5, 10.5, "not available in __OBJC2__") \
            __IOS_DEPRECATED(2.0, 2.0, "not available in __OBJC2__")   \
            __TVOS_UNAVAILABLE __WATCHOS_UNAVAILABLE __BRIDGEOS_UNAVAILABLE
#   endif
#endif
```
这个宏的实现是根据`__OBJC2__`这个条件来确定的。所谓的`__OBJC2__`，其实就是代表OC的版本2.0，是苹果在2006年发布的编程语言更新。可想而知，现在十几年过去了，标记了`!__OBJC2__`的代码都可以说是无效的代码，因为根本不会被编译。

那真正的“`Class`”去哪儿了?
#### `objc_class`
通过翻找`objc runtime`的源码，我们可以找到有一个`objc-runtime-new.h`的文件，在`__OBJC2__`条件下被`#include`到项目的`objc-private.h`中，而这个文件里正有我们想要的`"Class"`的定义。
```c
typedef struct objc_class *Class;
```
也就是说，所谓的`Class`，依然是一个`objc_class`结构体。只是，它的定义在`objc-runtime-new.h`中，如下(由于定义太长了，只截取一部分，感兴趣的同学可以自行翻阅源码)：
```c
struct objc_class : objc_object {
  objc_class(const objc_class&) = delete;
  objc_class(objc_class&&) = delete;
  void operator=(const objc_class&) = delete;
  void operator=(objc_class&&) = delete;
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

   ...
}
```
至此，我们才算只找到了`Class`的真身。

(Tips: 为什么苹果要这样欺骗我们？我也看到了一些讨论，诸如 https://stackoverflow.com/questions/60518472/why-objc-class-has-different-definition-between-runtime-h-and-objc-runtime-new-h)

光从`objc_class`结构中，我们几乎无法看出来它到底是个啥，它由3个成员组成：`superclass、cache和bits`。
`superclass`是指向父类，`cache`是一些缓存，而`bits`就是我们今天的主角——“可扩展项”。


### 扩展什么？
上面说到`class_data_bits`是可扩展项，那么我们能扩展什么呢？
继续翻看`class_data_bits`的定义，我们找到了`class_rw_t`，然后我们可以找到两个结构`class_ro_t`和`class_rw_ext_t`，`ro`其实就是`read only`，而`rw`就是`read write`, `ext`则代表`extension`。

通过梳理这些类型的结构，我们找到了以下关键词：
`class_ro_t`中有
```c
void *baseMethodList;
protocol_list_t * baseProtocols;
const ivar_list_t * ivars;

const uint8_t * weakIvarLayout;
property_list_t *baseProperties;
```
`class_rw_ext_t`中有
```c
method_array_t methods;
property_array_t properties;
protocol_array_t protocols;
```
至此，我们得出结论。我们可以扩展的项有：
1、methods
2、protocols
3、ivars
4、properties

而这些扩展项，有的是只读的，有的是可写的，下面我们来探讨一下该如何进行。

## 如何扩展OC类？

### methods

### protocols

### ivars

### properties