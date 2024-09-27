## C++11中的std::promise

#### std::promise到底是啥

promise实际上是std::future的一个包装，在讲解future时，我们并没有牵扯到改变future值的问题，但是如果使用thread以引用传递返回值的话，就必须要改变future的值，那么该怎么办呢？  
实际上，future的值不能被改变，但你可以通过promise来创建一个拥有特定值的future。什么？没听懂？好吧，那我就举个例子：

##### 例十：std::future的值不能改变，那么如何利用引用传递返回值

```cpp
constexpr int a = 1;
1
```

现在，把常量当成future，把a当作一个future对象，那我们想拥有一个值为2的future对象该怎么办？  
很简单：

```cpp
constexpr int a = 1;
constexpr int b = 2;
12
```

这样，我们就不用思考如何改动a的值，直接创建一个新常量就能解决问题了。  
promise的原理就是这样，不改变已有future的值，而是创建新的future对象。什么？还没听懂？好吧，记住这句话：

> future的值不能改变，promise的值可以改变。

#### std::promise常用成员函数

##### 构造&析构函数

| 函数                                                         | 类型         | 作用                                                      |
| ------------------------------------------------------------ | ------------ | --------------------------------------------------------- |
| promise()                                                    | 默认构造函数 | 构造一个空的promise对象                                   |
| template <class Alloc> promise(allocator\_arg\_t aa, const Alloc& alloc) | 构造函数     | 与默认构造函数相同，但使用特定的内存分配器`alloc`构造对象 |
| promise (const promise&) = delete                            | 复制构造函数 | （已删除）                                                |
| promise (promise&& x) noexcept                               | 移动构造函数 | 构造一个与`x`相同的对象并破坏`x`                          |
| ~promise()                                                   | 析构函数     | 析构对象                                                  |

##### 常用成员函数

| 函数                                                | 作用                                                         |
| --------------------------------------------------- | ------------------------------------------------------------ |
| 一般：                                              |                                                              |
| void set\_value (const T& val)                      |                                                              |
| void set\_value (T&& val)                           |                                                              |
| 当类型为引用：void promise<R&>::set\_value (R& val) |                                                              |
| 当类型为void：void promise::set\_value (void)       | 设置promise的值并将共享状态设为ready（将future\_status设为ready） |
| void特化：只将共享状态设为ready                     |                                                              |
| future get\_future()                                | 构造一个future对象，其值与promise相同，status也与promise相同 |

#### 例十一：std::promise的使用

以例七中的代码为基础加以修改：

```cpp
// Compiler: MSVC 19.29.30038.1
// C++ Standard: C++17
#include <iostream>
#include <thread>
#include <future> // std::promise std::future
using namespace std;

template<class ... Args> decltype(auto) sum(Args&&... args) {
	return (0 + ... + args);
}

template<class ... Args> void sum_thread(promise<long long> &val, Args&&... args) {
	val.set_value(sum(args...));
}

int main() {
	promise<long long> sum_value;
	thread get_sum(sum_thread<int, int, int>, ref(sum_value), 1, 10, 100);
	cout << sum_value.get_future().get() << endl;
	get_sum.join(); // 感谢评论区 未来想做游戏 的提醒
	return 0;
}
```

输出：

```plain
111
1
```

