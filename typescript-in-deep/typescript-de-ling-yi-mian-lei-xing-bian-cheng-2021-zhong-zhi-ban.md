---
description: '2021-08-25'
---

# TypeScript的另一面：类型编程（2021重制版）

### 前言

作为前端开发的趋势之一，TypeScript 正在为越来越多的开发者所喜爱，从大的方面来说，几乎九成的框架与工具库都以其写就（或者就是类似的类型方案，如 Flow）；而从小的方面来说，即使是写个配置文件（如 vite 的配置文件）或者小脚本（感谢 ts-node），TypeScript 也是一大助力。一样事物不可能做到每个人都喜欢，如 nodemon 的作者 Remy Sharp 就曾表示自己从来没有使用过 TS（见 [#1565](https://github.com/remy/nodemon/issues/1565#issuecomment-490429334)），在以后也不会去学习 TS，这可能是因为语言习惯的问题。而通常阻碍新人上手 TypeScript 的还有另外一座大山：学习成本高。

在学习 TypeScript 的开始阶段，很多同学对它是又爱又恨的，离不开它的类型提示和工程能力，却又经常为类型错误困扰，最后不得不用个 any 了事，这样的情况多了，TypeScript 就慢慢写成了 AnyScript...

这个问题的罪魁祸首其实就是部分同学在开始学习 TypeScript 时，要么是被逼上梁山，在一片空白的情况下接手 TS 项目，要么是不知道如何学习，那么良心的官方文档不看，看了几篇相关文章就觉得自己会了，最后遇到问题还是一头雾水。

这篇文章就是为了解决这后者的问题，尝试专注于 TypeScript 的类型编程部分（TS 还有几个部分？请看下面的解释），从最基础的泛型开始，到索引、映射、条件等类型，再到 is、in、infer 等关键字，最后是压轴的工具类型。打开你的 IDE，跟着笔者一节节的敲完代码，帮助你的 TypeScript 水平迈上新的台阶。

> 需要注意的是，本文并非 TypeScript 入门文章，并不适用于对 TypeScript 暂时没有任何经验的同学。如果你仍处于新手期，笔者在这里推荐 xcatliu 的 [TypeScript 入门教程](https://ts.xcatliu.com/) 以及 [官方文档](https://www.typescriptlang.org/zh/)，从我个人的经验来看，你可以在初期阅读入门教程，并在感到困惑时前往官方文档对应部分查阅。
>
> 在完成 TypeScript 的基础入门后，欢迎再次回到本篇文章。

#### TypeScript = 类型编程 + ES 提案

笔者通常将 TypeScript 划分成两个部分：

*   **预实现的 ES 提案**，如 装饰器、 可选链`?.` 、空值合并运算符`??`（和可选链一起在 [TypeScript3.7](https://devblogs.microsoft.com/typescript/announcing-typescript-3-7/) 中引入）、类的私有成员 `private` 等。除了部分极端不稳定的语法（说的就是你，装饰器）以外，大部分的 TS 实现实际上就是未来的 ES 语法。

    > 严谨的来说，现在的 ES 版本装饰器和 TS 版本装饰器已经是两个东西了，笔者先前在 [走近 MidwayJS：初识 TS 装饰器与 IoC 机制](https://juejin.im/post/6859314697204662279) 这篇文章中介绍了一些关于 TS 装饰器的历史，有兴趣的同学不妨一读。

    对于这一部分来说，无论你先前是只有 JavaScript 这门语言的使用经验，还是有过 Java、C#的使用经历，都能非常快速地上手，毕竟主要还是语法糖为主嘛。当然，这也是实际开发中使用最多的部分，毕竟和另一部分：**类型编程**比起来，还是这一部分更接地气。
*   **类型编程**，从一个简简单单的`interface`，到看起来挺高级的`T extends SomeType` ，再到各种不明觉厉的工具类型`Partial`、`Required`等，这些都属于类型编程的范畴。这一部分对代码实际的功能层面没有任何影响，即使你一行代码十个 any，遇到类型错误就 `@ts-ignore` （类似于`@eslint-ignore`，将会禁用掉下一行的类型检查），甚至直接开启 `--transpileOnly` （这一选项会禁用掉 TS 编译器的类型检查能力，仅编译代码，会获得更快的编译速度·），也不会影响你代码本身的逻辑。

    然而，这也就是类型编程一直不受到太多重视的原因：相比于语法，它会带来许多额外的代码量（类型定义代码甚至可能超过业务代码量）等问题。而且实际业务中并不会需要多么苛刻的类型定义，通常只会对接口数据、应用状态流等进行定义，通常是底层框架类库才会需要大量的类型编程代码。

    如果说，上一部分让你写的代码更甜，那么这一部分，最重要的作用是让你的代码变得更优雅健壮（是的，优雅和健壮并不冲突）。如果你所在的团队使用 Sentry 这一类监控平台，对于 JS 代码来说最常见的错误就是`Cannot read property 'xxx' of undefined`、`undefined is not a function`这种（见[top-10-javascript-errors](https://rollbar.com/blog/top-10-javascript-errors/)），虽然即使是 TS 也不可能把这个错误直接完全抹消，但也能解决十之八九了。

好了，做了这么多铺垫，是时候开始进入正题了，本文的章节分布如下，如果你已经有部分前置知识的基础（如泛型），可以直接跳过。

* 类型编程的基础：泛型
* 类型守卫与 is、in 关键字
* 索引类型与映射类型
* 条件类型、分布式条件类型
* infer 关键字
* 工具类型
* TypeScript 4.x 新特性

### 泛型

之所以上来就放泛型，是因为在 TypeScript 的整个类型编程体系中，它是最基础的那部分，所有的进阶类型都基于它书写。就像编程时我们不能没有变量，类型编程中的变量就是泛型。

假设我们有这么一个函数：

```typescript
function foo(args: unknown): unknown { ... }
```

* 如果它接收一个字符串，返回这个字符串的部分截取。
* 如果接收一个数字，返回这个数字的 n 倍。
* 如果接收一个对象，返回键值被更改过的对象（键名不变）。

**上面这些场景有一个共同点，即函数的返回值与入参是同一类型.**

如果在这里要获得精确地类型定义，应该怎么做？

* 把 `unknown` 替换为 `string | number | object` ？但这样代表的意思是这个函数接受任何值，其返回类型都可能是 string / number / object，虽然有了类型定义，但完全称不上是精确。

别忘记我们需要的是 **入参与返回值类型相同** 的效果。这个时候**泛型**就该登场了，我们先用一个泛型收集参数的类型值，再将其作为返回值，就像这样：

```typescript
function foo<T>(arg: T): T {
  return arg;
}
```

这样在我们使用 `foo` 函数时，编辑器就能实时根据我们传入的参数确定此函数的返回值了。就像编程时，程序中变量的值会在其运行时才被确定，泛型的值（类型）也是在方法被调用、类被实例化等类似的执行过程实际发生时才会被确定的。

泛型使得代码段的类型定义**易于重用**（比如后续又多了一种接收 `boolean` 返回 `boolean` 的函数实现），并提升了灵活性与严谨性。

另外，你可能曾经见过 `Array<number>` `Map<string, ValueType>` 这样的使用方式，通常我们将上面例子中 `T` 这样的未赋值形式成为 **类型参数变量** 或者说 **泛型类型**，而将 `Array<number>` 这样已经实例化完毕的称为 `实际类型参数` 或者是 **参数化类型**。

> 通常泛型只会使用单个字母。如 T U K V S等。我的推荐做法是在项目达到一定复杂度后，使用带有具体意义的泛型变量声明，如 `BasicBusinessType` 这种形式。

```typescript
foo<string>("linbudu");
const [count, setCount] = useState<number>(1);
```

上面的例子也可以不指定，因为 TS 会自动推导出泛型的实际类型，在部分 Lint 规则中，实际上也不推荐添加能够被自动推导出的类型值。

泛型在箭头函数下的书写：

```typescript
const foo = <T>(arg: T) => arg;
```

> 如果你在 TSX 文件中这么写，`<T>`可能会被识别为 JSX 标签，因此需要显式告知编译器：
>
> ```typescript
> const foo = <T extends SomeBasicType>(arg: T) => arg;
> ```

除了用在函数中，泛型也可以在类中使用：

```typescript
class Foo<T, U> {
  constructor(public arg1: T, public arg2: U) {}

  public method(): T {
    return this.arg1;
  }
}
```

单独对于泛型的介绍就到这里（因为单纯的讲泛型实在没有什么好讲的），在接下来的进阶类型篇章中，我们会讲解更多泛型的使用。

### 类型守卫、is in关键字

我们来从相对简单直观的知识点：类型守卫 开始，由浅入深的了解基于泛型的类型编程。

假设有这么一个字段，它可能字符串也可能是数字：

```typescript
numOrStrProp: number | string;
```

现在在使用时，你想将这个字段的联合类型缩小范围，比如精确到`string`，你可能会这么写：

```typescript
export const isString = (arg: unknown): boolean => typeof arg === "string";
```

看看这么写的效果：

```typescript
function useIt(numOrStr: number | string) {
  if (isString(numOrStr)) {
    console.log(numOrStr.length);
  }
}
```

![image](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dddd0455f5b942ce93e932d5651e0770\~tplv-k3u1fbpfcp-zoom-1.image)

看起来 `isString` 函数并没有起到缩小类型范围的作用，参数依然是联合类型。这个时候就该使用 `is` 关键字了：

```typescript
export const isString = (arg: unknown): arg is string =>
  typeof arg === "string";
```

这个时候再去使用，就会发现在 `isString(numOrStr)` 为 `true` 后，`numOrStr`的类型就被缩小到了`string`。这只是以原始类型为成员的联合类型，我们完全可以扩展到各种场景上，先看一个简单的假值判断：

```typescript
export type Falsy = false | "" | 0 | null | undefined;

export const isFalsy = (val: unknown): val is Falsy => !val;
```

这应该是我日常用的最多的类型别名之一了，类似的，还有 `isPrimitive` 、`isFunction`这样的类型守卫。

而使用 in 关键字，我们可以进一步收窄类型（`Type Narrowing`），思考下面这个例子，要如何将 " A | B " 的联合类型缩小到"A"？

```typescript
class A {
  public a() {}

  public useA() {
    return "A";
  }
}

class B {
  public b() {}

  public useB() {
    return "B";
  }
}
```

首先联想下 `for...in` 循环，它遍历对象的属性名，而 `in` 关键字也是一样，它能够判断一个属性是否为对象所拥有：

```typescript
function useIt(arg: A | B): void {
  'a' in arg ? arg.useA() : arg.useB();
}
```

如果参数中存在`a`属性，由于A、B两个类型的交集并不包含a，所以这样能立刻收窄类型判断到 A 身上。

由于A、B两个类型的交集并不包含 a 这个属性，所以这里的 `in` 判断会精确地将类型对应收窄到三元表达式的前后。即 A 或者 B。

再看一个使用字面量类型作为类型守卫的例子：

```typescript
interface IBoy {
  name: "mike";
  gf: string;
}

interface IGirl {
  name: "sofia";
  bf: string;
}

function getLover(child: IBoy | IGirl): string {
  if (child.name === "mike") {
    return child.gf;
  } else {
    return child.bf;
  }
}
```

> 关于字面量类型`literal types`，它是对类型的进一步限制，比如你的状态码只可能是 0/1/2，那么你就可以写成 `status: 0 | 1 | 2` 的形式，而不是用一个 `number` 来表达。
>
> 字面量类型包括 **字符串字面量**、**数字字面量**、**布尔值字面量**，以及4.1版本引入的模板字面量类型（这个我们会在后面展开讲解）。
>
> * 字符串字面量，常见如 `mode: "dev" | "prod"`。
> * 布尔值字面量通常与其他字面量类型混用，如 `open: true | "none" | "chrome"`。
>
> 这一类细碎的基础知识会被穿插在文中各个部分进行讲解，以此避免单独讲解时缺少特定场景让相关概念显得过于单调。

#### 基于字段区分接口

我在日常经常看到有同学在问类似的问题：登录与未登录下的用户信息是完全不同的接口，或者是

之前有个小哥问过一个问题，我想很多用 TS 写接口的小伙伴可能都遇到过，即登录与未登录下的用户信息是完全不同的接口（或者是类似的，需要基于属性、字段来区分不同接口），其实也可以使用 `in`关键字 解决：

```typescript
interface ILogInUserProps {
  isLogin: boolean;
  name: string;
}

interface IUnLoginUserProps {
  isLogin: boolean;
  from: string;
}

type UserProps = ILogInUserProps | IUnLoginUserProps;

function getUserInfo(user: ILogInUserProps | IUnLoginUserProps): string {
  return 'name' in user ? user.name : user.from;
}
```

或者通过字面量类型：

```typescript
interface ICommonUserProps {
  type: "common",
  accountLevel: string
}

interface IVIPUserProps {
  type: "vip";
  vipLevel: string;
}

type UserProps = ICommonUserProps | IVIPUserProps;

function getUserInfo(user: ICommonUserProps | IVIPUserProps): string {
  return user.type === "common" ? user.accountLevel : user.vipLevel;
}
```

同样的思路，还可以使用`instanceof`来进行实例的类型守卫，建议聪明的你动手尝试下。

### 索引类型与映射类型

#### 索引类型

在阅读这一部分前，你需要做好思维转变的准备，需要真正认识到 **类型编程实际也是编程**，因为从这里开始，我们就将真正将泛型作为变量进行各种花式操作了。

就像你写业务代码的时候常常会遍历一个对象，而在类型编程中我们也会经常遍历一个接口。因此，你完全可以将一部分编程思路复用过来。首先实现一个简单的函数，它返回一个对象的某个键值：

```typescript
// 假设key是obj键名
function pickSingleValue(obj, key) {
  return obj[key];
}
```

要为其进行类型定义的话，有哪些需要定义的地方？

* 参数`obj`
* 参数`key`
* 返回值

这三样之间存在着一定关联：

* `key`必然是 `obj` 中的键值名之一，且一定为 `string` 类型（通常我们只会使用字符串作为对象键名）
* 返回的值一定是 **obj 中的键值**

因此我们初步得到这样的结果：

```typescript
function pickSingleValue<T>(obj: T, key: keyof T) {
  return obj[key];
}
```

`keyof` 是 \*\*索引类型查询 \*\*的语法， 它会返回后面跟着的类型参数的键值组成的字面量联合类型，举个例子：

```typescript
interface foo {
  a: number;
  b: string;
}

type A = keyof foo; // "a" | "b"
```

是不是就像 `Object.keys()` 一样？区别就在于它返回的是联合类型。

> 联合类型 `Union Type` 通常使用 `|` 语法，代表多个可能的取值，实际上在最开始我们就已经使用过了。联合类型最主要的使用场景还是 条件类型 部分，这在后面会有一个完整的章节来进行讲解。

还少了返回值，如果你此前没有接触过此类语法，应该会卡住，我们先联想下`for...in`语法，遍历对象时我们可能会这么写：

```typescript
const fooObj = { a: 1, b: "1" };

for (const key in fooObj) {
  console.log(key);
  console.log(fooObj[key]);
}
```

和上面的写法一样，我们拿到了 key，就能拿到对应的 value，那么 value 的类型就更简单了：

```typescript
function pickSingleValue<T>(obj: T, key: keyof T): T[keyof T] {
  return obj[key];
}
```

> 这一部分可能不好一步到位理解，解释下：
>
> ```typescript
> interface T {
> a: number;
> b: string;
> }
>
> type TKeys = keyof T; // "a" | "b"
>
> type PropAType = T["a"]; // number
> ```
>
> 你用键名可以取出对象上的键值，自然也就可以取出接口上的键值（也就是类型）啦\~

但这种写法很明显有可以改进的地方：`keyof`出现了两次，以及泛型 T 其实应该被限制为对象类型。对于第一点，就像我们平时编程会做的那样：用一个变量把多处出现的存起来，记得，**在类型编程里，泛型就是变量**。

```typescript
function pickSingleValue<T extends object, U extends keyof T>(
  obj: T,
  key: U
): T[U] {
  return obj[key];
}
```

> 这里又出现了新东西 `extends`... 它是啥？你可以暂时把 `T extends object` 理解为**T 被限制为对象类型**，`U extends keyof T`理解为 泛型 U 必然是泛型 T 的键名组成的联合类型（以字面量类型的形式，比如T这个对象的键名包括a b c，那么U的取值只能是"a" "b" "c"之一，即 `"a" | "b" | "c"`）。具体细节我们会在 条件类型 一章讲到。

假设现在不只要取出一个值了，我们要取出一系列值，即参数 2 将是一个数组，成员均为参数 1 的键名组成：

```typescript
function pick<T extends object, U extends keyof T>(obj: T, keys: U[]): T[U][] {
  return keys.map((key) => obj[key]);
}

// pick(obj, ['a', 'b'])
```

有两个重要变化：

* `keys: U[]` 我们知道 U 是 T 的键名组成的联合类型，那么要表示一个内部元素均是 T 键名的数组，就可以使用这种方式，具体的原理请参见下文的 **分布式条件类型** 章节。
* `T[U][]` 它的原理实际上和上面一条相同，首先是`T[U]`，代表参数1的键值（就像`Object[Key]`），我认为它是一个很好地例子，表现了 TS 类型编程的组合性，你不感觉这种写法就像搭积木一样吗？

**索引签名 Index Signature**

在JavaScript中，我们通常使用 `arr[1]` 的方式索引数组，使用 `obj[key]` 的方式索引对象。说白了，索引就是你获取一个对象成员的方式，而在类型编程中，索引签名用于快速建立一个内部字段类型相同的接口，如

```typescript
interface Foo {
  [keys: string]: string;
}
```

那么接口 Foo 实际上等价于一个键值全部为 string 类型，不限制成员的接口。

> 等同于`Record<string, string>`，见 工具类型。

值得注意的是，由于 JS 可以同时通过数字与字符串访问对象属性，因此`keyof Foo`的结果会是`string | number`。

> ```typescript
> const o: Foo = {
> 1: "芜湖！",
> };
>
> o[1] === o["1"]; // true
> ```

但是一旦某个接口的索引签名类型为`number`，那么使用它的对象就不能再通过字符串索引访问，如`o['1']`，将会抛出错误， **元素隐式具有 "any" 类型，因为索引表达式的类型不为 "number"。**

#### 映射类型 Mapped Types

在开始映射类型前，首先想想 JavaScript 中数组的 map 方法，通过使用map，我们从一个数组按照既定的映射关系获得一个新的数组。在类型编程中，我们则会从一个类型定义（包括但不限于接口、类型别名）映射得到一个新的类型定义。通常会在旧有类型的基础上进行改造，如：

* 修改原接口的键值类型
* 为原接口键值类型新增修饰符，如 `readonly` 与 可选`?`

从一个简单场景入手：

```typescript
interface A {
  a: boolean;
  b: string;
  c: number;
  d: () => void;
}
```

现在我们有个需求，实现一个接口，它的字段与接口 A 完全相同，但是其中的类型全部为 `string`，你会怎么做？直接重新声明一个然后手写吗？这样就很离谱了，我们可是机智的程序员。

如果把接口换成对象再想想，假设要拷贝一个对象（假设没有嵌套，不考虑引用类型变量存放地址），常用的方式是首先 new 一个新的空对象，然后遍历原先对象的键值对来填充新对象。而接口其实也一样：

```typescript
type StringifyA<T> = {
  [K in keyof T]: string;
};
```

是不是很熟悉？重要的就是这个`in`操作符，你完全可以把它理解为 `for...in`/`for...of` 这种遍历的思路，获取到键名之后，键值就简单了，所以我们可以很容易的拷贝一个新的类型别名出来。

```typescript
type ClonedA<T> = {
  [K in keyof T]: T[K];
};
```

掌握这种思路，其实你已经接触到一些工具类型的底层实现了：

> 你可以把工具类型理解为**你平时放在 utils 文件夹下的公共函数，提供了对公用逻辑（在这里则是类型编程逻辑）的封装**，比如上面的两个类型接口就是。关于更多工具类型，参考 工具类型 一章。

先写个最常用的 `Partial` 尝尝鲜，工具类型的详细介绍我们会在专门的章节展开：

```typescript
// 将接口下的字段全部变为可选的
type Partial<T> = {
  [K in keyof T]?: T[k];
};
```

> `key?: value` 意为这一字段是可选的，在大部分情况下等同于 `key: value | undefined`。

### 条件类型 Conditional Types

在编程中遇到条件判断，我们常用 If 语句与三元表达式实现，我个人偏爱后者，即使是：

```javascript
if (condition) {
  execute()
}
```

这种没有 else 的 If 语句，我也习惯写成：

```javascript
condition ? execute() : void 0;
```

而 条件类型 的语法，实际上就是三元表达式，看一个最简单的例子：

```typescript
T extends U ? X : Y
```

> 如果你觉得这里的 extends 不太好理解，可以暂时简单理解为 U 中的属性在 T 中都有。

为什么会有条件类型？可以看到 条件类型 通常是和 泛型 一同使用的，联想到泛型的使用场景以及值得延迟推断，我想你应该明白了些什么。对于类型无法即时确定的场景，使用 条件类型 来在运行时动态的确定最终的类型（运行时可能不太准确，或者可以理解为，你提供的函数被他人使用时，根据他人使用时传入的参数来动态确定需要被满足的类型约束）。

类比到编程语句中，其实就是根据条件判断来动态的赋予变量值：

```typescript
let unknownVar: string;

unknownVar = condition ? "淘系前端" : "淘宝FED";

type LiteralType<T> = T extends string ? "foo" : "bar";
```

条件类型理解起来其实也很直观，唯一需要有一定理解成本的就是 **何时条件类型系统会收集到足够的信息来确定类型**，也就是说，条件类型有时不会立刻完成判断，比如工具库提供的函数，需要用户在使用时传入参数才会完成 条件类型 的判断。

在了解这一点前，我们先来看看条件类型常用的一个场景：**泛型约束**，实际上就是我们上面 索引类型 的例子：

```typescript
function pickSingleValue<T extends object, U extends keyof T>(
  obj: T,
  key: U
): T[U] {
  return obj[key];
}
```

这里的 `T extends object` 与 `U extends keyof T` 都是泛型约束，分别**将 T 约束为对象类型** 和 **将 U 约束为 T 键名的字面量联合类型**（不记得了？提示：`1 | 2 | 3`）。我们通常使用泛型约束来 **收窄类型约束**，简单的说，泛型本身是来者不拒的，所有类型都能被 **显式传入**（如 `Array<number>`） 或者 **隐式推导** （如 `foo(1)`），这样其实不是我们想要的，就像我们有时会检测函数的参数：

```javascript
function checkArgFirst(arg){
  if(typeof arg !== "number"){
    throw new Error("arg must be number type!")
  }
}
```

在 TS 中，我们通过泛型约束，要求传入的泛型只能是固定的类型，如 `T extends {}` 约束泛型至对象类型，`T extends number | string`将泛型约束至数字与字符串类型。

以一个使用条件类型作为函数返回值类型的例子：

```typescript
declare function strOrNum<T extends boolean>(
  x: T
): T extends true ? string : number;
```

在这种情况下，条件类型的推导就会被延迟，因为此时类型系统没有足够的信息来完成判断。

只有给出了所需信息（在这里是入参 x 的类型），才可以完成推导。

```typescript
const strReturnType = strOrNum(true);
const numReturnType = strOrNum(false);
```

同样的，就像三元表达式可以嵌套，条件类型也可以嵌套，如果你看过一些框架源码，也会发现其中存在着许多嵌套的条件类型，无他，条件类型可以将类型约束收拢到非常窄的范围内，提供精确的条件类型，如：

```typescript
type TypeName<T> = T extends string
  ? "string"
  : T extends number
  ? "number"
  : T extends boolean
  ? "boolean"
  : T extends undefined
  ? "undefined"
  : T extends Function
  ? "function"
  : "object";
```

#### 分布式条件类型 Distributive Conditional Types

分布式条件类型实际上不是一种特殊的条件类型，而是其特性之一（所以说条件类型的分布式特性更为准确）。我们直接先上概念： **对于属于裸类型参数的检查类型，条件类型会在实例化时期自动分发到联合类型上**。

> 原文: _Conditional types in which the checked type is a **naked type parameter** are called distributive conditional types. Distributive conditional types are automatically **distributed over union types** during instantiation_

先提取几个关键词，然后我们再通过例子理清这个概念：

* **裸类型参数**（类型参数即泛型，见文章开头的泛型章节介绍）
* **实例化**
* **分发到联合类型**

```typescript
// 使用上面的TypeName类型别名

// "string" | "function"
type T1 = TypeName<string | (() => void)>;

// "string" | "object"
type T2 = TypeName<string | string[]>;

// "object"
type T3 = TypeName<string[] | number[]>;
```

我们发现在上面的例子里，条件类型的推导结果都是联合类型（T3 实际上也是，只不过因为结果相同所以被合并了），并且其实就是类型参数被依次进行条件判断后，再使用`|`组合得来的结果。

是不是 get 到了一点什么？上面的例子中泛型都是裸露着的，如果被包裹着，其条件类型判断结果会有什么变化吗？我们再看另一个例子：

```typescript
type Naked<T> = T extends boolean ? "Y" : "N";
type Wrapped<T> = [T] extends [boolean] ? "Y" : "N";

// "N" | "Y"
type Distributed = Naked<number | boolean>;

// "N"
type NotDistributed = Wrapped<number | boolean>;
```

*   其中，Distributed类型别名，其类型参数（`number | boolean`）会正确的分发，即

    先分发到 `Naked<number> | Naked<boolean>`，再进行判断，所以结果是`"N" | "Y"`。
* 而 NotDistributed 类型别名，第一眼看上去感觉TS应该会自动按数组进行分发，结果应该也是 `"N" | "Y"` ？但实际上，它的类型参数（`number | boolean`）不会有分发流程，直接进行`[number | boolean] extends [boolean]`的判断，所以结果是`"N"`。

现在我们可以来讲讲这几个概念了：

* 裸类型参数，没有额外被`[]`包裹过的，就像被数组包裹后就不能再被称为裸类型参数。
* 实例化，其实就是条件类型的判断过程，就像我们前面说的，条件类型需要在收集到足够的推断信息之后才能进行这个过程。在这里两个例子的实例化过程实际上是不同的，具体会在下一点中介绍。
* 分发到联合类型：
  * 对于 TypeName，它内部的类型参数 T 是没有被包裹过的，所以 `TypeName<string | (() => void)>` 会被分发为 `TypeName<string> | TypeName<(() => void)>`，然后再次进行判断，最后分发为`"string" | "function"`。
  *   抽象下具体过程：

      ```typescript
      ( A | B | C ) extends T ? X : Y
      // 相当于
      (A extends T ? X : Y) | (B extends T ? X : Y) | (B extends T ? X : Y)

      // 使用[]包裹后，不会进行额外的分发逻辑。
      [A | B | C] extends [T] ? X : Y
      ```

      一句话概括：**没有被 `[]` 额外包装的联合类型参数，在条件类型进行判定时会将联合类型分发，分别进行判断。**

这两种行为没有好坏之分，区别只在于是否进行联合类型的分发，如果你需要走分布式条件类型，那么注意保持你的类型参数为裸类型参数。如果你想避免这种行为，那么使用 `[]` 包裹你的类型参数即可（注意在 `extends` 关键字的两侧都需要）。

### infer 关键字

在条件类型中，我们展示了如何通过条件判断来延迟确定类型，但仅仅使用条件类型也有一定不足：它无法从条件上得到类型信息。举例来说，`T extends Array<PrimitiveType> ? "foo" : "bar"`这一例子，我们不能从作为条件的 `Array<PrimitiveType>` 中获取到 `PrimitiveType` 的实际类型。

而这样的场景又是十分常见的，如获取函数返回值的类型、拆箱Promise / 数组等，因此这一节我们来介绍下 `infer` 关键字。

`infer`是 `inference` 的缩写，通常的使用方式是用于修饰作为类型参数的泛型，如： `infer R`，`R`表示 **待推断的类型**。通常 `infer` 不会被直接使用，而是与条件类型一起，被放置在底层工具类型中。如果说条件类型提供了延迟推断的能力，那么加上 `infer` 就是提供了基于条件进行延迟推断的能力。

看一个简单的例子，用于获取函数返回值类型的工具类型`ReturnType`:

```typescript
const foo = (): string => {
  return "linbudu";
};

type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// string
type FooReturnType = ReturnType<typeof foo>;
```

* `(...args: any[]) => infer R` 是一个整体，这里函数的返回值类型的位置被 `infer R` 占据了。
* 当 `ReturnType` 被调用，类型参数 T 、R 被显式赋值（T为 `typeof foo`，`infer R`被整体赋值为`string`，即函数的返回值类型），如果 T 满足条件类型的约束，就返回 infer 完毕的R 的值，在这里 R 即为函数的返回值实际类型。
*   实际上为了严谨，应当约束泛型T为函数类型，即：

    ```typescript
    // 第一个 extends 约束可传入的泛型只能为函数类型
    // 第二个 extends 作为条件判断
    type ReturnType<T extends (...args: any[]) => any> = T extends (...args: any[]) => infer R ? R : never;
    ```

`infer`的使用思路可能不是那么好习惯，我们可以用前端开发中常见的一个例子类比，页面初始化时先显示占位交互，像 `Loading` / 骨架屏，在请求返回后再去渲染真实数据。`infer`也是这个思路，**类型系统在获得足够的信息（通常来自于条件的延迟推断）后，就能将 infer 后跟随的类型参数推导出来**，最后通常会返回这个推导结果。

类似的，借着这个思路我们还可以获得函数入参类型、类的构造函数入参类型、甚至 Promise 内部的类型等，这些工具类型我们会在后面讲到。

另外，对于 TS 中函数重载的情况，使用 infer （如上面的 `ReturnType`）不会为所有重载执行推导过程，只有最后一个重载（因为一般来说最后一个重载通常是最广泛的情况）会被使用。

### 工具类型 Tool Type

这一章应该是本文“性价比”最高的一部分了，因为即使你在阅读完这部分后，还是不太懂这些工具类型是如何实现的，也不影响你把它用的恰到好处，就像 Lodash 不会要求你对每个使用的函数都熟知原理一样。

这一部分包括 **TS 内置工具类型** 与社区的 **扩展工具类型**，我个人推荐在完成学习后挑选一部分工具类型记录下来，比如你觉得比较有价值、现有或者未来业务可能会使用，或者仅仅是觉得很好玩的工具类型，并在自己的项目里新建一个`.d.ts`文件（或是 `/utils/tool-types.ts` 这样）存储它。

> **在继续阅读前，最好确保你掌握了上面的知识，它们是工具类型的基础。**

#### 内置工具类型

在上面我们已经实现了内置工具类型中被使用最多的一个:

```typescript
type Partial<T> = {
  [K in keyof T]?: T[k];
};
```

它用于将一个接口中的字段全部变为可选，除了索引类型以及映射类型以外，它只使用了`?`可选修饰符，那么我现在直接掏出小抄：

* 去除可选修饰符：`-?`，位置与 `?` 一致
* 只读修饰符：`readonly`，位置在键名，如 `readonly key: string`
* 去除只读修饰符：`-readonly`，位置同`readonly`。

恭喜，你得到了 `Required` 和 `Readonly`（去除 `readonly` 修饰符的工具类型不属于内置的，我们会在后面看到）:

```typescript
type Required<T> = {
  [K in keyof T]-?: T[K];
};

type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};
```

在上面我们实现了一个 pick 函数：

```typescript
function pick<T extends object, U extends keyof T>(obj: T, keys: U[]): T[U][] {
  return keys.map((key) => obj[key]);
}
```

类似的，假设我们现在需要从一个接口中挑选一些字段：

```typescript
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// 期望用法
// 期望结果 A["a"]类型 | A["b"]类型
type Part = Pick<A, "a" | "b">;
```

还是映射类型，只不过现在映射类型的映射源是传入给 `Pick` 的类型参数K。

既然有了`Pick`，那么自然要有`Omit`（一个是从对象中挑选部分，一个是排除部分），它和`Pick`的写法非常像，但有一个问题要解决：我们要怎么表示`T`中剔除了`K`后的剩余字段？

> Pick 选取传入的键值，Omit 移除传入的键值

这里我们又要引入一个知识点：`never`类型，它表示永远不会出现的类型，通常被用来将**收窄联合类型或是接口**，或者作为条件类型判断的兜底。详细可以看 [尤大的知乎回答](https://www.zhihu.com/search?type=content\&q=ts%20never)， 在这里我们不做展开介绍。

上面的场景其实可以简化为：

```typescript
// "3" | "4" | "5"
type LeftFields = Exclude<"1" | "2" | "3" | "4" | "5", "1" | "2">;
```

Exclude，字面意思看起来是排除，那么第一个参数应该是要进行筛选的，第二个应该是筛选条件！先按着这个思路试试：

**这里实际上使用到了分布式条件类型的特性，假设 Exclude 接收 T U 两个类型参数，T 联合类型中的类型会依次与 U 类型进行判断，如果这个类型参数在 U 中，就剔除掉它（赋值为 never）**。

> 接地气的版本：`"1"`在 `"1" | "2"` 里面吗( `"1" extends "1"|"2" -> true` )？ 在的话，就剔除掉它（赋值为`never`），不在的话就保留。

```typescript
type Exclude<T, U> = T extends U ? never : T;
```

那么 Omit就很简单了，对原接口的成员，剔除掉传入的联合类型成员，应用 Pick 即可。

```typescript
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```

剧透下，**几乎所有使用条件类型的场景，把判断后的赋值语句反一下，就会有新的场景**，比如 `Exclude` 移除掉键名，那反一下就是保留键名：

```typescript
type Extract<T, U> = T extends U ? T : never;
```

再来看个常用的工具类型 `Record<Keys, Type>`，通常用于生成以联合类型为键名（`Keys`），键值类型为`Type`的新接口，比如：

```typescript
type MyNav = "a" | "b" | "b";
interface INavWidgets {
  widgets: string[];
  title?: string;
  keepAlive?: boolean;
}
const router: Record<MyNav, INavWidgets> = {
  a: { widget: [""] },
  b: { widget: [""] },
  c: { widget: [""] },
};
```

其实很简单，把 `Keys` 的每个键值拿出来，类型规定为 `Type` 即可

```typescript
// K extends keyof any 约束K必须为联合类型
type Record<K extends keyof any, T> = {
  [P in K]: T;
};
```

注意，Record也支持 `Record<string, unknown>` 这样的使用方式， `string extends keyof any` 也是成立的，因为 `keyof` 的最终结果必然是 `string` 组成的联合类型（除了使用数字作为键名的情况...）。

在前面的 infer 一节中我们实现了用于获取函数返回值的`ReturnType`：

```typescript
type ReturnType<T extends (...args: any) => any> = T extends (
  ...args: any
) => infer R
  ? R
  : any;
```

其实把 infer 换个位置，比如放到入参处，它就变成了获取参数类型的`Parameters`:

```typescript
type Parameters<T extends (...args: any) => any> = T extends (
  ...args: infer P
) => any
  ? P
  : never;
```

如果再大胆一点，把普通函数换成类的构造函数，那么就得到了获取类构造函数入参类型的`ConstructorParameters`：

```typescript
type ConstructorParameters<
  T extends new (...args: any) => any
> = T extends new (...args: infer P) => any ? P : never;
```

> 加上`new`关键字来使其成为**可实例化类型声明**，即此处约束泛型为**类**。

这个是获得类的构造函数入参类型，如果把待 infer 的类型放到其返回处，想想 new 一个类的返回值是什么？实例！所以我们得到了实例类型`InstanceType`：

```typescript
type InstanceType<T extends new (...args: any) => any> = T extends new (
  ...args: any
) => infer R
  ? R
  : any;
```

这几个例子看下来，你应该已经 get 到了那么一丝天机，类型编程的确没有特别高深晦涩的语法，它考验的是你对其中基础部分如**索引**、**映射**、**条件类型**的掌握程度，以及举一反三的能力。下面我们要学习的社区工具类型，本质上还是各种基础类型的组合，只是从常见场景下出发，补充了官方没有覆盖到的部分。

#### 社区工具类型

> 这一部分的工具类型大多来自于[utility-types](https://github.com/piotrwitek/utility-types)，其作者同时还有 [react-redux-typescript-guide](https://github.com/piotrwitek/react-redux-typescript-guide) 和 [typesafe-actions ](https://github.com/piotrwitek/typesafe-actions)这两个优秀作品。
>
> 同时，也推荐 [type-fest](https://github.com/sindresorhus/type-fest) 这个库，和上面相比更加接地气一些。其作者的作品...，我保证你直接或间接的使用过（如果不信，一定要去看看，我刚看到的时候是真的震惊的不行）。

我们由浅入深，先封装基础的类型别名和对应的类型守卫：

```typescript
export type Primitive =
  | string
  | number
  | bigint
  | boolean
  | symbol
  | null
  | undefined;

export const isPrimitive = (val: unknown): val is Primitive => {
  if (val === null || val === undefined) {
    return true;
  }

  const typeDef = typeof val;

  const primitiveNonNullishTypes = [
    "string",
    "number",
    "bigint",
    "boolean",
    "symbol",
  ];

  return primitiveNonNullishTypes.indexOf(typeDef) !== -1;
};

export type Nullish = null | undefined;

export type NonUndefined<A> = A extends undefined ? never : A;
// 实际上TS也内置了这个工具类型
type NonNullable<T> = T extends null | undefined ? never : T;
```

> `Falsy` 和 `isFalsy` 我们已经在上面体现过了。

趁着对 infer 的记忆来热乎，我们再来看一个常用的场景，提取 Promise 的实际类型：

```typescript
const foo = (): Promise<string> => {
  return new Promise((resolve, reject) => {
    resolve("linbudu");
  });
};

// Promise<string>
type FooReturnType = ReturnType<typeof foo>;

// string
type NakedFooReturnType = PromiseType<FooReturnType>;
```

如果你已经熟练掌握了`infer`的使用，那么实际上是很好写的，只需要用一个`infer`参数作为 Promise 的泛型即可：

```typescript
export type PromiseType<T extends Promise<any>> = T extends Promise<infer U>
  ? U
  : never;
```

使用 `infer R` 来等待类型系统推导出`R`的具体类型。

#### 递归的工具类型

前面我们写了个`Partial` `Readonly` `Required`等几个对接口字段进行修饰的工具类型，但实际上都有局限性，如果接口中存在着嵌套呢？

```typescript
type Partial<T> = {
  [P in keyof T]?: T[P];
};
```

理一下逻辑：

* 如果不是对象类型，就只是加上`?`修饰符
* 如果是对象类型，那就**遍历这个对象内部**
* 重复上述流程。

是否是对象类型的判断我们见过很多次了, `T extends object`即可，那么如何遍历对象内部？实际上就是递归。

```typescript
export type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};
```

> `utility-types`内部的实现实际比这个复杂，还考虑了数组的情况，这里为了便于理解做了简化，后面的工具类型也同样存在此类简化。

那么`DeepReadobly`、 `DeepRequired`也就很简单了：

```typescript
export type DeepMutable<T> = {
  -readonly [P in keyof T]: T[P] extends object ? DeepMutable<T[P]> : T[P];
};

// 即DeepReadonly
export type DeepImmutable<T> = {
  +readonly [P in keyof T]: T[P] extends object ? DeepImmutable<T[P]> : T[P];
};

export type DeepRequired<T> = {
  [P in keyof T]-?: T[P] extends object | undefined ? DeepRequired<T[P]> : T[P];
};
```

尤其注意下`DeepRequired`，它的条件类型判断的是 `T[P] extends object | undefined`，因为嵌套的对象类型可能是可选的（undefined），如果仅使用object，可能会导致错误的结果。

另外一种省心的方式是不进行条件类型的判断，直接全量递归所有属性\~

#### 返回键名的工具类型

在有些场景下我们需要一个工具类型，它返回接口字段键名组成的联合类型，然后用这个联合类型进行进一步操作（比如给 Pick 或者 Omit 这种使用），一般键名会符合特定条件，比如：

* 可选/必选/只读/非只读的字段
* （非）对象/（非）函数/类型的字段

来看个最简单的函数类型字段`FunctionTypeKeys`：

```typescript
export type FunctTypeKeys<T extends object> = {
  [K in keyof T]-?: T[K] extends Function ? K : never;
}[keyof T];
```

`{[K in keyof T]: ... }[keyof T]`这个写法可能有点诡异，拆开来看：

```typescript
interface IWithFuncKeys {
  a: string;
  b: number;
  c: boolean;
  d: () => void;
}

type WTFIsThis<T extends object> = {
  [K in keyof T]-?: T[K] extends Function ? K : never;
};

type UseIt1 = WTFIsThis<IWithFuncKeys>;
```

很容易推导出 `UseIt1` 实际上就是：

```typescript
type UseIt1 = {
  a: never;
  b: never;
  c: never;
  d: "d";
};
```

> `UseIt`会保留所有字段，满足条件的字段其键值为字面量类型（即键名），不满足的则为never。

加上后面一部分：

```typescript
// "d"
type UseIt2 = UseIt1[keyof UseIt1];
```

这个过程类似排列组合：`never`类型的值不会出现在联合类型中

> ```typescript
> // never类型会被自动去除掉 string | number
> type WithNever = string | never | number;
> ```

所以`{ [K in keyof T]: ... }[keyof T]`这个写法实际上就是为了返回键名（准备的说，是**键名组成的联合类型**）。

那么非函数类型字段也很简单了，这里就不做展示了，下面来看可选字段`OptionalKeys`与必选字段`RequiredKeys`，先来看个小例子：

```typescript
type WTFAMI1 = {} extends { prop: number } ? "Y" : "N";
type WTFAMI2 = {} extends { prop?: number } ? "Y" : "N";
```

如果能绕过来，很容易就能得出来答案。如果一时没绕过去，也很简单，对于前面一个情况，`prop`是必须的，因此空对象 `{}` 并不能满足`extends { prop: number }`，而对于prop为可选的情况下则可以。

因此，我们使用这种思路来得到可选/必选的键名。

* `{} extends Pick<T, K>`，如果`K`是可选字段，那么就留下（OptionalKeys，如果是 RequiredKeys 就剔除）。
* 怎么剔除？当然是用`never`了。

```typescript
export type RequiredKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? never : K;
}[keyof T];
```

这里是剔除可选字段，那么 OptionalKeys 就是保留了：

```typescript
export type OptionalKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? K : never;
}[keyof T];
```

只读字段`IMmutableKeys`与非只读字段`MutableKeys`的思路类似，即先获得：

```typescript
interface MutableKeys {
  readonlyKeys: never;
  notReadonlyKeys: "notReadonlyKeys";
}
```

然后再获得不为`never`的字段名即可。

这里还是要表达一下对作者的敬佩，属实巧妙啊，首先定义一个工具类型`IfEqual`，比较两个类型是否相同，甚至可以比较修饰前后的情况下，也就是这里只读与非只读的情况。

```typescript
type Equal<X, Y, A = X, B = never> = (<T>() => T extends X ? 1 : 2) extends <
  T
>() => T extends Y ? 1 : 2
  ? A
  : B;
```

* 不要被`<T>() => T extends X ? 1 : 2`干扰，可以理解为就是用于比较的包装，这一层包装能够区分出来只读与非只读属性。即 `(<T>() => T extends X ? 1 : 2)` 这一部分，只有在类型参数 `X` 完全一致时，两个 `(<T>() => T extends X ? 1 : 2)` \` 才会是全等的，这个一致要求只读性、可选性等修饰也要一致。
* 实际使用时（以非只读的情况为例），我们为 X 传入接口，为 Y 传入去除了只读属性`-readonly`的接口，使得所有键都被进行一次与去除只读属性的键的比较。为 A 传入字段名，B 这里我们需要的就是 never，因此可以不填。

实例：

```typescript
export type MutableKeys<T extends object> = {
  [P in keyof T]-?: Equal<
    { [Q in P]: T[P] },
    { -readonly [Q in P]: T[P] },
    P,
    never
  >;
}[keyof T];
```

几个容易绕弯子的点：

* 泛型 Q 在这里不会实际使用，只是映射类型的字段占位。
* X 、 Y 同样存在着 **分布式条件类型**， 来依次比对字段去除 readonly 前后。

同样的有：

```typescript
export type IMmutableKeys<T extends object> = {
  [P in keyof T]-?: Equal<
    { [Q in P]: T[P] },
    { -readonly [Q in P]: T[P] },
    never,
    P
  >;
}[keyof T];
```

* 这里不是对`readonly`修饰符操作，而是调换条件类型的判断语句。

#### 基于值类型的 Pick 与 Omit

前面我们实现的 Pick 与 Omit 是基于键名的，假设现在我们需要按照值类型来做选取剔除呢？

其实很简单，就是`T[K] extends ValueType`即可：

```typescript
export type PickByValueType<T, ValueType> = Pick<
  T,
  { [Key in keyof T]-?: T[Key] extends ValueType ? Key : never }[keyof T]
>;

export type OmitByValueType<T, ValueType> = Pick<
  T,
  { [Key in keyof T]-?: T[Key] extends ValueType ? never : Key }[keyof T]
>;
```

> 条件类型承担了太多...

#### 工具类型一览

总结下我们上面书写的工具类型：

* 全量修饰接口：`Partial` `Readonly(Immutable)` `Mutable` `Required`，以及对应的递归版本。
* 裁剪接口：`Pick` `Omit` `PickByValueType` `OmitByValueType`
* 基于 infer：`ReturnType` `ParamType` `PromiseType`
* 获取指定条件字段：`FunctionKeys` `OptionalKeys` `RequiredKeys` ...

**需要注意的是，有时候单个工具类型并不能满足你的要求，你可能需要多个工具类型协作**，比如用 `FunctionKeys` + `Pick` 得到一个接口中类型为函数的字段。

**另外，实际上上面的部分工具类型是可以用重映射能力实现的更加简洁优雅的，这不尝试下？**

受限于篇幅（本文到这里已经1.3w字了），本来还想放上来的 type-fest 的工具类型就只能遗憾退场了，但我还是建议大家去读一读它的源码。相比于上面的 utility-types 更加接地气，实现思路也更加有趣。

### TypeScript 4.x 中的部分新特性

这一部分是相对于之前的版本新增的部分，主要包括了4.1 - 4.4（Beta）版本中引入的一部分与本文介绍内容有关的新特性，包括 模板字面量类型 与 重映射。

#### 模板字面量类型

[TypeScript 4.1](https://devblogs.microsoft.com/typescript/announcing-typescript-4-1/) 中引入了模板字面量类型，使得我们可以使用`${}` 这一语法来构造字面量类型，如：

```typescript
type World = 'world';

// "hello world"
type Greeting = `hello ${World}`;
```

模板字面量类型同样支持分布式条件类型，如：

```typescript
export type SizeRecord<Size extends string> = `${Size}-Record`

// "Small-Record"
type SmallSizeRecord = SizeRecord<"Small">
// "Middle-Record"
type MiddleSizeRecord = SizeRecord<"Middle">
// "Huge-Record"
type HugeSizeRecord = SizeRecord<"Huge">


// "Small-Record" | "Middle-Record" | "Huge-Record"
type UnionSizeRecord = SizeRecord<"Small" | "Middle" | "Huge">
```

还有个有趣的地方，模板插槽（`${}`）中可以传入联合类型，并且同一模板中如果存在多个插槽，各个联合类型将会被分别排列组合。

```typescript
// "Small-Record" | "Small-Report" | "Middle-Record" | "Middle-Report" | "Huge-Record" | "Huge-Report"
type SizeRecordOrReport = `${"Small" | "Middle" | "Huge"}-${"Record" | "Report"}`;
```

随之而来的还有四个新的工具类型：

```typescript
type Uppercase<S extends string> = intrinsic;

type Lowercase<S extends string> = intrinsic;

type Capitalize<S extends string> = intrinsic;

type Uncapitalize<S extends string> = intrinsic;
```

它们的作用就是字面意思，不做解释了。相关的PR见 [40336](https://github.com/microsoft/TypeScript/pull/40336)，作者Anders Hejlsberg 是 C# 与 Delphi 的首席架构师，同时也是TS的作者之一。

`intrinsic`代表了这些工具类型是由 TS 编译器内部实现的，其实也很好理解，我们无法通过类型编程来改变字面量的值，但我想按照这个趋势，TS类型编程以后会支持调用 Lodash 方法也说不定。

> TS 的实现代码：
>
> ```typescript
> function applyStringMapping(symbol: Symbol, str: string) {
> switch (intrinsicTypeKinds.get(symbol.escapedName as string)) {
>   case IntrinsicTypeKind.Uppercase: return str.toUpperCase();
>   case IntrinsicTypeKind.Lowercase: return str.toLowerCase();
>   case IntrinsicTypeKind.Capitalize: return str.charAt(0).toUpperCase() + str.slice(1);
>   case IntrinsicTypeKind.Uncapitalize: return str.charAt(0).toLowerCase() + str.slice(1);
> }
> return str;
> }
> ```

你可能会想到，模板字面量如果想截取其中的一部分要怎么办？这里可没法调用 slice 方法。其实思路就在我们上面提到过的 infer，使用 infer 占位后，便能够提取出字面量的一部分，如：

```typescript
type CutStr<Str extends string> = Str extends `${infer Part}budu` ? Part : never

// "lin"
type Tmp = CutStr<"linbudu">
```

再进一步，`[1,2,3]`这样的字符串，如果我们提供 `[${infer Member1}, ${infer Member2}, ${infer Member}]` 这样的插槽匹配，就可以实现神奇的提取字符串数组成员效果：

```typescript
type ExtractMember<Str extends string> = Str extends `[${infer Member1}, ${infer Member2}, ${infer Member3}]` ? [Member1, Member2, Member3] : unknown;

// ["1", "2", "3"]
type Tmp = ExtractMember<"[1, 2, 3]">
```

注意，这里的模板插槽被使用 `,` 分隔开了，如果多个带有 infer 的插槽紧挨在一起，那么前面的 infer 只会获得单个字符，最后一个 infer 会获得所有的剩余字符（如果有的话），比如我们把上面的例子改成这样：

```typescript
type ExtractMember<Str extends string> = Str extends `[${infer Member1}${infer Member2}${infer Member3}]` ? [Member1, Member2, Member3] : unknown;

// ["1", ",", " 2, 3"]
type Tmp = ExtractMember<"[1, 2, 3]">
```

这一特性使得我们可以使用多个相邻的 infer + 插槽，对最后一个 infer获得的值进行递归操作，如：

```typescript
type JoinArrayMember<T extends unknown[], D extends string> =
  T extends [] ? '' :
  T extends [any] ? `${T[0]}` :
  T extends [any, ...infer U] ? `${T[0]}${D}${JoinArrayMember<U, D>}` :
  string;

// ""
type Tmp1 = JoinArrayMember<[], '.'>;
// "1"
type Tmp3 = JoinArrayMember<[1], '.'>;
// "1.2.3.4"
type Tmp2 = JoinArrayMember<[1, 2, 3, 4], '.'>;
```

原理也很简单，每次将数组的第一个成员添加上`.`，在最后一个成员时不作操作，在最后一次匹配（`[]`）返回空字符串，即可。

又或者反过来？把 `1.2.3.4` 回归到数组形式？

```typescript
type SplitArrayMember<S extends string, D extends string> =
  string extends S ? string[] :
  S extends '' ? [] :
  S extends `${infer T}${D}${infer U}` ? [T, ...SplitArrayMember<U, D>] :
  [S];

type Tmp11 = SplitArrayMember<'foo', '.'>;  // ['foo']
type Tmp12 = SplitArrayMember<'foo.bar.baz', '.'>;  // ['foo', 'bar', 'baz']
type Tmp13 = SplitArrayMember<'foo.bar', ''>;  // ['f', 'o', 'o', '.', 'b', 'a', 'r']
type Tmp14 = SplitArrayMember<any, '.'>;  // stri 

```

最后，看到 `a.b.c` 这样的形式，你应该想到了 Lodash 的 get 方法，即通过 `get({},"a.b.c")` 的形式快速获得嵌套属性。但是这样要怎么提供类型声明？有了模板字面量类型后，只需要结合 infer + 条件类型即可。

```typescript
type PropType<T, Path extends string> =
    string extends Path ? unknown :
    Path extends keyof T ? T[Path] :
    Path extends `${infer K}.${infer R}` ? K extends keyof T ? PropType<T[K], R> : unknown :
    unknown;

declare function getPropValue<T, P extends string>(obj: T, path: P): PropType<T, P>;
declare const s: string;

const obj = { a: { b: {c: 42, d: 'hello' }}};
getPropValue(obj, 'a');  // { b: {c: number, d: string } }
getPropValue(obj, 'a.b');  // {c: number, d: string }
getPropValue(obj, 'a.b.d');  // string
getPropValue(obj, 'a.b.x');  // unknown
getPropValue(obj, s);  // unknown
```

#### 重映射

这一能力在 [TS 4.1](https://devblogs.microsoft.com/typescript/announcing-typescript-4-1/#key-remapping-mapped-types) 中引入，提供了在映射类型中重定向映射源至新类型的能力，这里的新类型可以是工具类型的返回结果、字面量模板类型等，用于解决在使用映射类型时，我们想要过滤/新增拷贝的接口成员，通常会将原接口成员的键作为新的转换方法参数，如：

```typescript
type Getters<T> = {
    [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K]
};

interface Person {
    name: string;
    age: number;
    location: string;
}

type LazyPerson = Getters<Person>;
```

转换后的结果：

```typescript
type LazyPerson = {
    getName: () => string;
    getAge: () => number;
    getLocation: () => string;
}
```

> 这里的 `string & k` 是因为重映射的转换方法（即 `as` 后面的部分）必须是可分配给 `string | number | symbol` 的，而 K 来自于 `keyof`，可能包含 `symbol` 类型，这样的话是不能交给模板字面量类型使用的。

如果转换方法返回了never，那么这个成员就被除去了，所以我们可以使用这个方法来过滤掉成员。

```typescript
type RemoveKindField<T> = {
    [K in keyof T as Exclude<K, "kind">]: T[K]
};

interface Circle {
    kind: "circle";
    radius: number;
}

// type KindlessCircle = {
//     radius: number;
// }
type KindlessCircle = RemoveKindField<Circle>;
```

最后，当与模板字面量一同使用时，由于其排列组合的特性，如果重映射的转换方法是一个由 模板字面量类型 组成的 联合类型，那么就会从排列组合得到多个成员。

```typescript
type DoubleProp<T> = { [P in keyof T & string as `${P}1` | `${P}2`]: T[P] }
type Tmp = DoubleProp<{ a: string, b: number }>;  // { a1: string, a2: string, b1: number, b2: number }
```

### 尾声

这篇文章确实很长很长，因不建议一次性囫囵吞枣的读完，建议选取几段有一定长度的连续时间，给它掰开了揉碎了好好读懂。写文不易，尤其是写这么长的文章，但是如果能帮助你的 TypeScript 更上一层楼，就完全值得了。

如果在之前，你从未关注过类型编程方面，那么阅读完毕后可能需要一定时间来适应思路的转变。还是那句话，认识到 **类型编程的本质也是编程**。当然，你也可以渐进式的开始实践这一点，比如从今天开始，从现在手头里的项目开始，从泛型到类型守卫，从索引/映射类型到条件类型，从使用工具类型到封装工具类型，一步步变成 TypeScript 高高手。
