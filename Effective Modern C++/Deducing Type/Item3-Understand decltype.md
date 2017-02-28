# Item3: Understand Decltype

decltype获取一个表达式，返回一个表达式的类型。
大多数情况其结果如常规想象，但是偶尔也会出一些令人意想不到的结果。

## Typical Case

这些情况下，decltype给出表达式确实的类型。

    const int i =0;             //decltype(i) is const int.
    bool f(const widget& w);    //decltype(f) is bool(const widget&).
    sturct Point {              //decltype(Point::x) is int.
        int x, y;
    };
    widget w;                   //decltype(w) is widget.

    if(f(w))...                 //decltype(f(w)) is bool.

    template <typename T>
    class vector {
        public:
        T& operator[](std::size_t index);
    };

    vector<int> v;              //decltype(v) is vector<int>.

    if(v[0] == 0)...            //decltype(v[0]) is int&.

在这些情况下，decltype乖乖的推导出expr的类型，不进行任何添油加醋。

## Trailing Return Type with Decltype

在C++11中，函数返回值可以采用trailing的方式，而且可以使用decltype.

比如：

    template <typename Container, typename Index>
    auto access(Container& c, Index i) ->decltype(c[i]) {   
        ...
        return c[i];        //access c[i].
    }

    vetcor<int> veci{ 1, 2, 3, 4 };
    access(veci,2) = 5;     //change the veci[2].

由于我们不知道容器Container以及索引Index的准确类型及其内容物类型，
我们直接拼写出返回的内容物的类型，需要通过类型推断。
一种方法是通过Container的内置类型成员(typename)，另一种就是利用decltype.
由于c与i的声明位置，所以需要使用trailing的方式。

这种情况下，auto是仅有占位功能，真正进行推断的是decltye，
decltype分辨出c[i]的类型是T&，所以可以通过access读写c[i]。

## Auto Return Type Deduction

再看item2中提到的, 自C++14起，
允许为多语句lambda以及函数进行自动返回值推断。如下：

    template <typename Container, typename Index>
    auto access(Container& c, Index i) {   
        ...
        return c[i];        //omit the referenceness.
    }

    vetcor<int> veci{ 1, 2, 3, 4 };
    access(veci,2) = 5;     //error, expression is a rvalue.

这种情况下，执行的template的类型推断，所以access返回的是T类型变量，是一个右值。

值得注意，所以为了实现access的原本功能，decltype trailing return type就不能忽略。

## Use decltype(auto)

同时在C++14中，提供了更加优雅的使用decltype的方式，
使得decltype类型推断被使用，如下：

    template <typename Container, typename Index>
    decltype(auto) access(Container& c, Index i) {   
        ...
        return c[i];        //use the decltype rules.
    }

    vetcor<int> veci{ 1, 2, 3, 4 };
    access(veci,2) = 5;     //change the veci[2].

decltype(auto)不仅能用在函数返回推断上，也可以用在通常声明上，使得类型推断自行使用
decltype的规则。

    widget w;
    const widget& crw = w;
    auto w2 = crw;              //auto type deduction, w2 is widget.
    decltype(auto) w3 = crw     //decltype type deduction, w3 is const widget&.

## Decltype's Behaviour

观察如下代码：

    int x = 0;      //decltype(x) is int.
    int y = 0;      //decltype((x)) is int&.

    decltype(true?x:0) i;   //true?x:0 is rvalue-expression, i is int. 
    decltype(true?x:y) i;   //true?x:y is lvalue-expression, i is int&.


产生了意想不到的结果。
因为decltype对name和expression的效果是不一样的。
当decltype作用于name时，产生的类型是T。
当decltype作用于lvalue-expression时，产生的是T&。

Trailing Return Type不容易出现类型的误写，
但decltype(auto)容易出现，如下：

    decltype(auto) foo() {
        int x = 0;
        return x;           //decltype（x） is int, so foo returns int.
    }

    decltype(auto) foo() {
        int x = 0;
        return (x);         //decltype((x)) is int&, so foo returns int&.
    }



## Things to Remember

- decltype大多数情况产生类型与表达式相应的类型，不会有任何修正。
- 对于lvalue-expression，decltype会产生T&.
- C++14支持decltype(auto)，表达使用decltype原则进行类型推断。
