# Item28:Understand *Reference Collapsing*

*universal reference*当接收左值时，表现为左值引用，接收右值时，表现为右值引用。其背后的内在机制是*reference collapsing*，即引用折叠。

    template<typename T>
    void func(T&& param);

当param是一个左值时，T为左值引用；当param是一个右值时，T为非引用类型:

    Widget widgetFactory();
    Widget w;
    func(w);                    // T deduced to be Widget&.
    func(widgetFactory());      // T deduced to be Widget.

## *Reference Collapsing*

注意在C++中，定义实实在在的"引用的引用"是非法的：

    int x;
    auto& & rx = x;     // error.

但是在函数模板中，会发生*reference collapsing*引用折叠：

    template<typename T>
    void func(T&& param);

    func(w);        // T is Widget&, T&& is Widget& &&?

引用折叠的机制很简单，两个引用只要其中一个为左值引用，则折叠为左值引用；若两个引用均为右值引用，则折叠为右值引用：

    func(w);        // T is Widget&, T&& is Widget&(& && collapse to &).

再看看*std::forward*如何工作：

    Widget fparam;

    template<typename T>
    void f(T&& fparam) {
        someFunc(std::forward<T>(fparam));
    }

    template<typename T>
    T&& std::forward(remove_reference_t<T>& param) {
        return static_cast<T&&>(param);
    }

当*fparam*是一个左值时，T为Widget&，生成*std::forward<Widget&>*:

    Widget& && std::forward(Widget& param) {
        return static_cast<Widget& &&>(param);
    }

*reference collapsing*：

    Widget& std::forward(Widget& param) {
        return static_cast<Widget&>(param);
    }

函数返回一个左值引用，是一个左值(rvalue)。
当*fparam*是一个右值时，T为Widget，生成*std::forward<Widget>*:

    Widget&& std::forward(Widget& param) {
        return static_cast<Widget&&>(param);
    }

函数返回一个右值引用，是一个右值(消亡值xvalue)。

*reference collapsing*发生在4种语法环境中：其一就是上面的模板实例化；其二是*auto*生成类型时。因为auto的类型推断和template基本相同，所以*reference collapsing*也类似：

    auto&& w1 = w;  // auto is Widget&, auto&& is Widget&, reference collapsing happen.
    auto&& w2 = widgetFactory();    // auto is Widget, auto&& is Widget&&, no reference collapsing.

*universal reference*只是一个概念上的抽象，绕开繁杂的类型推断和引用折叠，所以*universal reference*不是新的引用，只是由以下两点原因作用而成的：

- 因为类型推断区分了*lvalue*和*rvalue*，前者推断为T&；后者推断为T。
- *reference collapsing*发生。

另外还有两种发生*reference collapsing*的场景：其三，使用*typedefs*和*alias declarations*；

    template<typename T>
    class Widget {
    public:
        typedef T&& RvalueRefToT;
        ...
    }

    Widget<int&> w;

    typedef int& && RvalueRefToT;   // reference collapsing happen.

    typedef int& RvalueRefToT;

其四，使用*decltype*：当涉及*decltype*的类型分析时，出现指向引用的引用，发生*reference collapsing*。

## Things to Remember

- *reference collapsing*在四种场景中发生：模板实例化，*auto*类型生成，*typedefs*和*alias declarations*，*decltype*
- *reference collapsing*发生时，其中一个引用是左值引用，则为左值引用；任意一个引用为右值引用，则为右值引用。
- *universal reference*的条件：类型推断区分了*lvalue*和*rvalue*，前者推断为T&；后者推断为T；*reference collapsing*。