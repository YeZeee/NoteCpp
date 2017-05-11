# Item33:Use *decltype* on *auto&&* Parameters to Forward Them

C++14带来了*generic lambda*：

    auto f = [](auto x){ return normalize(x); };

其闭包相当于：

    class SomeCompilerGeneratedClassName {
    public:
        template<typename T>
        auto operator()(T x) const {
            return normalize(x);
        }
    }

*lambda*起到的作用相当于将参数x转发给内部函数normalize，如果需要完美转发，结构应该如下：

    auto f = [](auto&& x){ return normalize(std::forward<?>(x)); };

问题转化为如何填入*std::forward*的模板参数。因为auto和模板的类型推断基本一致，所以x的类型就包含了传入参数的*value category*。通过Item3可知道，使用*decltype*帮助推断传入的x的*value category*。

如果x绑定一个左值，那么x就是decltype(x)就会产生左值引用类型；如果x绑定一个右值，decltype(x)就会产生一个右值引用类型，这与模板直接传入值类型不同，hui发生*reference collapsing*,再看*std::forward*。

    template<typename T>
    T&& forward(std::remove_reference_t<T>& param) {
        return static_cast<T&&>(param);
    }

如果传入右值，使用decltype(x)作为*std::forward*的模板参数，那么T = Widget&&，并发生*reference collapsing*，和直接传入值类型(T = Widget)一样：

    Widget&& forward(Widget& param) {
        return static_cast<Widget&&>(param);
    }

这样通过decltype(x)作为*std::forward*的模板参数也可以实现完美转发：

    auto f = [](auto&& x){ 
        return normalize(std::forward<decltype(x)>(x));
    };

对于传入参数包：

    auto f = [](auto&&... xs){ 
        return normalize(std::forward<decltype(xs)>(xs)...); 
    };    

## Things to Remember

- 使用auto&&(*universal reference*)和*decltype*实现*lambda*的完美转发。