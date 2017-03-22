# Item15: Use *constexpr* Whenever Possible

从概念上来说，*constexpr*标识了一个值时，表明这是一个可以在编译期间就知道的值。但*constexpr*对于函数的意义并不像对于值一样简单。

constexpr function的返回值既不一定是const的，也不一定是编译期决定的。这可以理解为constexpr function的一项*feature*，因为这对于实现来说是非常友好的。

## *constexpr* Object

对于*constexpr*修饰的obj，既是*const*的，也是编译期决定的（其实更多的是在*translation*决定的，包含编译和链接）。

编译期决定的值是具有特殊性的：可以位于read—only memory；可以用于*integral constant expression*，比如数组大小、枚举值等。

    int sz;         // Non-constexpr variable.
    constexpr auto arraySize1 = sz;     // error! sz's value not 
                                        // known at compilation.
    std::array<int, sz> data1;          // error!
    constexpr auto arraySize2 = 10;     // Fine, 10 is a const
                                        // expression
    std::array<int, arraySize2> data2;  // Fine.

注意*const*并不提供编译期确定的保障：

    const auto arraySize = sz;          // Fine. arraySize is const.

    std::array<int, arraySize> data;    // error!

即对于object，所有*constexpr*都是*const*的，但*const*的并不都是*constexpr*。

## *constexpr* Function

当*constexpr*用于function时，情况就变得复杂的多。

- *constexpr*函数在传入参数均为编译期可确定参数将产生*constexpr*，即编译期返回值。
- *constexpr*函数在传入参数不全为编译期可确定参数，其行为和普通函数一致。即在运行期返回。  
>

    constexpr int pow(int base, int exp) noexcept { // pow's a func
        ...                                         // never throws.
    }
    constexpr auto numConds = 5;
    std::array<int, pow(3,numConds)> results;       // Fine.

pow()函数在该调用环境下产生了编译期可确定的变量。但如果传入的base和exp不全是编译期常量，那上述代码将会报错，因为array的模板参数有相应的要求：

    auto base = readFromDB("base");             // Get these values
    auto exp = readFromDB("exponent");          // at runtime.
    std::array<int, pow(base, exp)> results;    // error! call pow
                                                // at runtime.

所以为了满足在编译期返回常量，对*constexpr function*有实现的约束，而C++11和C++14中的约束不同。

在C++11中：  
*constexpr function*必须只存在一条*return*语句，不能存在其他任何语句，所以为了符合产生了许多不符合直觉的实现：（使用递归和?:）
    
    constexpr int pow(int base, int exp) noexcept {
        return (exp == 0 ? 1: base*pow(base, exp - 1));
    }

在C++14中：  
这一约束被放宽，即：

    constexpr int pow(int base, int exp) noexcept {
        auto result = 1;
        for(int i = 0;i < exp; ++i) result *= base;
        return result;
    }

*constexpr function*被限制只能够获取和返回*literal types*，即其值可以在编译期确定的类型。

在C++11中：  
所有内置类型（除了*void*）和用户定义类型都可以时*literal types*，因为构造函数与其他成员函数都可以是constexpr。

    class Point {
    public:
        constexpr Point(double xVal = 0, double yVal = 0) noexcept
            : x(xVal), y(yVal){}
        constexpr double xValue() const noexcept { return x; }
        constexpr double yValue() const noexcept { return y; }
        
        void setX(double newX) noexcept { x = newX; }
        void setY(double newY) noexcept { y = newY; }
    private:
        double x, y;
    };

因为构造函数是*constexpr*，如果在构造Point时，传入的xVal和yVal都是编译期常量的话，那么Point也将会是编译期常量。

    constexpr Point p1(-9, 27.7);      // Fine. "runs" constexpr                                        // ctor during compilation.
    constexpr double x = 28.8;
    constexpr Point p2(x, 5.3);         // Ditto.

同样的Point相应的*constexpr*成员函数也能够产生*constexpr*。

    constexpr Point midPoint(const Point& p1, const Point& p2) noexcept {
        return { (p1.xValue() + p2.xValue())/2, 
                 (p1.yValue() + p2.yValue())/2 };
    }

    constexpr auto mid = midPoint;      // Init constexpr object with
                                        // constexpr function.

这样的编程手段既完成了数据的抽象化，还通过*constexpr*将运行成本转移到编译期，提高程序的效率。同时还可以将这些数据直接创建在read-only-memory，使得这类数据可以用在*constant expression*中，用在数组长度、模板参数和枚举量这类参数上。

    p1.setX(1);             // error! p1 is const.

注意到两个set函数没有声明constexpr，因为在C++11中：所有constexpr成员函数默认const；函数返回值为void，非*literal type*。

但在C++14中，这两条也已经被忽略，可以进行这样的声明：

    constexpr void setX(double newX) noexcept {
        x = newX;
    }

    constexpr void setY(double newY) noexcept {
        y = newY;
    }

这可能有些违背直觉：

    p1.setX(1);             // error! p1 is const.

该函数的使用依旧是报错的，因为直接在非*constexpr*环境下执行该操作，是非法的，p1已经是一个*constexpr object*，对其进行任何改变都是非法的，但可以进行这样的操作:

    constexpr Point reflection(const Point& p) noexcept {
        Point result;                   // Create a non-const Point.
        result.setX(-p.xValue());
        result.setY(-p.yValue());
        return result;
    }

该函数内部声明了一个non-const Point，这个Point可以使用setX和setY，如果传入的p是编译期常量，那么result的一系列操作也可以在编译期完成，所以最后返回值可以是一个编译期常量。

    constexpr auto reflec = reflection(p1);    // Create a constexpr.
    std::array<int, static_cast<int>(reflec.xValue())> i;   // Fine.

## Clear Why Use *constexpr* Whenever Possible

- *constexpr object*和*constexpr function*比普通变量和函数拥有更大的适用范围，更强大的效率。
- 注意*constexpr*是函数接口的一部分，既该函数可以用于常量表达式*constant expression*。
- 如果移除*constexpr*有可能导致大量代码非法。所以设计*constexpr*务必小心和慎用。

## Things to Remember

- *constexpr object*是*const*的，而且在编译期初始化。
- *constexpr function*产生*constexpr*当传入编译期常量时。
- *constexpr*变量和函数拥有更广的适用范围和减少运行期销耗。
- *constexpr*是函数和对象接口的一部分。
