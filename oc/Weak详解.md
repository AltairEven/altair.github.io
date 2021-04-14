# Weak详解
在从MRC进化为ARC时，OC中多了两个关键词`__strong`和`__weak`，本文将详细描述`__weak`在OC中的作用原理。
## `weak/__weak`
我们在定义OC类时，有时需要定义`Property`，通常这些`Property`会使用一些修饰符，如下
```c
@interface TestProperty : NSObject

@property (nonatomic, assign) int anAssignInt;
@property (nonatomic, retain) id aRetainObj;
@property (nonatomic, strong) id astrongObj;
@property (nonatomic, copy) id aCopyObj;
@property (nonatomic, weak) id aWeakObj;
@property (nonatomic, unsafe_unretained) id anUnsafeUnretainedObj;

@end
```
这里的`assign/retain/strong/copy/weak/unsafe_unretained`都是其属性，用来修饰当前类对该`property`的引用类型。
我们先查看一下他们的属性
```c
unsigned int pCount = 0;
objc_property_t *properties = class_copyPropertyList(objc_getClass("TestProperty"), &pCount);
for (int n = 0; n < pCount; n++) {
    objc_property_t p = *(properties + n);
    NSLog(@"%s --- %s", property_getName(p), property_getAttributes(p));
}

print：
anAssignInt --- Ti,N,V_anAssignInt
aRetainObj --- T@,&,N,V_aRetainObj
astrongObj --- T@,&,N,V_astrongObj
aCopyObj --- T@,C,N,V_aCopyObj
aWeakObj --- T@,W,N,V_aWeakObj
anUnsafeUnretainedObj --- T@,N,V_anUnsafeUnretainedObj
```
对照[Property Type String](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101-SW6)和[Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)可以发现：
- 所有声明类型的`property`中，`assign`和`unsafe_unretained`修饰的`property`属性是一样的，并且都没有体现出属性描述（`assign`或`unsafe_unretained`信息）
- 使用`retain`和`strong`修饰的`property`属性描述是一样的，都用`&`表示是`retain`关系，这种关系在文档中描述为`The property is a reference to the value last assigned (retain).`
- 使用`copy`修饰的`property`的属性描述是`C`，即`The property is a copy of the value last assigned (copy).`
- 使用`weak`修饰的`property`的属性描述是`W`，即`The property is a weak reference (__weak).`
可见，使用`weak`修饰`property`，其实就是使用`__weak`修饰其对应的成员变量。

至于`weak`起得作用是什么，苹果在其文档[Property Declaration Attributes](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocProperties.html#//apple_ref/doc/uid/TP30001163-CH17-SW2)中也阐述了：
<blockquote>
    weak

    Specifies that there is a weak (non-owning) relationship to the destination object.
    If the destination object is deallocated, the property value is automatically set to nil.
    (Weak properties are not supported on OS X v10.6 and iOS 4; use assign instead.)
</blockquote>>

以及[Use Strong and Weak Declarations to Manage Ownership](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/EncapsulatingData/EncapsulatingData.html#//apple_ref/doc/uid/TP40011210-CH5-SW30)中
<blockquote>
    If you don’t want a variable to maintain a strong reference, you can declare it as __weak.

    Because a weak reference doesn’t keep an object alive, it’s possible for the referenced object to be deallocated while the reference is still in use. To avoid a dangerous dangling pointer to the memory originally occupied by the now deallocated object, a weak reference is automatically set to nil when its object is deallocated.
</blockquote>>

简单描述就是：`weak/__weak`修饰（弱引用）的变量，在释放（`deallocate`）后，会被自动置为`nil`。