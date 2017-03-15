# Declare Overriding Functions *override*

C++中涉及OOP的主要包含了类、继承、虚函数。虚函数的概念相当于在子类中覆盖(override)了基类函数的实现，而这一机制常常出现错误。

因为"overriding"和"overloading"是非常易于混淆的。

    class Base {
    public:
        virtual void doWork();      // Base class virtual func.
        ...
    };

    class Derived : public Base {
    public:
        virtual void doWork();      // Derived class virtual func.
        ...
    };

    std::unique_ptr<Base> uqb =         // Create a ptr of Base-typed
        std::make_unique<Derived>();    // pointing to Derived-obj.

    ...
    upb->doWork();                  // Call the doWork func.

## Requirement of Overriding

见上述代码，为了overriding机制生效，必须满足以下几个条件：

- 基类的函数必须声明为*virtual*
- 基类和子类的函数名必须一样(除了析构函数)
- 函数参数列表必须一样
- 函数constness必须一样
- 返回类型和异常修饰必须兼容

以上为C++98的内容，C++11新添加了：

- reference限定符必须相同 

包含有虚函数错误的代码往往是合法的，能够通过编译的，所以编译器往往不能提出警报：

    class Base {
    public:
        virtual void mf1() const;
        virtual void mf2(int x);
        virtual void mf3() &;
        void mf4() const;
    };

    class Derived : public Base {
    public:
        virtual void mf1();
        virtual void mf2(unsigned int x);
        virtual void mf3() &&;
        void mf4() const;
    };

以上代码不包含任何override机制的函数，都或多或少不满足要求。编译器可能不会提醒你，因为编译器也不明白你的需求是什么。

## *override* And *final*

为了解决这一问题，C++11提供了显式要求override的方法，提供了关键字*override*：

    class Derived : public Base {
    public:
        virtual void mf1() override;
        virtual void mf2(unsigned int x) override;
        virtual void mf3() && override;
        void mf4() const override;
    };

这样一来编译器必然报错，以上代码无法通过编译，因为编译器已经知道了你的需求是override，只有以下代码才能通过编译。

    class Base {
    public:
        virtual void mf1() const;
        virtual void mf2(int x);
        virtual void mf3() &;
        virtual void mf4() const;       // Add virtual
    };

    class Derived : public Base {
    public:
        virtual void mf1() const override;
        virtual void mf2(int x) override;
        virtual void mf3() & override;
        void mf4() const override;      // virtual is not necessary.
    };

C++11中的*final*和*override*是contextual keywords。意味着这两个关键字可只在某些情况下被看作关键字。比如*override*只有在成员函数尾巴才是关键字。

## Reference Qualifiers

reference qualifiers可以理解为修饰*this的qualifier，这点和member function的const类似：

    void doSomething(Widget& w);        // Accepts only lvalue.

    void doSomething(Widget&& w);       // Accepts only rvalue.

    class Widget {
    public:
        void doWork() &;        // Only when *this is lvalue.
        void doWork() &&;       // Only when *this is rvalue.
    ...
    };

    Widget makeWidget();
    Widget w;

    makeWidget().doWork();      // Use && version.
    w.doWork();                 // Use & version.

再看以下代码：

    class Widget {
    public:
        using DataType = std::vector<double>;
        ...
        DataType& data() { return values; }
    private:
        DataType values;
    };

    // Client code：
    Widget w;
    ...
    auto vals1 = w.data();      // Fine.copy the w.values to vals1.

    Widget makeWidget();

    auto vals2 = makeWidget().data();   // Copy too.

前一个用户代码，data()返回了一个value的左值引用，所以调用了copy-assignment，没有问题。第二个用户代码也是类似的操作，所以也没有问题。但是可以发现makeWidget()返回的是一个Widget的临时对象，是一个右值，在表达式结束后就被立刻销毁，所以使用拷贝整个values是浪费时间的，应该使用移动的方式节省时间，即调用vector的move-assignment。

    class Widget {
    public:
        using DataType = std::vector<double>;
        ...
        DataType& data() & { return values; }
        DataType&& data() & { return std::move(values); }
    private:
        DataType values;
    };


















