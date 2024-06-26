---
description: '2021-01-06'
---

# 知其然，知其所以然：TypeScript 中的协变与逆变

### 前言

在前一篇文章《淘宝店铺 TypeScript ESLint 规则集考量》中，我们提到了这一条规则：**method-signature-style**，它的作用是对 interface 中不同的函数声明方式进行约束，这里的声明方式主要有两种，_method_ 与 _property_，区别如下：

```typescript
// method
interface T1 {
  func(arg: string): number;
}

// property
interface T2 {
  func: (arg: string) => number;
}
```

首先，这两种方式被称为 method 与 property 的原因很明显，method 方式就像是在 Class 中定义方法一样，而 property 则是就像定义普通的接口属性，只不过它的值是函数类型。

在 TypeScript ESLint 官方对此规则的解释中，推荐使用 property 方式，而最重要的原因即是 property + 函数类型值的声明方式使得函数的类型能享受到更严格的类型校验（需要开启 [`strictFunctionTypes`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-6.html#strict-function-types)，或只开启 `strict` 即可）。

那么这一条配置又是啥？函数的类型校验为什么还能有更严格的方式，默认情况下又是怎样的？为什么使用 method 声明就享受不到了？

这篇文章我们就从这里开始，聊一聊 TypeScript 中的协变与逆变。

### 基础概念

直接上概念太劝退了，我们先从生活中随处可见的例子开始：

```typescript
class Animal {
  asPet() {}
}

class Dog extends Animal {
  bark() {}
}

class Corgi extends Dog {
  cute() {}
}
```

在这里，我们有三个依次派生的类，每个类在上一个类的基础上添加了一个独特的方法。我们使用 `≼` 符号表达子类型关系，`A ≼ B` 意为 A 是 B 的子类型，在这里的例子中，易得 `Corgi ≼ Dog ≼ Animal`。

现在我们多了一个函数：它接收一只狗作为参数，并尝试让它吠几声听听：

```ty
function makeDogBark(dog: Dog) {
  dog.bark();
}
```

现在来试着调用一下：

```typescript
makeDogBark(new Corgi());
makeDogBark(new Animal());
```

你很容易发现第一种是可以的，因为所有的柯基都是狗，都会吠，但第二种，并不是所有的动物都会吠，所以这里会抛出一个错误。通过这个非常简单的例子，你回忆了一下子类型和父类型，热身结束，要开始看点认真的了。

#### 使用函数作为入参

再看一个例子，假设现在我们有一个新的函数，**它接收一个函数作为参数**，其类型为 `Dog -> Dog`（即参数类型与返回值均为 Dog）。

```typescript
type DogFactory = (args: Dog) => Dog;

function transformDogAndBark(dogFactory: DogFactory) {
  const dog = dogFactory(new Dog());
  dog.bark();
}
```

这种情况下，我们要和 `Dog -> Dog` 进行比较的也一定是 `Corgi/Animal` -> `Corgi/Animal` 这样的类型，来排列组合一下几种情况：

* `Corgi -> Corgi`：我们在 `transformDogAndBark` 中，会传入一只狗，并让返回的狗狗叫两声听听，看起来好像没问题，但是 `Corgi -> Corgi` 函数只能接受柯基，内部可能调用了柯基才有的逻辑，如果我们传了个柴犬，那程序可能就崩溃了。但返回值没问题，因为不管是柯基还是柴犬都能叫嘛。
* `Animal -> Animal`：有了上一点的经验我们一看就知道不行，因为它的返回值可能是任何动物，但不是任何动物都会狗叫。
* `Corgi -> Animal`：第一点是参数类型有问题，第二点是返回值类有问题，这一点则是参数类型和返回值都有问题。
* `Animal -> Corgi`：只剩下这一个正确答案了，如果还不行的话就离谱了。还是先来分析一波，首先我们会传入一只狗，好的，没问题，Animal 有的方法 Dog 都有。接着我们会让返回的物种叫两声，这里返回的是柯基！它可以叫！所以没问题！

> Dog 派生自 Animal，但不会修改其内部的原有功能如繁殖、进食，即里氏替换原则：子类可以扩展父类的功能，但不能改变父类原有的功能，子类型（subtype）必须能够替换掉他们的基类型（base type）。

归纳一下上面的情况，我们会发现，作为参数的函数，它的入参允许是函数入参类型的父类型（实际入参 Animal，类型入参 Dog），不允许为子类型（实际入参 Corgi，类型入参 Dog），而它的返回值允许是函数返回类型的子类型（实际返回值 Corgi，类型返回值 Corgi），不允许是父类型（实际返回值 Animal，类型返回值 Dog）。

考虑到在前面最简单的例子中我们知道可用的入参类型会是函数入参类型的子类型，也就说，以下这一等式成立：

```
(Animal → Corgi) ≼ (Dog → Dog)
```

#### 协变与逆变

这个时候我们可以引入协变（_covariance_）与逆变（_contravariance_）的概念了，

> 看这两个单词，去掉意为变异性的 variance 后，还剩下 co 和 contra。如果说 contra 你不知道什么意思，那 co 你一定知道，如 Collaboration、Cooperation 这两个单词都是加上 co 就都多了协作的意思，懂伐？

这两个单词实际上最初应该来自于几何学领域中：**随着某一个量的变化，随之变化一致的即称为协变，而变化相反的即称为逆变。** 而在这里，我们称函数参数为逆变，函数返回值为协变，为什么？

考虑 `Corgi ≼ Dog`，如果它遵循协变，则有 `(T → Corgi) ≼ (T → Dog)`，即 **A、B 在被作为函数返回值类型以后仍然遵循一致的子类型关系**。而对于参数，由于其遵循逆变，则有 `(Dog → T) ≼ (Corgi → T)`，即 **A、B 被作为函数参数类型以后其子类型关系发生逆转**。

这一段可能有点猝不及防，我们来讲讲人话。对于我个人来说，习惯将 “随着某一个量的变化” 中的 变化，在 TypeScript 作为一个工具类型 **Wrapper** 理解，如：

```typescript
type AsFuncArgType<T> = (arg: T) => void;

type AsFuncReturnType<T> = (arg: unknown) => T;

// 不成立：Corgi -> void ≼ Dog -> void
type CheckArgType = AsFuncArgType<Corgi> extends AsFuncArgType<Dog> ? 1 : 2;

// 成立：unknown -> Corgi ≼ unknown -> Dog
type CheckReturnType = AsFuncReturnType<Corgi> extends AsFuncReturnType<Dog>
  ? 1
  : 2;
```

`Wrapper<B> ≼ Wrapper<A>`，这里的 Wrapper 可能是函数这种隐式的 包装，也可能是 Promise、Array、Record 等等这些显式将其作为类型参数的高阶类型。

在 `A ≼ B` 时，协变意味着 `Wrapper<A> ≼ Wrapper<B>`，而逆变意味着 `Wrapper<B> ≼ Wrapper<A>`。

而按照正确的检查逻辑（即我们上面的四种情况检查），函数的参数类型应该使用逆变的方式来进行检查，而返回值类型则是协变。

```
(Animal → Corgi) ≼ (Dog → Dog)
```

### 默认情况下与 strictFunctionTypes 下 的函数类型检查

还记得在前面我们说到 **method-signature-style** 这条规则时提到它会对函数类型会启用更严格的类型检查（因而会包括使用 property 属性声明的函数类型），这个 “更严格” 指的就是，对于函数参数启用逆变检查。啥意思？先来看看默认情况下与启用 strictFunctionTypes 的情况下分别是如何的，仍然使用 Animal、Dog 的例子，引入一个额外的 Cat，考虑以下函数：

```typescript
declare let f1: (x: Animal) => void;
declare let f2: (x: Dog) => void;
declare let f3: (x: Cat) => void;
```

为了检验函数类型之间的兼容性，我们对 f1 f2 f3 之间进行赋值操作：

> * 设有 f1 = f2 成立，则 f2 ≼ f1
> * 接着设有 f1: A1 -> B1，f2: A2 -> B2，若 f1 = f2 成立，则 A2 -> B2 ≼ A1 -> B1

* f1 = f2，即 `A2 -> B2 ≼ A1 -> B1`，即 `Dog -> void ≼ Animal -> void`
  * 上面说到如果“认真的”去检查函数类型的可分配性，函数参数的部分需要满足的是逆变的关系（_Animal → Corgi) ≼ (Dog → Dog_），即 Dog 是 Animal 的子类，则 `Animal -> void` 的左侧函数的入参类型需要是 Animal 的父类如 Creature。因此在开启 _strictFunctionTypes_ 的情况下此分配**不成立**。
  * 但在默认情况下，对于函数类型的检查，在参数部分是双向协变（_bivariantly_），即 `Dog ≼ Animal` 可以推导出 `Dog -> void ≼ Animal -> void` 与 `Animal -> void ≼ Dog -> void`
* f2 = f1，即 `Animal -> void ≼ Dog -> void`
  * 在严格检查（逆变）与默认情况下（双向协变）成立
* f2 = f3，即 `Cat -> void ≼ Dog -> void`
  * Cat 与 Dog 均是 Animal 的子类，它们之间不存在派生关系，因此甚至不能满足逆变、协变的发生条件，故在严格检查与默认情况下均不成立。

那么，为什么默认情况下函数的参数是使用双向协变检查的？我们来看官方的说明（原文：[Why are function parameters bivariant?](https://github.com/Microsoft/TypeScript/wiki/FAQ#why-are-function-parameters-bivariant)）

考虑如下代码：

```typescript
function trainDog(d: Dog) { ... }
function cloneAnimalAndDoSth(source: Animal, sth: (result: Animal) => void): void { ... }

let c = new Cat();
                                                                            cloneAnimalAndDoSth(c, trainDog);
```

最后一行的 cloneAnimal 调用很明显是会报错的，因为此函数内部会将传入的 source 交给第二个参数的函数进行处理，这里我们传了猫猫以及给狗狗使用的训练函数，明显是不可行的，但是在默认情况下这一段代码不会报错（因为默认双向协变嘛）。

我们知道这里实际是 `Dog -> void` 和 `Animal -> void` 的比较，如果按照逆变的转换， `Dog ≼ Animal`，则 `Animal -> void ≼ Dog -> voi`，那么这里 trainDog 是不能赋值给 sth 的！但是当然默认情况下由于执行函数参数的双向协变，所以并不会报错。

事实上，其实我们能非常神奇的从 Dog\[] 与 Animal\[] 的比较推导到 Dog -> void 和 Animal -> void 的比较，再回到 Dog 与 Animal 的比较：

* `Dog[] ≼ Animal[]` 是否成立？
* Dog\[] 上的每一个成员（属性、方法）是否都能对应的赋值给 Animal\[]？
  * 你以为我要问 Dog 和 Animal 的比较？错了，我要问的是，`Dog[].push ≼ Animal[].push` 是否成立？
  * 由 push 方法进一步推导，`Dog -> void ≼ Animal -> void` 是否成立？

这里你就开始感觉不对劲了，如果我们要求 `Dog[] ≼ Animal[]` 成立，按照这个推导关系，就要求 `Dog -> void ≼ Animal -> void` ，在逆变的情况下意味着 `Animal ≼ Dog`，这很明显是不对的。简单的来说， `Dog -> void ≼ Animal -> void` 是否成立本身就为 `Dog[] ≼ Animal[]` 提供了一个前提答案。

这一节即是本文前言中最核心的一个部分，使用 property 方式声明接口中的函数，其享受的更严格类型检查指的是什么。不妨来看一下使用 method 方式声明：

```typescript
interface Comparer<T> {
  compare(a: T, b: T): number;
}

declare let animalComparer: Comparer<Animal>;
declare let dogComparer: Comparer<Dog>;

animalComparer = dogComparer;
dogComparer = animalComparer;
```

这里的赋值操作其实和前面的大致类似，注意这里两个赋值是不会报错的，因为使用 method 方式声明时仍然保持默认的双向协变，如果改为 property 声明，则第一条赋值语句 `animalComparer = dogComparer;` 将不成立。

> 为什么使用 method 声明就享受不到了？这主要是为了仍然提供双向协变的校验给一些需要的场景，如 Array 的内部声明：
>
> ```typescript
> interface Array<T> {
>   push(...items: T[]): number;
>   concat(...items: ConcatArray<T>[]): T[];
>   join(separator?: string): string;
> }
> ```

### 其他场景

假设现在 Wrapper 不再是函数体了，直接一个简单的笼子 Cage 呢？先不考虑 Cage 内部的实现，只知道它同时只能放一个物种的动物，`Cage<Dog>` 能被作为 `Cage<Animal>` 的子类型吗？对于这一类型的比较，我们可以直接用实际场景来代入，即：

* 假设我需要一笼动物，但并不会对它们进行除了读以外的操作，那么你给我一笼狗我也是没问题的。也就意味着，此时 List 是 readonly 的，而 `Cage<Dog> ≼ Cage<Animal>` 成立。即在不可变的 Wrapper 中，我们允许其遵循协变。
* 假设我需要一笼动物，并且会在其中新增其他物种，比如兔子啊王八，这个时候你给我一笼狗就不行了，因为这个笼子只能放狗，放兔子进行可能会变异。也就意味着，此时 List 是 writable 的，而 `Cage<Dog>` `Cage<Rabit>` `Cage<Turtle>` 彼此之前是互斥的，我们称为 不变（_invariant_），用来放狗的笼子绝不能用来放兔子，即无法进行分配。
* 如果我们再修改下规则，现在一个笼子可以放任意物种的动物，狗和兔子可以放一个笼子里，这个时候任意的笼子都可以放任意的物种，放狗的可以放兔子，放兔子的也可以放狗，即可以互相分配，我们称之为双变（_Bivariant_）。

全文就到这里结束了，实际上协变与逆变对我来说也是比较新鲜的概念，并且由于我只学习过 TypeScript 这一门类型语言，其中的叙述可能还有着不尽准确之处，欢迎指正与共同交流。下一篇文章中，我们会谈到一个大部分人都比较感兴趣的话题：TypeScript 中的工具类型与类型体操，这会是一个比较庞大的内容，我可以拍着胸脯保证看完你就会 95% 的工具类型与类型体操了！
