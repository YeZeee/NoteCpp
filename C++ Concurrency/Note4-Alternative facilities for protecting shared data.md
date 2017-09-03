# Alternative facilities for protecting shared data
## Lock at an appropriate granularity

一个良好的多线程数据的互斥设计应该至少有以下两方面：

1. 锁的数据保护范围应该恰好只保护需要该锁保护的数据。（大小）
2. 锁应当只保护那些需要其保护的操作。（时间）

>Not only is it important to choose a sufficiently coarse lock granularity to ensure the required data is protected, but it’s also important to ensure
that a lock is held only for the operations that actually require it.

即，如果多个线程都在等待同一个资源，如果一个线程在其不需要该资源的情况下持有该资源的互斥锁，就会增加整个系统的时间开销。因此在持有锁的情况下：不要进行高时间消耗的行为，比如I/O。

> In particular, don’t do any really time-consuming activities like file I/O while
holding a lock.

*std::unique_lock*的灵活性使得能够在必要的时候释放锁，进行不相关的耗时操作：

    void get_and_process_data()
    {
        std::unique_lock<std::mutex> my_lock(the_mutex);
        some_class data_to_process=get_next_data_chunk();
        my_lock.unlock();
        result_type result=process(data_to_process);    // 1
        my_lock.lock();
        write_result(data_to_process,result);
    }

因为在代码1时，不用锁相关的资源，可以先释放锁，让其他线程可以访问该资源。

> In general, a lock should be held for only the
minimum possible time needed to perform the required operations

为了多线程的效率，应该是锁尽可能的持续最短的时间，比如进行两个int型的比较，由于*int*的拷贝成本很低，可以将两者先拷贝再比较拷贝副本的的大小，避免在比较过程中持有两者的锁：

    class Y
    {
    private:
        int some_detail;
        mutable std::mutex m;
        int get_detail() const {
            std::lock_guard<std::mutex> lock_a(m);
            return some_detail;
        }
    public:
        Y(int sd):some_detail(sd){}
        friend bool operator==(Y const& lhs, Y const& rhs) {
            if(&lhs==&rhs)
                return true;
            int const lhs_value=lhs.get_detail();
            int const rhs_value=rhs.get_detail();
            return lhs_value==rhs_value;
        }
    };

值得注意的是，通过以上的代码虽然减少了锁的持有时间，同时避免了死锁的出现，但同时也改变了这个操作的语义。

- 原语义，锁定两个*int*后，比较锁定之后的值并返回比较结果并释放锁。
- 后语义，锁定一个*int*，取副本，释放锁；锁定另一个*int*，取副本，释放锁；比较两个副本的值并返回结果。

这样两种语义下，其他线程可进行的操作是不一样的；比如前者无法在该操作中插入*swap*这样的操作，然而后者可以在两次取副本之间进行这样的操作:

>if you don’t hold the required locks for the entire duration of an operation, you’re exposing yourself to
race conditions

互斥设计就是要找到一个合适的*granularity*，但是这种设计可能是不存在的，因为不同的线程需要的数据保护等级不同，仅仅靠基本的*std::mutex*难以实现。C++标准库提供了一些替代*std::mutex*和*lock*的机制。

## Protecting shared data during initialization

有一些初始化是十分消耗资源的，所以实现总是希望在真正需要的时候进行这样的初始化，比如打开数据库，申请大块内存等等操作。

*lazy initialization*是一种常见的单线程代码：在使用资源前检查资源是否初始化，如若未初始化则初始化后使用：

    std::shared_ptr<resource> resource_ptr;
    void foo() {
        if(!resource_ptr)
            resource_ptr.reset(std::make_shared<resource>());
        resource_ptr->do();
    }

如果资源类型*resource*自身能够保证线程安全，那上述代码转化成多线程代码只需要保护初始化即可：

    std::shared_ptr<resource> resource_ptr;
    std::mutex resource_mutex;
    void foo() {
        std::unique_lock<std::mutex> lk(resource_mutex);
        if(!resource_ptr)
            resource_ptr.reset(std::make_shared<resource>());
        lk.unlock;          // because resource itself is safe for concurrency.
        resource_ptr->do();
    }

因为资源类型本身保证并发安全，所以在初始化之后就可以解开锁。

但上述代码有一个致命的缺陷，因为所有线程执行*foo*，都必然会先上锁，后检查资源是否初始化完成，这显然在初始化完毕后是不必要的，违反了多线程设计的原则：

>Not only is it important to choose a sufficiently coarse lock granularity to ensure the required data is protected, but it’s also important to ensure
that a lock is held only for the operations that actually require it.

为了解决这个问题，有许多“更好”的办法被应用，比如*infamous*的*double checked locking*模式：

    void UB_with_double_checked_locking() {
        if(!resource_ptr) {
            std::unique_lock<std::mutex> lk(resource_mutex);
            if(resource_ptr) {
                resource_ptr.reset(std::make_shared<resource>());
            }
        }
        resource_ptr->do();
    }

因为外部指针检查和内部的资源初始化不同步，有可能内部资源的初始化尚未完成，但指针检查已经指示资源已经初始化完成（空间分配完成），所以其他线程可能会在初始化未完成时执行*do*，造成*data race*，是一种未定义行为。

C++标准库提供了*std::once_flag*和*std::call_once*来处理这种情况，使用*std::once*一般可以比使用*lock*占用更少的消耗，特别是初始化完成后：

    std::shared_ptr<resource> resource_ptr;
    std::once_flag resource_flag;

    void init_resource() {
        resource_ptr.reset(std::make_shared<resource>());
    }

    void foo() {
        std::call_once(resource_flag, init_resource);
        resource_ptr->do();
    }

*std::call_once*保证同一个*std::once_flag*对象指示的可调用对象在多个线程中只会执行一次：

- 可调用对象的参数由被选中执行的线程的*std::call_once*传入的参数决定。
- 异常引起的函数退出不会归入“执行一次”的范畴。
- 如果某线程被选中执行的函数因为异常退出，并会返回给调用者。其他线程中的函数会被选中继续执行，知道成功执行一次。

*std::call_once*也可以用于成员变量的初始化：

    class X {
    public:
        void send(const data_packet& data) {
            std::call_once(connection_init_flag, &X::open_connection, this);
            connection.send(data);
        }
        void receive(data_packet& data) {
            std::call_once(connection_init_flag, &X::open_connection, this);
            connection.receive(data);
        }
    private:
        connection_info connection_details;
        connection_handle connection;
        std::once_flag connection_init_flag;
        void open_connection() {
            connection = connection_manager.open(connection_details);
        }
    };

另一个可能出现*data race*的初始化场景就是静态局部变量，因为静态局部变量在运行至其声明初进行初始化，不同的线程会出现*data race*，通过*std::call_once*可以避免：

    class my_class;
    std::once_flag my_class_flag;
    my_class& get_my_class_instance() {
        static my_class instance;
        return instance;
    }

    void foo() {
        std::call_once(my_class_flag, get_my_class_instance);
        ...
    }

## Protecting rarely updated data structures

“仅在初始化保护”其实是“保护稀少更新数据”的一种特殊情况。比如一样数据长时间只更新一次或者很少更新，平常表现基本等同于*read—only*数据。比如：

DNS储存着“域名-IP”的表，这种数据长时间不会进行更新。因此除去“写操作”以外，其他所有“读操作”可以同时访问数据并且不会出现*race condition*。那么就希望：

- “写操作”：只允许当前线程对数据进行访问，即*exclusive*。
- “读操作”：允许其他线程一起对数据进行访问，即*shared*。

如果使用*std::mutex*，因为*std::mutex*只有一种锁定状态，即“exclusive”。需要其他类型的*mutex*替代。

C++17提供了*std::shared_mutex*用于该控制（C++ concurrency in action举的是boost库的内容，因为那时*std::shared_mutex*未被纳入标准库），其具有两种锁定状态*shared*和*exclusive*：

    #include <map>
    #include <string>
    #include <mutex>
    #include <shared_mutex>

    class dns_entry;
    class dns_cache {
    private:
        std::map<std::string, dns_entry> entries;
        mutable std::shared_mutex entry_mutex;
    public:
        dns_entry find_entry(const std::string& domain) const {
            std::shared_lock<std::shared_mutex> slk(entry_mutex);
            auto it = entries.find(domain);
            return (it == entries.end())?dns_entry():it->second;
        }  // read operation

        void update_or_add_entry(const std::string& domain, const dns_entry& dns_details) {
            std::lock_guard<std::shared_mutex> lk(entry_mutex);
            entries[domain] = dns_details;
        }
    };

## Recursive locking

对已经锁定的*std::mutex*再次进行锁定是UB，但有些情况下期待能够递归锁定一个互斥器。C++标准库提供了*std::recursive_mutex*。一个线程可以反复给*std::recursive_mutex*多次上锁，但同时在释放锁时也是进行同样次数的解锁。比如一个类的成员函数彼此发生调用，使用*std::mutex*会导致UB，使用*std::recursive_mutex*可以避免。

多数情况下，*recursive_mutex*都是可以用其他*mutex*配合良好的设计替代，尽量不要使用*recursive_mutex*造成设计混乱。
