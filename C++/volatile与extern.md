## volatile

**易变性**：在汇编层⾯反映出来，就是两条语句，**下⼀条语句不会直接使⽤上⼀条语句对应的 volatile 变量的寄存器内容**，⽽是量新从内存中读取。 

**不可优化性**：volatile 告诉编译器，**不要对我这个变量进⾏各种激进的优化**，甚⾄将变量直接消除，保证程序员写 在代码中的指令，⼀定会被执⾏。 

**顺序性**：**能够保证 volatile 变量之间的顺序性**，编译器不会进⾏乱序优化。

**Volatile 变量可用于提供线程安全**，但是只能应用于非常有限的一组用例：多个变量之间或者某个变量的当前值与修改后值之间没有约束。

#### volatile的使用场景

##### 模式 #1：状态标志

也许实现 volatile 变量的规范使用仅仅是使用一个布尔状态标志，用于指示发生了一个重要的一次性事件，例如完成初始化或请求停机。

##### 模式 #2：一次性安全发布（one-time safe publication）

在缺乏同步的情况下，可能会遇到某个对象引用的更新值（由另一个线程写入）和该对象状态的旧值同时存在。

这就是造成著名的**双重检查锁定**（double-checked-locking）问题的根源，其中对象引用在没有同步的情况下进行读操作，产生的问题是您可能会看到一个更新的引用，但是仍然会通过该引用看到不完全构造的对象。

~~~C++
class SingletonExample {
    private:
    	static SingletonExample singletonExample;
    public:
    	SingletonExample getInstance(){
        if (singletonExample==null){
            synchronized (SingletonExample.class){
                if (singletonExample==null){
                    singletonExample=new SingletonExample();
                }
            }
        }
        return singletonExample;
    }
}

重排序造成的隐患

上述写法看似解决了问题，但是有个很大的隐患。实例化对象的那行代码（标记为error的那行），实际上可以分解成以下三个步骤：
	1.分配内存空间
	2.初始化对象
	3.将对象指向刚分配的内存空间

但是有些编译器为了性能的原因，可能会将第二步和第三步进行重排序，顺序就成了：
	1.分配内存空间
	2.将对象指向刚分配的内存空间
	3.初始化对象
// ***可能导致在还未完成初始化的对象被访问。*** //
// 正确写法
class SingletonExample {
    private:
    	volatile static SingletonExample singletonExample;
    public:
    	SingletonExample getInstance(){
        if (singletonExample==null){
            synchronized (SingletonExample.class){
                if (singletonExample==null){
                    singletonExample=new SingletonExample();
                }
            }
        }
        return singletonExample;
    }
}
~~~

为了解决上述问题，需要在uniqueSingleton前加入关键字[volatile](https://so.csdn.net/so/search?q=volatile&spm=1001.2101.3001.7020)。使用了volatile关键字后，重排序被禁止，所有的写（write）操作都将发生在读（read）操作之前。

##### 模式 #3：独立观察（independent observation）

安全使用 volatile 的另一种简单模式是：定期 “发布” 观察结果供程序内部使用。【例如】假设有一种环境传感器能够感觉环境温度。一个后台线程可能会每隔几秒读取一次该传感器，并更新包含当前文档的 volatile 变量。然后，其他线程可以读取这个变量，从而随时能够看到最新的温度值。

##### 模式 #4：“volatile bean” 模式

volatile bean 模式的基本原理是：很多框架为易变数据的持有者（例如 `HttpSession`）提供了容器，但是放入这些容器中的对象必须是线程安全的。

##### 模式 #5：开销较低的“读－写锁”策略

如果读操作远远超过写操作，您可以结合使用**内部锁**和 **volatile 变量**来减少公共代码路径的开销。

如下显示的线程安全的计数器，使用 `synchronized` 确保增量操作是原子的，并使用 `volatile` 保证当前结果的可见性。

## external

在 C 语⾔中，修饰符 extern ⽤在变量或者函数的声明前，⽤来说明 “此变量/函数是在别处定义的，要在此处引 ⽤”。 

注意 extern 声明的位置对其作⽤域也有关系，如果是在 main 函数中进⾏声明的，则只能在 main 函数中调⽤，在其它函数中不能调⽤。其实要调⽤其它⽂件中的函数和变量，只需把该⽂件⽤ #include 包含进来即可，为啥要⽤ extern？因为⽤ extern 会加速程序的编译过程，这样能节省时间。 

在 C++ 中 extern 还有另外⼀种作⽤，⽤于指示 C 或者 C＋＋函数的调⽤规范。

⽐如在 C＋＋ 中调⽤ C 库函数，**就需要在 C＋＋ 程序中⽤ extern “C” 声明要引⽤的函数**。这是给链接器⽤的，**告诉链接器在链接的时候⽤C 函数规范 来链接**。主要原因是 C＋＋ 和 C 程序编译完成后在⽬标代码中命名规则不同，⽤此来解决名字匹配的问题。

~~~C++
// 在c++中调用C的具体情况如下：

// CAdd.h
int cadd(int x, int y);

// CAdd.c
#include "CAdd.h"
#include <stdio.h>

int cadd(int x, int y) {
    printf("from C function.\n");
    return (x + y);
}

// cppmain.cpp
#include <stdio.h>

// 表示用c来编译
extern "C" {
#include "CAdd.h"
}

int main()
{
  int sum = cadd(1, 2);
  printf("1+2 = %d\n", sum);
  return 0;
}
~~~

**C语言没法直接调用C++的函数**，但可以**使用包裹函数来实现**。C++文件`.cpp`中可以调用C和C++的函数，但是C代码`.c`只能调用C的函数，所以可以**用包裹函数去包裹C++函数**，然后把这个**包裹函数以C的规则进行编译**，这样C就可以调用这个包裹函数了。

~~~C++
// CppAdd.h
int cppadd(int x, int y)；

// CppAdd.cpp
#include "CppAdd.h"
#include <stdio.h>

int cppadd(int x, int y) {
    printf("from C++ function.\n");
    return (x + y);
}


// CppAddWrapper.h
#pragma  once

#ifdef __cplusplus
extern "C"{
#endif

int cppaddwrapper(int x, int y)

#ifdef __cplusplus
}
#endif

// CppAddWrapper.cpp
#include "CppAddWrapper.h"
#include <stdio.h>
#include <CppAdd.h>

int cppaddwrapper(int x, int y){
	printf("from wrapper.\n");
	int sum = cppadd(x, y);
	return sum;
}

// main.c
#include "CppAddWrapper.h"
#include <stdio.h>

int main(){
	int sum = cppaddwraper(1, 2);
	printf("1 + 2 = %d\n", sum);
	return 0;
}
~~~





