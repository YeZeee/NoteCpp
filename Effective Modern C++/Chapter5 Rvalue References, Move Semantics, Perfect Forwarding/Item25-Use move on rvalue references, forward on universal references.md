# Item25:Use *std::move* on rvalue References, *std::forward* on Universal References

*rvalue references*总是和可以*move*的对象绑定在一起：

    class Widget {
    public:
        Widget(Widget&& rhs);   // move constructor. rhs definitely  
                                //refers to an object eligible to moveing.
        ...
    };

所以总是可以对rhs引用的对象进行移动，所以移动构造函数就应该进行这样的实现，对可移动的数据成员进行移动：

    class Widget {
    public:
        Widget(Widget&& rhs) : 
            name(std::move(rhs.name)), pImpl(std::move(rhs.pImpl)){}
        ...
    private:
        std::string name
        std::shared_ptr<Impl> pImpl;
    };

*universal references*(*forwarding references*)指代其绑定的对象有可能可以进行*move*：

    class Widget {
    public:
        template<typename T>
        void setName(T&& newName) {     // newName might be moving-able.
            name = std::forward<T>(newName);
        }
    };

## Why do so

对*rvalue reference*使用*std::forward*是可以的，因为*std::forward*可以实现*std::move*，但是这样的代码可读性差，易出错；对*universal reference*使用*std::mvoe*也是可以的，但是后果很严重，因为有可能篡改一些*lvalue*。

比如以下代码，可以通过编译：

    class Widget {
    public:
        template<typename T>
        void setName(T&& newName) {
            name = std::move(newName);
        }
    };
    
    std::string getWidgetName();
    Widget w;
    auto n = getWidgetName();       // n is lvalue.
    w.setName(n);                   // Move n to w.name.
                                    // Now n is unknown.

也有认为可以通过重载函数替代模板和万能引用来实现：

    class Widget {
    public:
        void setName(const std::string& newName) {
            name = newName;                 // copy version.
        }
        void setName(std::string&& newName) {
            name = std::move(newName);      // move version.
        }
    };

这样的代码好像可以替代模板与万能引用，但其实有许多缺点：  
使用两个函数替代一个函数带来更多的维护成本；同时原版本的实现更加有效率：  
假设有如下代码：

    w.setName("some string");

"some string"是一个字符串字面值，前一个版本可以生成函数：

    void setName(char& newName[12]) {
        name = newName;
    }

这个函数可以直接将字符串传给name的赋值操作符，而后一个版本需要先构造newName，然后将newName传给移动赋值操作符，这就比原版本多出了需要更多的操作，更低的效率。但是致命的还不在这两条原因。  

如果函数的参数数量增加，甚至用上了可变参数，重载的方法将无法使用，因为为了覆盖所有情况需要重载2^n个函数，所以只能使用*universal reference*的方法实现。

## When Use reference More Than One Times

有时传入的*rvalue reference*或者*universal reference*可能需要多次使用，*std::move*和*std::forward*应该只用在最后一次调用：

    template<typename T>
    void setSignText(T&& text) {
        sign.setText(text);
        auto now = std::chrono::system_clock::now();
        signHistory.add(now, std::forward<T>(text));
    }

原因很简单，在最后一次使用之前我们需要保证引用中的内容不能被移走。*std::move*在某些情况下，应该被*std::move_if_noexcept*替代。

## Return Value and *RVO*

如果函数是以值返回，如果想要返回的对象是绑定在*rvalue reference*或者*universal reference*，应该使用*std::move*和*std::forward*包装返回的引用：

    Matrix operator(Matrix&& lhs, const Matrix& rhs) {
        lhs += rhs;
        return std::move(lhs);      // move lhs into return value.
    }

通过这样，如果Matrix是可移动的，可以把参数右值lhs移动到返回的临时对象中去，对比不用的*std::move*：

    Matrix operator(Matrix&& lhs, const Matrix& rhs) {
        lhs += rhs;
        return lhs;     // copy lhs to return value.
    }

如果不用*std::move*，那么进行的就是copy，而lhs是一个建议进行移动的对象，增加了拷贝成本。

如果Matrix不支持移动，那么使用*std::move*的版本也能够成功匹配上复制操作，如果未来Matrix支持了移动操作，那么该函数的代码也不需要重新修改，增加了代码的可维护性。

同样的，对于*universal reference*：

    template<typename T>
    Fraction reduceAndCopy(T&& frac) {
        frac.reduce();
        return std::forward<T>(frac);   // move rvalue into return  
                                        // value. copy lvalue into return value.
    }

注意以上条件限于*reference*，而不是*local variable*，而且是按值返回：

    Widget makeWidget {
        ...
        Widget w;
        return w;       // RVO.
    }

    Widget makeWidget {
        ...
        Widget w;
        return std::move(w);        // move.
    }

对于*local variable*，不要在return中使用*std::move*和*std::forward*，因为这阻止了编译器的优化(*Return Value Optimization*).

RVO：编译器在用于返回的局部变量类型与返回类型一致的情况下可以省略*return value*的copy：这个局部变量包括return语句中创建的临时对象。有时RVO特指对临时对象的返回，而NRVO指代对*named value*的返回。

如果使用*std::move*阻止了RVO，反而多了移动操作，因为std::move(w)其实是一个指向*local variable*的引用，不满足*RVO*的条件。

但同时RVO是一项标准建议的优化，并不是标准(尽管大多数编译器都支持这门优化)。当没有RVO的时候，编译器会被要求使用*move*而不是*copy*进行返回，所以在return中使用*std::move*是徒劳无功的。

同样的理由可以用于返回*by-value parameter*的情形，因为parameter是不会进行RVO的，但是编译器会主动使用*move*而不是*copy*返回这个值，所以没有必要使用*std::move*：

    Widget makeWidget(Widget w) {
        ...
        return w;       // by-value parameter of same type of return.
    }

编译器会自动进行如下的实现:

    Widget makeWidget(Widget w) {
        ...
        return std::move(w);        // treat w as rvalue.
    }

对于返回*local variable*使用*std::move*，并不能优化代码，反而禁止了编译器的优化。只有某些情况下对*skocal variable*使用*std::move*才有意义(比如传入某些函数，而你已经不再需要这个变量了)。所以不要对return的变量使用*std::move*。

## Things to Remember

- 在最后一次使用引用的时候，对*rvalue reference*使用*std::move*,对*universal reference*使用*std::forward*。
- 对于返回*rvalue reference*和*universal reference*，执行第一条规则。
- 对于返回局部变量或者按值传递参数，不要使用*std::move*或者*std::forward*，这禁止了RVO。