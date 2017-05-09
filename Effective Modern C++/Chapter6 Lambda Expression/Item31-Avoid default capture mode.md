# Item31:Avoid Default Capture Mode

*lambda*表达式并没有给C++带来什么新的东西，但是极大的简化了工作，快速生成临时对象(比如*trivial predicates*)，对于STL的使用意义重大。关于*lambda*主要有以下几点概念:

- *lambda expression*：单纯的*lambda*表达式。
- *closure*：由*lambda*生成的运行时对象，包含有由捕捉模式规定的数据块。
- *closure class*：*closure*对应的类，由编译器自动生成。

## Capture Mode

- identifier：拷贝捕获
- identifier...	：拷贝捕获包
- identifier initializer  (C++14)：初始化拷贝捕获
- &identifier：引用捕获
- &identifier...：引用捕获包
- &identifier initializer	(C++14)：初始化引用捕获
- this：引用捕获母对象
- *this	(C++17)：拷贝捕获母对象

如果默认引用捕获，则特化捕获不能再是引用；如果默认捕获是拷贝，则特化捕获必须是引用或者*this，每个捕获只能出现一次。


## Avoid Default Reference Capture

C++11为*lambda*提供了两种捕捉模式：by-value和by-reference。默认的*by-reference*有可能导致悬空引用。这是因为一旦引用对象的生命比*lambda*要短，就会导致*lambda*内部的引用悬空：

    using FilterContainer = std::vector<std::function<bool(int)>>;      // a container class for callable object.

    FilterContainer filters     // container.
    filters.emplace_back(
        [](int value){ return value % 5 == 0; }
    );

如果*lambda*中的常数使用一个捕获数，捕获模式采用默认引用捕获：

    void addDivsorFilter() {
        auto calc1 = computeVal1();
        auto calc2 = computeVal2();

        auto divisor = computeDivisor(calc1, calc2);

        filters.emplace_back(
            [&](int value){ return value % divisor == 0; }      // divisor may be dangle!
        );
    }

默认采用引用捕获的一大缺点就是上述bug难以发现，因为没有明确divisor是一个引用：

    filters.emplace_back(
        [&divisor](int value){ return value % divisor == 0; }      // divisor may be dangle!
    );  

虽然这样的代码依旧是错误的，但是至少对divisor是一个引用有足够的提示。当然如果*lambda*执行时间得当如下述代码依旧是安全的：

    template<typename C>
    void workWithContainer(const C& container) {
        auto calc1 = computeVal1();
        auto calc2 = computeVal2();
        auto divisor = computeDivisor(calc1, calc2);
        if(std::all_of(
            std::begin(container), std::end(container), [&](const typename C::value_type& value){ return value % divisor == 0; })
        ) {
            ...
        }
    }

但如果*lambda*表达式如果希望被复用，经过copy到其他环境下使用，就会发生错误。
同时C++14支持了lambda的参数推断，所以参数列表可以简化：

    if(std::all_of(
        std::begin(container), std::end(container), [&](const auto& value){ return value % divisor == 0; })
    )

## Avoid Default Value Capture 

对之前的*lambda*的容器类使用默认值捕获，就避免了引用悬空的问题：

    filters.emplace_back(
        [=](int value){ return value % divisor == 0; }      // divisor may be dangle!
    );  

但是默认值捕获，可能会导致指针悬空：

    filters.emplace_back(
        [=](int value){ return value % *pdivisor == 0; }      // pdivisor may be dangle!
    );

当然不要使用裸指针，使用智能指针可以避免这个问题，但是我们不总是能够避免使用裸指针，最突出的情况就是this指针。假设有如下情况：

    class Widget {
    public:
        ...
        void addFilter() const;

    private:
        int divisor;
    };

    void Widget::addFilter() const {
        filters.emplace_back(
            [=](int value){ return value % divisor == 0; }      // pass compile, but not safe. 
        );
    }

我们无法捕获类的数据成员，只能捕获local object，所以以下代码是会报错的：

    filters.emplace_back(
        [divisor](int value){ return value % divisor == 0; }      // cannot capture divisor. divisor is a member.
    );

由于this是一个局部变量，使用默认捕获相当于捕获了引用捕获了this，编译器会使用this->divisor替代divisor：

    filters.emplace_back(
        [this](int value){ return value % divisor == 0; }      // not safe.
    );

那么如果对象Widget已经销毁，捕获的this悬空，再次调用容器中的*lambda*将会不安全。这种错误十分危险：

    void doSomeWork() {
        auto pw = std::make_unique<Widget>();
        pw->addFilter();
        ...                 // destory Widget; now filters holds dangling pointer.
    }

该问题的解决方案就是进行拷贝：

    void Widget::addFilter() const {
        auto divisorCopy = divisor;
        filters.emplace_back(
            [divisorCopy](int value){ return value % divisorCopy == 0; }     // copy the divisor.
        );
    }

C++17提供了拷贝捕获整个对象，这里不赘述。使用默认捕获十分危险，因为很容易导致一些难以发现的错误。

## Use Init Capture and Take Notice of Static 

C++14提供了初始化捕获：

    void Widget::addFilter() const {
        filters.emplace_back(
            [divisorCopy = divisor](int value){ return value % divisorCopy == 0; }     // copy the divisor.
        );
    }

*lambda*只能捕获automatic storage duration的变量，即，static变量lambda不会捕获：

    template<typename C>
    void workWithContainer(const C& container) {
        static auto calc1 = computeVal1();
        static auto calc2 = computeVal2();
        static auto divisor = computeDivisor(calc1, calc2);
        if(std::all_of(
            std::begin(container), std::end(container), 
            [=](const auto& value){ return value % divisor == 0; })
            // capture nothing.
        ) {
            ...
        }
        divisor++;
    }

## Things to Remember

- 默认引用捕获可能导致悬空引用。
- 默认拷贝捕获可能导致悬空指针(特别是this)，而且错误的指示*lambda*完全*self-contained*。