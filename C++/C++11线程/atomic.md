### std::atomic

mutex很好地解决了多线程资源争抢的问题，但它也有缺点：太……慢……了……  
以例四为标准，我们定义了100个thread，每个thread要循环10000次，每次循环都要加锁、解锁，这样固然会浪费很多的时间，那么该怎么办呢？接下来就是atomic大展拳脚的时间了。

#### 例五：std::atomic的使用

根据atomic的定义，我又修改了例四的代码：

```cpp
// Compiler: MSVC 19.29.30038.1
// C++ Standard: C++17
#include <iostream>
#include <thread>
// #include <mutex> //这个例子不需要mutex了
#include <atomic>
using namespace std;
atomic_int n = 0;
void count10000() {
	for (int i = 1; i <= 10000; i++) {
		n++;
	}
}
int main() {
	thread th[100];
	for (thread &x : th)
		x = thread(count10000);
	for (thread &x : th)
		x.join();
	cout << n << endl;
	return 0;
}
```

输出结果：1000000，正常

##### 代码解释

其实，`std::atomic_int`只是`std::atomic<int>`的别名罢了。  

> 原子操作是最小的且不可并行化的操作。

这就意味着即使是多线程，也要像同步进行一样**同步操作**atomic对象，从而省去了mutex上锁、解锁的时间消耗。

#### std::atomic常用成员函数

##### 构造函数

对，atomic没有显式定义析构函数

| 函数                             | 类型           | 作用                                                         |
| -------------------------------- | -------------- | ------------------------------------------------------------ |
| atomic() noexcept = default      | 默认构造函数   | 构造一个atomic对象（未初始化，可通过atomic\_init进行初始化） |
| constexpr atomic(T val) noexcept | 初始化构造函数 | 构造一个atomic对象，用`val`的值来初始化                      |
| atomic(const atomic&) = delete   | 复制构造函数   | （已删除）                                                   |

##### 常用成员函数

atomic能够直接当作普通变量使用，成员函数貌似没啥用，所以这里就不列举了，想搞明白的[点这里](http://cplusplus.com/reference/atomic/atomic/) （英语渣慎入，不过程序猿中应该没有英语渣吧）