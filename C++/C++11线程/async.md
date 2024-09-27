## C++11中的std::async

> 注：std::async定义在`future`头文件中。

#### 为什么大多数情况下使用async而不用thread

thread可以快速、方便地创建线程，但在async面前，就是小巫见大巫了。  
async可以根据情况选择同步执行或创建新线程来异步执行，当然也可以手动选择。对于async的返回值操作也比thread更加方便。

#### std::async参数

不同于thread，async是一个函数，所以没有成员函数。

| 重载版本                                     | 作用 |
| -------------------------------------------- | ---- |
| template <class Fn, class… Args>             |      |
| future<typename result\_of<Fn(Args…)>::type> |      |

    async (Fn&& fn, Args&&… args) | 异步或同步（根据操作系统而定）以args为参数执行fn  

同样地，传递引用参数需要`std::ref`或`std::cref` |
| template <class Fn, class… Args>  
  future<typename result\_of<Fn(Args…)>::type>  
    async (launch policy, Fn&& fn, Args&&… args); | 异步或同步（根据`policy`参数而定（见下文））以args为参数执行fn，引用参数同上 |

##### std::launch强枚举类（enum class）

std::launch有2个枚举值和1个特殊值：

| 标识符                   | 实际值（以Visual Studio 2019为标准） | 作用                                                         |
| ------------------------ | ------------------------------------ | ------------------------------------------------------------ |
| 枚举值：launch::async    | 0x1（1）                             | 异步启动                                                     |
| 枚举值：launch::deferred | 0x2（2）                             | 在调用future::get、future::wait时同步启动（std::future见后文） |
| 特殊值：launch::async    | launch::defereed                     | 0x3（3）                                                     |

#### 例六：std::async的使用

暂且不管它的返回值std::future是啥，先举个例再说。

```cpp
// Compiler: MSVC 19.29.30038.1
// C++ Standard: C++17
#include <iostream>
#include <thread>
#include <future>
using namespace std;
int main() {
	async(launch::async, [](const char *message){
		cout << message << flush;
	}, "Hello, ");
	cout << "World!" << endl;
	return 0;
}
```

你的编译器可能会给出一条警告：

```plain
warning C4834: 放弃具有 "nodiscard" 属性的函数的返回值
1
```

这是因为编译器不想让你丢弃async的返回值std::future，不过在这个例子中不需要它，忽略这个警告就行了。  
你的输出结果：

```plain
Hello, World!
1
```

不过如果你输出的是

```plain
World!
Hello,
12
```

也别慌，正常现象，多线程嘛！反正我执行了好几次也没出现这个结果。