相信现在很多小伙伴都在使用 TypeScript（以下简称 TS），在 TS 中除了一些常用的基本类型外，还有一些稍微高级一点的类型，这些就是我本次文章要讲的内容：索引类型与映射类型，希望小伙伴们看过这篇文章后能对 TS 有更深一步的理解。

### 索引类型

下面我通过一个官方的例子来说明下什么是索引类型：

```javascript
function pluck(o, names) {
  return names.map((n) => o[n])
}
```

这是个简单的函数，names 是一个数组，里面是 key 值，我们可以从“o”里面取出这些 key 值，理想情况下 names 里面的 key 应该都是“o”里面包含的，否则最终的结果里面就会有 undefined，这个函数返回的结果也应该是“o”中都包含的 value 值，那么我们如何才能做到这些类型约束呢，如果只用一些基础类型，很难达到满意的效果，下面使用索引类型改写下：

```typescript
function pluck<T, K extends keyof T>(o: T, names: K[]): T[K][] {
  return names.map((n) => o[n])
}

interface Person {
  name: string
  age: number
}
let person: Person = {
  name: 'Jarid',
  age: 35
}
let strings: string[] = pluck(person, ['name']) // ok, string[]
```

改写后这个函数是一个泛型函数，泛型为 T 和 K，其中 K 有点特殊，`K extends keyof T`，是什么意思呢，其中 keyof 就是**索引类型查询操作符**，我们从字面意思理解，它就是 T 的 key，就是 T 上已知的公共属性名的联合，对于上面的代码，`keyof Person`就是`'name'|'age'`,那么`K extends keyof T`就是`K extends 'name'|'age'`，这样我们就获取到了 Person 上所有 key 组成的一个联合类型，然后参数`o: T, names: K[]`，就很好理解了，names 就是 K 组成的一个数组。返回值中`T[K][]`我们需要拆开来看 T[K]和[]，就是 T[K]组成的一个数组，那么 T[K]是什么类型呢，它就是**索引访问**操作符，类似于 js 中对象的取值操作，不过这里取的是类型，因为 K 是`'name'|'age'`,所以 T[K]就是`string|number`，这些就是索引类型，其实也不难理解，下面再说下映射类型，它和索引类型结合起来可以做很多事情。

### 映射类型

映射类型也很容易理解，我们先看一个简单的例子

```typescript
type Keys = 'option1' | 'option2'
type Flags = { [K in Keys]: boolean }
```

这个就是一个简单的映射类型，其中的`in`可以理解为是我们平时用的`for...in`，就是去遍历 Keys，然后把 boolean 赋给每一个 key，上面的 Flags 得到的结果就是

```typescript
type Flags = {
  option1: boolean
  option2: boolean
}
```

很简单吧，那么这个东西有什么用处呢，请看下面的例子：

```typescript
// Person
type Person {
    name: string
    age: number
}
```

我们想把这个 Person 里面的属性都变成只读的，像这样：

```typescript
// Readonly Person
type Person {
    readonly name: string
    readonly age: number
}
```

如果我们有很多这样的类型，那么改起来会很麻烦，因为每次都要把这个类型重新写一遍。其实我们可以使用刚才的索引类型和映射类型来写一个泛型：

```typescript
type Readonly<T> = {
  readonly [P in keyof T]: T[P]
}
```

`[P in keyof T]`就是遍历 T 中的 key，`T[P]`就是当前的 key 的类型，其实`[P in keyof T]: T[P]`就是把 T 遍历了一遍，但是我们在属性前面加了个 readonly，这样我们调用这个泛型的时候，它就会把传入的类型的 key 遍历一遍，遍历的同时在前面加个 readonly，最终给我们返回一个新的类型。我们在调用的时候只需要这么用：

```typescript
type Readonly<T> = {
  readonly [P in keyof T]: T[P]
}
type Person {
    name: string
    age: number
}
type ReadonlyPerson = Readonly<Person>
```

索引类型和映射类型除了能实现 Readonly，还能实现很多有意思的东西，我们平时在使用 TS 的时候，TS 已经内置了一些常用的辅助泛型，刚才的 Readonly 就是其一，另外还有很多，我从 TS 的类型定义文件里找了一些，这些泛型从简单到复杂的都有，但基本上都是用上面提到的两个类型实现的，下面我们一起来分析一下。

### TS 常用的辅助泛型及其实现方式

首先来看第一个

```typescript
/**
 * Make all properties in T optional
 */
type Partial<T> = {
  [P in keyof T]?: T[P]
}
```

相信这个泛型很多人都用过，就是把类型都变成可选的，和刚才的 Readonly 是类似的实现方式，只是这个是在后面加了个问号，这样一来属性就变成可选的了。
与之相对的还有一个 Required

```typescript
/**
 * Make all properties in T required
 */
type Required<T> = {
  [P in keyof T]-?: T[P]
}
```

注意这个稍有点不同，它是`-?`,其实就是减去问号，这样就可以把问号去掉，从而变成必选的属性。再来看下一个

```typescript
/**
 * From T, pick a set of properties whose keys are in the union K
 */
type Pick<T, K extends keyof T> = {
  [P in K]: T[P]
}
```

如果你理解了最开始的那个 pluck 函数，这个就很好理解了，我们传入 T 和 K，其中 K 是 T 的 keys 组成的联合类型，再看返回值`[P in K]: T[P]`，就是把 K 遍历了一遍，同时赋值上原类型，那么综合来看 Pick 就是帮我们提取出某些类型的，比如通过`Pick<Person, 'name'>`我们就可以得到`{name: string}`，再来看下一个

```typescript
/**
 * Exclude from T those types that are assignable to U
 */
type Exclude<T, U> = T extends U ? never : T
```

这个泛型传入一个 T 和 U，然后它判断了 T 是否属于 U，属于的话返回 never 否则返回原类型 T，注意 never 在最终的类型中是不会存在的，所以它可以帮助我们消除某些属性，其实这个 Exclude 就是消除了`T extends U`的类型，比如我们使用`Exclude<'a'|'b','b'|'c'>`，最终会得到`'a'`，与之相反的有：

```typescript
/**
 * Extract from T those types that are assignable to U
 */
type Extract<T, U> = T extends U ? T : never
```

这个正好相反，是从 T 中取出 U 中拥有的类型。

有了 Exclude，我们就可以和 Pick 结合来实现另外一个：

```typescript
/**
 * Construct a type with the properties of T except for those in type K.
 */
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>
```

这个泛型是先使用了`Exclude<keyof T, K>`，去除了 keyof T 中的 K，然后又使用 Pick 取出了这些类型，这样我们就可以从 T 中去除 K 里面包含的 keys 了，达到和 Pick 相反的效果。

我们再来看另一个稍微复杂一点的

```typescript
type NonNullObject<O> = Pick<
  O,
  {
    [K in keyof O]: O[K] extends null | undefined ? never : K
  }[keyof O]
>
```

这个不是 TS 内置的类型，但也是一个很有用的类型，我们来一点一点分析。首先这个泛型使用了 Pick，我们知道 Pick 就是取出一些属性，我们先看传给 Pick 的第二个参数

```typescript

{
  [K in keyof O]: O[K] extends null | undefined ? never : K
}[keyof O]

```

它遍历了 O 的 keys，然后进行了一个判断，如果是`extends null | undefined`则返回 never，否则返回 K，K 就是 O 中的 key 值，注意这里和之前的一些泛型有些不一样,之前的都是`O[K]`，而这里的属性的值还是 K，最终我们得到的是类似`K:K`这样的东西，比如`{name: string, age: null}`这个，经过上面的转化会变成`{name:'name', age:never}`，可能有些小伙伴还不清楚为什么要这样转换，我们接着往下分析，经过这个转换之后，又进行了一个操作`[keyof O]`，对于 Person，keyof O 就是`'name'|'age'`，那么这里就就是`{name:'name', age:never}['name'|'age']`，这样就很清晰了，其实就是一个取值操作，这样我们就可以得到`'name'|never`，还记得 never 的特性吗，它可以帮我们消除一些类型，那么最终的就是'name'，这也是为什么我们写成类似 K:K 这样，就是要把 null|undefined 对应的 key 转换成 never，然后再通过 keyof 把他们全都取出来，别忘了最外面还有一个 Pick，这样我们就从原始类型中去除了 null|undefined。

另外还有一个比较有用的是 ReturnType

```typescript
/**
 * Obtain the return type of a function type
 */
type ReturnType<T extends (...args: any) => any> = T extends (
  ...args: any
) => infer R
  ? R
  : any
```

它可以帮我们取到函数返回值的类型，这个 ReturnType 接收的一个参数是函数，然后进行了一个判断`T extends (...args: any) => infer R`，就是判断是否是函数，这里有个东西是 infer，通过这个操作符我们可以获取 R 的引用，就是函数的返回值，最终再把 R 返回出去，就获得了函数 T 的返回值。

其实除了我分析的这些泛型，TS 还内置了其他的很多泛型，比如还有获取函数的参数的，获取构造函数类型的，总的来说各种泛型基本上都可以用索引类型和映射类型实现，希望大家看过这篇文章后能多多使用这两种类型，在自己的项目里也能开发一些常用的辅助泛型，来提升工作效率。

