---
title: Lambda演算介绍
template: default
---

<a href="https://sylambdacode.github.io/2025/07/19/LambdaCalculusIntroduction.html">Lambda演算介绍</a> © 2025 by <a href="https://sylambdacode.github.io/">sylambdacode</a> is licensed under <a href="https://creativecommons.org/licenses/by-nc/4.0/">CC BY-NC 4.0</a><img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;">


# 前言
本片文章旨在帮助有兴趣了解Lambda演算，但是没有相关数学基础的读者快速理解Lambda演算的基本思想。为了能够更好的满足这种需求，本篇文章将是非严谨的、启发式的。请那些希望学习严谨Lambda演算的读者阅读相关论文与书籍。


# 这是什么？
Lambda演算（Lambda Calculus）是数学家阿隆佐·邱奇（Alonzo Church）在20世纪30年代提出的一种形式系统，该系统是数学家阿隆佐·邱奇进行数学基础研究时提出的数学工具。该系统在后来被证明是一种具有与图灵机相同的计算能力的计算模型——即作为一种计算模型，Lambda演算是图灵完备的。Lambda演算通过简洁的几条规则，清晰的表述了函数的定义、函数的应用、变量等基本思想，对计算机科学领域的发展产生了重要影响。现在人们所熟知的各种函数式编程语言（例如Haskell、ML等）都是以Lambda演算作为理论基础的，并且许多非函数式编程语言也积极汲取了Lambda演算的思想，各类语言中的匿名函数便是典型的代表。


# 符号替换游戏？

要理解Lambda演算究竟在做什么，不妨设想这样一个符号替换游戏。

## 小菜一碟

> 声明：下面的符号序列中@是可替换的
>
> > !**@**#$%

> 替换：满足声明要求，使用&进行替换，我们得到：
>
> !**&**#$%

---

> 声明：下面的符号需求中😑是可替换的
>
> 😀😁😑😯😌

> 替换：满足声明要求，使用❤️进行替换，我们得到：
>
> 😀😁❤️😯😌

## 这也可以套娃？
> 声明：下面的符号序列中@是可替换的
>
> > 声明：下面的符号序列中%是可替换的
> >
> > > !**@**#$**%**

> 第一次替换：满足声明要求，使用&进行替换，我们得到：
>
> 声明：下面的符号序列中%是可替换的
>
> > !**&**#$**%**

> 第二次替换：继续满足声明要求，使用?进行替换，我们得到：
>
> !**&**#$**?**

## 简化
试着简化一下上面的繁琐描述：

> 声明：下面的符号序列中c是可替换的
>
> > ab**c**df

被简化为

> λc.abcdf

---
合在一起如何简化?

> 声明：下面的符号序列中c是可替换的
>
> > ab**c**df

> 替换：满足声明要求，使用K进行替换，我们得到：
>
> ab**K**df

被简化为

> (λc.abcdf)K = abKdf

---

相信你已经能够理解下面的简化表达代表了什么：

> (λc.(\f.abcdf))KL = (\f.abKdf)L = abKdL

正是刚刚所说的“套娃”！

# 重要概念
## 基础概念
上文用了很大的篇幅介绍了一个符号替换游戏，目的在于帮助各位读者理解Lambda演算中几个核心概念：

概念 | 举例 | 解释
---|---|---
变量 | a   | 字符是一个变量，该变量通常并不代表什么，仅仅是一个名字与占位符罢了
抽象化（函数定义）| λc.abcdf | 这意味着abcdf中的c是可以被替换的
函数应用 | (λc.abcdf)K | 这意味着使用字符K替换前面函数定义中的字符c，可以得到结果abKdf
函数应用 | ab | 这表明a可以代表一个函数，若提前规定a是λc.ccc，那么可以得到ab=(λc.ccc)b=bbb

通常情况下，Lambda演算中任何变量都没有特定的含义，这些变量仅仅是一个名字，或者是某一个占位符，并在函数应用时被替换。例如`(λa.aab)`，“λ”后面的a代表一个名字，标明a是可以被替换的，而“.”后面的a和b都代表占位符，并可能在函数应用的时候被替换。

若进行函数应用，例如`(λa.(aa)b)(λa.a)`，可以得到`((λa.a)(λa.a))b`（将`(λa.a)`作为一个整体替换前面的`aa`）。

可以发现结果`((λa.a)(λa.a))b`依然存在函数应用，比如`(λa.a)(λa.a)`部分，计算这部分可以得到结果`(λa.a)`，综合起来结果就是`(λa.a)b`。

最后计算`(λa.a)b`得到结果`b`。

**为了简化书写，我们通常规定函数应用是左结合的，例如`abcdf`的含义实际上为`(((ab)c)d)f`。因此，上面的例子`(λa.(aa)b)(λa.a)`可以书写为`(λa.aab)(λa.a)`。**

## 柯里化
柯里化是一个非常重要的概念，还记得上面所提到的“套娃”吗？以`λa.(λb.aab)`举例，由于柯里化思想，我们可以将上面的表达式简写为`λab.aab`。简单来说就是将多层单参数函数的嵌套简写为一层多参数函数。

使用JavaScript中的匿名函数举例可能会更容易理解：
```JavaScript
/**
 * 简写之前
 */
function (a) {
    return function (b) {
        return b(a(a)) // aab = (aa)b
    }
}
/**
 * 简写之后
 */
function (a, b) {
    return b(a(a))
}
```

**可以看出柯里化是右结合的，需要注意的是柯里化仅仅是为了书写上的便利，并没有改变原表达式的语义。例如`(λab.aab)KL=(λa.(λb.aab)KL)=(λb.KKb)L=KKL`。**



