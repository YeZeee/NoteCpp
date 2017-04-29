# Item26:Avoid Overloading on Universal References

## Use *perfect forwarding template*

*perfect forwarding template*具有超广的适应性和良好的性能，见如下实现：

    class NameLog {
    public:
        ...
        void logAndAdd(const std::string& name) {
            auto now = std::chrono::system_clock::now();
            log(now, "logAndAdd");
            names.emplace(name);
        }
    private:
        std::multiset<std::string> names;
    };

函数logAndAdd的参数为*const std::string&*，可以绑定以下对象：

    std::string myname("Darla");

    NameLog l;

    l.logAndAdd(myname);          // pass lvalue std::string.
    l.logAndAdd(std::string("David"));      // pass rvalue std::string.
    l.logAndAdd("shanshan");                // pass string literal.

第一个传值将myname(lvalue)绑定给name，最后在函数内部作为emplace的函数参数，调用*std::string*的复制构造函数，基本没有可优化的余地；第二个传值将一个rvalue绑定给name，最后在函数内部作为emplace的函数参数，调用*std::string*的复制构造函数，这里显然可以调用*std::string*的移动构造函数来节省时间；第三个传值多了一个临时对象的创建，见Item25。使用*perfect forwarding template*，可以完美优化传值：

    template<typename T>
    void logAndAdd(T&& name) {
        auto now = std::chrono::system_clock::now();
        log(now, "logAndAdd");
        names.emplace(std::forward<T>(name));
    }

第一个传值不变；第二个传值可以使用移动构造函数，第三个传值可以调用*std::string*的以字符串为参数的构造函数，避免临时对象的构造，完美提升代码效率。

## Do Not overloading *perfect forwarding template*

为*perfect forwarding template*重载函数是十分危险的，比如为logAndAdd重载一个以index为参数的函数：

    void logAndAdd(int idx) {
        auto now = std::chrono::system_clock::now();
        log(now, "logAndAdd");
        names.emplace(nameFromIdx(idx));
    }

    l.logAndAdd(22);        // call the non-template version.

    l.logAndAdd(22U);       // call the template version. error.

可以看到只有完美匹配上非模板的版本，才能正确的实现意图。所以重载*perfect forwarding template*函数是非常危险的，因为这个模板很容易实现一些非计划的重载，

由于构造函数经常是重载函数，使用*perfect forwarding constructor*也是非常危险的：

    class Person {
    public:
        template<typename T>
        explicit Person(T&& n)
            : name(std::forward<T>(n));
        explicit Person(int idx)
            : name(nameFromIdx(idx));
        Person(const Person& rhs);
        Person(Person&& rhs);
    private:
        std::string name;
    };

因为*perfect forwarding template*会和其他构造函数产生竞争。

    Person p("Nancy");

    auto cloneOfP(p);       // create new Person from p, template consturctor.
 
因为*perfect forwarding template*生成了比复制构造函数匹配优先级更高的函数:

    explicit Person(Person& n)
        : name(std::forward<Person&>(n));       // have high priority.

只有这个调用是使用复制构造函数：

    const Person cp("const Nancy");

    auto cloneOfP(cp)       // use copy ctor.


同理，继承类的拷贝和移动构造函数不使用基类的拷贝和移动构造函数，而是使用基类的*perfect forwarding constructor*,因为传入的参数并不是完美契合基类的拷贝和移动构造函数。

    class SpecialPerson : public Person {
    public:
        SpecialPerson(const SpecialPerson& rhs)
            : Person(rhs) {
            ...
        }

        SpecialPerson(SpecialPerson&& rhs)
            : Person(std::move(rhs)) {
            ...
        }
    };

所以有可能的化，避免重载一切*perfect forwarding template*，这不是一个良好的设计，容易带来出人意料的后果。但如果不可避免的要重载，解决方案见Item27。

## Things to Remember

- 重载*perfect forwarding template*会使得*perfect forwarding template*在某些出人意料的情况下被调用。
- *perfect forwarding constructor*不是一个良好的设计，会与其他的构造函数产生不可预料的竞争与替代，这种情况在继承类调用基类构造函数时也会发生。
