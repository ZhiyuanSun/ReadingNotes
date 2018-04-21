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