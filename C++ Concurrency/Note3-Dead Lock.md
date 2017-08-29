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

*deadlock*不仅仅发生在*lock*的情况下，只要线程之间有可能出现存在彼此等待对方的情况就会存在死锁的情况，利用*join*就能创造出死锁：

    void g(std::thread& t) {
        std::this_thread::sleep_for(2s);
        t.join();
    }

    int main() {
        std::this_thread::sleep_for(2s);
        std::thread t1;
        std::thread t2(g, std::ref(t1));
        t1 = std::thread(g, std::ref(t2));
        while(t1.joinable()||t2.joinable())
            ;
    }

主线程创建了两个子线程，两个线程都在等待对方的join(),从而出现*deadlock*。

### Avoid nested locks

避免嵌套的*lock*，可以完全避免由锁定导致的*deadlock*。当使用多个*lock*时，使用*std::lock*来避免死锁。

### Avoid calling user-supplied code while holding a lock

在存在*lock*的情况下，避免使用用户提供的代码。这和避免*lock*嵌套是一个道理，因为用户代码可能会涉及*lock*操作。但是有时这也是无法避免的，比如泛型编程和函数式编程，永远不可能知道用户提供的类型和操作是怎样的。

### Acquire locks in a fixed order

如果的确需要进行多个*lock*操作，并且无法应用*std::lock*，就应当遵循“在所有线程中按固定顺序进行*lock*相关操作”的原则。

有时这种要求的实现是很简单的，比如之前的*thread_safe stack*。因为*stack*会调用用户代码，所以只要用户代码不会进行改变*stack*的结构就能保证锁定的稳定顺序。

但有时就很难保证，具有隐藏风险导致锁定顺序被破坏，比如之前因为用户代码的不同就会导致*swap*死锁。这种情况可以使用*std::lock*解决，但是有些情况就不行。比如使用一系列的*mutex*分别保护双向链表的每个节点，这样可以实现多个线程同时访问一个链表。所有操作总是先锁定靠前节点再锁定靠后节点，可以避免死锁的情况，但是这就不能实现反向遍历链表。

### Use a lock hierarchy

使用有层次的*mutex*，即使得*mutex*自身具有限制顺序的能力，比如不能够在拥有低层的*mutex*的情况锁定高层的*mutex*。

    hierarchical_mutex high_level_mutex(10000);
    hierarchical_mutex low_level_mutex(5000);
    int do_low_level_stuff();
    int low_level_func() {
        std::lock_guard<hierarchical_mutex> lk(low_level_mutex);
        return do_low_level_stuff();
    }

    void high_level_stuff(int some_param);
    void high_level_func() {
        std::lock_guard<hierarchical_mutex> lk(high_level_mutex);
        high_level_stuff(low_level_func());
    }

    void thread_a(){
        high_level_func();
    }

    hierarchical_mutex other_mutex(100);
    void do_other_stuff();
    void other_stuff() {
        high_level_func();
        do_other_stuff();
    }

    void thread_b() {
        std::lock_guard<hierarchical_mutex> lk(other_mutex);
        other_stuff();
    }

*thread_a*可以正常工作，而*thread_b*就会触发*hierarchy_mutex*的异常。





### Extending these guidelines beyond locks


