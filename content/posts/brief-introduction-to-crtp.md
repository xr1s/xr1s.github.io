+++
title = "C++ CRTP 简介"
date = "2022-10-18"
type = "post"
tags = ["C/C++"]
aliases = ["/2018/05/10/brief-introduction-to-crtp/"]
+++

看标题就知道我打算重写一遍之前的 CRTP 文章，加上既然 C++23 快要出来了，我也研究了一下 C++23 的 CRTP 能不能有一些变化。

<!--more-->

题外话：Hugo 的 date 用的特么的是 UTC 时间，过了零点后我早上八点前写的文章它默认不会发布，得加上 buildFuture = true 这个配置项才会发布，害我调试了半天。

另外之前备份的 WordPress 数据也是直接 mysqldump 一把梭下来的，全是压成一片的转义符号，为了拿到正常的数据我还得先扔回 MySQL 里，大无语。

## 所谓 CRTP

所谓 CRTP，全称是 Curiously recurring template pattern，中文叫“奇异模板递归模式”，有着很奇异的名字。为什么说它奇异呢，是因为一个派生类的基类，它的模板参数中竟然有这个派生类自己。不过虽然名字里有一个奇异，它现如今已经被广泛地在各种类库中被使用了，甚至标准库中都有使用的例子，比如我们的 `std::enable_shared_from_this`，具体功能这里就不介绍了。

另一项可能可以说是进入标准库的来自于 Boost 的 operators.hpp 头文件，它也大量使用了 CRTP 的特性。不过这里说法其实并不准确，算是我硬凑到 C++ 的一个例子，进核心语言特性的是它其中的一部分 `boost::less_than_comparable` 这一族比较运算符的功能，进标准后被叫做三路比较运算符 `operator<=>`，也就是实现一个 `operator<=>` 编译器就能自动实现其它的比较运算符。

CRTP 的主要作用往往是基类需要获取一些派生类信息，或许是成员，或许是方法，或许是类型本身，再通过在基类中实现方法继承来扩展派生类本身，而且可以避免虚函数、动态分派带来的额外运行时代价的问题。比如 `std::enable_shared_from_this` 就是获取派生类类型本身来给派生类生成 `shared_from_this` 方法；而 `boost::less_than_comparable` 就是获取一个派生类方法 `operator<` 来给派生类生成其它的比较运算符。在其它语言中和 CRTP 比较类似的概念是 interface 的 default implementation，相比于此 CRTP 实现起来更加奇异一些，毕竟其非语言特性，只是 CRTP 没有运行时代价。

CRTP 往往因为用到了继承的能力而被误用，这里注意它**不能**被用于动态调度，也就是将一系列继承于不同模板参数的同一个 CRTP 模板类放到同一个容器里。有些人为了能运行时调度而加入了一个总的基类使得所有 CRTP 虚继承自此，这是完全错误的。CRTP 是用来消除运行时多态的，加入全局虚基类会凭空创造运行时代价，这就完全抹煞了 CRTP 的作用。

再说一遍，它的主要能力类似于其它语言的 default implementation，但它不是 interface。不要想着对不同的派生于 CRTP 的类型进行统一管理，因为模板参数的存在，它们不是相同的类型，不要对其进行虚继承等操作。就好像你不会觉得 `std::vector<int>` 和 `std::vector<long>` 是相同类型一样，它们也不会继承自同一个虚基类（虽然实现上确实会有非虚继承，但那是为了实现通用代码，避免实例化导致的代码膨胀，优化编译速度和产物体积）。

## 从委托模式开始

为什么选用委托模式，因为这是我觉得比较相似的一种模式，它还出现在 *Head First Design Pattern* 一书中，应该是属于比较简单常见的一种模式。

来简单实现一个会叫的鸭子类型：

```java
interface QuackBehavior {
    public void quack();
}

class Quack implements QuackBehavior {
    public void quack() { System.out.println("Quack"); }
}

// 哑巴
class MuteQuack implements QuackBehavior {
    public void quack() { /* pass */ }
}

// 正常鸭子
public abstract class Duck {
    QuackBehavior quackBehavior;

    public Duck(QuackBehavior quackBehavior) {
        this.quackBehavior = quackBehavior;
    }

    public void quack() { this.quackBehavior.quack(); }
}

class MallardDuck extends Duck {
    public MallardDuck() { super(new Quack()); }
}

// 橡皮鸭子
class RubberDuck extends Duck {
      public RubberDuck() { super(new MuteQuack()); }
}
```

它将“叫”的行为抽离出来，将不同类型的鸭子抽象成多个行为的组合，这样我们就不再关心鸭子到底是什么，我们只关心它的行为。将每个行为，比如“叫”、“游”等委托给具体的行为类型来执行，所以它被叫做委托模式。

不过其实这只是继承或者说组合的能力，还没到 CRTP 发挥自己作用的地方。只是 Java 没法多继承，所以搞了个委托模式来做。当然，现在的 Java 也支持 default implementation，咱先不讨论这个。

## C++ 中的 CRTP

在 C++ 里，我们可以直接多继承。实现上面的功能甚至不需要虚函数，直接继承就好了，Java 是默认虚函数、禁止多继承，那没办法。

```c++
struct Quack {
  void quack() { fmt::print("quack\n"); }
};

struct Dump {
  void quack() { fmt::print("silent\n"); }
};

struct Swim {
  void swim() { fmt::print("swim\n"); }
};

struct Float {
  void swim() { fmt::print("float\n"); }
};

// 绿头鸭，会叫，也会游泳
struct MullardDuck: Quack, Swim {};

// 橡胶鸭，不会叫，也不会游泳
struct RubberDuck: Dump, Float {};

int main() {
  MullardDuck mullard{"duck 007"};
  mullard.quack();

  RubberDuck doll{"007's duck doll"};
  doll.swim();
}
```

但是，当我们的基类需要依赖派生类的实现，那似乎就必须要虚函数了——不要在意菱形继承。

```c++
// 我们需要派生类实现 name 方法，来给基类提供额外的信息。
struct Named {
  // std::string_view 是 c++17 的，不过可以用 boost 的平替，所以不用太在意
  virtual std::string_view name() = 0;
};

struct Quack: virtual Named {
  void quack() { fmt::print("{} quacks\n", this->name()); }
};

struct Dump: virtual Named {
  void quack() { fmt::print("{} keeps silence\n", this->name()); }
};

struct Swim: virtual Named {
  void swim() { fmt::print("{} swims\n", this->name()); }
};

struct Float: virtual Named {
  void swim() { fmt::print("{} floats\n", this->name()); }
};
```

这样就简单了，只要派生类实现了 `Named`，也就是 `name` 方法，那么就可以无痛继承前面几个类型，并可以调用相应的 `quack` 或 `swim` 方法。

```c++
struct MullardDuck: Quack, Swim {
  MullardDuck(std::string name): name_{std::move(name)} { }

  virtual std::string_view name() override { return this->name_; }

private:
  std::string name_;
};

struct RubberDuck: Dump, Float {
  RubberDuck(std::string name): name_{std::move(name)} { }

  virtual std::string_view name() override { return this->name_; }

private:
  std::string name_;
};
```

但是众所周知，这个世界上有两件事是我们 cpper 所厌恶的，一是菱形继承，二是额外的运行时代价。这就是 CRTP 能帮我们消灭的敌人。

```c++
template <typename Named> struct Quack {
  void quack() {
    fmt::print("{} quacks\n", static_cast<Named *>(this)->name());
  }
};

template <typename Named> struct Dump {
  void quack() {
    fmt::print("{} keeps silence\n", static_cast<Named *>(this)->name());
  }
};

template <typename Named> struct Swim {
  void swim() {
    fmt::print("{} swims\n", static_cast<Named *>(this)->name());
  }
};

template <typename Named> struct Float {
  void swim() {
    fmt::print("{} floats\n", static_cast<Named *>(this)->name());
  }
};
```

发生了什么？我们给每个基类都添加了一个模板参数，而在定义派生类的时候，我们将派生类本身作为模板参数传入：

```c++
struct MullardDuck: Quack<MullardDuck>, Swim<MullardDuck> {
  MullardDuck(std::string name): name_{std::move(name)} { }

  std::string_view name() { return this->name_; }

private:
  std::string name_;
};

struct RubberDuck: Dump<RubberDuck>, Float<RubberDuck> {
  RubberDuck(std::string name): name_{std::move(name)} { }

  std::string_view name() { return this->name_; }

private:
  std::string name_;
};
```

在基类的实现中，由于 `this` 中没有 `name` 方法，这个方法在派生类——也就是模板参数 `Named` 中，所以我们需要将 `this` 强转回 `Named *`，然后调用派生类 `Named` 的 `name` 方法。这当然是没问题的，因为在实例化这些方法的时候，派生类已经被完整定义了，尽管在继承的时候还没有。这里没有菱形继承，也没有虚函数和虚继承，一切都是那么美好。

因为我们编码阶段保证了 `Named` 是 `Quack<Named>` 派生类，所以也不需要 `dynamic_cast`，这里必然能 `static_cast` 成功。读者可以试着自行修改一下代码，观察如果模板参数 `Named` 填错类型，或者派生类本身没有实现 `name` 方法，会报什么错误。

另外，正如前文所说，当我们已经有了派生类类型的情况下，获取派生类方法当然是最灵活最常见的一种用途，只是不仅限于此，在基类中同样可以拿到派生类的成员变量、静态成员或方法等等，只是不常用罢了。

最后需要注意的是，模板参数不同的基类不是同一个类型，所以我们没法使用基类指针来进行动态分派。这也没关系，CRTP 本来就不是用来帮助动态分派的，如果需要做一些动态的工作，那就需要另外继承了，也不在本文的话题之内。

## 更摩登的 CRTP

C++ 在与时俱进，CRTP 也随着进化，这里简单介绍一些随着 C++ 更新给 CRTP 带来的优化。

C++11 中，我们可以利用模板参数包来简化派生类的定义。

```c++
template <template <typename ...> class Bases>
struct Duck: Bases<Duck<Bases...>>... {
  Duck(std::string name)
      : name_{std::move(name)}
      , Bases<Duck<Bases...>>{*this}... {
  }

  std::string_view name() {
    return this->name_;
  }
private:
  std::string name_;
};
```

那么使用的时候只需要如下操作，就可以按需组装出不同的鸭子来了。

```c++
// 只有名字的鸭子（只有 name 方法）
using NamedDuck = Duck<>;               
// 能游泳的哑巴鸭子（有 quack, swim, name 方法）
using SwimDumpDuck = Duck<Dump, Swim>;  
// 能叫会飞会游泳的鸭子（有 quack, fly, swim, name 方法）
using FlyDuck = Duck<Quack, Fly, Swim>; 
```

这里模板展开的逻辑，大意就是 `Duck` 接收 template template parameter，并自动从这些参数中派生，且将自身完整类型填到基类的模板参数中。因此在使用的时候只需要将没有模板参数的基类作为参数填入模板中，不需要将自己再填进去，可以很大地简化使用、批量生成新类型。

后面的 C++20 和 C++23 内容在旧的文章中没有提到，是全新的。

到了 C++20，我们有了 concept，可以放到基类中约束派生类的实现，比如要求派生类提供某个方法的实现，用来优化代码错误诊断信息，对实现者或者 IDE 本身也有更好的提示作用。不过其实因为定义 CRTP 基类的时候派生类还没有被定义，基类上的 concept 获取不到派生类的方法，所以会失败。我们只能通过将 requires 子句放到基类方法上，通过延后实例化来绕过这个限制，只是如此又显得有些鸡肋了。

```c++
template <typename T>
concept Named = requires(T value) {
  { value.name() } -> std::same_as<std::string_view>;
};

// ❌ 会直接编译失败，提示 Duck 没有 name 方法，因为继承的时候还没有方法定义
template <Named T>
struct Quack {
  void quack() {
    fmt::print("{} quacks\n", static_cast<Named *>(this)->name());
  }
};

// ✅ 可以编译通过，但是只有在使用 quack 方法时才会实例化，并在不满足时报错
template <typename T>
struct Quack {
  void quack() requires Named<T> {
    fmt::print("{} quacks\n", static_cast<Named *>(this)->name());
  }
};
```

在 C++23 中，有了所谓的<a href="https://zh.cppreference.com/w/cpp/language/member_functions#.E6.98.BE.E5.BC.8F.E5.AF.B9.E8.B1.A1.E5.BD.A2.E5.8F.82" target="_blank">显式对象形参（Deducing this）</a>，可以对代码稍作简化，去掉代码中的 `static_cast`。

```c++
template <typename T>
struct Quack {
  void quack(this T &self) requires Named<T> {
    fmt::print("{} quacks\n", self.name());
  }
};
```

## 鸭子何必是鸭子

我们把思路打开，讨论一下 CRTP 在更具体场景下的应用。除了文章开头举的例子：`std::enable_shared_from_this` 和 `boost::less_than_comparable` 家族，还有一些比较直观的实例。

比如说，当我们提供了一个抽象的 Reader 接口，实现者可以为本地文件句柄、为 TCP 套接字，甚至可以为内存的一段 Buffer 实现 Reader 接口。

一方面，我们希望 Reader 接口本身的约束非常简单，只需要实现者提供一个可中断的 `ssize_t read(uint8_t buffer[], size_t n)` 实现，以降低实现者的负担。

但是另一方面，我们又希望 Reader 接口提供的方法要丰富，以提升调用方的使用体验。比如说：带超时的读 `read_timeout` 方法、读到某个条件满足为止 `read_until` 方法、不中断读到 EOF 方法 `read_to_end`、用户提供 buffer 的和实现来 alloc 内存的（比如直接返回 `std::string` 的方法）等等等等。

当然，我们注意到上面提到的一些方法其实都依赖最基础的 `read` 方法，只要用户实现了 `read` 方法本身，我们就有办法增加一些额外的逻辑来帮助实现这些额外的方法——这便明显是 CRTP 的用武之地。

另外，我也说过我最近在写 Rust。作为同为系统级编程语言的 Rust，为什么不见人们提及 CRTP 在 Rust 中的实现呢？暂且不论 Rust 的模板（泛型参数）做不到 C++ 这种表达能力，根本原因还是在于 Rust 语言特性就已经覆盖了 CRTP 的功能，简单了解一下 trait 的 default implementation 就能明白为什么 CRTP 在 Rust 中并没有什么价值，完全没必要采用 C++ 这种委曲求全的方式。而且其实我上面所说的 Reader 接口，就是参考 Rust 的 `std::io::Read` 的。

与之类似的，之所以我们不在 C++ 中使用 Java 中泛用的一些设计模式，也是因为语言本身可能就提供了更加便利或者高效的工具，不需要去绕这些远路。将一门语言的经验强行套到另一门语言上并不一定适合。

## 后记

关于 CRTP，我之前的旧的博文中提到很多人对 CRTP 作用理解为消除动态调用，但是最后实践上却又为了能将相似的对象储存到一起，使用虚函数加继承的方法储存指针动态派发，如此一来这里其实完全没有利用到静态分派的性能优势，而且和 CRTP 的真正作用相去甚远，因此对某些文章提出了批判。不过我最近又搜了一下，那都是 2016 或者更早以前的旧文章了，我看最新的搜索结果大多是正确的，我之前批评的那个博客也已经不存在了（其实换了一个域名，现在找到了）。

而且在老的那篇文章中，我自己对于 CRTP 的理解也存在一些误区。在之前的文章中，虽然我意识到 CRTP 是可以通过派生类的实现扩展派生类，但是我还是将 CRTP 和继承本身的作用搞混了。就比如我旧文章中一直在提的 CRTP 可以“模块化”鸭子类型，通过不断继承来附加新的功能，然而其实模块化是继承提供的能力，CRTP 只是利用了继承带来的模块化特性。我在比较早（可能数年前）就意识到了这点错误，只是一直懒于重新修改我的文章改正。

而我在最近一周重新拾起我的博客，一大原因是 C++23 又出来了，我在观看 cppcon 的时候注意到了 `std::forward_like` 这个新方法，学习的时候又被引导到了 deducing this 这个特性上，过程中突然意识到这个特性可以用来简化 CRTP。又恰好碰到这段时间我 WordPress 博客挂了，然后就想索性从头开始写一下 CRTP，也好更正一下以前的错误。

不过话说回来，deducing this 的最主要作用还是用来在 Lambda 中递归，以前的 C++ Lambda 表达式可不是那么容易写递归的，这就不在这篇文章的讨论范围之内了。
