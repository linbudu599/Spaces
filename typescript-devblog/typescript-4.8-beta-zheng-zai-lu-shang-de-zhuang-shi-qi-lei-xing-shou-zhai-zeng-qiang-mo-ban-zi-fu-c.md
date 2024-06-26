---
description: '2022-06-25'
---

# TypeScript 4.8 beta：正在路上的装饰器、类型收窄增强、模板字符串类型中的 infer

TypeScript 已于 2022.06.21 发布 4.8 beta 版本，你可以在 [4.8 Iteration Plan](https://github.com/microsoft/TypeScript/issues/49074) 查看所有被包含的 Issue 与 PR。如果想要抢先体验新特性，执行：

```bash
$ npm install typescript@beta
```

来安装 beta 版本的 TypeScript，或在 VS Code 中安装 [**JavaScript and TypeScript Nightly**](http://link.zhihu.com/?target=https%3A//marketplace.visualstudio.com/items%3FitemName%3Dms-vscode.vscode-typescript-next) 来更新内置的 TypeScript 支持。

本篇是笔者的第四篇 TypeScript 更新日志，上一篇是 「TypeScript 4.7 beta 发布：NodeJs 的 ES Module 支持、新的类型编程语法、类型控制流分析增强等」，你可以在此账号的创作中找到，接下来笔者也将持续更新 TypeScript 的 DevBlog 相关，感谢你的阅读。

> 另外，由于 beta 版本与正式版本通常不会有明显的差异，这一系列只会介绍 beta 版本而非正式版本。

### 正在路上的装饰器

在 4 月份的 TC39 双月会议上，装饰器提案成功进入到 Stage 3，这也是装饰器提案历经数个版本以后离 Stage 4 最近的一次。TypeScript 中同样大量使用了装饰器相关的语法，但实际上 TS 中的装饰器（experimental）、Babel 中的装饰器（legacy）都是基于第一版的装饰器提案进行实现的，而目前 TC39 中的装饰器提案已经迭代到了第三版。

> 如果你有兴趣了解更多装饰器的历史，可以阅读笔者的 [走近 MidwayJS：初识 TS 装饰器与 IoC 机制](https://juejin.cn/post/6859314697204662279) 中的介绍，或者贺师俊老师在 [是否应该在 production 里使用 typescript 的 decorator？](https://www.zhihu.com/question/404724504) 的回答。

随着新版装饰器提案的更新，TypeScript 势必也需要对应地进行支持，但由于其工作量较大，目前 4.8 beta 版本中并不包含新版装饰器的相关介绍（所以说正在路上），但其具体功能必然与装饰器提案 [proposal-decorators](https://github.com/tc39/proposal-decorators) 中的介绍基本是一致的。

虽然我们迎来了新版装饰器，但也无需担心旧版装饰器从此就被扫进历史的尘埃里了，对旧版装饰器的支持肯定是会被保留相当长一段时间的，语言支持、框架改进、用户接受，每一步都快不起来。我们可能要到 TypeScript 20.0 beta 版本中才会看到官方宣布将废弃对实验性装饰器的支持，希望那时笔者仍然在更新此专栏。

对于使用者来说，基本不用担心额外的学习成本，新版装饰器在大部分情况下能完全覆盖掉旧版装饰器的能力。但对于框架基础库开发者来说，两个版本装饰器之间的差异确实相当大，如装饰器的运行顺序以及元数据相关等。

如果你有兴趣了解新版装饰器的详细使用，可以阅读笔者此前发表的 [ECMAScript 双月报告：装饰器提案进入 Stage 3](https://mp.weixin.qq.com/s/6PTcjJQTED3WpJH8ToXInw) ，了解新版装饰器的功能、旧版装饰器的废弃原因，以及新版装饰器如何不通过反射元数据的方式实现依赖注入。

### 交叉类型与联合类型的类型收窄增强

TypeScript 4.8 版本对 `--strictNullChecks` 进行了进一步增强，主要体现在联合类型与交叉类型，以及类型收窄地表现上。

举例来说，作为 TypeScript 类型系统中的 Top Type ，unknown 类型包含所有的其他类型，实际上 unknown 和 `{} | null | undefined` 的效果是一致的：独特意义的 null、undefined 类型，加上万物起源的 `{}`。

为什么说 `{}` 是万物起源？基于 TypeScript 的结构化类型比较，两个类型之间的兼容性是通过它们内部的属性类型是否一致来比较的：

```typescript
class Cat {
  eat() {}
}

class Dog {
  eat() {}
}

function feedCat(cat: Cat) {}

feedCat(new Dog());
```

在这个例子中 feedCat 函数可以接受 Dog 类型的参数，原因就是 Dog 类型与 Cat 类型在结构化类型系统比较下被认为是一致的。

更进一步，如果此时 Dog 新增一个方法：

```typescript
class Cat {
  eat() {}
}

class Dog {
  eat() {}
  bark() {}
}

function feedCat(cat: Cat) {}

feedCat(new Dog());
```

此时这个例子仍然成立，原因就在于此时 Dog 类型相比 Cat 类型多了一个属性，在结构化类型系统的判断下可以认为 Dog 类型是 Cat 类型的子类型，就像这样：

```typescript
class Dog extends Cat {
  bark() {}
}
```

回到正题，由于 `{}` 就是一个空对象，因此**除 null、undefined 以外的一切基础类型，都可以被视为是继承于 `{}` 之后派生出来的**。

在 4.8 版本，现在 `unknown` 和 `{} | null | undefined` 可以互相兼容：

```typescript
declare let v1: unknown;
declare let v2: {} | null | undefined;

v1 = v2;
// 此前会报错，因为认为 unknown 包含的类型信息更多
v2 = v1;
```

同时，对于 `{}`，4.8 版本会将使用 `{}` 的交叉类型，如 `obj & {}` 直接简化为 `obj` 类型，前提是 `obj` 并非来自于泛型，且非 null / undefined。这是因为交叉类型要求同时满足两个类型，而只要 `obj` 不是 null / undefined 类型，就可以认为必定也符合 `{}` 类型，因此可以直接将 `{}` 从交叉类型中移除：

```typescript
type T1 = {} & string; // string
type T2 = {} & 'linbudu'; // 'linbudu'
type T3 = {} & object; // object
type T4 = {} & { x: number }; // { x: number }
type T5 = {} & null; // never
type T6 = {} & undefined; // never
```

而基于这一改动，你现在可以使用 `{}` 来剔除原类型中的 null 与 undefined，即原本的内置工具类型 NonNullable 实现会被更改为以下这种：

```typescript
type _NonNullable<T> = T extends null | undefined ? never : T;
type NonNullable<T> = T & {};
```

实现原理则是 `null & {}`、`undefined & {}` 会直接被判断为 never ，从而消失在联合类型结果中。

从 NonNullable 的实现变化我们可以知道，现在如果一个值不是 null 也不是 undefined ，那么它的值其实就等于它和 `{}` 进行交叉的值，也就说我们能写出以下的代码：

```typescript
function throwIfNullable<T>(value: T): NonNullable<T> {
  if (value === undefined || value === null) {
    throw Error('Nullable value!');
  }
  return value;
}
```

在过去，这个例子会抛出一个错误：类型 T 不能赋值给 `NonNullable<T>` 的类型，而现在我们知道如果剔除了 null 与 undefined ，那么 T 其实等价于 `T & {}`，也即 `NonNullable<T>` 。

最后，由于这些改动，现在 TypeScript 的类型控制流分析也得到了进一步的增强，现在 unknown 类型的变量会被完全视为 `{} | null | undefined`，因此会在 if else 的 truthy 分支中被收窄为 `{}`：

```typescript
function narrowUnknown(x: unknown) {
  if (x) {
    x; // {}
  } else {
    x; // unknown
  }
}
```

### 模板字符串类型中的 infer 提取

在 4.7 版本中 TypeScript 支持了 infer extends 语法，使得我们可以直接一步就 infer 到预期类型的值，而不需要再次进行条件语句判断：

```typescript
type FirstString<T> =
    T extends [infer S, ...unknown[]]
        ? S extends string ? S : never
        : never;
​
// 基于 infer extends
type FirstString<T> =
    T extends [infer S extends string, ...unknown[]]
        ? S
        : never;
```

4.8 版本在此基础上进行了进一步地增强，当 infer 被约束为一个原始类型，那么它现在会尽可能将 infer 的类型信息推导到字面量类型的级别：

```typescript
// 此前为 number，现在为 '100'
type SomeNum = "100" extends `${infer U extends number}` ? U : never;

// 此前为 boolean，现在为 'true'
type SomeBool = "true" extends `${infer U extends boolean}` ? U : never;
```

同时，TypeScript 会检查提取出的值能否重新映射回初始的字符串，如 SomeNum 中会检查 `String(Number("100"))` 是否等于 `"100"`，在下面这个例子中就是因为无法重新映射回去，而导致只能推导到 number 类型：

```typescript
// String(Number("1.0")) → "1"，≠ "1.0"
type JustNumber = "1.0" extends `${infer T extends number}` ? T : never;
```

### 绑定类型中的类型推导

TypeScript 中的泛型填充也会受到其调用方的影响，如以下示例：

```typescript
declare function chooseRandomly<T>(x: T): T;

const res1 = chooseRandomly(['linbudu', 599, false]);
```

此时 res1 的类型与函数的泛型 T 会被推导为 `Array<string | number | boolean>`，但如果我们换一个方法：

```typescript
declare function chooseRandomly<T>(x: T): T;
const [a, b, c] = chooseRandomly(['linbudu', 599, false]);
```

此时 a、b、c 被推导为了 string、number、boolean 类型，也就是说此时函数的泛型被填充为 `[string, number, boolean]` 这么个元组类型。

这一泛型填充方式被称为绑定模式（Binding Pattern），而在 4.8 版本中，禁用了基于绑定模式的类型推导，因为其对泛型的影响并不始终正确：

```typescript
declare function f<T>(x?: T): T;
const [x, y, z] = f();
```

在这一例子中，`[x, y, z]` 的绑定模式强行将泛型参数填充为了 `[any, any, any]`，而这是非常不合理的——你怎么能确定我是个数组结构？

### 对象字面量值与数组字面量值的全等比较提示

我们知道在 JavaScript 中 `{} === {}` 是不成立的，因为对象使用引用地址存储，而这实际上是在比较两个不同的引用地址。为了更好地避免代码中错误使用 `===` 来比较对象和数组类型，TypeScript 现在会进行错误提示：

```typescript
const obj = {};
// 此语句始终将返回 false，因为 JavaScript 中使用引用地址比较对象，而非实际值
if (obj === {}) {
}
```

类似的，此前如果你在 if 语句中错误地使用了函数，TypeScript 也会给你一个提示：

```typescript
const func = () => {};
// 此表达式将始终返回 true，你是否想要调用 func ？
if (func) {
}
```

### Compiler 优化

4.8 版本还对 tsc 进行了一些性能优化工作，包括监听模式 `--watch`，增量构建 `--incremental` 以及 Project References 下的 `--build` 模式。如现在监听模式下，会跳过那些不是因为用户操作而导致变更的文件。

你可以阅读 [#48784](https://github.com/microsoft/TypeScript/pull/48784) 了解更多优化信息。

### 破坏性变更

* lib.d.ts 更新
*   JavaScript 文件中不再允许对类型的导入，此前你可以导入一个类型来作为 JSDoc 描述：

    ```javascript
    import { IConfiguration } from 'foo';
    ​
    /**
     * @type {IConfiguration}
     */
    export const config = {};
    ```

    这一使用方式现在会抛出一个错误，实际上，更常见的方式是这样的：

    ```javascript
    /**
     * @type {import("foo").IConfiguration}
     */
    export const config = {};
    ​
    // CommonJs 下：
    module.exports = /** @type {import("foo").IConfiguration} */  {}
    ```

    而对于 JavaScript 文件中的类型导出，你可以使用 `@typedef` 来声明一个类型导出：

    ```javascript
    /**
     * @typedef {string | number} MyType
     */
    export { MyType }
    ​
    // 现在的使用
    /**
      * @typedef {string | number} FooType
     */
    ```

全文完，我们 4.9 beta 版本见:-)。

