# Lock at an appropriate granularity

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

>In general, a lock should be held for only the
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

互斥设计就是要找到一个合适的*granularity*，但是这种设计可能是不存在的，因为不同的线程需要的数据保护等级不同，仅仅靠基本的*std::mutex*难以实现。

## Alternative facilities for protecting shared data







