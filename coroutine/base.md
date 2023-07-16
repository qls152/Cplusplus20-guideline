# 基本概念 (是什么)

**协程(coroutine):** 是一种特殊的函数，其可以被暂停(suspend), 恢复执行(resume)。一个协程可

以被多次调用。

**协程(coroutine):** 分为stackless和stackful两种，所谓stackless协程是指协程被suspend时不

需要堆栈，而stackful协程被suspend时需要堆栈。C++中的协程属于stackless协程。

C++20开始引入的协程由于如下等原因难以学习

- 围绕协程实现的相应组件多(譬如co_wait, co_return, co_yield， promise，handle等组件)

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

- 以及相应的三个关键字 co_wait, co_return, co_yield

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

在图中，可以把协程看作生产者，调用者看作消费者，把协程的返回值(协程接口，看作泛管道)。

此外图中出现的编程人员主要负责根据c++标准所提供的接口，可自定义泛管道的一些行为，以控制协程进行不同的操作。

若要实现一个协程，需要首先提供一个协程接口，譬如Task，在协程接口中需要提供

- promise_type

- std::coroutine_handle<promise_type>

在协程的函数体中需要使用co_await, co_yield, co_return之一的关键词。

## hello协程工作过程

此部分主要讲解hello协程的工作过程

- a. 协程的调用同函数相同，在main函数中，通过如下形式调用协程

```c++
auto co = hello(3);
```

- b. 通过调用hello(3), 启动协程，协程立即暂停(suspend), 并返回协程接口Task对象给调用者

- c. 在main函数中，调用co实例的resume接口，通过coroutine handle恢复协程的执行

- d. 在协程中，进入for循环，初始化局部变量，并到达暂停点(suspend point), 暂停点由co_await expr确定

- e. 协程暂停后将控制权转移到main函数，main函数继续运行并从新恢复协程

- f. 协程恢复执行，继续for循环，i的值增加。再一次到达暂停点(suspend point)。转换控制权到main函数。

- g. 最终for循环结束，协程离开for循环。并将控制权返回到到main函数，main函数退出循环，并销毁协程。

## promise_type

promise_type可以用来控制协程的行为。





