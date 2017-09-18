# Using synchronization of operations to simplify code

C++标准库提供了许多同步操作，通过这些同步操作可以简化多线程的代码并且提高可读性。以下是一些适合于多线程的编程范式。

## Functional programing with future

C++11中加入了许多函数式编程（*functional programing*）需要的特性，包括*lambda*、自动类型推断*auto*、更好用的合并*std::bind*。因为严格的函数没有副作用，只需要关心输入和输出，可以极大避免*race condition*。  
*std::future*的特性，使得多线程编程中使用函数范式更加简单，可以更加简单的将单线程函数式代码改造成多线程函数式代码：  

比如一个快排程序：

    template<typename T>
    std::list<T> sequential_qsort(std::list<T> input) {
        if(input.empty())
            return input;
        std::list<T> result;
        result.splice(result.begin(), input, input.begin());
        const T& pivot = *result.begin();
        auto divide_point = std::partition(input.begin(), input.end(), [&pivot](const T& t) { return t < pivot; });

        std::list<T> lower_group;
        lower_group.splice(lower_group.end(), input, input.begin(), divide_point);

        auto lower(sequential_qsort(std::move(lower_group)));
        auto higher(sequential_qsort(std::move(input)));

        result.splice(result.end(), higher);
        result.splice(result.begin(), lower);
        return result;
    }

函数的接口是FP形式的，但是为了避免过多的复制操作，内部操作并不是严格的函数式。为了将其改造成多线程代码，可以开辟出一个新线程负责划分后的一半数据的快排：

    template<typename T>
    std::list<T> parallel_qsort(std::list<T> input) {
        if(input.empty())
            return input;
        std::list<T> result;
        result.splice(result.begin(), input, input.begin());
        const T& pivot = *result.begin();
        auto divide_point = std::partition(input.begin(), input.end(), [&pivot](const T& t) { return t < pivot; });

        std::list<T> lower_group;
        lower_group.splice(lower_group.end(), input, input.begin(), divide_point);

        bool flag = false;
        std::future<std::list<T>> lower_future;
        std::list<T> lower;
        if(lower_group.size() > 1000) {
            lower_future = std::async(std::launch::async,
                                    &parallel_qsort<T>,
                                    std::move(lower_group));
            flag = true;
        }
        else
            lower = parallel_qsort(std::move(input));


        auto higher(parallel_qsort(std::move(input)));

        result.splice(result.end(), higher);
        if(flag)
            result.splice(result.begin(), lower_future.get());
        else
            result.splice(result.begin(), lower);
        return result;
    }

这并不是并发快排的最佳实现，比如*std::partition*依旧是一个单线程操作。这样每一次递归调用快排就会开辟一个新的线程负责一半数据的快排，另一半在当前线程进行快排。但显然通过递归，线程数量会快速上升，导致*massive oversubcription*。（注意*std::async*的默认模式是取决于实现的）。另一方面使用封装的*spawn_task*会比*std::async*更好：

    template<typename F, typename A>
    std::future<std::result_of_t<F(A&&)>>
    spawn_task(F&& f, A&& a) {
        using result_type = std::result_of_t<F(A&&)>;
        std::packaged_task<result_type(A&&)> task(std::move(f));
        std::future<result_type> res(task.get_future());
        std::thread t(std::move(task), std::move(a));
        t.detach();
        return res;
    }

*spawn_task*使用*std::packaged_task*和*std::thread*封装了一个功能和*std::async*类似的类，虽然不能自动防止*massive oveersubscription*，但是更加适合扩展（比如加入线程池）。

## Synchronizing operations with message passing

CSP（*Communicating Sequential Processes*），即线程之间没有任何*shared data*，线程之间只有通过信道交换信息。






