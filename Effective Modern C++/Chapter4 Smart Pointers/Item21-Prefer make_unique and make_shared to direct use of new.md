# Prefer *std::make_unique* and *std::make_shared* to Direct Use of *new*

*std::make_shared*和*std::make_unique*是三个*make*函数之二，还有一个函数是*std::allocate_shared*。*make*函数的作用是接收参数包，完美转发参数包给对象的构造函数用于动态内存申请，并且返回指向该对象的智能指针。*std::allocate_shared*工作与*std::make_shared*类似，但额外接收一个空间适配器用于分配动态空间。

## *make* Perform Better When Writting Exception-Safe Code

使用*make*比不使用*make*在构造智能指针时不同：

    auto upw1(std::make_unique<Widget>());      // With make.
    std::unique_ptr<Widget> uwp2(new Widget);   // Without make.

    auto spw1(std::make_shared<Widget>());      // With make.
    std::shared_ptr<Widget> spw2(new Widget);   // Without make.

首先，使用new的方式需要重复对象的类型，make方式则不：源文件中重复类型导致编译成本提高，可能导致目标代码膨胀和代码不一致，同时也提高了拼写成本。第二个原因，*make*异常安全，而new不是，见下：

    void processWidget(std::shared_ptr<Widget> spw, int priority);

    int computePriority();

    processWidget(std::shared_ptr<Widget>(new Widget), computePriority());      // Potential resource leak.

当编译器将上述代码编译为目标代码：运行时，函数的调用必须要在函数参数评估完成后进行，所以在调用processWidget之前，下述操作必然已经发生：

- new Widget已经计算，Widget必然在堆上创建。
- shared_ptr<Widget>已经构造完成，并且指向new出的Widget。
- computePriority已经调用完毕。

然而编译器并没有被要求依次进行上述操作，于是有可能发生下面的情况："new Widget"的调用必须在shared_ptr<Widget>构造前进行；但是computePriority可以发生在两件事情的中间，之后或者之前:

- 进行"new Widget"
- 进行computePriority
- 进行shared_ptr<Widget>的构造

这样的目标代码显然不是异常安全的，一旦computePriority抛出异常，"new Widget"必然泄漏，因为shared_ptr<Widget>还没有构造出来去接管这个对象。但是使用*make*是异常安全的:

    processWidget(std::make_shared<Widget>(), computePriority());      // Exception safe.

运行时，不论两个函数那个先运行，都是异常安全的。当computePriority先行，并抛出异常，对象尚未构造；当computePriority后运行，新对象始终被一个智能指针所接管。因此*make*函数在异常安全方面比"new"的表现更好。

## *std::make_shared* is More Efficient

使用*std::make_shared*使得编译器能够产生更小更快的代码：

    std::shared_ptr<Widget> spw(new Widget);

该代码看上去进行了一次内存申请，其实进行了两次，因为*shared_ptr*额外需要一个控制块的申请；该申请会在构造函数中进行，所以直接使用"new"会进行两次内存申请，一个供给对象，一个供给控制块。

    auto spw = std::make_shared<Widget>();

如果使用*std::make_shared*，则只需要一次申请：这是因为*std::make_shared*直接申请了一块内存块，储存对象和控制块，提高了运行代码的速度。对*std::make_shared*的效率分析，同样适用与*std::allocate_shared*。

## Circumstances Where *make* Shouldnot be Used

- *make*函数不能用于声明自定义*deleter*
- *make*函数不能用于*braced—initializer*  

如下代码，调用的是非initializer_list版本的构造函数：

    auto upv = std::make_shared<std::vector<int>>(10, 20);
    auto spv = std::make_unique<std::vector<int>>(10, 20);

当使用*braced-initialzation*必须使用"new"，而不能使用*make*，除了直接调用*initializer*版本的构造函数：

    auto inilist = { 10, 20 };
    auto spv = std::make_shared<std::vector<int>>(inilist);

以上两个场景就是对*std::unique_ptr*的限制，而对于*std::shared_ptr*还有更多限制。

有些类会重载它们的*operator new*和*operator delete*，那么全局的内存分配以及释放机制对这些类将不再适合。这个重载的实现往往只会申请一块对象大小的内存，只管理对象所需要的资源，而这样的实现是不适合std::shared_ptr的，因为申请的内存还要包含控制块的大小。所以*std::make_shared*使用类自定义的*operator new*和*operator delete*不是一个好主意。

*std::make_shared*将对象的控制块放在同一块内存中，当引用计数归零，对象被析构，但是内存块并没有被释放，因为控制块还没有被析构。控制块中包含了引用计数等等信息，在引用计数归零后，控制块不一定就被销毁，因为控制块还有其他的登记记录，即第二个计数*weak count*,这个计数中记录了多少个*std::weak_ptr*指向这个内存块。因为*std::weak_ptr*需要通过查询控制块的*reference count*来知道自己是否悬空，所以只要有*std::weak_ptr*指向控制块，那么控制块就不能被析构，控制块所在的内存块就不能被释放。

如果对象本身是十分巨大的，而且最后一个*std::shared_ptr*和最后一个*std::weak_ptr*的析构的时间差也很明显，那么对象析构以及内存释放之间就会存在滞后：

    class ReallyBigType {...};

    auto pBigObj = std::make_shared<ReallyBigType>();
    ... // Create std::weak_ptr and std::shared_ptr to obj.
    ... // final std::shared_ptr to obj and obj destroyed here.
        // but std::weak_ptr to it remain.
    ... // during this period, memory formerly occupied by large
        // obj remains allocate.
    ... // final std::weak_ptr to it desturyed here.
        // memory for control block and obj is freed.

但如果使用"new"，就不会存在这样的滞后，因为控制块和对象的内存块是分开的：

    std::shared_ptr<ReallyBigType> pBigObj(new ReallyBigType());
    ... // Create std::weak_ptr and std::shared_ptr to obj.
    ... // final std::shared_ptr to obj destroyed here.
        // obj is destoryed and memory for obj is freed.
        // but std::weak_ptr to it remain.
    ... // during this period, memory formerly occupied by large
        // obj remains allocate.
    ... // final std::weak_ptr to it desturyed here.
        // memory for control block is freed.

当然为了异常安全，在使用"new"直接生成智能指针，务必保证new出的裸指针马上被传给智能指针的构造函数，即该语句中不要再做其他事情来防止编译器产生顺序不合适的代码:

    void processWidget(std::shared_ptr<Widget> spw, int priority);

    void cusDel(Widget* ptr);

    processWidget(std::shared_ptr<Widget>(new Widget,cusDel), computePriority());     // Exception unsafe.

    std::shared_ptr<Widget> spw(new Widget, cusDel);
    processWidget(spw, computePriority())      // Safe. but not optimal.

将构造放在单独的一条语句中，可以避免异常不安全。即使构造函数抛出异常(比如控制块的申请出现异常)，可以保证cusDel用于析构对象，并且释放内存。

但是从性能影响上看，内存不安全版本更好，因为其传入构造的是一个右值，而异常安全版本是一个左值。右值构造使用的是move，而左值进行的是copy；而copy一个*std::shared_ptr*要求原子操作的引用增加，削弱性能表现。

    processWidget(std::move(spw), computePriority());   // both efficient and exception safe.

使用std::move可以兼顾性能与异常安全，但是原指针将被设置为空指针。

## Things to Remember

- 对比"new"，*make*函数减少代码膨胀，加强异常安全，*make_shared*和*allocate_shared*还有更好的性能表现和更小的代码。
- 不使用*make*的情形：需要使用自定义*deleter*或者*braced-initializer*。
- 对于*std::shared_ptr*，不使用*make*的情景还有：对象具有自定义的内存管理机制、(出于内存空间的考虑)对于非常大的类型同时*std::shared_ptr*和*std_weak_ptr*析构时间差很大的情况。
