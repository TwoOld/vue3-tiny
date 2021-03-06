# vue3 剖析

## Composition API RFC

不得不说，vue 的语法，越来越 react 了，感觉 vue3 发布后，会有更多人专项 react 了

先看下 vue3 的计算属性和点击累加的代码

[vue-next 下载源码](https://github.com/vuejs/vue-next)

执行 npm run dev 调试打个包

根目录下创建 examples/vue/index.html

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <title>Document</title>
    <script src="../../packages/vue/dist/vue.global.js"></script>
  </head>

  <body>
    <div id="app"></div>
    <script>
      const { createApp, reactive, comppeted, effect } = Vue
      const AComponent = {
        template: `<button @click="increment">Count is {{state.count}}</button>`,
        setup() {
          const state = reactive({
            count: 0
          })
          effect(() => {
            console.log('count is changed', state.count)
          })
          function increment() {
            state.count += 1
          }
          return {
            state,
            increment
          }
        }
      }
      createApp().mount(AComponent, '#app')
    </script>
  </body>
</html>
```

这个**reactive**和 react-hooks 越来越像了, 大家可以去[Composition API RFC](https://vue-composition-api-rfc.netlify.com/#api-introduction)这里看细节

## 功能

编译器（Compiler）的优化主要在体现在以下几个方面：

- 使用模块化架构
- 优化 "Block tree"
- 更激进的 static tree hoisting 功能
- 支持 Source map
- 内置标识符前缀（又名 "stripWith"）
- 内置整齐打印（pretty-printing）功能
- 移除 source map 和标识符前缀功能后，使用 Brotli 压缩的浏览器版本精简了大约 10KB

运行时（Runtime）的更新主要体现在以下几个方面：

- 速度显著提升
- 同时支持 Composition API 和 Options API，以及 typings
- 基于 Proxy 实现的数据变更检测
- 支持 Fragments
- 支持 Portals
- 支持 Suspense w/ async setup()

最后，还有一些 2.x 的功能尚未移植过来，如下：

- 服务器端渲染
- keep-alive
- transition
- Compiler DOM-specific transforms
- v-on DOM 修饰符 v-model v-text v-pre v-onc v-html v-show

## typescript

全部由 typescript 构建，三大库的最终选择，ts 乃今后必学技能

## proxy 取代 deineProperty

除了性能更高以为，还有以下缺陷，也是为啥会有`$set`，`$delete`的原因
1、属性的新加或者删除也无法监听；
2、数组元素的增加和删除也无法监听

## reactive 模块

看源码

```typescript
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (readonlyToRaw.has(target)) {
    return target
  }
  // target is explicitly marked as readonly by user
  if (readonlyValues.has(target)) {
    return readonly(target)
  }
  return createReactiveObject(
    target,
    rawToReactive,
    reactiveToRaw,
    mutableHandlers,
    mutableCollectionHandlers
  )
}

function createReactiveObject(
  target: any,
  toProxy: WeakMap<any, any>,
  toRaw: WeakMap<any, any>,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>
) {
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }
  // target already has corresponding Proxy
  let observed = toProxy.get(target)
  if (observed !== void 0) {
    return observed
  }
  // target is already a Proxy
  if (toRaw.has(target)) {
    return target
  }
  // only a whitelist of value types can be observed.
  if (!canObserve(target)) {
    return target
  }
  const handlers = collectionTypes.has(target.constructor)
    ? collectionHandlers
    : baseHandlers
  observed = new Proxy(target, handlers)
  toProxy.set(target, observed)
  toRaw.set(observed, target)
  if (!targetMap.has(target)) {
    targetMap.set(target, new Map())
  }
  return observed
}
```

稍微精简下

```js
function reactive(target) {
  const handlers = collectionTypes.has(target.constructor)
    ? collectionHandlers
    : baseHandlers
  observed = new Proxy(target, handlers)
  return observed
}
```

基本上除了 set map weakset 和 weakmap，都是 baseHandlers，下面重点关注一下，Proxy 的语法 这里可能需要看看相关的 es6 知识点

```typescript
export const readonlyHandlers: ProxyHandler<any> = {
  get: createGetter(true),

  set(target: any, key: string | symbol, value: any, receiver: any): boolean {
    if (LOCKED) {
      if (__DEV__) {
        console.warn(
          `Set operation on key "${key as any}" failed: target is readonly.`,
          target
        )
      }
      return true
    } else {
      return set(target, key, value, receiver)
    }
  },

  deleteProperty(target: any, key: string | symbol): boolean {
    if (LOCKED) {
      if (__DEV__) {
        console.warn(
          `Delete operation on key "${key as any}" failed: target is readonly.`,
          target
        )
      }
      return true
    } else {
      return deleteProperty(target, key)
    }
  },

  has,
  ownKeys
}
```

## 关于 proxy

proxy.js

```js
let data = [1, 2, 3]
let o = new Proxy(obj, {
  get(target, key) {
    console.log('get', key)
    return Reflect.get(target, key)
  },
  set(target, key, val) {
    console.log('set', key, val)
    return Reflect.set(target, key, val)
  }
})

o.push(4)

// get push
// get length
// set 3 4
// set length 4
```

比 defineproperty 优秀的 就是数组和对象都可以直接触发 getter 和 setter， 但是数组会触发两次，因为获取 push 和修改 length 的时候也会触发

还可以用**Reflect**

```js
let data = [1, 2, 3]
let o = new Proxy(obj, {
  get(target, key) {
    console.log('get', key)
    return Reflect.get(target, key)
  },
  set(target, key, val) {
    console.log('set', key, val)
    return Reflect.set(target, key, val)
  }
})

o.push(4)
```

多次触发和深层嵌套问题，一会看 vue3 是怎么解决的

```js
let data = { name: { title: 'kkb' } }
let o = new Proxy(obj, {
  get(target, key) {
    console.log('get', key)
    return Reflect.get(target, key)
  },
  set(target, key, val) {
    console.log('set', key, val)
    return Reflect.set(target, key, val)
  }
})

o.name.title = 'xx'

// get name
```

## vue3 深度检测

baseHander

```js
function createGetter(isReadonly: boolean) {
  return function get(target: any, key: string | symbol, receiver: any) {
    const res = Reflect.get(target, key, receiver)
    if (typeof key === 'symbol' && builtInSymbols.has(key)) {
      return res
    }
    if (isRef(res)) {
      return res.value
    }
    track(target, OperationTypes.GET, key)
    return isObject(res)
      ? isReadonly
        ? // need to lazy access readonly and reactive here to avoid
          // circular dependency
          readonly(res)
        : reactive(res)
      : res
  }
}
```

返回值如果是 object，就再走一次 reactive，实现深度

## vue3 处理重复 trigger

很简单，用的 hasOwProperty, set 肯定会出发多次，但是通知只出去一次， 比如数组修改 length 的时候，hasOwProperty 是 true， 那就不触发

```js
function set(
  target: any,
  key: string | symbol,
  value: any,
  receiver: any
): boolean {
  value = toRaw(value)
  const hadKey = hasOwn(target, key)
  const oldValue = target[key]
  if (isRef(oldValue) && !isRef(value)) {
    oldValue.value = value
    return true
  }
  const result = Reflect.set(target, key, value, receiver)
  // don't trigger if target is something up in the prototype chain of original
  if (target === toRaw(receiver)) {
    /* istanbul ignore else */
    if (__DEV__) {
      const extraInfo = { oldValue, newValue: value }
      if (!hadKey) {
        trigger(target, OperationTypes.ADD, key, extraInfo)
      } else if (value !== oldValue) {
        trigger(target, OperationTypes.SET, key, extraInfo)
      }
    } else {
      if (!hadKey) {
        trigger(target, OperationTypes.ADD, key)
      } else if (value !== oldValue) {
        trigger(target, OperationTypes.SET, key)
      }
    }
  }
  return result
}
```

## 手写 vue3 的 reactive

刚才说的细节，我们手写一下

### effect

### computed

```js
// 用这个方法来模式视图更新
function updateView() {
  console.log('触发视图更新啦')
}
function isObject(t) {
  return typeof t === 'object' && t !== null
}

// 把原目标对象 转变 为响应式的对象
const options = {
  set(target, key, value, reciver) {
    // console.log(key,target.hasOwnProperty(key))
    if (!target.hasOwnProperty(key)) {
      updateView()
    }
    return Reflect.set(target, key, value, reciver)
  },
  get(target, key, reciver) {
    const res = Reflect.get(target, key, reciver)
    if (isObject(target[key])) {
      return reactive(res)
    }
    return res
  },
  deleteProperty(target, key) {
    return Reflect.deleteProperty(target, key)
  }
}
// 用来做缓存
const toProxy = new WeakMap()

function reactive(target) {
  if (!isObject(target)) {
    return target
  }
  // 如果已经代理过了这个对象，则直接返回代理后的结果即可
  if (toProxy.get(target)) {
    return toProxy.get(target)
  }
  let proxyed = new Proxy(target, options)
  toProxy.set(target, proxyed)
  return proxyed
}

// 测试数据
let obj = {
  name: 'Ace7523',
  array: ['a', 'b', 'c']
}

// 把原数据转变响应式的数据
let reactivedObj = reactive(obj)

// 改变数据，期望会触发updateView() 方法 从而更新视图
// reactivedObj.name = 'change'

// reactivedObj.array.unshift(4)
```

其他细节 track 收集依赖，trigger 触发更新

## vue3 其他模块细节

代码仓库中有个 packages 目录，里面主要是 Vue 3.0 的相关源码功能实现，具体内容如下所示。

### compiler-core

平台无关的编译器，它既包含可扩展的基础功能，也包含所有平台无关的插件。暴露了 AST 和 baseCompile 相关的 API，它能把一个字符串变成一棵 AST

### compiler-dom

针对浏览器的编译器。

### runtime-core

与平台无关的运行时环境。支持实现的功能有虚拟 DOM 渲染器、Vue 组件和 Vue 的各种 API, 可以用来自定义 renderer ，vue2 中也有 ，入口代码看起来

### runtime-dom

针对浏览器的 runtime。其功能包括处理原生 DOM API、DOM 事件和 DOM 属性等， 暴露了重要的 render 和 createApp 方法

```js
const { render, createApp } = createRenderer<Node, Element>({
  patchProp,
  ...nodeOps
})

export { render, createApp }
```

### runtime-test

一个专门为了测试而写的轻量级 runtime。比如对外暴露了**renderToString**方法，在此感慨和 react 越来越像了

### server-renderer

用于 SSR，尚未实现。

### shared

没有暴露任何 API，主要包含了一些平台无关的内部帮助方法。

### vue

用于构建「完整」版本，引用了上面提到的 runtime 和 compiler 目录。入口文件代码如下

```js
'use strict'

if (process.env.NODE_ENV === 'production') {
  module.exports = require('./dist/vue.cjs.prod.js')
} else {
  module.exports = require('./dist/vue.cjs.js')
}
```

所以想阅读源码，还是要看构建流程，这个和 vue2 也是一致的

未完待续
