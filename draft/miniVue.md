# 自己实现一个 MVVM

之前看了 `Vue` 2 和 3 的源码，可以说是受益匪浅，那么借着这股热头，现在我们参照 vue 来写一个简单的框架来实现 MVVM模型。

简单梳理下要实现的几个部分：

* `complier`: `template` 的编译，方便起见就不搞一套 virtualDom 了，ast 也不搞了，直接 element 搞起。
* `reactivity`: 响应式数据，参照 Vue3.0 的 `reactivity` 搞一个简单版的。
* `core`：框架本体

大概梳理完，我们开工。

## reactivity

首先我们要仿造 `reactivity` 实现一个简单的响应式数据，简单的说就是实现一下 `reactive` 和 `effect`。

### 实现 reactive 和 ref

首先我们先实现一个简单的 `proxy`：

```ts
const handler = {
    get(target: object, key: string | symbol) {
        return Reflect.get(target, key);
    },
    set(target: object, key: string | symbol, value: unknown) {
        Reflect.set(target, key, value);
        return true;
    }
};

export const reactive = function(target: object) {
    if (typeof target !== 'object') return target;
    let observed = new Proxy(target, handler);
    return observed;
}
```

ok,我们仅仅用 `proxy` 做了一层代理，其他啥也没干，不过现在有一些问题，首先多次对同一个对象调用 `reactive` 会产生多个代理对象，我们不希望这样，所以我们需要对结果做一个存储。再者，代理只对浅层的属性有效，如果某个 key 的 value 是个对象我们也需要对其做代理。我们来做一下改造：

```ts
function isObject(target: unknown) {
    return target !== null && typeof target === 'object';
}

const primitiveToReactive = new WeakMap();

const handler: ProxyHandler<object> = {
    get(target: object, key: string | symbol) {
        let result = Reflect.get(target, key);
        return isObject(result) ? reactive(result) : result;
    },
    set(target: object, key: string | symbol, value: unknown) {
        Reflect.set(target, key, value);
        return true;
    }
};

export const reactive = function(target: object) {
    if (!isObject(target)) return target;
    if (primitiveToReactive.has(target)) return primitiveToReactive.get(target);

    let observed = new Proxy(target, handler);

    primitiveToReactive.set(target, observed);
    return observed;
}
```

我们用一个 `WeakMap` 来储存我们处理过的对象和代理对象，防止对一个对象多次代理。注意我们在 getter 中对值进行是否对象的判断，这样的好处是我们就不用遍历对象了，当其真正被用到时再对其进行代理。我们验证下：

```js
let a = {
    deep: {
        some: 1
    }
};
let b = reactive(a);
let c = reactive(a);
console.log(b === c); // true
console.log(b.deep); // Proxy {some: 1}
```

好了，接着我们实现一下 `ref`，他是用来包装基本类型值的:

```ts
const ref = function(value: any) {
    if (isObject(value)) {
        if ('isRef' in value) return value;
        else return;
    }

    return Object.create(Object.prototype, {
        isRef: { value: true },
        value: {
            get() {
                return value;
            },
            set(newValue) {
                value = newValue;
            }
        }
    })
}

let a = ref('hello');
a.value // hello
a.value = 123
a.value // 123
```

这次相当于对一个值做了代理，返回一个代理对象，可以对该对象的 value 进行 get 和 set 操作。

好了现在我们实现了对数据的代理，代理的目的就是为了能够监听数据变化，现在我们在 getter 和 setter 中加上 `track` 和 `trigger` 方法，`track` 用来依赖收集，`trigger` 用来触发相应的动作：

```ts
const handler: ProxyHandler<object> = {
    get(target: object, key: string | symbol) {
        track(target, key);
        let result = Reflect.get(target, key);
        return isObject(result) ? reactive(result) : result;
    },
    set(target: object, key: string | symbol, value: unknown) {
        Reflect.set(target, key, value);
        trigger(target, key);
        return true;
    }
};

const ref = function(value: any) {
    if (isObject(value)) {
        if ('isRef' in value) return value;
        else return;
    }

    const result = Object.create(Object.prototype, {
        isRef: { value: true },
        value: {
            get() {
                track(result, 'value');
                return value;
            },
            set(newValue) {
                value = newValue;
                trigger(result, 'value');
            }
        }
    });
    return result;
}

// reactive ...
```

好了先占位，我们稍后再来实现这两个方法，在这之前我们先实现 `effect`。

### effect

`effect` 用于包装一个方法，调用并做一些处理

```ts
let activeEffect;

const effect = function(fn) {
    const effect = function(...args) {
        try {
            activeEffect = effect;
            return fn(...args);
          } finally {
            activeEffect = undefined;
          }
    }
    effect();
    return effect;
}
```

简单包装之后我们会调用原方法，在调用过程中我们会暂存当前的包装方法 `activeEffect = effect`，调用过程中会触发我们的依赖收集(`getter`)将 `activeEffect` 与我们的响应式对象及其 key 绑定。现在我们来实现 `track` 方法：

```ts
type Dep = Set<ReactiveEffect>
type KeyMap = Map<any, Dep>
const targetMap = new WeakMap<any, KeyMap>();

const track = function(target: object, key: string | symbol) {
    if (activeEffect === undefined) return;

    let keyMap = targetMap.get(target);
    if (!keyMap) targetMap.set(target, (keyMap = new Map()));

    let depsOfKey = keyMap.get(key);
    if (!depsOfKey) keyMap.set(key, (depsOfKey = new Set()));

    if (!depsOfKey.has(activeEffect)) depsOfKey.add(activeEffect);
}
```

首先我们需要 `targetMap` 来储存我们所有的依赖关系。`targetMap` 是一个 `WeakMap` 其 key 是我们的目标对象 `target`，其对应值是一个 Map,该 Map 的 key 是 `target` 中的某个 key，其对应的 value 是和该 key 所有绑定的 `effect` 的集合，是一个 Set。

那么只要当前 `activeEffect` 存在，我们就把它存进 `targetMap` 的对应位置中，这样就完成了一次依赖收集。

完成了依赖收集，我们接着来实现触发：

```ts
const trigger = function(target: object, key: string | symbol) {
    let keyMap = targetMap.get(target);
    if (!keyMap) return;
    let deps = keyMap.get(key);
    if (!deps) return;
    deps.forEach((effect) => {
        effect();
    });
}
```

很简单，拿 `target.key` 对应的 `effect` 数组，把里面的方法全调用一遍即可。如此，简易版的响应式数据就完成了我们来试一下。

```js
let data = reactive({
    title: 'hello'
});
let some = ref('world');
effect(function() {
    console.log(`${data.title} ${some.value}`);
}); // hello world

data.title = 'Hello'; // Hello world
some.value = 'World'; // Hello World
```

这样一个简单的 响应式数据的库就完成了，我们接着继续。