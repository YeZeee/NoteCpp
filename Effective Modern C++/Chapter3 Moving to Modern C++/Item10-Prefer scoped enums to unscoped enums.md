# Item10: Prefer Scoped Enums to Unscoped Enums

## Scoped and Avoid Converting
在一般的原则下，一个block{}代表了一个scope。但是enum是例外的，enum中的变量的作用域在enum所在的域中。

    enum Color { black, white, red };       // black, white..has the same scope as Color.

    auto white = false;                     // error! white already declared in Color.

C++11中，提供了更加符合常理的enum：scoped-enum。

    enum class Color{ black, white, red };  // black, white...are scoped in Color.

    auto white = false;                     // Fine. 

    Color cc = white;                       // error, white is bool.

    Color cc = Color::white;                // Fine.

    auto cc = Color::white;                 // cc is Color.

通过scoped-enum来防止枚举变量的泄露。  

同时enum具有和整型之间的隐式转换，而enum class则没有。

    enum Color { black, white, red };
    std::vector<std::size_t> primeFactors(std::size_t x);

    Color cc =red;

    if(cc < 14.5) {
        auto factors = primeFactors(cc);        // Implicitly convert happen.
    }

    enum class Color { black, white, red };

    Color cc = Color::red;

    if(cc < 14.5) {                             // error! cannot convert.
        auto factors = primeFactors(cc);        // error! cannot convert.
    }

    if(static_cast<double>(cc) < 14.5) {        // Fine.
        auto factors = primeFactors(static_cast<std::size_t>(cc));  // Fine.    
    }

## Forward enum Declaration.

注意enum是一个编译期确定的量，scoped-enum的另一个优越性就是可以进行前置声明。

    
    enum Color;         // error！ cannot forward-declaration.

    enum class Color;   // Fine.

这是不完全的，因为enum在C++11中也可以通过一些额外动作使得其可以进行前置声明。前置声明的好处在于可以减少编译。比如存在以下头文件：

    // file locstring.h
    #include <string>
    enum localized_string_id
    {
    /* very long list of ids */
    };

    std::istream& operator>>(std::istream& is, localized_string_id& id);
    std::string get_localized_string(localized_string_id id);

可以看到下方的两个函数是依赖于enum的。如果localized_string_id中的变量是频繁改变的。那所有包含了该头文件的组件都要重新编译，这要付出很高的成本。

如果使用前置声明，就可以直接在头文件中保留前置声明部分，并且为每一个编译单元实现各自的enum（注意enum是编译期确定的），一些不依赖于新加入的enumerator的单元就可以不用再次进行编译了。

## Underlying-Type

之所以在C++98中没有前置声明，是因为enum实现类似于一个打包宏定义。

    enum Color { black, white, red};

    #define black 0;
    #define white 1;
    #define red 2;

所以enum其实是有一个底层实现的intergal type的，可以是int，char...具体依赖于编译器自己实现的。

unscoped-enum是不确定的，但是scoped-enum是确定的，默认为int，但是在c++11后，enum也可以进行强类型的声明。

    enum class status;      // Underlying type is int.

    enum status;            // Unknown.

    enum class status :uint8_t;     // Underlying type is uint8_t.

    enum status :uint8_t;           // Underlying type is uint8_t.

## Where Unscoped-enum is Better Than Scoped-enum

虽然scoped-enum具有许多优点：防止隐式转换，防止命名空间的污染，具有前置声明之类的。但是有一个地方enum比scoped-enum更加适用-tuple：

    using UserInfo = std::tuple<string,     // Name. 
                                string,     // Email.
                                size_t>     // Reputation.

    UserInfo uInfo;     // Object of UserInfo.
    auto val = std::get<1>(uInfo);      // Get the field 1-email.

就和注释中所言，字段1代表了uInfo的email，但是1代表email总是不直观的。

    enum UserInfoField { uiName, uiEmail, uiReputation };
    auto val = std::get<uiEmail>(uInfo);        // Get the field uiemail.

这就利用了enum的隐式转换，使用scoped-enum显然就要费事的多。

或许可以通过外加的包装实现更简单的语法，但是注意field-1是一个template parameter，这意味着值需要在编译期间确定，enum具有这样的能力（或者宏），所以这层包装就需要用到constexpr function:

    template<typename E>
    constexpr auto
    toUType(E enumerator) noexcept {
        return static_cast<typename 
            std::underlyting_type<E>::type>(enumerator);
    }

    enum class UserInfoField { uiName, uiEmail, uiReputation };

    auto val = std::get<toUType(uiEmail)>(uInfo);

即使加上封装，还是不如enum来的简单，但是这又避免了污染命名空间。

## Things to Remember

- scoped-enum不会污染命名空间，而且只能通过cast转换为其他类型。
- scoped-enum和unscoped-enum都具有指定underlying-type的方法，不同的是，scoped-enum具有默认的int，而unscoped-enum没有。
- scoped-enum总是可以前置声明，而unscoped只有在指定underlying-type时才可以，注意enum工作在编译期。
