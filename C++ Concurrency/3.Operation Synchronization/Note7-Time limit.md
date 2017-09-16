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

*std::chrono::duration*模板用于表示时间段：第一个模板参数为表达时间段的类型，第二个模板参数为模板*std::ratio\<>*用于各种类型的*duration*的转换以及表达*tick period*（分数/秒）。

*std::chrono::duration*支持各种时间计算，以及不同时间类型的转换*duration_cast*。

以及常用的特化：

    std::chrono::nanoseconds
    std::chrono::microseconds
    std::chrono::milliseconds
    std::chrono::seconds
    std::chrono::minutes
    std::chrono::hours

比如：

    auto a_hour = hours(1);

	std::cout << "\n" << a_hour.count() << " hour is "
		<< duration_cast<minutes>(a_hour).count() << " minutes\nis "
		<< duration_cast<seconds>(a_hour).count() << " seconds\nis "
		<< duration_cast<milliseconds>(a_hour).count() << " milliseconds\n";

    auto some_seconds = seconds(30);
	std::cout << "\n" << a_hour.count() << " hour minus " << some_seconds.count()
		<< " seconds leaves " << (a_hour - some_seconds).count() << " seconds\n";


## time point

模板*std::chrono::time_point*用于表达时间点：第一个模板参数*Clock*用于表达时钟类型，第二个模板参数*Duration*使用*std::chrono::duration*的某个特化来表达从*epoch*开始的时间段（默认使用*Clock::duration*）。

*time_point*通过某个时间点*epoch*为基准加上某时间段来表达某个时间点，调用*time_since_epoch*来返回这个时间间隔。

*std::chrono::time_point*支持各种时间计算，以及不同时间类型的转换*duration_cast*。

比如：

	using namespace std::chrono;

	time_point<system_clock> t1;
	time_point<system_clock> t2 = system_clock::now();

	// show system time now and epoch
	auto epoch_time = system_clock::to_time_t(t1);
	auto now_time = system_clock::to_time_t(t2);
	std::cout << "now " << std::ctime(&now_time);
	std::cout << "epoch " << std::ctime(&epoch_time);

	// time arithmetic
	time_point<system_clock> t3 = t2 - hours(24);

	// use time_since_epoch.
	std::cout << "hours since epoch: "
		<< duration_cast<std::chrono::hours>(
			t2.time_since_epoch()).count()
		<< '\n';
	std::cout << "yesterday, hours since epoch: "
		<< duration_cast<std::chrono::hours>(
			t3.time_since_epoch()).count()
		<< '\n';

## _until and _for

标准线程库提供了一系列带有*_until*和*_for*后缀的方法。包括*sleep*、*condition_variable*、*future*以及前面没有提到的*timed_mutex*。  

- *_until*表达直到某个时间点*std::chrono::time_point*。
- *_for*表达等待一个时间段*std::chrono::duration*。

这类函数返回会是一个*bool*或者一个状态值，具体见库。
