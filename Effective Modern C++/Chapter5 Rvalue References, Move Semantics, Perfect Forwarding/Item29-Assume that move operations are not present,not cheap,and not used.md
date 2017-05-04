# Item29:Assume That Move Operations Are Not Present, Not Cheap, And Not Used

*move semantics*是C++11的一块重要的拼图，它带来了更高效的拷贝操作，更准确的拷贝语义。但是*move semantics*并不像想象中的那么高效、普遍。

- 许多类型不支持移动操作
- 许多类型的移动操作并不如想象中的高效

首先许多类型是不支持*move semantics*的。C++标准库在C++11进行了一次彻底的翻新，许多类型都加入了更加高效的移动操作，但也有许多类型并没有。  
所有C++11的标准模板库中的容器都支持移动操作，但是并不是所有容器的移动操作都是高效的：可能因为这类容器根本无法支持高效的移动操作；也可能因为容器元素无法配合容器实现高效的移动。

比如*std::array*，其实是一个带有STL接口的原生数组。其他STL容器的元素大都是储存在堆上的，而*std::array*是存储在栈上的。所以*std::vector*的移动，本质上只需要改变标记元素位置用的若干个指针就可以，复杂度O(1)。而*std::array*的移动，需要依次调用每一个元素的移动，复杂度O(n):


    std::vector<Widget> vw1;
    ...
    auto vw2 = std::move(vw1);  // move vw1 into vw2. runs in constant time. only ptrs in vw1 and vw2 are modified.

    std::array<Widget, 10000> aw1;
    ...

    auto aw2 = std::move(aw1);  // move aw1 into aw2. runs in linear time. All elements in aw1 are move into aw2.

再看Widget的拷贝操作，如果Widget的移动比拷贝高效，那么上述容器的移动依旧是比拷贝要高效的，所以std::array确实需要支持移动语义。

再看另一个例子*std::string*。*std::string*支持短字符串优化*small string optimization*(SSO)。如果*std::string*中的字符串足够小，那么字符串的存储将不会分配在堆上，而是分配在一块内置buffer上。所以对于小字符串的移动并不比拷贝高效。

即使有些类支持高效的移动操作，最后依旧可能被耗时的拷贝操作所替代。见Item14.有些容器操作要求强异常安全保证，只有当移动操作声明不抛出异常的情况下，才会使用*move*代替*copy*。

以下应用场景，*move semantics*不会有利于程序的效率:

- No move operations：类型不支持move。
- Move not faster：move并不比copy快。
- Move not usable：场景要求move操作不会抛出异常，然而move并没有声明*noexcept*。
- Source object is lvalue：除了个别例外，只有右值可以用作移动的来源。

## Things to Remember

- 总是假定*move operations*不存在、不效率、不可用。
- 在确认可以使用*move semantics*的情况下，无视上一条。

