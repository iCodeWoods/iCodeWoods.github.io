---
title: Runtime 温故而知新
categories:
  - iOS
date: 2019-05-16 15:51:00
tags: Runtime
---
# Runtime

本文基于 runtime 的最新版本 750，源码可以[点这里](https://opensource.apple.com/tarballs/objc4/)下载，可编译版本可以[点这里](https://github.com/RetVal/objc-runtime)下载。

<!-- more -->

## + load

```objective-c
// main.m
#import <Foundation/Foundation.h>

@interface XXObject : NSObject
@end

@implementation XXObject

+ (void)load {
	NSLog(@"XXObject load");
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"Hello, World!");
    }
    return 0;
}
```

通过断点，我们可以看到如下调用栈：

![Runtime_Breakpoint_Load](Runtime_Breakpoint_Load.png)

### load_images

```c++
// objc-runtime-new.mm
void
load_images(const char *path __unused, const struct mach_header *mh)
{
    // Return without taking locks if there are no +load methods here.
    if (!hasLoadMethods((const headerType *)mh)) return;

    recursive_mutex_locker_t lock(loadMethodLock);

    // Discover load methods
    {
        mutex_locker_t lock2(runtimeLock);
        prepare_load_methods((const headerType *)mh);
    }

    // Call +load methods (without runtimeLock - re-entrant)
    call_load_methods();
}
```

代码非常清晰：

1. `hasLoadMethods`：通过 `mach_header` 判断镜像中是否有 load 方法，如果没有直接返回
2. `prepare_load_methods`：准备好 load 方法
3. `call_load_methods`：调用 load 方法

> PS：mach_header 是一个结构体，里面记录了镜像的一些信息。headerType 在非 64 位下就是 mach_header，在 64 位下是 mach_header_64

那么 `load_images` 方法会加载哪些镜像呢？我们在 `hasLoadMethods` 处增加一个断点，输出 `path` 以及是否有 `load` 方法，截取其中一部分输出如下:

```c
load_images... path: "/usr/lib/system/introspection/libdispatch.dylib"
true

load_images... path: "/usr/lib/system/libsystem_trace.dylib"
true

load_images... path: "/usr/lib/system/libxpc.dylib"
true

load_images... path: "/System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation"
true

load_images... path: "/System/Library/Frameworks/Security.framework/Versions/A/Security"
false

load_images... path: "/usr/lib/libnetwork.dylib"
false

load_images... path: "/System/Library/Frameworks/CFNetwork.framework/Versions/A/CFNetwork"
false
    
// ...省略
```

下面我们就来一步步分析 load_images 所干的这三件事。

> 扩展阅读：Mach-O包含以下几种文件类型
>
> - Executable：应用的主要二进制
> - Dylib：动态链接库
> - Bundle：不能被链接，只能在运行时使用dlopen加载
> - Image：包含 Executable、Dylib 和 Bundle
> - Framework：包含Dylib、资源文件和头文件的文件夹

#### hasLoadMethods

第一步，如何判断一个镜像中是否有 `load` 方法？其实就是查找该镜像中是否有“非懒加载的类”（NonlazyClass）或“非懒加载的分类”（NonlazyCategory）。结合这个方法的方法名，就很容易这两个概念了——有 `+ load` 方法的类/分类就是“非懒加载的”，没有 `+ load` 方法的类/分类就是懒加载的。

```c++
// objc-runtime-new.mm
// Quick scan for +load methods that doesn't take a lock.
bool hasLoadMethods(const headerType *mhdr)
{
    size_t count;
    if (_getObjc2NonlazyClassList(mhdr, &count)  &&  count > 0) return true;
    if (_getObjc2NonlazyCategoryList(mhdr, &count)  &&  count > 0) return true;
    return false;
}
```

##### _getObjc2NonlazyClassList

那么如何获取镜像中的 NonlazyClassList/NonlazyCategoryList 呢？我们跳转到 `_getObjc2NonlazyClassList` 的定义：

```c++
// object-file.mm
#define GETSECT(name, type, sectname)                                   \
    type *name(const headerType *mhdr, size_t *outCount) {              \
        return getDataSection<type>(mhdr, sectname, nil, outCount);     \
    }                                                                   \
    type *name(const header_info *hi, size_t *outCount) {               \
        return getDataSection<type>(hi->mhdr(), sectname, nil, outCount); \
    }

//      function name                 content type     section name
GETSECT(_getObjc2SelectorRefs,        SEL,             "__objc_selrefs"); 
GETSECT(_getObjc2MessageRefs,         message_ref_t,   "__objc_msgrefs"); 
GETSECT(_getObjc2ClassRefs,           Class,           "__objc_classrefs");
GETSECT(_getObjc2SuperRefs,           Class,           "__objc_superrefs");
GETSECT(_getObjc2ClassList,           classref_t,      "__objc_classlist");
GETSECT(_getObjc2NonlazyClassList,    classref_t,      "__objc_nlclslist");
GETSECT(_getObjc2CategoryList,        category_t *,    "__objc_catlist");
GETSECT(_getObjc2NonlazyCategoryList, category_t *,    "__objc_nlcatlist");
GETSECT(_getObjc2ProtocolList,        protocol_t *,    "__objc_protolist");
GETSECT(_getObjc2ProtocolRefs,        protocol_t *,    "__objc_protorefs");
GETSECT(getLibobjcInitializers,       UnsignedInitializer, "__objc_init_func");
```

乍一看有点懵，其实很简单，就是 🍎 工程师为了 ~~偷懒~~ 简洁，定义了一个宏来批量实现方法，好比一个模板。

举个🌰，对于 `_getObjc2NonlazyClassList` 来说，其实这里是实现了两个方法名一样，但参数不同的方法：

```c++
classref_t *_getObjc2NonlazyClassList(const headerType *mhdr, size_t *outCount) {
    return getDataSection<classref_t>(mhdr, "__objc_nlclslist", nil, outCount);
}
classref_t *_getObjc2NonlazyClassList(const header_info *hi, size_t *outCount) {
    return getDataSection<classref_t>(hi->mhdr, "__objc_nlclslist", nil, outCount);
}
```

>  其实我第一眼看这段代码的时候，有一个疑问：为什么这里可以定义两个方法名一样、但参数不一样的方法呢？要知道在 OC 里，这样可是会报错的。过了一会我才突然想明白，这是 .mm 文件，C++ 是可以这样干的啊！差点被自己蠢哭😂

##### getDataSection

ok，现在只剩下 `getDataSection` 这个方法了：通过 headerType，获取镜像的指定 segment 的指定 section 中的数据。

```c++
// object-file.mm
// Look for a __DATA or __DATA_CONST or __DATA_DIRTY section 
// with the given name that stores an array of T.
template <typename T>
T* getDataSection(const headerType *mhdr, const char *sectname, 
                  size_t *outBytes, size_t *outCount)
{
    unsigned long byteCount = 0;
    T* data = (T*)getsectiondata(mhdr, "__DATA", sectname, &byteCount);
    if (!data) {
        data = (T*)getsectiondata(mhdr, "__DATA_CONST", sectname, &byteCount);
    }
    if (!data) {
        data = (T*)getsectiondata(mhdr, "__DATA_DIRTY", sectname, &byteCount);
    }
    if (outBytes) *outBytes = byteCount;
    if (outCount) *outCount = byteCount / sizeof(T);
    return data;
}
```

还是以 `_getObjc2NonlazyClassList` 为🌰，会先查找 `__DATA` 段中的 `__objc_nlclslist` 区块，如果找不到再找 `__DATA_CONST` 段中的 `__objc_nlclslist` 区块，以此类推。如果都找不到的话，就说明这个镜像中没有 `NonlazyClass`，也就是没有 `+ load` 方法。

为了验证，我们可以在 demo 工程的 Products 目录下，找到工程的编译产物——一个可执行文件（debug-objc）

![Runtime_Product_Exec](Runtime_Product_Exec.png)

然后我们将其拖入 [MachOView](https://sourceforge.net/projects/machoview/) 中查看，在 `__DATA` 段的 `__objc_nlclslist` section 中确实只有一个元素，其地址为 `0000000100001138`（因为此处是大端序，我们将其转化为小端序）

![Runtime_MachOView_NonLazyClass](Runtime_MachOView_NonLazyClass.png)

然后我们将可执行文件拖入 [Hopper](https://www.hopperapp.com/)，找到 `0000000100001138` 地址上的数据，确实是我们所写的 `XXObject`：

![Runtime_Hopper_XXObject](Runtime_Hopper_XXObject.png)

#### prepare_load_methods

这个函数的作用就是提前将需要调用 `load` 方法的类和分类分别加入对应的列表中，以供接下来的调用（后面会详述）。

```c++
// objc-runtime-new.mm
void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;

    runtimeLock.assertLocked();
	
    classref_t *classlist = _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        schedule_class_load(remapClass(classlist[i]));
    }
	
    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat);
    }
}
```

我们先看类的部分（为了方便快速理解大致逻辑、避免混乱，我们先暂时忽略 `remapClass`，后面会详述）。

##### schedule_class_load

`_getObjc2NonlazyClassList` 上文我们已经解释过了。获取当前镜像的所有非懒加载的类，然后进行遍历，调用 `schedule_class_load`：

```c++
// objc-runtime-new.mm
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    schedule_class_load(cls->superclass);

    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}
```

1. 通过 `RW_LOADED` 标志位判断类是否已经加载过，如果已经加载过则直接返回，保证**每个类只会加载一次**
2. 递归调用 `schedule_class_load` 对父类进行处理，保证**父类在子类之前**（这里只是"prepare"，不是调用，不过真正调用的时候父类也一定是在子类之前的，后面会详述）
3. 将类和类的 load 方法保存至一个全局列表（`loadable_classes`）中
4. 将类的 `RW_LOADED` 标志位置为 1

#####add_class_to_loadable_list

```C++
// objc-loadmethod.mm
void add_class_to_loadable_list(Class cls)
{
    IMP method;

    loadMethodLock.assertLocked();

    method = cls->getLoadMethod();
    if (!method) return;  // Don't bother if cls has no +load method
    
    if (PrintLoading) {
        _objc_inform("LOAD: class '%s' scheduled for +load", 
                     cls->nameForLogging());
    }
    
    if (loadable_classes_used == loadable_classes_allocated) {
        loadable_classes_allocated = loadable_classes_allocated*2 + 16;
        loadable_classes = (struct loadable_class *)
            realloc(loadable_classes,
                              loadable_classes_allocated *
                              sizeof(struct loadable_class));
    }
    
    loadable_classes[loadable_classes_used].cls = cls;
    loadable_classes[loadable_classes_used].method = method;
    loadable_classes_used++;
}
```

注意这里通过 `loadable_classes_used` 记录其个数，`loadable_classes` 列表的大小是动态创建的，当列表长度不够用时会重新申请内存。（我想这就是为什么如果程序中 `+ load` 方法太多时，会影响启动性能的原因之一吧。当然影响因素很多，后面我们还会详述）

> 上面的方法中，🍎 工程师还非常 ~~逗比~~ 有趣的加了一句注释：“如果类没有load方法，请不要打扰”。😂

##### add_category_to_loadable_list

分类的处理和类的处理差不多：将分类和分类的 load 方法保存至另一个全局列表 `loadable_categories` 中，通过 `loadable_categories_used` 记录其个数。我们就不再赘述了。

##### remapClass

上面我们说为了防止混乱而暂时忽略了一个方法 `remapClass`。其实通过 `_getObjc2NonlazyClassList` 获取的是 `classref_t`，通过注释我们可以看到它其实是一个“未映射”的 Class：

```c++
// classref_t is unremapped class_t*
typedef struct classref * classref_t;
```

而 `remapClass` 的作用就是将 `classref_t` 进行重映射，拿到其所对应的 Class：

```c++
// objc-runtime-new.mm
static Class remapClass(classref_t cls)
{
    return remapClass((Class)cls);
}

static Class remapClass(Class cls)
{
    runtimeLock.assertLocked();

    Class c2;

    if (!cls) return nil;

    NXMapTable *map = remappedClasses(NO);
    if (!map  ||  NXMapMember(map, cls, (void**)&c2) == NX_MAPNOTAKEY) {
        return cls;
    } else {
        return c2;
    }
}
```

从镜像中获取到的未映射的 `classref_t` 与其所对应的真正的 Class 是作为一对 key-value 存储在一个映射表里的，其中 `classref_t` 是 key，对应的 Class 是 value。

不过看到这我们还是不知道，这个映射表是哪来的？后面我们会详述。

下面我们就进入第三步：`call_load_methods`

#### call_load_methods

do while 中会先调用 `call_class_loads`，再调用 call_category_loads，这就保证了**类的 `+ load` 会在分类之前调用**。（还记得 `loadable_classes_used` 吗，我们上文提过哦。）

```c++
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```

网上有些博客说，我们重载 `+ load` 方法时需要手动添加 @autoreleasepool，甚至包括[Mike Ash 的博客](https://www.mikeash.com/pyblog/friday-qa-2009-05-22-objective-c-class-loading-and-initialization.html)也是这么说的。

但从这段源码我们可以看到，在 `runtime` 调用`+ load` 方法的前后已经加了 `objc_autoreleasePoolPush()` 和 `objc_autoreleasePoolPop()`，所以我们不需要再手动添加了。大抵是因为他们所写的博客太老了，之后🍎做了更新吧。

##### call_class_loads

`call_class_loads` 就是将我们上文所说的 `loadable_classes` 列表中的类一个一个拿出来，然后调用其 `+ load` 方法。因为之前写入 `loadable_classes` 列表时，是先写入的父类，后写入的子类，所以这里**父类的 load 会先于子类调用**。

```c++
static void call_class_loads(void)
{
    int i;
    
    // Detach current loadable list.
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    
    // Call all +loads for the detached list.
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        (*load_method)(cls, SEL_load);
    }
    
    // Destroy the detached list.
    if (classes) free(classes);
}
```

核心在于这一句，通过函数指针的方式执行 `classes[i].method`——也就是 `+ load`：

```c++
 (*load_method)(cls, SEL_load);
```

这样的调用方式就使得 `+ load` 有了一个非常有趣的特性。我们知道，通常情况下我们给一个对象发送消息，其实是调用了 `objc_msgSend` 方法，如果对象不能响应消息，会递归地查找其父类/元类并调用。

而此处 `+ load` 方法直接使用了函数地址进行调用，并不是通过 `objc_msgSend`，所以**如果子类没有实现 `+ load` 方法，那么 runtime 不会调用父类的 `+ load`。如果类和分类都实现了 `+ load` 方法，两个方法都会被调用**。

到这里 `load_images` 的整个流程就讲完了，不过如果你跟我一样好奇的话，查看 `SEL_load` 时会发现——它竟然是个 `NULL`！！！

```objective-c
// objc-runtime.mm
// Selectors
SEL SEL_load = NULL;
SEL SEL_initialize = NULL;
SEL SEL_resolveInstanceMethod = NULL;
SEL SEL_resolveClassMethod = NULL;
SEL SEL_cxx_construct = NULL;
SEL SEL_cxx_destruct = NULL;
SEL SEL_retain = NULL;
SEL SEL_release = NULL;
SEL SEL_autorelease = NULL;
SEL SEL_retainCount = NULL;
SEL SEL_alloc = NULL;
SEL SEL_allocWithZone = NULL;
SEL SEL_dealloc = NULL;
SEL SEL_copy = NULL;
SEL SEL_new = NULL;
SEL SEL_forwardInvocation = NULL;
SEL SEL_tryRetain = NULL;
SEL SEL_isDeallocating = NULL;
SEL SEL_retainWeakReference = NULL;
SEL SEL_allowsWeakReference = NULL;
```

至于为什么会这样，我们后面会详述。

#### Producer-Consumer

细心的读者可能已经发现了，`load_images` 里其实蕴含着一个经典的生产者-消费者问题：

- `prepare_load_methods` 负责“生产”：从镜像中获取实现了 `+ load` 方法的类/分类，将类/分类及其 load 方法写入 `loadable_classes`/`loadable_categories` 列表中；
- `call_load_methods` 负责“消费”：循环地从 `loadable_classes`/`loadable_categories` 列表中读取类/分类，执行其 `+ load` 方法

##### loadMethodLock

当然，🍎 已经加好了锁。因为类和分类的流程大同小异，所以我们以类为🌰：

```c++
// `prepare_load_methods` 负责“生产”
void add_class_to_loadable_list(Class cls)
{
    // ...
    loadMethodLock.assertLocked();
    // ...
    loadable_classes[loadable_classes_used].cls = cls;
    loadable_classes[loadable_classes_used].method = method;
    loadable_classes_used++;
}

// `call_load_methods` 负责“消费”
void call_load_methods(void)
{
    // ...
    loadMethodLock.assertLocked();
	// ...
    call_class_loads
}

static void call_class_loads(void)
{
    // ...
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    //...
    (*load_method)(cls, SEL_load);
    // ...
    if (classes) free(classes);
}
```

## _objc_init

还记得我们之前遗留的问题吗？remapClass 中的那个映射表是哪来的？`SEL_load` 等选择子为什么是 NULL？

之所以会遇到这些问题，其实是因为我们前面有些“急于求成”，从 `load` 方法处的断点定位到 `load_images` 后就直接一头扎了进去，而没有想想——`load_images` 是谁调用的？在 `load_images` 之前发生了什么？

我们在 `load_images` 处添加一个断点，通过调用栈可以看到另一个方法：`_objc_init`

![Runtime_Breakpoint_LoadImages](Runtime_Breakpoint_LoadImages.png)

其源码如下：

```c++
// objc-os.mm
/***********************************************************************
* _objc_init
* Bootstrap initialization. Registers our image notifier with dyld.
* Called by libSystem BEFORE library initialization time
**********************************************************************/
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    lock_init();
    exception_init();

    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
}
```

这里做了很多初始化的工作，并通过一个静态的布尔型变量保证该方法只会走一次。我们的关注点在于最后一行，这里出现了我们前面花了大量篇幅所讲的 `load_images`，那么这一整行代码是什么意思呢？

### _dyld_objc_notify_register

我们跳转到 `_dyld_objc_notify_register` 的定义，这里已经踏入 dyld 的领地了：

```c++
// dyld_priv.h
typedef void (*_dyld_objc_notify_mapped)(unsigned count, const char* const paths[], const struct mach_header* const mh[]);
typedef void (*_dyld_objc_notify_init)(const char* path, const struct mach_header* mh);
typedef void (*_dyld_objc_notify_unmapped)(const char* path, const struct mach_header* mh);

//
// Note: only for use by objc runtime
// Register handlers to be called when objc images are mapped, unmapped, and initialized.
// Dyld will call back the "mapped" function with an array of images that contain an objc-image-info section.
// Those images that are dylibs will have the ref-counts automatically bumped, so objc will no longer need to
// call dlopen() on them to keep them from being unloaded.  During the call to _dyld_objc_notify_register(),
// dyld will call the "mapped" function with already loaded objc images.  During any later dlopen() call,
// dyld will also call the "mapped" function.  Dyld will call the "init" function when dyld would be called
// initializers in that image.  This is when objc calls any +load methods in that image.
//
void _dyld_objc_notify_register(_dyld_objc_notify_mapped    mapped,
                                _dyld_objc_notify_init      init,
                                _dyld_objc_notify_unmapped  unmapped);
```

请允许我蹩脚的翻译一下其中的部分内容：

> “注册处理程序，以便 oc 的镜像在被 mapped、unmappped 和 initialized 时调用……当 dyld 初始化一个镜像时，会调用 init 函数，镜像的 + load 方法就在这个函数中被调用。”

（这里的 “init function”，也就是我们上文所说的 `load_images`）

所以 runtime 其实是注册了三个监听：`_dyld_objc_notify_mapped` 、 `_dyld_objc_notify_init` 、 `_dyld_objc_notify_unmapped`，对应着三个不同的回调。当一个镜像被 mapped、init 和 unmapped 时，dyld 就会通知 runtime 进行相应的处理。

 那么显然，想要解答我们之前遗留的困惑，就需要去看看在 “init” 之前的 “mapped” 中都做了些什么。

### map_images

```c++
//objc-runtime-new.mm
/***********************************************************************
* map_images
* Process the given images which are being mapped in by dyld.
* Calls ABI-agnostic code after taking ABI-specific locks.
*
* Locking: write-locks runtimeLock
**********************************************************************/
void
map_images(unsigned count, const char * const paths[],
           const struct mach_header * const mhdrs[])
{
    mutex_locker_t lock(runtimeLock);
    return map_images_nolock(count, paths, mhdrs);
}
```

#### map_images_nolock

`map_images_nolock` 的实现比较长，我们忽略掉一些相对不那么重要的内容，然后挑选其中一部分讲讲：

```c++
// objc-os.mm
void
map_images_nolock(unsigned mhCount, const char * const mhPaths[],
                  const struct mach_header * const mhdrs[])
{
    static bool firstTime = YES;

    // ...
    
    if (firstTime) {
        sel_init(selrefCount);
        arr_init();
        
        // ...
    }
    
    if (hCount > 0) {
        _read_images(hList, hCount, totalClasses, unoptimizedTotalClasses);
    }
    
    firstTime = NO;
}
```

我们可以看到，当第一次执行时，这里会做一些初始化的操作，下面我们就分别来看一下 `sel_init` 、 `arr_init` 、 `_read_images` 这三步都做了些什么。

##### sel_init

顾名思义，这里就是初始化 SEL 的地方，我们前面的问题 `SEL SEL_load = NULL;` 也将在这里得到解答。

```c++
// objc-sel.mm
/***********************************************************************
* sel_init
* Initialize selector tables and register selectors used internally.
**********************************************************************/
void sel_init(size_t selrefCount)
{
	// ...略

    // Register selectors used by libobjc

#define s(x) SEL_##x = sel_registerNameNoLock(#x, NO)
#define t(x,y) SEL_##y = sel_registerNameNoLock(#x, NO)

    mutex_locker_t lock(selLock);

    s(load);
    s(initialize);
    t(resolveInstanceMethod:, resolveInstanceMethod);
    t(resolveClassMethod:, resolveClassMethod);
    t(.cxx_construct, cxx_construct);
    t(.cxx_destruct, cxx_destruct);
    s(retain);
    s(release);
    s(autorelease);
    s(retainCount);
    s(alloc);
    t(allocWithZone:, allocWithZone);
    s(dealloc);
    s(copy);
    s(new);
    t(forwardInvocation:, forwardInvocation);
    t(_tryRetain, tryRetain);
    t(_isDeallocating, isDeallocating);
    s(retainWeakReference);
    s(allowsWeakReference);

#undef s
#undef t
}
```

类似地，这里也 ~~偷懒~~ 巧妙的使用了宏定义来起到模板的作用。我们以 `load` 为🌰，`s(load);` 相当于：

```objective-c
SEL_load = sel_registerNameNoLock("load", NO);
```

> `##` 是拼接，`#` 是转化为字符串

最终调用的是 `__sel_registerName`，这里会去一个全局的映射表中进行查找，如果表不存在则创建一个，如果在表中没有找到，则将 SEL name 和 SEL 作为一对 key-value 添加到映射表中。至此我们关于 SEL 的问题便得到了答案。

```c++
// objc-sel.mm
static SEL __sel_registerName(const char *name, bool shouldLock, bool copy) 
{
    SEL result = 0;

    // ...略
    
    if (namedSelectors) {
        result = (SEL)NXMapGet(namedSelectors, name);
    }
    if (result) return result;

    if (!namedSelectors) {
        namedSelectors = NXCreateMapTable(NXStrValueMapPrototype, 
                                          (unsigned)SelrefCount);
    }
    if (!result) {
        result = sel_alloc(name, copy);
        NXMapInsert(namedSelectors, sel_getName(result), result);
    }

    return result;
}
```

##### arr_init

回到 `map_images_nolock` 中，我们继续看另一个初始化：

```c++
void arr_init(void) 
{
    AutoreleasePoolPage::init();
    SideTableInit();
}
```

原来 `AutoreleasePool` 和 `SideTable` 都是在这里初始化的。有的同学可能对 `SideTable` 比较陌生，大名鼎鼎的引用计数表以及 `weak` 的实现原理等，全都和它有关。关于 `AutoreleasePool` 和 `SideTable`，我们会在其它文章中单独介绍。

##### _read_images

现在 `map_images_nolock` 中只剩下一个重头戏：`_read_images` 了。这个方法的实现长得令人发指（400行），~~笔者已经无力吐槽~~，主要是检查所有的 header_into，处理其中的 类、分类、协议、消息等信息，插入相应的映射表中。我们摘取其中比较重要的部分：

```c++
// objc-runtime-new.mm
/***********************************************************************
* _read_images
* Perform initial processing of the headers in the linked 
* list beginning with headerList. 
*
* Called by: map_images_nolock
*
* Locking: runtimeLock acquired by map_images
**********************************************************************/
void _read_images(header_info **hList, uint32_t hCount, int totalClasses, int unoptimizedTotalClasses)
{
    // ... 略
    TimeLogger ts(PrintImageTimes);
    // ... 略
    NXCreateHashTable
    ts.log("IMAGE TIMES: first time tasks");
    // ... 略
    _getObjc2ClassList
    readClass
	ts.log("IMAGE TIMES: discover classes");
    // ... 略
    _getObjc2ClassRefs
    remapClassRef
    ts.log("IMAGE TIMES: remap classes");
    // ... 略
    _getObjc2SelectorRefs
    sel_registerNameNoLock
    ts.log("IMAGE TIMES: fix up selector references");
    // ... 略
    _getObjc2MessageRefs
    fixupMessageRef
	ts.log("IMAGE TIMES: fix up objc_msgSend_fixup");
    // ... 略
    _getObjc2ProtocolList
    readProtocol
    ts.log("IMAGE TIMES: discover protocols");
    // ... 略
    _getObjc2ProtocolRefs
    remapProtocolRef
    ts.log("IMAGE TIMES: fix up @protocol references");
    // ... 略
    _getObjc2NonlazyClassList
    realizeClass
	ts.log("IMAGE TIMES: realize non-lazy classes");
    // ... 略
    _getObjc2CategoryList
    addUnattachedCategoryForClass
    remethodizeClass
	ts.log("IMAGE TIMES: discover categories");
    // ... 略
}
```

这里面每一处都是一次对所有 header_info 的遍历（但其实我不太明白在一次遍历中做这些事不行么？）

这里🍎已经写好了各个步骤耗时的监测，我们看上面的第一行代码：`TimeLogger ts(PrintImageTimes);` ，我们跳转到 `PrintImageTimes` 的定义：

```objective-c
// objc-env.h
// OPTION(var, env, help)

OPTION( PrintImages,              OBJC_PRINT_IMAGES,               "log image and library names as they are loaded")
OPTION( PrintImageTimes,          OBJC_PRINT_IMAGE_TIMES,          "measure duration of image loading steps")
OPTION( PrintLoading,             OBJC_PRINT_LOAD_METHODS,         "log calls to class and category +load methods")
OPTION( PrintInitializing,        OBJC_PRINT_INITIALIZE_METHODS,   "log calls to class +initialize methods")
OPTION( PrintResolving,           OBJC_PRINT_RESOLVED_METHODS,     "log methods created by +resolveClassMethod: and +resolveInstanceMethod:")
OPTION( PrintConnecting,          OBJC_PRINT_CLASS_SETUP,          "log progress of class and category setup")
OPTION( PrintProtocols,           OBJC_PRINT_PROTOCOL_SETUP,       "log progress of protocol setup")
OPTION( PrintIvars,               OBJC_PRINT_IVAR_SETUP,           "log processing of non-fragile ivars")
OPTION( PrintVtables,             OBJC_PRINT_VTABLE_SETUP,         "log processing of class vtables")
OPTION( PrintVtableImages,        OBJC_PRINT_VTABLE_IMAGES,        "print vtable images showing overridden methods")
OPTION( PrintCaches,              OBJC_PRINT_CACHE_SETUP,          "log processing of method caches")
OPTION( PrintFuture,              OBJC_PRINT_FUTURE_CLASSES,       "log use of future classes for toll-free bridging")
OPTION( PrintPreopt,              OBJC_PRINT_PREOPTIMIZATION,      "log preoptimization courtesy of dyld shared cache")
OPTION( PrintCxxCtors,            OBJC_PRINT_CXX_CTORS,            "log calls to C++ ctors and dtors for instance variables")
OPTION( PrintExceptions,          OBJC_PRINT_EXCEPTIONS,           "log exception handling")
OPTION( PrintExceptionThrow,      OBJC_PRINT_EXCEPTION_THROW,      "log backtrace of every objc_exception_throw()")
OPTION( PrintAltHandlers,         OBJC_PRINT_ALT_HANDLERS,         "log processing of exception alt handlers")
OPTION( PrintReplacedMethods,     OBJC_PRINT_REPLACED_METHODS,     "log methods replaced by category implementations")
OPTION( PrintDeprecation,         OBJC_PRINT_DEPRECATION_WARNINGS, "warn about calls to deprecated runtime functions")
OPTION( PrintPoolHiwat,           OBJC_PRINT_POOL_HIGHWATER,       "log high-water marks for autorelease pools")
OPTION( PrintCustomRR,            OBJC_PRINT_CUSTOM_RR,            "log classes with un-optimized custom retain/release methods")
OPTION( PrintCustomAWZ,           OBJC_PRINT_CUSTOM_AWZ,           "log classes with un-optimized custom allocWithZone methods")
OPTION( PrintRawIsa,              OBJC_PRINT_RAW_ISA,              "log classes that require raw pointer isa fields")

OPTION( DebugUnload,              OBJC_DEBUG_UNLOAD,               "warn about poorly-behaving bundles when unloaded")
OPTION( DebugFragileSuperclasses, OBJC_DEBUG_FRAGILE_SUPERCLASSES, "warn about subclasses that may have been broken by subsequent changes to superclasses")
OPTION( DebugNilSync,             OBJC_DEBUG_NIL_SYNC,             "warn about @synchronized(nil), which does no synchronization")
OPTION( DebugNonFragileIvars,     OBJC_DEBUG_NONFRAGILE_IVARS,     "capriciously rearrange non-fragile ivars")
OPTION( DebugAltHandlers,         OBJC_DEBUG_ALT_HANDLERS,         "record more info about bad alt handler use")
OPTION( DebugMissingPools,        OBJC_DEBUG_MISSING_POOLS,        "warn about autorelease with no pool in place, which may be a leak")
OPTION( DebugPoolAllocation,      OBJC_DEBUG_POOL_ALLOCATION,      "halt when autorelease pools are popped out of order, and allow heap debuggers to track autorelease pools")
OPTION( DebugDuplicateClasses,    OBJC_DEBUG_DUPLICATE_CLASSES,    "halt when multiple classes with the same name are present")
OPTION( DebugDontCrash,           OBJC_DEBUG_DONT_CRASH,           "halt the process by exiting instead of crashing")

OPTION( DisableVtables,           OBJC_DISABLE_VTABLES,            "disable vtable dispatch")
OPTION( DisablePreopt,            OBJC_DISABLE_PREOPTIMIZATION,    "disable preoptimization courtesy of dyld shared cache")
OPTION( DisableTaggedPointers,    OBJC_DISABLE_TAGGED_POINTERS,    "disable tagged pointer optimization of NSNumber et al.") 
OPTION( DisableTaggedPointerObfuscation, OBJC_DISABLE_TAG_OBFUSCATION,    "disable obfuscation of tagged pointers")
OPTION( DisableNonpointerIsa,     OBJC_DISABLE_NONPOINTER_ISA,     "disable non-pointer isa fields")
OPTION( DisableInitializeForkSafety, OBJC_DISABLE_INITIALIZE_FORK_SAFETY, "disable safety checks for +initialize after fork")
```

(*@ο@*) 哇～偌大的一个宝库！这里定义了大量的运行时环境变量，如果开启的话，不仅有助于我们学习 runtime 的源码，也大大方便了负责性能优化的同学做一些统计的耗时等。

我们就以刚刚说到的 `PrintImageTimes` 为例，它对应的运行时环境变量为 `OBJC_PRINT_IMAGE_TIMES` ，我们在 Xcode -> Project -> Scheme -> Edit Scheme 中将该环境变量设为 YES：

![Runtime_PrintOption_PrintImageTimes](Runtime_PrintOption_PrintImageTimes.png)

运行程序，就可以方便地看到各阶段的耗时了：

```c
objc[40242]: 0.39 ms: IMAGE TIMES: first time tasks
objc[40242]: 18.40 ms: IMAGE TIMES: discover classes
objc[40242]: 0.00 ms: IMAGE TIMES: remap classes
objc[40242]: 100.89 ms: IMAGE TIMES: fix up selector references
objc[40242]: 0.14 ms: IMAGE TIMES: fix up objc_msgSend_fixup
objc[40242]: 2.47 ms: IMAGE TIMES: discover protocols
objc[40242]: 0.67 ms: IMAGE TIMES: fix up @protocol references
objc[40242]: 3.50 ms: IMAGE TIMES: realize non-lazy classes
objc[40242]: 0.00 ms: IMAGE TIMES: realize future classes
objc[40242]: 3.00 ms: IMAGE TIMES: discover categories
2019-04-12 16:18:02.921120+0800 debug-objc[40242:2106148] Hello, World!
Program ended with exit code: 0
```



![runtime_process](Runtime_Process.png)



## + initialize

```objective-c
// ViewController.m
#import "ViewController.h"

@interface SubViewController : ViewController
@end

@implementation SubViewController

+ (void)initialize {
    NSLog(@"SubViewController initialize...");
}

@end

@implementation ViewController

+ (void)initialize {
    NSLog(@"ViewController initialize...");
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    SubViewController *subVC = [[SubViewController alloc] init];
}

@end

    .
    .
    .
    .
    .
    .
    
2019-05-15 16:55:02.331135+0800 Test[25870:1727530] ViewController initialize...
2019-05-15 16:55:02.334713+0800 Test[25870:1727530] SubViewController initialize...
```

那么如果我们把子类中的 `+ initialize` 删掉，会是什么样呢？

```objective-c
// ViewController.m
#import "ViewController.h"

@interface SubViewController : ViewController
@end

@implementation SubViewController
@end

@implementation ViewController

+ (void)initialize {
    NSLog(@"ViewController initialize... self = %@", self);
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    SubViewController *subVC = [[SubViewController alloc] init];
}

@end
    
    .
    .
    .
    .
    .
    .
    
2019-05-15 16:55:50.386359+0800 Test[25907:1728843] ViewController initialize... self = ViewController
2019-05-15 16:55:50.390092+0800 Test[25907:1728843] ViewController initialize... self = SubViewController
```

查看源码，可以发现 `initialize` 的调用是通过 `objc_msgSend`，所以如果子类没有实现，则会调用父类的方法。

```c++
void callInitialize(Class cls)
{
    ((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize);
    // ...
}
```

也就是说如果我们不加判断的话，一个类的 initialize 可能会执行多次，而这并不一定是我们想要的。

Q：那么有的同学可能会说，如果我们没有子类，或者子类实现了 `+ initialize` 不就没问题了么？

A：❌。我们来看下面这个🌰，现在我们让子类实现 `+ initialize`，这样子类就不会再调用父类的 `+initialize`了。然后我们给父类添加一个 KVO，猜猜现在输出是什么？

```objective-c
// ViewController.m

#import "ViewController.h"

@interface SubViewController : ViewController
@end

@implementation SubViewController

+ (void)initialize {
    NSLog(@"SubViewController initialize...");
}

@end

@interface ViewController ()
@property (nonatomic, strong) SubViewController *subVC;
@end

@implementation ViewController

+ (void)initialize {
    NSLog(@"ViewController initialize... self = %@", self);
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.subVC = [[SubViewController alloc] init];
    
    [self addObserver:self forKeyPath:@"view" options:NSKeyValueObservingOptionNew context:nil];
}

- (void)dealloc {
    [self removeObserver:self forKeyPath:@"view"];
}

@end
    
    .
    .
    .
    .
    .
    .
    
2019-05-15 16:52:14.792672+0800 Test[25830:1725181] ViewController initialize... self = ViewController
2019-05-15 16:52:14.796562+0800 Test[25830:1725181] SubViewController initialize...
2019-05-15 16:52:14.796983+0800 Test[25830:1725181] ViewController initialize... self = NSKVONotifying_ViewController
```

`ViewController` 的 `+ initialize` 还是被调用了两次。这是因为 KVO 动态地创建了一个子类（`NSKVONotifying_ViewController`），而这个子类不会重写 `+ initialize`，所以父类的 `+ initialize` 就又被调用了。

那么我们该如何解决呢？🍎 的文档中说：“If you want to protect yourself from being run multiple times, you can structure your implementation along these lines:”

```objective-c
+ (void)initialize {
    if (self == [ClassName self]) {
        // ... do the initialization ...
    }
}
```



## QA

Q：如果一个类实现了 `+ load` 方法，但是这个类没有被使用（import），那么它还会在 load_images 时加载嘛？
A：会。

Q：如果一个类实现了 `+ load` 方法，但是具体实现内容是空，那么还会在 load_images 时被加载嘛？
A：会。

Q：同一个镜像中，多个不同的类、多个不同的分类的 `+ load` 调用顺序是怎样的？
A：编译顺序。

Q：同一个镜像中，NonLazyClassA 依赖了 NonLazyClassB，那么它们的 `+ load` 调用顺序是怎样的？
A：还是编译顺序。

## References

- <https://opensource.apple.com/tarballs/objc4/>
- <https://developer.apple.com/videos/play/wwdc2016/406>
- <https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048>
- <https://www.mikeash.com/pyblog/friday-qa-2009-05-22-objective-c-class-loading-and-initialization.html>
- <http://clang.llvm.org/docs/AutomaticReferenceCounting.html#dealloc>
- <https://nshipster.com/method-swizzling/>

