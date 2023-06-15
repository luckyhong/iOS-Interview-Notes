### 1.实例对象的数据结构？

- 具体可以参看 `Runtime` 源代码，在文件 `objc-private.h` 的第 `127-232` 行。

  ```objective-c
  struct objc_object {
  	isa_t isa;
  	//...
  }
  ```

  本质上 `objc_object` 的私有属性只有一个 `isa` 指针。指向 `类对象` 的内存地址。

### 2.类对象的数据结构？

具体可以参看 `Runtime` 源代码。

> 类对象就是 `objc_class`。

```objective-c
struct objc_class : objc_object {
    // Class ISA;
    Class superclass; //父类指针
    cache_t cache;             // formerly cache pointer and vtable 方法缓存
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags 用于获取地址

    class_rw_t *data() { 
        return bits.data(); // &FAST_DATA_MASK 获取地址值
    }
```



它的结构相对丰富一些。继承自`objc_object`结构体，所以包含`isa`指针

- `isa`：指向元类
- `superClass`: 指向父类
- `Cache`: 方法的缓存列表
- `data`: 顾名思义，就是数据。是一个被封装好的 `class_rw_t` 。

### 3.元类对象的数据结构?

`元类对象` 和 `类对象` 数据结构是一致的的 ，参见第二题。

### 4.`Category` 的实现原理？

> 被添加在了 `class_rw_t` 的对应结构里。

`Category` 实际上是 `Category_t` 的结构体，在运行时，新添加的方法，都被以倒序插入到原有方法列表的最前面，所以不同的`Category`，添加了同一个方法，执行的实际上是最后一个。

拿方法列表举例，实际上是一个二维的数组。

`Category` 如果翻看源码的话就会知道实际上是一个 `_catrgory_t` 的结构体。

-- 例如我们在程序中写了一个 `Nsobject+Tools` 的分类，那么被编译为 `C++` 之后，实际上是：

```objective-c
static struct _catrgory_t _OBJC_$_CATEGORY_NSObject_$_Tools __attribute__ ((used,section),("__DATA,__objc__const"))
{
    // name
    // class
    // instance method list
    // class method list
    // protocol list
    // properties
}
```

`Category` 在刚刚编译完的时候，和原来的类是分开的，只有在程序运行起来后，通过 `Runtime` ，`Category` 和原来的类才会合并到一起。

`mememove`，`memcpy`：这俩方法是位移、复制，简单理解就是原有的方法移动到最后，根根新开辟的空间，把前面的位置留给分类，然后分类中的方法，按照倒序依次插入，可以得出的结论就就是，越晚参与编译的分类，里面的方法才是生效的那个。

### 5.如何给 `Category` 添加属性？关联对象以什么形式进行存储？

查看的是 `关联对象` 的知识点。

详细的说一下 `关联对象`。

`关联对象` 以哈希表的格式，存储在一个全局的单例中。

```objective-c
@interface NSObject (Extension)
@property (nonatomic,copy  ) NSString *name;
@end

@implementation NSObject (Extension)

- (void)setName:(NSString *)name {
    objc_setAssociatedObject(self, @selector(name), name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)name {
    return objc_getAssociatedObject(self,@selector(name));
}

@end
```

### 6.`Category` 有哪些用途？

- 给系统类添加方法、属性（需要关联对象）。
- 对某个类大量的方法，可以实现按照不同的名称归类。

### 7.`Category` 和 `Extension` 有什么区别

- `分类` 的加载在 `运行时`，`类拓展` 的加载在 `编译时`。
- `类拓展` 不能给系统的类添加方法。
- `类拓展` 只以声明的形式存在，一般存在 .m 文件中。

### 8.说一下 `Method Swizzling`? 说一下在实际开发中你在什么场景下使用过?

交换方法，可以结合 `Aspect` 的使用一起说，其实方法交换，应用于统计个页面停留时长，记录用户路径，Hook系统方法啥的有不少用途。

Method Swizzling 是一种在 Objective-C 运行时中动态交换方法实现的技术，它允许开发者在运行时修改一个类的方法实现，可以用于替换或扩展现有方法，或者在方法调用前后执行一些额外的代码。

在 Objective-C 中，每个类都有一个类对象（Class object），类对象中包含了该类的方法列表，每个方法都由一个 Method 结构体表示，Method 中包含了方法的名称、参数类型、返回值类型和指向实现代码的函数指针等信息。通过 Method Swizzling 技术，开发者可以动态修改类对象的方法列表，将某个方法的实现替换成另一个方法的实现。

在实际开发中，Method Swizzling 可以用于实现一些 AOP（面向切面编程）的功能，比如在某个方法调用前后打印日志、统计方法执行时间、检测方法是否被调用等等。另外，Method Swizzling 还可以用于解决一些框架或第三方库的 bug，或者在某些情况下改变某个方法的默认行为。

举个例子，假设我们有一个 UIViewController，我们想要在它的 viewDidAppear 方法被调用时打印一条日志，可以使用 Method Swizzling 技术实现：

```objective-c
@implementation UIViewController (Logging)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        SEL originalSelector = @selector(viewDidAppear:);
        SEL swizzledSelector = @selector(swizzled_viewDidAppear:);
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        method_exchangeImplementations(originalMethod, swizzledMethod);
    });
}

- (void)swizzled_viewDidAppear:(BOOL)animated {
    [self swizzled_viewDidAppear:animated];
    NSLog(@"viewDidAppear: %@", self);
}

@end
```

在上面的代码中，我们通过 category 扩展了 UIViewController 类，重写了 viewDidAppear 方法，并使用 Method Swizzling 技术将原有的 viewDidAppear 方法实现与我们自己的 swizzled_viewDidAppear 方法实现进行交换。这样，在每次调用原有的 viewDidAppear 方法时，实际上会调用我们自己的 swizzled_viewDidAppear 方法，从而实现了在方法调用前后打印日志的功能

### 9.如何实现动态添加方法和属性？

在 Objective-C 中，动态添加方法和属性可以使用 Objective-C Runtime 提供的函数实现。

动态添加方法的步骤如下：

1. 定义一个 C 语言函数，函数需要遵循以下格式：

```objective-c
void dynamicMethodIMP(id self, SEL _cmd) {
    // 实现代码
}
```

其中，`self` 是方法的调用者，`_cmd` 是方法的选择器。

2. 使用 `class_addMethod` 函数向类对象中添加方法，该函数的参数包括类对象、方法选择器、函数实现、方法的类型编码。

```objective-c
Class cls = [SomeClass class];
SEL selector = @selector(someMethod);
IMP implementation = (IMP) dynamicMethodIMP;
const char *types = "v@:";
class_addMethod(cls, selector, implementation, types);
```

其中，`v@:` 表示方法的返回值类型为 void，参数类型为 `id` 和 `SEL`。

3. 调用添加的方法。

```objective-c
id obj = [[SomeClass alloc] init];
[obj performSelector:@selector(someMethod)];
```

动态添加属性的步骤如下：

1. 定义属性的 getter 和 setter 方法，方法需要遵循以下格式：

```objective-c
// getter 方法
- (NSString *)propertyName {
    return objc_getAssociatedObject(self, @selector(propertyName));
}

// setter 方法
- (void)setPropertyName:(NSString *)propertyName {
    objc_setAssociatedObject(self, @selector(propertyName), propertyName, OBJC_ASSOCIATION_COPY_NONATOMIC);
}
```

其中，`objc_getAssociatedObject` 和 `objc_setAssociatedObject` 函数可以将一个对象与一个键值对关联起来，`OBJC_ASSOCIATION_COPY_NONATOMIC` 表示属性的内存管理方式。

2. 使用 `class_addMethod` 函数向类对象中添加 getter 和 setter 方法。

```objective-c
Class cls = [SomeClass class];
SEL getter = @selector(propertyName);
SEL setter = @selector(setPropertyName:);
const char *getterTypes = "@@:";
const char *setterTypes = "v@:@";
class_addMethod(cls, getter, (IMP)propertyNameGetter, getterTypes);
class_addMethod(cls, setter, (IMP)propertyNameSetter, setterTypes);
```

其中，`getterTypes` 表示 getter 方法的返回值类型为 `id`，参数类型为 `SEL`，`setterTypes` 表示 setter 方法的返回值类型为 void，参数类型为 `id` 和 `SEL`。

3. 在类对象中定义属性的内存管理方式。

```objective-c
objc_setAssociatedObject(self, @selector(propertyName), @"propertyValue", OBJC_ASSOCIATION_COPY_NONATOMIC);
```

以上是动态添加方法和属性的基本步骤，当然还需要根据实际情况进行适当的修改和补充。需要注意的是，动态添加方法和属性可能会影响代码可读性和可维护性，因此应该在必要的情况下使用，并且需要仔细评估其风险和影响。

### 10.说一下对 `isa` 指针的理解， 对象的`isa` 指针指向哪里？`isa` 指针有哪两种类型？

```
isa` 等价于 `is kind of
```

- 实例对象 `isa` 指向类对象
- 类对象指 `isa` 向元类对象
- 元类对象的 `isa` 指向元类的基类

`isa` 有两种类型

- 纯指针，指向内存地址
- `NON_POINTER_ISA`，除了内存地址，还存有一些其他信息

在Runtime源码查看isa_t是共用体。简化结构如下：

```objective-c
union isa_t 
{
    Class cls;
    uintptr_t bits;
    # if __arm64__ // arm64架构
#   define ISA_MASK        0x0000000ffffffff8ULL //用来取出33位内存地址使用（&）操作
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t nonpointer        : 1; //0：代表普通指针，1：表示优化过的，可以存储更多信息。
        uintptr_t has_assoc         : 1; //是否设置过关联对象。如果没设置过，释放会更快
        uintptr_t has_cxx_dtor      : 1; //是否有C++的析构函数
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000 内存地址值
        uintptr_t magic             : 6; //用于在调试时分辨对象是否未完成初始化
        uintptr_t weakly_referenced : 1; //是否有被弱引用指向过
        uintptr_t deallocating      : 1; //是否正在释放
        uintptr_t has_sidetable_rc  : 1; //引用计数器是否过大无法存储在ISA中。如果为1，那么引用计数会存储在一个叫做SideTable的类的属性中
        uintptr_t extra_rc          : 19; //里面存储的值是引用计数器减1

#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

# elif __x86_64__ // arm86架构,模拟器是arm86
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };

# else
#   error unknown architecture for packed isa
# endif

}
```

### 11.`Obj-C` 中的类信息存放在哪里？

类方法存储在元类。

- 对象方法、属性、成员变量、协议等存放在 Class 对象中。
- 类方法存放在 meta-class 对象中。
- 成员变量的具体指，存放在 instance 对象中。

### 12.一个 `NSObject` 对象占用多少内存空间？

结论：受限于内存分配的机制，一个 `NSObject`对象都会分配 `16Bit` 的内存空间。但是实际上在64位下，只使用了 `8bit`，在32位下，只使用了 `4bit`。

首先`NSObject`对象的本质是一个`NSObject_IMPL`结构体。我们通过以下命令将 `Objecttive-C` 转化为 `C\C++`

```
// 如果需要连接其他框架，可以使用 -framework 参数，例如 -framework UIKit
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m -o main.cpp
```

通过将`main.m`转化为`main.cpp` 文件可以看出它的结构包含一个`isa`指针：

```c
struct NSObject_IMPL {
    Class isa;
};
```

如果当前是继承自`NSObject`的`Person`类，结构如下：

```c
struct Person_IMPL {
    Class isa;
    // 自己的成员变量
    int _age;
    int _height;
};
```

or（参考上面 NSObject_IMPL 的结构）

```c++
struct Person_IMPL {
    struct NSObject_IMPL NSObject_IVARS;
    // 自己的成员变量
    int _age;
    int _height;
};
```

下面通过一个例子来验证一下以上的结论

初始化一个`NSObject`对象

```objective-c
NSObject *object = [[NSObject alloc] init];
```

导入运行时头文件 `#import <objc/runtime.h>`，利用 `class_getInstanceSize` 方法，传入实例的类，即可获取当前实例实际占用内存的大小

```objective-c
NSLog(@"object 实际占用内存大小为 %zd",class_getInstanceSize([object class]));
// 打印结果为 8
```

之后我们导入 `#import <malloc/malloc.h>`

```objective-c
NSLog(@"object 指针指向内存的大小为 %zd",malloc_size((__bridge const void *)object));
// 打印结果为 16
```

`Class_getInstanceSize`底层实现：对象在分配内存空间时，会进行内存对齐，所以在 `iOS` 中，分配内存空间都是 `16字节` 的倍数。**如果存在继承关系，则需要父类的大小**

```c++
size_t class_getInstanceSize(Class cls)
{
    if (!cls) return 0;
    return cls->alignedInstanceSize();
}
```

可以通过以下网址 ：[openSource.apple.com/tarballs](https://github.com/liberalisman/iOS-InterviewQuestion-collection/blob/master/Runtime/openSource.apple.com/tarballs) 来查看源代码。

### 13.说一下对 `class_rw_t` 的理解？

`rw`代表可读可写。

`ObjC` 类中的属性、方法还有遵循的协议等信息都保存在 `class_rw_t` 中：

```c++
// 可读可写
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro; // 指向只读的结构体,存放类初始信息

    /*
     这三个都是二位数组，是可读可写的，包含了类的初始内容、分类的内容。
     methods中，存储 method_list_t ----> method_t
     二维数组，method_list_t --> method_t
     这三个二位数组中的数据有一部分是从class_ro_t中合并过来的。
     */
    method_array_t methods; // 方法列表（类对象存放对象方法，元类对象存放类方法）
    property_array_t properties; // 属性列表
    protocol_array_t protocols; //协议列表

    Class firstSubclass;
    Class nextSiblingClass;
    
    //...
    }
```

### 14.说一下对 `class_ro_t` 的理解？

存储了当前类在编译期就已经确定的属性、方法以及遵循的协议。

```c++
struct class_ro_t {  
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
    uint32_t reserved;

    const uint8_t * ivarLayout;

    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
};
```

`baseMethodList`，`baseProtocols`，`ivars`，`baseProperties`三个都是一唯数组。

### 15.`Runtime` 消息解析

如果当前类没有对应的实例方法，系统会调用如下方法，可以选择在这个时机动态添加

```objective-c
+(BOOL)resolveInstanceMethod:(SEL)sel {

    if (sel == @selector(eat)) {
        class_addMethod([self class], sel, (IMP)vc_eat, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```

```objective-c
void vc_eat(id obj,SEL _cmd) {
    NSLog(@"这是Eat方法");
}
```

如果当前类没有对应的类方法，系统会调用如下方法，可以选择在这个时机动态添加

```objective-c
+(BOOL) resolveClassMethod:(SEL)sel {
    if(sel == @selector(newPerson)) {
        // 找到当前类的 metaClass
        Class MetaClass = objc_getMetaClass([NSStringFromClass(self) UTF8String]);
        class_addMethod(MetaClass,sel,(IMP)vc_newPerson,"v@:");
        return YES;
    }
    return [super resolveClassMethod:sel];
}
```

```objective-c
void vc_newPerson(id obj,SEL _cmd) {
    NSLog(@"当前是类方法 newPerson")；
}
```

如果在当前类，以上两个方法都没有实现，可以将消息转发给其他的类处理

```objective-c
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if(aSelector == @selector(someMethod)) {
        return [SomeClass new];
    }
       return [super forwardingTargetForSelector:aSelector];
}
```

### 16.如何运用 `Runtime` 字典转模型？

`Runtime` 遍历 `ivar_list`,结合 `KVC` 赋值。

### 17.如何运用 `Runtime` 进行模型的归解档

`Runtime` 遍历 `ivar_list`。

### 18.在 `Obj-C` 中为什么叫发消息而不叫函数调用？

在 Objective-C 中，方法调用通常被称为“发送消息”（send message），而不是“函数调用”（function call）。

这是由于在 Objective-C 中，方法的调用是通过 Objective-C Runtime 动态解析实现的，而不是像 C++ 中那样直接进行函数调用。在 Objective-C 中，每个对象都有一个 isa 指针，该指针指向该对象所属的类对象，类对象中包含了该类的方法列表，每个方法都由一个 Method 结构体表示，Method 中包含了方法的名称、参数类型、返回值类型和指向实现代码的函数指针等信息。当调用一个方法时，Objective-C Runtime 会根据方法的选择器（selector）在类对象的方法列表中查找对应的方法实现，然后执行该方法的实现代码。

因此，在 Objective-C 中，方法调用实际上是向一个对象发送一个消息，告诉这个对象要执行哪个方法。这与函数调用的静态绑定不同，而是动态绑定的。在编译时，编译器并不知道要调用哪个方法，而是在运行时根据方法的选择器动态查找对应的方法实现。

在 Objective-C 中，消息发送的语法通常采用方括号（[ ]）表示，如：

```objective-c
[myObject doSomethingWithArg:arg1 andArg:arg2];
```

这条语句实际上是向 `myObject` 对象发送了一个名为 `doSomethingWithArg:andArg:` 的消息，该消息带有两个参数 `arg1` 和 `arg2`。在运行时，Objective-C Runtime 会根据消息的选择器查找对应的方法实现并执行。

综上所述，Objective-C 中为方法调用称为“发送消息”，更符合其动态绑定的特性，强调了方法调用的动态性和灵活性。

### 19.结合 objc_msgsend 讲一下，接收消息的过程？

在 Objective-C 中，消息的接收过程一般是通过 `objc_msgSend` 函数来实现的。

`objc_msgSend` 函数是 Objective-C Runtime 中最重要的函数之一，它用于向一个对象发送消息。当调用一个对象的方法时，实际上是通过 `objc_msgSend` 函数向该对象发送了一个消息，消息中包含了要调用的方法的选择器和方法的参数。`objc_msgSend` 函数会根据对象的类信息和方法的选择器找到对应的方法实现并执行。

下面是 `objc_msgSend` 函数的简化实现：

```
id objc_msgSend(id self, SEL sel, ...) {
    IMP imp = class_getMethodImplementation(object_getClass(self), sel);
    return imp(self, sel, ...);
}
```

在上面的实现中，`self` 是消息的接收者，`sel` 是消息的选择器。`class_getMethodImplementation` 函数用于获取指定类中指定选择器的方法实现，`object_getClass` 函数用于获取对象的类对象。最后，`imp` 是方法的实现，通过调用 `imp` 函数来执行方法。

在实际运行时，`objc_msgSend` 函数会根据消息的选择器和接收者的 isa 指针查找方法实现。如果该类中没有与消息对应的方法实现，Objective-C Runtime 会递归查找该类的父类，直到找到方法的实现或者到达 NSObject 类为止。如果最终仍然找不到方法的实现，Objective-C Runtime 会调用 `doesNotRecognizeSelector` 方法抛出异常。

当找到方法的实现后，`objc_msgSend` 函数会将消息的参数传递给方法实现，并执行该方法。在方法执行的过程中，开发者可以使用 self 关键字引用消息的接收者，使用 _cmd 关键字引用消息的选择器。

需要注意的是，在 ARC 下，编译器会自动为我们插入 `objc_retainAutoreleasedReturnValue` 函数来管理返回值的内存，因此不需要手动管理返回值的内存。

综上所述，Objective-C 中的消息接收过程是通过 `objc_msgSend` 函数实现的，该函数根据消息的选择器和接收者的 isa 指针查找方法实现并执行，实现了 Objective-C 的动态绑定特性。

### 20.说一下 `Runtime` 的方法缓存？存储的形式、数据结构以及查找的过程？

`cache_t`增量扩展的哈希表结构。哈希表内部存储的 `bucket_t`。

`bucket_t` 中存储的是 `SEL` 和 `IMP`的键值对。

- 如果是有序方法列表，采用二分查找
- 如果是无序方法列表，直接遍历查找

###### cache_t结构体

```c++
// 缓存曾经调用过的方法，提高查找速率
struct cache_t {
    struct bucket_t *_buckets; // 散列表
    mask_t _mask; //散列表的长度 - 1
    mask_t _occupied; // 已经缓存的方法数量，散列表的长度使大于已经缓存的数量的。
    //...
}
```

```c++
struct bucket_t {
    cache_key_t _key; //SEL作为Key @selector()
    IMP _imp; // 函数的内存地址
	//...
}
```

散列表查找过程，在`objc-cache.mm`文件中

```c++
// 查询散列表，k
bucket_t * cache_t::find(cache_key_t k, id receiver)
{
    assert(k != 0); // 断言

    bucket_t *b = buckets(); // 获取散列表
    mask_t m = mask(); // 散列表长度 - 1
    mask_t begin = cache_hash(k, m); // & 操作
    mask_t i = begin; // 索引值
    do {
        if (b[i].key() == 0  ||  b[i].key() == k) {
            return &b[i];
        }
    } while ((i = cache_next(i, m)) != begin);
    // i 的值最大等于mask,最小等于0。

    // hack
    Class cls = (Class)((uintptr_t)this - offsetof(objc_class, cache));
    cache_t::bad_cache(receiver, (SEL)k, cls);
}
```

上面是查询散列表函数，其中`cache_hash(k, m)`是静态内联方法，将传入的`key`和`mask`进行`&`操作返回`uint32_t`索引值。`do-while`循环查找过程，当发生冲突`cache_next`方法将索引值减1。

### 21.是否了解 `Type Encoding`?

在 Objective-C 中，每个方法都有一个类型编码（Type Encoding），用于描述方法的返回值类型和参数类型。Type Encoding 是一个字符串，其中包含了多个字符和符号，每个字符和符号都代表了一个类型或类型修饰符。例如，`"v@:"` 表示方法的返回值类型为 void，参数类型分别为 id 和 SEL。

总之，Type Encoding 是 Objective-C Runtime 中用于描述类型信息的一种编码方式，它能够准确地描述方法的返回值类型和参数类型，是 Objective-C 动态特性的重要基础之一。

### 22.`Objective-C` 如何实现多重继承？

`Objective-C` 只支持多层继承，不支持多重继承。想实现的话，可以使用协议。

在 Objective-C 中，由于单继承的限制，无法直接实现多重继承。不过，Objective-C 提供了一些替代方案，可以实现类似多重继承的功能，包括组合（Composition）和协议（Protocol）。

1. 组合

组合是一种实现多重继承的常见方式。在 Objective-C 中，可以通过将一个类嵌入到另一个类中来实现组合。这个嵌入的类称为子对象（Subobject），可以在包含它的类中使用子对象的所有方法和属性。

例如，假设有两个类 A 和 B，它们都有一些方法和属性。要实现一个类 C，它包含了 A 和 B 的所有方法和属性，可以定义一个类 CC，将 A 和 B 嵌入到 CC 中。这样，CC 就可以通过调用 A 和 B 的方法来实现多重继承的功能。

2. 协议

协议是 Objective-C 中另一种实现多重继承的方式。协议是一组方法声明，定义了一组方法的名称、参数和返回值类型。类可以实现一个或多个协议，从而获得这些协议中所定义的方法。

例如，假设有两个类 A 和 B，它们都有一些方法和属性。要实现一个类 C，它包含了 A 和 B 的所有方法和属性，可以定义两个协议 PA 和 PB，分别包含 A 和 B 的方法声明。然后，让 C 实现 PA 和 PB 协议，就可以获得 A 和 B 的所有方法和属性。

需要注意的是，在使用组合和协议时，需要考虑名称冲突的问题。如果两个类中有相同的方法或属性，需要在组合或协议中进行适当的处理，以避免冲突。

综上所述，虽然 Objective-C 中无法直接实现多重继承，但可以通过组合和协议等方式来实现类似的功能。开发者可以根据具体的情况选择合适的方式来实现多重继承的需求。

下面是一个使用组合实现多重继承的示例：

假设有两个类 Animal 和 Flyable，代码如下：

```objective-c
// Animal.h
@interface Animal : NSObject

@property (nonatomic, strong) NSString *name;

- (void)eat;

@end


// Flyable.h
@protocol Flyable <NSObject>

- (void)fly;

@end
```

Animal 类定义了一个属性 name 和一个方法 eat，Flyable 协议定义了一个方法 fly。

现在我们要定义一个类 Bird，它既可以继承 Animal 类的属性和方法，又可以实现 Flyable 协议的方法。可以使用组合来实现：

```objective-c
// Bird.h
#import "Animal.h"
#import "Flyable.h"

@interface Bird : NSObject

@property (nonatomic, strong) Animal *animal;
@property (nonatomic, weak) id<Flyable> flyable;

- (void)eat;
- (void)fly;

@end


// Bird.m
@implementation Bird

- (instancetype)init {
    self = [super init];
    if (self) {
        _animal = [[Animal alloc] init];
    }
    return self;
}

- (void)eat {
    [self.animal eat];
}

- (void)fly {
    [self.flyable fly];
}

@end
```

在 Bird 类中，我们定义了一个 Animal 类型的属性 animal，用于继承 Animal 类的属性和方法。同时，我们还定义了一个 id<Flyable> 类型的属性 flyable，用于实现 Flyable 协议的方法。在 Bird 类的实现中，我们通过调用 animal 对象的 eat 方法来实现继承 Animal 类的方法，通过调用 flyable 对象的 fly 方法来实现实现 Flyable 协议的方法。

这样，Bird 类就可以同时继承 Animal 类和实现 Flyable 协议的方法了。使用组合的方式，我们可以很方便地实现多重继承的功能。

下面是一个使用协议实现多重继承的示例：

假设有两个类 Animal 和 Flyable，代码如下：

```objective-c
// Animal.h
@interface Animal : NSObject

@property (nonatomic, strong) NSString *name;

- (void)eat;

@end


// Flyable.h
@protocol Flyable <NSObject>

- (void)fly;

@end
```

Animal 类定义了一个属性 name 和一个方法 eat，Flyable 协议定义了一个方法 fly。

现在我们要定义一个类 Bird，它既可以继承 Animal 类的属性和方法，又可以实现 Flyable 协议的方法。可以使用协议来实现：

```objective-c
// Bird.h
#import "Animal.h"
#import "Flyable.h"

@interface Bird : Animal <Flyable>

@end


// Bird.m
@implementation Bird

- (void)fly {
    NSLog(@"%@ can fly", self.name);
}

@end
```

在 Bird 类中，我们通过继承 Animal 类，并实现 Flyable 协议中定义的 fly 方法，来实现多重继承的功能。Bird 类中不需要再定义 Animal 类的属性或者重复实现 Animal 类的方法，而是直接继承 Animal 类的属性和方法。同时，通过实现 Flyable 协议的方法 fly，也获得了实现 Flyable 协议的方法的能力。

这样，Bird 类就可以同时继承 Animal 类和实现 Flyable 协议的方法了。使用协议的方式，我们可以很方便地实现多重继承的功能。需要注意的是，协议只能定义方法，不能定义属性，因此在使用协议实现多重继承时，需要注意属性的继承问题。

### 23.`Category` 可不可以添加实例对象？为什么？

答案是不可以。 Category 的结构体内部没有容纳 Ivar 的数据结构。

有很详细的分析，具体可以看[这篇文章](https://www.jianshu.com/p/dcc3284b65bf)

### 24.`Obj-c`对象、类的本质是通过什么数据结构实现的？

结构体。譬如一个最常见的 `Nsobject` 对象。将其编译为 `C++` 代码后，实际上就是一个 `Nsobject_Impl` 结构体。

### 25.`Category` 在编译过后，是在什么时机与原有的类合并到一起的？

在编译过程中，Category 是独立编译的，它会生成一个单独的 Mach-O 文件。当程序运行时，Objective-C Runtime 会在加载 Mach-O 文件时动态地将 Category 合并到原有的类中。

具体来说，Objective-C Runtime 在加载类的 Mach-O 文件时，会遍历该文件中的所有 Category，并将 Category 中新增的方法和属性添加到原有的类中。如果 Category 中的方法和原有类中已有的方法发生了冲突，则 Category 中的方法会覆盖原有类中的方法。需要注意的是，Category 中不能新增实例变量，因为这会破坏原有类的布局。

在 Category 合并到原有类之后，程序就可以像使用原有类一样使用 Category 中新增的方法和属性了。

需要注意的是，Category 的加载顺序是不确定的。如果存在多个 Category，它们的加载顺序是不确定的，因此如果多个 Category 中有相同的方法或属性，可能会出现覆盖的情况。为了避免这种情况，开发者应该避免在多个 Category 中定义相同名称的方法或属性。

另外，如果在使用 Category 的过程中出现了编译错误或者链接错误，可以尝试清理 Xcode 缓存，重新编译。有时候缓存可能会导致问题，清理缓存可以解决这些问题。

综上所述，在编译过程中，Category 是独立编译的，它会生成一个单独的 Mach-O 文件。在程序运行时，Objective-C Runtime 会动态地将 Category 合并到原有的类中，以实现新增方法和属性的功能。