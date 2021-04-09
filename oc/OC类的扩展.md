# OC类的扩展

## 什么是对于OC类的扩展？
### OC类是什么？
学习过Objective-C(简称OC)语言的同学都知道，OC中绝大部分类都是继承自`NSObject`，而什么是`NSObject`呢？
我们看`NSObject.h`，会发现`NSObject`实际上是一个遵循了`NSObject Protocol`，包含一个`Class`属性并提供了若干方法的类。
- 首先它遵循了`NSObject`协议，这使得一个`NSObject`对象可以响应`NSObject Protocol`方法的调用；
- 其次它包含了一个类型为`Class`，变量名为`isa`的成员变量，但是这个`isa`被`OBJC_ISA_AVAILABILITY`修饰了；
- 然后它定义了一些基础的方法，有些可以供开发者调用，有些只在特定编译环境下可以调用。

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

(Tips: 为什么苹果要这样欺骗我们？我也看到了一些讨论，诸如[why-objc-class-has-different-definition-between-runtime-h-and-objc-runtime-new-h](https://stackoverflow.com/questions/60518472/why-objc-class-has-different-definition-between-runtime-h-and-objc-runtime-new-h))

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
```markdown
至此，我们得出结论。我们可以扩展的项有：
1、methods
2、protocols
3、ivars
4、properties
```
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
首先`class_ro_t`中的`baseMethodList`，即基本方法列表，几乎是定死的。它只在编译器设置了`ptrauth`（[pointer_authentication](https://developer.apple.com/documentation/security/preparing_your_app_to_work_with_pointer_authentication?language=objc)）时是动态生成的，但是依然是`不可修改`的。

通过搜索源码，我们发现除了类本身的`baseMethodList`之外，`method_list_t`只出现在`protocol_t`和`category_t`这两个结构中，那是不是可以通过添加`protocol`和`category`，就能够修改`method_list_t`的值呢？
##### `Category`
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
在`attachCategories`时，`category`中定义的方法，会以一个`method_list_t`的二维数组的形式，通过`prepareMethodLists`重命名和排序（即`fixupMethodList`，具体作用就是一些编译优化，可以类似参考[what-is-objc-msgsend-fixup-exactly](https://stackoverflow.com/questions/12228350/what-is-objc-msgsend-fixup-exactly) ），然后添加到`class`的`bits`中，并且是顺序添加。
也就是说，`category`确实提供了扩展`method`的可行性，并且是直接扩展方法列表。
##### `Protocol`
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
```c
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
##### `addMethod`
除了`Category`和`Protocol`之外，我们还在`addMethod`函数中找到了`method_list_t`的身影。而`addMethod`函数除了在`MethodizeClass`（`realizeClassWithoutSwift`）时会被调用之外，还在`class_addMethod`和`class_replaceMethod`是被调用，也就是我们熟知的“Runtime API”。

也就是说，使用`class_addMethod`和`class_replaceMethod`这两个“Runtime API”，也可以实现对类方法的扩展。

```markdown
Note:
- 经过查询，`method_array_t`并没有被修改，而只是用于一些值的传递。
```
### protocols
回过头来看`protocol_list_t`和`protocol_array_t`，首先我们发现与`method_array_t`相似，`protocol_array_t`也是只读的，所以我们主要看下`protocol_list_t`。

经过搜索，可以得知分别在`attachCategories`、`methodizeClass`、`protocol_addProtocol`和`class_addProtocol`时，`protocol_list_t`被合并到类中。此时，`Protocol`中定义的方法和属性，被一并合并到类的`protocol_list_t`中。

### ivars
接着来说`ivars`，也就是“成员变量”。
#### `ivar_list_t`和`ivar_t`
`class_ro_t`中定义的变量列表`ivar_list_t`其实就是`ivar_t`链表，我们看下`ivar_t`是啥：
```c
struct ivar_t {
#if __x86_64__
    // *offset was originally 64-bit on some x86_64 platforms.
    // We read and write only 32 bits of it.
    // Some metadata provides all 64 bits. This is harmless for unsigned 
    // little-endian values.
    // Some code uses all 64 bits. class_addIvar() over-allocates the 
    // offset for their benefit.
#endif
    int32_t *offset;
    const char *name;
    const char *type;
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw;
    uint32_t size;

    uint32_t alignment() const {
        if (alignment_raw == ~(uint32_t)0) return 1U << WORD_SHIFT;
        return 1 << alignment_raw;
    }
};
```
可以很清楚的看到，`ivar_t`其实就是定义了 变量名、类型、对齐方式、内存大小和偏移量的结构体。系统在分配类实例的内存空间时，会通过这些定义来分配内存。
#### 扩展成员变量
想要搞清楚怎么扩展成员变量，我们就得看看`ivar_list_t`在哪些时候被改变了，还是通过搜索大法来查一查···

通过搜索，我们可以看到只有两处，`ivar_list_t`被改变了，分别是`class_addIvar`和`free_class`，`free_class`很明显是在类释放的时候调用的，那么可以把目光聚焦在`class_addIvar`上。
源码如下：
```c
    ivar_list_t *oldlist, *newlist;
    if ((oldlist = (ivar_list_t *)cls->data()->ro()->ivars)) {
        size_t oldsize = oldlist->byteSize();
        newlist = (ivar_list_t *)calloc(oldsize + oldlist->entsize(), 1);
        memcpy(newlist, oldlist, oldsize);
        free(oldlist);
    } else {
        newlist = (ivar_list_t *)calloc(ivar_list_t::byteSize(sizeof(ivar_t), 1), 1);
        newlist->entsizeAndFlags = (uint32_t)sizeof(ivar_t);
    }

    uint32_t offset = cls->unalignedInstanceSize();
    uint32_t alignMask = (1<<alignment)-1;
    offset = (offset + alignMask) & ~alignMask;

    ivar_t& ivar = newlist->get(newlist->count++);
#if __x86_64__
    // Deliberately over-allocate the ivar offset variable. 
    // Use calloc() to clear all 64 bits. See the note in struct ivar_t.
    ivar.offset = (int32_t *)(int64_t *)calloc(sizeof(int64_t), 1);
#else
    ivar.offset = (int32_t *)malloc(sizeof(int32_t));
#endif
    *ivar.offset = offset;
    ivar.name = name ? strdupIfMutable(name) : nil;
    ivar.type = strdupIfMutable(type);
    ivar.alignment_raw = alignment;
    ivar.size = (uint32_t)size;

    ro_w->ivars = newlist;
```
大意就是，在存在老的成员变量时合并，不存在老的成员变量时直接创建，然后赋值给类的成员变量链表。

综上所述，想要扩展类的成员变量，除了在定义`interface`或者`extesion`外，只能通过“Runtime API”`class_addIvar`来额外添加。

### properties
剩下最后一个可扩展项`Property`，字面意思“财产”，我们在日常工作中，一般会称呼它为“属性”。
在类的结构中，统一有`property_list_t`和`property_array_t`两种类型的“属性列表”定义，没有猜错的话`property_array_t`应该还是不会被改动而只作为值传递，事实也确实如此···
#### `property_list_t`和`property_t`
`property_list_t`是`property_t`的链表结构，`property_t`定义如下：
```c
struct property_t {
    const char *name;
    const char *attributes;
};
```
它定义了一个“名称”和一个“属性”，并且通过`attributes`这个命名，可以猜测这可能是一个“属性列表”。

我们继续查看`attributes`，通过`copyPropertyAttributeList`方法，我们可以得知这个变量实际上是`objc_property_attribute_t *`类型。
也就是一个数组，内部包含了若干个`objc_property_attribute_t`结构：
```c
/// Defines a property attribute
typedef struct {
    const char * _Nonnull name;           /**< The name of the attribute */
    const char * _Nonnull value;          /**< The value of the attribute (usually empty) */
} objc_property_attribute_t;
```
“属性名称”和“属性值”分别是什么意思呢？我没有找到确切的文档描述，但是通过以下的代码可以打印出来看看，有兴趣的同学可以试试看。
```c
 int pCount = 0;
objc_property_t *properties = class_copyPropertyList([[TestClass class] class], &pCount);
for (int n = 0; n < pCount; n++) {
    objc_property_t p = *(properties + n);
    NSLog(@"%s --- %s", property_getName(p), property_getAttributes(p));
}
```
最后打印出来的是当前类所有的`properties`的名称和属性，名称就是我们定义的名称，属性是一些奇怪的字符，这些字符遵循如下规则：
```c
/*
  Property attribute string format:

  - Comma-separated name-value pairs. 
  - Name and value may not contain ,
  - Name may not contain "
  - Value may be empty
  - Name is single char, value follows
  - OR Name is double-quoted string of 2+ chars, value follows

  Grammar:
    attribute-string: \0
    attribute-string: name-value-pair (',' name-value-pair)*
    name-value-pair:  unquoted-name optional-value
    name-value-pair:  quoted-name optional-value
    unquoted-name:    [^",]
    quoted-name:      '"' [^",]{2,} '"'
    optional-value:   [^,]*

*/
```
关于属性的定义，具体可以参考：[Property Type String](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101-SW6)
#### 扩展Property
扩展`Property`，其实就是修改`property_list_t`。
`property_list_t`分别在`attachCategories`、`methodizeClass`、`class_addProperty`、`class_replaceProperty`和`protocol_addProperty`时被合并到类的properties中。

也就是说，我们可以通过`Category`、`Runtime API`或者`Protocol`来额外添加`Property`。

### Something else
- 我们都知道OC类也是C++的结构体，因此OC类也可以通过继承来扩展以上的四种扩展项；
- 另外，OC的类定义时，支持使用`extension`，这也为其提供了扩展能力。

```markdown
综上所述，我们已知可以提供的扩展方案包括：
- Category
- Protocol
- Runtime API
- Inherit
- Extension
```

## 我们需要注意些什么？
虽然OC为我们提供了诸多的手段，来给一个类提供扩展，但是实际操作时，我们还是有很多需要注意的点，下面我们就分别来介绍一下。
### Category
通过之前的描述，我们知道添加`Category`，可以为一个类提供除了`ivars`以外的所有扩展，可谓非常强大。
但是由于一些编译限制，以及运行时逻辑，导致我们使用这个方案时，需要注意以下几个问题：
- 方法重名
    - 我们在定义`Category`时，可以任意声明方法，难免会出现重名现象。出现重名后，由于OC的方法调用机制，会导致我们的代码可能无法按照预期执行。因此，我们必须尽量避免重名方法的出现。
    - 如果定义了重名方法，那么后定义的方法会被执行（如果是在不同文件中，则后编译的方法会被执行）；原类中的重名方法始终不会被调用到。
- 定义`Property`
    - 我们定义的`property`会被正确加到类的`properties`中，但是由于category无法提供`ivars`扩展，导致编译器无法为类创建对应的成员变量，所以我们需要为我们定义的`property`实现`getter`和`setter`方法。
    - 我们在定义`property`时，可以通过`objc_setAssociatedObject`添加关联属性，关联属性需要设置`objc_AssociationPolicy`，但是它不包括`weak`。
- 导入
    - 在`.m`(或`.mm`)文件中使用`Category`时，我们需要`import`对应的头文件
    - 如果`Category`被打包到`.a`或者`.framework`中，默认该`Category`并不会被编译，所以需要一些额外处理（参考[这里](https://blog.csdn.net/skylin19840101/article/details/51821932/)）

### Protocol
我们知道`Protocol`不会自动添加方法或者`Property`到类中，所以我们更多需要注意的，是如何定义和使用一个`Protocol`。
- 定义`Property`
    - `Protocol`中声明的`Property`，是在由遵循它的类定义时才生效，所以如果是`Category`遵循了该`Protocol`，需要自己实现`getter`和`setter`方法
- 使用
    - 防止出现循环引用
    - 由于`@optional`的声明在未实现时，不会引起编译错误，所以我们调用`Protocol`声明的`方法`（或`property`）时，需要先使用`respondsToSelector`判断调用方是否实现了该`方法`（或`property`的`getter`和`setter`）

### Runtime API
`Runtime API`为我们提供了非常强大的操作类内部结构的工具，但是我们在使用的时候，依然要小心谨慎，特别是进行方法替换的时候，需要明确目的。
### Inherit
继承的方式，可以完成几乎所有的扩展，但是继承之后就是一个新的类了，它会拥有一份新的结构。
继承时需要注意的，主要是变量权限和方法重写问题。
### Extension
`Extension`可以理解为在原类上继续做扩展，所以没有太多需要注意的，但是苹果还是列出了一些可能出现的问题，希望我们关注：
- 权限
    - 我们可以在`原类`和`Extension`中对于同一个`Property`定义不同的读写权限，所以如果我们在原类中没有定义`write`权限，那么我们必须要自己实现该`Property`的`setter`方法。
    - 原类的方法和`Property`默认是`public`的，而`Extension`中定义的可以认为是“`private`”的，但是如果你把`Extension`定义再独立的`.h`文件中，他们也会是`public`的。

# 结尾
奉上苹果的官方建议：[Customizing Existing Classes](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/CustomizingExistingClasses/CustomizingExistingClasses.html#//apple_ref/doc/uid/TP40011210-CH6-SW1)
