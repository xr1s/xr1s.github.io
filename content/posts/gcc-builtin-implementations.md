+++
title = "GCC 部分内建位运算函数轮子"
date = "2022-10-20"
type = "post"
tags = ["C/C++"]
aliases = ["/2018/08/23/gcc-builtin-implementation/"]
+++

某个空虚寂寞的晚上，随便造点轮子，只是常用的二进制位运算的函数而已。实现不一定是常数最优的，可读性优先。

裸的打表、分块打表、不到 log 级别的表不写，不屑。

## Counting Trailing Zeros

`__builtin_ctz`，返回从最低位开始的连续的 0 的个数；如果传入 0 则行为未定义。

`_BitScanForward` ，Visual Studio 中的内建函数，等价于 GCC 的 `__builtin_ctz`。

### 朴素

```c
int ctz64(uint64_t x) {
  for (int i = 0; i != 64; ++i)
    if (x >> i & 1) return i;
  return 0;
}
```
### 二分

```c
int ctz64(uint64_t x) {
  int r = 63;
  x &= ~x + 1;
  if (x & 0x00000000FFFFFFFF) r -= 32;
  if (x & 0x0000FFFF0000FFFF) r -= 16;
  if (x & 0x00FF00FF00FF00FF) r -= 8;
  if (x & 0x0F0F0F0F0F0F0F0F) r -= 4;
  if (x & 0x3333333333333333) r -= 2;
  if (x & 0x5555555555555555) r -= 1;
  return r;
}
```

这里用到了一个位运算技巧 `x & ~x + 1` 可以直接提取出 x 的*最低设置位*（Lowest Set Bit）。这是一个比较巧妙的方法，在 CTZ 中被广泛使用。它在树状数组中被也应用于定位前继节点（向前走，求前缀和）或者父节点（向后走，向上更新状态）。一个小细节是在补码表示的情况下，`~x + 1` 和 `-x` 是等价的，本文用 `~x + 1` 更清晰。

后面几行就是二分判断*最低设置位*在二进制表达的哪个位置。第 4 行判断是前半截还是后半截，第 5 行将 64 位二进制数分成两个 32 位，判断在 32 位的前半截还是后半截，依次类推。顺序反过来或者从 0 开始往上加都是可以的。

### IEEE-754 性质

```c
int ctz64(uint64_t x) {
  union {
    double f;
    uint64_t i;
  } v = {
    .f = x & ~x + 1,
  };
  return (v.i >> 52) - 1023;
}
```

利用了 IEEE-754 中定义的 binary 格式，在遇到 2 的幂次时，它指数域的值减去置偏值就是 2 右上那个指数，也就是我们需要求的值。上面的代码里没有管符号位，因为我们用的是 `uint64_t`，如果不是无符号整数可能要针对负数进行处理一下。

### de Bruijn 序列性质（映射查表）

```c
int ctz64(uint64_t x) {
  static const int bit_perm[64] = {
     0,  1,  2,  7,  3, 13,  8, 19,  4, 25, 14, 28,  9, 34, 20, 40,
     5, 17, 26, 38, 15, 46, 29, 48, 10, 31, 35, 54, 21, 50, 41, 57,
    63,  6, 12, 18, 24, 27, 33, 39, 16, 37, 45, 47, 30, 53, 49, 56,
    62, 11, 23, 32, 36, 44, 52, 55, 61, 22, 43, 51, 60, 42, 59, 58,
  };
  return bit_perm[0x218A392CD3D5DBFUL * (x & ~x + 1) >> 58];
}
```

这种方法本质上是找到了一个巧妙的映射函数，利用了 de Bruijn 序列的一个性质将 64 个 2 的幂 \\(\displaystyle\left\\{2^k\middle|k\in[0,64)\cap\mathbb Z\right\\}\\) 双射回 \\([0,64)\cap\mathbb Z\\) 区间内，结合之前的 `x & x + 1` 技巧，我们就可以直接得到答案。

代码中的魔数 `0x218A392CD3D5DBFUL` 就是 B(2, 6) 的 de Bruijn 序列，如果将它转化为二进制并且头尾相接成环，使用 6 位的窗口滑动过去，可以发现每个窗口都不同。所以将它左移 0~63 位，每次都能拿到不同的 6 位数字（无论取哪部分的 6 位）。这样的话只需要获取到 x 的*最低设置位*，因为它本身是一个 2 的幂，它乘以任何数都等价于左移一个 CTZ。又因为是双射，所以肯定是覆盖满值域定义域的，只需要把 0~63 对应 2 的几次幂全部求出来，存到数组里再映射就行。

关于 de Bruijn 具体可以参考<a href="https://en.wikipedia.org/wiki/De_Bruijn_sequence" target="_blank">维基百科 de Bruijn sequence</a>，我这里将生成 128 位 de Bruijn 序列和它的双射表的代码放上来，64 位改个参数就行：<a href="https://gist.github.com/xr1s/ee7ece7904efc6f4daecd691c598bf3f" target="_blank">GitHub Gists</a>。

## Find First Set

`__builtin_ffs` ，返回最低的非 0 位的下标，下标从 1 开始计数；如果传入 0 则返回 0 。

除 0 以外，满足恒等式 `__builtin_ffs(x) == __builtin_ctz(x) + 1` ，懒得罗嗦了。

```c
int ffs64(uint64_t x) {
  if (x == 0) return 0;
  return ctz64(x) + 1;
}
```

## Count Leading Zeros

`__builtin_clz`，返回从最高位开始连续的 0 的个数；当传入 0 则行为未定义。

`_BitScanReverse`，Visual Studio 中的内建函数，等价于 `__builtin_clz`。

一个数学特性是 \\(\mathrm{clz}\ x+\lfloor\log_2 x\rfloor+1=类型位宽\\)。

### 朴素

```c
int clz64(uint64_t x) {
  for (int i = 0; i != 64; ++i)
    if (x >> 63 - i & 1) return i;
}
```

### 二分

```c
int clz64(uint64_t x) {
  int r = 0;
  if ((x & 0xFFFFFFFF00000000) == 0) r += 32, x <<= 32;
  if ((x & 0xFFFF000000000000) == 0) r += 16, x <<= 16;
  if ((x & 0xFF00000000000000) == 0) r += 8,  x <<= 8;
  if ((x & 0xF000000000000000) == 0) r += 4,  x <<= 4;
  if ((x & 0xC000000000000000) == 0) r += 2,  x <<= 2;
  if ((x & 0x8000000000000000) == 0) r += 1,  x <<= 1;
  return r;
}
```

这个还是比较好理解的，不多解释了。

### IEEE-754 性质

因为 IEEE-754 会自动调整成科学计数法，所以直接将数字转为浮点数就可以使用了。可能需要担心的就是非规格化浮点数，但是我们处理的都是整数，碰不到。

```c
int clz64(uint64_t x) {
  union {
    double f;
    uint64_t i;
  } v = {
    .f = x,
  };
  return 1086 - (v.i >> 52);
}
```

首先也是拿到指数域，减去置偏值，得到的是 \\(\left\lfloor\log_2 x\right\rfloor\\)，我们要求前缀 0 个数，还要用 63 减一下，也就是 `63 - ((v.i >> 52) - 1023)` 化简后就是上面的。

同样这里也没有考虑负数的符号位，如果有负数直接返回 0 好了。

### de Bruijn 序列性质（映射查表）

```c
int clz64(uint64_t x) {
  static const int bit_perm[64] = {
    63, 62, 61, 56, 60, 50, 55, 44, 59, 38, 49, 35, 54, 29, 43, 23,
    58, 46, 37, 25, 48, 17, 34, 15, 53, 32, 28,  9, 42, 13, 22,  6,
     0, 57, 51, 45, 39, 36, 30, 24, 47, 26, 18, 16, 33, 10, 14,  7,
     1, 52, 40, 31, 27, 19, 11,  8,  2, 41, 20, 12,  3, 21,  4,  5,
  };
  x |= x >> 32;
  x |= x >> 16;
  x |= x >> 8;
  x |= x >> 4;
  x |= x >> 2;
  x |= x >> 1;
  x ^= x >> 1;
  return bit_perm[0x218A392CD3D5DBF * x >> 58];
}
```

和 ctz 本质是一样的，先单独拎出*最高设置位*，中间那一段 `|=` 就是干这个的，逻辑是把*最高设置位*右边填满 1，然后减一下，就只剩*最高设置位*了。

其它的逻辑和 de Bruijn 是一样的，代码生成改一下之前的脚本就可以了，这里就略过了。

## Count Leading Redundant Sign Bits

`__builtin_clrsb`，返回从<i>最高位（MSB）</i>开始和符号位相同的位数。

```c
int clrsb64(int64_t x) {
  if (x == 0 || x == -1) return 63;
  return clz64(x ^ x >> 63);
}
```

很简单，利用有符号整数的右移运算符会用符号填充高位的性质，如果本来是负数那么 `x >> 63` 就是 -1，如果本来是非负数那就是 0。然后和原数异或一次，如果原数是负数那所有 1 被翻转为 0，如果原数是正数那保持不变。最后求一次 CLZ 即可。

## Population Count

`__builtin_popcount` 统计二进制位中 1 的个数。

比较常用，研究比较多，骚优化不少。

### 朴素

```c
int popcount64(uint64_t x) {
  int r = 0;
  do r += x & 1;
  while (x >>= 1);
  return r;
}
```

### 一个优化

```c
int popcount64(uint64_t x) {
  int r = 0;
  for (; x; x &= x - 1) ++r;
  return r;
}
```

`x & x - 1` 的作用是将*最低设置位*置 0。很好理解，`x - 1` 在遇到最低位是 0 的时候会不停向上借位，直到碰到一个 1 也就是*最低设置位*为止。最后结果*最低设置位*以及更低位都被翻转，然后和原数按位与一下就把*最低设置位*置 0 了。

这样迭代次数和参数二进制表达中 1 的个数相同，稍微有些优化。

### 二分

```c
int popcount64(uint64_t x) {
  x = (x & 0x5555555555555555) + (x >> 1  & 0x5555555555555555);
  x = (x & 0x3333333333333333) + (x >> 2  & 0x3333333333333333);
  x = (x & 0x0F0F0F0F0F0F0F0F) + (x >> 4  & 0x0F0F0F0F0F0F0F0F);
  x = (x & 0x00FF00FF00FF00FF) + (x >> 8  & 0x00FF00FF00FF00FF);
  x = (x & 0x0000FFFF0000FFFF) + (x >> 16 & 0x0000FFFF0000FFFF);
  x = (x & 0x00000000FFFFFFFF) + (x >> 32 & 0x00000000FFFFFFFF);
  return x;
}
```

思路是这样，先把二进制按 2 位分组，每个 0x5 即 0b0101 就是分组结果，然后将这每两位中的 0 部分和 1 部分相加。再将加起来的结果按 4 位分组，每个 0x3 即 0b0011 就是分组结果，再将这 4 位中两位 0 部分代表数字和两位 1 部分代表数字相加。依此类推。

或者可以这么理解，先将输入看作 4 进制数字，然后将高位和低位相加；再将相加的结果看作 16 进制数字，然后将高位和低位相加；再将相加的结果看作 256 进制数字，然后将高位和低位相加；依次类推。之所以这么理解，是为了下面的一个优化：

```c
int popcount64(uint64_t x) {
  x = (x & 0x5555555555555555) + (x >> 1 & 0x5555555555555555);
  x = (x & 0x3333333333333333) + (x >> 2 & 0x3333333333333333);
  x = (x & 0x0F0F0F0F0F0F0F0F) + (x >> 4 & 0x0F0F0F0F0F0F0F0F);
  return x % 255;
}
```

这个优化是在第 4 行这一步过后直接 mod 255 就是结果。因为 \\(m^k\equiv1\pmod{m-1}\\) ，所以 \\(m\\) 进制的数字 \\(\overline{a_na_{n-1}\cdots a_3a_2a_1}\\) 有 \\(\sum\limits_{1\le k\le n}a_km^k\equiv\sum\limits_{1\le k\le n}a_k\pmod{m-1}\\) 就是按位和。上面已经分析过第三步之后就已经拿到一个 256 进制了，所以直接 mod 255 就是答案。而更上一步则需要 mod 16，这肯定会有问题，因为 64 bits 的 popcount 最高是 64，mod 16 显然会出错。

类似的，在算到第四轮的时候完全不需要维护更高位的部分，因为单个 16 进制已经可以保存下当前完整的 1 计数数了。只需要直接低位部分加上高位部分，最后截取低 8 位即可。该变种仍可继续优化，最后求和的部分可以一个乘法达成：

```c
int popcount64(uint64_t x) {
  x -= x >> 1 & 0x5555555555555555;
  x = (x & 0x3333333333333333) + (x >> 2 & 0x3333333333333333);
  x = x + (x >> 4) & 0x0F0F0F0F0F0F0F0F;
  return x * 0x0101010101010101 >> 56;
}
```

第 2 行因为是按照 2 bits 分组的，所以它的含义，我们只需要枚举 0~4 的映射就可以了，可以看出来和原来是等价的。这么做是为了减少一次运算。

| from | into |
|:----:|:----:|
|  00  |   0  |
|  01  |   1  |
|  10  |   1  |
|  11  |   2  |

第三行没变，第四行就是去掉了维护更高位部分的逻辑，上面说过 16 进制已经存得下了。而最后一行乘以 0x0101010101010101 是将每个 0xF 分组部分全部累加到最高的 8 位中，然后直接舍弃低位截取最高字节（右移 56 位）即可。

## Parity

`__builtin_parity` 就是求 `__builtin_popcount` 的奇偶性。

### 偷懒

```c
int parity64(uint64_t x) {
  return popcount64(x) & 1;
}
```

### 二分

```c
int parity64(uint64_t x) {
  x ^= x >> 1;
  x ^= x >> 2;
  x ^= x >> 4;
  x ^= x >> 8;
  x ^= x >> 16;
  x ^= x >> 32;
  return x & 1;
}
```

一个优化是部分打表，在 x 的值比 16 小了之后直接用表即可，由于 parity 只会返回 0 或 1，因此可以压位。

## 参考资料

* <a href="http://graphics.stanford.edu/~seander/bithacks.html" target="_blank" rel="noreferrer noopener">Bit Twiddling Hacks</a>
* <a href="https://en.wikipedia.org/wiki/Find_first_set" target="_blank" rel="noreferrer noopener">Find First Set – Wikipedia</a>
