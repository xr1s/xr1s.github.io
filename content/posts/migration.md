+++
title = "博客迁移"
date = "2022-10-15"
type = "post"
+++

之前那个 xr1s.me 大概是 OOM 了挂了，本来就是加了 Swap 才能跑的，其实重启一下应该就好了。

但是因为工作后也不怎么写博客，服务器都直接懒得维护了，现在打算就静态简单搞搞吧。目前用的是 Hugo，没有特别的理由，因为看到了个喜欢的主题——众所周知，我喜欢衬线体，显得我好像很有文化一样。

发里浮哨。

旧的博文都已经备份了，可能自己看一遍有没有大的问题再发出来，不过评论就没了。

目前业余时间主要在折腾 Rust，以后可能 Rust 文章会有一些。不过第一篇正经的文章可能会把 C++23 后的 CRTP 写一写。

后面测试一下代码和数学公式。

```python
import asyncio

import httpx


async def main():
    async with httpx.AsyncClient() as client:
      await client.get("https://xr1s.me")


if __name__ == "__main__":
    asyncio.run(main())
```

| 名称 | 积分形式 | 微分形式 |
|-|-|-|
| 高斯定律 | \\(\displaystyle\oiint_{\partial\Omega}\mathbf E\cdot\mathrm d\mathbf S=\frac1{\varepsilon_0}\iiint_\Omega\rho\ \mathrm dV\\) | \\(\displaystyle\nabla\cdot\mathbf E=\frac\rho{\varepsilon_0}\\) |
| 高斯磁定律 | \\(\displaystyle\oiint_{\partial\Omega}\mathbf B\cdot\mathrm d\mathbf S=0\\) | \\(\displaystyle\nabla\cdot\mathbf B=0\\) |
| 法拉第电磁感应定律 | \\(\displaystyle\oint_{\partial\Sigma}\mathbf E\cdot\mathrm d\mathbf\ell=-\frac{\mathrm d}{\mathrm dt}\iint_\Sigma\mathbf B\cdot\mathrm d\mathbf S\\) | \\(\displaystyle\nabla\times\mathbf E=-\frac{\partial\mathbf B}{\partial t}\\) |
| 麦克斯韦-安培定律 | \\({\displaystyle\oint_{\partial\Sigma}\mathbf B\cdot\mathrm d\mathbf\ell=\mu_0\left(\iint_\Sigma\mathbf J\cdot\mathrm d\mathbf S+\varepsilon_0\frac{\mathrm d}{\mathrm dt}\iint_\Sigma\mathbf E\cdot\mathrm d\mathbf S\right)}\\) | \\({\displaystyle\nabla\times\mathbf B=\mu_0\left(\mathbf J+\varepsilon_0\frac{\partial\mathbf E}{\partial t}\right)}\\) |
