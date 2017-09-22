# Memory Model

在C++中，内存是一块或者多块连续的*byte*序列，所有*byte*在程序中都有唯一的地址。

## Byte

*Byte*是C++程序中最小的寻址单元，被定义为可以容纳256个不同的值以及*basic execution character set*的连续*bit*。C++支持8*bits*及以上的*byte*，可以使用*CHAR_BIT*或者*std::numeric_limits<unsigned char>::digits*来查询一个*byte*有多少*bit*（*char*、*unsigned char*、*signed char*保证只使用一个*byte*进行储存。）。

## Obeject

*Object*是C++的基本概念，应当和OO中的*object*区分开来。*object*是一块包含以下特性的存储区域：

1. size
2. aligment requirement
3. storage duration
4. lifetime
5. type
6. value
7. name（optional）

类似于引用（*reference*）、位域（*bit field*）都不是*object*。

### Object representation and value representation

对象表达和值表达是两回事，对象表达更加“底层”：
- *object representation*是指以*object*所在地址的一个*byte*序列。
- *object*的*value representation*是*object*所占区域的该类型的值。

许多原因会导致两者的不一致，比如类型标准、内存对齐、位域，比如：

    #include <cassert>
    struct S {
        char c;  // 1 byte value
                // 3 bytes padding
        float f; // 4 bytes value
        bool operator==(const S& arg) const { // value-based equality
            return c == arg.c && f == arg.f;
        }
    };
    assert(sizeof(S) == 8);
    S s1 = {'a', 3.14};
    S s2 = s1;
    reinterpret_cast<char*>(&s1)[2] = 'b'; // change 2nd byte
    assert(s1 == s2); // value did not change

通过更改结构体的第二个字节从而改变了*object represention*，但是由于内存对齐，第二个字节对于类型S来说没有意义，所以*value representation*不变。

同样的例子还有浮点数规范中，多个*bit*序列对应值*NaN*。

### Subobjects

一个*object*内部可以包含若干*subobject*，比如：

- member objects
- base class subobjects
- array elements

如果一个*object*不是其他的*subobject*，则可以称为*complete object*。

*array elements*、*member objects*和*complete object*归类为*most derived object*，非位域的*most derived object*的*size*必然非零。

### Polymorphic objects


### Alignment requirement

内存对齐的由来：因为CPU对内存的随机访问是一个缓慢的过程，为了减少CPU对内存的访问次数出现了内存对齐（通过空间节省时间）。

比如一个32位的CPU字长为32bit、字节为8bit，其地址线就为30位，可以管理2^32字节的内存，其地址线最低两位总是为00。

该CPU希望访问一个地址为2的、大小为4字节的数据（即位于2、3、4、5）。那么CPU就要对内存进行两次访问，第一次取地址0(00)的数据并取2、3的数据，第二次取地址1(00)的数据并取4、5的数据。

但如果数据地址为4（即位于4、5、6、7），那么只需要进行一次对地址1(00)访。

- 因此规定对齐要求为*alignment requirement*的片段必须位于地址偏移*offset*的位置，并满足*offset mod alignment requirement == 0*.

即有：

    struct T{
        char data1;     // size 1
        // padding 3
        int data2;      // size 4
        some_type data3;    // size 2
        // padding 2
        int data4;      // size 4
        char data5;     // size 1
    };

可以得出T的大小为17，但其实T的大小为20。因为*struct*自身可以作为一个数组的成员，而数组必然连续，会破坏后续的*struct*的内存对齐决策，比如：

    T pp[2];

pp[0]的data5落在地址17的位置，而pp[1]的data2会落在21的位置，破坏了内存对齐。

- 因此规定，struct的*size*必须为其成员中的最大对齐要求的整数倍。

即有：

    struct T{
        char data1;     // size 1
        // padding 3
        int data2;      // size 4
        some_type data3;    // size 2
        // padding 2
        int data4;      // size 4
        char data5;     // size 1
        // padding 3
    };

C++可以通过*alignof*获得一个类型的对齐要求，需要注意的是对齐要求和大小是不一样的，虽然对于内建类型一般相同。

比如结构体T的大小为20，但其对齐要求为4，因为其最大成员的对其要求为4。

归结为如下规律：

1. 内建类型 alignment requirement 和 size 一致。
2. 成员的地址相对于结构地址的偏移为该成员alignment requirement的整数倍。
3. 结构的大小应该为成员中最大的alignment requirement的整数倍，且结构的alignment requirement和成员中最大的alignment requirement相同。
4. 注意考虑多态带来的额外成员。

## Memory location

*Memory location*是:

- an object of scalar type (arithmetic type, pointer type, enumeration type, or std::nullptr_t)
- or the largest contiguous sequence of bit fields of non-zero length

但注意有些语言特性会引入有些额外的*memory location*，比如虚函数和引用。

    struct S {
        char a;     // memory location #1
        int b : 5;  // memory location #2
        int c : 11, // memory location #2 (continued)
              : 0,  // start new byte
            d : 8;  // memory location #3
        struct {
            int ee : 8; // memory location #4
        } e;
    } obj; // The object 'obj' consists of 4 separate memory locations

## Threads and data races

任何线程都可以访问程序中的任何*object*（包括线程自己的自动变量以及*threadlocal*变量，因为可以通过指针和引用访问）。

不同的线程可能在没有同步和几口要求的情况下，同时访问（包括读取和改变）不同的*memory location*。

当一个表达式的值去写一个*memory location*，而另一个表达式访问或者更改这个*memory location*，就出现冲突，这样的冲突会在除以下情况之外演化为*data race*：

- 两个表达式操作在同一个线程或者同一个*signal handler*中执行
- 两个冲突表达式为原子操作(*std::atomic*)
- 其中一个操作比另一个操作先发生（*std::memory_order*）

除此之外的冲突表达式都会引起*data race*，*data race*产生*undefined behaviour*。

    std::atomic<int> cnt;
	auto f = [&cnt] { for(int p = 0; p < 100000; p++) cnt++; };
	std::thread t1(f), t2(f), t3(f);
	t1.join();
	t2.join();
	t3.join();  // OK.

    int cnt;
	auto f = [&cnt] { for(int p = 0; p < 100000; p++) cnt++; };
	std::thread t1(f), t2(f), t3(f);
	t1.join();
	t2.join();
	t3.join();  // undefined behaviour.

## Memory order

当一个线程从一个*memory location*读取一个值时，它可能会读到初始值、当前线程写入的值、其他线程写入的值。

*std::memory_order*是用于描述通常非原子的内存的访问是如何围绕原子操作进行排序的。





