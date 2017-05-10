# Use Init Capture to Move Objects Into Closures

C++11的*lambda*中无法移动一个对象进入*closure*，C++14中提供了一个新的机制，使得移动对象进入闭包变为可能，即*init capture*。*init capture*可以表达除了默认捕捉以外的所有形式的捕捉。

## *Init Capture*

*init capture*具有两个部分：一是闭包中的变量名，二是初始化闭包中变量的表达式。

    class Widget {
    public:
        ...
        bool isValidated() const;
        bool isProcessed() const;
        bool isArchived() const;
        ...
    private:
        ...
    };

    auto pw = std::make_unique<Widget>();

    auto func = [pw = std::move(pw)]{
        return pw->isValidated() && pw->isArchived();
    }

通过初始化捕捉就可以把变量移动到闭包中去。通过初始化捕捉还可以实现表达式的捕捉：

    auto func = [pw = std::make_unique<Widget>()]{
        return pw->isValidated() && pw->isArchived();
    }

原本*lambda*是无法捕捉一个表达式的，通过初始化捕捉可以捕捉一个表达式的值，又称初始化捕捉为*generalized lambda capture*。

## How to Achieve move-capture in C++11

*lambda*并不是一个全新的功能，*lambda*能做的任何事，使用*callable class*都可以实现:

    class IsValandArch {
    public:
        using DataType = std::unique_ptr<Widget>;
        explicit IsValAndArch(DataType&& ptr):
            pw(std::move(ptr)){}
        bool operator()() const {
            return pw->isValidated() && pw->isArchived();
        }
    private:
        DataType pw;
    };

    auto func = IsValAndArch(std::make_unique<Widget>());

同时还可以利用*std::bind*和*lambda*的参数：

    std::vector<double> data;

    auto func = [data = std::move(data)]{...};      // C++14 style.

    auto func = std::bind(
        [](const std::vector<double>& data){...}, 
        std::move(data))                            // C++11 style.

称第二种对象为*bind object*，注意到bind使用*const reference*做类型声明。这是因为在C++14的形式中，data是拷贝捕获，而*lambda*默认const，所以捕获参数不能被修改，在C++11的版本中，变量data模拟了被捕获变量，也应该是const的。

    auto func = [data = std::move(data)]() mutable {...};
    
    auto func = std::bind(
        [](std::vector<double>& data) mutable {...}, 
        std::move(data))

以上两个*mutable lambda*是类似实现。待捕捉对象先通过*bind*,*bind object*中包含*lambda*的一个*closure*。所以闭包的生命周期和*bind object*一样。

## Things to Remember

- 使用C++14中的*init cpature*移动捕捉对象或者捕获表达式。
- C++11可以通过*callable class*或者*std::bind*实现。