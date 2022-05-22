# Lambdas & Coroutines

## Lambdas：从一个例子开始

假设我们收集了下面的成绩：

```cpp
struct Rank {
    int id;
    int time;  // 用时
    int score; // 分数
};

std::vector<Rank> container;
```

标准库提供了 `std::sort`，我们怎样告诉标准库我们自定义的优先级？

对 `Rank` 进行排序？

- `time` 越短越好
- `score` 越高越好
- 综合两项？

让我们来的话，我们如何设计标准库？

### 一种做法——函数指针


```cpp
bool compareT(const Rank &a, const Rank &b) {
  return a.time < b.time;   // time 越短越好
}
bool compareS(const Rank &a, const Rank &b) {
  return a.score > b.score; // score 越高越好
}
```

`std::sort` 通过函数指针获取优先级，这样用就好啦：

```cpp
std::sort(container.begin(), container.end(), &compareT);
std::sort(container.begin(), container.end(), &compareS);
```

综合 `time` 和 `score` 排序：

```cpp
bool compare(const Rank &a, const Rank &b) {
  return 0.1*a.time - a.score
         < 0.1*b.time - b.score;
} // 0.1 是综合系数，越大的话时间对排名影响越大
```

想要有多种排序方式？不考虑模板，只能手动改系数：

    `compare1`, `compare2`, `compare3` ...

原因：函数本身无法储存可变状态（即 stateless）
解决：在外面再包一层对象，用对象储存我们的综合系数：

```cpp
struct Compare {
  typedef const Rank& ref;
  virtual bool operator()(ref a, ref b);
}; // 我们的 our::sort 的参数，当然与实际 std::sort 不同
struct OurCompare {
  float ratio = 0.1;
  bool operator()(ref a, ref b) {
    return ratio*a.time - a.score
           < ratio*b.time - b.score;
  }
};
```

这样使用：

```cpp
OurCompare c{0.2};
our::sort(container.begin(), container.end(), &c);
```

思想：一个对象，储存了我们所需的状态，以及指向我们函数的指针（上面的 `virtual` 变相把函数指针储存了，但与 lambda 不同）

### Lambda 的使用

```cpp
float ratio = 0.1;
auto compare = [&ratio](ref a, ref b) {
  return ratio*a.time - a.score
         < ratio*b.time - b.score;
};
std::sort(container.begin(), container.end(), compare);
ratio = 0.2;
std::sort(container.begin(), container.end(), compare);

std::string prefix = "Id: ";
std::for_each(container.begin(), container.end(), [&prefix](Rank &r) {
  std::cout << prefix << r.id << std::endl;
});
```

### Lambda：匿名⋅函数⋅对象

- 对象：本质是对象：储存了来自上下文的状态，函数指针等
- 函数：我们最关心的功能部分
- 匿名：既然是对象，可以匿名也是自然的。

## Coroutines：从一个例子开始

输出斐波那契数列：

```cpp
int a = 1, b = 1;
for (int i = 0; i < N; ++i) {
  std::cout << a << std::endl;
  a += b;
  std::swap(a, b);
} // 输出 1, 1, 2, 3, 5, …
```

生成数据以及输出是不同功能，如何解耦？我们可以把它包装成一个“一个一个输出数列数字的对象” `f`，`f.next()` 每次生成新值：

```cpp
class Fib {
private:
  int a = 0, b = 1;
  int next() {
    a += b;
    std::swap(a, b);
    return a;
  }
};

// ...

Fib f;
f.next(); // -> 1
f.next(); // -> 1
f.next(); // -> 2
// ...
```

更复杂的情形？

```cpp
int a = 1, b = 1;
for (int i = 0; i < N; ++i) {
  “return” a; // ？？？
  a += b;
  std::swap(a, b);
}
for (int j = 0; j < M; ++j) {
  “return” a; // ？？？
  a -= 2;
}
```

上面更复杂的情况有两个 `return` 的地方，如果我们还是只靠一个 `next` 函数来获取下一个值的话我们就需要记录下我们轮到哪个 `return` 了。随着我们原来直观的 for, cout 变复杂，我们把它展开为一个一个逐个输出的形式所需要记录的状态也越来越复杂。<del>但的确是可以人工转化过来的。</del>

![GCC](https://gcc.gnu.org/img/gccegg-65.png)

    这是GCC，它帮你完成了上面的工作。说：“谢谢GCC。”

### Coroutines To The Rescue (?)

用 C++ 新的 coroutine 功能，要完成以上的功能，我们写的核心代码如下：

```cpp
MyGenerator<int> fib() {
  int a = 1, b = 1;
  for (int i = 0; i < N; ++i) {
    co_yield a;
    a += b;
    std::swap(a, b);
  }
}
```

看起来的确是直接把上面 `cout` 循环改成了 `co_yield` 就可以了。等等，`MyGenerator<int>` 是什么？

<details>
<summary><code>MyGenerator</code> 的代码</summary>

```cpp
#include <coroutine>
template<typename T>
struct MyGenerator {
  struct promise_type;
  using handle_type = std::coroutine_handle<promise_type>;
  struct promise_type {
    T value_;
    std::exception_ptr exception_;
    MyGenerator get_return_object() {
      return MyGenerator(handle_type::from_promise(*this));
    }
    std::suspend_always initial_suspend() { return {}; }
    std::suspend_always final_suspend() noexcept { return {}; }
    void unhandled_exception() { exception_ = std::current_exception(); }
    template<std::convertible_to<T> From>
    std::suspend_always yield_value(From &&from) {
      value_ = std::forward<From>(from);
      return {};
    }
    void return_void() {}
  };
  handle_type h_;
  
  
  
  MyGenerator(handle_type h) : h_(h) {}
  ~MyGenerator() { h_.destroy(); }
  explicit operator bool() {
    fill();
    return !h_.done();
  }
  T operator()() {
    fill();
    full_ = false;
    return std::move(h_.promise().value_);
  }
private:
  bool full_ = false;
  void fill() {  
    if (!full_) {
      h_();
      if (h_.promise().exception_)
        std::rethrow_exception(h_.promise().exception_);
      full_ = true;
    }
  }
};
```

</details>

让我们略去亿点点细节。总体上，C++ coroutine 引入了下面这些概念：

- `coroutine_handle`：储存“状态信息”
- `awaitable`：涉及 `co_await`，这里不多介绍
- `promise_type`：储存返回值，可自定义 `coroutine` 运行
- `generator`：最外层的把逻辑隐藏起来的包装

为支持各种自定义引入了一大堆东西，但我们最关心的逻辑部分还是斐波那契数列部分的代码。

（现在 C++ 20 还需要自己写，但是 C++ 23 以及往后标准库可能会直接提供一些常用的。也可以用第三方库）

## 总结：Lambdas & Coroutines

- 语法糖？
  > 指计算机语言中添加的某种语法，这种语法对语言的功能没有影响，但是更方便程序员使用。
  > ——Wikipedia
- 从无状态函数到有状态的对象：

  思想：面对过程 ⟶ 面对对象？
