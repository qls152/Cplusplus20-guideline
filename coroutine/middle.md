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

# 深入理解promise_type类型

这部分的内容需要感谢Lewis Baker的blog。当coroutine被调用时，在coroutine body被执行之前，标准规定其要执行如下步骤

- 分配coroutine frame，用来存储coroutine所需要的数据。coroutine frame一般分配在堆上。可以通过定制operator new来将coroutine frame分配在栈上。

- coroutine所有参数拷贝到coroutine frame。**需要注意：若coroutine的参数为引用类型，那么其拷贝的按照引用拷贝到coroutine frame中.**

- 在coroutine frame中创建promise_type对象。它的目的是为了保存coroutine的状态，以及提供一些定制点控制coroutine的行为。

具体而言，编译器在底层将coroutine转换为如下方式

```c++
CoroutineParams params; // 拷贝coroutine的参数
promise_type P;

CoroutineInterface Obj = P.get_return_object();

// 执行coroutine源码
try {
    co_await P.initial_suspend();
    // coroutine的源代码
    // 在该部分，会将co_return ==> P.return_void()
    // co_return val ==> P.return_value(val)
    // co_yield val ==> P.yield_value(val)
} catch (...) {
    P.unhandle_exception();
}

co_await P.final_suspend();
```

上述需要做如下说明

- 在coroutine开始时，调用get_return_object()生成coroutine接口的对象，当coroutine碰到第一个暂停点(suspend)时返回给coroutine调用方

- 在coroutine内部，最终会将promise对象转换为co_await expr形式

- 当coroutine结束时，运行时会保证首先调用promise_type对象的析构函数，然后调用相应coroutine参数的析构函数

- 当析构完成后，会调用operator delete销毁coroutine frame

那么coroutine的suspend点和resume点是在哪里呢？

# Awaitable 和 awaiter

Awaitable: 用于操作符co_await的操作数。更具体一些来说，Awaitable是一种规范/约束。co_await的操作数是满足Awaitable这种规范的对象(Awaitable对象)。该约束要求

- coroutine是否需要暂停

- 在coroutine暂停后能够执行一些逻辑操作

- 在coroutine 恢复后可以执行一些逻辑操作

awaiter: 是一个类型，该类实现了三种接口

- await_ready

- await_suspend

- await_resume

每当编译器看到 co_await expr语句时，编译器会做大量的转换工作。

**对表达式求值**

**获得awaitable对象**

**转换co_await表达式**

获得awaitable对象的步骤可归结为如下过程

```c++
template <typename PromiseType, typename T>
decltype(auto) getAwaitable(PromiseType& p, T&& expr) {
    if constexpr (has_any_await_transform_member_v<PromiseType>) {
        return p.await_transform(static_cast<T&&>(expr));
    }
    return static_cast<T&&>(expr);
}

template <typename Awaitable>
decltype(auto) getAwaiter(Awaitable&& a) {
    if constexpr (has_member_operator_co_await_v<Awaitable>) {
        return static_cast<Awaitable&&>(a).operator co_await();
    }

    if constexpr(has_non_member_operator_co_await_v<Awaitable>) {
        return operator co_await(static_cast<Awaitable&&>(a));
    }

    return static_cast<Awaitable&&>(a);
}
```

上述需要做如下说明：

- awaitable对象由expr所产生的值和coroutine的promise_type对象确定

- 若coroutine的promise_type定义的await_transform成员函数，那么此时awaitable对象便是该函数的求值结果

- 若coroutine的promise_type未定义await_transform成员函数，那么直接将表达式的值转换为awaitable对象

- 随后，为了获得真正的co_await的参数（awaiter）,  会继续判断awaitable对象中是否重载了co_await操作符，若定义了相应的co_await操作符，那么调用该重载函数生成相应的awaiter

- 若awaitable对象中未重载co_await操作符，那么判断是否有非成员的co_await操作符，若存在，则调用该操作符生成相应的awaiter对象

- 若都不存在，那么直接将awaitable对象强制转换为awaiter对象

当编译器生成了awaiter对象后，会在coroutine体中生成如下伪代码

```c++
if (!awaiter.await_ready()) {
    using handle_t = std::coroutine_handle<PromiseType>;
    using await_suspend_result_t = decltype(awaiter.await_suspend(handle_t::from_promise(p)));

    // suspend point

    if constexpr (std::is_void_t<await_suspend_result_t>) {
        awaiter.await_suspend(PromiseType);
        // return to caller / resume coroutine
    } else {
        if (awaiter.await_suspend(PromiseType)) {
            // return to caller / resume coroutine
        }
    }
    // resume point
}
return awaiter.await_resume();
```

每个awaiter对象必须实现如下三个接口

- await_ready()

这个函数在coroutine被suspend之前调用。该接口决定coroutine是否被suspend，若该接口返回true, 则coroutine不会被suspend。

- await_suspend(awaitHdl)

这个函数在coroutine被suspend之后调用。awaitHdl是正在被suspend的coroutine的coroutine_handle。在这个接口中，你可以决定进行一些操作，譬如你可以resume正在suspend的其他coroutine等

-  await_resume()

这个函数在coroutine被resume之后调用。该函数所产生的值便是co_await表达式所产生的值。


