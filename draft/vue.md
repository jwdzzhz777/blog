# vue

为了更深入了解 Vuejs 我决定，边读源码边自己实现一些简单的功能。

> note:文中的代码会进行缩减/修改，不然太多环境判断之类的代码不好阅读。建议边看源代码边读。

## 数据绑定

为什么从数据绑定开始，一是因为它重要，而是因为它比较独立，不依赖框架，可以单独抽出来讲。

### Observer

总所周知，Vuejs 是通过 数据劫持(`Object.defineProperty()`) 配合观察者模式来实现的数据绑定，所以首先来看看 Vuejs 实现的观察者：

```js
export class Observer {
    value: any;
    dep: Dep;

    constructor (value: any) {
        this.value = value
        this.dep = new Dep()

        Object.defineProperty(value, '__ob__', {
            value: this,
            enumerable: true,
            writable: true,
            configurable: true
        })

        if (Array.isArray(value)) {
            // 这里重写了数组的方法
            { resetArrayMethods }

            this.observeArray(value)
        } else {
            const keys = Object.keys(value)
            for (let i = 0; i < keys.length; i++) {
                // 变成响应式属性
                defineReactive(obj, keys[i])
            }
        }
    }
    observeArray (items: Array<any>) {
        for (let i = 0, l = items.length; i < l; i++) {
            new Observer(items[i])
        }
    }
}
```

`Observer` 的功能比较简单，就是将一个对象包装成可观察对象（Observable），将自己（this）绑定在对象的 `__ob__` 字段上，劫持每一个字段。如果是数组，则会进行处理，重写数组方法，并将数组内每个对象继续包装成可观察对象。

> note: 这里以及下面的 `new Observer()` 源码中对应的是一个 `function observe` 大多是各种判断的我就直接忽略了，知道就好

这里简单插一下 Vuejs 是如何重写数组的：

```js
const methodsToPatch = [ 'push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse' ]
export const arrayMethods = Object.create(Array.prototype)

methodsToPatch.forEach(method => {
    // 原方法
    const original = arrayProto[method]
    Object.defineProperty(arrayMethods, method, function(...args) {
        // super()
        const result = original.apply(this, args)
        // 中间代码折起来了，也比较好理解，毕竟有些操作(push)会扩充数组，这些扩充的对象也要被包装成 可观察者对象

        // 通知改变
        this.__ob__.dep.notify()
    })
})
```

其实也比较简单，类似继承的方式搞一个新的 `prototype` 重写数组方法，依旧调用原方法的同时增加通知 `ob.dep.notify()`，最后重改原型链 `target.__proto__ = arrayMethods`

接下来我们来看看 `defineReactive` 方法中 Vuejs 如何处理的数据劫持：

```js
function defineReactive (
    obj: Object,
    key: string,
    val: any,
    shallow?: boolean
) {
    const dep = new Dep()
    // 前提：属性可枚举

    // 这里有属性的 getter/setter 已经被定义过的情况，同样的 Vuejs 没有忽略它
    // 满足自己功能的同时，也调用了原方法 getter/setter ? getter/setter.call(obj)
    const getter = property && property.get
    const setter = property && property.set
    if ((!getter || setter) && arguments.length === 2) {
        // 默认值咯
        val = obj[key]
    }

    // 万一也是个对象/数组，深度劫持
    let childOb = !shallow && new Observer(val)
    Object.defineProperty(obj, key, {
        get: function reactiveGetter () {
            const value = getter ? getter.call(obj) : val
            if (Dep.target) {
                dep.depend()
                // 如果是对象
                if (childOb) {
                    childOb.dep.depend()
                    // 如果是数组
                    if (Array.isArray(value)) {
                        dependArray(value)
                    }
                }
            }
            return value
        },
        set: function reactiveSetter (newVal) {
            const value = getter ? getter.call(obj) : val
            // 前后对比
            if (newVal === value || (newVal !== newVal && value !== value)) return

            if (setter) setter.call(obj, newVal)
            else val = newVal
            // 万一设置了个新对象/数组呢
            childOb = !shallow && observe(newVal)
            // 通知
            dep.notify()
        }
    })
}

```

比较好理解，改写了属性的 getter/setter 方法（同时兼容了本身就有 getter/setter 方法的情况）在 getter/setter 时进行操作。简单的说 `Observer` 就是改造目标对象，劫持属性。

这里要注意的是在方法 `defineReactive` 中 `new Dep` 形成了闭包，在 `getter` 时才调用 `dep.depend()`，`setter` 时才通知 `dep.notify()`，这个之后讲

接下来需要一个充当主体的角色，将值多路推送给观察者(Observer)，Vuejs 中就是 `Dep` 了：

### Dep

Dep 就是个内容发布者，它依托在一个可观察对象内（Observable）可以被多个观察者订阅（Watcher），多路推送消息。

```js
class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

// The current target watcher being evaluated.
// This is globally unique because only one watcher
// can be evaluated at a time.
Dep.target = null
const targetStack = []

function pushTarget (target?: Watcher) {
  targetStack.push(target)
  Dep.target = target
}

function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

结构也比较简单，`subs` 是用来存放所有的观察者（Watcher），`addSub` 和 `removeSub` 管理 `subs`，`notify` 用于通知所有的观察者（调用所有 `subs` 的 `update` 方法）。那么 `target` 相关的是什么。

首先我们知道的 `target` 装的就是观察者（Watcher）， `static target` 作为全局就有一点单例的味道，啥时候会用到它呢，我们简单梳理一下初始化的过程：

1. 各种初始化
2. observe data（重写 getter/setter）
3. 调用 `$mount` 方法，这个时候会 `new Watcher()`

这个 `Watcher` 接收一个 `render` 函数并在初始化调用（这个 `Watcher` 的作用是要渲染组件的 ，具体的下面讲），调用 `render` 函数时会获取对应的值，这个时候触发了我们重写的 `getter` ，触发的过程类似这样的：

```js
pushTarget(watcher)
getter()
popTarget()
```

那么结合上面的 `Observer`，每个子属性（`defineReactive`处理过）都有一个 `dep`，这个子属性本身可能也是个对象被处理成可观察对象（Observable）, 它也有个 `dep`，那么子属性的子属性..（套娃）这些 `dep` 都被同一个 `Watcher` 订阅了（上述的那个）。

如果这个时候遇到嵌套的情况，比如 `Router` 或 `Vue.extend` 这时候上面的流程还会再来一次，这时候会生成一个新的 观察者（Watcher2）来观察新的数据：

```js
pushTarget(watcher)
(getter() {
    pushTarget(watcher2)
    getter2()
    popTarget()
})()
popTarget()
```

类似于这样，这个时候 `targetStack` 就派上用场了，它保留了之前的 `Watcher`，因为是单例的，新的 `dep` 会被 `Watcher2` 给订阅，完事移除 `Watcher2` 继续完成之前的 `watcher` 的订阅，有一点洋葱圈模型的味道。

### Watcher

`Watcher` 作为观察者，其主要作用就是接收改变的通知并作出反应（渲染 or 通知给我们）。它的主要构成是这样的：

```js
class Watcher {
  vm: Component;
  cb: Function;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  getter: Function;
  value: any;

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    // options
    this.cb = cb
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    let value
    const vm = this.vm

    value = this.getter.call(vm, vm)
    traverse(value)

    popTarget()
    this.cleanupDeps()

    return value
  }

  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    const value = this.get()
    const oldValue = this.value
    this.value = value
    this.cb.call(this.vm, value, oldValue)
  }

  /**
   * Add a dependency to this directive.
   */
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }
}
```

我保留了一些我认为重要的部分，我们来看看逐个了解一下：

`get` ：首先构造函数其实干的事很少，做了一些简单的赋值后直接调用了 `get` 方法，`get` 上面有提到（Dep getter 触发），就是设置 `Dep.target` 调用 `expOrFn`，主要目的就是订阅，也就是我们所说的依赖收集。

`expOrFn`：它作为参数传入，其实就是一个 `getter` 触发器，从名字可以看出来它可能是一个表达式，也可能是一个方法。这里就要先提一下哪些情况会新建一个 `Watcher`：

* 初始化(`$mount`)：上面Dep 初始化有提到，这个时候 `expOrFn` 是一个 渲染函数 `() => vm._update(vm._render(), hydrating)` 解析模版渲染页面的同时触发data的 `getter`（具体后面讲），这个时候生成的 `Watcher` 意义重大，控制着 dom 的更新。

* `watch`、 `$watch`：这个时候 `expOrFn` 就是 key ，他会被 `parsePath` 处理成方法调用（`parsePath` 可以处理 `a.b.c` 这种形势的 key）， 还会传入一个 `cb` 方法，在接收更新之后调用。

```js
Vue.extend({
  watch: {
    key: (val) => cbFunction
  }
})
// or
vm.$watch('key', cbFunction)
```

* `computed`：稍稍有点不一样，`expOrFn` 就是我们定义的方法，方法内所有被触发 `getter` 的属性都会被当前的 `Watcher` 订阅，`Watcher` 接收到更新消息后还是会调用 `expOrFn` 然后将返回值设为 `value` (key 会被挂在 vm 上，取的就是这个 `value`)

```js
Vue.extend({
  computed: {
    key: cbFunction
  }
})
```

`traverse`: 深度遍历对象，把所有 `getter` 都触发一遍（我不干啥，我就看看）。

`addDep`：`Dep.depend` 就是调用此方法，用于把当前 `Watcher` 存入 `dep` 实例（来来回回有点绕）

`cleanupDeps`：以及 `deps`、`newDeps`、`depIds`、`newDepIds` 这些个玩意，你只要理解他们是防止重复订阅的就可以了。

`update`、`run`：通知更新（`dep.notify`）后调用的就是 `update` 方法,如果是同步的，调用 `run` 方法，`run` 干的事也很简单就是再调一次 `get` 要么重新渲染，要么 `cb` 给我们（此时会重新处理一次订阅）然后更新 `watch` 的 `value`。如果是异步的 调用 `queueWatcher`。

> note: 关于 `queueWatcher` 的东西在 `scheduler.js` 中，也就是所谓的调度器，简单理解就是放到任务队列延迟执行,执行的任务还是 `update` 方法。

#### scheduler

调度器不是很重要，简单提一下（需要了解 `nextTick` 、`event loop`），可跳过。

```js
const queue = []
let waiting = false
let flushing = false

function flushSchedulerQueue() {
  flushing = true;
  // 冲刷 queue
  flushing = waiting = false;
}

function queueWatcher (watcher: Watcher) {
  if (!flushing) {
    queue.push(watcher)
  } else {
    // if already flushing, splice the watcher based on its id
    // if already past its id, it will be run next immediately.
    let i = queue.length - 1
    while (i > index && queue[i].id > watcher.id) {
      i--
    }
    queue.splice(i + 1, 0, watcher)
  }
  // queue the flush
  if (!waiting) {
    waiting = true
    nextTick(flushSchedulerQueue)
  }
}
```

首先我们知道 `nextTick` 会把 `flushSchedulerQueue` 放进微任务队列(`microtask queue`)。那么 `waiting` 状态就比较好理解，他保证了 `microtask queue` 中只有一个 `flushSchedulerQueue` 任务。而 `flushing` 为 true 的情况发生在 `flushSchedulerQueue` 正在执行时，根据 `watcher.id` 往 `queue` 中插任务如果没有 ‘过号’ 那么新插进来的也会被执行掉，如果 ‘过号’ 了那么保留在 `queue` 等待下一次冲刷。
整个过程模型和微任务的流程模型简直一毛一样。

***

ok, 至此数据绑定相关的东西（也就是 `core/observer` 下的东西）我们都过了一遍了。现在我们把他们串联起来，来看看 `Vue` 的 ‘心路历程’

```js
// 应用初始化
_init() {
  // 初始化数据相关，当然前后还有其他的许多初始化，Lifecycle、Events 等等
  initState() {
    // 处理 data
    initData() {
      // 这里会将 data 可观察化 也就是设置 getter/setter
      observe() {
        new Observer();
      }
      proxy() // 代理到 vm 实例上
    }
    // data 处理好就可以被监听器订阅了
    initComputed() { new Watcher() }
    initWatch() { new Watcher() }
  }
  // 解析模版、渲染
  $mount() { 
    compileToFunctions() {
      // 这里会将模版处理成渲染函数，类似这样：
      vm.$option.render = with(this) {
        return _c('div', ...) // _c 就是我们用的 createElement 方法
      }
    }
    mountComponent() {
      updateComponent = () => vm._update(vm._render(), hydrating);
      new Watcher(vm, updateComponent); // 调用 updateComponent， 渲染页面， 触发 getter ，依赖收集
    }
  }
}
```

大概就这样，里面很多细枝末节就不提了，这些方法名都是 `Vue` 里面存在的，一搜就能找到，这里提一下 `proxy`：
此 `proxy` 不是我们想的 `es6 proxy` 他的作用是将 `$data` 的属性代理到 `vm` 实例上，方便我们 `this.xxx` 调用。

## Virtual DOM

ok 我们了解完了 `Vuejs` 数据绑定的过程，接下来我们看看编译渲染 DOM 的过程。之前提到，在实例化方法 `_init` 的最后会调用 `$mount` 来解析模版、渲染，现在我们来仔细看看。

### Compiler

首先是编译，`$mount` 中调用 `compileToFunctions` 将 `<template>` 文本转化成一个 `render` 函数，我们来看看 `compileToFunctions` 都做了什么：

```js
function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  const ast = parse(template.trim(), options)
  // ast 优化
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
}

function createCompiler (baseOptions: CompilerOptions) {
  function compile (
    template: string,
    options?: CompilerOptions
  ): CompiledResult {
    const finalOptions = Object.create(baseOptions)
    // 省略了一些 option 的处理

    const compiled = baseCompile(template.trim(), finalOptions)

    return compiled
  }

  return {
    compile,
    compileToFunctions: createCompileToFunctionFn(compile)
  }
}
```

提取了其中代码，把生成方法和 option 的处理部分删掉了，完整的可以到 `src/compile` 中去看，`compileToFunctions` 方法由 `createCompiler` 方法生成，不过主要的内容在 `baseCompile` 方法中，我们主要来看看这个方法。

里面就两步，`parse`，`generate`，我们依次来看。

#### parse

`parse`方法的代码都在 `compile/parse` 中，其主要功能就是将 `template` 编译成 AST。我们先了解以下语法树的基本结构：

```js
function createASTElement (
  tag: string,
  attrs: Array<ASTAttr>,
  parent: ASTElement | void
): ASTElement {
  return {
    type: 1,
    tag,
    attrsList: attrs,
    attrsMap: makeAttrsMap(attrs),
    rawAttrsMap: {},
    parent,
    children: []
  }
}
```

因为是将 `html` 代码转换成 AST 所以结构简单多了，主要是标签名 `tag` (`<tag>`) 和 属性 `attrsMap`（key: value）以及层级关系 `children`,`parent`。

接下来了解下 `html-parse` ，他是转换 AST 的核心代码，其本身来自于一个开源代码的扩展：[simplehtmlparser][simplehtmlparser]。代码就不贴了，自行了解下。

> note: 注意这不是 `Vuejs` 的源码，而是在这之上做的改造，我们借助它来简单了解

其实就是正则判断，判断 `template` 文本的开头是什么，对应有这么几种情况：

* `<!--`开头，也就是注释，这种情况再找到结尾 `-->` 直接删了。
* `<!DOCTYPE` 开头，文档类型，和上面一样，删了。
* `<name` 开头，开始标签(`start tag`)，执行 `parseStartTag`
* `</name` 开头，结束标签(`end tag`)，执行 `parseEndTag`

那么我们回归 `Vuejs` 本身，其实主要逻辑差不多，先简单看看流程模型：

```js
function parse(tamplate) {
  let currentParent, stack;
  parseHTML(tamplate, start, end) {
    if (startTag) {
      let match = parseStartTag(template)
      start(match)
    }
    if (endTag) {
      parseEndTag(template);
      end()
    }

    function parseStartTag() {}
    function parseEndTag() {}
  }

  function start() {}
  function end() {}
}
```

代码精简了很多，主要的解析流程就是如此，首先是 `parseHTML` 里面是如何解析的，我们结合上面了解的一起看：

```js
const attribute = /^\s*([^\s"'<>\/=]+)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
const dynamicArgAttribute = /^\s*((?:v-[\w-]+:|@|:|#)\[[^=]+\][^\s"'<>\/=]*)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
const ncname = `[a-zA-Z_][\\-\\.0-9_a-zA-Z${unicodeRegExp.source}]*`
const qnameCapture = `((?:${ncname}\\:)?${ncname})`
const startTagOpen = new RegExp(`^<${qnameCapture}`)
const startTagClose = /^\s*(\/?)>/

function parseStartTag(html) {
  const start = html.match(startTagOpen)
  if (start) {
    const match = {
      tagName: start[1],
      attrs: [],
      start: index
    }
    advance(start[0].length)
    let end, attr
    while (!(end = html.match(startTagClose)) && (attr = html.match(dynamicArgAttribute) || html.match(attribute))) {
      attr.start = index
      advance(attr[0].length)
      attr.end = index
      match.attrs.push(attr)
    }
    if (end) {
      match.unarySlash = end[1]
      advance(end[0].length)
      match.end = index
      return match
    }
  }
}
```

首先是 `parseStartTag` 方法，其通过正则查找类似 `<tagName` 的文本，记录其位置并通过子表达式得到 `tagName`，`advance` 方法用于将匹配到的值剔除，`while` 循环中不断通过正则查找并记录 `attribute` 然后删除，当匹配到 `startTagClose` 时结束(`>` 或 `/>`)，
记录子表达式（`/` 或 ` `），最后得到一个 `match` 对象，大概长这样：

```js
match = {
  tagName: 'div',
  attrs: [
    ['v-model="xxx"', 'v-model', '=', 'xxx']
  ],
  start: Number,
  end: Number,
  unarySlash: '/' || ''
}
```

得到 `match` 对象后调用 `startHandler` 进一步处理，现在的 `attrs` 还是字符串（不过已经把属性名和值都取到了，这个正则还是给力的），先处理成 `Map` ，然后才是创建 AST，然后经过一些处理（`v-for`、`v-if` 之类的属性），最后放入 `stack` 中。

```js
function start(match) {
  let tag = match.tagName;
  let attrs = ...; // 处理成 map
  let element: ASTElement = createASTElement(tag, attrs, currentParent)
  processPre(element)
  processRawAttrs(element)
  processFor(element)
  processIf(element)
  processOnce(element)

  if (!unary) {
    currentParent = element
    stack.push(element);
  } else {
    currentParent.children.push(element)
    element.parent = currentParent
  }
}
```

这里说明以下，`stack` 和 `currentParent` 在父级作用域，也就是 `parse` 方法中，用来处理父子关系。而 `unary` 是用来判断 `element` 是不是非特殊标签 (`<br/>`这种)，并且是不是 `/>` 结尾。如果是就传入 `stack` 继续处理，直到遇到 `endTag`：

```js
function parseEndTag (html) {
  let endTagMatch = html.match(endTag)
  advance(endTagMatch[0].length)
  ... // 主要还是判断那些个什么<br />啦<p /> 啦
  end(endTagMatch[1])
}

function end(tagName) {
  const element = stack[stack.length - 1]
  // pop stack
  stack.length -= 1
  currentParent = stack[stack.length - 1]
  currentParent.children.push(element)
  element.parent = currentParent
}
```

`end` 的主要功能就是取出 `stack` 栈顶，和其前面一项确认父子关系，把 `currentParent` 设置为取出后的栈顶。

简单的说：

* 解析到 `startTag` 就创建 `ASTElement`。
   如果是 `/>` 就和 stact 最后一项（`currentParent`）确认父子关系
   不是就将 `element` 推入 `stack`
* 解析到 `endTag` 就取出栈尾（`element`），`currentParent` 往前移(新的栈尾)，
  `element` 和 `currentParent` 确认父子关系

如此不断的循环。

这些就是 `parse` 的主要功能了，当然还有其他的，比如 `text-parse` 主要负责文本的处理(主要处理双括号的语法`{{expression}}`，转换成 `_s(expression)`)，而`filter-parse` 负责管道的处理 (`{{value | filter}}` => `_s(_f("filter")(value))`)，注意这里都是处理成字符串哦，至于 `_s`、`_f` 是什么，我们接下来讲。最后看一下例子，之后我们也会按照这个例子走下去：

```html
<div id="editor">
  <input v-model="input"></input>
  {{input | nothing}}
</div>

<script>
new Vue({
  el: '#editor',
  data: {
    input: 'input'
  },
  filter: {
    nothing(a) {
      return a;
    }
  }
})
</script>
```

`parse` 方法得到：

```js
ast = {
  type: 1,
  tag: 'div',
  attrsList: [{name: 'id', value: 'editor'}]
  attrsMap: {id: 'editor'},
  children: [
    {
      type: 1,
      tag: 'input',
      attrsList: [{name: 'v-model', value: 'input'}]
      attrsMap: {'v-model': 'input'},
      parent: ast,
      events: {
        input: {value: '"if($event.target.composing)return;input=$event.target.value"'}
      }
      children: []
    },
    {
      type: 2,
      expression: `"\n      "+_s(_f("nothing")(input))+"\n    "`,
      tokens: [{'@binding': "_f("nothing")(input)"}]
    }
  ]
}
```

> note: 提一嘴，`type` 1 是元素， 3 是注释，其他当 文本处理

可以看到，除了解析 `标签` , `属性` , `父子关系` 之外，遇到 `vue` 特性（`@click` 啥的）会有特殊的处理：`{{expression}}` 会被解析成 `_s(expression)`，`filter` 是 `_f(expression)`，`@name="expression"` 则会在 `ASTElement.event` 上多加一个字段 `name: expression`，`v-if` 则会在 `ASTElement` 上多加标记(`ifProcessed: expression`) 等等，就不一一列了。

#### codegen

`parse` 方法将模版解析成 `ast`，那么接下来 `generate` 方法将会把 `ast` 转化成一个 `render` 函数，我们来简单看看：

```js
function generate(ast) {
  const code = ast ? genElement(ast) : '_c("div")'
  return {
    render: `with(this){return ${code}}`
  }
}

function genElement(ast) {
  if (el.staticRoot) genStatic(el)
  else if (el.once) genOnce(el)
  else if (el.for) genFor(el)
  else if (el.if) genIf(el)
  else if (el.tag === 'template') genChildren(el)
  else if (el.tag === 'slot') genSlot(el)
  else {
    if (el.component) {
      code = genComponent(el.component, el, state)
    } else {
      let data = genData(el, state)
      let children = genChildren(el, state, true)
      let code = `_c('${el.tag}'${
        data ? `,${data}` : '' // data
      }${
        children ? `,${children}` : '' // children
      })`
    }
  }
  return code;
}

function genChildren(el) {
  return `[${el.children.map(c => genElement(c)).join(',')}]${
    normalizationType ? `,${normalizationType}` : ''
  }`
}

function genComponent (componentName, comment: ASTText): string {
  const children = el.inlineTemplate ? null : genChildren(el)
  return `_c(${componentName},${genData(el, state)}${
    children ? `,${children}` : ''
  })`
}

function genData (el: ASTElement, state: CodegenState): string {
  let data = '{'
  if (dirs) data += dirs + ','
  if (el.key) data += `key:${el.key},`
  if (el.ref) data += `ref:${el.ref},`
  if (el.refInFor) data += `refInFor:true,`
  if (el.pre) data += `pre:true,`
  if (el.component) data += `tag:"${el.tag}",`
  if (el.attrs) data += `attrs:${genProps(el.attrs)},`
  if (el.props) data += `domProps:${genProps(el.props)},`
  if (el.events) data += `${genHandlers(el.events, false)},`
  if (el.nativeEvents) data += `${genHandlers(el.nativeEvents, true)},`
  if (el.slotTarget && !el.slotScope) data += `slot:${el.slotTarget},`

  data = data.replace(/,$/, '') + '}'

  return data
}
```

截取了其中一小部分，主要就是通过 `genElement` 生成类似 `_c(tagName, Data, [children])` 这样的字符串，之后会被当作表达式调用，`_c` 方法其实就是 `render` 函数中提供的 `createElement` 方法。

接触过 `render` 函数的话应该很好理解，也就是说不管怎么样最终都会回到 `render` 函数，如果你直接用 `render` 函数还能提高性能（省去了 `parse` 的过程）。那么我们看看[官方文档][createElement]。

`createElement` 有三个参数，第一个参数是 `html` 的标签名字，直接通过 `element.tag` 得到(`parse` 解析取得)。第二个参数是一个数据对象，其值通过 `genData` 方法处理（普通属性、dom属性、style、class、事件等等），第三个参数是子虚拟节点的集合，其也由 `createElement` 构建而成，在 `genElement` 中循环处理 `element.children`。

最终 `generate` 会将 `ast` 包装成类似 `with(this){return _c(...)}` 的字符串，还是上面的例子，最终会输出这样：

```js
const render = `with(this){return _c('div',{attrs:{"id":"editor"}},[_c('input',{directives:[{name:"model",rawName:"v-model",value:(input),expression:"input"}],domProps:{"value":(input)},on:{"input":function($event){if($event.target.composing)return;input=$event.target.value}}}),_v("\n      "+_s(_f("nothing")(input))+"\n    ")])}`
```

最后我们通过 `Function` 将其转换成一个方法，并绑定在 `vm.options` 上：

```js
vm.options.render = new Function(code);

// 还是之前的例子，render 长这样子：
vm.options.render = function() {
  with (this) {
    return _c(
      'div',{attrs:{"id":"editor"}},
      [
        _c(
          'input',
          {
            directives:[
              {
                name:"model",
                rawName:"v-model",
                value:(input),
                expression:"input"
              }
            ],
            domProps:{"value":(input)},
            on:{"input":function($event){if($event.target.composing)return;input=$event.target.value}}
          }
        ),
        _v("\n      "+_s(_f("nothing")(input))+"\n    ")
      ]
    )
  }
}
```

注意了，之所以这么做，以及使用 `with` 语法，主要原因是 `Compiler` 并未对表达式进行解析处理，而是完整保留了表达式，通过 `with` 语句保证真正执行 `render` 函数时能够正确执行表达式，并从 `vm` 实例中取到正确的值。

ok,至此 `Compiler` 的任务完成。

### VNode

之前提到在生成 `render` 函数之后 `new Watcher` 时会传入一个渲染函数：

```js
const updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
```

抛去各种判断，`_render` 方法其实就是调用了 `render.call(vm)` 得到 `vnode`，而 `render` 中调用各种 `_x()`
 方法处理各种情况，这些代码都在 `core/instance/render-helpers` 中，有兴趣可以看下，其中最主要的 `_c`  也就是 `createElement` ，代码位于 `core/vdom/create-element` 中。其主要功能也很简单，说白了就是通过传入的 `tagName`、 `Data` 和 `[children]` 生成 `vnode`:

 ```js
function createElement(
  context: Component, // vm 实例
  tag: any,
  data: any,
  children: any
) {
  // 内置的 html 标签 或 svg 标签
  if (isHTMLTag(tag) || isSVG(tag)) {
    vnode = new VNode(tag, data, children, undefined, undefined, context)
  } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) { // 处理组件
    vnode = createComponent(Ctor, data, context, children, tag)
  } else {
    // 其他不知道的莫名其妙的标签名
    vnode = new VNode(tag, data, children, undefined, undefined, context)
  }
}
```

这里说一下，如果遇到不认识的 `tag` 会判断 `$options.components` 中有没有相应 `name` 的对象，返回相应的对象 `Ctor`，如果有，会执行 `createComponent` ：

```js
function createComponent(
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
) {
  if (isObject(Ctor)) {
    // 这个就是当前环境的 Vue class 
    // Vue.options._base = Vue
    const baseCtor = context.$options._base

    Ctor = baseCtor.extend(Ctor)

    const name = Ctor.options.name || tag
    const vnode = new VNode(
      `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
      data, undefined, undefined, undefined, context,
      { Ctor, data, data.on, tag, children }
    )
  }
}
```

其实就是执行 `Vue.extend(Ctor)` 生成一个带有 `componentOptions` 的 `vnode` 说到底最终都是生成 `vnode` 我们来了解以下它的结构： 

```js

```

[simplehtmlparser]:http://erik.eae.net/simplehtmlparser/simplehtmlparser.js
[createElement]:https://cn.vuejs.org/v2/guide/render-function.html#createElement-%E5%8F%82%E6%95%B0