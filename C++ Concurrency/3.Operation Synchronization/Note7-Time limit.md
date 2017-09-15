# Time limit

## std::chrono

*std::chrono*是C++11提供的时间库，其中有三个重要概念：

- *clock*：表示时钟。 
- *duration*：表示时间间隔。
- *time_point*：表示时间点。

## clocks

一个时钟至少要表达下列4个信息：

- *now*，现在的时间点。
- *type of value*，表达时间的类型。
- *tick period*，计数表达的间隔。
- *is_steady*，是否可以看作稳定时钟。

C++11标准库提供了3种时钟：

- *std::chrono::system_clock*
- *std::chrono::steady_clock*
- *std::chrono::high_resolution_clock*

### std::chrono::system_clock

- 方法*now*会返回当前操作系统时钟的时间点。
- 时钟的起始时刻不定，一般采用*Unix Time*。
- 是唯一可以与C标准库*std::time_t*进行转换的C++标准库时钟。
- 一般不稳定，因为系统时钟是可以随时更改的。

一般用于获取系统时钟，日期，不适合做测量计时。

### std::chrono::steady_clock

- 方法*now*会返回时钟的时间点。
- 时钟的起始时刻不定，和系统时钟无关。
- 总是稳定。

一般用于测量计时。

### std::chrono::high_resolution_clock

- 方法*now*会返回当前操作系统时钟的时间点。
- 提供当前系统最高精度的时钟，有可能会是*std::chrono::system_clock*和*std::chrono::steady_clock*别名。

## duration

*std::chrono::duration*模板用于表示时间段：第一个模板参数为表达时间段的类型，第二个模板参数为模板*std::ratio\<>*用于各种类型的*duration*的转换以及表达*tick period*。

*std::chrono::duration*支持各种时间计算，以及不同时间类型的转换*duration_cast*。

以及常用的特化：

    std::chrono::nanoseconds
    std::chrono::microseconds
    std::chrono::milliseconds
    std::chrono::seconds
    std::chrono::minutes
    std::chrono::hours



