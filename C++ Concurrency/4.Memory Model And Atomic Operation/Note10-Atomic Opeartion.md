# Atomic operation

原子操作是一种无法再分割的操作。如果对一个*object*的操作是*atomic*的，那所有对该*object*的操作都是*atomic*的。

另一方面，非原子操作是可以被半路切入的。因为CPU对内存的访问是一个多步过程，其中还涉及到相关的缓存，因此不同线程对一个*object*的非原子操作（包含有写操作）会导致*data race*。

## The standard atomic types

在C++标准库中，对*atomic type*的操作不一定是真正意义上的原子操作，即不一定是*lock free*的，是在对外的表达意义上为*atomic*，有可能借助*mutex*来实现原子性。注意有些非成员函数模板用于实现非*std::atomic*特化类型的原子操作，在标准库中仅有一处用到，即*std::shared_ptr*。

绝大部分的*atomic type*都提供了*is_lock_free*接口来查询。只有*std::atomic_flag*没有提供*is_lock_free*，其实是一个bool型，并且保证*lock free*。

*std::atomic*提供了许多相应的原子操作，包括算术、逻辑、复合运算和位运算，并且对一个原子类型的可行操作和非原子类型一致。

*std::atomic*的类型不具备传统意义上的拷贝和赋值语义，但是可以和内建类型进行转换实现类似的语义。通过load和store返回或者保存当前原子类型的值。

所有原子类型的操作都不会返回*object*本身，而是返回一个内建类型来避免通过引用实现对原子类型的非原子操作，导致*data race*，并且原子操作都保有一个*std::memory_order*的参数。

## Operation on std::atomic_flag

*std::atomic_flag*是最简单的原子类型，并且保证*lock free*，本质是一个*boolean flag*，其具有两种状态：*set*和*clear*。

*std::atomic_flag*应该使用*ATOMIC_FLAG_INIT*初始化，初始状态为*clear*。

    std::atomic_flag f=ATOMIC_FLAG_INIT;

*atomic_flag*不可进行复制操作，只能无异常默认构造，也不提供*load*和*store*操作。一旦*std::atomic_flag*构造完成，只能对其进行下列三个操作：

- 析构销毁
- *clear*：将状态设置为*clear*，即*flag*为*false*
- *test_and_set*：将*flag*设置为*true*，并且返回设置前的*flag*

有限的操作使得*std::atomic_flag*的应用场景很狭窄，一大用途就是用于实现自旋锁*spin lock*：

    class spin_lock_mutex { 
    public:
        spin_lock_mutex() = default;
        spin_lock_mutex(const spin_lock_mutex&) = delete;

        void lock() {
            while(inside_atomic_flag_.test_and_set(std::memory_order_acquire));
        }

        void unlock() {
            inside_atomic_flag_.clear(std::memory_order_release);
        }

    private:
        std::atomic_flag inside_atomic_flag_ = ATOMIC_FLAG_INIT;
    };

自旋锁在资源被其他线程占据时不会像互斥锁一样被挂起进入休眠，而是不断的检查内部的*flag*直到锁被释放。因此自旋锁的效率要比互斥锁高，但是会长时间占据CPU。如果资源占据时间较长，自旋锁消耗的CPU资源会比互斥锁高得多，反而降低了程序效率。


