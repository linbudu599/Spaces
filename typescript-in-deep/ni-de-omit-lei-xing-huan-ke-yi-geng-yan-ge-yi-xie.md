---
description: '2022-02-26'
---

# 你的 Omit 类型还可以更严格一些

> 本文是对在极客时间 与 [早早聊](https://www.zaozao.run/conf/c37) 的直播 中提到的 **Omit 工具类型** 的进一步说明，但你不需要已经观看过相关直播，本文会包括前置知识部分。

### Pick 与 Omit

Pick 与 Omit 都是 TypeScript 内置的工具类型，它们的作用类似，都是对接口做剪裁，如

```typescript
interface Foo {
	a: number;
	b: string;
	c: boolean;
}

// { a:number; }
type OnlyA = Pick<Foo, "a">;

// { b: string; c: boolean}
type ExcludeA = Omit<Foo, "a">;
```

它们俩的功能是相反的，这其实代表了 TypeScript 类型编程中的一个概念：基本上所有的工具类型都有其反向实现，其产生方式通常有两种，对一个工具类型进行简单的条件更改就能得到另一个功能相反的工具类型（如 Partial 与 Required），或反向类型基于正向类型实现（即 Pick 与 Omit）。

我们直接看 Pick 与 Omit 的实现：

```typescript
type Pick<T, K extends keyof T> = {
    [P in K]: T[P];
};

type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```

Pick 就是简单的使用了索引类型（索引签名与索引类型访问）与映射类型、keyof操作符，这里不做展开介绍。

> keyof 操作符常和接口结构一起使用，得到一组对象键值的字面量类型组成的联合类型，如 'a'|'b'|'c'。我们也常用 `keyof any` 表示成员未知的联合类型。

Omit 的实现有趣一些，它基于 Pick 类型实现，相反的，其第二个泛型参数 K 在传入 Pick 类型时进行了一次额外的转换，`Exclude<keyof T, K>`可能有点绕，但实际上：

```typescript
type Exclude<T, U> = T extends U ? never : T;
```

Exclude\<Set1, Set2> 的作用就是得到 Set1 相对于 Set2 的差集，即：

* Exclude<'a'|'b'|'c', 'a'|'d'|'e'> 的结果为 'b'|'c'，即 Set1 中有，而 Set2 中没有的部分。

> 关于通过分布式条件类型实现差集、补集、并集、交集及将其应用扩展到二维对象类型，见此账号同期发布的 「分布式条件类型全知」。

再回到 Pick 和 Omit 的场景，Exclude\<keyof Foo, 'a'> 的结果即为 'b'|'c'，相当于与 Pick 的思路反向而行，这样我们就得到了 Omit。

再看上面的 Omit 类型，你会发现一个很奇怪的地方：明明它们的作用都是裁剪对象，那么第二个泛型参数 K 应当被约束为 keyof T 才对，为什么只有 Pick 被约束了？

### 为什么 Omit 放水了？

> 你可以在 [#30825](https://github.com/microsoft/TypeScript/issues/30825) 中阅读更多讨论，以下部分只摘取其中的部分探讨与笔者自己的见解。

Omit 工具类型在 [TypeScript 3.5版本](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-5.html#the-omit-helper-type) 中被引入，就是我们现在看到的宽松版本。而实际上，最初的 PR [#30455](https://github.com/microsoft/TypeScript/issues/30455) 中，Omit 的实现的确是严格的：

```typescript
type Omit<ObjectType, KeysType extends keyof ObjectType> = Pick<ObjectType, Exclude<keyof ObjectType, KeysType>>;
```

但 [Daniel Rosenwasser](https://github.com/DanielRosenwasser) (TypeScript 的 PM，也是现在每一期 DevBlog 的写作者) 最终引入的实现却移除了严格约束，见 [#30552](https://github.com/Microsoft/TypeScript/pull/30552)。

关于做出决定的原因，Daniel 解释到主要是因为当时已有的 Exclude 工具类型，同样没有限制第二个参数需要为第一个参数的子集：

```typescript
type Exclude<T, U> = T extends U ? never : T;
```

因此团队成员认为对于 Omit 类型也不应该做此类的限制。

先插一下我个人的见解，我认为这两种情况其实并不一致，对于 Exclude ，即我们认为差集，在数学层面上差集并不要求此限制，即：

![差集](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3c9f5c1fbd44826bf75731d8b7ec72d\~tplv-k3u1fbpfcp-zoom-1.image)

如果我们要求参数2为参数1的子集，则其应该被称为补集（Complement）：

```typescript
export type Complement<A, B extends A> = Exclude<A, B>;
```

![补集](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd2fed6e003b4b7294b27f8cf7a9579b\~tplv-k3u1fbpfcp-zoom-1.image)

而 Omit 类型其实更贴近补集的情况，我们能够要被移除的部分就是原对象的子集，那么其 OmitKeys 也应该被约束才对。

在 #30825 中，npm之神 [Sindre Sorhus](https://github.com/sindresorhus) 也加入了讨论（为什么说之神呢，因为你用的 npm 包大概率底层直接或间接依赖了他的开源包），他指出在许多 TypeScript 类型工具库中，基本不会直接使用内置的 Omit 类型，而是自己实现一个严格版本。这些工具库包括 type-zoo、type-fest（目前最流行的类型库，也是 Sindre Sorhus 的作品）、utility-types 等。

而 TS 团队成员认为，如果将 Omit 更改为严格版本，会导致很多 DefinitelyTyped （@types/xxx 这种）的包出现问题，因此，**既然始终不能让所有人都满意，还是保持原有实现为好。**

而另一位团队核心成员 Ryan Cavanaugh 也指出，并不是在所有情况下此约束都会带来更好的效果，比如我们需要使用 keyof Obj2 来剔除 Obj1：

```typescript
type Omit1<T, K> = Pick<T, Exclude<keyof T, K>>;
type Omit2<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;

// 这里就不能用严格 Omit 了
declare function combineSpread<T1, T2>(obj: T1, otherObj: T2, rest: Omit1<T1, keyof T2>): void;

type Point3d = { x: number, y: number, z: number };

declare const p1: Point3d;

// 能够检测出错误，rest 中缺少了 y
combineSpread(p1, { x: 10 }, { z: 2 });
```

> 认真地说，我认为这其实不是 Omit 类型应该做的事，叫它 Remove 可能更合适...

后面还有很多很多脑洞大开的讨论，比如，通过 `lib:['omit.loose']` / `lib:['omit.strict']` 来显式控制行为，通过 TypeScript ESLint 的 [Ban Types](https://github.com/typescript-eslint/typescript-eslint/blob/main/packages/eslint-plugin/docs/rules/ban-types.md) 规则禁用掉对 Omit 类型的使用等等。

当然，既然我们今天看到的 Omit 类型还是宽松版本，就说明最后社区还是没有说服团队成员。Ryan 在最后的 Close Comment 中总结了几点原因：

* 并不是所有人都希望内置严格的 Omit 类型，支持者最多只有 70%。
* StrictOmit 是具有传染性的，可能导致一批下游依赖的类型声明出问题，让开发者选择是要宽松还是阉严格更符合直觉。
* 就算 TS 直接用掉了 Omit 这个名字，其实社区还可以用 StrictOmit、Except（type-fest中）、Remove 等名字（Sindre 批评说占用了这个名字却只实现了阉割功能）

### 扩展

以下扩展和本文主旨无关，属于对 Pick 和 Omit 类型的扩展，欢迎你将它们作为习题进一步独立研究。

* Pick 和 Omit 是通过键来裁剪的，请实现基于值裁剪的 PickByValue 与 OmitByValue 类型。
* 在上一题的基础上，实现严格的基于值的裁剪，如对于联合类型的键值类型，需要其完全的匹配（如 `PickByValue<T, 'a'|'b'|'c'>` 不能保留类型为 'a' 的键）。
