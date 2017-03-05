# Item6: Use the Explicitly Type Initializer Idiom When auto Deduce Undesired Types

## Proxy class

auto具有许多优点，但是某些情况下，auto会产生一个非期望的结果。如下：

    vector<bool> features(const Widget& w);     // A function return a vector<bool>.

bool的vector是一个特化类型，其中bool的存储是bit级别的。比如bit5指代了Widget优先级。

    Widget w;
    ...
    bool highPriority = features(w)[5];        // Get the Priority of w.
    ...
    process(w,highPriority);            // Do something with w.

这段代码没有任何问题。但如果使用auto去声明highPriority。

    auto highPriority = features(w)[5];        // Is auto giving the right type.

    process(w,highPriority);            // Undefined behaviour.

该代码的运行结果是不确定的。

因为vector<bool>是vector的一个特化类型。出于储存空间的考虑，bool只是概念性的存在于vector容器中，
operator[]返回的并不是bool reference to element，而是vector<bool>::reference。这是一个嵌套在vector<bool>中的类。

>The std::vector<bool> specialization defines std::vector<bool>::reference as a publicly-accessible nested class.

bool在vector<bool>中的存在方式是一个个的bit位。所以operator[]返回的是一个行为类似于bool的对象，该对象于bool之间存在隐式转换。
所以

    bool highPriority = features(w)[5];
    auto highPriority = features(w)[5]; 

前者触发了隐式转换，而后者并没有。后者的值取决于std::vector<bool>::reference的实现。

比如reference中包含了一个指示bit位置的指针。由于features返回了一个vector<bool>的对象，该对象是临时的调用了operator[]，最终highPriority被初始化为reference。然而待语句结束后，vector<bool>的临时对象已经消失，highPriority中的指针悬空，造成了未定义行为(比如而后发生了bool的隐式转换)。

    process(w,highPriority);            // Undefined behaviour. highPriority implicit 
                                           convert to bool with dangling pointer.

proxy class: 代理类，是为了模仿和补强某些类型存在的。比如std::vector<bool>::reference对于bool和智能指针对于raw指针。
有些代理类是暴露给用户的，比如智能指针，有些是隐藏的，比如std::vector<bool>::reference。

某些C++库中，使用一种expression templates的技术。见下表达式:

    Matrix sum = m1 + m2 + m3 + m4;

可以直接实现运算符重载operator+返回一个Matrix对象，这样一来每个operator+都会产生一个临时变量，

但是使用expression templates技术可以提高效率。operator+不再返回一个Matrix对象，而是返回一个proxy class比如Sum<Matrix，Matrix>，这是一个可以隐式转换为Matrix对象的类，同时还允许Sum从表达式初始化，means：

    Sum<Sum<Sum<Matrix，Matrix>，Matrix>，Matrix>

这样一来，减少了临时变量的拷贝和生成，这项使用显然是对用户隐藏的。

auto与invisable proxy class的相性不好，因为大多数invisible proxy class都是设计为短寿命的。

所以应该避免以下情况:

    auto tmp = expression of invisable proxy class type;

## How to Recognize the Proxy Class Type

一是通过文档，二是通过源码。熟悉类的设计，可以大幅降低这方面的错误。

- explicitly typed initializer idiom

auto并不是无法用在proxy class上的，如下：

    auto highPriority = static_cast<bool>(features(w)[5]);


feature(w)[5]返回了一个std::vector<bool>::operator[],然后应用强制转换成bool。
由于是在同一个语句中进行的，所以不会出现上述所言的悬空指针的情况，便不会出现undefined behaviour。
最后auto进行类型推断即可。

同时显式类型初始化语句也可以用于强调转换，使得某些隐式转换不被忽略。比如：

    double calcEpsilon();       // Return tolerance value.
    auto ep = static_cast<float>(calcEpsilon());


## Things to Remember

- "invisable" proxy class types 可以造成auto类型推断出“某种意义上不对”的类型。
- explicitly typed initializer idiom可以防止上述错误的发生。












