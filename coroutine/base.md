# 基本概念 (是什么)

**协程(coroutine):** 是一种特殊的函数，其可以被暂停(suspend), 恢复执行(resume)。一个协程可

以被多次调用。

**协程(coroutine):** 分为stackless和stackful两种，所谓stackless协程是指协程被suspend时不

需要堆栈，而stackful协程被suspend时需要堆栈。C++中的协程属于stackless协程。

C++20开始引入的协程由于如下等原因难以学习

- 围绕协程实现的相应组件多(譬如co_wait, co_return, co_yeid， promise，handle等组件)

- 灵活性高，有些组件提供的接口多，可由用户自己控制相应的实现

- 组件之间的关系也略复杂

- 编译器内部转换

首先，实现自己的第一个协程

# 体验协程

C++20与协程相关的组件包括

- 协程接口(也即协程返回类)

- promise对象

- coroutine handle

- awaitable 和 awaiter

- 以及相应的三个关键字 co_wait, co_return, co_yeid

下面看一个实际的hello协程的例子，其代码实现为

```c++
// coro_task.h
#include <coroutine>
#include <exception>
#include <type_traits>

class Task {
public:
  struct promise_type;
  using TaskHd1 = std::coroutine_handle<promise_type>;
private:
  TaskHd1 hd1_;

public:
  Task(auto h) : hd1_{h} {}
  ~Task() {
    if (hd1_) { hd1_.destroy(); }
  }

  Task(const Task&) = delete;
  Task& operator=(const Task&) = delete;

  bool resume() {
    if (!hd1_ || hd1_.done()) {
        return false;
    }
    hd1_.resume();
    return true;
  }

public:
  struct promise_type {
    /* data */
    auto get_return_object() {
        return Task{TaskHd1::from_promise(*this)};
    }
    auto initial_suspend() { return std::suspend_always{};}
    void unhandled_exception() { std::terminate();}
    void return_void() {}
    auto final_suspend() noexcept { return std::suspend_always{}; } 
  };
};

```

```c++
// main.cc
#include <iostream>

#include "coro_task.h"

 Task hello(int max) {
    std::cout << "hello world\n";
    for (int i = 0; i < max; ++i) {
        std::cout << "hello " << i << "\n";
        co_await std::suspend_always{};
    }
    
    std::cout << "hello end\n";
}

int main() {
    auto co = hello(3);
    while (co.resume()) {
        std::cout << "hello coroutine suspend\n";
    }
    return 0;
}
```

编译器构建命令为
```c++
g++ -o main main.cc coro_task.h -std=c++20
```

运行结果如下

```c++
hello world
hello 0
hello coroutine suspend
hello 1
hello coroutine suspend
hello 2
hello coroutine suspend
hello end
hello coroutine suspend
```

从总体上来看，协程和调用者之间的关系可由下图表示

![hello](./1.png)



