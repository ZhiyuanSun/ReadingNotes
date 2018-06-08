# ARC
Compile option -fobjc-arc
## Ownership qualifiers
### __strong
Default when not explicitly decleared.

```oc
//Pay attension to the lifecycle of object, add release at the end of scoop / or when RC is 0
//No ARC:
{
    id obj = [[NSObject alloc] init];//RC++
    [obj release];
}

//With ARC:
{
    id __strong obj = [[NSObject alloc] init];
}
```

__strong / __weak / __autorelase will make object initilized to be nil without assignment.

### __weak
Solve the circular reference.

```oc
//Example for circular reference.
@interface Test : NSObject {
    id __strong obj_;
}
- (void)setObject:(id __strong)obj;
@end

@implementation Test
- (id)init {
    self = [super init];
    return self;
}

- (void)setObject:(id __strong)obj {
    obj_ = obj;
}

// Circular reference.
{
    id test0 = [[Test alloc] init];
    id test1 = [[Test alloc] init];
    [test0 setObject:test1];
    [test1 setObject:test0];
}

//Even worse
{
    id test = [[Test alloc] init];
    [test setObject:test];
}
```

__weak variable cannot have ownership of the object, the object mush be retained somewhere else.
Otherwise it might got disposed after the value was assigned.

ARC下，__strong 声明的变量会在调用函数返回时，等效的非 ARC 代码 在函数外接管内存所有权。\__weak声明的变量则无法在函数外接管,因为已经在方法内部由 autoreleasepool 接管。

### __unsafe__unretained
Excluded from ARC mechanism.
Similar behavior when assign with value, but will not guarantee the variable being set to nil when the referenced object is disposed.

### __autorelase
No explicitly declearation, but working implicitly.
```oc
//No ARC
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
id obj = [[NSObject alloc] init];
[obj autorelease];
[pool drain];

//With ARC
@autoreleasepool {
    id __autorelasing obj = [[NSObject alloc] init];
    //assignment equals to method call
}
```
不必显示地使用此修饰符， __autorelasing，因为：
```oc
@autoreleasepool {
    id __strong obj = [NSMutableArray array];
    /*
    * id obj = objc_msgSend(NSMutableArray, @selector(array));
    * objc_retainAutoreleaseReturnValue(obj);
    * objc_release(obj);// due to __strong
    */
}

// arrray 方法
+ (id) array {
    id obj = [[NSMutableArray alloc] init];
    return obj;
    /*
    * id obj = objc_msgSend(NSMutableArray, @selector(alloc));
    objc_msgSend(obj, @selector(init));
    return objc_autoreleaseReturnValue(obj);
    */
}
```
1. 编译器会在ARC下，将非初始化函数(NOT init/copy/new/mutableCopy)内部中被 return 的对象自动注册到 autoreleasepool.这种过程由编译器检测 return 所属的方法名来自动完成。
    1. 实际上是在 `return obj;` 处执行了 `return objc_autoreleaseReturnValue(obj);`，该方法会将所有权向非初始化函数外部转移，参见[此链接](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-autoreleasereturnvalue)
    2. 对应外部 objc_retainAutoreleaseReturnValue，这两个函数连续出现时（fully ARC），obj 不会被注册到 autoreleasepool，而是直接转交到外界。正常来说只有一个被出现，才会真正执行。
    3. More details: [How does objc_retainAutoreleasedReturnValue work?](http://www.galloway.me.uk/2012/02/how-does-objc_retainautoreleasedreturnvalue-work/) and [关于__autoreleasing，你真的懂了吗？](https://blog.csdn.net/junjun150013652/article/details/53149145)

2. 访问 __weak 变量，得到的一定是注册在 autoreleasepool 中的对象。此举是为了保证该对象不会在访问过程中废弃。
```oc
//This

id __weak obj1 = obj0;
NSLog(@"class=%@", [obj1 class]);

//is equivalent to this

id __weak obj1 = obj0;
id __autorelasing tmp = obj1;
NSLog(@"class=%@", [tmp class]);
```
3. id 和 对象的指针，多重指针存在时会附加 __autorelasing。也就是重新创建类似上面的中间变量。