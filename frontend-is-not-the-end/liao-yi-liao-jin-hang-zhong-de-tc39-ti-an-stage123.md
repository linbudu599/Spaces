---
description: '2021-06-16'
---

# 聊一聊进行中的TC39提案（stage1/2/3）

### 前言

最近看到了一些很有趣的ES提案，如Record与Tuple数据类型，思路来自RxJS的Observable，借鉴自函数式编程的throw Expressions，带来更好错误处理的`Error Cause`等，可以认为一旦这些提案完全进入到ES新特性中，前端er们的工作效率又会upup，这篇文章就来介绍一下我认为值得关注的ES提案。

作为前端同学，即使你没有去主动了解过，应该也或多或少听说过ECMA、ECMAScript、TC39、ES6（这个当然了）这些词，你可能对这些名词代表的概念一知半解甚至是从未了解过，但这很正常，不知道这些名词的关系并不影响你将ES新特性用的如臂使指。但了解一下也不亏？所以在开始正式介绍各种提案前，我们有必要先了解一下这些概念。

> 以下关于背景的介绍大部分来自于雪碧老师的[JavaScript20年-创立标准](https://cn.history.js.org/part-2.html)一节。

* [ECMA（European Computer Manufacturers Association，欧洲计算机制造商协会）](https://www.ecma-international.org/)，这是一个国际组织，主要负责维护各种计算机的相关标准。我们都知道JavaScript这门语言最早来自于网景（Netscape），但网景在和微软（IE）的竞争落得下风，为了避免最终Web脚本主导权落入微软手中，网景开始寻求ECMA组织的帮助，来推动JavaScript的标准化。
* 在1996年，JavaScript正式加入了ECMA大家庭，我们后面会叫它ECMAScript（下文简称ES）。TC39则是ECMA为ES专门组织的技术委员会（Technical Committee），39这个数字则是因为ECMA使用数字来标记旗下的技术委员会。TC39的成员由各个主流浏览器厂商的代表构成（因为毕竟最后还要这些人实现嘛）。
* ECMA-262即为ECMA组织维护的第262条标准，这一标准是在不断演进的，如现在是[2020年6月发布的第11版](https://www.ecma-international.org/publications-and-standards/standards/ecma-262/)。同样的，目前最为熟知的是[2015年发布的ES6](https://262.ecma-international.org/6.0/index.html)。你还可以在[TC39的ECMA262官网](https://tc39.es/ecma262/)上看到ES2022的最新草案。
* ECMA还维护着许多其他方面的标准，如[ECMA-414](https://www.ecma-international.org/publications-and-standards/standards/ecma-414/)，定义了一组ES规范套件的标准；[ECMA-404](https://www.ecma-international.org/publications-and-standards/standards/ecma-404/)，定义了JSON数据交换的语法；甚至还有120mm DVD的标准：[ECMA267](https://www.ecma-international.org/publications-and-standards/standards/ecma-267/)。
* 对于一个提案从提出到最后被纳入ES新特性，TC39的规范中有五步要走：
  * stage0（**strawman**），任何TC39的成员都可以提交。
  * stage1（**proposal**），进入此阶段就意味着这一提案被认为是**正式**的了，需要对此提案的场景与API进行详尽的描述。
  * stage2（**draft**），演进到这一阶段的提案如果能最终进入到标准，那么在之后的阶段都不会有太大的变化，因为理论上只接受增量修改。
  * state3（**candidate**），这一阶段的提案只有在遇到了重大问题才会修改，规范文档需要被全面的完成。
  * state4（**finished**），这一阶段的提案将会被纳入到ES每年发布的规范之中。
* 有兴趣的同学可以阅读 [The TC39 process for ECMAScript features](https://2ality.com/2015/11/tc39-process.html) 了解更多。

### Record & Tuple（stage2）

[proposal-record-tuple](https://github.com/tc39/proposal-record-tuple) 这一提案为JavaScript新增了两种数据结构：Record（类似于对象） 和 Tuple（类似于数组），它们的共同点是都是不可变的（Immutable），同时成员只能是原始类型以及同样不可变的Record和Tuple。正因为它们的成员不能包含引用类型，所以它们是 **按值比较** 的，成员完全一致的Record和Tuple如果进行比较，会被认为是相同的（'==='会返回true）。

> 你可能会想到社区其实对于数据不可变已经有不少方案了，如ImmutableJS与Immer。而数据不可变同样是React中的重要概念。

使用示例：

```js
// Record
const proposal = #{
  id: 1234,
  title: "Record & Tuple proposal",
  contents: `...`,
  keywords: #["ecma", "tc39", "proposal", "record", "tuple"],
};


// Tuple
const measures = #[42, 12, 67, "measure error: foo happened"];
```

个人感想：会是很有用的新成员，尤其是在追求性能优化下以及React项目中，gkdgkd。

### `.at()` Relative Indexing Method （stage 3）

[proposal-relative-indexing-method](https://github.com/tc39/proposal-relative-indexing-method)提案引入了`at()`方法，用于获取可索引类(Array, String, TypedArray)上指定位置的成员。

在过去JavaScript中一直缺乏负索引相关的支持，比如获取数组的最后一个成员需要使用`arr[arr.length-1]`，而无法使用`arr[-1]`。这主要是因为JavaScript中`[]`可以对所有对象使用，所以`arr[-1]`返回的是key为`-1`的属性值，而非索引为-1（从后往前排序）的数组成员。

而要获取数组的第N个成员，通常使用的方法是`arr[arr.length - N]`，或者`arr.slice(-N)[0]`，两种方法都有各自的缺陷，因此`at()`就来救场了。

另外，还存在获取数组最后一个成员的提案，[proposal-array-last](https://github.com/tc39/proposal-array-last) （stage1）与获取数组最后一个符合条件的成员的提案 [proposal-array-find-from-last](https://github.com/tc39/proposal-array-find-from-last)。

个人感想：来得有点晚，但也不算晚。

### Temporal （stage 3）

[proposal-temporal](https://github.com/tc39/proposal-temporal)主要是为了提供标准化的日期与时间API，这一提案引入了一个全局的命名空间Temporal（类似于Math、Promise）来引入一系列现代化的日期API（JavaScript的Date API谁用谁知道嗷，也难怪社区那么多日期处理库了），如：

*   `Temporal.Instant` 获取一个固定的时间对象：

    ```javascript
    const instant = Temporal.Instant.from('1969-07-20T20:17Z');
    instant.toString(); // => '1969-07-20T20:17:00Z'
    instant.epochMilliseconds; // => -14182980000
    ```
*   `Temporal.PlainDate` 获取calendar date：

    ```javascript
    const date = Temporal.PlainDate.from({ year: 2006, month: 8, day: 24 }); // => 2006-08-24
    date.year; // => 2006
    date.inLeapYear; // => false
    date.toString(); // => '2006-08-24'
    ```
*   `Temporal.PlainTime` 获取wall-clock time（和上面一样，不知道咋翻译）：

    ```javascript
    const time = Temporal.PlainTime.from({
      hour: 19,
      minute: 39,
      second: 9,
      millisecond: 68,
      microsecond: 346,
      nanosecond: 205
    }); // => 19:39:09.068346205

    time.second; // => 9
    time.toString(); // => '19:39:09.068346205'
    ```
*   `Temporal.Duration` 获取一段时间长度，用于比较时间有奇效

    ```javascript
    const duration = Temporal.Duration.from({
      hours: 130,
      minutes: 20
    });

    duration.total({ unit: 'second' }); // => 469200
    ```

更多细节参考[ts39-proposal-temporal docs](https://tc39.es/proposal-temporal/docs/index.html)。

个人感想：同样，来得有点晚，但也不算晚。

### Private Methods (stage 3)

[private-methods](https://github.com/tc39/proposal-private-methods) 提案为JavaScript Class引入了私有属性、方法以及getter/setter，不同于TypeScript中使用`private`语法，这一提案使用`#`语法来标识私有成员，在阮老师的[ES6标准入门](https://es6.ruanyifeng.com/#docs/class#%E7%A7%81%E6%9C%89%E5%B1%9E%E6%80%A7%E7%9A%84%E6%8F%90%E6%A1%88)中也提到了这一提案。

> 所以这个提案已经过了多少年了...

参考阮老师给的例子：

```javascript
class IncreasingCounter {
  #count = 0;
  get value() {
    console.log('Getting the current value!');
    return this.#count;
  }
  increment() {
    this.#count++;
  }
}
```

类似的，还有一个同样处于stage3的提案[proposal-class-fields](https://github.com/tc39/proposal-class-fields)引入了`static`关键字。

个人感想：对我来说用处比较小，因为毕竟都是写TS，几乎没有什么机会在JavaScript中写Class了。但是这一提案成功被引入后，可能会使得TS到JS的编译产物变化，即直接使用JS自身的`static`、`#`语法。比如现在这么一段TS代码：

```typescript
class A {
	static x1 = 8;
}
```

编译结果是：

```javascript
"use strict";
class A {}
A.x1 = 8;
```

而在`static`被引入后，则会直接使用`static`语法。

### Top-level `await` (stage4)

> 我记得这篇文章开始写的时候，这个提案还在stage3的，我到底鸽了多久...

[proposal-top-level-await](https://github.com/tc39/proposal-top-level-await)这个提案感觉就没有啥展开描述的必要了，很多人应该已经用上了。简单地说，就是你的`await`语法不再和`async`强绑定了，你可以直接在应用的最顶层使用`await`语法，Node也从14.8开始支持了这一提案。

个人感想：可以少写一个async函数了，奈斯奥。

### Import Assertions (stage 3)

[proposal-import-assertions](https://github.com/tc39/proposal-import-assertions) 这一提案为导入语句新增了用于标识模块类型的断言语句，语法如下：

```javascript
import json from "./foo.json" assert { type: "json" };
import("foo.json", { assert: { type: "json" } });
```

> 注意，对JSON模块的导入最开始属于这一提案的一部分，后续被独立出来作为一个单独的提案：[proposal-json-modules](https://github.com/tc39/proposal-json-modules)。

这一提案最初起源于为了在JavaScript中更便捷的导入JSON模块，后续出于安全性考虑加上了`import assertions`来作为导入不可执行模块的必须条件。

这一提案同样解决了模块类型与其MIME类型不符的情况。

个人感想：和现在如火如荼的ESM、Bundleless工具应该会有奇妙的化学反应。

### Error Cause (stage 3)

[proposal-error-cause](https://github.com/tc39/proposal-error-cause)这一提案目前由[吞吞老师](https://github.com/legendecas)在推进，其目的主要是为了便捷的传递导致错误的原因，如果不使用这个模块，想要清晰的跨越多个调用栈传递错误上下文信息，通常要这么做：

```javascript
async function getSolution() {
  const rawResource = await fetch('//domain/resource-a')
    .catch(err => {
      // How to wrap the error properly?
      // 1. throw new Error('Download raw resource failed: ' + err.message);
      // 2. const wrapErr = new Error('Download raw resource failed');
      //    wrapErr.cause = err;
      //    throw wrapErr;
      // 3. class CustomError extends Error {
      //      constructor(msg, cause) {
      //        super(msg);
      //        this.cause = cause;
      //      }
      //    }
      //    throw new CustomError('Download raw resource failed', err);
    })
  const jobResult = doComputationalHeavyJob(rawResource);
  await fetch('//domain/upload', { method: 'POST', body: jobResult });
}

await doJob(); // => TypeError: Failed to fetch
```

而按照这一提案的语法：

```javascript
async function doJob() {
  const rawResource = await fetch('//domain/resource-a')
    .catch(err => {
      throw new Error('Download raw resource failed', { cause: err });
    });
  const jobResult = doComputationalHeavyJob(rawResource);
  await fetch('//domain/upload', { method: 'POST', body: jobResult })
    .catch(err => {
      throw new Error('Upload job result failed', { cause: err });
    });
}
```

简单了很多，对吧？

个人感想：可以说是很需要的语法了，我在Node应用中处理错误确实就是各种`ArgsValidationError`、`RoleRejectedError`自定义错误类满天飞，维护起来头都大了。

### Decorators (stage 2)

[proposal-decorators](https://github.com/tc39/proposal-decorators)这一提案...，或许是我们最熟悉的老朋友了。但是此装饰器非彼装饰器，历时五年来装饰器提案已经走到了第三版，仍然卡在stage 2。

这里引用我早前的一篇文章来简单讲述下装饰器的历史：

> 首先我们需要知道，JS 与 TS 中的装饰器不是一回事，JS 中的装饰器目前依然停留在 [stage 2](https://github.com/tc39/proposal-decorators) 阶段，并且目前版本的草案与 TS 中的实现差异相当之大（TS 是基于第一版，JS 目前已经第三版了），所以二者最终的装饰器实现必然有非常大的差异。
>
> 其次，装饰器不是 TS 所提供的特性（如类型、接口），而是 TS 实现的 ECMAScript 提案（就像类的私有成员一样）。TS 实际上只会对**stage-3**以上的语言提供支持，比如 TS3.7.5 引入了可选链（[Optional chaining](https://github.com/tc39/proposal-optional-chaining)）与空值合并（[Nullish-Coalescing](https://github.com/tc39/proposal-nullish-coalescing)）。而当 TS 引入装饰器时（大约在 15 年左右），JS 中的装饰器依然处于**stage-1** 阶段。其原因是 TS 与 Angular 团队 PY 成功了，Ng 团队不再维护 [AtScript](https://linbudu.top/posts/2020/08/10/\[atscript-playground]\(https://github.com/angular/atscript-playground\))，而 TS 引入了注解语法（**Annotation**）及相关特性。
>
> 但是并不需要担心，即使装饰器永远到达不了 stage-3/4 阶段，它也不会消失的。有相当多的框架都是装饰器的重度用户，如`Angular`、`Nest`、`Midway`等。对于装饰器的实现与编译结果会始终保留，就像`JSX`一样。如果你对它的历史与发展方向有兴趣，可以读一读 [是否应该在 production 里使用 typescript 的 decorator？](https://www.zhihu.com/question/404724504)（贺师俊贺老的回答）

个人感想：和类的私有成员、静态成员提案一样，目前使用最广泛的还是TS中的装饰器，但是二者的思路完全不同，因此我猜想原生装饰器的提案不会影响TypeScript的编译结果。

### Iterator Helpers （stage 2）

[proposal-iterator-helpers](https://github.com/tc39/proposal-iterator-helpers)提案为ES中的Iterator使用与消费引入了一批新的接口，虽然实际上，如Lodash与[Itertools](https://www.npmjs.com/package/itertools)（思路来自于Python3中的[itertools](https://docs.python.org/library/itertools.html)）这样的工具库已经提供了绝大部分能力，如filter、filterMap等。其他语言如Rust、C#中也内置了非常强大的Iterator Helpers，见[Prior Art](https://github.com/tc39/proposal-iterator-helpers#prior-art)。

示例：

```javascript
function* naturals() {
  let i = 0;
  while (true) {
    yield i;
    i += 1;
  }
}

const evens = naturals()
  .filter((n) => n % 2 === 0);

for (const even of evens) {
  console.log(even, 'is an even number');
}
```

个人感想：虽然目前很少会直接操作Generator和Iterator了，但这些毕竟是语言底部的东西，了解使用与机制还是有好处的。我上一次接触Iterator，还是为Nx编写插件时为其提供Async Iterator接口，但也是直接囫囵吞枣的使用[rxjs-for-await](https://github.com/benlesh/rxjs-for-await)这个库。对这个提案的关注可能会相对少一些。

### `throw` Expressions (stage 2)

[proposal-throw-expressions](https://github.com/tc39/proposal-throw-expressions)这一提案主要提供了`const x = throw new Error()`的能力，这并不是`throw`语法的替代品，更像是面向表达式(Expression-Oriented)的补齐。

```javascript
function getEncoder(encoding) {
  const encoder = encoding === "utf8" ? new UTF8Encoder() 
                : encoding === "utf16le" ? new UTF16Encoder(false) 
                : encoding === "utf16be" ? new UTF16Encoder(true) 
                : throw new Error("Unsupported encoding");
}
```

个人感想：错误处理又可以更自然美观一些了，奈斯！

### Set Methods (stage 2)

[proposal-set-methods](https://github.com/tc39/proposal-set-methods)这一提案为Set新增了一批新的方法，如：

* `intersection`/`union`/`difference`：基于交集/并集/差集创建新的Set
* `isSubsetOf/isSupersetOf`：判断是否是子集/超集

个人感想：Set的话用的比较少，但很明显这些方法会是一个不错的能力增强。

### Upsert(Map.prototype.emplace) (stage 2)

[proposal-upsert](https://github.com/tc39/proposal-upsert)这一提案为Map引入了emplace方法，在当前Map上的key已存在时，执行更新操作，否则执行创建操作。

个人感想：确实是很甜的语法糖，感觉底层框架、工具库用Map多一些。

### Observable (stage 1)

[proposal-observable](https://github.com/tc39/proposal-observable)这一提案，其实懂的同学看到Observable已经懂这个提案是干啥的了，它引入了RxJS中的Observable、Observer（同样是next/error/complete/start）、Subscriber(next/error/complete)以及部分Operators（RxJS：我直接好家伙），同样支持高阶Observable，在被订阅时才会开始推送数据（Lazy-Data-Emitting）。

```javascript
function listen(element, eventName) {
    return new Observable(observer => {
        // Create an event handler which sends data to the sink
        let handler = event => observer.next(event);

        // Attach the event handler
        element.addEventListener(eventName, handler, true);

        // Return a cleanup function which will cancel the event stream
        return () => {
            // Detach the event handler from the element
            element.removeEventListener(eventName, handler, true);
        };
    });
}
```

估计是因为还在stage1的关系，目前支持的操作符只有of、from，但按照这个趋势下去RxJS中的大部分操作符都会被吸收过来。

个人感想：感觉需要非常久的时间才能看到未来结果，因为RxJS自身强大的多，海量操作符如果要吸收过来可能会是吃力不讨好的。同时，RxJS的学习成本还是有的，我不认为大家会因为它被吸收到JS语言原生就会纷纷开始学习相关概念。

### Promise.try (stage 1)

[proposal-promise-try](https://github.com/tc39/proposal-promise-try)提案引入了`Promise.try`方法，这一方法其实很早就在[bluebird](http://bluebirdjs.com/docs/api/promise.try.html)中提供了，其使用方式如下：

```javascript
function getUserNameById(id) {
    return Promise.try(function() {
        if (typeof id !== "number") {
            throw new Error("id must be a number");
        }
        return db.getUserById(id);
    }).then((user)=>{
        return user.name
});
}
```

`Promise.try`方法返回一个promise实例，如果方法内部抛出了错误，则会走到`.catch`方法。上面的例子如果正常来写，通常会这么写：

```javascript
function getUserNameById(id) {
    return db.getUserById(id).then(function(user) {
        return user.name;
    });
}
```

看起来好像没什么区别，但仔细想想，假设下面一个例子中，id是错误的，

`db.getUserById(id)`返回了空值，那么这样user.name无法获取，将会走`.catch`，但如果不返回空值而是抛出一个同步错误？Promises的错误捕获功能的工作原理是所有同步代码都位于.then中，这样它就可以将其包装在一个巨大的`try/catch`块中（所以同步错误都能走到`.catch`中）。但是在这个例子中，`db.getUserById(id)`并非位于`.then`语句中，这就导致了这里的同步错误无法被捕获。简单的说，如果仅使用`.then`，只有第一次异步操作后的同步错误会被捕获。

而是用`Promise.try`，它将捕获`db.getUserById(id)`中的同步错误（就像`.then`一样，区别主要在try不需要前面跟着一个promise实例），这样子所有同步错误就都能被捕获了。

### Do Expression (stage 1)

[proposal-do-expressions](https://github.com/tc39/proposal-do-expressions)这个提案和`throw` Expressions一样，都是面向表达式（Expression-Oriented）的语法，函数式编程的重要优势之一。

看看示例代码：

```javascript
let x = do {
  let tmp = f();
  tmp * tmp + 1
};

let y = do {
  if (foo()) { f() }
  else if (bar()) { g() }
  else { h() }
};
```

对于像我一样没接触过函数式编程的同学，这种语法可能确实很新奇有趣，而且对能帮助更好的组织代码。这一提案还存在着一些注意点：

* 在`do {}`中不能仅有声明语句，或者是缺少else的if，以及循环。
* 空白的`do {}`语句效果等同于`void 0`。
* `await`/`yield`标识继承自上下文

对于异步版本的`do expression`，存在一个尚未进入的提案[proposal-async-do-expressions](https://github.com/tc39/proposal-async-do-expressions)，旨在使用`async do {}`的语法，如：

```javascript
// at the top level of a script

async do {
  await readFile('in.txt');
  let query = await ask('???');
  // etc
}
```

### Pipeline Operator (stage 1)

> 目前star最多的提案，似乎没有之一？

[proposal-pipeline-operator](https://github.com/tc39/proposal-pipeline-operator)提案引入了新的操作符`|>`，目前对于具体实现细节存在[两个不同的竞争提案](https://github.com/tc39/proposal-pipeline-operator/wiki)。这一语法糖的主要目的是大大提升函数调用的可读性，如`doubleNumber(number)`会变为`number |> doubleNumber`的形式，对于链式的连续函数调用更是有奇效，如：

```javascript
function doubleSay (str) {
  return str + ", " + str;
}
function capitalize (str) {
  return str[0].toUpperCase() + str.substring(1);
}
function exclaim (str) {
  return str + '!';
}
```

在管道操作符下，变为如下形式：

```javascript
let result = exclaim(capitalize(doubleSay("hello")));
result //=> "Hello, hello!"

let result = "hello"
  |> doubleSay
  |> capitalize
  |> exclaim;

result //=> "Hello, hello!"
```

确实大大提高了不少可读性对吧？你可能会想，上面都是单个入参，那多个呢，如下图示例：

```javascript
function double (x) { return x + x; }
function add (x, y) { return x + y; }

function boundScore (min, max, score) {
  return Math.max(min, Math.min(max, score));
}

let person = { score: 25 };

let newScore = person.score
  |> double
  |> (_ => add(7, _))
  |> (_ => boundScore(0, 100, _));

newScore //=> 57
```

等同于

```javascript
let newScore = boundScore( 0, 100, add(7, double(person.score)) )
```

> `_`只是形参名称，你可以使用任意的形参名称。

### Partial Application Syntax（stage 1）

[proposal-partial-application](https://github.com/tc39/proposal-partial-application)这一提案引入了新的柯里化（也属于柯里化吧，如果你看了下面的例子觉得不属于，请不要揍我）方式，即原本我们使用bind方法来预先固定一个函数的部分参数，得到一个高阶函数：

```javascript
function add(x, y) { return x + y; }

const addOne = add.bind(null, 1);
addOne(2); // 3

const addTen = x => add(x, 10);
addTen(2); // 12
```

使用Partial Application Syntax，写法会是这样的：

```javascript
const addOne = add(1, ?); 
addOne(2); // 3

const addTen = add(?, 10); 
addTen(2); // 12
```

我们上一个列举的提案[proposal-pipeline-operator](https://github.com/tc39/proposal-pipeline-operator)，其实可以在Partial Application Syntax的帮助下变得更加便捷，尤其是在多参数情况下：

```javascript
let person = { score: 25 };

let newScore = person.score
  |> double
  |> add(7, ?)
  |> boundScore(0, 100, ?);
```

* 目前的实现暂时不支持await
* 关于更多细节，参考 [Pipeline operator: Seeking champions](https://docs.google.com/presentation/d/1for4EIeuVpYUxnmwIwUuAmHhZAYOOVwlcKXAnZxhh4Q/edit#slide=id.g79e9b7164e\_0\_531) 以及 [Pipeline operator draft](https://tc39.es/proposal-pipeline-operator/)。

### await.opts (stage 1)

[proposal-await.ops](https://github.com/tc39/proposal-await.ops)这一提案为await引入了`await.all/race/allSettled/any`四个方法，来简化Promise的使用。实际上它们也正是`Promise.all/race/allSettled/any`的替代者，如：

```javascript
// before
await Promise.all(users.map(async x => fetchProfile(x.id)))

// after
await.all users.map(async x => fetchProfile(x.id))
```

### Array Unique (stage 1)

[proposal-array-unique](https://github.com/tc39/proposal-array-unique)主要是为了解决数组去重的问题，我们以往使用的`[...new Set(array)]` 无法很好的处理非原始类型的值，这一提案引入了`Array.prototype.uniqueBy()`方法来进行数组的去重，类似于[Lodash.uniqBy](https://www.lodashjs.com/docs/lodash.uniqBy)。

个人感想：新的面试题出现了，请实现`Array.prototype.uniqueBy()`。2333，但是这个方法能原生支持还是很棒的。
