---
title: 多态的多态
author: 孙耀珠
tags: 编程语言
ruby: true
---

尤记得本科的面向对象编程课有一道经典例题：C++ 的多态性体现在何处？标准答案着眼于 C++ 的虚函数解释了动态派发的机制。自那以后的很长一段时间里，我对多态的认识就固化在了子类型多态上；直到博士期间开始搞编程语言的研究，才发现学术界对多态的定义远不局限于此。

英语里有个单词叫做 [autological](https://en.wikipedia.org/wiki/Autological_word)，是说一个词可以形容其本身，比方说「名词」本身就是个名词，「阳平」这两个字的声调本身就是阳平，而「多态」本身也很多态。编程语言中最常见的多态有四种：**特设多态**、**参数多态**、**子类型多态**和**行多态**。通俗来讲，只要是为不同类型的数据或操作提供了相同的名字就可以叫多态。

本文只是一篇科普性质的文章，所以基本上只以实际编程语言为例，不会用任何形式化的语言来描述这些类型系统。如果希望深入学习相关的理论知识，建议阅读 [TAPL](https://www.cis.upenn.edu/~bcpierce/tapl/) 或者 [PFPL](https://www.cs.cmu.edu/~rwh/pfpl/)；Giuseppe Castagna 在韩国 [SIGPL Summer School 2019](https://sigpl.or.kr/school/2019s/) 的特邀讲座也很值得一看。

<!--more-->

- 目录
{:toc}

## 特设多态

「[特设多态]{ad-hoc polymorphism}」其实就是我们常说的「[重载]{overload}」，日语更通俗地叫它「[多重定義]{オーバーロード}」。称其为特设是因为这种多态并不像全称量化一样适用于所有类型，而是手动为某些特定类型提供不同的实现。

举例来说，C 语言不支持函数重载，于是绝对值函数搞出了 `abs`（整数）、`fabs`（浮点数）等不同的名字。多年后，支持函数重载的 C++ 则为同一个名字 `abs` 提供了各种参数类型的版本：

```cpp
int         abs(int         x);  // __Z3absi
long        abs(long        x);  // __Z3absl
long long   abs(long long   x);  // __Z3absx
float       abs(float       x);  // __Z3absf
double      abs(double      x);  // __Z3absd
long double abs(long double x);  // __Z3abse
```

在实际的 C++ 编译器实现中，为了能在链接时对这些重载函数进行区分，会有一个「[命名粉碎]{name mangling}」的环节将它们重新命名为独一无二的符号，就像每行行末注释里的一样。

C++ 中还有大量的隐式类型转换，这也可以被视为特设的「[强制多态]{coercion polymorphism}」。这两种特设多态的结合给函数的静态派发带来了巨大的复杂度，以至于 C++ 有一份相当长的标准被称为「[[重载决议]{overload resolution}](https://en.cppreference.com/w/cpp/language/overload_resolution)」。通常来说，隐式类型转换是弱类型的标志，它在类型方面带来了相当大的不可预测性。

那么特设多态在函数式编程语言中的支持如何呢？ML 系的语言都不支持函数重载，其中最具代表性的 OCaml 甚至完全不支持运算符重载，导致整数加法 `+` 和浮点数加法 `+.` 等都不是同一个符号。还好 SML 和 F# 为运算符重载开了口子：SML 跟 Java 一样，预先重载了一些内建运算符，但不允许用户进行重载；而 F# 完全向用户开放了运算符重载。

### 类型类

Haskell 语言对于特设多态的解决方案则是「[类型类]{type class}」，这个方案最早由 Philip Wadler 和他的学生 Stephen Blott 在 [POPL 1989](https://doi.org/10.1145/75277.75283) 的论文中提出，可以视为对 SML `eqtype` 的扩展。`eqtype` 表示一个类型支持相等比较，而类型类将其推广到了任意操作。譬如我们可以定义一类数值类型，这类类型支持前面提到的 `abs` 函数：

```haskell
class Num a where
  abs :: a -> a
  ……
instance Num Int where
  abs n = if n `geInt` 0 then n else negate n
  ……
```

这里 `class` 定义了 `Num` 类型类支持的函数及其类型，`instance` 则表明整数类型是 `Num` 类型类的实例，并提供了绝对值函数的具体定义。就像函数重载一样，我们也可以为其他类型定义这个类型类的实例。有了这种通用的解决方案，SML 中的 `eqtype` 相当于是 Haskell 中的一个特例——`Eq` 类型类。如今，Rust 的 [`trait`](https://doc.rust-lang.org/book/ch10-02-traits.html) 和 C++20 的 [`concept`](https://en.cppreference.com/w/cpp/language/constraints) 都以各自的方式实现了类型类的功能，越来越多的语言设计者参悟了如何让特设多态不那么特设。

## 参数多态

「[参数多态]{parametric polymorphism}」在函数式编程中简称多态，在面向对象编程中又称「[泛型]{generics}」。之所以叫参数多态是因为它引入了类型参数，譬如允许我们在函数定义中使用类型变量，而不必为各种具体的类型定义不同的版本。譬如以 Haskell 语言为例，我们可以为所有类型定义一个统一的恒等函数：

```haskell
id :: a -> a
id x = x

id True  {- True :: Bool -}
id 'a'   {- 'a'  :: Char -}
```

这里 Haskell 标准语法省略了类型参数的引入和消去，如果我们打开 GHC 的 [ExplicitForAll](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/exts/explicit_forall.html) 和 [TypeApplications](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/exts/type_applications.html) 语言扩展，我们可以看得更清楚一点：

```haskell
id :: forall a. a -> a
id x = x

id @Bool True  {- True :: Bool -}
id @Char 'a'   {- 'a'  :: Char -}
```

第一行类型声明中的 `forall a` 引入了一个隐式的类型参数，之所以用这个关键字是因为它对应于逻辑中的全称量化。不过有趣的是，与之对偶的存在量化在 Haskell 中复用了 `forall` 关键字，详见 GHC 的语言扩展 [ExistentialQuantification](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/exts/existential_quantification.html)。以 `@` 开头的就是显式指定的类型参数，当然，就算我们不写它们也能自动推断出来。

### Hindley–Milner

看到上文中的 Haskell 代码，大家可能会产生一个疑问：为什么类型参数默认是隐式的？要回答这个问题，就不得不提 Haskell 所采用的 Hindley–Milner 类型系统。HM 类型系统的最大好处是已有经过证明的算法（譬如 Algorithm W）可以做完整的类型推断，所以我们无须显式标注任何类型。不过 HM 类型系统也为此引入了对多态的两大限制：

- Rank-1：函数类型里面不可以嵌套多态类型。
- 直谓性：类型变量不可以实例化为多态类型。

鉴于第一条 Rank-1 限制，类型变量只可能是顶层的 `forall` 引入的，Haskell 就干脆默认隐式声明了。因为函数参数不能是多态类型，所以 HM 类型系统依赖于 `let` 表达式来引入多态类型的变量。也正因如此，HM 类型系统中的 `let x = e1 in e2` 和 `(\x -> e2) e1` 并不等价，因为前者 `x` 可以是多态类型，而后者不行。在类型推断中，`let` 会让 `e2` 每一处使用 `x` 的地方都各自独立地对类型变量进行实例化，这样就达到了多态的效果。不止是 Haskell，其他 ML 系语言（包括 SML、OCaml、F# 等）也都以 HM 类型系统为基础。

### 值限制

提到 HM 类型系统，另一个绕不过去的话题是 ML 系语言中臭名昭著的「[值限制]{value restriction}」。因为 `let` 的多态规则本质上是为变量的类型创建了多个实例，但如果这些实例因为副作用而有所联系时，就能一下子摧毁 ML 引以为傲的类型安全。这里以 OCaml 为例：

```ocaml
let r : 'a option ref = ref None
r := Some 48  (* r : int option ref *)
match !r with (* r : string option ref *)
| None     -> ""
| Some str -> "HKG" ^ str
```

第二行和第三行的 `r` 指向相同的内存地址，但 ML 为它们的类型建立了不同的实例，导致第三行能够通过类型检查，却有可能触发运行时错误。为了让多态不破坏命令式代码的类型安全，ML 系语言目前普遍采用的解决方案是 Andrew Wright 在 1995 年提出的值限制：只有 `let x =` 右侧在语法上是值的时候才能拥有多态类型，否则一律按单态处理（OCaml 的实现会创建一个弱类型变量 `'_weak` 根据未来的信息来推断这个未知的单态类型）。这种限制简单粗暴地认为任何不是值的表达式都可能带有副作用，虽然实现起来简单，但会干扰一些纯函数的编写，比如多态函数的部分应用：

```ocaml
let rev = List.fold_left (fun acc x -> x :: acc) []
(* rev : '_weak1 list -> '_weak1 list *)
rev [1; 2; 3; 4]
(* rev : int list -> int list *)
rev ['a'; 'b'; 'c'; 'd']
(* Error: This expression has type char but an expression was expected of type int *)
```

当然，想要让 `rev` 恢复多态类型，我们可以显式加上它的参数 `l`，也就是所谓的 eta-expansion。因为 `fun l -> ……`  在语法上是值，所以 `rev` 的类型就能推断为多态的 `'a list -> 'a list` 了。

另一方面，Haskell 是纯函数式语言，没有 OCaml 那样隐式的副作用，因此也没有值限制（不过乱用 `unsafePerformIO` 就能以相同的方式摧毁 Haskell 的类型安全）。Haskell 有个名字很像的限制叫做「[[单态限制]{monomorphism restriction}](https://www.haskell.org/onlinereport/haskell2010/haskellch4.html#x10-930004.5.5)」，但这个限制与类型安全和副作用都没有关系，而且基本上只要写了类型签名就能避免。单态限制在 GHC 中可以自由开关，如今在 GHCi 中是默认关闭的。

### Rank-N

HM 类型系统只支持顶层的 Rank-1 多态，这对应于逻辑学中的「[前束范式]{prenex normal form}」。不少 Haskell 用户不满足于此，所以 GHC 提供了 [RankNTypes](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/exts/rank_polymorphism.html) 语言扩展支持 `->` 内层的 `forall`。因为 `forall` 出现在 `->` 右侧等价于出现在外层，如 `() -> forall a. a -> a` 等价于 `forall a. () -> a -> a`，所以真正有趣的情形是 `forall` 出现在 `->` 左侧：

```haskell
hipoly :: (forall a. a -> a) -> (Bool, Char)
hipoly f = (f True, f 'a')
```

想要更强的表达能力也要付出相应的代价——Rank-3 及以上的完整类型推断已被 A. J. Kfoury 和 J. B. Wells 在 [LFP 1994](https://doi.org/10.1145/182409.182456) 的论文中证明是不可能的。不过在手动标注一些类型的前提下，GHC 仍然能够进行相当实用的类型推断，其算法在 Simon Peyton Jones 等人的 [JFP 2007](https://doi.org/10.1017/S0956796806006034) 论文中有详尽叙述。需要注意的是，就算开了扩展，`hipoly` 的类型签名也不能省略，因为 Haskell 的类型推断算法默认函数参数以及模式匹配绑定的变量不是多态类型。

### 非直谓性

介绍完 Rank-N，我们再来看看 HM 类型系统的另一个限制——[直谓性]{predicativity}。这个词同样来自逻辑学，表示不允许自指，比如 `type T = forall a. a` 中的 `a` 不可以包含 `T` 自身。在直谓多态中，这意味着多态不是头等公民，我们不能使用多态类型来实例化类型变量。在 Haskell 中比较典型的例子是处理局部副作用时会用到的 `Control.Monad.ST`：

```haskell
($) :: (a -> b) -> a -> b
runST :: (forall s. ST s d) -> d

runST $ do { …… }  -- a := (forall s. ST s d) -> d
```

在这个例子中，`$` 运算符的类型参数 `a` 得实例化成一个多态类型，这样的多态就不是直谓性的。不过好在 GHC 对 `$` 的类型检查进行了特殊处理，这样写并不会报错；如果想亲眼目睹类型错误，可以试试没有经过特殊处理的 `id runST`。从另一个角度来看，直谓多态只支持在 `->` 类型运算符里嵌套多态类型，比如 `(forall a. a) -> ()`；而非直谓多态支持在任何多态类型里嵌套多态类型，比如 `[forall a. a]`。过去二十年来，非直谓多态系统的类型推断算是相对活跃的研究课题；GHC 也见证了研究的进展——[ImpredicativeTypes](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/exts/impredicative_types.html) 扩展以前是基于 [ICFP 2006](https://doi.org/10.1145/1159803.1159838) 论文实现的，可惜不太可靠，而最近 GHC 9.2 基于 [ICFP 2020](https://doi.org/10.1145/3408971) 论文中的 Quick Look 算法更好地支持了非直谓多态。在成功突破 HM 类型系统的两大限制之后，经过 GHC 扩展的 Haskell 已经能表达比 Hindley–Milner 更强大的 Girard–Reynolds 多态演算了，也就是大名鼎鼎的 System F。

### 实现

虽然参数多态在函数式编程语言中大同小异，但泛型在面向对象编程语言中的设计千差万别，实现也不尽相同。泛型的两种主流实现方式是「[类型擦除]{type erasure}」和「[单态化]{monomorphization}」，前者以 [Java](https://openjdk.org/projects/valhalla/design-notes/in-defense-of-erasure)、[Haskell](https://gitlab.haskell.org/ghc/ghc/-/wikis/dependent-haskell) 为代表，后者以 [C++](https://en.cppreference.com/w/cpp/language/templates)、[Rust](https://davidtw.co/media/masters_dissertation.pdf)、[Go](https://github.com/golang/proposal/blob/master/design/generics-implementation-dictionaries-go1.18.md) 为代表。值得一提的是，Java 5.0 和 Go 1.18 之前并没有泛型，它们的泛型特性都是在学术界的协助下追加的，这两项工作（名为 Featherweight Generic {Java,Go}）分别发表在 [OOPSLA 1999](https://doi.org/10.1145/320385.320395) 和 [OOPSLA 2020](https://doi.org/10.1145/3428217) 上。在 Java 中，泛型列表的两个实例 `List<Integer>` 和 `List<Boolean>` 都会翻译到 `List<Object>`；而 Go 则会将 `List[int]` 和 `List[bool]` 翻译到两个不同的单态类型。虽然单态化会生成更多的代码，但它生成的代码比 Java 擦除类型的代码更高效，因为 Java 泛型的类型变量一定都是装箱了的（即 `Object` 的派生类），而 Go 可以用原始类型来实例化类型变量。另一方面，Java 的类型转换不支持类型变量，譬如 `(a)x`；而 Go 支持等价的类型断言 `x.(a)`。

C++ 的泛型是通过模板实现的，模板的实例化相当于泛型的单态化。不过 C++ 的模板比一般的泛型更为强大：模板的参数并不局限于类型参数、参数可以有默认值、模板支持特化等等。其中模板特化可以说是参数多态和特设多态的结合体：

```cpp
template<typename T> void f(T x) { /* primary template */ }
template<> void f(int x) { /* specialization  T := int */ }
```

在第一行的主模板中，我们可以定义默认的泛型实现；而对于一些需要特设实现的类型，我们可以像第二行一样对模板进行特化，譬如为整数类型写一个单独的定义。如果活用 C++ 模板的各种特性，我们甚至可以在编译期间进行任意计算，因为模板元编程已经被证明是图灵完备的。

## 子类型多态

众所周知，面向对象编程有三大特性：封装、继承和多态。而研究类型系统的学者会说，面向对象编程所需的类型系统区别于函数式演算的最大特征就是子类型。面向对象编程所说的第三大特性，学名正是「[子类型多态]{subtype polymorphism}」。如果 S 是 T 的子类型（即 S <: T），那么一个 T 类型的对象可以安全地被一个 S 类型的对象所代换；在此前提下，子类型多态是说一个类型为 T 的对象的成员函数既有可能调用 T 本身的实现，也有可能调用到 T 的子类型（如 S）的实现。

面向对象编程语言通常使用「[名义子类型]{nominal subtyping}」，也就是说子类型关系是通过名字显式声明的。在 C++、Java、C#、Swift 等语言中，定义类时可以声明它继承于什么，那么这个派生类不仅能复用基类的实现，而且成为了基类的子类型；反过来说，没有继承关系的类之间也不会有子类型关系。常见编程语言中的异类是 OCaml 和 TypeScript，它们使用的是学术界更青睐的「[结构子类型]{structural subtyping}」，也就是说子类型关系跟类型的名字没有任何关系，也不需要显式声明，而是由类型的实际结构通过一系列子类型规则决定的。在这些结构类型系统中，类名并不会同时充当对象的类型，这与主流的面向对象编程相去甚远，因此下面讨论的子类型多态均基于传统的名义类型系统。

### 动态实现

之前提到的特设多态和参数多态，往往都是在编译期间静态实现的；而子类型多态需要获知一个对象的动态类型，所以通常在运行时实现。这里我们以 C++ 为例：

```cpp
struct Animal {
  virtual void say() = 0;
};
struct Fox : Animal {
  void say() override;
};
void call(Animal *a) { a->say(); }
```

C++ 对于函数调用**默认**是静态绑定的，也就是说调用哪个成员函数完全取决于对象所标注的类型，比如 `a->say()` 就一定会调用 `Animal::say()`。但实际上 `a` 可能是 `Fox` 的实例，我们在派生类定义了不同于基类的实现，比如 `Fox::say()`。到底 `a` 是哪个类的实例我们在运行时才能知道，所以函数绑定就要延迟到运行时再进行，这就是所谓的「[动态派发]{dynamic dispatch}」。要让 C++ 进行动态派发，我们必须在基类的接口前面加上 `virtual` 关键字。这样一来，C++ 就会为每个实例附上「[虚函数表]{vtable}」，以记录各个虚函数实现的函数指针。值得一提的是，Rust 的 `trait` 对象也支持动态派发，但它没有把虚函数表存到实例里，而是使用「[胖指针]{fat pointer}」同时指向实例和虚函数表。而在 Smalltalk 这类基于「[消息传递]{message passing}」的动态编程语言中，对象的成员随时都能动态变更，因此所有消息（成员函数）都是动态传递（调用）的，我们甚至能自定义 `messageNotUnderstand:` 来动态处理未知消息。

### 静态实现

C++ 也可以用「[奇异递归模板模式]{curiously recurring template pattern}」静态实现子类型多态，即用泛型模拟多态。直观上讲，类型参数在这里充当了虚函数的索引，构造派生类实例时其实际类型会被静态记录下来。因为不同的派生类继承于基类模板的不同实例，所以 `Animal<T>::say()` 中的静态类型转换会把函数派发给对应的派生类 `T`：

```cpp
template<typename T>
struct Animal {
  void say() {
    static_cast<T*>(this)->say();
  }
};
struct Fox : Animal<Fox> {
  void say();
};
template<typename T>
void call(Animal<T> *a) { a->say(); }
```

当然，这样静态模拟子类型多态会丧失一些表达能力，比如我们无法将这些派生类的对象装进同一个容器：`Animal` 是模板而不是具体的类，所以我们无法直接写 `vector<Animal*>`；如果改成诸如 `vector<Animal<T>*>` 的形式，那显然就没法装下 `T` 以外的派生类的对象了。

## 行多态

「[行多态]{row polymorphism}」与子类型多态一样，是一种主要服务于对象（或记录）的多态形式。不同于子类型多态在实践中常常以名义子类型的形态出现，行多态理论上只适用于结构类型系统。目前学术界对于行多态的最佳实践莫衷一是，不同文献中的设计各异其趣，想了解「行的四种写法」可以移步游客账户的[知乎文章](https://zhuanlan.zhihu.com/p/108627098)。这里我们不去罗列理论，而是以实际的编程语言 PureScript 为例：

```haskell
addFields :: forall (r :: Row Type). { foo :: Int, bar :: Int | r } -> Int
addFields o = o.foo + o.bar + 1

addFields { foo: 1, bar: 2, baz: 3 }  -- r := ( baz :: Int )
addFields { foo: 1 }                  -- Type Error!
```

因为我们在第二行的函数定义中访问了记录的两个字段，所以 PureScript 会像第一行一样将其类型推断为 `{ foo :: Int, bar :: Int | r }`，这里的类型变量 `r` 代表一行类型，也就是该记录尚未知晓的剩余字段，相当于扮演了「[宽度子类型化]{width subtyping}」的角色。简而言之，所谓的行多态就是量化范围为「[行]{row}」的参数多态。

不过要注意：行多态**不能**完全取代子类型！单单使用行多态的一大缺陷是我们无法将不同类型的记录装进同一个容器，比如 `[ { x: 1, y: 2 }, { x: 1, z: 3 } ]`，个中缘由跟静态实现子类型多态的时候几乎一样。反过来，对于下面基于行多态的记录更新操作：

```haskell
incCount :: forall r. { count :: Int | r } -> { count :: Int | r }
incCount o = o { count = o.count + 1 }

(incCount { count: 0, uuid: "xxx" }).uuid  -- "xxx"
```

如果我们直接把行多态去掉（删掉 `forall r.` 和 `| r`），就算 PureScript 有子类型多态也无法通过类型检查，因为函数返回类型中 `count` 以外的字段都丢失了。这就需要 PureScript 进一步支持有界多态，然后我们把函数签名改成 `forall a <: { count :: Int }. a -> a` 才行。

除了行多态，其实还有别的方法能支持多态的对象，比如谢宁宁等人在 [ECOOP 2020](https://doi.org/10.4230/LIPIcs.ECOOP.2020.27) 的论文中证明「[互斥多态]{disjoint polymorphism}」能够模拟行多态和有界多态。互斥多态借助的利器是交集类型，这一想法可以追溯到 Benjamin Pierce 的博士毕业论文。Rust 之父 Graydon Hoare 对互斥交集类型也很关注，他曾在[推特](https://web.archive.org/web/20201114005825/https://twitter.com/graydon_pub/status/1327415381061902339)评论道：“Maybe John Reynolds really did almost solve everything at once with Forsythe, if we just manage to get its intersection types right.”

## 结语

直觉上，大家一定觉得一门编程语言支持的特性越多越好；然而在类型系统领域，让不同特性和谐共处往往是十分艰巨的话题，甚至有些特性是相互矛盾的。上文提到的参数多态和子类型就是最典型的例子：虽然这两个概念单独考虑都不算太复杂，但在它们组合而成的 F-sub 系统中，子类型关系竟然是不可判定的。这时候，语言实现通常会牺牲完备性来换取可判定的算法；当然也可以通过像行多态一样改变编程语言的设计来巧妙地避开问题，这就得看语言设计者的知识水平了。
