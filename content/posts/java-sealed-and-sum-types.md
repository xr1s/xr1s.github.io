+++
title = "Java Sealed 与和类型"
date = "2026-10-05"
type = "post"
tags = ["Java"]
+++

最近工作上遇到从 C++ 到 Java 的全业务代码迁移重构任务。因为数据库是 MongoDB，加上之前的 C++ 代码里也没固定 schema，全部用的 `nlohmann::json` 一路透传，前端传什么就存什么。这导致迁移遇到了非常大的阻碍。

这次重构的一个重要目标，也就包含要固定一套明确的 schema，让 MongoDB 的 Documents 能够直接映射到代码中的 model，并由 model 约束可接受的字段，避免前端传入任意字段并继续落库。

首先我们完整扫了线上表获取所有字段的可能值和类型，然后果然就遇到「同一个字段具有复数类型」的情况。就是说可能存在同一个字段名，有些是用 String 储存，有些是 BSON Document。

这种场景，我正好想到可以利用 Java 的 sealed 类型，把字段可能对应的几种结构用同一个 sealed 类型表示，这样就能保证同一个类型能有不同表示了。

我也是简单和同事介绍了一下 sealed 类型，但同事疑惑：「我们目标是希望建立类型固定（静态）、语义明确的 model，但是 sealed 类型怎么既可以是 A 又可以是 B，这还能算是静态吗？」

好问题，我也为此特地写了这篇文章给同事介绍 Java sealed 类型与和类型。

## 具体场景

举个现实的例子：

优惠券，可能有满减折扣、百分比折扣、第二杯半价、免邮费等等。

传统模式可能直接设计成一个大而全的类型，根据一个 `type` 字段判断是哪种折扣，填相应的字段，别的都留 `null`。

这样除了类型表意不明确之外还容易出错，搞出脏数据，还有莫名 Null Pointer Exception。

用 sealed 类型可以做到每个类只储存自己必须的字段，甚至可以直接保证每个类型里的字段都永远非空，直接解决 Null Reference, the Billion-Dollar Mistake。

```java
public sealed abstract class Coupon {
  private final String id;
  private final Instant expiresAt;

  // 构造器、getter、参数校验等样板代码略

  /** 满 100 减 20 */
  public static final class AmountOff extends Coupon {
    private final Money threshold;
    private final Money discount;
  }

  /** 打 8 折，最多减 50 */
  public static final class PercentOff extends Coupon {
    private final int percent;
    private final Money cap;
  }

  /** 第 2 件 5 折 */
  public static final class NthOff extends Coupon {
    private final int nth;
    private final int percent;
  }

  /** 免运费 */
  public static final class FreeShipping extends Coupon {
  }
}
```

## 代数类型

从类型理论的角度看，刚才的优惠券不只是几个 Java 类的组合，而是一个由和类型与积类型共同构成的代数类型。

代数类型可以理解为用已有类型组合出新类型，最基本的组合方式有两种：和类型表示「几种可能之一」，积类型表示「多个字段同时存在」。

### 和类型

和类型看起来好像违背了类型更静态的要求，实际上在和类型适用的场景下，它往往会使得类型更明确更健壮，可以杜绝很多非法状态，避免 God class（大而全的上帝类）。

和类型 A+B，含义是和类型的元素要么属于 A 要么属于 B、不是属于 A 就是属于 B（因此即使 A B 中有相同元素，两者也会出现在 A+B 中，所以它是求和，不是求并）。和类型本质上是把这个约定放到类型里，让编译器帮你判断有没有违反约定，让非法组合无法被构造。

和类型的实际例子：`std::variant<A, B>`、`std::optional<T>`、`std::expected<T, E>`。

笼统（不精确）地说，其实所有的枚举类型都是和类型，`bool` 也可以视为 `true` | `false` 的枚举，`int32_t` 可以视为 \\(-2^{31}\\) | \\(-2^{31}+1\\) | \\(-2^{31}+2\\) | ... | -1 | 0 | 1 | ... | \\(2^{31}-2\\) | \\(2^{31}-1\\) 的枚举，他们也都是和类型。

### 积类型

积类型 A×B，其实更好理解，它的定义就是两个类型的笛卡尔积。用数学语言表达就是 \\( A \times B = \\{ (a, b) | a \in A \land b \in B \\} \\)。  
（和类型用数学表达稍微有点麻烦，因为两个类型中相等的元素，要同时出现在和类型里，这就要给值标记属于 A 还是属于 B，可以理解为笛卡尔积一个标记然后再做并，但写起来麻烦懒得写了）。

积类型的实际例子：`std::pair<A, B>`、`std::tuple<A, B, C>`。

其实所有 struct 都是积类型，tuple 不过就是成员没有名字的匿名 struct 罢了。

所以二维坐标也是积类型，它是横坐标和纵坐标的积；复数是积类型，它是实部和虚部的积类型。

再举更实际的例子，RGB 24 位色就是一种积类型，它是 \\(R \times G \times B\\) 的积类型。其中 R 的取值是 0\~255, G 的取值是 0\~255, B 的取值是 0\~255，所以他们的积类型取值是 \\(256^3 = 2^{24} = 16777216\\)。

钟表时间时分秒是积类型，它是 \\(H \times M \times S\\) 的积类型，所有可能的组合是 00:00:00 ~ 23:59:59 的取值，共 86400 种，恰好是一天的秒数。

### 局限性

但是与之相对的，年月日就无法用积类型精准表达了，因为日的合法取值范围依赖于年和月的具体值。如果统一用 1~31 的范围来做乘法，会允许构造出很多非法日期，比如 2026-02-29，2026-09-31 等等。这样我们就只能依赖运行时检查来规避错误的日期。

这是题外话了，普通的积类型只能表达字段之间的**组合**关系，无法表达字段之间的**依赖**关系。如果希望把这种跨字段约束直接编码进类型系统，我们就需要**依赖类型 Dependent Types**。但 Java 甚至 Haskell 里都无法**完整**表达这种字段之间相互依赖、字段依赖值的概念。只能使用较宽泛类型，并运行时增加校验以禁止构造非法值。依赖类型在主流语言中并不常见，只出现在 Idris、Agda 等基于类型论的函数式语言，以及 Lean、Coq 等证明助手中。

## 更多场景

重新回到业务建模，看看这种把多种可能性写进类型的方式还可以应用在哪些场景中。

### 互斥属性

如果一个对象存在多种彼此互斥的形态，可以用 sealed 类型表示它们的共同抽象，并为每种形态定义独立的结构。

除了上面提到的优惠券，还有一些实际业务中经常用到的场景：

| 场景     | 分类                                                                                  |
| -------- | ------------------------------------------------------------------------------------- |
| 定时任务 | `Cron(expr)` / `FixedRate(interval)` / `OneShot(runAt)`                               |
| API 响应 | `Success<T>(data)` / `Failure(code, message)`                                         |
| 通知渠道 | `Email(addr)` / `SMS(phone)` / `Push(deviceToken, platform)` / `Webhook(url, secret)` |
| 支付方式 | `CreditCard(no, expiry, cvv)` / `BankTransfer(iban)` / `Alipay(buyerId)` / `COD`      |
| 配送方式 | `Express(address)` / `SelfPickup(storeId)` / `Locker(lockerId)`                       |
| 登录凭证 | `Password(user, hash)` / `Otp(phone, code)` / `OAuth(provider, idToken)`              |
| IM 消息  | `Text(content)` / `Image(url, w, h)` / `Voice(url, duration)` / `Location(lat, lng)`  |
| 广告素材 | `SingleImage` / `Carousel(images)` / `Video(url, duration, cover)`                    |
| 存储配置 | `Local(path)` / `S3(bucket, region)` / `Gcs(bucket, project)`                         |

### 状态机

和类型其实还有一个极其适用且常见的场景：状态机。

```java
class Order {
  OrderStatus status;          // 6 个枚举值
  Instant paidAt;              // 未支付时 null
  String transactionId;        // 未支付时 null
  String carrier, trackingNo;  // 未发货时 null
  Instant shippedAt;           // ...
  String cancelReason, refundId;
}
```

这样项目代码里只能留一堆 `if (status == SHIPPED) { ...getTrackingNo()... }`，一旦搞错状态就 NPE，代码健壮全靠文档约定。

如果换成和类型，就把约定落到了类型里、代码里，由编译器保证：

```java
public sealed abstract class Order {
  private final String id;

  // 构造器、getter、参数校验等样板代码略

  public static final class Draft extends Order {
    private final List<Item> items;
  }

  public static final class AwaitingPayment extends Order {
    private final Money amount;
    private final Instant expiresAt;
  }

  public static final class Paid extends Order {
    private final String transactionId;
    private final Instant paidAt;
  }

  public static final class Shipped extends Order {
    private final String carrier;
    private final String trackingNo;
    private final Instant at;
  }

  public static final class Completed extends Order {
    private final Instant completedAt;
  }

  public static final class Cancelled extends Order {
    private final String reason;
    private final Optional<String> refundId;
  }
}
```

把状态转移写成**返回下一个状态类型的方法**，非法转移就直接编译不过了：

```java
// 在 AwaitingPayment 中：
Paid pay(String txId) { return new Paid(getId(), txId, Instant.now()); }

// 在 Paid 中：
Shipped ship(String carrier, String no) { ... }
```

### 穷尽性检查

这是和类型带来的最重要的心智优化，也是 sealed 类型区别于普通子类型派生的重大区别。因为 sealed 类型的直接子类型是有限的，那么编译器就能分析你对 sealed 类型的枚举是否完整。

```java
Money discountOf(Coupon c, Cart cart) {
  return switch (c) {  // 注意：没有 default 也能编译通过
    case AmountOff c ->
        cart.subtotal().gte(c.getThreshold())? c.getDiscount() : Money.ZERO;
    case PercentOff c ->
        Money.min(cart.subtotal().times(c.getPercent()).divide(100), c.getCap());
    case NthOff n       -> n.applyTo(cart);
    case FreeShipping f -> cart.shippingFee();
  };
}
```

如果产品新增一个功能，在上面的优惠券中要新增一种「赠品券」。那么我们只要新增一个子类，所有没处理它的 `switch` 全部编译报错，相当于编译器直接输出一份 TODO 清单。

而在 `enum status` + 可空字段的写法，或者普通子类派生的写法里，加一个枚举或者新的继承，但是遗漏了某一处判断的情况下，代码照样能编译通过上线，然后在某个角落静默地算错价格。等着被客诉吧你。

利用编译器自带穷尽性检查，我们可以把很多运行时的逻辑错误提前到编译期发现。这也是我个人很喜欢类型系统强大的语言的原因。

## 结语

回到最开始的一个问题：

> 和类型看起来好像违背了类型更静态的要求，实际上在和类型适用的场景下，它往往会使得类型更明确更健壮

这个原因就是：

我们在很多适用和类型的场景下，实际上在用积类型（大而全的 God class，靠 null 来杜绝非法状态）。

把乘法改成加法，状态数自然少了，非法状态就少了。定义域更紧凑，类型更健壮，语义也更明确。

## 后记

这一篇就当是康复训练了。本身也不是什么很困难的概念。本来就只是写给同事看的概念推销。

已经很久没写博客，也是怠惰了。

阻止我写博客还有一个原因是每次想写就想把细节挖很深，希望一点纰漏都不出，结果反而导致我每次写博客都非常花时间。

这其实是好事，每次写完之后，自己能感觉到在反复的思考和试验中，对知识的理解更深刻了。

但反过来又因为太耗时间，一片博文可能花费数周甚至一个多月才能完成。因为各种原因，不太愿意花费这么多的时间去做这些事情。

现在有了 vibe coding，感觉也可以辅助写作，很多知识、思路、纰漏、实验都可以让 agents 帮我快速验证，可以省下很多时间。

不过我还是保证博文本身是一个字一个字手敲，文章本身就不要 vibe 出来了。一方面我很讨厌 LLM 味很浓的文字；另一方面，我写博客本身也是为了自己能更深刻地理解知识，如果 LLM 架空大脑，那写作就真没啥意义了。

LLM 还是用来应付工作比较好。

感觉后续也可以单开一篇文章来讨论一下我对于 vibe coding 和「古法编程」的看法和思考了。时代的进步，还是很有意思的。另一方面，我特地捡起博客来坚持写文，也可以说明我对于 vibe coding 的态度，加上目前我仍然坚持手搓代码（虽然确实很少了，不知道还有没有 5%）、坚持 Review 所有 LLM 产出代码。

这个小站也可以多一点思考，而不是纯技术。呵呵，又是一个坑。
