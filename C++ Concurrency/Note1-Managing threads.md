# Managing Threads

## *std::thread*

C++线程通过类*std::thread*来实现。有以下特点：

- *std::thread*通过获得一个*callable object*（作为线程入口）构造。
- 当*std::thread*构造完成时，新线程开始运作。
- 当*std::thread*被销毁之前，必须手动保证其相关线程是*join*还是*detach*，否则触发*std::terminate*。
- 在决定*join or detach*之前，线程可能已经工作完毕。
- 如果一个线程是*detach*的，即使*std::thread*被销毁，线程也可以继续工作，但必须要保证线程所用的内存数据可用。

## *ensure that the data accessed by the thread is valid*

在整个线程的工作中，线程访问的数据必须是合法的，这一点和初始线程（以*main*为入口）一样。但这一点在多线程中更加复杂。

    struct func{
        int& i;
        func(int& i_):i(i_){}
        void operator()(){
            for(unsigned j=0;j<1000000;++j){
                do_something(i);
            }
        }
    };
    void oops() {
        int some_local_state=0;
        func my_func(some_local_state);
        std::thread my_thread(my_func);
        my_thread.detach();
    }

如上错误代码，*my_func*是一个*callable object*，其引用成员*i*绑定了一个局部变量*some_local_state*。然后*my_thread*使用*my_func*构造了一个新的线程并且*detach*。新线程中的引用*i*依旧指向局部变量*some_local_state*，如果*oops*先于新线程结束，并且销毁了局部变量，则新线程出现未定义行为。
最简单的办法就是避免*callable object*拷贝这些变量而不是和外部共享这些变量。在建立新线程时必须要注意*callable object*是否包含有*shared data*。

## *join or detach*

*join*就是等待线程完全结束，当线程结束时，线程相关的数据都会被销毁，对象*std::thread*和线程不再存在关系，即单个线程只能*join*一次，方法*joinable*会返回*false*。

当*std::thread*被销毁之前，必须对其*joinable*进行检查，不论其是由于正常还是异常遭到销毁：



    struct func;        // callable obj.

    void f(){
        int local_state = 0;
        func my_func(local_state);
        std::thread t(my_func);
        try{
            do_something();
        }
        catch(...){
            t.join();
            throw;
        }
        t.join();
    }

或者使用RAII风格的*guard*，避免了与异常打交道：

    class thread_guard{
    public:
        thread_guard(std::thread& _t):t(_t) {};
        thread_guard(const thread_guard&) = delete;
        thread_guard& opetator=(const thread_guard&) = delete;
        ~thread_guard(){
            if(t.joinable()){
                t.join();
            }
        }
    private:
        std::thread& t;
    };

    struct func;        // callable obj.

    void f(){
        int local_state = 0;
        func my_func(local_state);
        std::thread t(my_func);
        thread_guard g(t);
        do_something();
    }

*detach*就是使得线程与对象*std::thread*脱离关系，脱离之后将无法再在初线程中通过*std::thread*管理该线程，对象*std::thread*的方法*joinable*将会返回*false*。通过*detach*可以实现后台进程，后台进程依赖*C++ Runtime*脱离初线程自行运作。

## Passing arguments to a thread function

*std::thread*的构造函数可以通过额外可变参数的方式来传递参数给其关联的线程函数：

    void foo(int i, const string& s);

    std::thread t(foo, 1, "123");

但是值得注意的是，不管线程函数的参数列表如何表达，传入的额外参数总是“拷贝”的形式传给线程函数。以上代码看上去相当于调用了：

    foo(1, "123");

但是实际上，通过*std::thread*的构造函数先按“拷贝”的方式将“123”的指针（类型为const char*）传入内部。见如下代码：

    void f(int i,std::string const& s);
    void oops(int some_param){
        char buffer[1024];
        sprintf(buffer, "%i",some_param);
        std::thread t(f,3,buffer);
        t.detach();
    }

*std::thread*对象*t*的构造将参数*buffer*（类型为 char*）拷贝传入线程内部。线程函数*f*将会调用该指针副本构造函数参数*std::string*。但是构造*std::string*的时候指针副本所指向的buffer不一定还存在，初线程可能已经跳出*oops*并且销毁了栈数组*buffer*。线程将出现未定义行为。防止这类未定义行为的解决方案即在*buffer*尚未销毁前，构造*string*：

    void f(int i,std::string const& s);
    void oops(int some_param){
        char buffer[1024];
        sprintf(buffer, "%i",some_param);
        std::thread t(f,3,std::string(buffer));
        t.detach();
    } 

同样存在希望新线程能够改变初线程的状态，这时期望传入引用：

    void update_data_for_widget(widget_id w,widget_data& data);
    void oops_again(widget_id w){
        widget_data data;
        std::thread t(update_data_for_widget,w,data);
        display_status();
        t.join();
        process_widget_data(data);
    }

由于*std::thread*的构造函数复制了*widget_data*对象data，函数*update_data_for_widget*更新的是*widget_data*对象的副本。*std::thread*的应用情景很像*std::bind*，解决方案就是使用*std::ref*：

    std::thread t(update_data_for_widget,w,std::ref(data));

这样以来*std::thread*的构造函数将会将*data*的引用而不是副本传入内部线程。

*std::thread*对于线程函数的处理机制和*std::bind*是一样的，比如将成员函数作为线程函数：

    class X {
    public:
        void func();
    };

    X x;
    thread t(&X::pp,&x);

另一个应用场景是当传入参数只可以*move*，而不可以*copy*的情况，比如*std::unique_ptr*:

    void func(std::unique_ptr<X> pX);

    std::unique_ptr<X> pX = make_unique<X>();
    std::thread(func, std::move(pX));

这样pX所有的对象将会通过*move*的方式传入线程存储，再传给线程函数。值得注意的是，*std::thread*也是一种*move-only*类型，因为其管理一定资源，因此一个线程只能与一个*std::thread*关联。

## Transferring ownership of a thread

*std::thread*对象是*moveable*的，当希望在两个对象之间传递对于线程的管理权限，比如函数内部生成了一个线程，但希望在函数结束后能够继续管理这个线程，就需要将线程的管理权传递给外层的*std::thread*。

转移线程所有权的操作通过*std::thread*的移动构造函数和移动复制操作符实现，但一个*std::thread*实例只能维护一个线程，否则将会终止程序：

    std::thread g(){
        void some_other_function(int);
        std::thread t(some_other_function,42);
        return t;
    }

    void f(std::thread t);

    void some_func();
    f(std::thread(some_func));
    std::thread t(some_func);
    f(std::move(t));

由于*std::thread*可以*move*，*thread_guard*也可以完全持有*std::thread*，而不用引用来管理*thread*：

    class scoped_thread{
    public:
        scoped_thread(std::thread _t) :t(std::move(_t){
            if(!t.joinable())
                throw std::logic_error("no associated thread!");
        }
        scoped_thread(const scoped_thread&) = delete;
        scoped_thread& operator=(const scoped_thread&) = delete;
        ~scoped_thread(){
            if(t.joinable)
                t.join();
        }
    private:
        std::thread t;
    }

可以使用容器管理线程：

    void func(i);

    void f(){
        std::vector<std::thread> threads;
        for(int i = 0; i < 10; ++i) {
            threads.emplace_back(func,i);   // direct construct thread.
            // threads.push_back(std::thread(func,i)); // move temp thread object to vector.
        }
        for(auto&& t : threads) {
            if(t.joinable())
                t.join();
        }
    }

## Choosing the number of threads at runtime

*std::thread::hardware_concurrency()*可以获取当前硬件可以进行的并行处理。


