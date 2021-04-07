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
methods在`ro`和`rw`中的存储方式，分别为`method_list_t`和`method_array_t`，首先我们分析一下这两个结构。
#### method_list_t和method_array_t
```c
// Two bits of entsize are used for fixup markers.
// Reserve the top half of entsize for more flags. We never
// need entry sizes anywhere close to 64kB.
//
// Currently there is one flag defined: the small method list flag,
// method_t::smallMethodListFlag. Other flags are currently ignored.
// (NOTE: these bits are only ignored on runtimes that support small
// method lists. Older runtimes will treat them as part of the entry
// size!)
struct method_list_t : entsize_list_tt<method_t, method_list_t, 0xffff0003, method_t::pointer_modifier> {
    bool isUniqued() const;
    bool isFixedUp() const;
    void setFixedUp();

    uint32_t indexOfMethod(const method_t *meth) const {
        uint32_t i = 
            (uint32_t)(((uintptr_t)meth - (uintptr_t)this) / entsize());
        ASSERT(i < count);
        return i;
    }

    bool isSmallList() const {
        return flags() & method_t::smallMethodListFlag;
    }

    bool isExpectedSize() const {
        if (isSmallList())
            return entsize() == method_t::smallSize;
        else
            return entsize() == method_t::bigSize;
    }

    method_list_t *duplicate() const {
        method_list_t *dup;
        if (isSmallList()) {
            dup = (method_list_t *)calloc(byteSize(method_t::bigSize, count), 1);
            dup->entsizeAndFlags = method_t::bigSize;
        } else {
            dup = (method_list_t *)calloc(this->byteSize(), 1);
            dup->entsizeAndFlags = this->entsizeAndFlags;
        }
        dup->count = this->count;
        std::copy(begin(), end(), dup->begin());
        return dup;
    }
};
```
说白了，`method_list_t`就是一个存放`method_t`的list。
```c
class method_array_t : 
    public list_array_tt<method_t, method_list_t, method_list_t_authed_ptr>
{
    typedef list_array_tt<method_t, method_list_t, method_list_t_authed_ptr> Super;

 public:
    method_array_t() : Super() { }
    method_array_t(method_list_t *l) : Super(l) { }

    const method_list_t_authed_ptr<method_list_t> *beginCategoryMethodLists() const {
        return beginLists();
    }
    
    const method_list_t_authed_ptr<method_list_t> *endCategoryMethodLists(Class cls) const;
};
```
也是一个存放`method_t`的list。

那么就是说，如果我们想要扩展方法，就必须得往这两个list中插入我们想要增加的方法，或者把list中的方法实现替换成我们想要的实现。

#### 如何扩充method list
首先`class_ro_t`中的`baseMethodList`，即基本方法列表，几乎是定死的。它只在编译器设置了`ptrauth`（https://developer.apple.com/documentation/security/preparing_your_app_to_work_with_pointer_authentication?language=objc ）时是动态生成的。但是它依然是`不可修改`的。

通过搜索源码，我们发现除了类本身的`baseMethodList`之外，`method_list_t`只出现在`protocol_t`和`category_t`这两个结构中，那是不是可以通过添加`protocol`和`category`，就能够修改`method_list_t`的值呢？
我们先看`category`：
```c
struct category_t {
    const char *name;
    classref_t cls;
    WrappedPtr<method_list_t, PtrauthStrip> instanceMethods;
    WrappedPtr<method_list_t, PtrauthStrip> classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
    
    protocol_list_t *protocolsForMeta(bool isMeta) {
        if (isMeta) return nullptr;
        else return protocols;
    }
};
```
`category`结构提供了两个`method_list_t`，一个是`instanceMethods`，另一个是`classMethods`。字面意思理解，就是一个实例方法列表和一个类方法列表。
另外，我们可以发现`method_list_t *methodsForMeta(bool isMeta)`这个方法，将这两个成员返回，而这个函数，在`attachCategories`(`realizeClassWithoutSwift`和`_read_images`)时被调用。
```c
// Attach method lists and properties and protocols from categories to a class.
// Assumes the categories in cats are all loaded and sorted by load order, 
// oldest categories first.
static void
attachCategories(Class cls, const locstamped_category_t *cats_list, uint32_t cats_count,
                 int flags)
```
在`attachCategories`时，`category`中定义的方法，会以一个`method_list_t`的二维数组的形式，添加到`class`的`bits`中，并且是顺序添加。
也就是说，`category`确实提供了扩展`method`的可行性，并且是直接扩展方法列表。

继续看`protocol`：
```c
struct protocol_t : objc_object {
    const char *mangledName;
    struct protocol_list_t *protocols;
    method_list_t *instanceMethods;
    method_list_t *classMethods;
    method_list_t *optionalInstanceMethods;
    method_list_t *optionalClassMethods;
    property_list_t *instanceProperties;
```
`protocol`中确实定义了实例方法和类方法，并且区分了`optional`和`required`。但是通过查看源码，并没有发现`protocol`方法添加到类上的时机。
我们可以写一段程序验证一下我们的想法：
```Objective-C
@protocol MyProtocol <NSObject>

- (void)add;
+ (void)minus;

@end

@interface TestProtocolMethodList : NSObject <MyProtocol>

@end

@implementation TestProtocolMethodList

- (void)addTest {}

//- (void)add {}

+ (void)minusTest {}

//+ (void)minus {}

@end

unsigned int iCount = 0;
Method *iMethods = class_copyMethodList([[TestProtocolMethodList class] class], &iCount);
for (int n = 0; n < iCount; n++) {
    Method m = *(iMethods + n);
    SEL sel = method_getName(m);
    NSLog(@"%@", NSStringFromSelector(sel));
}
unsigned int cCount = 0;
Method *cMethods = class_copyMethodList(objc_getMetaClass([NSStringFromClass([TestProtocolMethodList class]) UTF8String]), &cCount);
for (int n = 0; n < cCount; n++) {
    Method m = *(cMethods + n);
    SEL sel = method_getName(m);
    NSLog(@"%@", NSStringFromSelector(sel));
}
```
通过打印，我们会发现:
- 在没有实现`MyProtocol`定义的方法时，日志输出的只有`TestProtocolMethodList`本身自带的方法（`addTest`和`minusTest`）
- 在实现了`MyProtocol`定义的方法后，日志输出`TestProtocolMethodList`的方法列表里，包含了所有的方法（貌似是废话）

这样印证了，只是遵循`protocol`是不会默认添加方法到方法列表的。


### protocols

### ivars

### properties