# Waiting for one-off events

C++标准库使用*std::future*来标示一个一次性事件：

- *std::future*本身是用于提供一个链接异步操作的途径，可以通过：*std::async*|*std::packet_task*|*std::promist*来构造
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

## std::packet_task，associating a task with a future







