---
description: '2023-04-29'
---

# TypeScript 5.1 beta：函数返回值类型优化、Getter/Setter类型优化、JSX 增强

TypeScript 已于 2023.04.18 发布 5.1 beta 版本，你可以在 [5.1 Iteration Plan](https://github.com/microsoft/TypeScript/issues/53031) 查看所有被包含的 Issue 与 PR。如果想要抢先体验新特性，执行：

```bash
$ npm install typescript@beta
```

来安装 beta 版本的 TypeScript，或在 VS Code 中安装 [**JavaScript and TypeScript Nightly**](http://link.zhihu.com/?target=https%3A//marketplace.visualstudio.com/items%3FitemName%3Dms-vscode.vscode-typescript-next) 来更新内置的 TypeScript 支持。

本篇是笔者的第七篇 TypeScript 更新日志，上一篇是 「TypeScript 5.0 beta 发布：新版 ES 装饰器、泛型参数的常量修饰、枚举增强等」，你可以在此账号的创作中找到（或在掘金/知乎搜索**林不渡**），接下来笔者也将持续更新 TypeScript 的 DevBlog 相关，感谢你的阅读。

> 另外，由于 beta 版本与正式版本通常不会有明显的差异，这一系列通常只会介绍 beta 版本而非正式版本。

点评：这个版本的改动应该是这个系列开始发表以来最少的一次，同时改动点也基本全部关注在类型相关。由于笔者暂时没有时间去关注目前 TS 团队的工作重心，就先大胆地猜测下，原因应该是需要给 5.0 中的装饰器留出更多的观察时间与维护精力吧\~

### 函数返回值类型优化

对于所有刚从 JavaScript 迈向 TypeScript 的开发者来说，最让人头皮发麻的就是理解类型这么个东西，而在学习类型时，我们会下意识地搜寻脑海中的记忆，来尝试从陌生的领域中找到那么一点熟悉感。很多时候这么做能帮助你更好地理解新知识，但有时候它也会坑你一把，比如 JS/TS 中的 undefined、null 与 void。

在 JavaScript 中，undefined 和 null 可以被理解为“没有值”和“有值，但是个空值”。而在 TypeScript 中，它们都是**有意义的类型**：

```typescript
let undef: undefined = undefined;
let nullish: null = null;
```

要表示一个“什么都没有”的类型，你要使用的应该是 never 。

类似的在 JS/TS 中存在差异的同名兄弟，还有一个 void 类型，在 JavaScript 中我们偶尔会见到 `void(0)` 、 `void 0` 这样的写法，其实这两种都是 `void expression` 的语法，void 操作符会立刻执行后面的表达式，并返回一个 undefined 值。而在 TypeScript 中，void 仅用于描述一个没有有效 return 语句的函数的返回值：

```typescript
// () => void
const f1 = () => {};

const f2 = (): void => {};
```

对于初学者来说，这里就是一个很迷的地方。在 JavaScript 中，如果一个函数没有显式的 return 语句，或是有 return 语句但是没有 return 一个值，那么它的实际计算值会是 undefined。但在 TypeScript 中，这个函数的类型只能被标记为 void 而不是 undefined，即，你可以认为这个函数“没有返回值”，而不能说这个函数“返回了一个 undefined 值”。是不是很难理解？类似的，如果一个函数返回值类型被标记为 undefined，那么它就必须在代码里写 `return undefined`！

这么一个诡异的现象在 5.1 版本中终于得到了解决，即无有效 return 语句的函数，其返回值类型能够被标注为 undefined，但如果不进行类型标注，推导得到的值仍然是 void：

```typescript
const f1: () => undefined = () => {
  return;
};

const f2 = (): undefined => {};

// () => void
const f3 = () => {};
```

类似的，此前在 `--noImplicitReturns` 这条配置约束下，如果你将函数类型标记为 undefined，TypeScript 会要求你在所有分支都存在一个无意义的 return 语句：

```typescript
function f(): undefined {
  if (Math.random()) {
    return;
  }
  // 一定需要这行，否则会报错认为不是所有路径都返回值
  return;
}
```

而在 5.1 版本这也得到了修正，现在 TS 能将无 return 语句的分支返回值类型也兼容到 undefined 了。

### Getter/Setter 类型优化

在 JavaScript 中一个常见的场景是“接受一个输入值-进行转换-存储-读取”，即最初的输入值与最终的读取值可能是不同的：

```typescript
class Thing {
  private _size = 0;

  get size() {
    return this._size;
  }

  set size(value) {
    let num = Number(value);

    if (!Number.isFinite(num)) {
      this._size = 0;
      return;
    }

    this._size = num;
  }
}
```

在这个例子中，size setter 接受的 value 可以是任意的，反正我一定会给你转成数字，而这个时候就出现了一对 Getter / Setter 的类型差异。TypeScript 4.3 支持了这一功能，用于实现更灵活的 Setter 逻辑：

```typescript
class Thing {
  private _size: number = 0;

  get size(): number {
    return this._size;
  }

  set size(value: string | number | null) {
    let num = Number(value);

    if (!Number.isFinite(num)) {
      this._size = 0;
      return;
    }

    this._size = num;
  }
}
```

但它也不是那么随便的，Setter 指定的输入类型必须包括 Getter 的类型：

```typescript
class Thing {
  private _size: number = 0;

  // "Get" 访问器的返回类型必须可分配给其 "Set" 访问器类型
  get size(): number {
    return this._size;
  }

  // 不再包含 number
  set size(value: string | boolean | null) {
    let num = Number(value);

    if (!Number.isFinite(num)) {
      this._size = 0;
      return;
    }

    this._size = num;
  }
}
```

而在 TS 5.1 版本中，移除了 "Get" 访问器的返回类型必须可分配给其 "Set" 访问器类型 这一限制，因为实际场景中其实还真就存在说 Getter / Setter 的类型风马牛不相及的情况，

### JSX(TSX) 增强

#### 类型检查

此前 TypeScript 对 JSX 标签的类型检查还处于比较粗暴的阶段，就是直接检查其是否是 JSX.Element 类型（所以一定需要全局存在 JSX 的类型命名空间），如果你是字符串，或者是 `Promise<JSX.Element>` 类型，诶，那不好意思，即使渲染器支持你，我也觉得你是个不合法的 JSX 标签：

```typescript
const J1 = (): JSX.Element => {};

<J1 />; // √

const J2 = (): Promise<JSX.Element> => {};

<J2 />; // X

const J3 = (): undefined => {};

<J3 />; // X
```

而在 JSX 的命名空间下，其实是存在一个更合适的类型描述的，即 ElementType，它能够真正表示所有能够作为 JSX 标签的类型：

```typescript
type ElementType<P = any> =
  | {
      [K in keyof JSX.IntrinsicElements]: P extends JSX.IntrinsicElements[K]
        ? K
        : never;
    }[keyof JSX.IntrinsicElements]
  | ComponentType<P>;

interface IntrinsicElements {
  a: React.DetailedHTMLProps<
    React.AnchorHTMLAttributes<HTMLAnchorElement>,
    HTMLAnchorElement
  >;
  // ...
}

type ComponentType<P = {}> = ComponentClass<P> | FunctionComponent<P>;
```

#### 携带命名空间的 JSX 标签

Meta 的 I18N 框架 [FBT](https://facebook.github.io/fbt/) 中有大量使用命名空间的 JSX 标签：

```jsx
<fbt desc="plural example">
  You have
  <fbt:plural
    count={getLikeCount()}
    name="number of likes"
    showCount="ifMany"
    many="likes">
     a like
  </fbt:plural>
  on your
  <fbt:plural
    count={getPhotoCount()}
    showCount="no">
     photo
  </fbt:plural>.
</fbt>

<fbt desc="buy prompt">
  Buy a new
  <fbt:enum enum-range={{
    CAR: 'car',
    HOUSE: 'house',
    BOAT: 'boat',
    HOUSEBOAT: 'houseboat',
  }} value={enumVal} />!
</fbt>
```

这是 JSX Spec 中明确规定合法的使用形式，但此前 TypeScript 并不能正确地识别 `fbt:enum` 这样的标签，而是会将 `fbt` 与 `enum` 分开解析，这一问题现在也在 5.1 中得到了解决。

全文完，我们 TS 5.2 见 :-)
