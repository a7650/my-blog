### 前言

前段时间在 GitHub 上发现了一个 TypeScript 练习题的仓库：[typescript-exercises](https://github.com/typescript-exercises/typescript-exercises)，里面一共 16 道题，包含了 TS 的初级到高级的用法，我从这些题中挑选了几道，我们一起来看下，看看你能作对几道，正好可以巩固下知识：

### 练习题

> 每道题都有 index.ts 文件和 test.ts 文件，index 代码中的某部分是缺少的，需要我们去完善。test 中是一些测试，测试通过即为解答成功。

> 某些题目中的 test 或者辅助代码可能比较多，本文就不把这些 test 全部贴出来了，我会在下面每个题目的后面写上该题目的 GitHub 地址，你可以在点进去查看详细的题目，建议将整个项目 clone 下来，在自己的电脑上查看。

#### exercises-7

##### GitHub 地址：[exercises-7](https://github.com/typescript-exercises/typescript-exercises/tree/master/src/exercises/7)

##### 题目

```ts
// index.ts
export function swap(v1, v2) {
  return [v2, v1]
}
```

```ts
// test.ts
import { swap } from './index'

const pair1 = swap(123, 'hello') // 此时pair1的类型为any
typeAssert<IsTypeEqual<typeof pair1, [string, number]>>() // 这一行会报错因为any和[string, number]不等
```

##### 解答

从 test 中可以知道，我们需要使`swap(123, 'hello')`的返回结果的类型为`[string, number]`，就是将入参反过来，可以先观察下 swap 函数，它的参数和返回值没有指定任何类型，这里我们可以使用泛型来完善它：

```ts
export function swap<T1, T2>(v1: T1, v2: T2): [T2, T1] {
  return [v2, v1]
}
```

这样一来，TS 就会推断出`swap(123, 'hello')`的返回值类型是`[string, number]`了。

#### exercises-8

##### GitHub 地址：[exercises-8](https://github.com/typescript-exercises/typescript-exercises/blob/master/src/exercises/8)

##### 题目

```ts
// index.ts
interface User {
  type: 'user'
  name: string
  age: number
  occupation: string
}

interface Admin {
  type: 'admin'
  name: string
  age: number
  role: string
}

/*
  定义PowerUser类型，使它拥有User和Admin的所有字段，其中的type字段为 "powerUser"
*/
type PowerUser = unknown

export type Person = User | Admin | PowerUser
```

##### 解答

这里是需要我们完善 PowerUser 类型，使它拥有 User 和 Admin 的所有字段，并且它的 type 字段为 "powerUser"，很显然，这里需要使用联合类型：

```ts
type PowerUser = User & Admin & { type: 'powerUser' } // 得到的类型为any
```

如果这样写的话，你就会发现 PowerUser 类型变成了 any，这是为什么呢？观察 User 和 Admin 类型可以发现这两个类型中都含有 type 且 type 都为特定的字符串，这样的话我们直接用`&`操作符是有问题的，因为一个字符串不可能同时是`'user'`和`'admin'`，那么该怎么办呢？其实只需要将 User 和 Admin 中的 type 去掉，然后再合并就可以了：

```ts
type PowerUser = Omit<User, 'type'> &
  Omit<Admin, 'type'> & {
    type: 'powerUser'
  }
```

这样一来就达到题目的要求了，其中用了辅助泛型`Omit`。

#### exercises-9

##### GitHub 地址：[exercises-9](https://github.com/typescript-exercises/typescript-exercises/tree/master/src/exercises/9)

##### 题目

```ts
// index.ts
```

```ts
// test.ts
```

##### 解答
