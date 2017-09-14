# Waiting for one-off events

C++标准库使用*std::future*来标示一个一次性事件：

- *std::future*本身是用于提供一个链接异步操作的途径，可以通过：*std::async*|*std::packaged_task*|*std::promist*来构造
- *std::future*可以绑定一个异步操作返回的数据块，*std::future*的持有线程可以通过*wait*|*get*|*valid*等方式进行操作，如果异步操作尚未产生返回，则该方法会产生阻塞。

## std::async,returning values from background task

*std::async*通过接受*function object*产生一个*std::future*。*std::async*有一个控制选项：

- std::launch::async
- std::launch::deferred
- std::launch::async|std::launch::deferred or default.

选项一，总是启用新线程执行*callable object*（asynchronous execution）；  
选项二，在当前线程，第一次获取结果时执行*callable object*（lazy evaluation）；  
选项三，依赖实现。

如下代码实现并行加法：

    template<typename Iterator>
    struct add_block {
        auto operator()(const Iterator _start, const Iterator _stop) {	
            return std::accumulate(_start, _stop, 0);
        }
    };

    template<typename Iterator>
    auto parallel_add(const Iterator _begin, const Iterator _end, typename Iterator::value_type _init) {
        auto arraySize = std::distance(_begin, _end);
        if(!arraySize)
            return _init;
        size_t maxNumHardwareThread = std::thread::hardware_concurrency();
        size_t minAmountPerBlock = 25;
        size_t maxNumSoftwareThread = (arraySize + minAmountPerBlock - 1) / minAmountPerBlock;
        auto numThread = std::min(maxNumSoftwareThread == 0?1:maxNumSoftwareThread, maxNumHardwareThread);
        auto amountPerBlock = arraySize / numThread;
        std::vector<std::future<typename Iterator::value_type>> vecResultPerFuture;
        vecResultPerFuture.reserve(numThread - 1);
        auto _start = _begin;
        auto _stop = _start;
        for(int i = 0; i < numThread - 1; i++) {
            std::advance(_stop, amountPerBlock);
            vecResultPerFuture.emplace_back(std::async(std::launch::async,add_block<Iterator>(), _start, _stop));
            std::advance(_start, amountPerBlock);
        }
        std::vector<typename Iterator::value_type> vecResult;
        vecResult.resize(numThread);
        for(int i = 0; i < numThread - 1;i++) {
            vecResult[i] = vecResultPerFuture[i].get();
        }
        vecResult[numThread - 1] = std::accumulate(_start, _end, _init);
        return std::accumulate(std::begin(vecResult), std::end(vecResult), 0);
    }

## std::packaged_task，associating a task with a future

*std::packgaed_task*可以包装一个*callable object*来使得其可以异步调用，返回内容通过*std::future*来访问。可以用于构建线程池；以及其他任务管理。  
*std::packaged_task*的模板参数是一个函数类型（注意函数类型中存在隐式转换）。  
*std::packaged_task*自身也是一个*callable object*，可以进一步封装也可以直接调用。  
*std::packaged_task*是*movable*和*swapable*的。

    std::mutex m;
    std::deque<std::packaged_task<void()> > tasks;
    bool gui_shutdown_message_received();
    void get_and_process_gui_message();
    void gui_thread()
    {
        while(!gui_shutdown_message_received())
        {
            get_and_process_gui_message();
            std::packaged_task<void()> task;
            {
                std::lock_guard<std::mutex> lk(m);
                if(tasks.empty())
                continue;
                task=std::move(tasks.front());
                tasks.pop_front();
            }
            task();
        }
    }
    std::thread gui_bg_thread(gui_thread);
    template<typename Func>
    std::future<void> post_task_for_gui_thread(Func f)
    {
        std::packaged_task<void()> task(f);
        std::future<void> res=task.get_future();
        std::lock_guard<std::mutex> lk(m);
        tasks.push_back(std::move(task));
        return res;
    }

来自书上的例子，*gui_thread*通过一个任务队列接收任务并且执行；*post_task_for_gui_thrad*通过向一个任务队列加入任务来发布任务，并且将该任务的*future*返回给外层，注意*get_future*对于一个任务只能调用一次。

## std::promise, make promises

*std::promise*用于设置一个异步接受的值，*movable*。

    void accumulate(std::vector<int>::iterator first,
                    std::vector<int>::iterator last,
                    std::promise<int> accumulate_promise)
    {
        int sum = std::accumulate(first, last, 0);
        accumulate_promise.set_value(sum);  // Notify future
    }
    
    void do_work(std::promise<void> barrier)
    {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        barrier.set_value();
    }
    
    int main()
    {
        // Demonstrate using promise<int> to transmit a result between threads.
        std::vector<int> numbers = { 1, 2, 3, 4, 5, 6 };
        std::promise<int> accumulate_promise;
        std::future<int> accumulate_future = accumulate_promise.get_future();
        std::thread work_thread(accumulate, numbers.begin(), numbers.end(), std::move(accumulate_promise));
        accumulate_future.wait();  // wait for result
        std::cout << "result=" << accumulate_future.get() << '\n';
        work_thread.join();  // wait for thread completion
        std::promise<void> barrier;
        std::future<void> barrier_future = barrier.get_future();
        std::thread new_work_thread(do_work, std::move(barrier));
        barrier_future.wait();
        new_work_thread.join();
    }

来自cppreference的例子，在当前线程使用一个*std::future*获取一个*std::promise*的未来结果，让后将*promise*移动给新线程。新线程执行完成后设置*promise*的值，当前线程可以通过*future*获得该值。

*std::promise\<void>*用于不需要任何值的情况，可以单纯的用于阻塞或者定时或者标示某些代码块已经执行完成。

## Saving exception for the future

*std::future*除了返回值以外也可以返回异常，将异常抛到当前*std::future*所在线程进行处理，有以下几种情况会使得*std::future*产生异常。

- *std::async*和*std::packaged_task*执行的代码块抛出异常。
- *std::promise*设置异常。
- *std::packaged_task*和*std::promise*进行了非法操作。

使用get(),重新抛出异常，比如情况一:

    double sqrt_root(int i) { 
        if(i < 
        0)
            throw std::out_of_range("i<0");
        return sqrt(i);
    }

    void foo() {
        std::future<double> f = std::async(std::lauch::async, sqrt_root, -1);
        try {
            f.get();
        } catch(const std::out_of_range& e){
            std::cout << "out_of_range: " << e.what() << std::endl;
        }
    }

情况二：

    void some(std::promise<double> p) {
        try {
            p.set_value(some_func_may_throw());

        } catch(const std::exception& e) {
            p.set_exception(std::current_exception());
        }
    }

    void foo() {
        std::promise<double> p;
        std::future<double> f = p.get_future();
        std::thread t1(some, std::move(p));
        try {
            f.get();
        } catch(const std::exception& e) {
            std::cout << "out_of_range: " << e.what() << std::endl;
        }
    }

抛出*std::future_error*，情况三：

    void foo() {
        std::future<double> f;
        {
            std::promise<double> p;
            f = p.get_future();
        }
        try {
            f.get();
        } catch(cosnt std::future_error& e) {
            if(e.code() == std::future_errc::broken_promise)
			std::cout << "exception: " << e.what() << std::endl;
        }
    }


## Waiting from multiple threads

*std::future*用于在线程间（或者非线程间）一次性传递各种结果，并且对一个实例的操作没有任何同步行为。如果在多个线程对同一个*std::future*进行操作就会产生*data race*。  
因此*std::future*被设计对持有数据为*unique ownership*，一次绑定只能进行一次*get*操作，不应该在多个线程引用一个*std::future*实例。  

为了满足多个线程获取同一个消息，*std::shared_future*被设计为*shared ownership*。*std::future*只能被*move*，数据可以被传递并且只能被*get*一次。而*std::shared_future*可以被*copy*，多个*future*可以关联同一块数据。

注意为了多个线程访问一块数据不出现*data race*，应该在不同线程*copy*一个*std::shared_future*而不是引用它，因为对于一个实例的不同成员函数并没有同步，一个线程只应该保留一个*std::shared_future*。

    std::promise<std::string> p;
    std::shared_future<std::string> sf(p.get_future());

    std::promise<int> p;
    std::future<int> f(p.get_future());
    std::shared_future<int> sf(std::move(f));

使用auto来推断*std::shared_future*类型：

    std::promise< std::map< SomeIndexType, SomeDataType, SomeComparator,
    SomeAllocator>::iterator> p;
    auto sf=p.get_future().share()


