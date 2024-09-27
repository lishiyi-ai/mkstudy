## C++11中的std::future

我们已经知道如何使用async来异步或同步执行任务，但如何获得函数的返回值呢？这时候，async的返回值std::future就派上用场了。

#### 例七：使用std::future获取线程的返回值

在之前的所有例子中，我们创建线程时调用的函数都没有返回值，但如果调用的函数有返回值呢？

```cpp
// Compiler: MSVC 19.29.30038.1
// C++ Standard: C++17
#include <iostream>
// #include <thread> // 这里我们用async创建线程
#include <future> // std::async std::future
using namespace std;

template<class ... Args> decltype(auto) sum(Args&&... args) {
	// C++17折叠表达式
	// "0 +"避免空参数包错误
	return (0 + ... + args);
}

int main() {
	// 注：这里不能只写函数名sum，必须带模板参数
	future<int> val = async(launch::async, sum<int, int, int>, 1, 10, 100);
	// future::get() 阻塞等待线程结束并获得返回值
	cout << val.get() << endl;
	return 0;
}
1234567891011121314151617181920
```

输出：

```plain
111
1
```

##### 代码解释

我们定义了一个函数sum，它可以计算多个数字的和，之后我们又定义了一个对象`val`，它的类型是`std::future<int>`，这里的`int`代表这个函数的返回值是int类型。在创建线程后，我们使用了future::get()来**阻塞**等待线程结束并获取其返回值。至于sum函数中的折叠表达式（fold expression），不是我们这篇文章的重点。

#### std::future常用成员函数

##### 构造&析构函数

| 函数                           | 类型         | 作用                                                         |
| ------------------------------ | ------------ | ------------------------------------------------------------ |
| future() noexcept              | 默认构造函数 | 构造一个空的、无效的future对象，但可以**移动分配**到另一个future对象 |
| future(const future&) = delete | 复制构造函数 | （已删除）                                                   |
| future (future&& x) noexcept   | 移动构造函数 | 构造一个与`x`相同的对象并破坏`x`                             |
| ~future()                      | 析构函数     | 析构对象                                                     |

##### 常用成员函数

| 函数                                                         | 作用                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 一般：T get()当类型为引用：R& future<R&>::get()当类型为void：void future::get() | 阻塞等待线程结束并获取返回值。若类型为void，则与`future::wait()`相同。只能调用一次。 |
| void wait() const                                            | 阻塞等待线程结束                                             |
| template <class Rep, class Period> <br />future_status wait_for(const chrono::duration<Rep,Period>& rel_time) const; | 阻塞等待rel_time（rel_time是一段时间），<br/>若在这段时间内线程结束则返回future_status::ready<br/>若没结束则返回future_status::timeout<br/>若async是以launch::deferred启动的，则不会阻塞并立即返回future_status::deferred<br/> |

#### std::future\_status强枚举类

见上文`future::wait_for`解释

#### 为啥要有void特化的std::future？

std::future的作用并不只有获取返回值，它还可以检测线程是否已结束、阻塞等待，所以对于返回值是void的线程来说，future也同样重要。

##### 例八：void特化std::future

```cpp
// Compiler: MSVC 19.29.30038.1
// C++ Standard: C++17
#include <iostream>
#include <future>
using namespace std;
void count_big_number() {
	// C++14标准中，可以在数字中间加上单
	// 引号 ' 来分隔数字，使其可读性更强
	for (int i = 0; i <= 10'0000'0000; i++);
}
int main() {
	future<void> fut = async(launch::async, count_big_number);
	cout << "Please wait" << flush;
	// 每次等待1秒
	while (fut.wait_for(chrono::seconds(1)) != future_status::ready)
		cout << '.' << flush;
	cout << endl << "Finished!" << endl;
	return 0;
}
```

如果你运行一下这个代码，你也许就能搞懂那些软件的加载画面是怎么实现的。