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

和之前一样我们来梳理一遍从创建 app 到页面渲染的流程。3.0 对项目结构也做了调整，之前 `import Vue from 'vue'` 是一个构造方法，而现在是一个集合，其大部分的东西都在 `runtime-core` 包中，而创建 app 的方法也在其中：

```js
const { render: baseRender, createApp: baseCreateApp }=createRenderer({
  patchProp,
  ...nodeOps
})

// use explicit type casts here to avoid import() calls in rolled-up d.ts
export const render=baseRender as RootRenderFunction<Node, Element>

export const createApp: CreateAppFunction<Element>=(...args) => {
  const app=baseCreateApp(...args)

  const { mount }=app
  app.mount=(container): any => {
    if (isString(container)) {
      container=document.querySelector(container)!
    }
    const component=app._component
    if (
      __RUNTIME_COMPILE__&&
      !isFunction(component)&&
      !component.render&&
      !component.template
    ) {
      component.template=container.innerHTML
    }
    // clear content before mounting
    container.innerHTML=''
    return mount(container)
  }

  return app
}
```

调用 `baseCreateApp` 方法获得 app 实例，`baseCreateApp` 通过 `createRenderer` 方法获得，然后扩展 `mount` 方法,主要通过 `selector` 得到真正的 dom 元素。

`createRenderer` 方法内置了很多方法，将之前 `patch` 相关的全部流程都放在了里面，最后返回一个 `render` 方法和一个通过 createApi 生成的 `createApp` 方法：

```ts
function createRenderer<
  HostNode extends object=any,
  HostElement extends HostNode=any
>(
  options: RendererOptions<HostNode, HostElement>
): {
  render: RootRenderFunction<HostNode, HostElement>
  createApp: CreateAppFunction<HostElement>
} {
  function patch() {...}

  ... 各种方法这里不贴了

  const render: RootRenderFunction<HostNode, HostElement>=(
    vnode,
    container: HostRootElement
  ) => {
    if (vnode==null) {
      if (container._vnode) {
        unmount(container._vnode, null, null, true)
      }
    } else {
      patch(container._vnode||null, vnode, container)
    }
    flushPostFlushCbs()
    container._vnode=vnode
  }

  return {
    render,
    createApp: createAppAPI(render)
  }
}
```

`render` 方法主要是调用 `patch` 方法。在这之前，我们先来看看 `createAppAPI` 方法：

```ts
function createAppAPI<HostNode, HostElement>(
  render: RootRenderFunction<HostNode, HostElement>
): CreateAppFunction<HostElement> {
  return function createApp(rootComponent: Component, rootProps=null) {
    const context=createAppContext()

    let isMounted=false

    const app: App={
      _component: rootComponent,
      _props: rootProps,
      _container: null,
      _context: context,
      get config() {
        return context.config
      },
      set config(v) {},
      use(plugin: Plugin, ...options: any[]) {...},
      mixin(mixin: ComponentOptions) {...},
      component(name: string, component?: Component): any {...},
      directive(name: string, directive?: Directive) {...},
      mount(rootContainer: HostElement): any {
        if (!isMounted) {
          const vnode=createVNode(rootComponent, rootProps)
          // store app context on the root VNode.
          // this will be set on the root instance on initial mount.
          vnode.appContext=context

          render(vnode, rootContainer)
          isMounted=true
          app._container=rootContainer
          return vnode.component!.proxy
        }
      },
      unmount() {...},
      provide(key, value) {...}
    }

    return app
  }
}
```

其接受 `render` 函数生成一个 `createApp` 方法，终于找到你了，`createApp` 其定义了一个 app 实例初始的结构，里面有我们熟悉的方法：`use`、`mixin` 等等，就是之前的 `Vue.use`、`Vue.minxin` 等方法，最后返回这个实例，可以看出和之前的区别，Vuejs2.0 在 `new Vue` 时会各种初始化 `initLifecycle`、`initEvents`、`initRender` 等等，现在统统没有了，都放在 了 `mount` 方法中。

### mount 过程

之前提到 `Vue` 实例上暴露的 `mount` 主要用来获取 dom element (`rootContainer`)，然后调用这个内置 `mount`，该方法简单来说就干了两件事 `createVNode` 和 `render`。

这里 `createVNode` 主要就是生成一个根 `vnode`。 该 `vnode` 是空的，其中 `vnode.type` 存了 `createApp` 的第一个参数，也就是 options （当然还会处理 `createApp` 的第二个参数 `rootProps` 相当于给实例穿了 `props`，存在 `vnode.props` 中）。`vnode` 的结构大致长这样：

```ts
const vnode: VNode = {
    _isVNode: true,
    type,
    props,
    key: (props !== null && props.key) || null,
    ref: (props !== null && props.ref) || null,
    scopeId: currentScopeId,
    children: null,
    component: null,
    suspense: null,
    dirs: null,
    transition: null,
    el: null,
    anchor: null,
    target: null,
    shapeFlag,
    patchFlag,
    dynamicProps,
    dynamicChildren: null,
    appContext: null
  }
```

调用 `render` 时传入了刚刚生成的 `vnode` 和 dom element，现在我们回到 `createRenderer`, `render` 方法正常情况其直接调用了 `createRenderer` 的 `patch` 方法，那么我们来看看 `patch` 方法：

```ts
function patch(
    n1: HostVNode|null, // null means this is a mount
    n2: HostVNode,
    container: HostElement,
    anchor: HostNode|null=null,
    parentComponent: ComponentInternalInstance|null=null,
    parentSuspense: HostSuspenseBoundary|null=null,
    isSVG: boolean=false,
    optimized: boolean=false
  ) {
    // patching & not same type, unmount old tree
    if (n1!=null&&!isSameVNodeType(n1, n2)) {
      anchor=getNextHostNode(n1)
      unmount(n1, parentComponent, parentSuspense, true)
      n1=null
    }

    const { type, shapeFlag }=n2
    switch (type) {
      case Text:
        processText(...)
        break
      case Comment:
        processCommentNode(...)
        break
      case Fragment:
        processFragment(...)
        break
      case Portal:
        processPortal(...)
        break
      default:
        if (shapeFlag&ShapeFlags.ELEMENT) {
          processElement(...)
        } else if (shapeFlag&ShapeFlags.COMPONENT) {
          processComponent(
            n1,
            n2,
            container,
            anchor,
            parentComponent,
            parentSuspense,
            isSVG,
            optimized
          )
        } else if (__FEATURE_SUSPENSE__&&shapeFlag&ShapeFlags.SUSPENSE) {
          ; (type as typeof SuspenseImpl).process()
        }
    }
  }
```

首先是判断新老 `vnode` 是否相似。注意 `vnode.type` 存的是最原始的 `option` 和 `templete`，所以 `isSameVNodeType` 也主要是判断 `vnode.type` 是否变化，和 `key` 是否一致（这里和旧版本的 `isSameVNode` 稍稍有些不一样，但功能是一样的，相似即可复用），如果变化则会调用 `unmount`，清除去上一次的渲染结果。接着 `type` 有各种情况，我们就不展开了，主要看看正常情况，也就是组件的情况 `processComponent`：

```ts
function processComponent(
  n1: HostVNode|null,
  n2: HostVNode,
  container: HostElement,
  anchor: HostNode|null,
  parentComponent: ComponentInternalInstance|null,
  parentSuspense: HostSuspenseBoundary|null,
  isSVG: boolean,
  optimized: boolean
) {
  if (n1==null) {
    mountComponent(
      n2,
      container,
      anchor,
      parentComponent,
      parentSuspense,
      isSVG
    )
  } else {
    const instance=(n2.component=n1.component)!

    if (shouldUpdateComponent(n1, n2, parentComponent, optimized)) {
      if (
        __FEATURE_SUSPENSE__&&
        instance.asyncDep&&
        !instance.asyncResolved
      ) {
        updateComponentPreRender(instance, n2)
        return
      } else {
        // normal update
        instance.next=n2
        // instance.update is the reactive effect runner.
        instance.update()
      }
    } else {
      // no update needed. just copy over properties
      n2.component=n1.component
      n2.el=n1.el
    }
  }
}
```

`n1,n2` 就是 `oldVnode，vnode`，当 `oldVnode` 不存在则调用 `mountComponent` 初始化、渲染，存在则更新视图，我们现在看看初始化的过程：

```ts
function mountComponent(
    initialVNode: HostVNode,
    container: HostElement,
    anchor: HostNode|null,
    parentComponent: ComponentInternalInstance|null,
    parentSuspense: HostSuspenseBoundary|null,
    isSVG: boolean
  ) {
    const instance: ComponentInternalInstance=(initialVNode.component=createComponentInstance(
      initialVNode,
      parentComponent
    ))

    // resolve props and slots for setup context
    setupComponent(instance, parentSuspense)

    setupRenderEffect(
      instance,
      parentSuspense,
      initialVNode,
      container,
      anchor,
      isSVG
    )
  }
```

首先是生成一个 `compomentInstance` 实例并挂载在 `vnode.component` 上，这个实例存了 `vnode` 本身、一堆字段和一个 `emit` 方法，代码就不贴了各位自己搜，接着 `setupComponent` 方法用于一系列初始化，`setupRenderEffect` 则利用 `reactivity` 的 `effect` 处理渲染方法。到了这一步总算开始初始化过程了，也就是之前的各种 `intxxx` 方法，现在全都放在了 `setupComponent` 方法，那么我们看看发生了什么：

### setupComponent

直接上代码：

```ts
function setupComponent(
  instance: ComponentInternalInstance,
  parentSuspense: SuspenseBoundary | null,
  isSSR = false
) {
  isInSSRComponentSetup = isSSR
  const propsOptions=instance.type.props // type 就是 options, type.props 就是 props 的 options
  const { props, children, shapeFlag } = instance.vnode // vnode.props 就是 createApp 的第二个参数 props
  // 根据 propsOptions 和 props 设置 instance.props
  resolveProps(instance, props, propsOptions)
  resolveSlots(instance, children)

  // setup stateful logic
  let setupResult
  if (shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
    setupResult = setupStatefulComponent(instance, parentSuspense)
  }
  isInSSRComponentSetup = false
  return setupResult
}
```

首先是处理 `props`，`instance.type` 存的是原始的 `options`，所以 `instance.type.props` 就是我们设置的 props 规则（`props: {data: string}` 这种）,而实际传入的值在其父 `vnode.props` 中（创建 app 时父 `vnode` 就是在 `mount` 中创建的 `vnode` 此时会处理 `createApp` 的第二个参数 `rootProps` 并放入 `vnode`）。接着通过 `resolveProps` 来处理 `props` ,具体代码不展开了，`props` 会存在 `instance.props` 并设置代理 `instance.propsProxy`。

接着是 `resolveSlots`，也就是处理 `slot` 将 `children` 和其 `children。name` 以 map 的形式存在 `instance.slot` 中，这里也不贴代码了，当然初始化 app 没有 slot。

接下来是 `setupStatefulComponent` 方法，3.0 新增了 `setup` 方法来代替之前的 `data`、`methods` 等等功能，而该方法就是来处理 `setup`，同时处理实例的代理：

```ts
function setupStatefulComponent(
  instance: ComponentInternalInstance,
  parentSuspense: SuspenseBoundary | null
) {
  const Component = instance.type as ComponentOptions

  // 0. create render proxy property access cache
  instance.accessCache = {}
  // 1. create public instance / render proxy
  // instance 的代理，
  instance.proxy = new Proxy(instance, PublicInstanceProxyHandlers)
  // 2. create props proxy
  // the propsProxy is a reactive AND readonly proxy to the actual props.
  // it will be updated in resolveProps() on updates before render
  // propsProxy 是 instance.props 的代理，让其只读
  const propsProxy = (instance.propsProxy = isInSSRComponentSetup
    ? instance.props
    : shallowReadonly(instance.props))
  // 3. call setup()
  const { setup } = Component
  if (setup) {
    const setupContext = (instance.setupContext =
      setup.length > 1 ? createSetupContext(instance) : null)

    currentInstance = instance
    const setupResult = callWithErrorHandling(
      setup,
      instance,
      ErrorCodes.SETUP_FUNCTION,
      [propsProxy, setupContext]
    )
    currentInstance = null

    if (isPromise(setupResult)) {
      if (isInSSRComponentSetup) {
        // return the promise so server-renderer can wait on it
        return setupResult.then(resolvedResult => {
          handleSetupResult(instance, resolvedResult, parentSuspense)
        })
      } else if (__FEATURE_SUSPENSE__) {
        // async setup returned Promise.
        // bail here and wait for re-entry.
        instance.asyncDep = setupResult
      }
    } else {
      handleSetupResult(instance, setupResult, parentSuspense)
    }
  } else {
    finishComponentSetup(instance, parentSuspense)
  }
}
```

首先是设置代理，`accessCache` 对象用于存储各个 key 来自哪里(data 或 props 或 instance 本身)，`proxy` 为 `instance` 自身的代理，每次对实例进行操作 `this.xxx` (非 `$` 开头) 都会在 `accessCache` 中查找其来源，然后对其来源进行操作 (当然若没有就会一个一个找，并存在 `accessCache` 中，代码在 `PublicInstanceProxyHandlers` 中，就不贴了)，接着会为 prop 单独设置一个代理 `propsProxy` 当 `this.xxx` 的key 'xxx' 来自与 props 时实际操作的是 `propsProxy`，而 `propsProxy` 被设置为只读的，毕竟 props 是由父 vnode 决定，不可修改。

接下来是处理 `setup` 如果我们使用了 `setup` 首先会生成一个 `setupContext` 作为调用时的入参，里面有三个字段，`attrs` 和 `slots` 是 instance 对应属性的代理（之所以是代理就是防止修改）`emit` 就是 `instance.emit`，用于发射事件。接着就会以 `propsProxy, setupContext` 为入参调用 `setup` 并拿到结果(当然会处理异步的情况) 。拿到结果 `setupResult` 后调用 `handleSetupResult` 来处理结果。

```ts
function handleSetupResult(
  instance: ComponentInternalInstance,
  setupResult: unknown,
  parentSuspense: SuspenseBoundary | null
) {
  if (isFunction(setupResult)) {
    // setup returned an inline render function
    instance.render = setupResult as RenderFunction
  } else if (isObject(setupResult)) {
    // setup returned bindings.
    // assuming a render function compiled from template is present.
    instance.renderContext = setupResult
  }
  finishComponentSetup(instance, parentSuspense)
}
```

结果有两种情况，一种是对象，里面放着方法、响应式数据等等，直接绑定在 `instance.renderContext` 上。还有一种是函数，也就对应着之前 `render` 属性，也就是渲染函数如果是渲染函数后期就不用执行 compiler 过程了直接将其赋值给 `instance.render`。

上述完成后会执行 `finishComponentSetup` 进行最后的工作，我们来看看代码：

```ts
function finishComponentSetup(
  instance: ComponentInternalInstance,
  parentSuspense: SuspenseBoundary | null
) {
  const Component = instance.type as ComponentOptions
  if (!instance.render) {
    if (__RUNTIME_COMPILE__ && Component.template && !Component.render) {
      // __RUNTIME_COMPILE__ ensures `compile` is provided
      Component.render = compile!(Component.template, {
        isCustomElement: instance.appContext.config.isCustomElement || NO
      })
      // mark the function as runtime compiled
      ;(Component.render as RenderFunction).isRuntimeCompiled = true
    }

    instance.render = (Component.render || NOOP) as RenderFunction

    // for runtime-compiled render functions using `with` blocks, the render
    // proxy used needs a different `has` handler which is more performant and
    // also only allows a whitelist of globals to fallthrough.
    if (__RUNTIME_COMPILE__ && instance.render.isRuntimeCompiled) {
      instance.withProxy = new Proxy(
        instance,
        runtimeCompiledRenderProxyHandlers
      )
    }
  }

  // support for 2.x options
  if (__FEATURE_OPTIONS__) {
    currentInstance = instance
    applyOptions(instance, Component)
    currentInstance = null
  }
}
```

如果没有提供render函数，则会执行 `compile` 过程，也就是 `template => ast => render 函数` 的过程，此过程相关的代码在 `compiler-xxx` 中，就不在展开了，大体上和上一版本差不多（大概），最后生成 render 函数并挂载在 `instance` 和 `instance.type` 上。

`withProxy` 又是一层 instance 的代理，上一篇有提到，在调用 render 函数时会使用 `with (instance) {...}` 语句来保证执行时拿到 instance 中对应的值，这一次也不例外，不过不是直接用 `with (instance)` 而是 `with (instance.withProxy)`，`withProxy` 的 handler 和 `instance.proxy` 基本一样，都会走 `accessCache` 来获取正确的值。

当然如果我们不使用新的一套 (`setup`) 3.0 还是兼容老的一套的写法，`applyOptions` 中就会对旧版本的属性进行处理，其中包括 `watch`、`computed`、各种生命周期、`data`、`methods` 等等，`data` 函数的结果会 `reactivity` 化，然后还会处理一些共有的属性比如 `directive`、`components` 等。

ok 处理完了各种属性和render函数，我们要开始渲染页面了，让我们回到 `mountComponent` 方法，来看看调用的 `setupRenderEffect`:

### setupRenderEffect

直接看代码

```ts
function setupRenderEffect(
    instance: ComponentInternalInstance,
    parentSuspense: HostSuspenseBoundary|null,
    initialVNode: HostVNode,
    container: HostElement,
    anchor: HostNode|null,
    isSVG: boolean
  ) {
    // create reactive effect for rendering
    instance.update=effect(function componentEffect() {
      if (!instance.isMounted) {
        const subTree=(instance.subTree=renderComponentRoot(instance))
        // beforeMount hook
        if (instance.bm!==null) {
          invokeHooks(instance.bm)
        }
        patch(null, subTree, container, anchor, instance, parentSuspense, isSVG)
        initialVNode.el=subTree.el
        // mounted hook
        if (instance.m!==null) {
          queuePostRenderEffect(instance.m, parentSuspense)
        }
        // activated hook for keep-alive roots.
        if (
          instance.a!==null&&
          instance.vnode.shapeFlag&ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE
        ) {
          queuePostRenderEffect(instance.a, parentSuspense)
        }
        instance.isMounted=true
      } else {
        // updateComponent
        // This is triggered by mutation of component's own state (next: null)
        // OR parent calling processComponent (next: HostVNode)
        const { next }=instance

        if (next!==null) {
          updateComponentPreRender(instance, next)
        }
        const nextTree=renderComponentRoot(instance)
        const prevTree=instance.subTree
        instance.subTree=nextTree
        // beforeUpdate hook
        if (instance.bu!==null) {
          invokeHooks(instance.bu)
        }
        // reset refs
        // only needed if previous patch had refs
        if (instance.refs!==EMPTY_OBJ) {
          instance.refs={}
        }
        patch(
          prevTree,
          nextTree,
          // parent may have changed if it's in a portal
          hostParentNode(prevTree.el as HostNode) as HostElement,
          // anchor may have changed if it's in a fragment
          getNextHostNode(prevTree),
          instance,
          parentSuspense,
          isSVG
        )
        instance.vnode.el=nextTree.el
        if (next===null) {
          // self-triggered update. In case of HOC, update parent component
          // vnode el. HOC is indicated by parent instance's subTree pointing
          // to child component's vnode
          updateHOCHostEl(instance, nextTree.el)
        }
        // updated hook
        if (instance.u!==null) {
          queuePostRenderEffect(instance.u, parentSuspense)
        }
      }
    }, prodEffectOptions)
  }
```

通过 `effect` 函数处理 `componentEffect` 方法，并将返回的 `effect` 方法赋值给 `instance.update`，我们之前已经了解了 `effect` 的作用，所以我们现在来看看 `componentEffect`。

`renderComponentRoot` 实际就是调用 render 方法（`instance.render!.call(withProxy || proxy)`）并拷贝实例的 `attrs` 属性，最后返回 `vnode`。在调用 `render` 函数的过程中会从触发实例中响应式数据的 `getter` 结合之前讲的，这些数据就和 `componentEffect` 绑定起来，一旦数据变化，就会调用 `componentEffect` 达到重新渲染的目的。

`isMounted` 标记是否已经渲染，其区分是初始化过程还是更新过程。

得到 `vnode` 后执行 `patch` 流程。`patch` 流程大体上和之前类似，做了许多优化，针对不同节点不同处理，同时 diff 也做了稍许变动，大体上算法还是差不多（对撞指针），不过如果 diff 的列表里没有一个设置 key ，那么循环一遍就好了。具体代码也不贴了。

同时生命周期的回调也在这处理，不过新版有点不同，我们知道之前生命周期的函数是作为 options 声明的，直接在 `$mount` 时对应阶段调用就行，不过新版在 `setup` 中有些不同，我们是这样用的：

```ts
const { createApp, onMounted } = Vue

createApp({
  setup() {
    onMounted(() => {
      ...
    })

    return ...;
  }
})
```

在 `setup` 函数中通过 `onMounted` 函数处理，这样如何确认生命周期方法来自于哪个实例呢，首先我们先回到 `setupStatefulComponent` 方法，在处理 `setup` 函数或者 2.0 版本写法时我们都全局存了 instance

```ts
currentInstance = instance
const setupResult = callWithErrorHandling(
  setup,
  instance,
  ErrorCodes.SETUP_FUNCTION,
  [propsProxy, setupContext]
)
currentInstance = null

currentInstance = instance
applyOptions(instance, Component)
currentInstance = null
```

这样我们在处理生命周期 hooks 的时候就可以拿到当前实例了，我们来看看代码

```ts
function injectHook(
  type: LifecycleHooks,
  hook: Function & { __weh?: Function },
  target: ComponentInternalInstance | null = currentInstance,
  prepend: boolean = false
) {
  if (target) {
    const hooks = target[type] || (target[type] = [])
    // cache the error handling wrapper for injected hooks so the same hook
    // can be properly deduped by the scheduler. "__weh" stands for "with error
    // handling".
    const wrappedHook =
      hook.__weh ||
      (hook.__weh = (...args: unknown[]) => {
        if (target.isUnmounted) {
          return
        }
        // disable tracking inside all lifecycle hooks
        // since they can potentially be called inside effects.
        pauseTracking()
        // Set currentInstance during hook invocation.
        // This assumes the hook does not synchronously trigger other hooks, which
        // can only be false when the user does something really funky.
        setCurrentInstance(target)
        const res = callWithAsyncErrorHandling(hook, target, type, args)
        setCurrentInstance(null)
        resumeTracking()
        return res
      })
    if (prepend) {
      hooks.unshift(wrappedHook)
    } else {
      hooks.push(wrappedHook)
    }
  }
}
```

实际上新版生命周期方法都会调用 `injectHook`，其中 `type` 是生命周期的名字(如 `onMounted`)，`hook` 是我们的处理方法，`target` 是当前实例。`target` 在未指定的情况下默认指向我们全局存的 `currentInstance`，仔细想想，处理 `setup` 函数在渲染之前，只有在渲染的时候才会处理 children，才有可能出现其他的 componentInstance，并不会出现叠加的情况，所以一个全局变量就够了。

内容比较好理解，简单的说就是把我们的处理方法放入实例对应的属性中（数组）`target[type].push(wrappedHook)`。注意 `pauseTracking` 和 `resumeTracking` 两个方法，他们是 `reactivity` 中的依赖收集的开关，也就是说 Vue 不希望在生命周期函数中发生依赖收集，因为这些方法可能出现在某个 effect 处理中。

***

ok 至此从新建app 到渲染的过程就梳理完了，的确相比之前变化还算是蛮大的，不过核心的部分都还没有变，再加上现在还是 alpha 版，所以现在只是简单梳理下，并和 2.0 做了个对比，所以很多地方都没有提到，有兴趣可以自己去看。当然这篇并不是完了，等到正式版更新肯定还会有所不同，所以届时还会继续更新。

[my-vue]:https://github.com/jwdzzhz777/blog/blob/master/articles/vue.md
[vue-next]:https://github.com/vuejs/vue-next
[alpha4]:https://github.com/vuejs/vue-next/releases/tag/v3.0.0-alpha.4
[official blog]:https://www.vuemastery.com/blog/top-ways-to-learn-Vue-3/