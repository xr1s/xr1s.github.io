+++
title = "C++ 用户定义 Ranges 算子"
date = "2023-08-13"
type = "post"
tags = ["C/C++"]
+++

写这篇文章只是我在奇怪为什么没人介绍怎么自定义 C++ Ranges 标准库的 View Adaptor。虽然说 C++20 标准其实还没定义要怎么实现，但是不至于连 range-v3 的都没有吧？

C++20 新引入的 Range 的必要性无需赘述，它解耦了迭代和计算的逻辑，抽象出「范围」的概念，使得相比于旧的 Algorithm 库，Range 库中的算法更容易被组合，借此可以写出更清晰、运行效率更高的代码。

文中的代码故意不一次性完整给出，是为了读者能自己理解并整理代码。如果需要完整代码的话可以直接翻到文章最后看 [GitHub Gists](https://gist.github.com/xr1s/3aed5b933f256e8f4cf9a4c3f76a5c27)。

文章中涉及到少量 CRTP 的知识，可以前置阅读 [C++ CRTP 简介](/2022/10/18/brief-introduction-to-crtp/)。

## 概念

### 范围 Range

从定义上来讲，Range 就是有 `begin` 和 `end` 方法的类型。

另外有 `end` 方法并不意味着 Range 必须是个有限的区间，事实上它可以是无限的。比如 `istream_iterator`。

当然还有很多特殊情况，比如 C 风格的数组，尽管没有方法，它也被特别提及并定义为一个 Range；比如虽然有着 `begin` 和 `end` 方法，但是返回的类型并不是迭代器 `concept input_or_output_iterator` 的类型，这又不是 Range。过于细节的这里就不作阐述了，能理解就行。

```c++
template <class T>
concept range = requires(T &t) {
  std::ranges::begin(t);
  std::ranges::end(t);
};
```

从概念上来讲，Range 表达的是我们将要处理的一个数据序列。它不一定是连续的，不仅可能像 `std::forward_list` 一样在内存中不连续，它甚至可能是在 `std::vector` 选中的几个满足特定条件的一些元素，比如数组中所有的偶数数字这种非连续的序列。

### 视图 View

首先，View 是 Range，这意味着它有所有 Range 要求的属性，比如 `begin` 方法、`end` 方法、可迭代等等。

View 是视图，视图和容器的区别在于它不持有数据，它只是数据的一种展现方式，因此它的复制是 \\(O\(1\)\\) 的。这里的定义涉及到所有权的概念，但是因为这个概念在其它语言、应用场景（比如数据库视图）下也是互通的，所以不再展开。如果你理解 C++17 的 `std::string_view`，那你应该也能理解这里的 View。

C++ 的定义中，它要求 `T` 的视图 `std::ranges::view<T>` 是关于 `T` 的范围 `std::ranges::range<T>`、是继承自 `std::ranges::view_base` 或仅继承自 `std::ranges::view_interface<T>` 的，是继承自 `std::ranges::view_interface<T>` 的，且 `T` 是可移动的 `std::movable<T>`。

也就是

```c++
template <class T>
concept view =
    std::ranges::range<T> && std::movable<T> && std::ranges::enable_view<T>;

template <class T>
concept std::ranges::enable_view<T> =
    std::derived_from<T, view_base> || __is_derived_from_view_interface<T>;
```

### 范围适配器 Range Adaptor

需要注意 Range Adaptor 和 Range 的区别。Range 终究可以被迭代，这意味着它代表着的是数据；而 Range Adaptor 代表的是计算，或者说操作。Range Adaptor 和 Range 的关系是，我们可以通过 Range Adaptor 在 Range (View) 之间进行转换。

举个例子，`std::ranges::views::filter` 是一种 Range Adaptor，`std::ranges::transform` 是另一种 Range Adaptor。

### 范围适配器闭包 Range Adaptor Closure

Range Adaptor Closure 是 Range Adaptor 未传入 Range 的调用结果。

举个比较实际的例子，下面的代码中，

1. `container` 是一个 `std::vector<int>` 容器，也是 Range；
2. `std::ranges::views::filter` 和 `std::ranges::transform` 分别是两种 Range Adaptor 对象；
3. `filter([](int x) { return x % 2 != 0; })` 是 Range Adaptor Closure 对象；
4. `transform([](int x) { return x * x * x; })` 也是 Range Adaptor Closure 对象；
5. `container_view` 是 View，并且 `container` 和 `filter` 的中间的计算结果也是另一个 View。

```c++
std::vector<int> container = {-27, 50, 51, 72, 89, 14, 8, -43, -20, 62};

auto container_view = container
  | std::ranges::views::filter([](int x) { return x % 2 != 0; })
  | std::ranges::views::transform([](int x) { return x * x * x; });
```

## 实现

实现起来其实并不复杂，除了 View 和迭代器的定义之外，说白了就是多给 Adaptor Closure 实现一个 `operator|` 的事情。

这里顺便说一个八卦，Google C++ 编程规范中提到：

> Define overloaded operators only if their meaning is obvious, unsurprising, and consistent with the corresponding built-in operators. For example, use `|` as a bitwise- or logical-or, not as a shell-style pipe.
>
> 只有当重载运算符的含义显而易见、不足为奇并且与相应的内置运算符一致时，才定义重载运算符。例如，使用 `|` 作为位或逻辑或，而不是作为 shell 样式的管道。


不知道他们现在如何看待这个问题。

首先我们来实现一个 View，选一个 C++23 还没有的而且比较简单的来实现，比如 `chain`，含义是将两个 Range 前后串联起来。

从简单开始，为了避免复杂模板对代码的混淆影响到读者的理解，先只考虑实现一个只接受一对参数的 `ChainView`，也就是只接受 `chain(view, another_view)` 或 `view | chain(another_view)` 形式。

```c++
template <std::ranges::viewable_range LHS, std::ranges::viewable_range RHS>
class ChainView: public std::ranges::view_base {
};
```

`std::ranges::view_base` 其实是个空基类，它没有任何成员。这里继承的原因是标准要求所有 View 都要自此派生。

```c++
template <std::ranges::viewable_range LHS, std::ranges::viewable_range RHS>
class ChainView: public std::ranges::view_base {
private:
  LHS lhs;
  RHS rhs;
};
```

这里的 `std::ranges::views::all_t` 是无操作的 View Adaptor，它的作用在于避免复制，用于在 `ChainView` 中只保存数据的视图。

也就是说当输入的 Range 本身是持有数据的容器时， `all_t` 避免将容器复制到 `ChainView` 中，而只是保存容器的引用；而当 Range 已经是 View 的时候，它不会进行任何操作，因此也不会进行复制。

然后我们来实现 View 的核心逻辑——迭代器 Iterator，首先我们要保存底层迭代器类型方便后续处理。

```c++
template <std::ranges::viewable_range LHS, std::ranges::viewable_range RHS>
class ChainView: public std::ranges::view_base {
private:
  using LHSIterator = std::ranges::iterator_t<LHS>;
  using RHSIterator = std::ranges::iterator_t<RHS>;
  static_assert(
      std::same_as<
          typename std::iterator_traits<LHSIterator>::value_type,
          typename std::iterator_traits<RHSIterator>::value_type>,
      "简化起见, 迭代器返回的类型必须相同. 可以扩展成 std::common_type.");
  using ValueType = std::iterator_traits<LHSIterator>::value_type;

  class Iterator {
    // TODO
  };

  LHS lhs;
  RHS rhs;

public:
  ChainView() = default;

  ChainView(LHS lhs, RHS rhs): lhs{std::move(lhs)}, rhs{std::move(rhs)} {}
};
```

我们为 `ChainView` 实现模板实参推导，后面无论是通过 `chain()` 调用构造，还是用户手动构造的时候可以简化传参：


```c++
template <std::ranges::viewable_range LHS, std::ranges::viewable_range RHS>
ChainView(LHS &&, RHS &&) -> ChainView<
    std::ranges::views::all_t<LHS>,
    std::ranges::views::all_t<RHS>>;
```

接下来定义 Iterator，先简单搭好一些框架：

```c++
  class Iterator {
  private:
    ChainView *view = nullptr;
    std::variant<LHSIterator, RHSIterator> inner;

    friend class ChainView;

    // 构造器需要传入引用的 view
    // 因为在 ++iterator 跨越 LHS 和 RHS 的时候
    // 需要拿到 lhs.end() 和 rhs.begin()
    Iterator(ChainView *view, LHSIterator it)
        : view{view}, inner{std::move(it)} {}

    Iterator(ChainView *view, RHSIterator it)
        : view{view}, inner{std::move(it)} {}

  public:
    // concept std::ranges::input_range 需要这些定义
    // 满足 input_range 的 View 才能和标准库 Adaptor 互操作
    using difference_type = ptrdiff_t;
    using value_type = ValueType;

    // concept std::sentinel_for 需要实现默认构造函数
    // concept std::ranges::range 需要实现 sentinel_for
    Iterator() = default;
  };

  Iterator begin() { return Iterator{this, this->lhs.begin()}; }

  Iterator end() { return Iterator{this, this->rhs.end()}; }
};
```

然后来实现最重要的逻辑，`operator++` 和 `operator*`：

```c++
template <std::ranges::viewable_range LHS, std::ranges::viewable_range RHS>
class ChainView: public std::ranges::view_base {
private:
  class Iterator {
  public:
    Iterator &operator++() {
      std::visit(
          [this]<typename T>(T &&iterator) {
            using Iterator = std::remove_cvref_t<T>;
            // iterator 到达 lhs.end() 时，需要转移到 rhs.begin()
            if constexpr (std::same_as<Iterator, LHSIterator>) {
              ++std::forward<T>(iterator);
              if (std::forward<T>(iterator) == this->view->lhs.end()) {
                this->inner = this->view->rhs.begin();
              }
            }
            // iterator 到达 rhs.end() 后不需要转移，可以被识别为结束
            if constexpr (std::same_as<Iterator, RHSIterator>) {
              this->inner = ++std::forward<T>(iterator);
            }
          },
          this->inner);
      return *this;
    }

    // concept std::input_or_output_iterator 需要后置自增
    Iterator operator++(int) {
      Iterator that = *this;
      ++*this;
      return that;
    }

    ValueType &operator*() const & {
      return std::visit(
          []<typename T>(T &&iterator) -> ValueType & {
            using Iterator = std::remove_cvref_t<T>;
            if constexpr (std::same_as<Iterator, LHSIterator>) {
              return *std::forward<T>(iterator);
            }
            if constexpr (std::same_as<Iterator, RHSIterator>) {
              return *std::forward<T>(iterator);
            }
            std::unreachable();
          },
          this->inner);
    }
};
```

到此为止，我们已经实现了最核心的逻辑接口，已经可以手动构造 `ChainView` 进行迭代了：

```c++
int main() {
  std::vector<int> vi = {0, 2, 4, 6, 8};
  std::deque<int> di = {1, 3, 5, 7, 9};

  ChainView cv{vi, di};
  for (const auto &i: cv) {
    std::println("{}", i);
  }
}
```

但是目前还没法使用 `chain(vi, di)` 或者 `vi | chain(di)` 形式调用，没那味儿。接下来我们实现一个 Adaptor。这个 Adaptor 应该可以接受双参数，这样才能用 `chain(vi, di)` 的形式直接构造 `ChainView`，这个 Adaptor 也要能接受单个参数，才能使用 `vi | chain(di)` 的形式调用。

我们先来实现双参数的，直接返回 `ChainView` 即可。

```c++
struct ChainAdaptor {
  template <std::ranges::viewable_range LHS, std::ranges::viewable_range RHS>
  auto operator()(LHS &&lhs, RHS &&rhs) const {
    return ChainView{std::forward<LHS>(lhs), std::forward<RHS>(rhs)};
  }
};

inline constexpr ChainAdaptor chain;

int main() {
  std::vector<int> vi = {0, 2, 4, 6, 8};
  std::deque<int> di = {1, 3, 5, 7, 9};

  for (const auto &i: chain(vi, di)) {
    std::println("{}", i);
  }
}
```

为了实现 `vi | chain(di)` 的调用形式，我们还需要实现一个 Closure 来暂存 `di`。

```c++
template <std::ranges::viewable_range View>
struct ChainAdaptorClosure {
private:
  View view;

public:
  explicit ChainAdaptorClosure(View view): view{std::move(view)} {}

  template <std::ranges::viewable_range LHS>
  friend auto operator|(LHS &&lhs, ChainAdaptorClosure<View> self) {
    return ChainView{std::forward<LHS>(lhs), self.view};
  }
};

template <std::ranges::viewable_range View>
ChainAdaptorClosure(View &&)
    -> ChainAdaptorClosure<std::ranges::views::all_t<View>>;

struct ChainAdaptor {
  template <std::ranges::viewable_range View>
  auto operator()(View &&view) const {
    return ChainAdaptorClosure{std::forward<View>(view)};
  }
};

int main() {
  std::vector<int> vi = {0, 2, 4, 6, 8};
  std::deque<int> di = {1, 3, 5, 7, 9};

  auto view = vi | chain(di)
      | std::ranges::views::filter([](int value) { return value % 3 != 0; })
      | std::ranges::views::transform(
                  [](int value) { return std::to_string(value); });
  std::ranges::for_each(view, [](auto value) { std::println("{}", value); });
}
```

最后一点，是 Closure 之间的组合，两个 Closure 之间可以执行 `operator|` 操作结合成全新的 Closure，然后整体传递给其它 Range 一起运算。这里我们还会需要一个 Closure 来暂存两个 Closure 运算的结果。但是我实在是懒得写了，可以自行意会一下，这个方法没有实现不影响 ChainView 的大部分使用场景。

到此为止，已经实现一个可以参与到标准库中的 Adaptor 了。

可能有人会想试一试能否参与到 [range-v3](https://github.com/ericniebler/range-v3) 库的互相调用。理论上可以实现，但是上面的代码需要稍作改动，读者可以自行研究一下报错看看还缺了什么。

两个提示：

1. `std::ranges::view_base` 和 `ranges::view_base` 是两个不同的类型；
2. range-v3 要求用户自定义的 View 满足 `ranges::default_constructible`，但是 `std::ranges::ref_view` 并不是默认可构造的。

## 标准库帮助类

这里开始会有一些 CRTP 相关知识，再次提醒如果不知道 CRTP 的主要作用是用来给派生类生成方法，可以先阅读 [C++ CRTP 简介](/2022/10/18/brief-introduction-to-crtp/)。

实际上，如果用户要自己自定义 Range Adaptor，上面的代码有很多都可以重复利用，于是标准库也提供了一些帮助类，比如 `std::ranges::view_interface` 和 `std::ranges::range_adaptor_closure` 模板类。这两个类都是 CRTP 模板基类，需要派生类实现了某个方法才能继承。

`std::ranges::view_interface` 会根据用户定义 View 类型的 `begin()` 和 `end()` 方法，以及迭代器的类型生成常见的容器方法，比如 `empty()`、`cbegin()`、`cend()`、`size()`、`front()` 、`back()`，甚至 `data()` 等。

`std::ranges::range_adaptor_closure` 会根据用户 Closure 的 `operator()` 方法生成与 Range 运算的 `operator|` 方法和与另一个 Closure 运算的 `operator|` 方法，有效降低重复逻辑。

另外还有一处逻辑重复比较多的代码，就是通过 Adaptor 构造 Closure 的。很明显不论你是实现哪种 Range View Adaptor，他们 Adaptor 实现的重复度都极高，都是相似两种 `operator()`，一种直接返回 View 一种返回 Closure，而这个 Closure 也可以用一个通用类型实现。但是可惜标准库没有提供这种东西。这里可以参考 libstdc++ 的 `std::ranges::views::__adaptor::_RangeAdaptor` CRTP 模板基类，只需要实现 `operator()` 以及定义两个 static 常量，就可以直接生成 Adaptor 和复用 libstdc++ 的 Closure 类型，也就是 `std::ranges::views::__adaptor::_Partial` 类，可以大幅减少重复逻辑。就是可惜没有进标准库。

# 成品

其实说是成品还有瑕疵，因为还不支持 Closure 之间的计算。难度应该不大，等我有兴趣了再改。

不过如果一周内没加上那就有可能永远都不会加上了。

[C++ Chain Range Adaptor](https://gist.github.com/xr1s/3aed5b933f256e8f4cf9a4c3f76a5c27).
