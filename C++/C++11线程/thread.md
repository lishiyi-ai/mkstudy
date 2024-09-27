## C++11的std::thread

在C中已经有一个叫做pthread的东西来进行多线程编程，但是并不好用 （如果你认为句柄、回调式编程很实用，那请当我没说），所以c++11标准库中出现了一个叫作std::thread的东西。

### std::thread常用成员函数

##### 构造&析构函数

| 函数                                   | 类别           | 作用           |
| -------------------------------------- | -------------- | -------------- |
| thread() noexcept                      | 默认构造函数   | 创建一个线程， |
| 什么也不做                             |                |                |
| template <class Fn, class… Args>       |                |                |
| explicit thread(Fn&& fn, Args&&… args) | 初始化构造函数 | 创建一个线程， |
| 以`args`为参数                         |                |                |
| 执行`fn`函数                           |                |                |
| thread(const thread&) = delete         | 复制构造函数   | （已删除）     |
| thread(thread&& x) noexcept            | 移动构造函数   | 构造一个与`x`  |
| 相同的对象,会破坏`x`对象               |                |                |
| ~thread()                              | 析构函数       | 析构对象       |

##### 常用成员函数

| 函数                            | 作用                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| void join()                     | 等待线程结束并清理资源（会阻塞）                             |
| bool joinable()                 | 返回线程是否可以执行join函数                                 |
| void detach()                   | 将线程与调用其的线程分离，彼此独立执行（此函数必须在线程创建时立即调用，且调用此函数会使其不能被join） |
| std::thread::id get\_id()       | 获取线程id                                                   |
| thread& operator=(thread &&rhs) | 见移动构造函数（如果对象是joinable的，那么会调用`std::terminate()`结果程序） |

##### 例一：thread的基本使用

```cpp
// Compiler: MSVC 19.29.30038.1
// C++ Standard: C++17
#include <iostream>
#include <thread>
using namespace std;
void doit() { cout << "World!" << endl; }
int main() {
	// 这里的线程a使用了 C++11标准新增的lambda函数
	// 有关lambda的语法，请参考我之前的一篇博客
	// https://blog.csdn.net/sjc_0910/article/details/109230162
	thread a([]{
		cout << "Hello, " << flush;
	}), b(doit);
	a.join();
	b.join();
	return 0;
}
```

输出结果：

```plain
Hello, World!
1
```

或者是

```plain
World!
Hello,
12
```

##### 例二：thread执行有参数的函数

```cpp
// Compiler: MSVC 19.29.30038.1
// C++ Standard: C++17
#include <iostream>
#include <thread>
using namespace std;
void countnumber(int id, unsigned int n) {
	for (unsigned int i = 1; i <= n; i++);
	cout << "Thread " << id << " finished!" << endl;
}
int main() {
	thread th[10];
	for (int i = 0; i < 10; i++)
		th[i] = thread(countnumber, i, 100000000);
	for (int i = 0; i < 10; i++)
		th[i].join();
	return 0;
}
```

你的输出**有可能**是这样

```plain
Thread 2 finished!Thread 3 finished!
Thread 7 finished!
Thread 5 finished!

Thread 8 finished!
Thread 4 finished!
Thread 6 finished!
Thread 0 finished!
Thread 1 finished!
Thread 9 finished!
12345678910
```

##### 例三：thread执行带有引用参数的函数

```cpp
// Compiler: MSVC 19.29.30038.1
// C++ Standard: C++17
#include <iostream>
#include <thread>
using namespace std;
template<class T> void changevalue(T &x, T val) {
	x = val;
}
int main() {
	thread th[100];
	int nums[100];
	for (int i = 0; i < 100; i++)
		th[i] = thread(changevalue<int>, nums[i], i+1);
	for (int i = 0; i < 100; i++) {
		th[i].join();
		cout << nums[i] << endl;
	}
	return 0;
}
```

如果你尝试编译这个程序，那你的编译器一定会报错

这是怎么回事呢？原来thread在传递参数时，是以右值传递的：

```auto
template <class Fn, class... Args>
explicit thread(Fn&& fn, Args&&... args)
12
```

划重点：`Args&&... args`  
很明显的右值引用，那么我们该如何传递一个左值呢？`std::ref`和`std::cref`很好地解决了这个问题。  
`std::ref` 可以包装按引用传递的值。  
`std::cref` 可以包装按const引用传递的值。  
针对上面的例子，我们可以使用以下代码来修改：

```cpp
// Compiler: MSVC 19.29.30038.1
// C++ Standard: C++17
#include <iostream>
#include <thread>
using namespace std;
template<class T> void changevalue(T &x, T val) {
	x = val;
}
int main() {
	thread th[100];
	int nums[100];
	for (int i = 0; i < 100; i++)
		th[i] = thread(changevalue<int>, ref(nums[i]), i+1);
	for (int i = 0; i < 100; i++) {
		th[i].join();
		cout << nums[i] << endl;
	}
	return 0;
}
```

这次编译可以成功通过，你的程序输出的结果应该是这样的：

```plain
1
2
3
4
...
99
100
1234567
```

### 注意事项

+   线程是在thread对象被定义的时候开始执行的，而不是在调用join函数时才执行的，调用join函数只是阻塞等待线程结束并回收资源。
+   分离的线程（执行过detach的线程）会在调用它的线程结束或自己结束时释放资源。
+   线程会在函数运行完毕后自动释放，不推荐利用其他方法强制结束线程，可能会因资源未释放而导致内存泄漏。
+   **没有执行`join`或`detach`的线程在程序结束时会引发异常**