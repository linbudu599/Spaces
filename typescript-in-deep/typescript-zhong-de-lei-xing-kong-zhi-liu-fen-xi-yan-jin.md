---
description: '2022-01-27'
---

# TypeScript 中的类型控制流分析演进

### 前言

关于 TypeScript 的控制流分析（Control Flow Analysis）是笔者在开始此专栏的相关筹划时就已经预设的题材之一，原本并没有想这么早就开始写，但，灵感来的就是这么突然:)。&#x20;

### 类型收窄与类型守卫&#x20;

在业务开发中，我们可能经常遇到一个场景，随着请求的成功与失败，一个类型可能有多种不同的值，但我们希望只使用一个函数来处理不同的情况：&#x20;

```typescript
interface SuccessResult {
  data: unknown;
  code: number;
}

interface FailureResult {
  error: unknown;
  code: number;
}

function handler(input: SuccessResult | FailureResult) {
  return new Promise((resolve, reject) => {
    if (success) {
      resolve(input.data);
    } else {
      reject(input.error);
    }
  });
}
```

很显然我们会想到使用 if...else 判断请求是否成功，比如可能会这么写：

```typescript
function isSuccess(res: SuccessResult | FailureResult): boolean {
  return "data" in res;
}

function handler(input: SuccessResult | FailureResult) {
  return new Promise((resolve, reject) => {
    if (isSuccess(input)) {
      resolve(input.data);
    } else {
      reject(input.error);
    }
  });
}
```

这么写看起来很自然，从逻辑出发也没问题，但你会发现编译器还是会无情的警告你，在这里的 if...else 语句块中两个 input 的类型都是 `SuccessResult | FailureResult`，而尝试访问的属性 data 以及 error 都不是公共的。

要解决这种情况，我们可以使用 is 关键字，简单的替换掉判断函数的返回值类型即可：

```typescript
function isSuccess(res: SuccessResult | FailureResult): res is SuccessResult {
  return "data" in res;
}
```

除了多声明一个类型守卫以外，你也可以直接在 handler 函数中判断：

```typescript
function handler(input: SuccessResult | FailureResult) {
  return new Promise((resolve, reject) => {
    if ("data" in input) {
      resolve(input.data);
    } else {
      reject(input.error);
    }
  });
}
```

好了，这下世界太平了，在两个语句中根据判断，input 被对应的推导为了预期的联合类型分支。以上的两种方式其实都是“类型守卫”的体现，区别只不过在当我们将判断逻辑提取到这个函数的外部时，需要使用 is 关键字来显式的提供类型信息。

但是为什么换成了 `is` 关键字以后，编译器就变得这么智能了？不仅知道当条件满足时的类型，还分析出了条件不满足时的类型？而在第二种方式中，当我们使用内联的判断，也能够获得类型信息？还有这些类型信息？到底是被谁消费的？

> 实际上上面还有一些新的概念，这里的 input 的类型（`SuccessResult | FailureResult`）我们称为可辨识联合类型（Discriminated Unions 或 Tagged Unions），我们在 『TypeScript 的另一面：类型编程』 与 「TypeScript 4.6 beta 发布」这两篇文章中都有详细的介绍，这里就不再赘述了。

再来看一个类似的，在多级 if...else 语句中进行类型收窄，这一例子来自于 「TypeScript 中的 never 类型」。

```typescript
const strOrNumOrBool: string | number | boolean = false;

if (typeof strOrNumOrBool === "string") {
  console.log("str!");
} else if (typeof strOrNumOrBool === "number") {
  console.log("num!");
} else if (typeof strOrNumOrBool === "boolean") {
  console.log("bool!");
} else {
  const _exhaustiveCheck: never = strOrNumOrBool;
  throw new Error(`Unknown input type: ${_exhaustiveCheck}`);
}
```

在这个例子中，随着每一次的条件判断，变量 `strOrNumOrBool` 的类型分支中就会有一条被锁定在对应的语句块中，而在后续的判断中就少了这一分支，如在第一次判断完毕后，类型分支就只剩下了 `number | boolean`，在最后的 else 块中，由于所有类型分支已被穷尽，就只剩下了 never 类型，在这里我们正是利用这一点，加上 never 类型作为 Bottom Type 只能被赋值给 never 类型本身这一点，来进行一个编译时的检查，确保所有类型分支被穷尽。

我想写到这里你可能已经明白些什么了，在上面的两个例子中控制流分析的作用实际上是一致的，都是基于逻辑的判断将具有联合类型的变量或参数的类型在局部收窄到某一精确地分支，即，如果 `typeof str === 'string'` 成立，那么 `number | boolean | string` 的联合类型在此处将被收窄到仅 `string` 类型。

在 TypeScript 中，以上这些基于代码的可达性进行路径的执行分析，从而将局部变量的类型收窄到某一范围的操作就称为（基于类型的）控制流分析。控制流随着路径“流动”，随着赋值操作“调整”，随着条件判断语句（如类型守卫）“分裂”，从而实现在每一处的变量类型预分析：

```typescript
function example() {
  let x: string | number | boolean;

  x = Math.random() < 0.5;

  // boolean
  console.log(x);

  if (Math.random() < 0.5) {
    x = "hello";
    // string
    console.log(x);
  } else {
    x = 100;
    // number
    console.log(x);
  }

  // string | number
  return x;
}
```

由于我还没学过编译原理，这里直接介绍维基百科中 + 我个人理解缝合的内容：

控制流分析（Control Flow Analysis）是一种静态代码分析技术，用于确定程序的控制流。控制流被表达为控制流图（Control Flow Graph），它本质是指计算控制流程的方法。控制流图是一个程序的抽象表现，代表了程序执行时所有可能的执行路径，通过图的形式表达了程序内语句块的可能执行流向（条件判断，循环等）。

在控制流分析的基础上，实际上还存在着数据流分析（Data Flow Analysis），它在控制流图的基础上，沿着其指引出来的路径，将程序内部的变量进行一系列的赋值、读取等操作，这也就意味着，控制流分析必然是位于数据流分析之前的。

### 基于类型的控制流分析的演进

TypeScript 中的控制流分析实际上分为两个部分，代码逻辑的控制流分析，这一部分从诞生以来就有，正如我们所说控制流分析的目的是生成代表了程序执行结构的控制流图（Control Flow Graph），而在 1.8 版本中，对于逻辑的控制流分析获得了一次增强，包括我们今天常用的 Unreachable code（无法执行到的代码，如 return、throw 语句后）、Implicit Returns（Error：不是所有路径都返回值）、Case clause fall-throughs（case 语句非空时多个 case 被连续执行）等等都是在这次更新引入的，参考 [Control flow analysis errors](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-1-8.html#control-flow-analysis-errors)。

而另一部分，对于类型的控制流分析则是在后续引入的，我们接下来开始介绍它的演进过程，注意，这里只会包括出现在 DevBlog / ChangeLog 中，比较值得关注的类型控制流分析演进。

在开始前，我们还是要区分一下，**Type Checker** 和 **Type-based Control Flow Analysis** 之间是存在差异的，我个人理解后者实际上是前者的子集，或者说一部分功能的基础，比如在类型控制流分析成功的将局部变量的类型收窄后，还需要 Type Checker 检测接下来对此变量的操作是否满足当前的类型约束。而在某些情况下则没有控制流分析的参与，如在 TypeScript 3.5 中引入的对非可辨识联合类型的检查增强：

```typescript
type Point = {
  x: number;
  y: number;
};
type Label = {
  name: string;
};

const thing: Point | Label = {
  x: 0,
  y: 0,
  name: true,
};
```

我们知道，以上的例子是一定会报错的，因为 name 被赋值了错误的类型，但在 3.5 版本前对于这种不可辨识的联合类型，的确是不会验证所有属性是否都符合要求的类型的（name 属性的存在是不会报错的，因为它也是合法的属性），而这就是 Type Checker 的工作了。

#### TypeScript 2.0：诞生

在 2.0 版本，对于类型的控制流分析被正式引入，主要实现了局部变量与参数的控制流分析，如在这一次就已经支 if...else 语句内局部变量类型的推断，以及分析所有可能的语句执行路径。本次的核心关注点是为具有联合类型的局部变量以及参数在特定的位置收窄到特定的类型分支，参考 [Control Flow Analysis for Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-0.html#control-flow-based-type-analysis)，值得一提的是**可辨识联合类型也是在此版本中被引入的**。

由于示例已在开头介绍过，这里就不再赘述。

#### TypeScript 3.2：解构 + rest 操作符、可辨识属性

在 3.2 版本中，对于对象类型的 解构 + `...` 扩展操作符的类型控制流分析也得到了支持。我们经常使用 `{ foo, ...restProps }` 这种形式来剔除对象的属性，3.2 版本的此特性即支持了 `restProps` 的类型推断，甚至在泛型的情况下：

```typescript
function excludeTag<T extends { tag: string }>(obj: T) {
  let { foo, ...rest } = obj;
  return rest;
}
const taggedPoint = { x: 10, y: 20, foo: "point" };
const point = excludeTag(taggedPoint); // { x: number, y: number }
```

对于这里的 rest 变量，类型最初为 `Pick<T, Exclude<keyof T, "foo">>`，后续被简化成 `Omit<T, "foo">`（`Omit` 类型在 3.5 版本被引入，但再过了几个版本才替换了实现）

在 3.2 版本中，对于可辨识联合属性的定义进一步获得了扩展，才使得只需要这一属性在各个类型分支中都存在一个不接受泛型的特异类型即可，如字面量类型、null 以及 undefined 等等，在以下的例子中我们就使用不同类型的 error 属性作为可辨识属性：

```typescript
type Result<T> = { error: Error; data: null } | { error: null; data: T };

function unwrap<T>(result: Result<T>) {
  if (result.error) {
    // Here 'error' is non-null
    throw result.error;
  }
  // Now 'data' is non-null
  return result.data;
}
```

#### TypeScript 3.7：asserts 关键字、never 函数支持

3.7 版本其实对我来说是一个很特别的版本，一方面我大概就是在这个版本开始接触 TS 的，另一方面这一版本引入了很多和我们日常开发息息相关的新特性，比如我至今清晰地记得可选链 `?.` 和空值合并 `??` 就是在 3.7.5 版本引入的，除此以外还有支持了类型别名的递归（Recursive Type Alias）：

```typescript
type ValueOrArray<T> = T | Array<ValueOrArray<T>>;
```

对 Project References 的改进（直接引用源码而非编译产物），以及条件判断中未调用的函数的检查：

```typescript
function doOtherThing() {}

function doSomeThing() {
  // 会抛出错误，询问你是否是想要调用这个函数
  if (doOtherThing) {
  }
}
```

以及新的指令 `ts-nocheck` 等等。

回到本文的正题，这一版本与控制流分析相关的部分主要有两个，首先是**断言函数的控制流分析支持**：

我们知道在 NodeJS 中提供了 assert 方法，用于在条件语句不满足时抛出一个错误，我们先写一个简单的这一版本以前的 assert 方法，

```typescript
function yell(str: any) {
  assert(typeof str === "string");

  return str.toFixed();
}

function assert(condition: boolean) {
  if (!condition) throw new Error();
}
```

在上面这种情况中，有了之前的经验我们应该知道这里的 str 如果能走到 return 语句，那么它的类型一定会是 string，这里我调用了一个一定不存在于 string 上的方法，它也没有抛出错误，当然，如果我们直接在调用函数内部做条件语句的判断，控制流分析就能够正确的识别（但这样就很不酷了）：

```typescript
function yell(str) {
  if (typeof str !== "string") {
    throw new TypeError("str should have been a string.");
  }
  // Error caught!
  return str.toFixed();
}
```

assert 这个类型守卫被提出到函数外部，而且看起来它也不能简单的使用 is 关键字（需要抛出错误），为了解决这个问题，TypeScript 3.7 版本专门引入了 `asserts` 关键字，我们使用它改写前面的 assert 方法：

```typescript
function assert(condition: any, msg?: string): asserts condition {
  if (!condition) {
    throw new AssertionError(msg);
  }
}
```

使用 `asserts condition` 意味着，一旦这个函数正常的 return 了，那么在此函数调用方的接下来的控制流中，这里的 condition 都是成立的，就相当于隐式的包含了类型守卫的判断。而 is 关键字也可以和 asserts 一同使用来提供进一步的类型收窄：

```typescript
function assertIsString(val: any): asserts val is string {
  if (typeof val !== "string") {
    throw new AssertionError("Not a string!");
  }
}
```

类似的，`assertIsString` 意味着如果此断言函数成功的 return，那么在调用方接下来的作用域中此变量的类型都将被确定为 string，直到再次发生更改。

另外一个在 3.7 版本中引入的类型控制流分析演进则是**对于返回值类型为 never 的函数的控制流分析支持**，这实际上是上面 asserts 相关工作的一部分（二者在同一个 PR 中被实现），功能也相对类似，假设我们有一个一言不合就抛出错误的函数和眼巴巴等着的调用方：

```typescript
function fail(message?: string): never {
  throw new Error(message);
}

function f1(x: string | undefined) {
  if (x === undefined) fail("undefined argument");
  x.length;
}

function f2(x: number): number {
  if (x >= 0) return x;
  fail("negative number");
}

function f3(x: number): number {
  if (x >= 0) return x;
  fail("negative number");
  x; // Unreachable code error
}
```

* 在 f1 中，如果 x 没有定义，那么 fail 会直接抛出一个错误，类似于 assert，这就意味着函数的后续代码不会再执行了，相当于一个隐式的类型守卫存在。如果 x 成功执行到了下面的代码，那么类型就应该被收窄到 string。
* 在 f2 中，如果 `x >= 0` 不成立，fail 将抛出错误。所以这里会报错： f2 缺少返回值语句，且返回值类型中未显式包含 any（即隐式包含了嘛），以及不是所有路径都能返回值。但实际上，如果能返回，此函数的返回值类型就是 number，所以这里应该报错的，是如果去掉了这里的 fail 调用时才真的“不是所有路径都能返回值”。
* 在 f3 中，大致类似于 f2 ，除了最后一行的 x，由于此前无法对 never 类型对控制流图的影响做出分析，这里“无法执行到的代码”并不会被检测到。

> 以上的“如果”均为 3.9 版本中的改进后表现

#### TypeScript 3.9：可辨识联合类型的交集

> 这一特性可能更像 Type Checker 而不是类型控制流分析，但我认为它是一个经典的例子。

类型的交集同样是 TypeScript 中常见的部分，如两个接口的交集就是它们具有一致类型的公共属性组成的新接口，如果没有交集则是 never。3.9 版本中对此情况同样做了进一步的支持，来看以下的例子：

```typescript
declare function smushObjects<T, U>(x: T, y: U): T & U;

interface Circle {
  kind: "circle";
  radius: number;
}
interface Square {
  kind: "square";
  sideLength: number;
}

declare let x: Circle;
declare let y: Square;

let z = smushObjects(x, y);
console.log(z.kind);
```

我们尝试用 smushObjects 方法创建一个 Circle 与 Square 的交集，再读取这个交集的 kind 字段，在 3.9 版本以前，这段代码能够正常执行，这里的 kind 类型为 never（因为 `'circle'` 与 `square` 并没有交集）。但细看我们就会发现这个结果也算不上正确，因为 Circle 与 Square 本就没有交集—— 3.9 版本现在能够提早的认识到这一点，并直接将 z 的类型推断为 never，而不是在最后去读取这个不存在的交集的属性时再定义为 never。

#### TypeScript 4.0：Class 中属性的类型

接下来，直到 4.0 版本中我们才再次看到比较大的类型控制流分析的新特性，此次主要是支持了对 Class 中属性的控制流分析：

```typescript
class Square {
  // 均能够被推导为 number
  area;
  sideLength;

  constructor(sideLength: number) {
    this.sideLength = sideLength;
    this.area = sideLength ** 2;
  }
}
```

对于未声明类型的属性，也能够通过构造函数中的赋值操作分析出此属性的类型。

#### TypeScript 4.3：泛型的上下文类型收窄

上下文相关类型（Contextual Typing）其实也是一个可以单独写一篇文章的题材，为了避免后面没东西写:-)，这里就只先简单的介绍一下好了。

我们知道，TypeScript 是有推导能力的，能够从变量的赋值推导出变量的类型，如：

```typescript
let name = "linbudu"; // string
```

这里的 `string` 来自于 `"linbudu"` 的类型，你可以把这理解为正向推导，或者说 **TS 的类型系统根据用户的输入推导出了类型**。但实际上还存在着反向的类型推导即，**用户的输入依赖 TS 的类型系统推导**（是不是有点像 **控制正转** 和 **控制反转** ？），比如我们写一个简单的事件监听函数：

```typescript
window.onerror = function (message, url, line, column, error) {};
```

正当你皱着眉头准备给这里的五个参数加上类型的时候，你惊喜的发现这里的五个参数已经都是强类型的了！因为在 `lib.dom.d.ts` 这一声明文件中已经存在 `window.onerror` 方法的定义，在我们将其赋值给一个函数时，由于类型需要兼容，在不显式指定类型的情况下，那么**这里函数的每一个位置的参数都会和预先定义的对应位置的参数类型 match 上**，这就是上下文类型的体现，即：_Contextual typing occurs when the type of an expression is implied by its location_，不翻译，因为译不出来那种味道。

> 你可能会问，如果你想显式标注和内置不一定的类型参数呢？比较推荐的方式是使用声明文件的形式修改 `onerror` 的类型定义来获得上下文类型的推导，或者，你也可以标注类型参数为原定义的超集——为什么是超集？请阅读本专栏中的「知其然，知其所以然：TypeScript 中的协变与逆变」一文。
>
> 在上面的例子中，除了参数以外，实际上这里我们指定的函数的返回值也由上下文类型赋予了类型，比如这里是 any。

要理解这里“泛型的上下文类型”到底是啥玩意，我们得先从简单的例子看起：

```typescript
function f1<T extends string | undefined>(x: T): string {
  if (x) {
    x.length;
    return x;
  }
}
```

这个例子看起来好像没什么问题，x 的类型被约束为 `string | undefined`，在 if 语句内部，通过控制流分析能够把 undefined 这一分支剔除掉，因此内部的类型是 string，直接返回这个变量看起来也没问题。但很不幸的，在此版本之前，这里会抛出一个错误：类型 `T` 不可分配给类型 `string`，即 `string | undefined` 不可分配给类型 `string`。

* 你会好奇，为什么控制流到这里就失效了？其实不然，这里失效的原因其实是 x 的类型并不是给定的，而是来自于泛型！虽然泛型是具有约束的，但约束并不会像可辨识联合类型那样直接参与到类型的控制流分析当中。
* 另外，这里实际上也有上下文类型的影子，由于我们声明了函数的返回值类型为 `string`，所以这里 `return x` 也被要求是 `string` 类型。

在 [#15576](https://github.com/microsoft/TypeScript/pull/15576) 中首先对这一情况做了改进，在属性访问、方法调用的主体（foo.bar、foo.baz()）的类型包括具有 nullable 约束的泛型参数时，控制流分析在介入前会使用这一主体当前的实际表现类型来作为分析的原型，以此来使得类型的收窄工作能够兼容原有的约束。

说人话：这里的 x ，其实际类型来自于泛型参数，而约束为 `string | undefined`，则在这种情况下，它的类型 `T` 会被替换为约束，接着再被控制流分析收窄到类型 string ，而不再只是一个孤零零的泛型参数。

再来看一个稍微复杂点的例子巩固下：

```typescript
function makeUnique<T, C extends Set<T> | T[]>(
  collection: C,
  comparer: (x: T, y: T) => number
): C {
  if (collection instanceof Set) {
    return collection;
  }

  // 错误，类型 C 上不存在属性 sort
  collection.sort(comparer);

  // 错误，类型 C 上不存在属性 length
  for (let i = 0; i < collection.length; i++) {
    let j = i;
    while (
      // 错误，类型 C 上不存在属性 length
      j < collection.length &&
      // 错误，元素隐式拥有 any 类型，因为类型 Set<T> | T[] 上不存在基于 number 的索引类型
      comparer(collection[i], collection[j + 1]) === 0
    ) {
      j++;
    }

    // 错误，类型 C 上不存在属性 splice
    collection.splice(i + 1, j - i);
  }
  return collection;
}
```

看起来好像复杂了点，但本质其实是完全一样的。

* 函数参数 collection 的类型 C，约束为 `Set<T> | T[]`，而同时函数返回值类型也来自于 C。
* 如果不满足 `instanceof` 的判断而导致 if 语句没有执行，则接下来的作用域内，collection 的类型：
  * 在此版本前，会保留泛型参数 C，导致这时只能获取约束 `Set<T> | T[]` 的交集的属性/方法，而 sort 方法明显不在，后续的报错也来自于相同的原因。
  * 在此版本后，这里的 collection 类型会被替换为 `Set<T> | T[]`，并根据已执行的控制流分析工作将其收窄到 `T[]`

简单地说，此次更新就是支持了变量类型为具有约束的泛型参数时在参与控制流分析时，使用约束作为其实际的表现类型来参与类型控制流的分析。

#### TypeScript 4.4：可辨识联合类型的全面增强

4.4 版本中，我们迎来了对可辨识联合类型中的守卫属性的增强，包括支持独立类型守卫声明、支持解构守卫属性、较复杂的判断，一个个来看：

独立类型守卫声明，我们知道可以通过简单的判断来在局部缩窄类型：

```typescript
function foo(arg: unknown) {
  if (typeof arg === "string") {
    console.log(arg.toUpperCase());
  }
}
```

但是如果我们把这一判断语句提取出来，这里的控制流分析就失效了：

```typescript
function foo(arg: unknown) {
  const argIsString = typeof arg === "string";
  if (argIsString) {
    // error!
    console.log(arg.toUpperCase());
  }
}
```

而在 4.4 版本支持了对使用 const 的独立声明、只读属性以及没有被篡改的函数入参的类型控制流分析，

> 为什么强调是 const？
>
> 这是因为只有对于使用 const 声明的变量才能确保其类型不会再发生变化，因为 let、var 声明的变量仍然可能发生重新赋值。而 const 声明的变量其类型也会被收窄到最小的分支，对于原始类型通常是由值推导出的字面量类型（对于对象类型，其使用 const 和 let 都是推导到与属性一致的接口结构）：
>
> ```typescript
> // 字符串字面量类型 'linbudu'
> const foo = "linbudu";
> // 布尔字面量类型 true
> const bool = true;
> // 数字字面量类型 599
> const num = 599;
>
> // string
> let bar = "linbudu";
> ```
>
> 对于 let 类型的声明，你可以通过使用 3.4 版本引入的常量断言（as const）来达成类似的效果，如：
>
> ```typescript
> let bar = "linbudu" as const;
> ```

支持解构守卫属性，即支持先将守卫属性解构赋值，再进行判断：

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; sideLength: number };

function area(shape: Shape): number {
  const { kind } = shape;

  if (kind === "circle") {
    return Math.PI * shape.radius ** 2;
  } else {
    return shape.sideLength ** 2;
  }
}
Try;
```

较复杂的判断，一个常见的场景是对多个 DOM 元素选择器同时判断：

```typescript
// 这三个值均为 HTMLDivElement | null
const ele1 = document.querySelector<HTMLDivElement>("#ele1");
const ele2 = document.querySelector<HTMLDivElement>("#ele2");
const ele3 = document.querySelector<HTMLDivElement>("#ele3");

if (ele1 && ele2 && ele3) {
  // 在这里三个变量都被收窄到了 HTMLDivElement
}
```

这也常见于多个守卫属性排列组合成多条特异的类型分支的情况。

#### TypeScript 4.5：模板字符串类型的可辨识属性支持

TypeScript 4.5 版本中，引入了对模板字符串类型（引入于 4.1 版本）的类型守卫以及控制流分析支持，你可以在笔者 4.5 版本的 DevBlog 解说中了解更多，示例：

```typescript
export interface Success {
  type: `${string}Success`;
  body: string;
}

export interface Error {
  type: `${string}Error`;
  message: string;
}

export function handler(r: Success | Error) {
  if (r.type === "HttpSuccess") {
    const token = r.body;
  }
}
```

我在此前的分享过曾经重点的介绍过模板字符串类型，因为我坚定的认为它是最贴近实际业务、最让开发者舒爽的类型系统 feature 之一，最主要的原因就是因为它让类型和业务逻辑不再割裂了，如业务类型、状态码这一类基础的使用，还有 Vuex、URL Parser（如工业聚老师的 Farrow）这一类上层框架的强类型支持等等。而现在模板字符串类型的类型守卫支持更是进一步的补全了它的能力，就现在，用起来！

#### TypeScript 4.6：参数类型的可辨识属性支持

紧接着就是我们最新的 4.6 beta 版本了，此次引入了对参数类型（元组类型）的可辨识联合类型的类型守卫、控制流分析支持：

```typescript
type Args = ["a", number] | ["b", string];

const f1: Func = (kind, payload) => {
  if (kind === "a") {
    payload.toFixed(); // 'payload' narrowed to 'number'
  }
  if (kind === "b") {
    payload.toUpperCase(); // 'payload' narrowed to 'string'
  }
};
```

这一特性对于一些仍然使用回调函数的方法来说能够帮助提供更好的类型提示，如 `fs.readFile` 的类型现在是这样的：

```typescript
function readFile(
  path: PathLike | number,
  callback: (err: NodeJS.ErrnoException | null, data: Buffer) => void
): void;

fs.readFile("./inexist-path", (err, data) => {
  if (err) {
    console.log(err);
  } else {
    console.log(data);
  }
});
```

我们知道这一类错误优先的回调函数中，error 参数 和 data 参数必然会有一个是 undefined，但类型声明中并没有体现出这一点。而现在，随着对于参数的控制流分析支持，现在你可以这么声明：

```typescript
type ReadFileCallbackArgs =
  | [err: undefined, data: Buffer]
  | [err: Error, data: undefined];

declare function readFile(
  path: string,
  cb: (...args: ReadFileCallbackArgs) => void
): void;

readFile("hello", (err, data) => {
  if (!err) {
    return data.byteLength; // Buffer
  } else {
    throw err.message; // Error
  }
});
```

实际上，这一支持被引入前还有一个更基础的部分，即对于通过解构赋值的可辨识联合类型参数，也支持了控制流分析，更进一步的是，如果有多个变量通过同一条解构赋值语句声明，则对于其中的可辨识属性的条件判断能够自动的分析出其他非可辨识属性的类型，我们来看例子：

```typescript
type Action = { kind: "A"; payload: number } | { kind: "B"; payload: string };

function f10({ kind, payload }: Action) {
  if (kind === "A") {
    payload.toFixed();
  }
  if (kind === "B") {
    payload.toUpperCase();
  }
}
```

在这个例子中，我们在参数部分就预先进行了解构，其中的 kind 为可辨识属性，对 kind 属性的判断能够自动的分析出对应的 payload 类型。同时除了参数部分的预解构，函数体内的解构以及 switch case 语句中对其的控制流分析都是支持的：

```typescript
function f11(action: Action) {
  const { kind, payload } = action;
  if (kind === "A") {
    payload.toFixed();
  }
  if (kind === "B") {
    payload.toUpperCase();
  }
}

function f12({ kind, payload }: Action) {
  switch (kind) {
    case "A":
      payload.toFixed();
      break;
    case "B":
      payload.toUpperCase();
      break;
    default:
      payload; // never
  }
}
```

### 杂谈 & 总结

我们从类型控制流分析的诞生一路走了下来，从它的发展历程其实你也能或多或少感觉到 TS 团队致力于的方向：提供更精确、更严格的类型控制流分析。同时，我们可以看到在 4.3 版本后每一个小版本都携带着分析能力的不断补完和增强，我个人认为这可能意味着官方在接下来的迭代重心又开始回到了类型系统的部分，很难不狂喜。

我们下篇文章见\~
