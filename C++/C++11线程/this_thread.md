## C++11中的std::this\_thread

上面讲了那么多关于创建、控制线程的方法，现在该讲讲关于线程控制自己的方法了。  
在`<thread>`头文件中，不仅有std::thread这个类，而且还有一个std::this\_thread命名空间，它可以很方便地让线程对自己进行控制。

#### std::this\_thread常用函数

> std::this\_thread是个命名空间，所以你可以使用`using namespace std::this_thread;`这样的语句来展开这个命名空间，不过我不建议这么做。

| 函数                                                         | 作用                                               |
| ------------------------------------------------------------ | -------------------------------------------------- |
| std::thread::id get\_id() noexcept                           | 获取当前线程id                                     |
| template<class Rep, class Period>                            |                                                    |
| void sleep\_for( const std::chrono::duration<Rep, Period>& sleep\_duration ) | 等待`sleep_duration`（`sleep_duration`是一段时间） |
| void yield() noexcept                                        | **暂时**放弃线程的执行，将主动权交给其他线程       |
| （放心，主动权还会回来）                                     |                                                    |

#### 例十二：std::this\_thread中常用函数的使用

```cpp
#include <iostream>
#include <thread>
#include <atomic>
using namespace std;
atomic_bool ready = 0;
// uintmax_t ==> unsigned long long
void sleep(uintmax_t ms) {
	this_thread::sleep_for(chrono::milliseconds(ms));
}
void count() {
	while (!ready) this_thread::yield();
	for (int i = 0; i <= 20'0000'0000; i++);
	cout << "Thread " << this_thread::get_id() << " finished!" << endl;
	return;
}
int main() {
	thread th[10];
	for (int i = 0; i < 10; i++)
		th[i] = thread(::count);
	sleep(5000);
	ready = true;
	cout << "Start!" << endl;
	for (int i = 0; i < 10; i++)
		th[i].join();
	return 0;
}
```

我的输出：

```plain
Start!
Thread 8820 finished!Thread 6676 finished!

Thread 13720 finished!
Thread 3148 finished!
Thread 13716 finished!
Thread 16424 finished!
Thread 14228 finished!
Thread 15464 finished!
Thread 3348 finished!
Thread 6804 finished!
```

你的输出几乎不可能和我一样，不仅是多线程并行的问题，而且每个线程的id也可能不同。