# Item7: Distinguish Between () and {} when creating objects

在modern C++中，初始化从语法上分类在大致有以下三种：parentheses、equals、braces。

    int x(0);       // Initializer is an int parentheses.
    int y = 0;      // Initializer follows '='.
    int z{ 0 };     // Initializer is in braces.

还可以见到：

    int z = { 0 }；      // Initializer uses '=' and braces.

这种情况在C++中和只用braces是一样的。

## Initialization is not Assignment

首先必须意识到，initialization并不是assignment。对于built-in类型，没有什么问题；但对于定义型的类型
区别赋值和初始化在于调用的函数的不同。

    Widget w1;          // Call default constructor.
    Widget w2 = w1;     // Not an assignment; calls copy constructor.
    w1 = w2;            // Assignment, calls copy assignment(operator=).

## Uniform Initialization
在C++98时期，没有语法支持一些初始化，比如STL容器内一系列值的初始化。
C++11为了解决这类问题，引入了一个uniform initialization，至少在概念上可以运用于所有初始化场景的初始化语法。在概念上可以称为"uniform initialization",在句法上可以称为"braced initialization"。
比如：

    std::vector<int> v{ 1, 2, 3 };      // Initialize the vector with a particular set                                       // of value.

Braced-initialization还可以用于类成员非static对象的默认初始化。

    class ex {
        ...
    private:
        int x{ 0 };     // Fine, braced-initialization.
        int y = 0;      // Fine, copy-initialization.
        int z(0);       // Wrong!
    };

另一方面，不可复制对象(uncopyable)也可以用braced-initialization。

    std::atomic<int> ai1{ 0 };      // Fine.
    std::atomic<int> ai2(0);        // Fine.
    std::atomic<int> ai3 = 0;       // Wrong! Connot copy-initialization.

从上面的例子可以看出braced-initialization在所有情况的初始化下都是适用的。

值得注意一点：braced-initialization是不允许进行隐式缩窄转换(implicit narrowing convertion)。
(但是在clang中可以啊。。。)

    double x, y;
    int sum1{ x + y};       // Error! Requiring a narrowing convertion. 
    int sum2(x + y);        // Fine.
    int sum3 = x + y;       // Fine.

还有一点有价值的内容是关于默认构造函数的。

    Widget w1(10);          // Calls the constructor with one argument 10.
    Widget w2();            // Declare a function that return type is Widget, Do not                             // Calls the default constructor.
    Widget w3{};            // Calls the default constructor.

可以看出braced-initialization防止了一些语法习惯带来的意外。

## Some Superising Behaviour About Braced-initialization

由于braced-initialization和std::initialize_list之间纠缠的关系，braced-initialization会出现一些出乎意料的情况。比如Item2中所言的auto问题（在C++17中有更改）。还有就是涉及到构造函数的调用问题：

    class Widget{
    public:
        Widget(int i, bool b);
        Widget(int i, double b);
        ...
    };

    Widget w1(10, true);        // Calls first ctor.
    Widget w2{10, true};        // Calls first ctor.
    Widget w3(10, 5.0);         // Calls second ctor.
    Widget w4{10, 5.0};         // Calls secong ctor.

如果说构造函数中没有涉及到任何initializer_list的parameter，那么{}和()的对于ctor的调用行为是一致的。
但是如果涉及了initializer_list的parameter，情况就不一样了：

    class Widget{
    public:
        Widget(int i, bool b);
        Widget(int i, double b);
        Widget(std::initializer_list<long double> il);  // Initializer_list parameter.
        ...
    };

    Widget w1(10, true);        // Calls first ctor.
    Widget w2{10, true};        // Calls third ctor. 
                                // 10 and true will convert to long double.
    Widget w3(10, 5.0);         // Calls second ctor.
    Widget w4{10, 5.0};         // Calls third ctor. 
                                10 and 5.0 will convert to long double.

在这种情况下，w2和w4会优先调用新的构造函数，即使该构造函数显然没有其他函数更加匹配传入参数的形式。
甚至拷贝和移动构造函数也会被Initializer_list-parameter-ctor劫持。

？？？此处貌似和程序验证不符合，未查明原因(MSVC和clang均不符合)

    class Widget{
    public:
        Widget(int i, bool b);
        Widget(int i, double b);
        Widget(std::initializer_list<long double> il);  // Initializer_list parameter.
        operator float() const;         // Convert to float.
        ...
    };

    Widget w5(w4);              // Calls copy-ctor.
    Widget w6{ w4 };              // Calls Initializer_list-parameter-ctor.
                                // w4 converts to float through operator float()
                                // then converts to long double.
    Widget w7(std::move(w4));       // Calls move-ctor.
    Widget w8{ std::move(w4) };     // Calls Initializer_list-parameter-ctor.
                                    // w4 converts to float through operator float()
                                    // then converts to long double.


## Initializer_list and Constructor.

当一个non-aggregate class类型braced-initialization，重载方案选择遵守以下两条规则：

- 如果有initializer-list ctor，则候选函数只有initializer-list ctor，并且{}列表至少有一个元素
则整个参数列表作为initializer_list传入ctor。
- 如果没有initializer-list ctor匹配(包含转换)，其他所有构造函数都可以成为候选函数，参数列表分开传入ctor。

如果{}为空并且类有默认构造函数，则跳过前一条原则。
如果是copy-list-initialization，如果选拔出的ctor是explicit的，则ill-formed。

    class Widget{
    public:
        Widget(int i, bool b);
        Widget(int i, double b);
        Widget(std::initializer_list<bool> il);  // Initializer_list parameter.
        ...
    };

    Widget w{ 1, 5.0 };         // error! Requires narrowing conversion.

如上即使这里拥有完美配对的Widget(int i, double b)，但是该构造函数不在候选之中，
反而是使用了initializer_list ctor，同时缩窄转换带来错误。

如果初始化列表中的元素与initializer_list的元素不存在转换，则符合原则二：

    class Widget{
    public:
        Widget(int i, bool b);
        Widget(int i, double b);
        Widget(std::initializer_list<string> il);  // Initializer_list parameter.
        ...
    };

    Widget w{ 1, 5.0 };         // Fine. Use non-initializer_list constructor.

由于1和5.0无法隐式转换为string，所以候选才加入了前两个构造函数。

接着如果是空的初始化列表并且有默认构造函数，则跳过原则一：

    class Widget{
    public:
        Widget()；           // Default constructor.
        Widget(std::initializer_list<string> il);  // Initializer_list parameter.
        ...
    };

    Widget w1;          // Call the default ctor.
    Widget w2{};        // Call the default ctor.
    Widget w3();        // Define a function that return Widget.
    Widget w4({});      // Call the initializer_list one.
    Widget w5{{}};      // ditto.

在设计类的构造函数时应该尽量避免和vector类似的情况：

    std::vector<int> v1{ 10, 20 };  // Initialize v1 with two elements in the list.

    std::vector<int> v2(10, 20);    // Initialize v2 with ten elements (20).
  
设计类时应当尽量使用户不论使用{}还是()都获得一样的结果。比如，原本类中不含有initializer_list ctor，但是后来添加了，如果像vector一样的设计，会导致原本用户的代码调用不同的构造函数。这与普通的重载不一样，因为initializer_list ctor总是trump其他构造函数，导致更大的问题。


## Choose Braces or Parentheses

作为类用户，选择Braces和Parentheses有两种方法，以其中一个为主，不到万不得已时使用另一个，并且持之以恒。
两种方法各有所长。

作为模板类设计，这个选择便十分关键了，见下：

    template<typename T, typename... Ts>
    void doSome(Ts&&... params) {
        create local object from params.
    }

可以有以下两种实现：

    T localobj(std::forward(params)...);        // Using paren.
    T localobj{ std::forward(params)... };      // Using brace.

对于vector就会产生不同的效果：

    doSome<std::vector<int>>(10, 20);

前者生成10个元素，初始化为20；后者生成2个元素，初始化为10、20。
这个问题，在标准库设计中也存在，std::make_shared和std::make_unique。

## Things to Remember

- braced-initialization可以应用到更广泛的情景，避免缩窄转换(clang好像并不会)，避免某个混淆语句。
- 注意构造函数重载中，initializer_list constructor的特殊。
- std::vector< numeric type>构造在选用{}和()的不同。
- 关注模板中使用{}和()进行实现的不同。