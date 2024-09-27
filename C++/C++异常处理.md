## 异常的概念

​	异常是程序在运行过程中出现非正常情况的处理机制。当出现异常时程序会停止运行并调用异常处理程序。

## 异常的分类

#### 内置异常

~~~C++
std::exception:  // 所有标准异常类的积累。
std::logic_error:  // 表示程序逻辑错误。
	invalid_argument // 表示无效参数。
	length_error // 指出某些行为“可能超过了最大极限”。比如传入的字符串长度太长。
	out_of_range // 指出参数值“不在预期范围内”。比如vector超过索引范围。
	domain_error // 指出专业领域范围内的错误。
	future_error // 用来指出当使用非同步系统调用时发生的逻辑差错。注意，此范围内的运行期差错是一般由class system_error发出。
std::runtime_error:  // 表示运行时错误。
	range_error // 内部计算时发生区间错误。
	overflow_error // 算术运算时发生上溢位。
	underflow_error // 算术运算时发生下溢位。
	system_error // 用来指出因底层操作系统而发生的错误。C++标准库可能在并发环境中抛出这个异常。
	// new操作失败，bad_alloc异常，除非用的是new的nothrow版本针对标准程序库的IO部分，提供一个特殊的异常类ios_base::failure。当数据流由于错误或由于到达文件尾端而发生状态改变时，就可能抛出这个异常。
~~~

#### 自定义异常(继承自内置异常)

​	除了使用内置异常，我们还可以创建**自定义异常类**来处理特定错误。自定义异常类可以**继承自内置异常类**以实现异常的精细控制。

## 异常的处理方式

~~~C++
// try-catch语句
try {
  // 可能会抛出异常的代码块
} catch (exception1_type exception1_var) {
  // 处理异常类型1
} catch (exception2_type exception2_var) {
  // 处理异常类型2
}

// throw 语句
throw exception_object;

// noexcept 修饰符 
// 使用 noexcept 可以优化程序性能。当程序遇到一个没有 noexcept 修饰符的函数，会假设这个函数可能抛出异常，导致额外的代码执行。
void function_name() noexcept {
  // 不会抛出异常的函数
}

// finally 语句块
try{
  // 可能抛出异常的代码块
} catch(...) {
  // 处理异常
} finally {
  // 无论有无异常都执行
}
~~~

## 异常处理流程

​	1.程序执行到可能抛出异常的代码时，这段代码必须嵌入到try块中。
​	2.如果在try块中的代码引发了异常，程序会跳转到与抛出异常类型匹配的catch块中，**catch块负责处理异常**。
​	3.最后程序会从catch块中退出，向下执行任何后续的代码。

## 标准异常类

~~~C++
std::exception // 所有标准异常类的基类。
std::bad_alloc // 在申请内存失败时抛出。
std::bad_cast // 将一个指向派生类的基类指针强制转换为派生类指针时，如果类不是目标类型，则抛出。
std::ios_base::failure // I/O 操作失败时抛出。
std::out_of_range // 数组访问越界时抛出。
std::invalid_argument // 提供无效参数时抛出。
std::length_error //  尝试创建一个超出可分配内存大小的vector、string等时发生。
std::logic_error // 人为 bug，如出现未在程序中考虑到的条件等，通常还有一些 derived class 总结不同的情况（此类异常可以在编译环境中静态检查出来）。
std::runtime_error // 运行时错误，一般情况下是在单个文件运行时出错。（常常由文件、网络等外部原因引起）
~~~

## 异常的传递

## C++ 11 新增异常功能

#### noexcept操作符

​	noexcept是一个C++11新增的操作符，它用于指明一个函数是否可能抛出异常。当一个函数有可能抛出异常时，我们可以在其声明或定义前使用 noexcept 关键字告知编译器这一信息。这个操作符的作用是帮助编译器进行优化，提高代码的效率。

~~~C++
#include <iostream>
using namespace std;

void func1() noexcept {
    cout << "func1: No exceptions here!" << endl;
}

void func2() {
    throw runtime_error("Exception in func2");
}

int main() {
    cout << boolalpha;
    cout << "func1 is noexcept: " << noexcept(func1()) << endl; // 输出 true
    cout << "func2 is noexcept: " << noexcept(func2()) << endl; // 输出 false
}
~~~

#### 异常列表

在 C++11 中还新增了异常列表，也叫函数 try 块。这个功能允许我们在函数定义中捕获异常并处理，与在函数体中进行异常处理不同，这让我们可以将所有异常处理代码放在同一个位置，提高代码的可读性和可维护性。

在函数体开始之前，在函数参数列表后立即使用 try 关键字并紧跟着花括号，在花括号中包含函数体，需要同时声明和捕获异常。需要注意的是，如果一个函数既有函数 try 块，又有普通的 try-catch 块，它们的处理顺序是不一样的，**函数 try 块中的异常处理将先于普通的 try-catch 块执行**。

try函数块的应用场景一般局限于下面几个：

（1）构造函数初始化列表；
		程序执行的任何时刻都可能发生异常，如果构造函数在执行初始值列表是发生异常，而此构造函数体内的try语句块还未生效，所以无法处理构造函数初始值列表抛出的异常。

（2）基类构造含数据；

（3）析构函数；

~~~C++
#include <iostream>
#include <stdexcept>
using namespace std;

class MyClass {
public:
    MyClass() : num(42) {
        cout << "Constructing MyClass" << endl;
    }

    ~MyClass() noexcept(false) {
        cout << "Destructing MyClass" << endl;
        throw runtime_error("Exception in destructor");
    }

    void DoSomething() noexcept(false) {
        cout << "Doing something" << endl;
        throw runtime_error("Exception in DoSomething");
    }

private:
    int num;
};

void f(MyClass& mc) try {
    mc.DoSomething();
} catch (const exception& e) {
    cerr << "Caught exception in f(): " << e.what() << endl;
}

int main() {
    MyClass mc;
    f(mc);
    return 0;
}
~~~

## 异常抛出

#### 提前结束函数

提前结束函数会带来很多的后续处理代码，例如下面2方面：

​	1、对于业务：正常的业务流程终端，如何保证业务的事务性。
​	2、对于资源：比如打开的文件描述fd需要关闭，malloc申请的内存需要释放等工作。

~~~c++
#include <iostream>
using namespace std;

void test_func(){

    cout << "test_func throw exception begin" << endl;
    __throw_invalid_argument("test invalid argument");
    cout << "test_func throw exception end" << endl; //此打印不会执行
}
int main()
{
    try
    {
        test_func();
    }
    catch(const std::exception& e)
    {
        std::cerr << e.what() << '\n';
    }
    return 0;
}

~~~

#### 栈展开

​	异常发生后，throw抛出异常表达式类型需要找到能够处理该类型异常的catch block进行处理，如果当前函数没有找到相应的catch分支块，着沿着调用链向上匹配，直到找到匹配的catch block。

~~~~C++
void test_func(){
    try{
        cout << "call test_func1" << endl;
        test_func1();
        cout << "====call test_func1 end====" << endl;
    }
    catch(const std::invalid_argument& e){
        std::cerr << e.what() << '\n';
    }

}

void test_func1(){
    cout << "call test_func2" << endl;
    test_func2();
    cout << "===call test_func2 end===" << endl;
}
void test_func2(){
    try{
        cout << "call test_func3" << endl;
        test_func3();
        cout << "===call test_func3 end===" << endl;
    }catch(const std::overflow_error & e){
        std::cerr << e.what() << '\n';
    }
}

void test_func3(){
    try{
        cout << "throw invalid_argument exception" << endl;
        __throw_invalid_argument("test invalid argument");
        cout << "throw invalid_argument exception end" << endl;
    }catch(const std::bad_alloc& e){
        std::cerr << e.what() << '\n';
    }
}

int main()
{
    cout << "call test_func" << endl;
    test_func();
    cout << "=call test_func end=" << endl;
    return 0;
}
~~~~

#### 未捕获异常

​	上面示例中异常在最后都找到了捕获的catch分支。如果异常没有匹配到catch分支会怎么样？**程序将使用terminate结束程序**

~~~C++
#include <iostream>
using namespace std;
void test_func1();
void test_func2();
void test_func3();

void test_func(){

        cout << "call test_func1" << endl;
        test_func1();
        cout << "====call test_func1 end====" << endl;
}

void test_func1(){
    cout << "call test_func2" << endl;
    test_func2();
    cout << "===call test_func2 end===" << endl;
}
void test_func2(){
    try{
        cout << "call test_func3" << endl;
        test_func3();
        cout << "===call test_func3 end===" << endl;
    }catch(const std::overflow_error & e){
        std::cerr << e.what() << '\n';
    }
}

void test_func3(){
    try{
        cout << "throw invalid_argument exception" << endl;
        __throw_invalid_argument("test invalid argument");
        cout << "throw invalid_argument exception end" << endl;
    }catch(const std::bad_alloc& e){
        std::cerr << e.what() << '\n';
    }
}
class A {
public:
    A(){};
    ~A(){}

};

int main()
{
    cout << "call test_func" << endl;
    test_func();
    cout << "=call test_func end=" << endl;
    return 0;
}


~~~

## 析构函数抛出异常的问题

析构函数从语法上是可以抛出异常的，但是这样做很危险，请尽量不要这要做。原因在《More Effective C++》中提到两个：
（1）如果析构函数抛出异常，则**异常点之后的程序不会执行**，如果析构函数在**异常点之后执行了某些必要的动作比如释放某些资源，则这些动作不会执行，会造成诸如资源泄漏的问题**。
（2）通常异常发生时，c++的异常处理机制在**异常的传播过程中会进行栈展开**（stack-unwinding），因发生异常而逐步退出复合语句和函数定义的过程，被称为栈展开。在**栈展开的过程中就会调用已经在栈构造好的对象的析构函数来释放资源**，**此时若其他析构函数本身也抛出异常，则前一个异常尚未处理，又有新的异常，会造成程序崩溃**。

#### 解决办法

​	如果close抛出异常就结束程序，通常调用abort完成：

~~~c++
DBConn::~DBconn()
{
    try
    {
	    db.close(); 
    }
    catch(...)
    {
        abort();
    }
}
~~~

​	吞下因调用 close 而发生的异常

~~~C++
DBConn::~DBConn
{
    try{ db.close();}
    catch(...) 
    {
        //制作运转记录，记下对close的调用失败！
    }
}
~~~

​	重新设计 DBConn 接口，使其客户有机会对可能出现的异常作出反应

~~~C++
class DBConn
{
public:
    ...
    void close() //供客户使用的新函数
    {
        db.close();
        closed = true;
    }
    ~DBConn()
    {
        if(!closed)
        {
            try        //关闭连接(如果客户不调用DBConn::close)
            {       
                  db.close();
            }
            catch(...) //如果关闭动作失败，记录下来并结束程序或吞下异常
            { 
                制作运转记录，记下对close的调用失败；
                ...
           }
        }
    }
private:
    DBConnection db;
    bool closed;
};
~~~

#### C++异常声明列表

~~~C++
void function() throw (int, double, A, B, C);
~~~

