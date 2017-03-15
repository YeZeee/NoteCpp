# Prefer *const_iterator*s to *iterator*s

在C++11中完善了对*const_iterator*的支持：
- 对*const_iterator*的获取，即cbegin、cend等成员函数、以及C++14中的非成员函数。

- iterator仅做指示作用的算法，如insert、erase对const_iterator支持。   
> 

    std::vector<int> values; 
    ...
    auto it = std::find(values.cbegin(), values.cend(), 1983);
    values.insert(it, 1998);

这些对*const_iterator*的支持，也提升了泛型编程，以及C++标准化的水平：

    template<typename C, typename V>
    void findAndInsert(C& container,
                       const V& targetV, 
                       const V& insertV) {
        using std::cbegin;
        using std::cend;

        auto it = std::find(cbegin(container),      // Use non-member
            cend(container), targetV);              // version.
        
        container.insert(it,insertV);
    }

使用非成员函数的cbegin..，该模板函数不仅对STL标准容器提供了支持，还对那些符合标准容器的第三方容器以及原生数组提供支持。

再看看cbegin非成员函数的实现方法：

    template<typename C>
    decltype(auto) cbegin(const C& container) {
        return std::begin(container);
    }

令人奇怪的是函数最后返回的是begin而不是使用cbegin成员函数。解释如下：因为container是对传入容器的const引用，begin将会调用container的const版本的.begin()，最后返回的是*const_iterator*。这样既达到了目的，又实现了对没有cbegin成员函数的容器的适配，提高了泛型模板的覆盖面。

## Things to Remember

- 尽量使用*const_iterator*
- 在泛型编程中尽量使用non-member function形式的begin、end、rbegin...
