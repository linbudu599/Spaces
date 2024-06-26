---
description: '2021-11-24'
---

# TypeScript never 类型详解

### 前言

本篇文章是 **TypeScript 的另一面：类型编程** 系列的第 1 篇，这一系列将发布在同名专栏中（见 [知乎专栏](https://www.zhihu.com/column/c\_1446787480888053760) 或 [掘金专栏](https://juejin.cn/column/7034105175489019940)）。同时，这一系列的文章将主要继承于笔者在去年的同名文章（[原版](https://juejin.cn/post/6885672896128090125)，[炒冷饭版](https://juejin.cn/post/7000360236372459527)）内容中各部分，并进行进一步扩展深入，除本篇的 never 类型以外，还将包括如条件类型与协变 & 逆变、infer 与递归 & 尾递归、TypeScript 中的控制流分析、TypeScript 工具链的探索等部分，欢迎关注。&#x20;

至于会有这个系列的原因，首先是因为这篇文章原版其实阅读还蛮广的，不论在内网外网，可能都是我目前阅读最高、受众最广的一篇。而在这篇文章中，我尝试把绝大部分的内容都塞进去，导致并不是所有人都能读下来，或者读下来都能还保持神志清醒。 同时，原文中的部分内容在今天的笔者看来也显得有些太浅太泛了（水平有限，还望见谅）。所以就有了这个系列，区别于一股脑填鸭，这个系列的每一篇文章只会关注一个独立的问题，因此保证了内容纯粹（就跟笔者的人格一样纯粹） + 篇幅精简。

最重要的是，本系列文章默认你已经拥有 TypeScript 的基础，阅读过之前的专栏文章更好。 另外，本文以及本系列内容部分来自于笔者将在 12 月 QCon+[「TypeScript 在中大型项目中的落地实践」](https://qconplus.infoq.cn/2021/beijing2nth/track/1240) 专题中进行的分享，同样欢迎关注。&#x20;

本篇主要从以下部分出发，介绍 TypeScript 的 never 类型，由于笔者才疏学浅，未曾了解过其他计算机语言中的类型系统知识，如果出现错漏，还望不吝指出。

* TypeScript 中的三个特殊类型：any、unknown、never
* 如果只用 never，我们可以做到什么？&#x20;
* never 在工具类型中的作用

在开始前，感谢 [雪碧老师](https://www.zhihu.com/people/doodlewind)、[阿伟](https://www.zhihu.com/people/aweiu)、[寻找海蓝](https://www.zhihu.com/people/xunzhaohailan)、[三七二十](https://www.zhihu.com/people/san-qi-er-shi) 等前辈的 TypeScript 相关分享，本文内容也大量受到他们的影响，再次向以上前辈的分享精神致敬。 > P.S. 你可以在 [GitHub](https://github.com/linbudu599/Type-Programming-in-TypeScript) 获取所有示例代码。

### never、any、unknown

在 TypeScript 中，有这么几个类型可能一直困扰着初学者们，any、unknown 以及 never，这都是啥玩意啊？？啥时候用 any，啥时候用 unknown，never 好像没见过人用？

先想想我们一般啥时候用 any，比如某个变量实际上就是某个类型，但是由于中途各种操作你没做的严丝合缝，到某一步类型报错了，这个时候可以先 as 成 any，再 as 成你想要的类型，然后你就又有类型提示了（当然，我觉得直接 `as any` 的情况比较多）。

```typescript
// 要是你在公司这么写代码被打了，别把我供出来
const foo = {} as any as Function;
```

as 意味着什么？你指着编译器的脸告诉它，这个变量的类型就是这个，不服憋着。

为什么要 as 两次？不能直接 `as Function`？好问题！因为 ~~TS 编译器会用报错狠狠的抽你~~ as 实际上只能转换存在父子类型的关系，对于风马牛不相及的关系它是不理你的，所以你需要先 as 成 any，像中介一样强行把原类型和新类型关联起来。如果要稍微规范一点，应该先 as 成原类型和新类型的父类型，再 as 成新类型，如雪碧老师的[例子](https://www.zhihu.com/question/355283769/answer/2136229141)：

```typescript
// Deer、Horse的公共父类型
interface Animal {}

interface Deer extends Animal {
  deerId: number;
}

interface Horse extends Animal {
  horseId: number;
}

let deer: Deer = { deerId: 0 };

// 并不能一步到位
let horse = deer as Horse;

// 先提升成共同的父类型，再定位到子类型
let horse = deer as Animal as Horse;
```

后来，我们有了 unknown，编译器对于关联不相关的两个类型的提示也变成了 “求求你先 as 成 unknown 吧”，那么 unknown 和 any 有啥区别？

首先，在 TypeScript 的类型系统中，any 与 unknown 都属于 Top Type（在大部分类型语言中都有这么个玩意，如 PHP 中的 `mixed`，Kotlin 中的 `Any?` 等），也就是说在类型层级中它们位于顶点，但 any 类型的变量可以被赋值以任意类型的值，而 unknown 则只能接受 unknown 与 any。二者的出发点其实是一致的，那就是快速表示一个未知/动态的值，但 any 显然更加无拘无束：

```typescript
let foo: any;
foo.bar().baz();
```

这样都不会报错！和 JavaScript 还有啥区别（狗头），使用 any 意味着你在这里其实完全放弃了类型检查，更可怕的是 any 的传染性，一个变量被声明为 any，那么接下来所有基于其操作派生来的值就都被打上了隐式 any（如果没有类型断言或者基于控制流分析的类型收窄）。但 unknown 不一样，它就像是类型安全版本的 any：因为类型检查仍然存在。

```typescript
let bar: unknown;

// 这里是会报错的
// @ts-expect-error
bar.baz().foo();
```

对于被声明为 unknown 的变量，你没法直接读写它，而是必须先指定类型，显式指定、类型守卫、编译器的自动分析都行，比如类型守卫：

```typescript
function isString(input: unknown): input is string {
  return typeof input === "string";
}
```

既然有 Top Type，那么就应该要有 Bottom Type，在 TypeScript 中 never 就是那个 Bottom Type。Bottom Type 意味着**一个不表示任何类型的类型**，在 Kotlin 中它是 `Nothing`，在 Rust 中则用 `!` 修饰一个没有返回值的类型。你可能觉得，`string` 已经挺具体了，'linbudu' 这种字面量类型就更具体了，但 never 还要更具体。它是所有类型的子类型，是类型系统的最底层，也就意味着没有任何类型可以赋给它，除了 never 本身。

在 TypeScript 中，一个必定抛出错误的函数，它的返回值就是 never。说到这里你可能会想还有一个特殊的小伙伴 void（我们经常会写 `Promise<void>` 来表示一个直接 resolve 掉的 Promise），而 void 和 never 的区别就在于，返回 void 的函数其内部还是会调用 return 语句，只不过它啥也没返回，void 代表啥类型也没有（甚至 void 其实并不应该被看做一个类型），而返回 never 的函数其内部压根就没有调用 return 语句，never 代表返回压根就不存在，哪来的类型捏。

了解了 never 的基础概念，接下来要准备用用它了:)

### 最基本的 never 使用

never 单独使用的场景是非常少的，但也不是没有，我们前面说过，它是任何类型的子类型，没有比它更 narrow 的类型，所以就不可能被复制给 never 类型。我们可以使用这一点来确保在 if...else 或者 swicth case 语句中，所有可能的(类型)分支都被穷举到。在 TSConfig 中，有个类似的 `noImplicitReturns` 选项，它确保了函数的所有路径分支都必须返回值，但很明显粒度（也可以说力度）并不够。

一个简单的例子，某个变量使用联合类型定义：

```typescript
const strOrNum: string | number = "foo";

if (typeof strOrNum === "string") {
  console.log("str!");
} else if (typeof strOrNum === "number") {
  console.log("num!");
}
```

看起来好像没啥问题，假设某天这个变量的联合类型又多了一个成员：

```typescript
const strOrNumOrBool: string | number | boolean = false;
```

而一个很经常出现的场景是，你需要这个变量所有可能的联合类型都被处理到，并对每一个联合类型的成员进行特殊处理。在上面的代码中，如果你忘记了处理 boolean 类型的情况，TypeScript 也不会报错（当然不会了，如果它能智能到这种程度，干脆改名叫 AIScript 好了）。

那么，要怎么在漏处理类型分支的时候抛出错误？首先，由于 TypeScript 的类型收窄能力，我们能在每一个 else 的 语法块中将变量的类型收窄到对应的值。如果我们在上面的语句中加一个 兜底的 else 语句：

```typescript
const strOrNumOrBool: string | number | boolean = "foo";

if (typeof strOrNumOrBool === "string") {
  console.log("str!");
} else if (typeof strOrNumOrBool === "number") {
  console.log("num!");
} else {
  // ...
}
```

在最后的 else 语句块中，如果我们还使用这个变量，那么它就会被智能推导为 boolean 类型。这肯定不是我们想看到的，它都走到兜底语句块了还有未收窄过的类型。所以我们简单粗暴的把它赋值为 never（回到最开始的例子）：

```typescript
const strOrNumOrBool: string | number | boolean = false;

if (typeof strOrNumOrBool === "string") {
  console.log("str!");
} else if (typeof strOrNumOrBool === "number") {
  console.log("num!");
} else {
  const _exhaustiveCheck: never = strOrNumOrBool;
}
```

好的，报错来了：**不能将类型“boolean”分配给类型“never”。ts(2322)**

我们再加上一个处理 boolean 类型的分支：

```typescript
if (typeof strOrNumOrBool === "string") {
  console.log("str!");
} else if (typeof strOrNumOrBool === "number") {
  console.log("num!");
} else if (typeof strOrNumOrBool === "boolean") {
  console.log("bool!");
} else {
  const _exhaustiveCheck: never = strOrNumOrBool;
}
```

现在就没问题了，因为在穷举完所有类型分支后，`strOrNumOrBool`的类型当然就也是 never 啦。这样做只是从 TypeScript 类型层面避免了遗漏，为了安全起见，我们可以在 else 兜底语句中抛出一个错误：

```typescript
// ...
else {
  const _exhaustiveCheck: never = strOrNumOrBool;
  throw new Error(`Unknown input type: ${_exhaustiveCheck}`);
}
```

一个类似的场景，枚举 + switch case 语句，可能是最常见的组合之一：

```typescript
enum PossibleType {
  Foo = "Foo",
  Bar = "Bar",
  Baz = "Baz",
}

function checker(input: PossibleType) {
  switch (input) {
    case PossibleType.Foo:
      console.log("foo!");
      break;
    case PossibleType.Bar:
      console.log("bar!");
      break;
    case PossibleType.Baz:
      console.log("baz!");
      break;
    default:
      const _exhaustiveCheck: never = input;
      break;
  }
}
```

现在这个是没有问题的，可是一旦你在枚举值中新增了一个成员（见示例），就会出现不能赋值给 never 的报错提示。

### 在工具类型中大显身手

好的，又到了喜闻乐见的工具类型环节，多少同学曾为了一个洋洋洒洒几十行的工具类型挠破头。其实吧，对于类型体操这一类，我感觉会中级的基本上就够了，你在写业务的时候来一个巨复杂巨绕十几个泛型参数的工具类型，你看同事会不会捶你。

回到 never，never 在工具类型中的作用其实可以概括为三个方面：

* **作为泛型参数的默认值，以支持工具类型内部，基于入参个数变化的类型处理逻辑。**
* **与 infer + 条件类型结合，提取符合特定 Type Structure 的特定位置的值，如果待提取的类型参数不符合此结构，则返回 never。**
* **通过将接口中的部分属性指定为 never，去除其中的某些属性，以此来实现裁剪/组装/扩展接口。**

首先是作为泛型参数的默认值，上一个稍微绕点的例子：

```typescript
type Equal<X, Y, A = X, B = never> = (<T>() => T extends X ? 1 : 2) extends <
  T
>() => T extends Y ? 1 : 2
  ? A
  : B;
```

严格来说，这个例子和第一点并没有特别大的关系，但这会是后面比较重要的工具类型的基础，同时也普遍被反馈比较难理解，所以这里扔上来提前讲解一下。

这个结构是不是还挺诡异的，1 和 2 是什么东西，为什么 extends 里面还有泛型，这个 A 和 B 又是干啥的？

定睛一看（对不起，最近郭德纲和于谦两位老师的相声听多了，定睛一看老是想到...），其实就是两个东西作比较：`(<T>() => T extends X ? 1 : 2)` 和 `(<T>() => T extends X ? 1 : 2)`，再定睛一看，这不就是一样的两坨吗。等等，T 又哪来的？

再一个个来，首先这个结构就是，如果 `(<T>() => T extends X ? 1 : 2)` 能够 extends `(<T>() => T extends Y ? 1 : 2)`，那就返回 A，否则返回 B。

> 关于 extends，这涉及到协变与逆变相关的部分，在这里你可以简单理解为，**左边的类型更加狭窄具体，右边的类型更加宽松广泛时（即，右边类型中有的在左边肯定有！）**，extends 成立：
>
> ```typescript
> // "TRUE"
> type Test = { foo: string; bar: boolean } extends { foo: string }
>   ? "TRUE"
>   : "FALSE";
> ```

这个泛型 T，你会发现它实际上也没有具体的值，而是作为整个 `(<T>() => T extends X ? 1 : 2)` 结构的一部分参与到比较当中（毕竟才刚定义嘛，通过`<T>`）。在这个例子中我们其实只是用这个套了一层作为条件，来比较 X 与 Y 是否相等。诶，怎么又变相等了，说好的 extends 是一个狭窄一个宽松呢？

仔细看老伙计，这里不是简单的 `X extends Y`，不然我为啥要外面套一层这东西？通过将 X 与 Y 放置在临时函数的返回值中间接判定，确保了只有在 X 与 Y 完全一致时才能通过 extends。这里的完全一致指修饰符啥的也一致，比如两个接口的话，接口 A 的属性 `foo` 是只读 `readonly` 的，那接口 B 的属性 `foo` 就也得是只读的，一个简单例子：

```typescript
// fail
type TestEqual1 = Equal<[], readonly [], "pass", "fail">;
// pass
type TestEqual2 = Equal<[], [], "pass", "fail">;
```

再看这里的 A 和 B，其实就是决定了 extends 通过与否对应的返回值，这里的 `A = X` 和 `B = never` 其实和它实际的场景有关，完全可以不写或者自由发挥。

下一点，与 infer + 条件类型结合，说到这我可就不困了啊，毕竟这是最简单的场景了，甚至可以直接看例子就行。当然，前提是你了解 infer。

```typescript
type FuncReturnType<T extends (...args: any) => any> = T extends (
  ...args: any
) => infer R
  ? R
  : never;
```

基本上所有进行单次提取行为的工具类型都是这个结构，比如 TS 4.5 新增的 `Awaited`（或者叫 PromiseValue），提取一个 Promise 的内部类型；同样是内置的 Parameters（提取函数参数类型）、ConstructorParameters （提取构造函数参数类型），都是这么个套路，没啥意思。

然后是不带 infer 玩，never 和条件类型搞小团体的版本，这种单次判断也很简单，比如 Exclude、Extract 两兄弟：

```typescript
type Exclude<T, U> = T extends U ? never : T;
type Extract<T, U> = T extends U ? T : never;
```

看起来好像很朴素，那是因为这里的 T 和 U 通常是联合类型为多，如：

```typescript
interface Tmp1 {
  foo: string;
  bar: string;
  baz: string;
}

interface Tmp2 {
  foo: string;
  baz: string;
}

// "bar"
type ExcludedKeys = Exclude<keyof Tmp1, keyof Tmp2>;
// "foo" | "baz";
type ExtractedKeys = Extract<keyof Tmp1, keyof Tmp2>;
```

> 当然，你也可以把 keyof 封装进去，即
>
> ```typescript
> type ExcludedKeys<T, U> = keyof T extends keyof U ? never : keyof T;
> type ExtractedKeys<T, U> = keyof T extends keyof U ? keyof T : never;
> ```
>
> 为什么不呢？

我希望看到这里，你已经摸到了一丝 never 实际作用的曙光，即它在工具类型里到底起什么作用，如果没有也没关系，不然我这摊子就摆不下去了。接下来我们来看一些更常见、更强大的 never 应用吧。

先来看这样一个接口：

```typescript
interface ITmp {
  foo: number;
  bar: string;
  baz: never;
}
```

你猜使用这个接口的对象能不能有 baz 属性？事出无常必有妖，所以这里肯定是不行。这么一个接口有作用？好像是没有，显式的声明 never 意味着你都知道有哪些属性是需要剔除（而且需要手动声明）的了。那如果我们通过各种体操动作，最后得到这个接口呢？

这也是 never 的第三种作用，也是我个人认为最强大最 Amazing 的一种：**通过将接口中的部分属性指定为 never，去除其中的某些属性，以此来实现裁剪/组装/扩展接口。**

我们可以通过使用 映射类型 + 条件类型 很容易的得到这种结构的接口，如

```typescript
type ProcessedTypeWithNonFuncPropAsNever<T extends object> = {
  [K in keyof T]-?: T[K] extends Function ? K : never;
};

interface IInterfaceWithFuncProps {
  foo: string;
  bar: string;
  func1: () => void;
  func2: () => void;
}
```

> 好吧这个名字有点小长，将就一下好了，一直 foo bar baz 怪敷衍的。
>
> 这里的 `-?` 意为去除 `?:` 标志。

这个得到的结果是：

```typescript
type Result = ProcessedTypeWithNonFuncPropAsNever<IInterfaceWithFuncProps>;

type Result = {
  foo: never;
  bar: never;
  fun1: "func1";
  func2: "func2";
};
```

这个好像没啥用啊？函数还可以更近一步，获取所有函数类型的键组成的联合类型，比如 "foo1" | "foo2"，只需要加上一点小小的东西：

```typescript
type FuncTypeKeys<T extends object> = {
  [K in keyof T]-?: T[K] extends Function ? K : never;
}[keyof T];

// "func1" | "func2"
type Result = FuncTypeKeys<IInterfaceWithFuncProps>;
```

> 把 `T[K] extends Function ? K : never;` 改成 `T[K] extends Function ? T[K] : never;` 就可以直接拿到值啦。

这个可能简单了点，看个稍微复杂点的：

```typescript
type MutableKeys<T extends object> = {
  [P in keyof T]-?: Equal<
    { [Q in P]: T[P] },
    { -readonly [Q in P]: T[P] },
    P,
    never
  >;
}[keyof T];

type Equal<X, Y, A = X, B = never> = (<T>() => T extends X ? 1 : 2) extends <
  T
>() => T extends Y ? 1 : 2
  ? A
  : B;
```

如果前面没有讲过 `Equal` 这个类型的作用的话，那这个 `MutableKeys` 还是要皱眉看它个三五分钟才能看懂的，但讲过就好办多了。我们前面讲过 `Equal` 中的 `extends` 需要两边的类型完全一致，**包括修饰符如 readonly 以及 ?**，所以我相信你大概还是没看懂，拆开来看：

* 首先用映射类型把这个接口的键提取出来：`[P in keyof T]`
* 然后再对这个键提取一次：`{ [Q in P]: T[P] }` 与 `{ -readonly [Q in P]: T[P] }`
* 使用 `Equal` 比较上面的两个提取，并在比较通过时返回第一步中首次提取的键。

对于第二步，我们实现一个中介类型：

```typescript
interface IInterfaceWithReadonlyProps {
  readonly foo: string;
  bar: string;
  readonly func1: () => void;
  func2: () => void;
}

type Tmp<T extends object> = {
  [P in keyof T]-?: { [Q in P]: T[P] };
};

type A1 = Tmp<IInterfaceWithReadonlyProps>;
```

这个 A1 的类型大概是这样：

```typescript
type A1 = {
  readonly foo: {
    readonly foo: string;
  };
  bar: {
    bar: string;
  };
  readonly func1: {
    readonly func1: () => void;
  };
  func2: {
    func2: () => void;
  };
};
```

其实就是 `K:V` 变成了 `K:(K:V)` 的形式，为啥要这么做嘞？

我们在比较时为 `Equal` 中的 `Y` 传入了去掉最外层 readonly 的版本，就变成了 `readonly foo: { readonly foo: string }` vs `foo: { readonly foo: string }` ！很明显这两个比较是通不过的，那么就说明 `foo` 肯定是只读的，MutableKeys 里不准有它！

比起来 readonly，可能 `?:` optional 更多一些，那实现个 `OptionalKeys` 吧：

```typescript
export type OptionalKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? K : never;
}[keyof T];
```

我们前面说过，extends 的左边不能比右边更宽松，所以，`{} extends { a:number }` 肯定不成立，但是 `{} extends { a?: number }` 可以，所以使用 `Pick` 择出来可选类型就好了。

好了，美好时光总是短暂的，我也没估计读到这里大概要多久，十分钟应该要吧，让我们最后再看一个实用的工具类型，然后请关闭手机/电脑屏幕，做五分钟眼保健操。

我们经常遇到这种场景：

* 某个对象同时只能且必须满足多个接口之一，比如如果你是普通用户，那么肯定就不是 VIP。
* 某个对象中同时只能有多组属性之一，比如你要么有普通用户才有的属性（距离下一次赠送 VIP 体验券还有多久），要么有 VIP 用户才有的属性（VIP 等级）
* 某个对象中的数个属性必须同时存在或不存在，比如你要么同时有 VIP 用户才有的 VIP 等级 和 VIP 到期时间，要么啥也冇。

这些场景其实全部可以抽象为一种，互斥类型，而用 never 来实现可能是最优雅简单的办法。比如第二种情况，其实就是 `{ sendVIPExpTime: number; vipLevel: never; } | { sendVIPExpTime: never; vipLevel: number; }` ，实现也很简单：

```typescript
type Without<T, U> = { [P in Exclude<keyof T, keyof U>]?: never };

type XOR<T, U> = (Without<T, U> & U) | (Without<U, T> & T);
```

我觉得这个应该不用解释了吧，看看效果：

```typescript
interface Foo {
  foo: string;
}

interface Bar {
  bar: string;
}

// foo + bar:never | bar + foo:never
type FooOrBar = XOR<Foo, Bar>;

const fooOrBar1: FooOrBar = { foo: "foo" };
const fooOrBar2: FooOrBar = { bar: "bar" };
// Error
// @ts-expect-error
const fooOrBar3: FooOrBar = { baz: "baz" };
// Error
// @ts-expect-error
const fooOrBar4: FooOrBar = { foo: "foo", bar: "bar" };
```

以及至少满足一种备选类型的情况：

```typescript
// 必须有sharedProp，container、module至少有其中一个
// 如果去掉 Partial，则是必须至少有一个
type ComposedOption = { sharedProp: string } & Partial<
  XOR<
    {
      container: {
        containerId: number;
      };
    },
    {
      module: { modId: number };
    }
  >
>;

const option: ComposedOption = {
  sharedProp: "foo",
  container: {
    containerId: 599,
  },
};
```

写到这里，全文也该结束了。其实这篇文章还是超出了我对精简的预期，毕竟很多知识点都是牵一发而动全身（尤其是工具类型部分），如果直接扔出来的话就只能我和读者四眼懵逼了。本篇文章作为此系列的第一篇，应该是存在着很多不足的，比如我听着相声写下这篇文章，让它显得不是很严肃...，但也希望你能从中学到一些，再提醒一下，你可以在 [GitHub](https://github.com/linbudu599/Type-Programming-in-TypeScript) 找到此系列的所有代码，我们~~明年~~下次再见。

