# Use *shared_ptr* for Shared-ownership Resource Management

C++原始的手动生命周期管理(RAII)可以严格的控制变量的生命周期与资源管理。然而垃圾回收(garbage collection)机制是十分方便而且诱人的。*std::share_ptr*正是为了同时享受GC带来的方便与资源的可预测控制而设计的。

*std::share_ptr*表达的是*shared-ownership*语义，即多个指针共享同一个对象，协同处理对象的销毁。使用GC机制，使得用户端不再需要手动管理对象的生命周期及资源的释放，同时保证了销毁的确定性和可预测性。

## Reference Count

*std::share_ptr*使用*reference count*引用计数的方式，追踪管理对象的指针数量。当一个*std::share_ptr*被构造(除了移动构造)为指向某个对象，引用计数增加；当一个*std::share_ptr*被析构，引用计数减少；还有拷贝控制，也对引用计数产生影响。

移动操作使得旧*std::share_ptr*为空指针，不影响引用计数，所以移动操作比拷贝操作更加高效，包含构造、赋值的情形。


引用计数对*std::share_ptr*的性能有一定的影响：

- *std::share_ptr*比裸指针大一倍，因为增加了一个指向引用计数的指针。
- 为了多个指针能够访问引用计数，引用计数被动态创建在堆上。因为指向的对象无法储存这个计数。Item21中会解释使用*std::make_shared*来避免动态分配对性能的影响。
- 增加和减少引用计数的操作必须为原子操作。因为不同线程中，对引用计数的读和写有可能同时发生。因为原子操作比非原子操作更慢，所以*std::share_ptr*一般比裸指针性能要差。

## *std::share_ptr*'s *Control Block*

*std::share_ptr*默认采用*delete*作为*deleter*，但也支持自定义的*deleter*。但是与*std::unique_ptr*的设计不同，*deleter*的类型是*std::unique_ptr*类型的一部分，而*std::shared_ptr*不同：

    auto logingDel = [](Widget* pw) {
        makeLogEntry(pw);
        delete pw;  
    };

    std::unique_ptr<Widget, decltype(logingDel)> upw(new Widget, loggingDel);       // deleter type is part of ptr type.

    std::shared_ptr<Widget> spw(new Widget, logingDel);     // deleter type is not part of ptr type.

*std::shared_ptr*的设计更加灵活，考虑到两个*std::shared_ptr*可以拥有各自的自定义*deleter*:

    auto customDeleter1 = [](Widget* pw){...};
    auto customDeleter2 = [](Widget* pw){...};

    std::shared_ptr<Widget> pw1(new Widget, customDeleter1);
    std::shared_ptr<Widget> pw2(new Widget, customDeleter2);

因为两个*std::shared_ptr*具有相同的类型，那么它们就可以放进同一个容器：

    std::vector<std::shared_ptr<Widget>> vpw{ pw1, pw2 };

同样的，就可以对具有不同的*deleter*但指向对象相同的两个*std::shared_ptr*进行赋值操作。

另一个不同点在于，使用自定义*deleter*并不会影响*std::shared_ptr*的大小，它始终包含两个指针。这意味这不论*deleter*有多大，都不影响*std::shared_ptr*的大小，因为*deleter*是被动态分配的(可能就在堆上，取决于allocator)。

其实*std::shared_ptr*包含有两个指针，其中一个指向目标对象，另一个指向的不仅仅是引用计数，而是一块控制块(control block)。这个控制块中包含了Reference count、Weak count、Other Data(e.g. custom deleter if specified, allocator if specified...)。Control Block的具体实现可能涉及虚函数和继承。

控制块在第一个*std::shared_ptr*构造时创建。一般来说，构造*std::shared_ptr*指向一个对象，构造函数无法获知这个对象是否已经被其他指针指向，所以对控制块有以下约定：

- *std::make_shared*总是创建控制块。
- 当一个*std::shared_ptr*通过*std::unique_ptr*来构造时，总是创建控制块。
- 当一个*std::shared_ptr*通过一个裸指针来构造时，总是创建控制块。

当希望构造*std::shared_ptr*不会产生新的控制块时，使用*std::shared_ptr*或者*std::weak_ptr*作为构造函数的参数。使用*std::shared_ptr*的原则就是一个对象只有一个control block，一个对象对应多个control block将会反复销毁一个对象，导致未定义行为。

因此最好不要尝试使用裸指针初始化*std::shared_ptr*，使用*std::make_shared*替代(见Item21)，当使用自定义*deleter*时，无法使用*std::make_shared*，使用new直接替代，以防止产生裸指针。

一个十分容易使用裸指针去初始化*std::shared_ptr*的场景就是使用this指针初始化智能指针：

    std::vector<std::shared_ptr<Widget>> processedWidgets;  // processedWidgets keep track of Widgets.

    class Widget {
    public:
        ...
        void process() {
            ...
            processedWidgets.emplace_back(this);    // Lead to 'raw pointer constructor'.
        }
        ...
    }

这段代码可以通过编译，并且将this指针传入*std::shared_ptr*的构造函数，导致但对象多控制块，最终导致未定义行为。为了解决这个问题，引入*std::enable_shared_from_this*：

    class Widget : public std::enable_shared_from_this<Widget> {
    public:
        ...
        void process() {
            ...
            processedWidgets.emplace_back(shared_from_this());
        }
        ...
    };

*std::enable_shared_from_this*是一个基类模板，它提供一个*shared_from_this*的方法，能够返回包装了this的*std::shared_ptr*，从而避免裸指针this初始化*std::shared_ptr*。

这样的设计样式称作*The Curiously Recurring Template Pattern(CRTP)*，即奇异递归模板样式。即通过使基类的模板参数包含了继承类型的信息，使得基类成员函数能够实现普通OO不能实现的功能。

在C++17中，*std::enable_shared_from_this*中包含了一个*std::weak_ptr*用于追踪和记录控制块。在使用*shared_from_this*获得指针之前，需要确保控制块已经存在。所以*std::weak_ptr*必须在调用*shared_from_this*之前记录控制块。

## About *std::shared_ptr*

*std::shared_ptr*涉及到动态分配控制块、可能大的*deleter*和空间分配器、虚函数机制、原子操作，所以在性能上和裸指针有着较大的差距，因为不存在没有完美的解决资源管理的方案。

在普通场景中，*std::shared_ptr*使用默认的*deleter*和*allocator*以及使用*std::make_shared*，控制块只有3个字的大小，性能依旧良好；常用操作比如解除引用的开销不比裸指针大；涉及引用计数改变的操作可能包含了1到2个原子操作，可能比非原子操作消耗更大，但是对于单个计算机指令，依旧为单指令；还有虚函数机制只发生一次，即对象销毁的时候。

通过以上代价实现了资源管理的自动化，所以使用*std::shared_ptr*在大多数场景下是合适的；同时在不需要*shared—ownership*的情况下，使用*std::unique_ptr*会有更好的表现，而且*std::unique_ptr*转化为*std::shared_ptr*也十分方便，但牢记反向转换是不允许的，即使计数值为1.

同时，*std::shared_ptr*不具备代替原生数组的能力，因为*std::shared_ptr*所有API都是面向对象实现的，不存在*std::shared_ptr<T[]>*的特化。注意通过设置包含*delete[]*的可调用对象作为*deleter*，将*std::shared_ptr*指向原生数组可以通过编译，但这并不是一个好主意：一方面，*std::shared_ptr*不提供下标访问，只能通过指针算术来实现访问；另一方面，*std::shared_ptr*保持继承类-基类指针转化，对于数组来说这样的操作是未知的。使用容器类替代数组是更好的方案。

## Things to Remember

- *std::shared_ptr*提供了GC机制，用于管理资源和变量的生命周期。
- *std::shared_ptr*比*std::unique_ptr*更大，包含了控制块，要求原子性的引用计数操作。
- 默认析构操作采用*delete*，支持自定义*deleter*。*deleter*的类型不影响*std::shared_ptr*的类型。
- 尽量避免使用裸指针初始化*std::shared_ptr*。


