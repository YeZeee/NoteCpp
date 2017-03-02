# Item5: Prefer auto to Explicit Type Declartions

auto不仅能够减少拼写，同时也可以防止手写类型带来的性能和正确性问题。
有时，auto带来的类型推断契合于当前算法，但从程序的角度来说，却是错的。
所以引导auto得到想要的类型是十分关键的。

## auto Make Code More Robust

比如：

    int x；  // do not initialize x.

这行代码为初始化x，所以x可能是未定义的，也有可能是值初始化的，具体看环境。

    template<typename It>
    void dwim(It b, It e) {
        for(;b != e; ++b) {
            typename std::iterator_traits<It>::value_type
            currValue = *b;         // Dereferce b and assgin it to currValue;
            ...
        }
    }

这段代码用到了traits，仅仅是声明一个变量便要用到traits等模板编程技巧，易错而且复杂。
再看使用auto的实现版本：

    auto x；  // Wrong！ do not initialize x.

    template<typename It>
    void dwim(It b, It e) {
        for(;b != e; ++b) {
            auto currValue = *b;         // Dereferce b and assgin it to currValue;
            ...
        }
    }

auto类型推断是从initializer开始的，所以auto修饰变量必须初始化，避免了局部变量未初始化带来
的未定义行为。

auto类型推断大大减少了各类typename的声明。

同时auto还能推断出那些只有编译器才能表达的类型，比如lambda，见下：

    auto derefUPLess =                              // Compare *p1 and *p2
        [](const std::unique_ptr<Widget>& p1,
           const std::unique_ptr<Widget>& p2)
           { return *p1 < *p2; };

在C++14中，Lambda的参数也可以用auto表示（注意本质是模板）

    auto derefUPLess =                              // Compare *p1 and *p2
        [](auto& p1, auto& p2)
           { return *p1 < *p2; };

## What's a std::function Object

std::function是C++11泛化化函数指针的产物。函数指针只能指向同型函数，但是
std::function可以代表任何callable对象。

    bool(const std::unique_ptr<Widget>& p1,     // Signature for comparison function.
         const std::unique_ptr<Widget>& p2)
    
    std::function<(const std::unique_ptr<Widget>& p1,
           const std::unique_ptr<Widget>& p2)> func     // Create func.

lambda表达式是一个callable对象，所以也可以通过std::function refer to。

    
    std::function<(const std::unique_ptr<Widget>& p1,
           const std::unique_ptr<Widget>& p2)> 
        derefUPLess = [](const std::unique_ptr<Widget>& p1,
           const std::unique_ptr<Widget>& p2)
           { return *p1 < *p2; };

所以std::function是可以替代auto的。

但是可见，拼写复杂度auto远低于std::funtion。更重要的是，
auto储存lambda表达式的闭包所需的空间与闭包大小相同，
然而std::function相当于实例模板产生了一个function对象，其中一个固定空间的变量储存有闭包，
当这个空间不足以包含这个闭包，function的构造函数在heap上为闭包申请空间。
所以std::function往往比auto占用更多的内存。
同时由于函数调用等等原因，std::function总比auto要慢，还有可能造成内存耗尽的异常。
测试如下：

    clock_t beg, ed;
    beg = clock();
    for (int i = 0; i < 100000; ++i) {
        auto f = [](vector<int> v1, vector<int> v2) {return v1.size() > v2.size(); };
    }
    ed = clock();
    cout << "do not use auto:" << ed - beg << endl;
    beg = clock();
    for (int i = 0; i < 100000; ++i) {
        std::function<bool(vector<int> v1, vector<int> v2)> f =
            [](vector<int> v1, vector<int> v2) {return v1.size() > v2.size(); };
    }
    ed = clock();
    cout << "use auto:" << ed - beg << endl;

>do not use auto:1

>use auto:112

## auto Prevents "type shortcuts"

    std::vector<int> v;
    unsigned sz = v.size();

v.size()的返回值应该是std::vector<int>::size_type, 
在32位系统中，size_type和unsigned长度一致，均为32位。
但在64位系统中，则不，size_type为64位。
所以这行代码不具有移植性。
auto则没有任何问题。

    std::unordered_map<std::string, int> m;

    for(const std::pair<std::string,int>& p : m) {
        ...
    }

这部分代码也有错误，因为std::unordered_map的key部分是const的，
所以遍历类型应该是std::pair<const std::string,int>。
所以上述代码相当于通过拷贝每个pair产生一个std::pair<std::string,int>的临时对象，
然后p指向该对象。测试如下：

    std::unordered_map<std::string, int> m{ make_pair("123",1) };
    clock_t beg, ed;
    beg = clock();
    for (int i = 0; i < 100000; ++i) {
        for (const std::pair<std::string, int>& p : m);     // Copy element.
    }
    ed = clock();
    cout << "do not use auto:" << ed - beg << endl;
    beg = clock();
    for (int i = 0; i < 100000; ++i) {
        for (auto& p : m);
    }
    ed = clock();
    cout << "use auto:" << ed - beg << endl;

>do not use auto:464

>use auto:143

这不仅仅带来的是性能上的提升，更多的是程序的合理性与正确性。
对p取地址：前者带来的是对临时对象，后者是对m中元素。
临时对象将会过程结束后销毁，带来指针操作上的隐患。

显式的拼写类型经常会带来类型转换和类型不匹配，带来性能和可靠性上的损耗。

同时auto也降低了重构成本，比如一个函数的返回值为int，而后改成long，
显式声明需要修改所有位置，而auto自动更新。

很显然auto也并不完美，auto类型推断依赖于initializer，
initializer expression可能并不是我们想要的类型。
同时auto也带来了可读性上的问题。

## Things to Remember

- auto使得变量必须初始化，免疫了类型不匹配带来的转换，进一步防止转换带来的可靠性和性能的问题，
方便于重构代码，减小拼写成本。
- auto的使用也容易陷入一些陷阱，见Item2和Item6。