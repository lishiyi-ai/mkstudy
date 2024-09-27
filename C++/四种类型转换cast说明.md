## static_cast(编译时转换，没有运行时类型检查来保证转换的安全性)

​	支持基本类型转换、类的上行转换(安全)

~~~C++
#include <iostream>
using namespace std;

class Base
{
public:
    Base() {};
    virtual void Show() { cout << "This is Base class"; }
};
class Derived :public Base
{
public:
    Derived() {};
    void Show() { cout << "This is Derived class"; }
};
int main()
{
    Derived* der = new Derived;
    auto  Derived = static_cast<Base*> (der);
    //向上转换一直是安全的
    Derived->Show();
    system("pause");
}
~~~

类的下行转换（不安全）

​	将父类指针、引用转换为子类指针、引用，但**需要程序员自己检查**，因此这种转换方式也不存在额外的开销。

## const_cast(编译时转换，没有运行时类型检查来保证转换的安全性)

​	const_cast主要用于添加或删除const修饰符。它可以用于将const对象转换为非const对象，但这并不意味着你可以修改该对象——**只有当对象本身不是const时**，**这样的转换才是安全的**。
​	**需要注意的是，const_cast通常对指针和引用进行转换，而无法直接移除内置类型的const/volitale属性**，换言之，这种语法直接提供了一个具有写权限的指针或引用，可以通过间接访问的方式，修改常量。

~~~C++
//弥补了**static_cast无法转换const/volitale的不足**，将expression的const/volitale属性移除，仅限于底层const属性. 
const_cast<type>(expression);

const int i = 42;
int *p = const_cast<int*>(&i); // 去除const修饰符
// *p = 43; // 未定义行为！因为i本身是const的，所以不应该被修改。

int j = 50;
const int *cp = &j;
int *jp = const_cast<int*>(cp); // 添加const修饰符是安全的，因为j本身不是const的。
*jp = 55; // 合法且安全，因为j不是const的。
	
//
const int a = 10;	
~~~

#### 关于const的内存分配

~~~C++
// 当const的常量使用右值进行赋值时，不会被分配内存，而是直接保存在符号表中，只有在使用时候才会初始化内存。
const int a = 5;
const int*pointer_const = &a; // 在这里才会分配出内存。

// 在编译时分配内存。
int tmp = m;
const int a = tmp; 
~~~

## dynamic_cast(编译时转换，没有运行时类型检查来保证转换的安全性)

​	该转换是**运行时转换**，**其余都是编译时转换**。主要用于**安全的向下进行转换**。同时**当指针是智能指针时**，使用**dynamic_cast向下转换不能成功**，**需使用dynamic_point_cast来进行转换**。
​	对指针进行转换，**失败返回null**，成功返回type类型的对象指针，对于引用的转换，失败抛出一个 **bad_cast**，**成功返回type类型的引用**；
​	dynamic_cast转换中分两种情况：
​		1.当基类指针指向派生对象时能够安全转换。
​		2.基类指针指向基类时会做检查，转换失败，返回结果0。

~~~C++
#include <iostream>
using namespace std;

class Base
{
public:
    Base() {};
    virtual void Show() { cout << "This is Base class" << endl; }
};
class Derived :public Base
{
public:
    Derived() {};
    void Show() { cout << "This is Derived class" << endl; }
};
int main()
{
    //第一种情况
    Base* base = new Derived;
    Derived* der = dynamic_cast<Derived*>(base);
    //基类指针指向派生类对象时能够安全转换
    der->Show();
    //第二种情况
    Base *base1 = new Base;

    if (Derived* der1 = dynamic_cast<Derived*> (base1))
    {
        der1->Show();
    }
    else
    {
        cout << "error!";
    }
    delete(base);
    delete(base1);
    system("pause");
}

// 引用
int main()
{
    // 基类引用子类
    Derived b;
    Base& base1 = b;
    Derived& der1 = dynamic_cast<Derived&>(base1);
    der1.Show();

    // 基类引用基类
    Base a;
    Base& base2 = a;
    // 引用则和指针不同，指针在C++11中存在空指针，而引用不存在空引用，会引发bad_cast异常。
    try
    {
        Derived& der2 = dynamic_cast<Derived&>(base2);
    }
    catch(bad_cast)
    {
        cout << "bad_cast error!!" << endl;
    }
    system("pause");
}

~~~

## reinterpret_cast(编译时转换，没有运行时类型检查来保证转换的安全性)

​	操作结果是一个指针到其他指针的**二进制拷贝**，**没有类型检查**。reinterpret_cast提供了**最低级别的类型转换**，它可以将**任何类型的指针转换为任何其他类型的指针**，也可以将**任何整数类型转换为任何类型的指针**，**以及反向转换**。然而，这种转换通常是不安全的，需要开发者非常小心。

~~~C++
int i = 42;
int *p = &i;
char *cp = reinterpret_cast<char*>(p); // 将int*转换为char*

int address = 0x1234; // 假设这是一个有效的地址
int *ptr = reinterpret_cast<int*>(address); // 将整数转换为指针类型
~~~

## C++智能指针转换

C++20 智能指针转换加入了右值引用。

1、**std::static_pointer_cast()**：
	当指针是智能指针时候，向上转换，用static_cast 则转换不了，此时需要使用static_pointer_cast。

2、**std::dynamic_pointer_cast()**：
	当指针是智能指针时候，向下转换，用dynamic_cast 则转换不了，此时需要使用dynamic_pointer_cast。

3、**std::const_pointer_cast()**：
	功能与std::const_cast()类似

4、**std::reinterpret_pointer_cast()**：
	功能与std::reinterpret_cast()类似

~~~C++
#include <iostream> // std::cout std::endl
#include <memory> // std::shared_ptr std::dynamic_pointer_cast std::static_pointer_cast
 
class base
{
public:
    virtual ~base(void) = default;
};
 
class derived : public base
{
};
 
class test : public base
{
};
 
int main(void)
{
    std::cout << std::boolalpha;
 
    // 两个不同的派生类对象
    auto derivedobj = std::make_shared<derived>();
    auto testobj = std::make_shared<test>();
 
    // 隐式转换 derived->base
    std::shared_ptr<base> pointer1 = derivedobj;
 
    // static_pointer_cast derived->base
    auto pointer2 = std::static_pointer_cast<base>(derivedobj);
 
    // dynamic_pointer_cast base->derived
    auto pointer3 = std::dynamic_pointer_cast<derived>(pointer1);
    std::cout << (pointer3 == nullptr) << std::endl;
 
    // dynamic_pointer_cast base->derived
    auto pointer4 = std::dynamic_pointer_cast<test>(pointer1);
    std::cout << (pointer4 == nullptr) << std::endl;
 
    return 0;
}
~~~

~~~C++
#include <memory>
#include <cassert>
#include <cstdint>
 
int main()
{
    std::shared_ptr<int> foo;
    std::shared_ptr<const int> bar;
    
    foo = std::make_shared<int>(10);
    
    bar = std::const_pointer_cast<const int>(foo);
    
    std::cout << "*bar: " << *bar << std::endl;
    *foo = 20;
    std::cout << "*bar: " << *bar << std::endl;
    
    std::shared_ptr<std::int8_t[]> p(new std::int8_t[4]{1, 1, 1, 1});
    std::shared_ptr<std::int32_t[]> q = std::reinterpret_pointer_cast<std::int32_t[]>(p);
    
    std::int32_t r = q[0];
    
    std::int32_t x = (1 << 8) | (1 << 16) | (1 << 24) | 1;
    assert(r == x);
    
    return 0;
}
~~~

