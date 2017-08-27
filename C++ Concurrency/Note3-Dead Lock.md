# Dead lock

## Dead lock problem and solution
当并发编程中，使用了多个*mutex*用于竞争资源的管理，就有可能使得多个线程同时出现无限期的阻塞，即*dead lock*。比如线程1占用了资源1并阻塞等待资源2，线程2占用了资源2并阻塞等待资源1，那线程1和2都会无限期阻塞下去。

为了解决*dead lock*问题，最直接的手段就是确定一个*mutex*的执行顺序，并严格按照其进行：如果总是将锁定*mutex A*放在锁定*mutex B*之前，那么就不会出死锁。在*mutexes*具有不同的用途时，这种方法能够工作的很好。

但是存在多个*mutex*用于一个用途的情况，最常见的就是一个类型的不同实例。比如存在有两个实例进行交换的友元操作*swap*:

    class obj;
    void swap(obj&,obj&);

    class X {
    private:
        obj data;
        std::mutex m;
    public:
        X(const obj& _data) : data(_data) {}
        friend void swap(X& lhs, X& rhs) {
            if(&lhs == &rhs)
                return;
            std::lock_guard<std::mutex> g1(lhs.m);
            std::lock_guard<std::mutex> g2(rhs.m);
            swap(lhs.data, rhs.data);
        }
    };

*swap*操作总是先锁定第一参数的*mutex*，后锁定第二参数的*mutex*。当有两个线程同时执行了对两个实例进行*swap*操作，但是参数顺序相反，就会出现死锁。为了解决该问题，标准库提供了*std::lock*解决该问题：

    class X {
    public:
        friend void swap(X& lhs, X& rhs) {
            if(&lhs == &rhs)
                return;
            std::lock(lhs.m, rhs.m);            // 1
            std::lock_guard<std::mutex> g1(lhs.m, std::adopt_lock);     // 2
            std::lock_guard<std::mutex> g2(rhs.m, std::adopt_lock);     // 3
            swap(lhs.data, rhs.data);
        }
    }

代码1用于锁定两个*mutex*并且防止死锁，代码2和3用于创建*RAII*形式的*mutex*管理，*std::adopt_lock*指明*mutex*已经锁定，*std::lock_guard*只需要接管*mutex*即可。这样就可以保证函数不论是正常退出还是异常退出时，*mutex*被正确解锁；*std::lock*也有可能抛出异常，但是其保证抛出异常后所有*mutex*处于*unlock*状态。

*std::lock*可以帮助解决多个*std::mutex*同时锁定时的*dead lock*问题，但是对于分离式的无能为力。多线程编程时遵循一些原则可以帮助实现*deadlock-free*的代码。

## Further guideline for avoiding deadlock




### Avoid nested locks



### Avoid calling user-supplied code while holding a lock




### Acquire locks in a fixed order



### Use a lock hierarchy


### Extending these guidelines beyond locks


