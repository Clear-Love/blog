---
title: "C++小技巧"
date: 2024-08-19T15:34:56+08:00
draft: true
---

## lambda表达式与立即调用

今天用C++刷leecode的时候，看到别人提交的一段奇怪的代码，记录一下

```cpp
auto init = [](){
    ios::sync_with_stdio(0), cin.tie(0), cout.tie(0);
    return 'c';
}();
```
这段代码用到了一个技巧：Lambda 表达式与立即调用，Lambda 表达式：[](){ ... } 定义了一个无参数、返回值类型为 char 的 Lambda 表达式。

立即调用：() 表示立即调用这个 Lambda 表达式，并返回其结果。

这段代码的作用是：在程序开始时立即执行 Lambda 表达式中的代码，即设置输入输出流的同步方式，并绑定输入输出流的缓冲区。这样可以加快程序的输入输出速度，并减少程序的延迟。

这种技巧的好处是：可以在程序开始时立即执行一些初始化操作，而不需要等到程序的其他部分调用这些操作。这样可以加快程序的启动速度，并减少程序的延迟。


## lambda 表达式和递归

在C++中，lambda表达式可以捕获外部变量，但是不能直接递归调用自己。如果需要递归调用lambda表达式，可以使用一个函数指针或者一个函数对象来实现。

例如，下面的代码定义了一个lambda表达式，它捕获了一个函数指针，并使用这个函数指针来递归调用自己：

```cpp
#include <bits/stdc++.h>

using namespace std;

int main() {
    function<int(int)> dfs;
    dfs = [&](int n) {
        if (n == 0) {
            return 1;
        } else {
            return n * dfs(n - 1);
        }
    };

    cout << dfs(5) << endl; // 输出 120

    return 0;
}
```

在这个例子中，lambda表达式f捕获了一个函数指针self，并使用这个函数指针来递归调用自己。这样就可以实现递归调用lambda表达式了。

之前我都是这样使用c++的递归，但是有一次刷题的时候尽然发现超时了，我连忙研究了一下别人的写法

![](https://pan.lmio.xyz/pic/40fb76b29f8e5be206ebfed900c0544d.png)

- 通过代码为下面这种写法

```cpp
#include <bits/stdc++.h>

using namespace std;

int main() {
    auto dfs = [&](auto&& dfs, int n) {
        if (n == 0) {
            return 1;
        } else {
            return n * dfs(dfs, n-1);
        }
    };

    cout << dfs(dfs, 5) << endl; // 输出 120

    return 0;
}
```

为什么使用显示传递lambda表达式，就可以提高效率避免超时呢？

这是因为C++的lambda表达式在编译时会生成一个匿名类，这个匿名类会捕获外部变量，并生成一个operator()函数。在递归调用时，每次都会生成一个新的匿名类对象，这会导致额外的开销。而使用显示传递lambda表达式，可以避免生成新的匿名类对象，从而提高效率。

如果dfs 采用捕获的方式传递, 相当于：

```cpp
#include <bits/stdc++.h>

using namespace std;

int main()
{
  std::function<int (int)> dfs = std::function<int (int)>();
    
  class __lambda_7_11
  {
    public: 
    inline /*constexpr */ int operator()(int n) const
    {
      if(n == 0) {
        return 1;
      } else {
        return n * dfs.operator()(n - 1);
      } 
      
    }
    
    private: 
    std::function<int (int)> & dfs;
    public: 
    // inline /*constexpr */ __lambda_7_11(const __lambda_7_11 &) noexcept = default;
    // inline /*constexpr */ __lambda_7_11(__lambda_7_11 &&) noexcept = default;
    __lambda_7_11(std::function<int (int)> & _dfs)
    : dfs{_dfs}
    {}
    
  } __lambda_7_11{dfs};
  
  dfs.operator=(__lambda_7_11);
  std::cout.operator<<(dfs.operator()(5)).operator<<(std::endl);
  return 0;
}

```

如果使用显示传递lambda表达式，相当于：

```cpp
#include <bits/stdc++.h>

using namespace std;

int main()
{
    
  class __lambda_6_16
  {
    public: 
    template<class type_parameter_0_0>
    inline /*constexpr */ auto operator()(type_parameter_0_0 && dfs, int n) const
    {
      if(n == 0) {
        return 1;
      } else {
        return operator*(n, dfs(dfs, n - 1));
      } 
      
    }
    
    #ifdef INSIGHTS_USE_TEMPLATE
    template<>
    inline /*constexpr */ int operator()<__lambda_6_16 &>(__lambda_6_16 & dfs, int n) const
    {
      if(n == 0) {
        return 1;
      } else {
        return n * dfs.operator()(dfs, n - 1);
      } 
      
    }
    #endif
    
    
  };
  
  __lambda_6_16 dfs = __lambda_6_16{};
  std::cout.operator<<(dfs.operator()(dfs, 5)).operator<<(std::endl);
  return 0;
}

```

可以看到，方式一依赖于function的类型擦除，std::function 通过类型擦除实现了持有任意类型的可调用对象。这种灵活性意味着每次调用时需要解析和调用存储在 std::function 内部的实际可调用对象，这可能导致额外的运行时开销。

方式二，通过将自身传递给自身来实现递归，更接近于普通函数。这种方式避免了 std::function 带来的额外开销，方便编译器进行内联优化，可以更好地利用编译器的优化。

总之，虽然第一种写法比较方便，但如果代码递归深度比较高，可以能会导致超时，建议使用第二种写法。