# Sharing data between threads

## Problem with sharing data between threads

当多个线程的共享数据是只读*readonly*的情况，是不会产生额外要分析的问题的。但当多个线程其中包含又对共享数据写权限的线程时候，就会产生额外的问题。

在编程过程中，往往有许多不变式*invariants*帮助我们理解代码的含义，但是这种不变式往往会在数值更新时被破坏，越是复杂的数据类型，越是在更新时进行多的操作的类型，这个破坏不变式的过程越容易在多线程编程中引发问题。

比如一个双向链表*doubly linked list*包含不定式：

- 节点A的后向指针指向节点B，则有节点B的前向指针指向节点A。

在某些链表操作就会短时间内破坏这个不定式，如删除节点N：

1. 找到节点N
2. 将节点N的前节点的后向指针指向节点N的后节点
3. 将节点N的后节点的前向指针指向节点N的前节点
4. 删除节点N

在步骤2-3之间，双向链表的一个重要的不定式被破坏。若在其他线程的逻辑中有用到该不定式的代码，该代码将会产生未定义后果。

这是在并发编程中常见的BUG原因：*race condition*，也称*data race*。

### Race condition and avoiding problematic race conditions

*race condition*通常指代那些会导致BUG的竞争条件，其带来的BUG通常是未定义行为。

*race condition*主要是各线程对共享数据的修改导致的。其带来的BUG对时间十分敏感，在DEBUG的模式下很容易被忽略。当修改操作被连续的计算机指令执行时，*race condition*的概率较低；随着计算机负荷的上升，有问题的竞争操作的发生概率也随之提高。

解决*race condition*的主要办法有以下几种：

1. 使用保护机制封装共享数据，保证只有一个线程能改变数据并且其他线程只能在该操作尚未发生或者已经结束的情况下访问数据，即*mutex*。
2. 改造共享数据结构的设计，使得该结构进行的修改操作不会破坏数据结构的不变性，即*lock-free programing*。
3. *software transactional memory*，STM

## Protecting shared data with mutexes

解决*race condition*的一种通用方法就是将访问共享数据的代码块设置为互斥，即*mutually exclusive*。这样一来当有线程访问共享数据，其他线程就必须等待访问结束后再进行访问。

这一过程使用*mutex*来实现，在访问共享数据前，*lock*一个和该数据关联的*mutex*；在访问结束后，*unlock*这个*mutex*。

标准线程库保证了当一个线程锁定一个*mutex*，其他试图锁定该*mutex*的线程都要等候至该*mutex*被解锁。

互斥器*mutex*听起来十分简单，但非银弹：
1. 构造好代码结构保护目标数据
2. 防止*race condition*在接口之间传递
3. *mutex*自身存在*deadlock*等问题

### Using mutexes in C++

C++标准库通过*std::mutex*实现*mutex*：
- *lock()*锁定*mutex*，当不能锁定时阻塞。
- *unlock()*解锁*mutex*。

并且有*RAII*风格的*std::lock_guard*来替代手动的*unlock*。

    class listForThread {
    public:
        void pushToList(int _val) {
            std::lock_guard<std::mutex> lguark(m_mutexForList);
            m_list.push_back(_val);
        }
        bool findInList(int _val) {
            std::lock_guard<std::mutex> lguark(m_mutexForList);
            return std::find(std::begin(m_list), std::end(m_list), _val) != m_list.end();
        }
    private:
        std::list<int> m_list;
        std::mutex m_mutexForList;
    };

通过上述代码，实现了*list*的*find*和*push*两种操作的互斥。

但是需要非常注意的是，即使所有操作都使用*std::mutex*实现操作互斥，但是函数接口仍然有可能会将*race condition*泄露。比如：

    class listForThread {
    public:
        typename std::list<int>::iterator beginOfList() {
            std::lock_guard<std::mutex> lguark(m_mutexForList);
            ...
            return m_list.begin();
        }
    }

该代码将内部数据的一个迭代器向外传递。外部代码则可以绕过类接口直接访问内部链表，导致*race condition*。因此函数接口涉及指针、引用和迭代器时，需要格外注意产生*race condition*的泄露。良好的互斥设计应该包含整个数据访问的过程，而不是仅仅是简单的函数开头与结尾。

### Structuring code for protecting shared data

使用*mutex*保护数据的代码设计十分复杂，不仅仅是使用*std::lock_guard*和注意函数是否有指针引用接口就完事的。某些隐藏性很强的接口也有可能会使得*race condition*泄露：

    class data {
    public:
        void some_func();
    prvate:
        int i;
    };

    class data_wraper {
    public:
        template<typename Func>
        void someFunc(Func f) {
            std::lock_guard<std::mutex> g(m_mutex);
            f(m_data);
        }
    private:
        data m_data;
        std::mutex m_mutex;
    };

    data* p;
    void maliciousFunc(data& _d) {
        p = &_d;
    }

    void foo() {
        data_wraper d;
        d.someFunc(maliciousFunc);
        p->somefunc();
    }

这种*race condition*泄露的问题的根本原因是没有做到：将所有涉及访问目标数据块的操作标示为互斥。因此使用*std::mutex*遵守以下原则以避免*race condition*外泄：

- 不要将被保护数据的指针和引用传出*mutex lock*的范围，不论是显式的还是隐式，比如传参、返回、传递给未知实现的函数。

### Spotting race conditions inherent in interfaces

即使严格遵守了上述原则，函数接口依旧有可能导致潜在的*race condition*，这是由于函数本身的操作决定的，不仅仅发生在互斥设计中，也会发生在*lock-free programing*中。

因为并发实现中，可能会使某些函数失去原本的意义，该函数所得到的结果是不可信的，比如*stack*：

    template<typename T,typename Container=std::deque<T> >
    class stack
    {
    public:
        explicit stack(const Container&);
        explicit stack(Container&& = Container());
        template <class Alloc> explicit stack(const Alloc&);
        template <class Alloc> stack(const Container&, const Alloc&);
        template <class Alloc> stack(Container&&, const Alloc&);
        template <class Alloc> stack(stack&&, const Alloc&);
        bool empty() const;
        size_t size() const;
        T& top();
        T const& top() const;
        void push(T const&);
        void push(T&&);
        void pop();
        void swap(stack&&);
    };


栈的操作只有5种，在并发编程中，函数*empty*和*size*将不再可信。因为其一旦完成了操作，则其他线程就可以访问改变*stack*，在该线程所得的信息将失去意义。假设*stack*的5种操作都加上了互斥保护，存在以下接口的组合使用：

    stack<int> s;
    if(!s.empty()) {
        const int i = s.top();  // 1
        s.pop();                // 2
        do_something(i);
    }

其他线程的操作可以插入上述每一行代码之间，导致未定义问题，甚至使程序崩溃。比如在代码1和2之间可以插入*pop*使得源代码的遍历功能失效。

想要将*pop*和*top*纳入互斥保护，就需要有新的实现接口来使得两个操作纳入一个互斥保护，但这又引入了新的异常安全的问题：

比如存在数据类型*stack\<vector\<int\>\>*。接口*topandpop*为了互斥保护，同时实现了*pop*和*top*的功能。但是*topandpop*为了返回被*pop*的对象，必须调用*std::vector*的构造函数，这一过程可能会抛出异常，但是抛出异常时，*stack*已经被改变。分离的*pop*和*top*操作不会有这样的问题，因为当*top*操作抛出异常时，*pop*操作还没有执行。为了解决*race condition*和*exception safety*纠缠的问题，需要设计好函数接口，有如下几种方法：

#### Pass a reference

为了使得构造函数不在接口内部被调用，可以使用引用参数作为返回接口：

    std::vector<int> result;
    some_stack.pop(result);

这样就要求用户代码必须先自行进行目标对象的构造，并且要求返回类型是可赋值类型。

#### Require a nothrow copy constructor or move constructor

许多类型的复制构造函数是*nothrow*的，即使复制构造函数会抛出异常，还有许多类型的移动构造函数也是*nothrow*的。可以使用编译期检查*std::is_nothrow_copy_constructible*和*std::is_nothrow_move_constructible*。

#### Return a pointer to the popped item

可以使用指针返回替代值返回，这样可以使得构造函数在改变*stack*之前被调用。但是这就涉及到了内存管理，而且对于简单类型而言，这样的开销甚至超出了简单的值返回。可以使用智能指针作为返回类型。

如下实现了引用参数返回和智能指针返回，解决了*race condition*和*exception safety*纠缠的问题：

    struct empty_stack: std::exception{
        const char* what() const noexcept;
    };

    template<typename T>
    class threadsafe_stack {
    public:
        threadsafe_stack() {}
        threadsafe_stack(const threadsafe_stack& _stack) {
            std::lock_guard<std::mutex> g(_stack.m);
            m_stack = _stack.m_stack;
        }
        threadsafe_stack& operator=(const threadsafe_stack&) = delete;

        void push(T _val) {
            std::lock_guard<std::mutex> g(m_mutex);
            m_stack.push(_val);
        }
        std::shared_ptr<T> pop() {
            std::lock_guard<std::mutex> g(m_mutex);
            if(m_stack.empty())
                throw empty_stack();
            std::shared_ptr<T> ret(std::make_shared<T>(m_stack.top()));
            m_stack.pop();
            return ret;
        }
        void pop(T& _val) {
            std::lock_guard<std::mutex> g(m_mutex);
            if(m_stack.empty())
                throw empty_stack();
            _val = m_stack.top();
            m_stack.pop();
        }
        bool empty() const {
            std::lock_guard<std::mutex> g(m_mutex);
            return m_stack.empty();
        }

    private:
        std::mutex m_mutex;
        std::stack<T> m_stack;
    };

实现去除了*copy assginment operator*，保留了*copy constructor*。同时在*copy constructor*也使用了互斥保护。

以上描述的都是因为*mutex*保护范围不够充分导致*race condition*的情况，但是如果*mutex*保护范围过大，会导致线程阻塞时间过长，不仅抹除并发所带来的性能提升，甚至导致其性能还不如单线程实现。

当程序需要多个*mutex*协作阻塞时，比如同一个类的不同实例，有可能出现多个线程都被*mutex*阻塞，等待其他线程，即死锁*deadlock*。

## Deadlock：the problem and solution






