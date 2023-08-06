# 一些问题

在[base.md](./base.md)中主要介绍了C++20新引入的coroutine相关的概念及使用，而没有深入讲解相应的原理。本部分便是对其的补充。

本部分主要回答如下问题：

- coroutine是如何找到coroutine接口的promise type的？

- coroutine接口中不定义promise type是否可行？

- promise type是如何影响coroutine行为？

- co_await expr的工作原理是什么？

- 什么是Awaitable？ 

- 什么是Awaiter？

- 如何在coroutine 中调用子coroutine？

最后，以一个教复杂的实践结束本部分。

# std::coroutine_traits

C++20提供了std::coroutine_traits用来帮助coroutine找到其coroutine接口中的promise type类。

std::coroutine_traits的声明如下

```c++
template< class R, class... Args >
struct coroutine_traits;
```

其中R为coroutine的coroutine接口类型， Args为coroutine的参数类型。

在标准中，一个可能的coroutine_traits实现如下

```c++
template<class, class...>
struct coroutine_traits {};
 
template<class R, class... Args>
requires requires { typename R::promise_type; }
struct coroutine_traits<R, Args...>
{
    using promise_type = typename R::promise_type;
};
```

由上述实现可知，标准中默认 promise_type必须为R的一个可访问成员才能使coroutine定义合法。否则coroutine定义是非法的。

在初始化coroutine frame时，编译器会根据std::coroutine_trait来确定与coroutine相关的promise_type。

具体而言，假设一个coroutine接口定义为

```c++
struct CoroTask {
   struct promise_type {
       // promise type相关的接口实现
   };
};
```

而一个coroutine定义为

```c++
CoroTask coro(std::string, int);
```

那么编译器在调用coro时，会根据std::coroutine_traits<CoroTask, std::string, int>实例化的结果来找到CoroTask中的promise_type接口。

**coroutine接口中不定义promise type是否可行？**

std::coroutine_traits是允许偏特化的，因此可以不用在coroutine接口中定义promise_type依然可以保证coroutine正常work。

譬如，假设我们想让coroutine的返回值为std::string, 并在主线程中输出该返回值。那么可以偏特化std::coroutine_traits如下所示

```c++
    template <typename... Args> 
    struct std::coroutine_traits<std::string, Args...> {
        struct promise_type {
            std::string get_return_object() noexcept {
                return "hello promise_type";
            }
            std::suspend_never initial_suspend() const noexcept { return {}; }
            std::suspend_never final_suspend() const noexcept { return {}; }
            void unhandled_exception() noexcept { std::terminate(); }
            void return_void() {}
        };
    };

```

其相应的coroutine定义为

```c++
std::string coro(int max) {
    for (int i = 0; i <= max; ++i) {
        std::cout << "coro index: " << i << "\n";
    }
    std::cout << "coro end\n";
    co_return;
}
```

使用coroutine也很简单，即在主线程中直接调用

```c++
int main() {
    auto task = coro(5);
    std::cout << task << "\n";
    return 0;
}
```
相应的输出结果为
```c++
coro index: 0
coro index: 1
coro index: 2
coro index: 3
coro index: 4
coro index: 5
coro end
hello promise_type
```

上述简单的测试用例只是为了说明 

- coroutine的返回值(coroutine接口可以是任何类型)，

- coroutine接口可以不定义promise_type, 但需要在偏特化的std::coroutine_traits中定义promise_type

- coroutine中真正使用的promise_type为std::coroutine_traits中的promise_type

