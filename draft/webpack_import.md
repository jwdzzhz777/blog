# webpack 之 dynamic imports

说一下背景，事情是这样的公司用的是 Vue，正常的 vue-router 路由的写法应该是这样的：

```js
import Home from '../views/Home.vue'

const routes = [
  {
    path: '/',
    name: 'Home',
    component: Home
  },
  {
    path: '/about',
    name: 'About',
    // route level code-splitting
    // this generates a separate chunk (about.[hash].js) for this route
    // which is lazy-loaded when the route is visited.
    component: () => import(/* webpackChunkName: "about" */ '../views/About.vue')
  }
]
```

如果想路由的页面支持路由懒加载并分块打包就需要用到 `import()` 语法，打包出来的代码是这样的：

```bash
  File                                 Size               Gzipped

  dist/js/chunk-vendors.65a588bb.js    117.44 KiB         41.47 KiB
  dist/js/app.85fc59df.js              6.58 KiB           2.47 KiB
  dist/js/about.292c2c3b.js            0.44 KiB           0.31 KiB
  dist/css/app.b229284b.css            0.42 KiB           0.26 KiB
```

这都没什么问题。

直到有一天，感觉这样的写法太过于繁琐，我们希望简化 `() => import()` 的操作，于是有了下面的代码：

```js
const routes = [
  {
    path: '/',
    name: 'Home',
    filePath: 'Home'
  },
  {
    path: '/about',
    name: 'About',
    filePath: 'About'
  }
]

routes.map(route => void (route.component = () => import('../views/' + route.filePath + '.vue')));
```

我们用一个字段 `filePath` 来描述组件文件的位置，因为我们的页面都放在 `views` 文件夹下，所以我们拼接成文件的路径 `'../views/' + route.filePath + '.vue'`，运行起来也没毛病。

直到有一天我们对包进行分析发现了个问题，这种写法相比原生写法生成的代码块要多，体积也要大，定位发现这样的写法会对 views 目录下所有的 .vue 文件进行打包。我们新建一个空的 empty.vue 文件再次打包：

```bash
  File                                  Size               Gzipped

  dist/js/chunk-vendors.65a588bb.js     117.44 KiB         41.47 KiB
  dist/js/app.8859f6c5.js               6.65 KiB           2.51 KiB
  dist/js/chunk-2d217a0e.5cdeb313.js    0.46 KiB           0.32 KiB
  dist/js/about.292c2c3b.js             0.44 KiB           0.31 KiB
  dist/css/app.b229284b.css             0.42 KiB           0.26 KiB
```

正常写法打包：

```bash
  File                                 Size               Gzipped

  dist/js/chunk-vendors.65a588bb.js    117.44 KiB         41.47 KiB
  dist/js/app.85fc59df.js              6.58 KiB           2.47 KiB
  dist/js/about.292c2c3b.js            0.44 KiB           0.31 KiB
  dist/css/app.b229284b.css            0.42 KiB           0.26 KiB
```

也就是说 empty.vue 并没有真正用到但还是被分割并打包了，基本上 `views` 下所有文件不管是否被用到都会被打包分割。webpack 处理依赖关系有问题了。没办法只能看看 webpack 究竟干了啥。

***

我们都知道 webpack 的原理是从入口开始解析文件，将代码传换为 ast，找出依赖并递归。 当遇到 `import()` 语法时会进行代码分割，具体是怎么操作的，我们来看看。

我们找到 `webpack/lib/dependencies/ImportParserPlugin.js`，这个是处理import 语法的：

```js
apply(parser) {
    parser.hooks.importCall.tap("ImportParserPlugin", expr => {
        const param = parser.evaluateExpression(expr.arguments[0]);

        let chunkName = null;
        let mode = "lazy";
        let include = null;
        let exclude = null;
        const groupOptions = {};

        const {
            options: importOptions,
            errors: commentErrors
        } = parser.parseCommentOptions(expr.range);

        if (param.isString()) {
            // ...
        } else {
            const dep = ContextDependencyHelpers.create(
                ImportContextDependency,
                expr.range,
                param,
                expr,
                this.options,
                {
                    chunkName,
                    groupOptions,
                    include,
                    exclude,
                    mode,
                    namespaceObject: parser.state.module.buildMeta.strictHarmonyModule
                        ? "strict"
                        : true
                },
                parser
            );
            if (!dep) return;
            dep.loc = expr.loc;
            dep.optional = !!parser.scope.inTry;
            parser.state.current.addDependency(dep);
            return true;
        }
    });
}
```

`expr` 为 import 语法的表达式，正常情况是 `string` 类型的 (`'./xxx'`这种)，但是当其为动态表达式时会有特殊的处理。

首先通过 `parser.evaluateExpression` 方法来解析该表达式，会得到如下信息：

```js
// import('./views' + fileName + '.vue')

let param = {
    prefix: './views/',
    postfix: '.vue',
    expression: {
        // ...
    }
}
```

将固定代码和表达式提取了出来，再通过 `ContextDependencyHelpers` 来处理：

```js
let dep = {
    options: {
        request: './views',
        regExp: /^\.\/.*\.vue$/,
        mode: 'lazy',
        chunkName: null,
        groupOptions: {},
        include: null,
        exclude: null,
        namespaceObject: true
    },
}
```

也就是将 `prefix` 作为 `request`，将 `postfix` 处理为 `/^\.\/.*\.vue$/` 正则表达式，该表达式匹配所有的 `./xxx.vue` 的路径。也就是说表达式在这个时候完完全全被忽略了，因为 webpack 是对文本处理，并不知道上下文内容，所以表达式对他来说可能是 anything。所以它通过 `request` 和 `regExp` 去匹配文件，也就是 `views` 文件夹下所有的 `.vue` 文件。所有匹配到的文件都会被当作 *被依赖的*。最后我们在 `webpack/lib/dependencies/RequireContextPlugin.js` 可以打印出我们被找到的文件:

```js
[
    {
        context: '/path/to/project/views',
        request: './a.vue'
    },
    {
        context: '/path/to/project/views',
        request: './b.vue'
    },
    // ...
]
```

最初我以为会和 `require.context` 有关，因为如果切分为三个参数和它对得上，现在看来并不是，而且该过程同样也适用与 `require.context`。

那么怎么办呢，我们可以手动添加 `include` 和 `exclude` 来缩小范围：

```js
import(/* webpackExclude: /.*\/views\/empty\.vue$/ */ '../views/' + route.filePath + '.vue'));
```

但是手动优化也太蠢了，而且这样写实际就一个 `import()` ，没法做一些其他操作，比如给 chunk 起名，比如分别设置 `prefetch` 和 `preload`。所以...还是老老实实一个个写吧。