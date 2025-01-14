本篇文章为我在日常coding过程中使用OC进行了一些骚操作或者被虐得很惨的记录，可能会记得比较乱，因为有时候我也不知道应该怎么分类，但是我会努力哒！(๑•̀ㅂ•́)و✧

1. 实例对象所属的类成为类对象，而类对象所属的类被成为元类`MetaClass`。

  类的类型为`Class`类型，而`Class`类型为一个`objc_class`结构体类型指针。该结构体大致成员变量如下所示：
  ```c

  struct objc_class {
    Class isa;
    Class super_class;
    const char *name
    long version;
    long info;
    long instance_size;
    struct objc_ivar_list *ivars;
    struct objc_method_list *methodLists;
    struct objc_cache *cache;
    struct objc_protocol_list *protocols;
  }

  ```  
  * **isa** 指针是和Class同类型的objc_class结构体指针，类对象的指针指向其所属的类，即元类（上文已经说过啦~），元类中存储着类对象的类方法，当访问某个类的类方法时会通过该isa指针从元类中寻找方法对应的函数指针；
  * **super_class** 为该类所继承的父类对象，如果该类已经是最顶层的根类则为NULL（比如NSObjc或NSProxy）；
  * **ivars** 是一个指向objc_ivar_list类型的指针，用来存储每一个实例变量的地址；
  * **info** 为运行期使用的一些标识，比如`CLS_CLASS(0x1L)`该类为普通类，`CLS_META(0x2L)`表示该类为元类；
  * **methodLists** 为存放该类的方法列表，根据info中的标识信息，存储的方法可为实例方法或类方法；
  * **cache** 用于缓存最近使用的方法。系统在调用方法时先去cache中search，为空时才回重新去methodLists中寻找。

2. 实例对象是类对象`alloc`或`new`操作创建的，该操作会拷贝实例所属的类成员变量，但并不拷贝类定义的方法。
  * 调用实例方法时，系统会根据`isa`指针去类和父类的方法列表`methodLists`中寻找与发送的消息对应的`selector`指向的方法。
  * 任何带有以指针开始并指向类结构的结构都可以被看做`objc_object`。
  * 在OC中（或者一个面向对象语言）一个对象最重要的特点是可以给它发送消息。

  总结以上两点，我做了一张图说明，希望能帮助大家稍微的理清这其中的关系，如下：

  <img src="https://i.loli.net/2018/04/20/5ad9a2650b34d.jpeg" width = "100%" height = "100%" align=center />

  * 当发送消息给实例对象时，“消息”是在寻找这个对象的类的方法列表。当发送消息给类对象时，“消息”是在寻找类的元类方法列表。

  * 元类，其也是一个对象，我们同样也可以调用它的方法。所有的元类都使用根元类作为他们的类，那根元类的元类呢？对，就是根元类的元类就是它们自己，其`isa`指针指向它自己。此处再放一张图来帮助大家梳理一遍这其中的关系，如下所示：

    <img src="https://i.loli.net/2018/04/20/5ad9a48da9cdc.jpeg" width = "100%" height = "100%" align=center />

    * 实例对象的`isa`指针指向该实例对象的类，类的`isa`指针指向元类；
    * 类的`super class`指向其父类，上文已经说了，如果该类为根类，则其父类为nil；
    * 元类的`isa`指针指向根元类（注意不是父元类），若该类本身就是根元类，则指向其本身。
    * 元类的`super class`指向父元类，若该类为根元类则指向该根类。

3. `id`类型。其为`objc_object`结构类型的指针，该类型的对象可转换为任何一种对象，类似C语言中的`void *`。

4. `@property` = `ivar` + `gettter` + `setter`，即属性是添加了 **存取方法** 方法的成员变量。

5. 针对`String`的`copy`和`strong`的理解。
  * 若为可变数据类型，即当前为`NSMutableString`，分别设定`copyString`和`strongString`，进行的赋值操作如下所示：
  ```objc
    @property (copy) copyString;
    @property (strong) strongString;

    NSMutableString *string = @"2333";
    copyString = string;
    strongString = string;

    [ts appendString:@"4666"];

    NSLog(@"%@", copyString); // 2333
    NSLog(@"%@", strongString); // 23334666
  ```
  由上可见，当用`strong`修饰可变数据类型`NSMutableString`时，其会因为原始数据的值的改变而改变。

 * 当为不可变数据类型，即`NSString`时，分别设定`copyString`和`strongString`，进行如上所示操作时两者均不会改变，来看`copyString`的setter方法实现：

     ```ObjC
     - (void)setCopyString:(NSString *)copyString {
       [_copyString release];
       // 拷贝了参数内容，创建了一块新的内存
       _copyString = [copyString copy];
     }
     ```

     接下来看`strongString`的setter方法实现：
   ```ObjC
     - (void)setStrongString:(NSString *)strongString {
       [_strongString release];
       // copy了指针
       [strongString retain];
       _strongString = strongString;
     }
 ```

6. `#import`和`#include`的区别
  * `#import`确保引用的文件只会被引用一次，不会引起交叉编译；
  * 两者均把后边的文件名所代表的文件拷贝到指令所在的文件；
  * `#import`会链入该头文件的全部信息，包括实例变量和方法；
  * `@class`只告诉编译器其后跟内容为类的名称，不用管该类是如何定义的，且一般在头文件中使用。
  * 使用`#import`的优点：可解决头文件中的循环依赖问题

7. `nonatomic`和`atomic`
  * `atomic`：默认值。只有一个线程可以访问，至少在当前线程的读取是安全的，但由于使用 **同步锁** 开销过大，会损耗性能（macOS性能较好，可不考虑该问题），其是一个原语操作，编译器会自动生成一些互斥加锁的相关代码，避免变量读写不同步等的问题。保证必须当前一个线程执行完相关的`setter`方法后，另一个线程才执行`setter`；
  * `nonatomic`：不保证`setter/getter`的原语执行，故可能会取到不完整的值。因此我们可以得到一个约束：**多线程环境下的原子操作是非常非常非常必要的！！！**
  * 解释两个概念：原语和原语操作，
    - **原语**：内核或微内核提供核外调用的过程或函数，是一段用机器指令编写的、完成特定功能的程序代码，且执行过程不允许中断；
    - **原子操作**：在多进程（或线程）的操作系统中不能被其它进程（或线程）打断的操作。当该操作不能完成时，必须回到该操作之前的状态，原子操作是不可拆分的，原子操作是中断且安全的。其本质实现还用到了 **自旋锁** ，自旋锁大概的意思是，当被其它对象使用时，待使用对象一直在循环等待并查看是否被释放。

8. `@Property`关键词及其相关关键字的理解：
  * 根据被修改的可能性，、@Property中关键字的排列推荐为：原子性、读写性、内存管理特性；
  * **原子性：** automatic和nonautomatic。决定了该属性是否为原子性的，即在多线程的操作中，不能被其它线程打断的特性，一旦使用了该变量的操作不能被完整执行时，将会回到该变量操作之前的状态，但原子性即automatic因为是原语操作（保证setter/getter的原语执行），会损耗性能，在iOS开发中一般不用，而在macOS开发中随意。
  * **读写性：** readOnly和readWrite。默认为readWrite，编译器会帮助生成serter/getter方法，而readOnly只会帮助生成getter方法。 // 此处可拓展，非要修改readOnly修饰的变量怎么办，可用KVC，又可继续拓展KVC相关知识。
  * **内存管理特性：** assign、weak、strong、unsafe_unretained。
    - assign：一般用于值类型，比如int、BOOL等（还可用于修饰OC对象）；
    - weak：用于修饰引用类型（弱引用，只能修饰OC对象）；
    - strong：用于修饰引用类型（强引用）；
    - unsafe_unretained：只用于修饰引用类型（弱引用），与weak的区别在于，被unsafe_unretained修饰的对象被销毁后，其指针并不会被自动置空，此时指向了一个野地址。

9. `Block`的理解：
  * Block与函数指针非常类似，但Block能够访问函数以外、词法作用域以外的外部变量的值；
  * Block不仅实现了函数的功能，还携带了函数的执行环境；
  * Block实际上是指向结构体的指针；（可参考[这篇文章](https://www.cnblogs.com/yoon/p/4953618.html)）
  * Block会把进入其内部的基本数据类型变量当做常量处理。】
  * Block执行的是一个回调，并不知道其中的对象合适被释放，所以为了防止在使用对象之前就被释放掉了，会自动给其内部所使用的对象进行retain一次。
  * Block使用`copy`修饰符进行修饰，且不能使用`retain`进行修饰，因为`retain`只是进行了一次回调，但block的内存还是放在了栈空间中，在栈上的变量随时会被系统回收，且Block在创建的时候内存默认就已经分配在栈空间中，其本身的作用域限于其创建时，一旦在超出其创建时的作用域之外使用，则会导致程序的崩溃，故使用`copy`修饰，使其拷贝到堆空间中，block有时还会用到一些本地变量，只有将其copy到堆空间中，才能使用这些变量。

10. 循环引用的几种情况：
  * **NSTimer**：
  * **block**：
  * **delegate**：

11. Objective-C中的反射机制
  Foundation框提供了反射API，可通过API把字符串转为`SEL`操作，且因为OC的动态性，这些操作都发生在**运行时**。我们可以在运行时选择需要创建的实例，并动态的选择调用方法，这些操作可以由服务器下发的参数进行控制。

  什么意思呢？比如说，我们可以通过后台推送过来的数据进行动态跳转，跳转到页面后再根据返回的数据执行对应的操作。比如，
  ```json
  // 假设返回了这些数据
  {
      "vc" : "homeViewController",
      "methon" : "reloadData",
      "propertys" : [
        {"url" : "www.baidu.com"},
        {"title" : "百度"},
      ],
  }
  ```
  我们可以这么写，
  ```ObjC
    Class class = NSClassFromString(dict[@"vc"]);
    UIViewController *vc = [[class alloc] init];
    NSDictionary *parameter = dict[@"propertys"];
    [parameter enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull key, id  _Nonnull obj, BOOL * _Nonnull stop) {
        if ([vc respondsToSelector:NSSelectorFromString(key)]) {
            [vc setValue:obj forKey:key];
        }
    }];
    [self.navigationController pushViewController:vc animated:YES];
    SEL selector = NSSelectorFromString(dict[@"method"]);
    [vc performSelector:selector];

  ```

12. iOS 中有以下几种随机数取法：
  * `random()` ：伪随机数，需要自己加种子，否则每次产生的随机数都是一样的。，一般都是以当前时间做种子。不推荐；
  * `arc4random()` ：真随机数，但如果要获取一定范围内的随机数，需要自己做模运算；
  * `arc4random_uniform()` ：真随机数，只需要填入范围上边界数字即可获取到对应范围内的真随机数。

持续更新中.....

13. 如果在 `category` 中声明了一个和原类同名的方法，或和该类的另一个 `category` 中的方法同名，那么在运行时究竟哪个方法会被执行是不确定的。

14. 用 `Objective-C` 编写的程序不能直接编译成可令机器读懂的机器语言，而是在程序运行时，通过运行时（runtime）把程序转译成可令机器读懂的机器语言；用 `C++` 编写的程序，编译时就直接编译成了可令机器读懂的机器语言。这也就是为什么把 `Objective-C` 视为一门动态开发语言的原因。

15. 开发语言的三个不同层次：
* 传统的面向过程的语言开发。
* 改进的开发面向对象的语言。
* 动态开发语言。

16. **在头文件中尽量减少其他头文件的引用**

* 通过 `#import` 修饰符来建立被引用类的指针。用 `#import` 建立类之间的复合关系时，也暴露类所引用类的实体变量和方法，可以使用 `@class` 来告诉编译器：这是一个类。
* `#import` 引用类同一个头文件，或者这些文件是依次引用的，如 A->B、B->C、C->D，当最开始的那个头文件有变化时，后面所有引用它的类都需要重新编译，如果自己的类有很多的话，这将耗费大量时间，使用 `@class` 则不会。
* 注意「类循环引用」的问题。

16. 尽量使用模块方式与多类建立复合关系
* `#include` 做的事情其实就是简单的复制、粘贴，将目标 `.h` 文件中的内容一字不落的复制到当前文件中。
* `#import` 实际上与 `#include` 是一样的，不过 `Objctive-C` 为了避免重复引用可能带来的编译错误（比如 B 和 C 都引用类 A，D 又同时引用了 B 和 C，这样 A 中定义的东西就在 D 中被定义了两次，造成重复）而加入了 `#import`，保证每个头文件只会被引用一次。`#import` 的实现是通过对 `#ifdef` 一个标志进行判断，然后再引入 `#define` 这个标志来避免重复引用。
* **使用 `.pch` 预编译头文件**不实用。确实能缩短编译时间，但在工程中不能随处访问的东西，却都暴露了。
* 使用模块（`Modules`）来解决。在 `Build Settings` 中搜索 `Modules` 修改为 `YES`。默认情况下，模块功能在所有的新工程中都是开启的，语法修改如下：
    ```objc
    @import UIKit;
    @import MaKit;
    ```
* 如果只导入框架中自己需要的部分可以这么做：
  ```objc
  @import UIKit.UIView;
  ```
* 在技术上，我们不需要把所有 `#import` 都换成 `@import`，因为编译器会隐式的转换它们，但建议尽可能的使用新语法。

17. 尽量避免使用 `#define`，`#define` 预处理指令不包含任何的类型信息，仅仅是在编译前做替换操作，它们在重复定义时不会发出警告，容易在整个程序中产生不一样的值。

18. 处理隐藏的返回类型，优先选择 `instancetype` 而非 `id`。

19. `NSLog` 并不是向 Xcode 控制台中输出信息，而是向苹果系统日志（Apple System Log，ASL）中输出错误信息。因此我们要把 `NSLog` 看作是 `printf` 和 `syslog` 的结合体：在调试时将消息发送到 Xcode 控制台，在设备上运行时将消息发送到系统全局日志。然后 `NSLog` 记录的数据就可以被任何拿到物理设备的人获取。在发布应用之前把 `NSLog` 从代码中删除。
- 在发布版本中禁用 `NSLog`：
  ```objc
  #ifdef DEBUG
  #define NSLog(...) NSLog(__VA_ARGS__);
  #else
  #define NSLog(...)
  #endif
  ```
20. `%x` 和 `%n` 分类符对攻击者来说非常有用。

21. `strcpy` 缓冲区溢出攻击。如果输入超出了固定字符长度的字符，超出的那部分字符就会覆盖相邻栈变量的内存，这就意味着，攻击者可以重载函数的返回地址，让程序执行恶意代码，攻击者可以把恶意代码直接放在输入当中，也可以放在内存中的其它位置。

22. 防止 `XSS` 攻击：
  - 设置黑名单。告知用户哪些字符不允许输入。
  - 设置白名单。只允许输入哪些字符。
  - 显示字符时，选转为 `HTML` 字符串，再输出，保证了 `<` 和 `>` 等特殊字符被转译。

23. 空指针（NULL 指针）是指没有存储任何内存地址的指针。野指针，是指向“垃圾内存”的指针。

24. `@autoreleasepool`。在 ARC 下，没有办法手动通知系统对某个对象执行 `autorelease`，当给一个对象设置了 `__autorelease` 修饰符修饰时，相当于这个对象在 MRC 下给这个对象发送了 `autorelease` 消息，注册到了 `autorelease pool` 中。

25. 指针地址对齐。为了加快内存的 CPU 访问，包括 macOS 和 iOS 在内的几乎所有系统架构都使用了**指针地址对齐**概念，其指在分配堆中的内存时往往采用**偶数倍**或以 2 为倍数的内存地址作为地址边界。

26. **标记指针**。由于指针地址对齐和 64 位超大地址的出现，指针地址仅仅作为内存的地址比较浪费，故可以在指针地址中保存或附加更多的信息，进而引入了**标记指针**的概念。其指的是那些指针中包含特殊属性或信息的指针。

27. 利用标记指针处理 `NSNumber`，直接可以把实际的值保存到指针中，而无须再去访问堆中的数据，提高内存访问速度和整体运算速度。

28. 标记指针堆 `isa` 指针的优化。在 OC 中所有的类都继承自 `NSObject`，因此每个对象都有一个 `isa` 指针指向它所属的类。在 32 位和 64 位环境下， `isa` 指针会产生不同的变化。
  - 在 32 位环境下，对象的引用计数都保存在一个外部的表中，对引用计数的增减操作都要先锁定这个表，操作完成后才解锁，效率比较慢。
  - 在 64 位环境下，`isa` 指针也是 64 位，实际作为指针的部分只用到其中的 33 位，剩余的部分会运用到**标记**指针的概念。其中的 19 位将保存对象的引用计数，这样对引用计数的操作只需要**原子**的修改这个指针即可。如果引用计数超过 19 位，才会将引用计数保存到外部表，情况较少，故效率可以大大提高。

29. 兼容 32 位和 64 位环境下代码编写事项（其实没啥用了）。
  - **不要将长整型数据赋予整型**。
  - **善用 `NSInteger` 来处理 32 位和 64 位之间的转换**。`NSInteget` 在 32 位运行时是 32 位整数，在 64 位运行时是 64 位整数。
  - **创建数据结构要注意固定大小和对齐**。
  - **选择一种紧凑的数据表示类型**。

30. 常量字符串和一般字符串的区别。
  - 由于编译器的优化，相同内容的常量字符串的地址值是完全相同的。
  - 如果使用常量字符串来初始化一个字符串，那么这个字符串也将是相同的常量。
  - 对常量字符串永远不要 `release`。

31. 在访问集合时要优先考虑使用快速美剧。
  - 使用快速枚举，枚举更安全。因为枚举会监控枚举对象的变化，如果在枚举的过程中枚举对象发送变化会抛出一个异常。
  - 多个枚举可以同时进行，因为在循环中被循环对象是禁止修改的。

32. 同一数组（`NSArray`）可以保存不同的对象，但不能存储 `float`、`int`、`double` 等基本类型和 `nil`，否则存储基本类型都会被设置为 0，不能存储 `nil` 是因为数组必须用 `nil`。

33. `autorelease pool` 提供一种机制：让对象延迟 `release`。这个对象放弃所有权，但又想避免立即释放（如何函数的返回值）。有些时候，可能会使用自己的 `autorelease` 池块。
  - 通常情况下，应该使用 `release`，而不是 `autorelease`，只有在不适合立即回收对象的情况下，才应该使用 `autorelease`。
  - 当返回一个新创建的（拥有）的对象时，应该使用 `autorelease` 而不是 `release` 来释放所有权。
  - 对于拥有 `alloc` 返回的对象而言，失去释放所有权之前，应先失去对该对象的引用。

34. 对象的 `isa` 实例变量指向对象的类。

35. `alloc` 和 `init` 不仅进行对象的内存分配，还要对它的 `isa` 实例变量和 `retain count` 初始化。

36. 对象销毁或者被移除一定考虑所有权的释放。
  - 从集合中移除对象，集合要释放对被移除对象的所有权。
  - 防止出现父对象被释放前而子对象的所有权已经释放。
  - 释放对象前，要确保其他对象对该对象的所有权已经释放。

37. 编译指令。

    指令 | 含义
    ---- | ----
    `@private` | 变量只限于声明它的类访问
    `@protected` | 变量可以被声明它的类及继承该类的类使用。没有明确指定访问范围的变量默认为 `@protected`
    `@public` | 变量可以在任何位置访问
    `@package` | 变量可以在同一个 `framework` 中访问

38. 动态属性。在 OC 2.0 中增加了一个新的关键字 `@dynamic`，用于定义动态属性。动态属性相对于 `@synthesis` 不是由编译器自动生成 `setter` 和 `getter`，也不是由开发者自己写的 `setter` 或 `getter`，而是在运行时动态添加的 `setter` 和 `getter`。

    实现动态属性需要在代码中覆盖 `resolveInstanceMethond` 来动态添加 `name` 的 `setter` 和 `getter`。这个方法在每次找不到方法时都会被调用。`NSObject` 的默认实现就是抛出异常。

39. 在覆盖基类的方法决定是否调用 `super`，基于打算如何重新重写方法，可以注意以下亮点：
  - 如果打算**补充**基类实现的行为，调用 `super`。
  - 如果打算**替换**基类实现的行为，不调用 `super`。

40. 在 OC 中，所有的方法都是虚方法。实现纯虚方法依赖协议来实现。

41. 类的对象支持归档和解档，该类必须遵循 `NSCoding` 协议；必须实现对对象进行编码（`encodeWithCoder:`）和解码（`initWithCoder：`）的方法。

42. `KVC` 的实现原理主要是运用了 `isa-swizzling` 技术（类型混合指针机制），通过其来实现内部查找定位。`isa` 指针指向的是对象的类，这个类也是一个对象，有自己的权限，根据类的定义编译而来。类对象负责维护一个方法调度表，该表本质上是由指向类方法的指针组成的，类对象中还保留一个基类指针，该指针也有自己的方法调度表和基类，还有所有通过继承得到的公共和保护的实例变量。`isa` 指针对消息分发机制和 Cocoa 对象的动态能力很关键。

43. 在 `swift` 中可以使用 `extension` 对类的实现进行拆分，在 `ObjC` 中可以选择使用 `category` 对类的实现进行拆分。

44. 内省是对象揭示自己作为一个运行时对象的详细信息的一种能力，这些详细信息包括对象在继承树上的位置、对象是否遵循特定的协议，以及是否可以响应特定的消息。

45. `isEqual` 方法先检查指针的等同性，然后是类的等同性，最好调用对象的比较器进行比较。

46. 使用 `new` 创建对象时，实际发生了两个步骤：第一个步骤，为对象分配内存，也就是说对象活动存储其实例变了的内存快；第二步，自动调用 `init` 方法，初始化对象使其处于可用状态。没有被初始化的指针都是 `nil`。

47. 使用类拓展隐藏私有信息。

48. 父对象应该强引用子对象，子对象变量应该弱引用父对象。

49. 对一些不支持 `__weak` 引用的类，可通过 `Unsafe Unretained` 引用来暗度陈仓。

50. 类别的一些内容：
  - 子类体现了类的上下级关系，而类别是类间的平级关系。
  - 类别具有替换特性，如果类别方法与类内某个方法具有同样的方法签名，类别里的方法将会替换类的原有方法。
  - 类别是为类**增加外部方法**的话，类扩展是用做类的**内部拓展**。

51. 类簇。基于抽象工程模式，可以用于隐藏实现的详细细节，为调用者提供一个简单的接口。看《编写高质量代码：改善 OC 程序的 61 个建议》第 48 个建议。

52. `alloc` 方法使用应用程序默认的虚存区，区是一个按页对齐的内存区域，用于存放应用程序分配的对象和数据。除了分配内存之外，还做 了：
  - 将对象的保持数设置为 1。
  - 使初始化对象的 `isa` 实例变量指向对象的类。对象类是一个根据类定义编译得到的运行时对象。
  - 将其他所有的实例变量初始化为 0（或与 0 等价的类型，比如 `nil`，`NULL`，`0.0`）

53. 在创建对象时，通常应该在处理之前检查返回值是否为 nil。

54. 需要 OC 对象的存取器来帮助进行引用计数。

55. 通过调用 `[xxx setValueForKey:xxx]` 要比 `[xxx setValue:xxx]` 要慢得多，因为编译器无法检查传递给 `valueForKey:` 的字符串是否有效，同时效率也变成了原来的 5%，如果需要获取值的运行参数，则使用 `[xxx performSelector: xxx]` 是直接消息发送速度的 2 倍，比 `valueForKey:` 快 10 倍。

56. KVO 和 Cocoa 绑定是基于 KVC 的，其速度不会很快。

57. **OpenUDID 是什么？**实际上是跟着 app 走，每次重装 app 都会重新生成一个 id，一般都会把它放到 keychain 中进行系统级的持久化。

58. `NSUserDefaults` 实际上是在 Library 文件夹下生成一个 plist 文件，如果该文件太大，读取时会比较耗时，因为加载的时候是直接全部 load 到内存中。头条主端通过测试，200 多个缓存数据，通过符号断点 `+[NSUserDefaults standardUserDefaults]` 确定最早一次的  `+load()` 从执行到结束耗时 1.8ms

59. `mach_absolute_time` 获取当前时间的「纳秒」，需要 mach 库。

60. 忽略警告的大概做法:
	```swift
	#pragma clang diagnostic push
	#pragma clang diagnostic ignored "-Wunguarded-availability"
	[UNUserNotificationCenter currentNotificationCenter].delegate = [TTNotificationCenterDelegate sharedNotificationCenterDelegate];
	#pragma clang diagnostic pop
	```
  
61. 想要成为一个 `AppDelegate` 需要：
	* 继承 `UIResponder` 和 `UIApplicationDelegate` 协议
	* 在 `main.m` 中通过 `UIApplicationMain` 进行初始化

62. ARC 在「编译」的时候插入内存管理代码

63. bitcode。上传 app store 的时候实际上上传的是一个「平台无关的代码」，用户在下载 app 的时候 App Store 会根据用户的机型翻译成对应的机器代码进行下发。

64. Link 链接
* 解决依赖
* 确定地址引用
* Mach-O 结构
* 生成可执行文件

65. Xcode 提示一个符号找不到声明是在「语法解析生成 AST」时出错。

66. 打包提示`missing symbols`是在「链接」出错。

67. OC 中的 ARC 是在编译的**机器码**生成支持的。

68. 代码中使用了静态库中的某个方法，是在**链接**时确定符号地址的

69. OC 中在方法里跑另外一个 方法/代码块的做法：
	```objc
	- (void)_enterFullScreenWithAngle:(CGFloat)angle animted:(BOOL)animated {
		void (^animation)() = ^{
			
		};
		
		animation()
	}
	```