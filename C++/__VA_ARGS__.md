### 用可变参数宏(variadic macros)传递可变参数表

你可能很熟悉在函数中使用可变参数表，如:

~~~C++
void printf(const char* format, ...);
~~~

一直以来，可变参数表还是只能应用在真正的函数中，不能使用在宏中。

C99编译器标准终于改变了这种局面，它允许你可以定义可变参数宏(variadic macros)，这样你就可以使用拥有可以变化的参数表的宏。可变参数宏就像下面这个样子:

~~~c++
#define debug(...) printf(__VA_ARGS__)
~~~

### 可变参数的宏里的‘##’操作说明

~~~C++
#define debug(format, ...) fprintf (stderr, format, ## __VA_ARGS__)
~~~

这里，如果可变参数被忽略或为空，‘##’操作将使预处理器（preprocessor）去除掉它前面的那个逗号。如果你在宏调用时，确实提供了一些可变参数，GNU CPP也会工作正常，它会把这些可变参数放到逗号的后面。象其它的pasted macro参数一样，这些参数不是宏的扩展。

### 怎样写参数个数可变的宏



~~~C++
// 一种流行的技巧是用一个单独的用括弧括起来的的 “参数” 定义和调用宏, 参数在 宏扩展的时候成为类似 printf() 那样的函数的整个参数列表。
#define DEBUG(args) (printf("DEBUG: "), printf args)
 
if(n != 0) DEBUG(("n is %d\n", n));

// gcc 有一个扩展可以让函数式的宏接受可变个数的参数。 但这不是标准。另一种 可能的解决方案是根据参数个数使用多个宏 (DEBUG1, DEBUG2, 等等), 或者用 逗号玩个这样的花招:

#define DEBUG(args) (printf("DEBUG: "), printf(args))
#define _ ,    DEBUG("i = %d" _ i);
~~~