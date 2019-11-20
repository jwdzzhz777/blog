# 从 Eggjs 项目中抽组件(Typescript)
我们写项目的时候最喜欢干的优化应该就是就是解耦了, 把一块功能独立出来封装成独立的插件，清晰整洁，强迫症表示一本满足。
***
本文就是我从 `eggjs` 项目中抽出一个组件的过程和坑。

首先 **啥是插件？**

一个插件其实就是一个『迷你的应用』，和应用（app）几乎一样：
* 它包含了 `Service`、`中间件`、`配置`、`框架扩展等等`。
* 它没有独立的 `Router` 和 `Controller`.
* 它没有 `plugin.js`，只能声明跟其他插件的依赖，而不能决定其他插件的开启与否。 *摘抄自官网*

所以抽的时候根据现有结构把相关代码抽出来就好了。

**那就开始**

先把代码抽出来，放在 `/lib/genome` 中，做成内置的插件，参考[渐进式开发][egg-url]。

我要抽的是功能类似于 `swagger` 的自动生成 `api` 的插件，
主要涉及到这些地方：
* 一个拓展的 `helper` 方法
* 一个用于拦截路由的中间件
* 插件自身默认的 `config` 配置
* `app.ts` 中注册中间件
* 一个页面
* 其他代码

那么抽完大概就这样：

```
.
├── app
│   ├── extend
│   │   └── helper.ts
│   ├── lib
│   │   ├── decorators.ts
│   │   ├── types.ts
│   │   └── utils.ts
│   ├── middleware
│   │   └── genome.ts
│   └── view
│       └── api.njk
├── app.ts
├── config
│   └── config.default.ts
├── index.ts
└── package.json
```
> note：`package.json` 是需要的，在这里主要用来声明插件
> ```
>"eggPlugin": {
>    "name": "genome",
>    "dependencies": [
>        "nunjucks"
>    ]
>}
> ```

因为还在项目中所以不需要 `tsconfig.json` 以及 `package.json` 的其他配置。
而且 `egg-ts-helper` 也会自动添加 `d.ts` 文件。
自己在写一个额外的 `index.d.ts` 就可以使用了。

```ts
import 'egg';

declare module 'egg' {
    interface Controller {
        /** 此方法只有 api 装饰器修饰过的 controller 才有！！ */
        getMetadata(name: string): any;
    }
}
```
接下来像正常的插件一样使用就可以了，在 `plugin.ts` 中通过 `path` 来挂载插件：
```ts
nunjucks: {
    enable: true,
    package: 'egg-view-nunjucks'
},
genome: {
    enable: true,
    path: join(__dirname, '../app/lib/plugin/egg-genome')
}
```
> note: 因为插件没有 `plugin.ts` 所以如果插件依赖其他插件的话只能在项目中手动挂载依赖项

**运行一下**

`/path` 路由成功跳转我的页面了

![success][success_url]

**抽成单独的组件**

接下来就是正式抽离项目了
就像正常的项目一样初始化并吧代码复制进来，不一样的是需要加上自己的 `tsconfig.ts`了，而且 `package.json` 也要完整一下。添加一下依赖：
```json
"dependencies": {
    "reflect-metadata": "^0.1.13"
},
"devDependencies": {
    "egg": "^2.25.0",
    "tslib": "^1.10.0",
    "tslint": "^5.20.1",
    "tslint-config-egg": "^1.0.0",
    "typescript": "^3.7.2"
},
```
> note: 注意添加 `tslib` 和 `tslint-config-egg` 两个依赖

因为发布的插件必须是 `.js` 的所以需要把 `.ts` 翻译成 `.js` 过后再发布，增加 `script`:

```json
"scripts": {
    "pub": "yarn tsc && yarn publish --non-interactive && yarn clean",
    "tsc": "tsc -p tsconfig.json",
    "clean": "tsc -b --clean"
},
```
添加发布的文件和关键字
```json
"files": [
    "index.js",
    "app.js",
    "*.d.ts",
    "app/view/**",
    "app/**/*.js",
    "config/**/*.js"
],
"keywords": [
    "egg",
    "egg-plugin",
    "eggPlugin",
    "genome",
    "egg-genome"
]
```
**ok, 发布，试试效果**

emmmmmmmmmm...

![error][error_url]

一开始我还以为是引入的问题 导致的不能解构赋值，一直在 `tsconfig.json` 的 `"module": "commonjs",` 这一行改来改去，
后来发现 哪怕我 `import * as xxx from` 引入虽然不报错，但得到的压根是 `undefined`。
应该是入口 `index.ts` 有问题。

于是查了查，再看看别人的插件，看来是我对 ts 的了解果然还是不够深，其实是 `index.d.ts` 的编写有问题，我只写了额外的 `egg` 声明，
插件的模块声明没有写..好在我们是 ts 写的组件 自动生成一下就好了。 `tsconfig.json` 加这么一行 `"declaration": true,`

这样所有文件就会自动生成 `.js` 和 `.d.ts` 文件了。

然后 `package.json` 也改一下：
```json
"files": [
    "index.js",
    "app.js",
    "*.d.ts",
    "app/view/**",
    "app/**/*.js",
    "config/**/*.js",
    "app/**/*.d.ts",
    "config/**/*.d.ts"
],
```

最后把原先 `index.d.ts` 中的东西挪到 `index.ts` 中
```ts
export * from './app/lib/types';
export * from './app/lib/decorators';

import 'egg';

declare module 'egg' {
    interface Controller {
        /** 此方法只有 api 装饰器修饰过的 controller 才有！！ */
        getMetadata(name: string): any;
    }
    interface Context {

    }
}
```

**最后打包发布**
ok, 成功。
[附上项目地址][github_url]

***

总结一下：其实过程中并没有遇到什么很大的问题，都是一些小细节(也有可能是我忘了)，写这个的目的也是鼓励自己多做尝试，多写一些有意思的东西，并在一开始就做好单独抽出来的准备，进行[渐进式开发][egg-url]。毕竟结果不重要，但过程很享受。

[egg-url]:https://eggjs.org/zh-cn/tutorials/progressive.html
[success_url]:https://cvws.icloud-content.com/B/AfLrlZkkbHBsgIYBqUTqSaFGqayZAa3imDgQTt98xo7_ipgtJ1CCo2zM/1574058545720.jpg?o=AhVJcy0E71YTpKD_CDO6TjHr0aN16uC0r5JSmtDR6Znr&v=1&x=3&a=CAogzs1irz-Vt5VlGVcpp1mctT8wo7wLUrNFza2hA0VlMlISHRCs0bny5y0YzMjw8uctIgEAUgRGqayZWgSCo2zM&e=1574077998&k=Q78IGZbo7oJkQd91SqS1Gg&fl=&r=43660daa-035d-4380-8673-063383d43900-1&ckc=com.apple.clouddocs&ckz=com.apple.CloudDocs&p=25&s=Wyi_aNXLVMBxezVKQ8Nollvm36M&cd=i
[error_url]:https://cvws.icloud-content.com/B/AWpt8OSBrfQY4DIwsKISlzSLB76RAbwIFd701ksC5c6XMkM96NXIK7Tz/1574061095736.jpg?o=AoulR7ykUfCgvnQsTz1zrg8coPm1HQQe-Vgn2tunVDsC&v=1&x=3&a=CAog3MeD5-7oHrS8eHBrNk0-BWTkTZCeQ5Pbatyi5NlDzN4SHRCfnOvq5y0Yv5Oi6-ctIgEAUgSLB76RWgTIK7Tz&e=1574062033&k=8R9JPYvwGWPlOuGlkN93EA&fl=&r=7dd2ec8c-104d-47b4-9a16-2b5dfdd9aba2-1&ckc=com.apple.clouddocs&ckz=com.apple.CloudDocs&p=25&s=54AMPPUNZ8thjfrtEzr-UIbpl2U&cd=i
[github_url]:https://github.com/jwdzzhz777/genome
