# 背景简述
> 在日常开发过程中是否有过这样的需求：不修改原来的函数，但是又想在函数的执行前后插入一些代码。这个方式就是面向切面（AOP），在iOS开发中比较知名的框架就是[Aspects](https://github.com/steipete/Aspects)，而饿了么新出的[Stinger](https://github.com/eleme/Stinger)框架先不讨论，Aspects的源码精炼巧妙，很值得学习深究，本文主要从源码和应用层面来介绍下

# 源码解析

## 先提出几个问题
带着问题去阅读更容易理解

1. Aspects实现的核心原理是什么
2. 哪些方法不能被hook
3. hook的操作是否可以只对某个实例生效，对同一个类的其他实例不生效？
4. block是如何被存储和调用的

## 基本原理
正常来讲想实现AOP，可以利用runtime的特性进行method swizzle，但Aspects就是造好的轮子，而且更好用，下面简述下Aspects的基本原理

### runtime的消息转发机制
在OC中，所有的消息调用最后都会通过`objc_msgSend()`方法进行访问
1. 通过`objc_msgSend()`进行消息调用，为了加快执行速度，这个方法在runtime源码中是用汇编实现的
2. 然后调用`lookUpImpOrForward()`方法，返回值是个`IMP`指针，如果查找到了调用函数的`IMP`，则进行方法的访问
3. 如果没有查到对于方法的`IMP`指针，则进行消息转发机制
4. 第一层转发：会调用`resolveInstanceMethod:、resolveClassMethod:`，这次转发是方法级别的，开发者可以动态添加方法进行补救
5. 第二层转发：如果第一层转发仍然没有找到`SEL`，则会进行第二层转发，调用`forwardingTargetForSelector:`，可以把调用转发到另一个对象，这是类级别的转发，调用另一个类的相同的方法
6. 第三层转发：如果第二层转发返回`nil`，则会进入这一层处理，这层会调用`methodSignatureForSelector:、forwardInvocation:`，这次是完整的消息转发，因为你可以返回方法签名、动态指定调用方法的Target
7. 如果转发都失败，就会crash

### Aspects的基本原理
对外暴露的核心API

```objc
/**
作用域：针对所有对象生效
selector: 需要hook的方法
options：是个枚举，主要定义了切面的时机（调用前、替换、调用后）
block: 需要在selector前后插入执行的代码块
error: 错误信息
*/
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                           withOptions:(AspectOptions)options
                            usingBlock:(id)block
                                 error:(NSError **)error;
/**
作用域：针对当前对象生效
*/
- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                           withOptions:(AspectOptions)options
                            usingBlock:(id)block
                                 error:(NSError **)error;

```
上面介绍了消息的转发机制，而`Aspects`就是利用了消息转发机制，通过hook第三层的转发方法`forwardInvocation:`，然后根据切面的时机来动态调用block。接下来详细分析巧妙的设计
1. 类A的方法m被添加切面方法
2. 创建一个类A的子类B，并hook子类B的`forwardInvocation:`方法拦截消息转发，使`forwardInvocation:`的`IMP`指向事先准备好的`__ASPECTS_ARE_BEING_CALLED__`函数（后面简称`ABC`函数），block方法的执行就在`ABC`函数中
3. 把类A的对象的isa指针指向B，这样就把消息的处理转发到类B上，类似`KVO`的机制，同时会更改`class`方法的IMP，把它指向类A的`class`方法，当外界调用`class`时获取的还是类A，并不知道中间类B的存在
4. 对于方法m，类B会直接把方法m的`IMP`指向`_objc_msgForward()`方法，这样当调用方法m时就会走消息转发流程，触发`ABC`函数

## 详细分析
### 执行入口
```objective-c
- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error {
    return aspect_add(self, selector, options, block, error);
}

static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
    __block AspectIdentifier *identifier = nil;
    // 添加自旋锁，block内容的执行时互斥的
    aspect_performLocked(^{
        if (aspect_isSelectorAllowedAndTrack(self, selector, options, error)) {
            // 获取容器，容器的对象以关联对象的方式添加到了当前对象上，key值为`前缀+selector`
            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
            // 创建标识符，用来存储SEL、block、切面时机（调用前、调用后）等信息
            identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
            if (identifier) {
                [aspectContainer addAspect:identifier withOptions:options];

                // Modify the class to allow message interception.
                aspect_prepareClassAndHookSelector(self, selector, error);
            }
        }
    });
    return identifier;
}

```
执行入口调用了`aspect_add(self, selector, options, block, error)`方法，这个方法时线程安全的，接下来一步步解析具体做了什么
### 过滤拦截：`aspect_isSelectorAllowedAndTrack()`
精简版的源码，已经添加了注释

```objective-c
static BOOL aspect_isSelectorAllowedAndTrack(NSObject *self, SEL selector, AspectOptions options, NSError **error) {
    static NSSet *disallowedSelectorList;
    static dispatch_once_t pred;
    dispatch_once(&pred, ^{ // 初始化黑名单列表，有些方法时禁止hook的
        disallowedSelectorList = [NSSet setWithObjects:@"retain", @"release", @"autorelease", @"forwardInvocation:", nil];
    });

    // 第一步：检查是否在黑名单内
    NSString *selectorName = NSStringFromSelector(selector);
    if ([disallowedSelectorList containsObject:selectorName]) {
        ...
        return NO;
    }

    // 第二步： dealloc方法只能在调用前插入
    AspectOptions position = options&AspectPositionFilter;
    if ([selectorName isEqualToString:@"dealloc"] && position != AspectPositionBefore) {
        ...
        return NO;
    }
    // 第三步：检查类是否存在这个方法
    if (![self respondsToSelector:selector] && ![self.class instancesRespondToSelector:selector]) {
        ...
        return NO;
    }

    // 第四步：如果是类而非实例(这个是类，不是类方法，是指hook的作用域对所有对象都生效)，则在整个类即继承链中，同一个方法只能被hook一次，即对于所有实例对象都生效的操作，整个继承链中只能被hook一次
    if (class_isMetaClass(object_getClass(self))) {
        ...
    } else {
        return YES;
    }
    return YES;
}

```

1. 不允许hook`retain`、`release`、`autorelease`、`forwardInvocation:`，这些不多解释
2. 允许hook`dealloc`，但是只能在`dealloc`执行前，这都是为了程序的安全性设置的
3. 检查这个方法是否存在，不存在则不能hook
4. Aspects对于hook的生效作用域做了区分：所有实例对象&某个具体实例对象。对于所有实例对象在整个继承链中，同一个方法只能被hook一次，这么做的目的是为了规避循环调用的问题（详情可以了解下`supper`关键字）

### 关键类结构
#### AspectOptions
是个枚举，用来定义切面的时机，即原有方法调用前、调用后、替换原有方法、只执行一次（调用完就删除切面逻辑）
```objc
typedef NS_OPTIONS(NSUInteger, AspectOptions) {
    AspectPositionAfter   = 0,            /// 原有方法调用前执行 (default)
    AspectPositionInstead = 1,            /// 替换原有方法
    AspectPositionBefore  = 2,            /// 原有方法调用后执行
    
    AspectOptionAutomaticRemoval = 1 << 3 /// 执行完之后就恢复切面操作，即撤销hook
};

```
#### AspectIdentifier类
简单理解话就是一个存储model，主要用来存储hook方法的相关信息，如原有方法、切面block、切面时机等
```objective-c
@interface AspectIdentifier : NSObject
...其他省略
@property (nonatomic, assign) SEL selector; // 原来方法的SEL
@property (nonatomic, strong) id block; // 保存要执行的切面block，即原方法执行前后要调用的方法
@property (nonatomic, strong) NSMethodSignature *blockSignature; // block的方法签名
@property (nonatomic, weak) id object; // target，即保存当前对象
@property (nonatomic, assign) AspectOptions options; // 是个枚举，表示切面执行时机，上面已经有介绍
@end

```
#### AspectsContainer类
容器类，以关联对象的形式存储在当前类或对象中，主要用来存储当前类或对象所有的切面信息
```
@interface AspectsContainer : NSObject
...其他省略
@property (atomic, copy) NSArray <AspectIdentifier *>*beforeAspects; // 存储原方法调用前要执行的操作
@property (atomic, copy) NSArray <AspectIdentifier *>*insteadAspects;// 存储替换原方法的操作
@property (atomic, copy) NSArray <AspectIdentifier *>*afterAspects;// 存储原方法调用后要执行的操作
@end

```
### 存储切面信息
存储切面信息主要用到了上面介绍的`AspectsContainer`、`AspectIdentifier`这两个类，主要操作如下（注释写的已经很详细）
1. 获取当前类的容器对象`aspectContainer`，如果没有则创建一个
2. 创建一个标识符对象`identifier`，用来存储原方法信息、block、切面时机等信息
3. 把标识符对象`identifier`添加到容器中
```objective-c
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
    ...
    // 获取容器对象，主要用来存储当前类或对象所有的切面信息，容器的对象以关联对象的方式添加到了当前对象上，key值为`前缀+selector`
    AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
    // 创建标识符，用来存储SEL、block、切面时机（调用前、调用后）等信息
    identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
    if (identifier) {
        // 把identifier添加到容器中
        [aspectContainer addAspect:identifier withOptions:options];
        ...
    }
    return identifier;
}
```
### 创建中间类
这一步的操作类似kvo的机制，隐式的创建一个中间类，一：可以做到hook只对单一对象有效，二：避免了对原有类的侵入

这一步主要做了几个操作
1. 如果已经存在中间类，则直接返回
2. 如果是类对象，则不用创建中间类，并把这个类存储在`swizzledClasses`集合中，标记这个类已经被hook了
3. 如果存在kvo的情况，那么系统已经帮我们创建好了中间类，那就直接使用
4. 对于不存在kvo且是实例对象的，则单独创建一个继承当前类的中间类`midcls`，并hook它的`forwardInvocation:`方法，并把当前对象的isa指针指向`midcls`，这样就做到了hook操作只针对当前对象有效，因为其他对象的isa指针指向的还是原有类
```objective-c
static Class aspect_hookClass(NSObject *self, NSError **error) {
	Class statedClass = self.class;
	Class baseClass = object_getClass(self);
	NSString *className = NSStringFromClass(baseClass);

    // Already subclassed
	if ([className hasSuffix:AspectsSubclassSuffix]) {
		return baseClass;

        // We swizzle a class object, not a single object.
	}else if (class_isMetaClass(baseClass)) {
        return aspect_swizzleClassInPlace((Class)self);
        }else if (statedClass != baseClass) {
        // Probably a KVO class. Swizzle in place. Also swizzle meta classes in place.
        return aspect_swizzleClassInPlace(baseClass);
        }

    // Default case. Create dynamic subclass.
	const char *subclassName = [className stringByAppendingString:AspectsSubclassSuffix].UTF8String;
	Class subclass = objc_getClass(subclassName);

	if (subclass == nil) {
	    subclass = objc_allocateClassPair(baseClass, subclassName, 0);
            // hook forwardInvocation方法
	    aspect_swizzleForwardInvocation(subclass);
            // hook class方法，把子类的class方法的IMP指向父类，这样外界并不知道内部创建了子类
	    aspect_hookedGetClass(subclass, statedClass);
	    aspect_hookedGetClass(object_getClass(subclass), statedClass);
	    objc_registerClassPair(subclass);
	}
    // 把当前对象的isa指向子类，类似kvo的用法
	object_setClass(self, subclass);
	return subclass;
}
```
### 替换forwardInvocation：方法
从下面的代码可以看到，主要功能就是把当前类的`forwardInvocation:`替换成`__ASPECTS_ARE_BEING_CALLED__`，这样当触发消息转发的时候，就会调用`__ASPECTS_ARE_BEING_CALLED__`方法

对于`__ASPECTS_ARE_BEING_CALLED__`方法是`Aspects`的核心操作，主要就是做消息的调用和分发，控制方法的调用的时机，下面会详细介绍
```objective-c
// hook forwardInvocation方法，用来拦截消息的发送
static void aspect_swizzleForwardInvocation(Class klass) {
    // If there is no method, replace will act like class_addMethod.
    IMP originalImplementation = class_replaceMethod(klass, @selector(forwardInvocation:), (IMP)__ASPECTS_ARE_BEING_CALLED__, "v@:@");
    if (originalImplementation) {
        class_addMethod(klass, NSSelectorFromString(AspectsForwardInvocationSelectorName), originalImplementation, "v@:@");
    }
    AspectLog(@"Aspects: %@ is now aspect aware.", NSStringFromClass(klass));
}
```

### 自动触发消息转发机制
`Aspects`的核心原理是消息转发，那么必要出的就是怎么自动触发消息转发机制

runtime中有个方法`_objc_msgForward`，直接调用可以触发消息转发机制，著名的`JSPatch`框架也是利用了这个机制

假如要hook的方法叫`m1`，那么把`m1`的`IMP`指向`_objc_msgForward`，这样当调用方法`m1`时就自动触发消息转发机制了，详细实现如下

```objective-c
static void aspect_prepareClassAndHookSelector(NSObject *self, SEL selector, NSError **error) {

    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);
    if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
        ...
        // We use forwardInvocation to hook in. 把函数的调用直接触发转发函数，转发函数已经被hook，所以在转发函数时进行block的调用
        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
    }
}
```
### 核心转发函数处理
上面一切准备就绪，那么怎么触发之前添加的切面block呢，首先我们梳理下整个流程
1. 方法`m1`的`IMP`指向了`_objc_msgForward`，调用`m1`则会自动触发消息转发机制
2. 替换`forwardInvocation:`，把它的`IMP`指向`__ASPECTS_ARE_BEING_CALLED__`方法，消息转发时触发的就是`__ASPECTS_ARE_BEING_CALLED__`

上面操作可以直接看出调用方法`m1`则会直接触发`__ASPECTS_ARE_BEING_CALLED__`方法，而`__ASPECTS_ARE_BEING_CALLED__`方法就是处理切面block用和原有函数的调用时机，详细看下面实现步骤

1. 根据调用的`selector`，获取容器对象`AspectsContainer`，这里面存储了这个类或对象的所有切面信息
2. `AspectInfo`会存储当前的参数信息，用于传递
3. 首先触发函数调用前的block，存储在容器的`beforeAspects`对象中
4. 接下来如果存在替换原有函数的block，即`insteadAspects`不为空，则触发它，如果不存在则调用原来的函数
5. 触发函数调用后的block，存在在容器的`afterAspects`对象中

```objective-c
static void __ASPECTS_ARE_BEING_CALLED__(__unsafe_unretained NSObject *self, SEL selector, NSInvocation *invocation) {
    AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
    AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
    AspectInfo *info = [[AspectInfo alloc] initWithInstance:self invocation:invocation];

    // Before hooks. 方法执行之前调用
    aspect_invoke(classContainer.beforeAspects, info);
    aspect_invoke(objectContainer.beforeAspects, info);

    // Instead hooks. 替换原方法或者调用原方法
    BOOL respondsToAlias = YES;
    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
        aspect_invoke(classContainer.insteadAspects, info);
        aspect_invoke(objectContainer.insteadAspects, info);
    }else {
        Class klass = object_getClass(invocation.target);
        do {
            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
                [invocation invoke];
                break;
            }
        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
    }

    // After hooks. 方法执行之后调用
    aspect_invoke(classContainer.afterAspects, info);
    aspect_invoke(objectContainer.afterAspects, info);

    ...
    // Remove any hooks that are queued for deregistration.
    [aspectsToRemove makeObjectsPerformSelector:@selector(remove)];
}

```

# 总结
Aspects的核心原理是利用了消息转发机制，通过替换消息转发方法来实现切面的分发调用，这个思想很巧妙而且应用很广泛，很多三方库都利用了这个原理，值得学习

目前这个库已经很长时间没有维护了，原子操作的支持使用的还是自旋锁，目前这种锁已经不安全了

另外使用这个库是需要注意类似原理的其他框架，可能会有冲突，如`JSPatch`，不过`JSPatch`已经被封杀了，但类似需求有很多



**最后欢迎关注笔者公众号**：【码上work】，本公众号致力于浅显易懂的方式讲解移动/全栈开发、有趣算法、计算机基础知识等，帮你构建完整的知识体系，一起成为顶级开发者。公众号回复`12345`有大量学习资料：

1. iOS电子书（整理好组织结构）
2. 全栈开发视频教程（全套）
3. 机器学习视频（全套）
4. 区块链视频教程（全套）

![](https://tva1.sinaimg.cn/large/0082zybply1gc6pkgpfwhj3092050glt.jpg)