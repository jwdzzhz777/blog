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

`has` 比较简单，就多了个调用 `track`。

#### ref

如果说 `reactive` 负责代理引用类型，`ref` 则负责包装基本类型，生成一个带有 `_isRef = true` 标记的对象:

```ts
const convert = (val) =>
  isObject(val) ? reactive(val) : val

function ref(value?: unknown) {
  if (isRef(value)) {
    return value
  }
  value = convert(value)
  const r = {
    _isRef: true,
    get value() {
      track(r, TrackOpTypes.GET, 'value')
      return value
    },
    set value(newVal) {
      value = convert(newVal)
      trigger(
        r,
        TriggerOpTypes.SET,
        'value',
        __DEV__ ? { newValue: newVal } : void 0
      )
    }
  }
  return r
}
```

简单的说就是包装成对象，其值通过 `value` 字段获得，这样就可以在 `getter`、`setter` 时调用 `track` 和 `trigger`。如果是对象会将其包装成响应式对象 `reactive(val)`

好了，现在我们已经知道了 `reactive & ref` 是如何包装、处理值和对象使其变为响应式数据的，但是还没看 `track`、`trigger` 方法是干嘛的，其实他两就是和 `effect` 联系起来的‘桥梁’。接下来我们了解下 `effect`.

### effect

先来看看源码：

```ts
function effect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions = EMPTY_OBJ
): ReactiveEffect<T> {
  if (isEffect(fn)) {
    fn = fn.raw
  }
  const effect = createReactiveEffect(fn, options)
  if (!options.lazy) {
    effect()
  }
  return effect
}

function createReactiveEffect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions
): ReactiveEffect<T> {
  const effect = function reactiveEffect(...args: unknown[]): unknown {
    return run(effect, fn, args)
  } as ReactiveEffect
  effect._isEffect = true
  effect.active = true
  effect.raw = fn
  effect.deps = []
  effect.options = options
  return effect
}

function run(effect: ReactiveEffect, fn: Function, args: unknown[]): unknown {
  if (!effect.active) {
    return fn(...args)
  }
  if (!effectStack.includes(effect)) {
    cleanup(effect)
    try {
      effectStack.push(effect)
      activeEffect = effect
      return fn(...args)
    } finally {
      effectStack.pop()
      activeEffect = effectStack[effectStack.length - 1]
    }
  }
}
```

简单的说 `effect` 会将方法 `fn` 处理成

```js
const effect = function (...args) {
    return run(effect, fn, args)
}
```

> note: 为了区分，下面 `effect` 指得是这个生成的方法 `Effect` 指的是原方法。

并返回，同时还会调用一把(注意调用的是 `effect` 不是原方法 `fn`)。当然还会给生成的 `effect` 绑定一些属性。我们先来看看 `run` 方法。

`run` 方法的主要目的是：将全局的 `activeEffect` 绑定为当前 `effect` 并调用 `fn`。也就是当 `Effect` 处理并运行 `fn` 时我们能通过 `activeEffect` 得到当前 `Effect` 生成的 `effect` 函数并做一些绑定 (`getter` 和 `effect`)。

`effectStack` 是一个单纯的数组，用于暂存 `effect`,配合 `activeEffect` 使用。目的是处理嵌套情况，也就是 `fn` 中有处理其他 `Effect` 时。每调用 `effect` 就将其 `push` 入 `effectStack`，处理完成将其 `pop` 出，`activeEffect` 始终指向了 `effectStack` 的栈顶，保证了 `activeEffect` 始终能指向当前处理的 `effect`。当然，`effectStack` 还处理了递归调用的情况 `!effectStack.includes(effect)`。

> note: 还别说，`effectStack`、`activeEffect` 和 Vuejs2.0 的 `Dep.target`、`targetStack` 有异曲同工之妙。

### track

到了这里就有点熟悉了，和 Vuejs2.0 的依赖收集差不多，在调用 `fn` 时会触发某些对象属性的 `getter` 从而触发 `track` 方法，完成数据与方法的绑定，现在我们回过头来看看 `getter handler` 中的 `track`：

```js
function track(target, type, key) {
  if (!shouldTrack || activeEffect === undefined) {
    return
  }
  let depsMap = targetMap.get(target)
  if (depsMap === void 0) {
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key)
  if (dep === void 0) {
    depsMap.set(key, (dep = new Set()))
  }
  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
  }
}
```

首先我们来了解下 `targetMap`、`depsMap`、`dep` 分别是什么：

* `targetMap` 是一个 `WeakMap`，其 key 是我们要处理成响应式对象的目标。也就是我们 `reactive(target)` 的 `target`,对应的 value 是 `depsMap`。也就是它储存了所有被响应式处理的所有目标对象。
* `depsMap` 是一个 `Map` 其 key 是 `targetMap` 中 `depsMap` 的 key(也就是 `target` 对象)的某个 key (不一定是全部)，而对应的 value 是 `Dep`。
* `Dep` 是一个 `Set` 其储存了 `target[key]` 所绑定的所有 `effect`。

这是他们的定义：

```ts
type Dep = Set<ReactiveEffect>
type KeyToDepMap = Map<any, Dep>
const targetMap = new WeakMap<any, KeyToDepMap>()
```

他们储存了所有被响应式处理的源对象和对应 `effect` 的关系：

```js
targetMap: {
    target1: {
        `target's key`: [effect1, effect2, ...],
        ...
    },
    target2: ...,
    ...
```

我们回过头来看看 `track` 它其实主要就是绑定这个关系，先从 `targetMap` 看看当前 `target` 有没有被处理过并取出depsMap，没有就为其加一个空 `Map`，再同样的看看 `depsMap` 有没有存当前处理的 key 并取出对应 `dep`，没有就为其加一个空 `Set`。
最后，只要 `dep` 中 没有 `activeEffect` 就向其添加，同时往 `activeEffect.deps` 数组中添加 `dep` (这个主要用来处理 `computed` 嵌套)

### trigger

`track` 已经为我们绑定好了关系，现在配合之前的 `setter handler` 我们来看看如何响应这层关系：

```ts
function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  extraInfo?: DebuggerEventExtraInfo
) {
  const depsMap = targetMap.get(target)
  if (depsMap === void 0) {
    // never been tracked
    return
  }
  const effects = new Set<ReactiveEffect>()
  const computedRunners = new Set<ReactiveEffect>()
  // schedule runs for SET | ADD | DELETE
  if (key !== void 0) {
    addRunners(effects, computedRunners, depsMap.get(key))
  }
  // also run for iteration key on ADD | DELETE
  if (type === TriggerOpTypes.ADD || type === TriggerOpTypes.DELETE) {
    const iterationKey = isArray(target) ? 'length' : ITERATE_KEY
    addRunners(effects, computedRunners, depsMap.get(iterationKey))
  }
  const run = (effect: ReactiveEffect) => {
    scheduleRun(effect, target, type, key, extraInfo)
  }
  // Important: computed effects must be run first so that computed getters
  // can be invalidated before any normal effects that depend on them are run.
  computedRunners.forEach(run)
  effects.forEach(run)
}

function addRunners(
  effects: Set<ReactiveEffect>,
  computedRunners: Set<ReactiveEffect>,
  effectsToAdd: Set<ReactiveEffect> | undefined
) {
  if (effectsToAdd !== void 0) {
    effectsToAdd.forEach(effect => {
      if (effect.options.computed) {
        computedRunners.add(effect)
      } else {
        effects.add(effect)
      }
    })
  }
}

function scheduleRun(
  effect: ReactiveEffect,
  target: object,
  type: TriggerOpTypes,
  key: unknown,
  extraInfo?: DebuggerEventExtraInfo
) {
  if (effect.options.scheduler !== void 0) {
    effect.options.scheduler(effect)
  } else {
    effect()
  }
}
```

因为是代理对象，`trigger` 分几种情况： SET | ADD | DELETE，对应修改对象值、增加字段、删除字段。

首先从 `targetMap` 中拿到当前 `target` 的关系结构。然后需要两个 `Set` 作为 `runner`，分别负责 `effect`(`effects`)和 `computed`(`computedRunners`)。

我们通过 `addRunners` 方法向两个 `Set` 中放入对应 key 所关联的所有 `effect` (`computed`有对应的标记 `effect.options.computed`)。

如果是 ADD | DELETE 的情况，会改变原对象的迭代器属性(数组的话是length)，那么也要把迭代器属性的 deps 放入 `runner`（如果有的话）。

最后循环调用 `effects`、`computedRunners` 中的 `effect`。注意 `computedRunners` 优先，因为有些 `effect` 是基于 `computed` 的。

> 关于 `scheduleRun`：`scheduler` 是 `effect` 方法作为 option 传入的，它允许你 `trigger` 时自己处理 `effect` 默认是直接调用。

这样，响应式数据每次修改都会触发其绑定的 `effect` 了。

### computed

如果把 `effect` 看作是 Vuejs2.0 的 `watch` 方法，那么差个 `computed` 就齐活了。这个 `computed` 就是我们熟悉的 `computed`，我们来看看源码：

```ts
function computed(getterOrOptions) {
  let getter: ComputedGetter
  let setter: ComputedSetter

  if (isFunction(getterOrOptions)) {
    getter = getterOrOptions
    setter = NOOP
  } else {
    getter = getterOrOptions.get
    setter = getterOrOptions.set
  }

  let dirty = true
  let value

  const runner = effect(getter, {
    lazy: true,
    // mark effect as computed so that it gets priority during trigger
    computed: true,
    scheduler: () => {
      dirty = true
    }
  })
  return {
    _isRef: true,
    // expose effect so computed can be stopped
    effect: runner,
    get value() {
      if (dirty) {
        value = runner()
        dirty = false
      }
      // When computed effects are accessed in a parent effect, the parent
      // should track all the dependencies the computed property has tracked.
      // This should also apply for chained computed properties.
      trackChildRun(runner)
      return value
    },
    set value(newValue: T) {
      setter(newValue)
    }
  } as any
}
```

首先 `computed` 接受方法和对象，如果是方法 `getter` 就是该方法，如果是对象则拿到 `getter` 和 `setter`。

接着 `computed` 会用 `effect` 处理 `getter` 生成 `runner`，不过它有点特殊: 1.`lazy: true` 他不会立即执行。 2. `computed: true` 其被标记为 `computed` 会优先与其他 `effect` 触发。 3. `scheduler: () => void dirty = true` 其设置了 `scheduler` 即便触发了也不会执行 `getter`，而只是将标记 `dirty` 设置为 `true`。

最后返回了一个类似 `ref` 的对象，其 `setter` 调用的是传入的对象，其 `getter` 返回的是 `runner()` 的返回值。

大体上就是这样，像是做了一层简单的代理，将 `getter` 包装成了对象，但不止如此，我们来看看细节。

`runner`：首先其用 `effect` 处理 `getter` 但其似乎不想让其成为正真的 `effect`，而只是想利用 `effect` 的依赖关系处理，即便 `runner` 被触发也没有任何处理(除了 `dirty`)

`dirty`: `computed` 对象并不想每次 `get` 都调用对应方法，所以其通过闭包存 `value`，通过 `dirty === true` 标记决定是否从原方法获取结果，两种情况 `dirty = true`，一是初始化时，二是 `runner` 这个 `effect` 被触发时（也就是知道了方法中的响应式对象的值变化了）。

`trackChildRun`：其用来处理 `computed` 对象被 `effect(fn)` 的 `fn` 中调用的情况（称`pEffect`）。我们知道 `runner` 不处理任何 `trigger`，所以当 `computed` 中的响应式数据变化时，其触发流程并不是 `reactive => computed => pEffect` 而是 `reactive` 直接触发 `pEffect`：

```ts
function trackChildRun(childRunner: ReactiveEffect) {
  if (activeEffect === undefined) {
    return
  }
  for (let i = 0; i < childRunner.deps.length; i++) {
    const dep = childRunner.deps[i]
    if (!dep.has(activeEffect)) {
      dep.add(activeEffect)
      activeEffect.deps.push(dep)
    }
  }
}
```

当 `runner` 调用完成，其 `effect` 已经从 `effectStack` 推出，如果此时在父 `pEffect` 的处理中,那么当前 `activeEffect` 就是 `pEffect`。此时调用 `trackChildRun` 将 `runner.deps` 的内容全部推向 `activeEffect.deps`，同时这些 `dep` 中也插入 `activeEffect`。也就是说 当前 `activeEffect` 也就是 `pEffect` 继承了 `runner` 的所有 `trigger`，`computed` 中的所有响应式数据变化会直接触发调用了 `computed` 的 `getter` 的 `effect`。

ok, 至此 `reactivity` 的内容我们看完了，简单的说和之前版本的差不多，同样是改造数据的 `getter` 和 `setter` 不过现在用了 `proxy`，同样的 `getter` 依赖收集，`setter` 通知，现在从 `watcher` 变成了 `effect` 核心的思想还是没变，不过现在更灵活，之前触发依赖收集只有在视图更新时、设置了 `watch`、`computed` 时，现在我们可以在任何地方调用 `effect` ，可以说更加灵活，可读性更好。

那么核心的看完了我们接下来看看还有什么变化。

## createApp

[my-vue]:https://github.com/jwdzzhz777/blog/blob/master/articles/vue.md
[vue-next]:https://github.com/vuejs/vue-next
[alpha4]:https://github.com/vuejs/vue-next/releases/tag/v3.0.0-alpha.4
[official blog]:https://www.vuemastery.com/blog/top-ways-to-learn-Vue-3/