## co_await 和 Awaitable

`co_await` 的作用是让协程暂停下来，等待某个操作完成之后再恢复执行。
C++定义了一个协议规范，只要我们的类型按照这个规范实现好，就可以在 `co_await` 使用了。

#### await_ready()

返回类型是 `bool`。协程在执行 `co_await` 的时候，会先调用 `await_ready()` 来询问“操作是否已完成”，如果函数返回了 `true` ，协程就不会暂停，而是继续往下执行。实现这个函数的原因是，异步调用的时序是不确定的，如果在执行 `co_await` 之前就已经启动了异步操作，那么在执行 `co_await` 的时候异步操作有可能已经完成了，在这种情况下就不需要暂停，通过`await_ready()`就可以到达到这个目的。

#### await_suspend()

`await_suspend()`，有一个类型为 `std::coroutine_handle<>` 的参数，返回类型可以是 `void` 或者 `bool` 。如果 `await_ready()` 返回了 `false` ，意味着协程要暂停，那么紧接着会调用这个函数。该函数的目的是用来接收协程句柄（也就是`std::coroutine_handle<>` 参数），并在异步操作完成的时候通过这个句柄让协程恢复执行。协程句柄类似于函数指针，它表示一个协程实例，调用句柄上的对应函数，可以让这个协程恢复执行。

`await_suspend()` 的返回类型一般为 `void`，但也可以是 `bool` ，这时候的返回值用来控制协程是否真的要暂停，这里是第二次可以阻止协程暂停的机会。如果该函数返回了 `false` ，协程就不会暂停（注意返回值的含义跟 `await_ready()` 是相反的）。

#### await_resume()

- `await_resume()`，返回类型可以是 `void` ，也可以是其它类型，它的返回值就是 `co_await` 操作符的返回值。当协程恢复执行，或者不需要暂停的时候，会调用这个函数。

#### 预定义的Awaitable类型

`std::suspend_never` 和 `std::suspend_always` 。顾名思义，这两个类型分别表示“不暂停”和“要暂停”，实际上它们的区别仅在于 `await_ready()` 函数的返回值， `std::suspend_never` 会返回 true，而 `std::suspend_always` 会返回 false。

#### 协程的返回类型和promise_type

`promise_type` 要实现的第一个函数是 `get_return_object()` ，用来创建协程的返回值。在协程内，我们不需要显式地创建返回值，这是由编译器隐式调用 ==`get_return_object()`== 来创建并返回的。

~~~cpp
  class promise_type {
    public:
        Task get_return_object() { return {}; }
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void unhandled_exception() {}
      	void return_value() {}
        void return_void() {}
      	void yield_value() {}
    };
~~~

## co_return

`co_return total` 这个表达式等价于 `promise_type.return_value(total)` ，也就是说，返回的数据会==通过 `return_value()` 函数传递给 `promise_type` 对象==， `promise_type` 要实现这个函数才能接收到数据。除此之外，还要想办法让返回值 `Task` 能访问到这个数据。为了减少数据传递，我们可以在 `promise_type` 和 `Task` 之间共享同一份数据。

## co_yield

`co_yield` 的作用是，返回一个数据，并且让协程暂停，然后等下一次机会恢复执行。 `co_yield value` 这个表达式等价于 `co_await promise_type.yield_value(value)` ， `co_yield` 的参数会传递给 `promise_type` 的 `yield_value()` 函数，再把这个函数的返回值传给 `co_await` 。上文提到，传给 `co_await` 的参数要符合 Awaitable 规范，所以 `yield_value()` 的返回类型也要满足这个规范。在这里就可以使用预定义的 `std::supsend_never` 或 `std::suspend_always` ，通常会使用后者来让协程每次调用 `co_yield` 的时候都暂停。

#### final_supsend()

协程在结束的时候，会调用 `final_suspend()` 来决定是否暂停，如果这个函数返回了要暂停，那么协程不会自动释放，此时协程句柄还是有效的，可以安全访问它内部的数据。
不过，这时候释放协程就变成我们的责任了，我们必须在适当的时机调用协程句柄上的 `destroy()` 函数来手动释放这个协程。

~~~cpp
~Task() {
    coroutine_handle_.destroy();
}
~~~

## 异常处理

~~~~cpp
try {

    co_await promise_type.initial_suspend();

    //协程函数体的代码...
}
catch (...) {

    promise_type.unhandled_exception();
}

co_await promise_type.final_suspend();
~~~~

协程主要的执行代码都被 try - catch 包裹，假如抛出了未处理的异常， `promise_type` 的 `unhandled_exception()` 函数会被调用，我们可以在这个函数里面做对应的异常处理。由于这个函数是在 `catch` 语句中调用的，我们可以在函数内调用 `std::current_exception()` 函数获取异常对象，也可以调用 `throw` 重新抛出异常。

调用了 `unhandled_exception()` 之后，协程就结束了，接下来会继续调用 `final_suspend()` ，与正常结束协程的流程一样。C++规定 `final_suspend()` 必须定义成 `noexcept` ，也就是说它不允许抛出任何异常。
