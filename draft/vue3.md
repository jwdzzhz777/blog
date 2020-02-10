# Vuejs3.0 源码解读

之前阅读一遍 `Vuejs` 的源码，并做了[记录][my-vue]，现在 `vue3` 都出了 `alpha` 版了，正好再来看看 `Vue3` 做了哪些改变。

> note: 本篇是在[之前][my-vue]的基础上做的延展，所以和 `Vuejs2.0` 类似的部分不会再提。同样的，代码部分会有删减/修改，建议配合源码食用。

源码地址：[vue-next repository][vue-next]

当前阅读版本为 [v3.0.0-alpha.4][alpha4]

在这之前，最好先了解下 `Vue3` 这里有官方的[文章][official blog]。

## 有什么不一样

对于我们使用者来说 3.0 对于 2.0 多少会有些不同，简单说下：

### new App

首先是 `Vue` 本身，其不再是一个构造方法了，而是一个对象，或者是一个集合，集合了各种方法，也就是说我们构建一个 `app` 的方法不同了：

```js
// old
import Vue from 'vue'
import App from './App.vue'

Vue.config.ignoredElements = [/^app-/]
Vue.use(/* ... */)
Vue.mixin(/* ... */)
Vue.component(/* ... */)
Vue.directive(/* ... */)

new Vue({
  render: h => h(App)
}).$mount('#app')

// new
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

app.config.ignoredElements = [/^app-/]
app.use(/* ... */)
app.mixin(/* ... */)
app.component(/* ... */)
app.directive(/* ... */)

app.mount('#app')
```

### reactivity data

`Vue3` 将响应式数据那块的代码单独提出，成了一个单独的组件 `reactivity`，可以脱离 `Vue` 单独使用，就像之前劫持数据，数据变化通知观察者，现在直接将数据/代理和方法绑定，数据变化直接调用绑定方法：

```js
import { ref, computed, effect } from 'vue'

let a = ref('hello');
let b = ref('world');
let h = computed(() => `${a.value} ${b.value}`);

effect(function f() {
    console.log(h.value);
});

b.value = 'vue';

// 'hello world'
// 'hello vue'
```

### Composition API

之前向 `Vue` 添加逻辑我们会用各种字段拓展：`data`、`methods`、`computed` 等等。现在有一种全新的语法，通过 `setup` 方法配合 `reactivity` 来统一处理以上方法:

```vue
<template>
  <input :value="input" @input="update"></input>
</template>

<script>
import { ref, onMounted } from 'vue'

export default {
    setup() {
        const input = ref('hello')
        const update = e => {
            input.value = e.target.value
        }

        onMounted(() => console.log('component mounted!'))

        return {
            input,
            update
        }
    }
}
</script>
```

当然包括生命周期也放在了 `setup` 函数中处理，函数的好处，一是便于阅读，二是能从类型检查中收益（Typescript）。

不过 `Vue3` 对 2.0 版本做了兼容，也就是你也可以直接沿用之前的用法，不冲突。

这些是我认为比较重要的变化，现在我们来透过源码看看这些变化。

## reactivity

整个 `Vue3` 的重构个人感觉最重要的就是它了，它将 响应式数据 这一核心从框架中抽出来，其核心还是观察者模式，改变了实现的方式，现在我们来一起了解下。

```js
import { ref, computed, effect } from 'vue';

let a = ref('hello');
let b = ref('world');
let h = computed(() => `${a.value} ${b.value}`);

effect(function f() {
    console.log(h.value);
});

b.value = 'vue';

// 'hello world'
// 'hello vue'
```

还是刚才的例子，它发生了什么？我们知道 `computed` 方法，每次 `a` 或 `b` 的值变化会带动 `h` 的变化，而每次 `h` 的变化都会调用 `f`, 也就是说，数据和方法绑定了起来，观察者从之前的对象变成了现在的函数。大致逻辑知道了，我们来一个一个看看。

### reactive && ref

要实现响应式功能，首先需要处理 对象/值 ，`reactivity` 提供两个方法，    其中 `reactive` 负责代理并封装对象，`ref` 负责包装 值/对象，并生成一个新的对象。主要就是劫持 `getter` 和 `setter`，并做一些处理：

#### reactive

```js
import { mutableCollectionHandlers } from './collectionHandlers'
import { mutableHandlers } from './baseHandlers'

const rawToReactive = new WeakMap<any, any>()
const reactiveToRaw = new WeakMap<any, any>()

function reactive(target) {
  return createReactiveObject(
    target,
    rawToReactive,
    reactiveToRaw,
    mutableHandlers,
    mutableCollectionHandlers
  )
}

function createReactiveObject(
  target: unknown,
  toProxy: WeakMap<any, any>,
  toRaw: WeakMap<any, any>,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>
) {
  if (!isObject(target)) {
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
  return observed
}
```

简单的说，`reactive` 就是将传入对象做了一层代理 `observed = new Proxy(target, handlers)` 并返回。但是如果我们多次对一个对象调用 `Proxy`，就会产生多个该对象的代理，所以就需要对代理对象进行缓存。`Vue` 通过两个 `WeakMap` (key 为 Object 的 Map) 来存储 `traget` 与 `observed` :

```js
rawToReactive.set(target, observed)
reactiveToRaw.set(observed, target)
```

两者互相绑定，有点枚举的味道。如果传入对象以及被代理了，或其本身就是其他对象的代理对象，那么就不做处理了。

在看看代理的 `handlers` 如何处理：

#### get handler

```js
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: object, key: string | symbol, receiver: object) {
    // 重写内置函数 'includes', 'indexOf', 'lastIndexOf'
    if (isArray(target) && hasOwn(arrayIdentityInstrumentations, key)) {
      return Reflect.get(arrayIdentityInstrumentations, key, receiver)
    }
    const res = Reflect.get(target, key, receiver)
    // 默认返回内置 Symbol 字段值
    if (isSymbol(key) && builtInSymbols.has(key)) return res
    if (shallow) {
      track(target, TrackOpTypes.GET, key)
      // TODO strict mode that returns a shallow-readonly version of the value
      return res
    }
    // 如果是 ref 生成的对象返回其 value。其本身自带同样的 `getter` 直接返回。
    if (isRef(res)) {
      return res.value
    }
    track(target, TrackOpTypes.GET, key)
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

简单的说，`getter` 就是传递原对象对应 `key` 的值。并调用 `track` 方法。

`createGetter` 用于创建 `get handler`。而 `isReadonly`，`shallowd` 分别对应创建只读代理和浅代理（有对应的方法，这里不贴了自己去看源码）。调用对应方法会传入相应的标志。

因为 `Proxy` 是浅引用，所以要代理整个对象的话需要递归代理。所以当结果为对象时返回 `reactive(res)`（放在 get 很赞）同时如果是 `readOnly` 对象返回 `readonly(res)` （就是创建只读版的 `reactive`）。当然要保持浅引用的话也就调用相应方法 (`shallow = true`) 这样对应值是对象就直接返回。

#### set handler

```js
function createSetter(isReadonly = false, shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    if (isReadonly && LOCKED) return true

    const oldValue = (target as any)[key]
    if (!shallow) {
      value = toRaw(value)
      if (isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }

    const hadKey = hasOwn(target, key)
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
    if (target === toRaw(receiver)) {
      const extraInfo = { oldValue, newValue: value }
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, extraInfo)
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, extraInfo)
      }
    }
    return result
  }
}
```

简单的说，也就是给被代理的对象 `target` 对应的 `key` 设置新的值 `Reflect.set(target, key, value, receiver)` ，并调用 `trigger`。当然也有一些特别情况：

* `isReadonly = true` 只读对象不给设置
* 当被代理对象上的对应值 `target[key]` 为 `ref` 方法生成对象，更新值并 `return` （其本身有 `set handler`）
* this 被指向其他对象时，不调用 `trigger`

#### has handler

```js
function has(target: object, key: string | symbol): boolean {
  const result = Reflect.has(target, key)
  track(target, TrackOpTypes.HAS, key)
  return result
}
```

`has` 比较简单，就多了个调用 `track`

#### ref



[my-vue]:https://github.com/jwdzzhz777/blog/blob/master/articles/vue.md
[vue-next]:https://github.com/vuejs/vue-next
[alpha4]:https://github.com/vuejs/vue-next/releases/tag/v3.0.0-alpha.4
[official blog]:https://www.vuemastery.com/blog/top-ways-to-learn-Vue-3/