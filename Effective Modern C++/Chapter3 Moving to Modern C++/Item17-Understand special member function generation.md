# Item17: Understand Special Member Function Generation

对于C++类来说，有一些成员函数是特别的：  
在C++98中，包括默认构造函数(default constructor)、析构函数(destructor)、复制构造函数(copy constructor)、复制赋值操作符(copy assginment operator)。
这类函数在某些条件下，会由编译器自动生成，生成的函数隐含*public*和*inline*，而且总是非*virtual*的，除非基类函数的析构函数声明为*virtual*，则继承类的生成的析构函数为*virtual*。  

在C++11中，这类函数又多了两个成员：移动构造函数(move constructor)、移动赋值操作符(move assignment operator)。其编译器生成的作用(与copy类似)是对非静态类成员进行"memberwise moves"，对于基类部分调用相应的移动方法。  
这些move操作并不是真正的move，该move操作是依赖与copy实现的，因为类型本身不存在move这个概念。

    class Widget {
    public:
        Widget();           // default constructor.
        Widget(const Widget&);      // copy constructor.
        Widget(Widget&&);           // move constructor.
        Widget& operator=(const Widget&);   // copy assignment operator.
        Widget& operator=(Widget&&);    // move assignment operator.
    };e

move的自动生成与copy有一点不同：

两个copy操作是独立的，如果自行实现其中一个，另一个依旧可以由编译器实现；但move不独立，自行实现其中一个，另一个将不会实现。理由是一旦你自行实现move，就意味着你将不使用默认的"memberwise move"的语义，那么编译器没有理由实现一个错误语义的move操作。





