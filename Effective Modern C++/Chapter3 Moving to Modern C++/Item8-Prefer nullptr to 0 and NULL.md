# Item8: Prefer nullptr to 0 and NULL

在C++中，字面值0是一个int，在contxet中，0有可能被解释为null pointer。
但是这是后置位的情形，0依旧是一个int而不是null pointer。

NLL这个宏依赖于实现，有可能是int(0),也有可能是long(0)，NULL对于指针也具有一样的问题。

在C++98中，对于指针和整数的参数重载具有存在陷阱：

    void f(int);
    void f(bool);
    void f(void*);

    f(0);       // Call f(int).

    f(NULL);    // Depend on implementation, might not complie. but  
                // never calls f(void*).

因为NULL的实现可以是0，也可以是0L，而0L转换给void*，int，bool是平等的，导致歧义，造成报错。

另一方面，nullptr的好处在于其不具有整数值，它能够转换为指向任意类型的null pointer。使用nullptr，就能避免上述的重载问题。

    f(nullptr)  // Call f(void*).

同时使用nullptr，也可以增强代码可读性。

    auto result = find(/*arg*/);

    if(result != 0)...

    if(result != nullptr)...

很显然下面的代码表面了result是一个指针。

当模板进入代码时，nullptr的作用更加明显：

    int f1(std::shared_ptr<Widget> spw);
    double f2(std::unique_ptr<Widget> upw);
    bool f3(Widget* pw);

    std::mutex f1m, f2m, f3m;

    using MuxGuard = std::lock_guard<std::mutex>;
    ...
    {
        MuxGuard g(f1m);
        auto result = f1(0);
    }
    ...
    {
        MuxGuard g(f2m);
        auto result = f2(NULL);
    }
    ...
    {
        MuxGuard g(f3m);
        auto result = f3(nullptr);
    }
以上代码具有高度的重复性，可以将其模板化：

    template<typename FuncType, typename MuxType, typename PtrType>
    decltype(auto) lockAndCall(FuncType func,
                                MuxType mutex,
                                PtrType ptr) {
        using MuxGuard = std::lock_guard<MuxType>;
        MuxGuard g(mutex);
        return func(ptr);
    }
    
    auto result1 = lockAndCall(f1, f1m, 0);         // error!
    auto result1 = lockAndCall(f2, f2m, NULL);      // error!
    auto result1 = lockAndCall(f3, f3m, nullptr);   // fine!
    
在第一个模板函数中，PrtType被推断为int，这就导致在模板内部func要接收一个int，
对于f1来说，相当于用一个int去初始化shared_ptr\<Widget>，这是错误的(因为0可以指代指针，但是int是不可以的)，对于第二个也是类似的情况。

而nullptr是没有这方面的问题的。传入nullptr时，PrtType被推断为std::nullptr_t。而nullptr_t是可以转化为Widget*和shared_ptr\<Widget>的。

## Things to Remember

- 使用nullptr代替NULL和0。
- 避免重载整数和指针类型。
