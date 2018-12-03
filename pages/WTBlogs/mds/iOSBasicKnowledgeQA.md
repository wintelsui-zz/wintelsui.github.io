# iOS常见知识问答

-------

##❏ 锁
http://www.cocoachina.com/cms/wap.php?action=article&id=22402

https://github.com/bestswifter/blog/blob/master/articles/ios-lock.md

===================================
OSSpinLock :
```
OSSpinLock lock = OS_SPINLOCK_INIT;
OSSpinLockLock(&lock);
OSSpinLockUnlock(&lock);
```

===================================
synchronized : 
```
@synchronized(self) {}
```

===================================

```
dispatch_semaphore :
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0); 
dispatch_semaphore_signal(semaphore); 
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
```

===================================

```
pthread_mutex :
pthread_mutex_t lock;
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
pthread_mutex_init(&lock, &attr);
pthread_mutexattr_destroy(&attr);
pthread_mutex_lock(&lock);
pthread_mutex_unlock(&lock);
```

===================================

NSLock:遵循协议 @protocol NSLocking
```
- (void)lock;
- (void)unlock;
- (BOOL)tryLock;
- (BOOL)lockBeforeDate:(NSDate *)limit;
```

===================================
os_unfair_lock:iOS 10+ 推荐替换不再安全的OSSpinLock
```
os_unfair_lock_t unfairLock;
unfairLock = &(OS_UNFAIR_LOCK_INIT);
os_unfair_lock_lock(unfairLock);
os_unfair_lock_unlock(unfairLock);
```

===================================
=================================== 

-------
##❏ KVO
键值监听，是观察者模式，用于监听属性的改变，主要方法包括:
添加监听 
```
- (void)addObserver:(NSObject *)observer 
         forKeyPath:(NSString *)keyPath 
            options:(NSKeyValueObservingOptions)options 
            context:(nullable void *)context;
```
移除监听 
```
- (void)removeObserver:(NSObject *)observer 
            forKeyPath:(NSString *)keyPath 
               context:(nullable void *)context;
```
监听回调 
```
- (void)observeValueForKeyPath:(nullable NSString *)keyPath 
                      ofObject:(nullable id)object 
                      change:(nullable NSDictionary<NSKeyValueChangeKey, id> *)change 
                      context:(nullable void *)context;
```

原理
KVO 通过isa-swizzling（类型混合指针）机制来实现，对象在被addObserver后，isa 指针指向了派生出的新类“NSKVONotifying_X元对象名X”，这个类继承自原类，并重写类被观察对象的的 Set 方法（原对象必须有该属性的 Set 方法），在新 Set 方法中添加了
```
- (void)willChangeValueForKey:(NSString *)key;
- (void)didChangeValueForKey:(NSString *)key;
```
等方法以回调给观察者。
手动触发回调，则需要调用
```
- (void)willChangeValueForKey:(NSString *)key;
- (void)didChangeValueForKey:(NSString *)key;
```


-------
##❏ KVC
通过Key名直接访问对象的属性，或者给对象的属性赋值.
```
- (void)setValue:(nullable id)value forKey:(NSString *)key;
```
接受者会按照一定顺序搜索设置属性方法（搜索 setKey, _key, _isKey, key, isKey）;
如果未找到设置方法，则会调用-setValue:forUndefinedKey:方法返回NSUndefinedKeyException异常；
如果找到方法，但是设置参数为 nil 或者类型不匹配，则会调用setNilValueForKey:方法并返回NSInvalidArgumentException异常。

```
- (nullable id)valueForKey:(NSString *)key;
```
接受者按照getKey:、key、isKey顺序搜索获取属性值方法；
如果未找到方法，则调用+accessInstanceVariablesDirectly方法查看类是否允许按照_key, _isKey, key, isKey顺序读取；
如果不允许或者最终未找到，则调用-valueForUndefinedKey方法，并返回NSUndefinedKeyException异常。

-------
##❏ 响应者链，事件传递
一系列直接或间接继承自UIResponder的类组成响应者链，用于在接收到触摸事件后沿着响应者链传递事件，响应者链始于 AppDelegate，UIWindow。

当系统检测到触摸事件后，被打包为 UIEvent 加入到UIApplication 的事件队列，UIApplication将事件传递给 UIWindow 处理，
调用
```
- (BOOL)pointInside:(CGPoint)point 
          withEvent:(nullable UIEvent *)event;
```
方法由层级下而上遍历，确定视图是否在点击区域内，以缩小传递范围；
再
```
- (UIView *)hitTest:(CGPoint)point 
          withEvent:(nullable UIEvent *)event;
```
方法自上而下遍历以确定响应者链，最终将事件传递给触摸视图。 

-------
##❏ weak原理
weak 对象弱引用
Runtime 维护了一个weak哈希表，用于存储指向某个对象的所有weak指针，Key是所指对象的地址，Value是weak指针的地址数组。

阶段：
1：objc_initWeak，初始化新的weak指针指向对象的地址。
2：objc_storeWeak，添加引用时，更新指针指向，创建对应的弱引用表。
3：clearDeallocating，根据对象地址获取weak指针地址的数组，遍历数组将其中的数据设为nil，将对象地址从weak表中删除。



-------
##❏ atomic
atomic原子属性

Set 方法:——reallySetProperty(…)
```
objc_retain(newValue);
spinlock_t& slotlock = PropertyLocks[slot];
slotlock.lock();
oldValue = newValue;
slotlock.unlock();
objc_release(oldValue);
```

Get 方法:——objc_getProperty(…)
```
spinlock_t& slotlock = PropertyLocks[slot];
slotlock.lock();
id value = objc_retain(oldValue);
slotlock.unlock();
return objc_autoreleaseReturnValue(value);
```

而其中
spinlock_t锁实际为
```
using spinlock_t = mutex_tt<LOCKDEBUG>;
```
而mutex_tt为
```
class mutex_tt : nocopy_t {
    os_unfair_lock mLock;
}
```
其内部是os_unfair_lock,iOS 10之后苹果推荐使用os_unfair_lock来代替不在安全的OSSpinLock

Atomic 是否线程安全？
Atomic 并不能保证线程安全，它只能提升正确率，atomic只是在属性的 Get/Set方法中赋值时添加了锁；
A，B，C三个线程同时发起修改访问属性时，并不能完全保证 A 线程写后读取到的值是 A 写入的值，
也无法保证直接使用 _xxx方式 访问实例变量，与使用 self. 的 Get/Set方法访问属性间获得正确的值。 

-------
##❏ block 种类，copy 用途

参考：https://www.jianshu.com/p/f0870fa95aac

Block根据存储位置分为：

NSGlobalBlock：（全局block），位于数据区。 
NSStackBlock：（栈block），位于栈区。 
NSMallocBlock：（堆block），位于堆区。

全局block的特点是未引用外部变量（auto变量），它不会受到copy的影响；
栈block,一般作为函数的参数，它引用了外部变量（auto变量），编译器会把它创建在栈上；
堆block，一般作为对象属性，由于栈block 只属于创建时的作用域，作用域结束，栈block 可能随时会被系统释放，
所以对栈block 进行copy 操作后将栈block 拷贝到堆上，变成了堆block，已防止 block 被提前释放，
所以在声明 block 作为属性的时候要使用 copy。 

-------
##❏ weak, retain, strong, copy...
Assign：只是简单赋值，不更改引用计数，适用于基础数据类型和C数据类型，主要存在于栈上；
Retain/Strong：指向并拥有该对象，引用计数+1。
Copy：创建一个引用计数为1的对象，主要用于拥有可变类型的不可变对象，以及用于 block；
Weak：弱引用，不更改引用计数，对象销毁后，访问对象得到nil，详见原理。


参考：https://clang.llvm.org/docs/AutomaticReferenceCounting.html#id10 中 4.1.1 Property declarations 介绍

assign implies __unsafe_unretained ownership.
copy implies __strong ownership, as well as the usual behavior of copy semantics on the setter.
retain implies __strong ownership.
strong implies __strong ownership.
unsafe_unretained implies __unsafe_unretained ownership.
weak implies __weak ownership.

With the exception of weak, these modifiers are available in non-ARC modes.



-------
##❏ 消息转发
+ (BOOL)resolveInstanceMethod:(SEL)sel;
是否要为对象处理调用（添加方法）

- (id)forwardingTargetForSelector:(SEL)aSelector;
将 Selector 转发给其他已实现该方法的对象

+ (NSMethodSignature *)instanceMethodSignatureForSelector:(SEL)aSelector;
- (void)forwardInvocation:(NSInvocation *)anInvocation;
生成方法签名，并转发


//类方法
+ (BOOL)resolveClassMethod:(SEL)sel;
- (id)forwardingTargetForSelector:(SEL)aSelector;
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;
- (void)forwardInvocation:(NSInvocation *)anInvocation;



-------
##❏ 浅拷贝，深拷贝
深复制：内容拷贝。源对象和副本指向的是不同的两个对象，源对象引用计数器不变，副本计数器设置为1

浅复制：指针拷贝，源对象和副本指向的是同一个对象，对象的引用计数器+1，相当于retain。

copy和mutableCopy：mutableCopy过对象类型改变都是深拷贝


-------
##❏ synthesize dynamic
@synthesize：
编译时，会自动生成 Get/Set方法。但手动添加方法优先。

@dynamic:
编译时，不会自动生成Get/Set方法。编译时无影响，但是如果未手动实现Get/Set，在调用Get/Set方法时，会由于为实现方法而崩溃。（unrecognized selector sent to instance） 

-------
##❏ NSInvocation
```
- (void)runLoginInvocation{
    NSString *account = @"lily";
    NSString *password = @"123123";
    
    NSMethodSignature  *signature = [[self class] instanceMethodSignatureForSelector:@selector(loginAccount:password:)];
//    NSMethodSignature  *signature = [NSMethodSignature signatureWithObjCTypes:"v@:@@"];//NSInvocation.h:_NSObjCValueType
    
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:signature];
    invocation.target = self;
    invocation.selector = @selector(loginAccount:password:);
    
    [invocation setArgument:&account atIndex:2];//Index从2开始 0 为 Target，1 为 selector
    [invocation setArgument:&password atIndex:3];
    
    [invocation invoke];//执行方法
//    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
//        [invocation invoke];//执行方法
//    });
}

- (void)loginAccount:(NSString *)account password:(NSString *)password{
    NSLog(@"account:%@ password:%@",account,password);
} 
```

-------
##❏ __block与__weak
__block不管是ARC还是MRC模式下都可以使用，可以修饰对象，也可以修饰基本数据类型。
__weak只能在ARC模式下使用，也只能修饰对象（NSString），不能修饰基本数据类型（int）。 __block对象可以在block中被重新赋值，__weak不可以。 

-------
##❏ 多线程 

-------
##❏ runtime 

-------
##❏ NSObject isa 元类 