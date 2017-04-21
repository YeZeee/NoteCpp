# Use *std::weak_ptr* for *std::shared_ptr* Like Pointers That Can Dangle

实现一个行为和*std::shared_ptr*相似，但不参加*shared—ownership*的智能指针在某些场景是很方便的，即这个指针不会影响引用计数。这种指针存在一个*std::shared_ptr*不存在的问题，就是指针悬空。一个真正的智能指针需要跟踪指针是否悬空，*std::weak_ptr*就是为了这个问题而实现的。

## About *std::weak_ptr*

*std::weak_ptr*的API很奇特，甚至不像一个指针：不能解除引用、不能进行指针运算、不能比较，*std::weak_ptr*更向是*std::shared_ptr*的一个增强。

*std::weak_ptr*往往通过*std::shared_ptr*构造：

    auto spw = std::make_shared<Widget>();  // Create a share_ptr pointing to a Widget. ref_count set 1.
    ...

    std::weak_ptr<Widget> wpw(spw);     // Create a weak_ptr pointting to the same Widget. ref_count is 1.

    spw = nullptr;  // ref_count is 0. Widget is destroyed.

    if(wpw.expired()) ...   // wpw is dangled.

我们可能希望检测一个*std::weak_ptr*是否为空，然后解除引用，但是*std::weak_ptr*没有解除引用操作。即使这里存在解除引用操作，但在检查与引用的操作间隔，另一个线程可能恰好销毁了对象，导致访问未定义行为。

因此在这里需要一个原子操作，使得检查和解除引用一气呵成。这可以通过使用*std::weak_ptr*构造一个*std::shared_ptr*实现：

    std::shared_ptr<Widget> spw1 = wpw.lock();  // If wpw dangles, spw1 is nullptr.

    auto spw2 = wpw.lock();

    std::shared_ptr<Widget> spw3(wpw);  // If wpw dangles，throw exception(std::bad_weak_ptr).

使用lock()来实现时，若wpw悬空，则*std::shared_ptr*为空指针；使用*std::shared_ptr*的构造函数实现时，若wpw悬空，则抛出异常*std::bad_weak_ptr*。

## How can *std::weak_ptr* be Useful

见如下函数：

    std::unique_ptr<const Widget> loadWidget(WidgetID id);

如果loadWidget是一个成本很高的call，而且同一个Id可能反复使用的。一个很可靠的优化方式就是缓存其返回值；但对每一个Widget进行阻塞缓存同样影响了性能，所以还可以进一步优化：销毁不再使用的Widget。

这种情形下，使用*std::unique_ptr*作为返回值不再合适，因为调用者希望获得缓存指针的同时，还可以自行决定缓存的生命周期。这个缓存指针需要报告指针是否悬空，因为使用者一旦完成了使用，就会销毁对象，见如下实现：

    std::shared_ptr<const Widget> fastLoadWidget(WidgetID id) {
        static std::unordered_map<WidgetID, std::weak_ptr<const　Wdiget>> cache;
        auto objPtr = cache[id].lock();
        if(!objPtr) {
            objPtr = loadWidget(id);
            cache[id] = objPtr;
        }
        return objPtr;
    }

在该实现中，，cache是一个hash表，其中存储了id对应*std::weak_ptr*:

- 当使用新的id读取时，表中没有id对应的key，构造一个空的*std::weak_ptr*，
lock()返回一个空*std::shared_ptr*初始化objPtr，调用loadWidget()获得Widget对象，并将刚构造的*std::weak_ptr*指向对象，最后返回objPtr；
- 当使用已经存在的id读取时，lock()返回一个指向对应对象的*std::shared_ptr*初始化objPtr并返回，大大提高了读取性能。

值得注意的是，*std::weak_ptr*依赖于*std::shared_ptr*，所以返回值必然为*std::shared_ptr*。当客户端使用完id对应的最后一个*std::shared_ptr*，对象被销毁，再次使用该id时，需要重新调用loadWidget，因此依旧有重构空间。

再看另一个设计样式，观察者样式：发布者(subject)是一个会改变状态的对象，观测者(observer)是一个会接收改变通知并刷新通知的对象。通常发布者中包含指向观测者的指针保证能够在状态改变时，更改通知；但发布者不关心观测者的资源问题，只关心观测者是否还存在，以防止访问一个已经不存在的观测者。*std::weak_ptr*十分契合这样的需求，即发布者可以包含一个元素为指向发布者的*std::weak_ptr*的容器。

最后一个例子：如果存在A,B,C三个对象，A和C共享B(即AC中含有一个*std::shared_ptr*指向B)；B中也要含有一个指针指向A，那这个指针有如下几种选择：

- 裸指针：如果A被析构了，而C存在，B依旧存在，但是该裸指针无从知道A是否已经析构，对该指针的解除引用将导致未定义行为。
- *std::shared_ptr*：在这个设计下，AB分别持有对方，即存在循环指针(A to B to A to B)，导致A和B都无法被析构(引用计数至少为1)，必然导致内存泄漏，
- *std::weak_ptr*：可以防止以上的问题，如果A被析构，B中的指针将会悬空，B可以检测到；A，B也可成功的依次析构，因为*std::weak_ptr*不影响引用计数。

这种应用场景其实并不普遍。比如，一些严格层次的数据结构(树...)。子节点通常只被父节点持有，因此父节点中的指向子节点指针可以使用*std::unique_ptr*，而子节点中指向父节点的指针可以直接使用裸指针(因为父节点的生命周期总是比子节点更加长)。

*std::weak_ptr*和*std::shared_ptr*性能相似，都使用一样的控制块，涉及原子操作。值得注意的是:*std::weak_ptr*不影响*shared ownership*的引用计数，但是影响*weak count*，详见Item21。

## Things to Remember

- 使用*std::weak_ptr*，当指针表现为可悬空的情况下。
- *std::weak_ptr*常用在缓存、观察者列表、防止*shared_ptr*循环导致无法析构的错误。
