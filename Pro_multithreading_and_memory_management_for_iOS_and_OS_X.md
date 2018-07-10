# Life Before ARC
## Reference counting
|Action for object|method|
|---|---|
|create|alloc/new/copy/mutableCopy|
|retain|retain|
|release|release|
|dispose|dealloc|

Principle for reference counting memory management:
* You have ownership of any objects you create.
* You can take ownership of an object using retain.
* When no longer needed, you must relinquish ownership of an object you own.
* You must NOT relinquish ownership of an object you don't own.

### You have ownership of any objects you create
YES:
alloc / new / copy / mutableCopy 
allocMyObject / newThatObject / copyThis / mutableCopyYourObject

NO:
allocate / newer / copying / mutableCopyed

### You can take ownership of an object using retain
```oc
id obj = [NSMutableArray array];
[obj retain];
```

### When no longer needed, you must relinquish ownership of an object you own
```oc
id obj = [[NSObject alloc] init];
[obj release];

// or 
id obj = [NSMutableArray array];
[obj retain];
[obj release];
```

More complex,
```oc
// Method has the ownership, and pass it to the caller. 不在方法内处理任何引用计数，处理权留在 method 外
- (id) allocObject {
    id obj = [[NSObject alloc] init];
    return obj;
}

// 引用计数已经被autorelease pool处理，在 method 外需要retain 才有所有权
- (id) object {
    id obj = [[NSObject alloc] init];
    [obj autrelease];
    return obj;
}
```

## alloc / retain / release / dealloc implementation

### alloc

Method stacks get called:
> -retainCount
    - __CFDoExternRefOperation
    - CFBasicHashGetCountOfKey

> -retain
    - __CFDoExternRefOperation
    - CFBasicHashAddValue

> -retain
    - __CFDoExternRefOperation
    - CFBasicHashRemoveValue

implementation:

```c
int __CFDoExternRefOperation(uintptr_t op, id obj) {
    CFBasicHashRef table = get hashtale from obj;
    switch (op) {
        case OPERATION_retainCount:
            count = CFBasicHashGetCountOfKey(table, obj);
        case OPERATION_retain:
            CFBasicHashAddValue(table);
        case OPERATION_release:
            count = CFBasicHashRemoveValue(table, obj);
            return 0 == count;
    }
}
```

Advantage of using hashtable compared to using headers:
* No alignment issue for header area.
* By iterating through the hash table entries, memory blocks for each object are reachable.

The latter one helps in debugging. When memory itself is broken, the area is reachable from the table.
Also it helps in checking for memory leak.

#### autorelease
Behave like automatic variable in C language:
```oc
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
id obj = [[NSObject alloc] init];
[obj autorelease];
[pool drain];
```

NSRunLoop会自动创建autorealse pool.

implementation:
```cpp
class AutoReleasePoolPage {

    static inline void *push() {
        // Creat and owns NSAutoreleasePool
    }

    static inline void pop(void *token) {
        // Disposal of NSAutoreleasePool
        releaseAll();
    }

    static inline autorelease(id obj) {
        // add object
        AutoReleasePoolPage *pool = get active AutoReleasePoolPage;
        pool->add(obj);
    }

    id *add(id obj) {
        // add object to array
    }

    void releaseAll() {
        // call release for all object in the internal array
    }
};

void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}
void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
id objc_autorelease(id obj) {
    return AutoreleasePoolPage::autorelease(obj);
}
```

Debugging method [NSAutoreleasePool showPools];

You cannnot autorelease an autorelease pool since it's method has been overriden.

# ARC
Compile option -fobjc-arc
## Ownership qualifiers
### __strong
Default when not explicitly decleared.

```oc
//Pay attension to the lifecycle of object, add release at the end of scoop / or when RC is 0
//No ARC:
{
    id obj = [[NSObject alloc] init];
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

__weak variable cannot have ownership of the object, it mush be assigned with a __strong variable. Otherwise it might got release after the value was assigned.

ARC下，__strong 声明的变量会在调用函数返回时，接管内存所有权。\__weak声明的变量无法接管。