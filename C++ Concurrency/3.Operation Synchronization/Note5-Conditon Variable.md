# Condition variable

## Why do not use shared data to synchronize operation

并发编程中，经常会碰见多个线程之间的操作同步*operation synchronization*。实现操作同步很容易想到利用一个共享数据进行同步，即线程A会通过改变一个*flag*发出同步信号，使得线程B通过检查*flag*实现同步：

    bool flag;
    std::mutex flag_mutex;

    void wait_flag() {
        std::unique_lock<std::mutex> lk(flag_mutex);
        while(!flag) {
            lk.unlock();
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            lk.lock();
        }
        dosomething();
    }

线程B每隔100ms上一次锁并检查*flag*，如果*flag*为假，继续检查；如果*flag*为真，则执行*dosomething*。

但是这种设计有几大缺陷：

1. 检查时间间隔难以把握，如果时间太短，导致检查频繁，大量占用资源；如果时间太长，同步反应过慢，产生延迟。
2. 设计逻辑难看

## Condition variable

*std::condition_variable*提供了在一个线程唤醒其他线程的特性。*std::condition_variable*通过可以通过一个*std::mutex*来提供同步功能；*std::condition_variable_any*可以通过任何一个*mutex-like*对象进行同步，但需要消耗额外资源。

    std::mutex mut;
    std::queue<data_chunk> data_queue;
    std::condition_variable data_cond;
    // thread A
    void data_preparation_threadA() {
        while(more_data_to_prepare()) {
            data_chunk const data=prepare_data();
            std::lock_guard<std::mutex> lk(mut);
            data_queue.push(data);
            data_cond.notify_one();
        }
    }
    // thread B
    void data_processing_thread() {   
        while(true)
        {
            std::unique_lock<std::mutex> lk(mut) 
            data_cond.wait(lk,[]{return !data_queue.empty();});
            data_chunk data=data_queue.front();
            data_queue.pop();
            lk.unlock();        // unlock before processing
            process(data);
            if(is_last_chunk(data))
                break;
        }
    }

线程A作用是写入数据并在写入数据后唤醒线程B，线程B的作用是在线程A唤醒后执行数据处理。

线程A得到数据块后先申请队列的锁，上锁后将数据块*push*进入队列后，调用*notify_one*唤醒线程B，单次循环结束释放锁；

线程B进入循环，先要求申请队列的锁，然后调用*condition_variable*的*wait*，*wait*有两种形式：

    void wait( std::unique_lock<std::mutex>& lock );

    template< class Predicate >
    void wait( std::unique_lock<std::mutex>& lock, Predicate pred );

其中第二中实现相当于：

    while (!pred()) {
        wait(lock);
    }

每当调用*wait*时，wait先检验条件*pred*，如果*pred*为真则返回；否则调用*wait*第一种形式，释放锁并且阻塞当前线程直到被线程A唤醒，被唤醒后锁定并且继续检查*pred*，如果*pred*为真则返回。因为需要进行灵活的锁定控制，因此传入参数为*std::unique_lock*。

注意，在代码中调用一次*wait*时，*pred*可能会被调用任意次，不要使用一些带有边际效益的处理作为*pred*，因为存在*spurious wake*，即线程A并没有做唤醒操作，而线程B被唤醒了。

## Thread safe queue

    template<typename T>
    class threadsafe_queue {
    public:
        typename std::queue<T>::size_type size() const {
            std::lock_guard<std::mutex> lk(data_mutex);
            return data.size();
        }

        bool empty() const {
            std::lock_guard<std::mutex> lk(data_mutex);
            return data.empty();
        }

        void push(T new_val) {
            std::lock_guard<std::mutex> lk(data_mutex);
            data.push(new_val);
            data_condition.notify_one();
        }

        bool try_pop(T& val) {
            std::lock_guard<std::mutex> lk(data_mutex);
            if(data.empty())
                return false;
            val = data.front();
            data.pop();
            return true;
        }

        std::shared_ptr<T> try_pop() {
            std::lock_guard<std::mutex> lk(data_mutex);
            if(data.empty())
                return std::shared_ptr<T>();
            std::shared_ptr<T> ret = std::make_shared<T>(data.front());
            data.pop();
            return ret;
        }

        void wait_and_pop(T& val) {
            std::unique_lock<std::mutex> ulk(data_mutex);
            data_condition.wait(ulk, [this] { return !data.empty(); });
            val = data.front();
            data.pop();
        }

        std::shared_ptr<T> wait_and_pop() {
            std::unique_lock<std::mutex> ulk(data_mutex);
            data_condition.wait(ulk, [this] { return !data.empty(); });
            std::shared_ptr<T> ret(std::make_shared<T>(data.front()));
            data.pop();
            return ret;
        }
    private:
        std::queue<T> data;
        mutable std::mutex data_mutex;
        std::condition_variable data_condition;
    };

注意成员函数中存在*const*成员函数、复制构造函数和复制赋值操作符，这类操作都会涉及到*std::mutex*的上锁，但是对*const std::mutex*上锁是不可能的，所以应该使用*mutable*修饰*std::mutex*。

