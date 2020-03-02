# 自己实现一个 MVVM

之前看了 `Vue` 2 和 3 的源码，可以说是受益匪浅，那么借着这股热头，现在我们参照 vue 来写一个简单的框架来实现 MVVM 模型，注意我们参照 3.0 的写法，。

简单梳理下要实现的几个部分：

* `complier`: `template` 的编译，方便起见就不搞一套 virtualDom 了，ast 也不搞了，直接 element 搞起。
* `reactivity`: 响应式数据，参照 Vue3.0 的 `reactivity` 搞一个简单版的。
* `vnode`: 虚拟dom，用于 diff。
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

## compiler

现在我们来做一个模版的编译器，用来处理模版，将其处理成 `render` 函数。`render` 函数是 javascript 版的模版方案，转成函数把表达式已经模版语法处理好，方便多次调用。

最优的处理应该是对模版字符串处理，也就是挂载元素的 `innerHTML`，我们简单点直接对 element 处理。

```ts
const compile = function(element: Element) {
    let code = `with(this) {return ${process(element)}}`;
    return new Function(code);
};
```

我们通过 `new Function` 来生成函数，所以我们其实是要将模版转化成脚本字符串，这样可以保留表达式，同时我们借助 `with() {}` 语法，保证表达式能从环境中取到对应的值。

注意，我们不需要把所有逻辑都放在脚本字符串中，我们可以只提取重要信息，再调用方法统一处理，我们假设一个 `createElement(tagName, options, children)`(同 Vue 的 `createElement`) 方法该方法接受三个参数：`tagName` 为元素节点的 tagName，`options` 是元素的各种属性（我们只实现属性和事件），`children` 元素的子元素。这样我们只需将模版处理成这样即可：`createElement(tagName, {attrs: ..., event: ...}, [createElement(...), ...])`。所以我们要做的就是拼接字符串：

```ts
const process = function(element: Element | Text) {
    let code = `_c("${element.localName}",`;

    code += processAttrs(element);

    let children = toArray(element.childNodes).map(process);
    code += `,[${children.join(',')}]`;

    return code += ')';
};
```

我们的处理过程大致分三步骤，标签名、属性、子元素。标签我们直接拿到元素节点的标签名并拼接字符串即可(用`_c` 代替 `createElement`)，属性最后处理成类似对象的字符串即可，子元素递归调用 `process` 处理，拼接上必要的代码(`,;)[]`之类的)。这样大致的骨架就出来了。

要注意的是我们只处理了元素节点，还有文本节点的情况 `<div>文本</div>` 我们来优化下代码：

```ts
const noSpaceAndLineBreak = /\s*|[\r\n]/g;
const escape = /({{([\s\S]+?)}})+/g;

const process = function(element: Element | Text) {
    let code = '';
    // 元素节点
    if (element instanceof Element) code = `_c("${element.localName}",`;
    // 文本节点
    else if (element instanceof Text) {
        let text = element.wholeText.replace(noSpaceAndLineBreak, ''); // 去掉空格会车
        let newText = text.replace(escape, function(match: string) {
            // 处理 ref 的情况 用 _v 方法包起来
            return `\${_v(${match.slice(2, -2)})}`;
        });
        return `\`${newText}\``;
    }
    else return;

    code += processAttrs(element);

    let children = toArray(element.childNodes).map(process);
    code += `,[${children.join(',')}]`;

    return code += ')';
};
```

文本节点和元素节点有所不同，它没有标签名和属性，只有内容，所以我们直接输出字符串，后续交给 `createElement` 处理。同时文本节点可能会出现 `{{value}}` 的语法，我们要对其处理：

首先我们通过正则去掉多余的换行符和空格（`noSpaceAndLineBreak`,有点粗糙），接着我们通过正则 `escape` 去替换 `{{xxx}}` 注意，双括号内的是变量，外面的是静态字符串，我们处理成 es6 的 `` `${}` `` 语法供render函数调用，这里我们用的也是 es6 语法处理，别搞混了。还要注意的是，这里变量可能是通过 `ref` 方法处理的响应式对象，可能要用 `xxx.value` 取值，所以我们包一层 `_v()` 方法后续再处理，所以最终我们的效果是这样的： `` xxx{{yyy}}xxx => `xxx${_v(yyy)}xxx` ``。

好了，我们继续来实现 `processAttrs` 处理元素属性：

```ts
const processAttrs = function({ attributes }: Element) {
    let code: string[] = [];
    let options: elmOption = {
        attrs: [],
        event: []
    };
    let attrs: any[] = Array.prototype.slice.call(attributes);
    attrs.forEach(({name, value}) => {
        if (name[0] === ':') options.attrs.push(`${name.slice(1)}:${value}`);
        else if (name[0] === '@') options.event.push(`${name.slice(1)}:${value}`);
        else options.attrs.push(`${name}:"${value}"`);
    });

    Object.keys(options).forEach(key => {
        if (options[key].length > 0) code.push(`${key}:{${options[key].join(',')}}`);
    });

    return `{${code.join(',')}}`;
}
```

元素属性有三种情况：原生的元素属性、`:`开头的动态属性、`@`开头的事件。我们规范下 options 的结构：`{attr: {key: value or expression}, event: {key: expression}}`，注意原生属性是静态的，其 value 要包上双引号 `"xxx"`，其余的保留表达式，不过 key 要把前缀删掉。最终效果这样:

```html
<div class="aaa" :value="bbb" @click="ccc"></div>
<!--处理成 文本-->
{
    attrs: {
        class: "aaa",
        value: bbb
    },
    event: {
        click: ccc
    }
}
```

我们是处理成文本，所以留意里面的文本和表达式。（同样有 `ref` 的情况，不过这里是单纯表达式，可以后续处理）。

`compile` 方法我们基本实现了，我们来试试：

```html
<div id="app">
    输入框：
    <input class="name" :value="input" @input="update"></input>
    <div>我的输入：{{input}}</div>
</div>

<script>
let a = document.querySelector('#app');
console.log(compile(a));
// 结果：
(function anonymous(
) {
with(this) {return _c("div",{attrs:{id:"app"}},[`输入框：`,_c("input",{attrs:{class:"name",value:input},event:{input:update}},[]),``,_c("div",{},[`我的输入：${_v(input)}`]),``])}
})
</script>
```

ok继续，在实现 `createElement` 之前我们先想想我们要干嘛，现在我们可以直接创建元素，但是我们肯定不希望每次视图更新都去替换元素，我们需要复用，所以我们需要有一个 diff 过程，而直接创建元素去 diff 太浪费，所以我们还需要一个虚拟 dom 来方便我们 diff。

## virtual dom

virtual dom 所要收集的信息和我们 compiler 过程的差不多，也是 tagName、属性和事件：

```ts
class VNode {
    tagName?: string;
    attrs: elmOption = {};
    event: EventOption = {};
    children: VNode[] = [];
    el: Element | Text | undefined;
    nodeValue?: string;
    type: NodeType;

    constructor({tagName, attrs, event, children, nodeValue, type}: VNodeOption) {
        this.tagName = tagName;
        attrs && (this.attrs = attrs);
        event && (this.event = event);
        children && (this.children = children);
        this.nodeValue = nodeValue;
        this.type = type;
    }
}
```

另外的我们需要存 `type` 节点类型来支持文本节点，`nodeValue` 为文本元素的值，`el` 用于存放真实元素对象。有了这些就足够描述一个真实元素了（目前），我们来实现一下 `createElement` ：

```ts
const createElement: createElement = function(tagName, { attrs, event }, children = []) {
    let vnodeList: VNode[] = children.map(item => {
        if (typeof item === 'string') return createText(item);
        else return item;
    });
    return new VNode({
        tagName,
        attrs,
        event,
        children: vnodeList,
        type: NodeType.Element
    });
};

const createText: createText = function(val) {
    return new VNode({
        nodeValue: val,
        type: NodeType.Text
    });
};
```

之前数据处理的很相似了，所以我们直接拿到去生成 vnode 即可。要注意的是之前文本节点我们直接就传了字符串，所以这里要做一下处理。

好了有了这些我们可以去实现主体代码了。

## createApp

我们用函数来构建实例，实例提供 `mount` 方法挂载页面，大概流程就是 `compiler` 处理模版得到 render 函数，处理 `setup` 得到数据，`effect` 包装 render 调用过程。我们来实现下：

```ts
const createApp = function(options) {
    let instance = createInstance(options);
    let app = {
        $option: options,
        component: instance,
        mount(selector) {
            ...
        }
    };

    return app;
};

const createInstance = function(options) {
    let instance = {
        $option: options,
        _c: createElement,
        _v: getValue,
        proxy: {}
    };

    return instance;
}
```

app 实例中 `$option` 储存原始配置，`mount` 用于向目标元素挂载，`component` 存组件实例，其中会有 `_c`、`_v` 供 render 函数调用，还会存各种处理的结果。大致结构就是这样，我们来实现下 `mount` 方法：

```ts
mount(selector) {
    let el = document.querySelector(selector);
    if (!el) return;

    let instance = this.component;
    instance.el = el;
    instance.render = compile(el);
    
    processSetup(instance);

    let vnode = instance.render!.call(instance.proxy);
    let oldVNode = instance.vnode;
    instance.vnode = vnode;

    patch(oldVNode, vnode, instance);

    return app;
}

const processSetup = function(instance) {
    let { setup } = instance.$option;
    if (setup) {
        let setupResult = instance.setupResult = setup.call(instance, createElement);
        instance.proxy = new Proxy(instance, {
            get: (target, key) => {
                if (key in setupResult) return setupResult[key];
                else return Reflect.get(target, key);
            },
            set: (target, key, value) => {
                if (key in setupResult) setupResult[key] = value;
                return true;
            },
            // 一定要有 has 不然 with 语句拿不到
            has(target, key) {
                return key in setupResult || Reflect.has(target, key);
            }
        });
    }
}
```

`mount` 方法接受一个元素选择器，我们可以通过选择器拿到我们要挂载的 dom 元素，首先将目标 dom 树编译成 render 函数。接着处理 `setup` 方法拿到我们的数据对象，并代理到 instance 上，这样我们就可以用 `this.xxx` 获取对应数据。接着我们调用 render 方法获得 `vnode`。 有了 vnode 我们就可以去生成元素挂载啦。

我们先来实现初始化时候的挂载，这个时候只需要生成一个 dom 树并替换调原先的 dom 树即可：

```ts
const patch = function(oldVNode, vnode, instance) {
    if (!oldVNode) {
        let el = vnodeToElm(vnode);
        if (el && instance.el) instance.el.parentNode!.replaceChild(el, instance.el);
        return;
    }
}

const vnodeToElm = function(vnode: VNode) {
    if (vnode.type ===  NodeType.Text) {
        let el = document.createTextNode(getValue(vnode.nodeValue) || '');
        vnode.el = el;
        return el;
    };
    if (!vnode.tagName) return;

    let el = document.createElement(vnode.tagName);

    for (let key in vnode.attrs) {
        el.setAttribute(key, getValue(vnode.attrs[key]));
    }
    
    for (let key in vnode.event) {
        el.addEventListener(key, vnode.event[key]);
    }
    if (vnode.children.length > 0) {
        vnode.children.forEach(v => {
            let child = vnodeToElm(v);
            child && el.appendChild(child);
        });
    }
    vnode.el = el;
    return el;
}
```

我们通过 vnode 上的属性去创建新元素，并为其一个个添加属性，最后通过 `replaceChild` 将旧节点整个替换。最后我们会在 vnode 上保存对应的元素 el 我们来看看效果：

```html
<html>
    <body>
        <div id="app">
            输入框：
            <input class="name" :value="input" @input="update"></input>
            <div>我的输入：{{input}}</div>
            <button @click="clickHandler">点击</button>
            你点了：{{data.count}} 下
        </div>
    </body>
    <script src="./dist/bundle.js"></script>
    <script>
        const { reactive, ref, createApp } = miniVue;

        createApp({
            setup() {
                let input = ref(123);
                let data = reactive({
                    count: 0
                });
                return {
                    input,
                    data,
                    update: (val) => void (input.value = val.target.value),
                    clickHandler: () => void (data.count += 1)
                }
            }
        }).mount('#app');
    </script>
</html>
```

![结果][result]

很好,页面成功的挂载了，不过现在还不是响应式的，我们继续让其变成响应式，我们用 `effect` 方法老包裹 render 方法处理和 patch 过程：
```ts
// in mount function
instance.update = effect(function() {
    let vnode = instance.render!.call(instance.proxy);
    let oldVNode = instance.vnode;
    instance.vnode = vnode;

    patch(oldVNode, vnode, instance);
});
```

这样第一次处理并调用 render 函数时会进行依赖收集，之后每次数据变化都会调用 `instance.update` 来实时刷新页面，我们接着完善下 `patch` 方法，添加下 diff 过程。

## diff

我们简单实现下 diff 过程，首先我们只同级比较，比较时只判断元素类型和 tagName (我们不实现 key)，相似就复用，不相似就替换。下面是代码：

```ts
const patch = function(oldVNode: VNode | undefined, vnode: VNode, instance: ComponentInstance) {
    if (!oldVNode) {
        // 同上
    }

    if (!isSameVNode(oldVNode, vnode)) {
        let el = vnodeToElm(vnode);
        if (el && oldVNode.el) oldVNode.el.parentNode!.replaceChild(el, oldVNode.el);
    } else {
        if (vnode.type === NodeType.Text && oldVNode.nodeValue !== vnode.nodeValue) {
            vnode.nodeValue && (oldVNode.el!.nodeValue = vnode.nodeValue);
        } else {
            updateAttrs(oldVNode, vnode);

            vnode.children.forEach((child: VNode, index: number) => void patch(oldVNode.children[index], child, instance));
        }

        vnode.el = oldVNode.el;
    }
}

const isSameVNode = function(oldVNode: VNode, vnode: VNode) {
    return oldVNode.type === vnode.type && oldVNode.tagName === oldVNode.tagName;
}

const updateAttrs = function(oldVNode: VNode, vnode: VNode) {
    if (!(oldVNode.el instanceof Element)) return;
    let { attrs = {} } = vnode;
    let { attrs: oldAttrs = {} } = oldVNode;
    // 设置新的和修改的属性
    for (let key in attrs) {
        if (!(key in oldAttrs) || oldAttrs[key] !== attrs[key]) {
            oldVNode.el!.setAttribute(key, getValue(attrs[key]));
        }
    }
    // 删除没有的属性
    for (let key in oldAttrs) {
        if (!(key in attrs)) {
            oldVNode.el!.removeAttribute(key);
        }
    }
}

const getValue = function(target: any) {
    return isRef(target)? target.value : target;
}
```

我们对每一个节点进行比较，不相似就类似于初始化的过程直接整个 dom 替换，相似就复用之前的元素。复用时要注意文本节点的话判断并替换 `nodeValue` 即可。

元素节点节点需要对新老属性进行判断 `updateAttrs`，简单的说以新的为准，属性都改成新的值，没有的就从旧的中删除。因为真实 dom 元素都存在 `vnode.el` 中，所以我们可以直接进行操作。这里要注意，拿到的值可能为 `ref` 对象，所以需要简单的判断下来支持 `ref`。

最后递归处理 `children`，整个过程也就处理完了。现在我们的页面应该是响应式的了，我们来看看：

![最终结果][finish]

ok，一个简单 mvvm 框架就实现了。

[🔗源码地址][repo]

[result]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/miniVue/1583054792819.jpg
[finish]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/miniVue/myVue.gif
[repo]:https://github.com/jwdzzhz777/miniVue