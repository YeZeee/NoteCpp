# Use *unique_ptr* for Exclusive-ownership Resource Management

## Smart Pointers
裸指针是强大的，但同时也是令人厌烦的：

- 裸指针的声明没有指出其指向的是对象还是数组
- 裸指针的声明没有表明是否在完成使用后销毁其指向的对象，即声明没有指出该指针是否持有(*owns*)对象
- 即使需要销毁对象，裸指针也没有表明如何销毁该对象，是采用*delete*还是使用不同的销毁机制
- 使用*delete*时，不知道是使用*delete*还是*delete[]*
- 对指针指向的对象使用析构函数时，很难保证只进行一次析构。错过析构导致内存泄漏，多次析构导致未定义行为
- 无从知道一个指针是否为野指针，指针指向的对象被析构后，指针仍然指向该内存产生野指针

智能指针是一条解决上述问题的途径。智能指针是对裸指针的一次封装，保留指针特性的同时避免许多指针容易带来的错误。在C++11中，带来了4中智能指针：*std::auto_ptr std::unique_ptr std::shared_ptr std::weak_ptr*，用于管理对象的生命周期及资源，防止内存泄漏。

*std::auto_ptr*是一个有设计缺陷的智能指针，在移动成为语义的同时，*std::unique_ptr*可以完全替代*std::auto_ptr*，而且更加强大高效。*std::auto_ptr*已经被标准所抛弃。

## *std::unique_ptr*

- *std::unique_ptr*默认情况下和裸指针具有一样的大小，大多数操作，同样的高效。
- *std::unique_ptr*表现为*exclusive ownership*(专属所有)语义。
- 一个非空的*std::unique_ptr*总是独立持有它所指向的对象。
- 移动一个*std::unique_ptr*意味着转让所有权，即dst指针获得对象，src指针设置为空指针。
- *std::unique_ptr*是*move-only*类型，不支持copy语义，因为*std::unique_ptr*不允许共享对象。
- 当*std::unique_ptr*被销毁时，其指向的对象将在这时刻前进行销毁。

## Common Use for *std::unique_ptr*

*std::unique_ptr*可以用于工厂函数的返回类型。
比如我们拥有以下的层次结构：

    class Investment{...};
    class Stock : public Investment{...};
    class Bond : public Investment{...};
    class RealEstate : public Investment{...};

工厂函数通常在堆上创建一个对象通过指针返回给用户，而用户需要对该对象的资源管理负责，通过使用*std::unique_ptr*，保证用户不需要该对象时，对象随着*std::unique_ptr*的销毁而销毁，防止内存泄漏。

    template<typename... Targs>
    std::unique_ptr<Investment>
    makeInvestment(Targs... params);

用户代码：

    {
        ...
        auto pInvestment = makeInvestment(argments);
        ...
    }   // destroy *pInvestment.

同时*std::unique_ptr*也可以用于所有权交接的场景，比如工厂函数返回值移动至容器，容器按照序列移动给某对象的数据成员，最后这个对象会被销毁(带动*std::unique_ptr*销毁以及*std::unique_ptr*指向的对象销毁)。在这个交接过程中，如果出现非典型程序分支或者异常，使用裸指针带来非常高的资源管理成本，而使用*std::unique_ptr*就没有这个问题。

## Use Custom *deleters*

通常，对象的销毁通过*delete*实现，但*std::unique_ptr*可以在构造时配置自定义的*deleter*。比如在对象被销毁之前，需要将对象信息记录进日志：

    auto delInvmt = [](Investment* pInvestment) {
        makeLog(pInvestment);
        delete pInvestment;
    };

    template<typename... Targs>
    std::unique_ptr<Investment, decltype(delInvmt)>
    makeInvestment(Targs&&... params) {
        std::unique_ptr<Investment, decltype(delInvmt)>
            pInv(nullptr, delInvmt);
        if(...){ 
            pInv.reset(new Stock(std::forward<Targs>(params)...));
        } else if(...){
            pInv.reset(new Bond(std::forward<Targs>(params)...));
        } else if(...){
            pInv.reset(new RealEstate(std::forward<Targs>(params)...));
        }
        return pInv;
    }

以上程序有以下几个注意点：
- delInvmt是一个自定义的*deleter*。
- *std::unique_ptr*的第二个模板参数是*deleter*的类型。
- 将裸指针赋予*std::unique_ptr*是非法的。只可以通过初始化或者reset设置智能指针的内含指针。
- 使用完美转发传递参数。

在C++14中，可以实现函数的返回值自动推断(Item3),可以实现更加优雅的表达：

    template<typename... Targs>
    auto makeInvestment(Targs&&... params) {
        auto delInvmt = [](Investment* pInvestment) {
            makeLog(pInvestment);
            delete pInvestment;
        };
        std::unique_ptr<Investment, decltype(delInvmt)>
            pInv(nullptr, delInvmt);
        ...
        return pInv;
    }

默认的*std::unique_ptr*使用*delete*作为*deleter*，这种情况下，*std::unique_ptr*和裸指针有同样的大小。当使用了自定义的*deleter*后，*std::unique_ptr*将会变大(两倍)。当使用*function*作为*deleter*时，*std::unique_ptr*将会多存储一个函数指针；而当使用*function object*时，膨胀将取决于对象；使用*stateless function object*时(E.g. 无捕获的lambda表达式)，不会有膨胀。所以相同的功能，使用*stateless function object*空间表现更好:

    auto delInvmt = [](Investment* pInvestment) {
        makeLog(pInvestment);
        delete pInvestment;
    };
    template<typename... Targs>
    std::unique_ptr<Investment, decltype(delInvmt1)>    // Return type has sizeof(Investment*).
    makeInvestment(Targs... args);
    
    void delInvmt2(Investment* pInvestment) {
        makeLog(pInvestment);
        delete pInvestment;       
    }
    template<typename... Targs>
    std::unique_ptr<Investment, void(*)(Investment*)>   // Return type has sizeof(Investment*)+sizeof(void(*)(Investment*)).
    makeInvestment(Targs... args);

*std::unique_ptr*还可以用于Pimpl(编译防火墙)的实现，见Item22。

## Tips of *std::unique_ptr*

*std::unique_ptr*拥有一个用于数组的特化类型：*std::unique_ptr<T[]>*,所以可以清楚的区分出一个*std::unique_ptr*是指向数组还是对象，防止API的混用：指向对象的*std::unique_ptr*没有下标访问操作(operator[])，数组则没有解除引用的操作(operator*和operator->)。

但通常容器类(*std::array* *std::vector*)是替代原生数组更好的方案。

*std::unique_ptr*表达了*exclusive ownership*的语义，这可能局限了它的使用范围。但C++11提供更加实用和高效的方法，即*std::unique_ptr*可以转化为*std::shared_ptr*：
    
    std::shared_ptr<Investment> sp = makeInvestment(...);   // Converts std::unique_ptr to std::shared_ptr

所以*std::unique_ptr*十分适合用于工厂函数的返回值，因为工厂函数不管其生产的对象是共用的还是独有的。这样的转换使得*std::unique_ptr*的使用更加灵活。

## Things to Remember

- *std::unique_ptr*是一个小型的、快速的、move-only的智能指针，用于*exclusive ownership*的对象的资源管理。
- 默认，*std::unique_ptr*使用*delete*作为*deleter*，*deleter*可以自定义；同时*stateful deleter*和*function pointer*会提高*std::unique_ptr*的大小，*stateless deleter*则不。
- *std::unique_ptr*可以转化为*std::shared_ptr*