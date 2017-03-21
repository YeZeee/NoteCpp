# Item14: Declare Functions *noexcept* if They Won't Emit Exceptions

C++98中的exception specifications带来糟糕的维护成本，在C++11中给出了更加简单直接的异常标识，就是直接指出代码是否会抛出异常，由此带来了关键字*noexcept*。代码是否会抛出异常关乎到用户代码的调用效率和异常安全。

    int foo() throw();          // C++98 style. less optimizable.
    int foo() noexcept;         // C++11 style. most optimizable.

同时一个函数是否会抛出异常对于编译器优化也十分关键。比如foo抛出了异常，违反了无异常声明。noexcpt可能会在程序终止前进行栈展开，而throw版本必须进行栈展开。这就给予编译器更灵活的空间去优化。

## In Standard Library

*noexcept*在C++标准库的优化也十分重要。

比如vector的push_back函数在需要reserve的情况下，需要将旧空间中的元素转移至新空间中。

在C++98中，一致使用copy。这样的好处是异常安全，因为即使抛出异常，原空间中的所有元素在完成copy之前都不会有改变；同时这样带来的开销也是巨大的。

在C++11中，将使用"move if you can, but copy if you must"的策略：move的方式可以带来更好的效率，但是move是异常不安全的，因为在移动过程中抛出异常而原本空间中的元素已经被改变，如果反向还原也有可能会抛出新的异常。

所以在C++11中，如果元素的move操作被标识不抛出异常，那这类函数(eg. deque::insert)就将采用move而非copy的方式转移元素。这样可以提高许多效率。

另一个例子是swap函数，swap函数的异常性质是由用户的swap函数的异常决定的。比如对于数组的swap：

    template<class T, size_t N>
    void swap(T (&a)[N], T (&b)[N])
        noexcept(noexcept(swap(*a, *b)));

这里使用到了*conditionally noexcept* 

## Use *noexcept* With Caution

许多函数其实是异常中立（*exception neutral*）的，即函数自己并不会抛出异常，但是其调用的函数会。最好不要为了使用noexcept去对外刻意隐藏这些异常，比如catch了所有异常、或者转换为错误码之类的方式，无疑会增加代码的复杂度，以及维护成本，换取的性能可能反而被这些处理所淹没。

对于一部分函数*noexcept*是十分重要的，所以这类函数默认为*noexcept*:

- memory deallocation function(i.e., operator delete\operator deletep[])
- destructor

在C++11中，这已经上升到了语言规则的层次，所有memory deallocation function和destructor，不论是编译自动生成还是用户定义的，都应该默认为*noexcept*（并不是必须，只是非常非常应该）。

只有一种情况，destructor非默认*noexcept*，即当有数据成员（包含基类）的destructor会抛出异常（e.g. "noexcept(false)"）。这种类型在使用标准库算法与容器时抛出异常都将是未定义行为。

## *wide contracts* and *narrow contracts*

有些库会将函数区分为*wide contracts*和*narrow contracts*。*wide contracts*函数不对传入参数添加约束，不必照顾程序的状态。这类函数永远不会出现未定义行为，比如vector::size，我们申请了一块内存并将它强制转换为vector，但这个情况下size()的输出是合理的，符合定义的，但是该程序的行为确实没有定义保障。

而*narrow contracts*函数会对传入参数增加限制，并且需要照顾到程序状态。如果传入参数违反了限制，那么程序的结果是未定义的：

    void f(cosnt std::string& s) noexcept;      // Prediction: s.size()>=32.

如果传入string的长度小于32，那么该函数将会是未定义的。但是保证该前提是用户代码的义务，而不是f()的义务，所以f()不会进行参数检查，也不会抛出任何异常，所以声明*noexcept*的理由是充分的。

但是如果f()的实现进行了参数检查（防御式编程），因为处理异常往往比处理未定义行为要简单的多。所以f()将抛出异常，但是由于*noexcept*将会导致程序直接终止。

所以往往这种库设计时只会为*wide contracts*函数保留*noexcept*。

## Compiler Offer no Help About Inconsistencies Between Implementations and Exception Specifications

    void setup();            // Functions defined elsewhere.
    void cleanup();

    void doWork() noexcept {
        setup();
        ...
        cleanup();
    }

doWork的异常声明和实现是矛盾的，因为setup和cleanup均没有声明*noexcept*。但是setup和cleanup却有可能在文档中说明了不会抛出任何异常，可能由于其他原因（比如库设计时间过去久远...）而没有声明*noexcept*，。所以doWork声明*noexcept*是完全合理的，编译器可能不会提出警告。

## Things to Remember

- *noexcept*是函数接口的一部分，和*const*...一样，函数调用时可能会依赖于它。
- 编译器对*noexcept*函数的优化更强。
- *noexcept*对于swap和move操作等十分重要，对memory deallocation函数和destructor具有特殊的机制。
- 绝大多数函数都是异常中立的。
- 编译器对于函数的实现和异常声明上的矛盾有可能不会进行检查与警报。
