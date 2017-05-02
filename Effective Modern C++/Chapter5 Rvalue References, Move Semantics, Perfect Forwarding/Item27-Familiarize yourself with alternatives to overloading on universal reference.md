# Item27:Familiarize Yourself With Alternatives to Overloading on Universal Reference

Item26阐明了同时使用*perfect forwarding template*和*overloading function*是很危险的。所以当碰见这种情况的时候一般有以下几种方法：

- *Abandon overloading*：放弃重载，使用不同的函数名分别实现*perfect forwarding template*和其他函数的功能。
- *Pass by const T&*：放弃*perfect forwarding template*，使用*const T&*传参，缺点是不够高效，性能明显有损耗。
- *Pass by value*：放弃*perfect forwarding template*，使用按值传参，见Item41。

那当不可避免的同时使用*perfect forwarding template*和*overloading function*的时候，应该如何实现？

## Using *Tag Dispatch*

接着Item26的例子：  
第一种方法，为函数加上标签。函数的外层封装依旧不变，因为这是面向用户代码的，那么目标就是通过函数内部自动甄别传入参数的类型，来调用相应的重载，那么实现如下：

    class NameLog {
    public:
        ...
    template<typename T>
    void logAndAdd(T&& name) {
        logAndAddImpl(std::forward<T>(name), std::is_integaral<std::remove_reference_t<T>>());
    }
    private:
        std::multiset<std::string> names;
    };

*std::is_integral*是一个*type trait*，可以无视*cv-qualifier*来判断一个类型是否为整数，使用*std::remove_reference*来移除引用，最后调用*call*操作符产生一个*tag*，传个内层函数：

    template<typename T>
    void logAndAddImpl(T&& name, std::true_type) {
        auto now = std::chrono::system_clock::now();
        log(now, "logAndAdd");
        names.emplace(nameFromIdx(idx));
    }

    template<typename T>
    void logAndAddImpl(T&& name, std::false_type) {
        auto now = std::chrono::system_clock::now();
        log(now, "logAndAdd");
        names.emplace(std::forward<T>(name));
    }

*std::false*和*std::true_type*就是所谓的*tag*，可以帮助决定最终调用哪一个函数，注意没有为*tag*声明参数名：因为*tag*只是用于提示编译器最后调用哪个函数，在运行期间不起任何作用，最后有可能可以被编译器优化取代，这正是我们所期待的，这项设计被称为*tag dispatch*。

## Constraining template that take *universal references*

注意使用*tag dispatch*的基石是使用单个函数(非重载)作为用户端的API，但对于特殊函数，这是无法实现的，比如构造函数。Item26提到*perfect forwarding template*是十分贪婪的，经常能够完美匹配传入参数，而在匹配竞争中受到不合理的调用。这个时候，我们就希望限定*perfect forwarding template*在某些情况下起作用，而不是总是能够被匹配上。  

    std::enable_if<condition>::type

这项技术的关键就是*SFINAE*。在标准库中给出了*std::enable_if*，它在条件正确的情况下，会有类型成员*value*，而在条件错误的情况下，没有这个成员。

    class Person {
    public:
        template<typename T, 
                 typename = std::enable_if_t<!std::is_same_v<Person, std::decay_t<T>>>>
        explicit Person(T&& n)
            : name(std::forward<T>(n)) {}
        explicit Person(int idx)
            : name(nameFromIdx(idx)) {}
    private:
        std::string name;
    };

*std::enable_if_t*可以得到*std::enable_if*的类型成员*value*；*std::is_same_v*可以得到一个bool值反应两个模板参数是否同类型；*std::decay_t*可以得到类型T退化后的类型(删除*cv-qualifier*和引用，对数组和函数退化为指针)。当传入一个希望使用copy或者move构造函数的参数，*perfect forwarding template*参与匹配，但是*std::enable_if*没有类型成员value，匹配失败但是不报错(*SFINAE*)，最后这个匹配被踢出匹配队列，拷贝构造函数或者移动构造函数当选。

但是可以发现仍然不能解决继承带来的匹配竞争，因为基类和继承类在*std::is_same_v*中得到的是false:

    class Person {
    public:
        template<typename T, 
                 typename = std::enable_if_t<!std::is_base_of_v<Person, std::decay_t<T>>>>
        explicit Person(T&& n)
            : name(std::forward<T>(n)) {}
        explicit Person(int idx)
            : name(nameFromIdx(idx)) {}
    private:
        std::string name;
    };   

*std::is_base_of_v*可以判断后一个类型是否为前一个类型的继承类，如果是用户定义类型同类型也被判断为true(内置类型为false)。最后在加上整数判断:

    class Person {
    public:
        template<typename T, 
                 typename = std::enable_if_t<!std::is_base_of_v<Person, std::decay_t<T>> &&
                 !std::is_integral_v<std::remove_reference_t<T>> >>
        explicit Person(T&& n)
            : name(std::forward<T>(n)) {}
        explicit Person(int idx)
            : name(nameFromIdx(idx)) {}
    private:
        std::string name;
    };       

以上内容属于C++模板元编程的内容。

## Trade-offs





