### 前言

前段时间在 GitHub 上发现了一个 TypeScript 练习题的仓库：[typescript-exercises](https://github.com/typescript-exercises/typescript-exercises)，里面一共 15 道题，包含了 TS 的初级到高级的用法，我从这些题中挑选了几道，我们一起来看下，看看你能作对几道，正好可以巩固下知识，废话不多说，我们直接开始：

### 练习题

> 某些题目中的代码可能比较多，本文就不把这些全部贴出来了，我会在下面每个题目的后面写上该题目的 GitHub 地址，你可以在点进去查看详细的题目，建议将整个项目 clone 下来，在自己的电脑上查看。

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
/**
  这个题目太长了，比较占地方，大家就点击上方的链接查看吧，这里我只贴出来题目要求：
  删除UsersApiResponse和AdminsApiResponse类型，并使用通用类型ApiResponse来为每一个API函数指定响应的格式
*/
export type ApiResponse<T> = unknown
```

##### 解答

我们可以先看下题目中的响应数据的格式

```ts
type AdminsApiResponse =
  | {
      status: 'success'
      data: Admin[]
    }
  | {
      status: 'error'
      error: string
    }
type UsersApiResponse =
  | {
      status: 'success'
      data: User[]
    }
  | {
      status: 'error'
      error: string
    }
```

每个响应数据都有成功和失败的状态，成功时返回的是 data，失败时返回的是 error，error 的类型都是 string，只有 data 不一样。

按照题目的要求，我们需要写一个通用的类型，这里我们可以用泛型来进行重构：

```ts
export type ApiResponse<T> =
  | { status: 'success'; data: T }
  | { status: 'error'; error: string }
```

使用的时候只需要传入泛型就可以了：

```ts
export function requestAdmins(
  callback: (response: ApiResponse<Admin[]>) => void
) {
  callback({
    status: 'success',
    data: admins
  })
}
```

其他函数同理，也是这样改造。

前面几道题都是比较简单的，下面我们看下稍微复杂点的题目。

#### exercises-10

##### GitHub 地址：[exercises-10](https://github.com/typescript-exercises/typescript-exercises/tree/master/src/exercises/10)

##### 题目

```ts
// index.ts
/**
具体题目点击上方链接可以看到，这里只贴出来题目要求：
  Exercise:
    我们不想重新实现所有的数据请求功能。
    让我们装饰一下旧的基于回调的函数，
    其结果与promise兼容。
    函数最终应返回一个promise，这个promise会resolve最终的数据或reject一个error，
    函数名为promisify。

    更高难度的练习:

    创建一个函数promisifyAll，
    它接受带有函数的对象，
    并返回一个新对象，
    其中每个函数都是promised的。

    const api = promisifyAll(oldApi);
*/
```

```ts
// test.ts
```

##### 解答

这道题就是需要我们把旧的基于回调的 API 改成 promise，我们先来看下这些 API 的实现：

```ts
const oldApi = {
  requestAdmins(callback: (response: ApiResponse<Admin[]>) => void) {
    callback({
      status: 'success',
      data: admins
    })
  },
  requestUsers(callback: (response: ApiResponse<User[]>) => void) {
    callback({
      status: 'success',
      data: users
    })
  },
  requestCurrentServerTime(callback: (response: ApiResponse<number>) => void) {
    callback({
      status: 'success',
      data: Date.now()
    })
  },
  requestCoffeeMachineQueueLength(
    callback: (response: ApiResponse<number>) => void
  ) {
    callback({
      status: 'error',
      error: 'Numeric value has exceeded Number.MAX_SAFE_INTEGER.'
    })
  }
}

export const api = {
  requestAdmins: promisify(oldApi.requestAdmins),
  requestUsers: promisify(oldApi.requestUsers),
  requestCurrentServerTime: promisify(oldApi.requestCurrentServerTime),
  requestCoffeeMachineQueueLength: promisify(
    oldApi.requestCoffeeMachineQueueLength
  )
}
```

需要实现一个 promisify 函数，这个函数接收一个旧的 api 函数，返回一个新的函数，这个新的函数是已经被 promise 包装好的函数，我们用 js 可以很轻易的实现这个 promisify：

```js
export function promisify(fn) {
  return () =>
    new Promise((resolve, reject) => {
      fn((response) => {
        if (response.status === 'success') {
          resolve(response.data)
        } else {
          reject(new Error(response.error))
        }
      })
    })
}
```

然后我们把它改成 TS 版的，使能够正确推导出类型。首先就是要确定 promisify 的参数 fn，这个 fn 的类型就是 oldApi 中的函数类型，只需要照着写就行了：

```ts
// 这个函数的参数是一个callback函数，callback的参数是response: ApiResponse<T>，泛型T是接口的返回值，如：Admin[]
type CallbackBasedAsyncFunction<T> = (
  callback: (response: ApiResponse<T>) => void
) => void
```

然后再确定 promisify 的返回值，如上文所说，它的返回值应该是个新的函数，这个新的函数返回一个 promise：

```ts
type PromiseBasedAsyncFunction<T> = () => Promise<T>
```

函数内部的逻辑比较简单，没什么需要改的地方，只有一个就是返回`new Promise`的时候记得加上泛型 T，最终我们 promisify 函数就是这样的：

```ts
export function promisify<T>(
  fn: CallbackBasedAsyncFunction<T>
): PromiseBasedAsyncFunction<T> {
  return () =>
    new Promise<T>((resolve, reject) => {
      fn((response) => {
        if (response.status === 'success') {
          resolve(response.data)
        } else {
          reject(new Error(response.error))
        }
      })
    })
}
```

然后再试一下：

```ts
const requestAdmins = promisify(oldApi.requestAdmins)
```

可以正确得到 requestAdmins 的返回值类型`PromiseBasedAsyncFunction<Admin[]>`。

这道题还有个更高难度的练习，就是要实现一个 promisifyAll 函数，将对象中的所有函数都改为基于 promise 的。

要实现这个函数，首先我们要确定的是函数的参数和返回值是什么，其实参数就是`oldAPi`这个对象，类型就是 oldApi 的类型：

```ts
type OldApi = {
  requestAdmins(callback: (response: ApiResponse<Admin[]>) => void): void
  requestUsers(callback: (response: ApiResponse<User[]>) => void): void
  requestCurrentServerTime(
    callback: (response: ApiResponse<number>) => void
  ): void
  requestCoffeeMachineQueueLength(
    callback: (response: ApiResponse<number>) => void
  ): void
}
```

我们肯定不能直接就把这个类型用在函数中，可以想办法简化一下，这里面的每一个函数除了 callback 的参数 response 不同，其余的都是相同的，那么我们可以把这个不同之处提取出来：

```ts
type ApiResponses = {
  requestAdmins: Admin[]
  requestUsers: User[]
  requestCurrentServerTime: number
  requestCoffeeMachineQueueLength: number
}

type SourceObject = {
  [K in keyof ApiResponses]: CallbackBasedAsyncFunction<ApiResponses[K]>
}
```

这里用到了上一问题里创建的 CallbackBasedAsyncFunction 类型，其实 SourceObject 可以再进一步，把 ApiResponses 作为泛型：

```ts
type SourceObject<T> = { [K in keyof T]: CallbackBasedAsyncFunction<T[K]> }
```

我们只需要传入 ApiResponses 就可以得到 OldApi：

```ts
type OldApi = SourceObject<ApiResponses>
```

现在我们可以得到 promisifyAll：

```ts
function promisifyAll<T>(obj: SourceObject<T>): unknown {}
```

这里有人可能会认为在使用时需要传入泛型 T，即:`promisifyAll<ApiResponses>(oldApi)`，这里是不需要的，刚才的 ApiResponses 只是我们思考时的产物，你在使用时只需要`promisifyAll(oldApi)`就可以了，这是因为泛型是双向的，只要你传入了 oldApi，TS 就会自己推导出泛型 T 为 ApiResponses。因为在 SourceObject 中对 T 做了遍历操作，其实这里我们默认了 T 为对象形式，那么为了不出现意外情况，需要再给 T 加上泛型约束：

```ts
function promisifyAll<T extends { [key: string]: any }>(
  obj: SourceObject<T>
): unknown {}
```

然后还有返回值的类型，有了 SourceObject 的基础，返回值的类型就很好写了，可以使用上一问题中的 CallbackBasedAsyncFunction 类型：

```ts
type SourceObject<T> = { [K in keyof T]: CallbackBasedAsyncFunction<T[K]> }
type PromisifiedObject<T> = { [K in keyof T]: PromiseBasedAsyncFunction<T[K]> }
function promisifyAll<T extends { [key: string]: any }>(
  obj: SourceObject<T>
): PromisifiedObject<T> {}
```

然后还有函数内部的逻辑，其实函数内部的逻辑是比较简单的，我们先来用 js 实现一版：

```js
function promisifyAll(obj) {
  const result = {}
  for (const key of Object.keys(obj)) {
    result[key] = promisify(obj[key])
  }
  return result
}
```

然后和之前的结合，改成 TS 版的，内部逻辑比较简单，就不分析了，只要最终返回值是`PromisifiedObject<T>`就可以：

```ts
type SourceObject<T> = { [K in keyof T]: CallbackBasedAsyncFunction<T[K]> }
type PromisifiedObject<T> = { [K in keyof T]: PromiseBasedAsyncFunction<T[K]> }
function promisifyAll<T extends { [key: string]: any }>(
  obj: SourceObject<T>
): PromisifiedObject<T> {
  const result = {} as PromisifiedObject<T>
  for (const key of Object.keys(obj) as (keyof T)[]) {
    result[key] = promisify(obj[key])
  }
  return result
}
```

使用时：

```ts
export const api = promisifyAll(oldApi)
```

可以正确拿到api的类型。

### 总结

这几道题你是否能成功解出来呢？解不出来也没关系，我们可以慢慢学。这里我推荐你把这个[git仓库](https://github.com/typescript-exercises/typescript-exercises)的代码拉下来，这里面从易到难共15道题，你可以从第一题最简单的开始，自己尝试下，相信在这个过程中你一定能学到一些知识。
