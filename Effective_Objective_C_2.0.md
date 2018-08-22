Effective OC 2.0

# OC入门

1. 消息结构。
    * 消息结构，执行的代码由runtime决定；函数调用，由编译器决定
    * 函数调用，如果函数是多态的，函数调用需要用virtual table确定使用哪个实现。
    * 消息结构总在运行的时候才查找方法，编译器不在意接受消息的对象是谁，遇到问题也在运行时处理，称为“dynamic binding”

2. 尽量少的引入头文件
    * 如果不需要使用太多，在h文件中用@class声明即可，称之为“向前声明”，在m文件中再接入h，这样可以减少不必要的依赖关系，加速编译。同时，可以解决循环引用的问题。
    * 对于多数超类和接口的继承，头文件的引入是必须的。因此，这些定义最好独立，自成一体。
    * 特别的，对于delegate，没必要单独写一个头文件。与delegate的实现放在一起即可

3. 多用字面量。注意不应含有nil，只能创建Foundation框架的对象

4. 多用类型常量，而不是宏定义。
    * static可以把const变量局限在编译单元中.m，否则会创建外部符号，可能会导致冲突。
    * extern则会把对象注册到常量表中，并在数据段中储存。 extern最好有命名空间

5. 使用枚举表示状态（不重复）／选项（重复，利用left shift）
    * 利用typedef节省enum，利用冒号声明枚举变量的类型，从而使用向前声明。
    * 注意使用NS_OPTION宏和NS_ENUM宏。NS_OPTION 展开的时候，会考虑是否为C++模式，如果是，不能隐式转换枚举类型结果并且赋值。switch时，不应有deault

6. 理解property
    * 使用interface/public/private描述属性的坏处，会导致对象布局在编译期间就已经固定（偏移量）。因此，类定义修改后必须重新编译，否则会不兼容。
    * OC选择把实例变量当作存储偏移量的特殊变量，并交给类对象保管。偏移量会在runtime时查找，如果类定义改变，存储的偏移量也会改变。甚至可以在runtime中增加新的实力变量，这就是稳固的ABI应用程序二进制接口，ABI定义了生成代码时的规范。从而实例变量可以在extension和m文件中定义。
    * 还有一种解决方法，是避开实例变量，通过存取的方式。使用@property，会自动合成autosynthesis访问方法，该合成过程发生在编译时，所以ide看不到。使用@property，也会自动生成带下划线的实例变量，也可以使用@systhesis来手动定义实例变量的名字。如果甚至不希望自动生成getter和setter，可以使用@dynamic来重写，哪怕不重写，也没有编译器警告。

    * attribute
        * nonatomic／atomic：是否使用同步锁，默认原子有锁。
        * readwrite ／ readonly：readwrite时，如果有@systhesis实现，自动生成getter setter；readonly时，只有属性由@synthesis实现时，编译器才会合成真正的获取方法。
        * assign ／ copy ／ weak ／ strong ／unsafe_unretained：纯量，primary type，无引用计数
        * getter = <name>，定义特殊形式方法名
        注意在创建这些属性的时候，要符合它们自身的要求

7. 在对象内部，尽量直接使用实例变量
    * 使用属性的好处：KVO， property attributes，方便断点
    * 使用实例变量的好处：快
    * 折衷：写入靠属性，读取靠实例变量。特别注意，初始化时写入用实例变量。或者要懒加载，读取要靠属性触发特定动作。

8. 对象同等性。
    * isEqual => hash。hash还被用作collection的index，重写太多会有性能问题。对象放入collection中就不要修改了

9. 以类族模式隐藏实现细节。class cluster。oc中无法声明抽象类，但是如果基类没有init方法。。。那就是抽象基类了。继承抽象基类必须非常小心。合理实现。

10. 在已经存在的类中通过关联对象存放自定义数据。
    * 用于无法继承实现子类的情况，不同关联对象由key区分，存储对象时要指定内存管理方式objc_AssociationPolicy，来声明等效@property属性。
        * objc_setAssociatedObject(id object, void *key, id value, objc_AssociationPlicy policy)设置关联对象
        * objc_gettAssociatedObject(id object, void *key)获得关联对象
        * objc_removeAssociatedObject(id object)移除所有关联对象
        * 此处的key是不透明指针，无法访问内部属性，从而只有指针完全一致才能认为是同一个对象，所以要创建静态全局变量作为key。
    * 常用于将block或其他属性绑定到一个系统对象（无法定义初始属性，无法继承创建子类）上。

11. objc_msgSend
    * C语言使用静态绑定的方法调用函数。在编译期即可决定运行时调用的函数，函数地址被硬编码到指令中。如果无法在编译时确定，则需要动态绑定。
    * OC中，向对象传递消息，就是使用的动态绑定。
    * Id returnValue = [someObject messageName:parameter]
        * someObject receiver
        * messageName: selector
        * messageName:parameter message
    * 等效于 Id retrunValue = objc_msgSend(someObject, @selector(messageName:), parameter);
    * 先在receiver的类／父类中查找方法，找到后转跳到其IMP，并缓存到 fast map，不然就做message forwarding。
    * 在此过程中会有tail call optimisation，减少创建stack frame的数量

    * message forwarding
        * \+ (BOOL) resolveInstanceMethod:(SEL)selector 如果出现了不认识的 selector，该类有机会动态处理／准备处理，一般需要有了IMP，重新绑定SEL。
        * \- (id)forwardingTargetForSelector:(SEL)selectore 用于把SEL交给其他对象处理，目前为止，消息本身无法被修改。
        * \- (void)forwardInvocation:(NSInvocation *)invocation 完整转发。

12. KVC和KVO
    * addObserver: forKeyPath: options: context:
    * \- (void)observeValueForKeyPath:(NSString *)keyPath ofObject(id)object change:(NSDictionary *)change context:(void *) context
    * 合理删除不用的监听

13. 用method swiziiling 调试“黑盒方法”
    * (void)Method_exchangeImplementations(Method m1, Method m2)
    * (Method)class_getInstanceMethod(Class cls, SEL sel)
