# When Using the *Pimpl* Idiom, Difine Special Member Functions in the Implementation File

## the *Pimpl* Idiom

*Pimpl*(pointer to implementation)是一门用于减少编译成本的技术：

    class Widget {          // In header "Widget.h"
    public:
        Widget();
        ...
    private:
        std::string name;
        Gadget g;           // User-defined type.
    }

Widget是一个拥有std::string, Gadget类型成员变量的类，定义在"Widget.h"中；任何需要调用Widget的源代码都需要包含这个头文件，在包含这个头文件的同时也就包含了声明Gadget的头文件，如果"Gadget.h"是一个经常改变内容的头文件，那么就大大增加了编译成本。

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ~Widget();
        ...
    private：
        struct Impl;        // Declare implementation struct.
        Impl *pImpl;        // and pointer to it.
    }

使用Pimpl，将数据成员封装进一个声明了的结构体(不定义)中，再用指针指向这个结构体。因为"Widget.h"中没有使用这些类型，所以就不用包含这些头文件，对这些头文件做修改不影响包含"Widget.h"的源代码，提高了编译效率。

但注意Impl是一个只声明而未定义的类型(*incomplete type*)，只有少数对它的行为是合法的，比如声明指向它的指针。以上代码只完成了声明，之后需要进行定义：

    #include "Widget.h"     // In impl. file "Widget.cpp".
    #include "Gadget.h"
    #include <string>

    struct Widget::Impl {   // Define Impl.
        std::string name;
        Gadget g;
    };

    Widget::Widget() : pImpl(new Impl)
    {}

    Widget::~Widget() {
        delete pImpl;
    }

通过*Pimpl*，分离了头文件"Widget.h"对于"Gadget.h"的依赖。于是当Gadget.h的内容发生变化时，只需要重新编译"Widget.cpp"，而不需要重新编译其它包含"Widget.h"的用户源代码，提高了编译期间的效率。

## Use *std::unique_ptr* instead of raw pointer

在*Pimpl*中，使用智能指针*std::unique_ptr*替代裸指针会产生一个编译错误*cannot delete an incomplete type*：

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ...
    private：
        struct Impl;        // Declare implementation struct.
        std::unique_ptr<Impl> pImpl;        // and *std::unique_ptr* to it.
    }

然后定义：

    #include "Widget.h"     // In impl. file "Widget.cpp".
    #include "Gadget.h"
    #include <string>

    struct Widget::Impl {   // Define Impl.
        std::string name;
        Gadget g;
    };

    Widget::Widget() : pImpl(std::make_unique<Impl>()){}    // Create std::unique_ptr.

因为*std::unique_ptr*可以自动管理资源，所以我们理应可以不再自行定义析构函数，而交给编译器自行实现，这是没有问题的，问题在于在哪实现。我们无法对一个*incomplete type*的变量进行如*delete*和*sizeof*之类的操作，编译错误意味着生成析构函数参与编译的位置处Impl还没有被定义。

对编译错误信息层层查看可以发现：编译器默认生成的析构函数时内联的，内联位置Impl还没有被定义，而析构函数调用*std::unique_ptr*的析构，*std::unique_ptr*的析构又会对Impl进行默认的*delete*，最后static_assert进行对象是否为*imcomplete type*的判断时出现了fail。

解决这个问题只要对析构函数进行显示的定义即可：

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ~Widget();          // Declare destructor.
        ...
    private：
        struct Impl;        // Declare implementation struct.
        std::unique_ptr<Impl> pImpl;        // and *std::unique_ptr* to it.
    }

然后定义：

    #include "Widget.h"     // In impl. file "Widget.cpp".
    #include "Gadget.h"
    #include <string>

    struct Widget::Impl {   // Define Impl.
        std::string name;
        Gadget g;
    };

    Widget::Widget() : pImpl(std::make_unique<Impl>()){}    // Create std::unique_ptr.

    Widget::~Widget() = default;    // Default destructor.

因为显式定义了析构函数，所以move操作都不会被编译器生成；所以如果需要支持move操作，需要显式定义：

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ~Widget();          // Declare destructor.
        
        Widget(Widget&& rhs) = default;     // Wrong.
        Widget& operator=(Widget&& rhs) = default;  // Wrong.
        ...
    private：
        struct Impl;        // Declare implementation struct.
        std::unique_ptr<Impl> pImpl;        // and *std::unique_ptr* to it.
    }

移动赋值函数需要销毁当前对象持有的Impl；移动构造函数默认生成的异常处理中包含了对Impl的销毁，因此上述代码不能够通过编译，处理方法同析构函数：

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ~Widget();          // Declare destructor.
        
        Widget(Widget&& rhs);
        Widget& operator=(Widget&& rhs);
        ...
    private：
        struct Impl;        // Declare implementation struct.
        std::unique_ptr<Impl> pImpl;        // and *std::unique_ptr* to it.
    }

然后定义：

    #include "Widget.h"     // In impl. file "Widget.cpp".
    #include "Gadget.h"
    #include <string>

    struct Widget::Impl {   // Define Impl.
        std::string name;
        Gadget g;
    };

    Widget::Widget() : pImpl(std::make_unique<Impl>()){}    // Create std::unique_ptr.

    Widget::~Widget() = default;    // Default destructor.
    Widget::Widget(Widget&& rhs) = default;
    Widget& Widget::operator=(Widget&& rhs) = default;

如果Impl中的成员支持copy操作，那么Widget也可以支持copy操作：1、因为编译器无法自行实现包含*move-only*类型成员的类的copy操作，2、即使编译器能够实现，也只能进行浅层拷贝，无法进行深层拷贝；所以需要手动实现copy操作：

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ~Widget();          // Declare destructor.
        
        Widget(Widget&& rhs);
        Widget& operator=(Widget&& rhs);
        Widget(const Widget& rhs);
        Widget& operator=(const Widget& rhs);
        ...
    private：
        struct Impl;        // Declare implementation struct.
        std::unique_ptr<Impl> pImpl;        // and *std::unique_ptr* to it.
    }

然后定义：

    #include "Widget.h"     // In impl. file "Widget.cpp".
    #include "Gadget.h"
    #include <string>

    struct Widget::Impl {   // Define Impl.
        std::string name;
        Gadget g;
    };

    Widget::Widget() : pImpl(std::make_unique<Impl>()){}    // Create std::unique_ptr.

    Widget::~Widget() = default;    // Default destructor.
    Widget::Widget(Widget&& rhs) = default;
    Widget& Widget::operator=(Widget&& rhs) = default;

    Widget::Widget(const Widget& rhs) : pImpl(nullptr) {
        if(rhs.pImpl) pImpl = std::make_unique<Impl>(*rhs.pImpl);
    }
    Widget& Widget::operator=(const Widget& rhs) {
        if(!rhs.pImpl) pImpl.reset();
        else if(!pImpl) pImpl = std::make_unique<Impl>(*rhs.pImpl);
        else *pImpl = *rhs.pImpl;
        return *this;
    }

copy操作的定义中需要考虑传入参数的情况、现对象的情况、指针为空的情况。总之，得益于编译器对Impl自动实现的一系列copy操作，使得函数的实现变得十分简单。

## When using *std::shared_ptr*

*Pimpl*的实现一般使用*std::unique_ptr*因为其指针逻辑很显然是独占的，但也可以考虑一下使用*std::shared_ptr*的情况：

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ...
    private：
        struct Impl;        // Declare implementation struct.
        std::shared_ptr<Impl> pImpl;        // and *std::unique_ptr* to it.
    }

然后定义：

    #include "Widget.h"     // In impl. file "Widget.cpp".
    #include "Gadget.h"
    #include <string>

    struct Widget::Impl {   // Define Impl.
        std::string name;
        Gadget g;
    };

    Widget::Widget() : pImpl(std::make_shared<Impl>()){}    // Create std::shared_ptr.

可以发现使用*std::shared_ptr*不再有上述繁杂的特殊函数定义，之所以如此：  
*std::shared_ptr*和*std::unique_ptr*的*deleter*的实现方式不同，*deleter*的类型是*std::unique_ptr*的一部分，所以*deleter*再编译期就可以连接上*std::unique_ptr*，这样具有更好的运行期空间时间效率；而*std::shared_ptr*不同，*deleter*其实是*std::shared_ptr*实例的一部分，会有更大的数据结构和运行期成本，但是*deleter*不必要在编译期就被连接上。

## Things to Remember

- *Pimpl*是一项通过减少编译头文件间(类用户代码和类实现)依赖来降低编译成本的技术。
- 对于使用*std::unique_ptr*的*Pimpl*，需要手动定义类特殊函数来支持实现，尽量使用编译器的默认实现。
- 对于*std::shared_ptr*没有上述要求。
