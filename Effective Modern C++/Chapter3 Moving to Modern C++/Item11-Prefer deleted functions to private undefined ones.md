## Prefer deleted Functions to Private Undefined Ones

在C++中，有一类成员函数是特殊的，这些成员函数在某些情况下会被编译器默认实现：'member function5'，即：默认构造函数、复制构造函数、复制赋值操作符、移动构造函数、移动赋值操作符。

但是有时候这类函数是不被需要的，所以需要从实现中删除该实现。

## Declare Them private But Not Define Them or Use = delete.

在C++98中，要实现上面所言的特性，可以通过在private访问控制下声明函数，同时不实现他们，就可以达到目的。比如basic_ios的复制构造函数和复制赋值操作符：

    template<class charT, class traits = char_traits<charT>>
    class basic_ios : public ios_base {
    public:
        ...
        
    pirvate:
        basic_ios(const basic&);                // copy-constructor.
        basic_ios& operator=(const basic&);     // copy-assginment.
    }

C++11中提供了= delete来进行实现deleted function：

    template<class charT, class traits = char_traits<charT>>
    class basic_ios : public ios_base {
    public:
        ...
        basic_ios(const basic&) = delete;                // copy-constructor.
        basic_ios& operator=(const basic&) = delete;     // copy-assginment.
    }
    
使用= delete的好处在于：
- public的访问控制比private能够提供更好的纠错报告。
- = delete可以在编译期发现，而老方法有可能要在link期才能发现

## When Deleted Function is Not the Member Function

还有一个好处就是，= delete可以用在非成员函数上：

    bool isLucky(int number);       // Judge if a number is Lucky num.

    isLucky('a');
    isLucky(true);
    isLucky(3.5);           // A series of unexpected usage.

由于隐式转换带来的非期望的使用方式，可以使用= delete禁止：

    bool isLucky(char) = delete;         // Reject char.
    bool isLucky(bool) = delete;         // Reject bool.
    bool isLucky(double) = delete;       // Reject double and float.

注意double参数的重载同时禁止了float，因为c++对于float进行隐式转换总是优先转换为double而不是int。
牢记被delete的函数是会参加函数重载匹配的，而且优先级不会有改变。

还有一个点deleted function的优越性在于可以禁止一个模板的某些实现：

    template<typename T>
    void processPointer(T* ptr);        // Accept a pointer.

    template<>
    void processPointer<void>(void*) = delete;

    template<>
    void processPointer<char>(char*) = delete;

因为char\*可以代表c风格的字符串，void\*无法解除引用，需要将这两个实现禁止，以方便编译器报错。同样的，对于const指针也许也是不适用的：
    
    template<>
    void processPointer<const void>(const void*) = delete;

    template<>
    void processPointer<const char>(const char*) = delete;

甚至对于const volatile void*也是不适合的：

    template<>
    void processPointer<const volatile void>(const volatile void*) = delete;

    template<>
    void processPointer<const volatile char>(const volatile char*) = delete;

对于类内的模板函数也只能用delete，因为无法特化模板使之有不同的访问控制：

    class Widget {
    public:
        ...
        template<typename T>
        void processPointer(T* ptr) {}
        ...
    private:
        template<>
        void processPointer<void>(void*);       // error!
    }

    class Widget {
    public:
        ...
        template<typename T>
        void processPointer(T* ptr) {}
        ...
    }

    template<>
    void Widget::processPointer<void>(void*) = delete;

对于C++98的老方法，类外不能用，类内不一定能用，能用还可能在链接期才起作用，所以C++11的delete可以完全取代该方法。

## Things to Remember

- 总是使用= delete。
- 所有function都可以被delete，包含模板的特化实现、非成员函数...