### std::mutex

`std::mutex`是 C++11 中最基本的互斥量，一个线程将mutex锁住时，其它的线程就不能操作mutex，直到这个线程将mutex解锁。根据这个特性，我们可以修改一下上一个例子中的代码：

#### 例四：std::mutex的使用

```cpp
// Compiler: MSVC 19.29.30038.1
// C++ Standard: C++17
#include <iostream>
#include <thread>
#include <mutex>
using namespace std;
int n = 0;
mutex mtx;
void count10000() {
	for (int i = 1; i <= 10000; i++) {
		mtx.lock();
		n++;
		mtx.unlock();
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

执行了好几次，输出结果都是1000000，说明正确。

#### mutex的常用成员函数

（这里用`mutex`代指`对象`）

| 函数             | 作用                                                         |
| ---------------- | ------------------------------------------------------------ |
| void lock()      | 将mutex上锁。如果mutex已经被其它线程上锁，那么会阻塞，直到解锁；如果mutex已经被同一个线程锁住，那么会产生死锁。 |
| void   unlock()  | 解锁mutex，释放其所有权。如果有线程因为调用lock()不能上锁而被阻塞，则调用此函数会将mutex的主动权随机交给其中一个线程；如果有线程因为调用lock()不能上锁而被阻塞，则调用此函数会将mutex的主动权随机交给其中一个线程；如果mutex不是被此线程上锁，那么会引发未定义的异常。 |
| bool try\_lock() | 尝试将mutex上锁。如果mutex未被上锁，则将其上锁并返回true；如果mutex已被锁则返回false。 |