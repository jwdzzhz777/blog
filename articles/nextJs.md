# next.js 使用记录

最近正在写博客(不出意外就是this)，考虑到页面用什么框架，想了许久最后选择了 `next.js`，不为别的就为了 `seo` 虽然内容本身就放在 `Issue` 中可以被搜索引擎找到，但多学点总没错。

### Quick Start
根据[教程][quick_start]来就是了，如果是要用 `typescript` ,还需要依赖 `@types`:
```
npm install --save-dev typescript @types/react @types/node
```
`tsconfig.json` 也会自动帮你生成。
直接在 `pages` 下面新建个文件 `index.tsx`:
```tsx
const Home=() => <h1>Hello world!</h1>;

export default Home;
```
然后运行 `next dev` 就跑起来啦。

### alias

写代码的时候引入其他位置的文件总是要用相对路径很麻烦，所以我一般会用 `webpack` 给一些路径设置别名。<br>
`next.js` 设置别名在配置文件中，默认是没有的，所以我们手动新建一个 `next.config.js` 文件。<br>
比如我要给 `components` 文件夹设置别名：
```js
var path = require('path');

const resolve = (dir) => {
    return path.join(__dirname, dir);
};

module.exports = {
    webpack: (config) => {
        Object.assign(config.resolve.alias, {
            'components': resolve('components')
        });
        return config;
    }
}
```
直接在 `webpack` 配置下修改 `config` 配置就行了。然后试一试：
```tsx
import Some from 'components/some';

const Home = () => {
    return (
        <>
            <h1>Hello world!</h1>
            <Some />
        </>
    );
};

export default Home;
```
> note: 如果项目接入了 `typescript` 可能会报**找不到模块**，
> 在 `tsconfig.json` 的 `compilerOptions` 下设置 `"baseUrl": "."` 即可。

### 环境变量
同样的，我有时候也会设置一些环境变量，也是直接在配置文件中去设置，比如我想设置该项目请求的域名：

```js
module.exports = {
    webpack: ...,
    env: {
        API_HOST: 'api.com'
    }
}
```
代码中就可以直接用 `process.env.API_HOST` 了。<br>
当然这是构建时的配置(`Build-time`)，我们也可以做运行时的配置(`Runtime`):
```js
// next.config.js
module.exports = {
    env: {
        API_HOST: 'api.com'
    }
    serverRuntimeConfig: {
        host: procsss.env.API_HOST
    },
    publicRuntimeConfig: {
        staticFolder: '/static',
    },
}
```
`serverRuntimeConfig` 存放仅有在服务端可以获取的配置, 而 `publicRuntimeConfig` 下的配置在服务端和客户端均可获取。
```js
// pages/index.js
import getConfig from 'next/config'

const { serverRuntimeConfig, publicRuntimeConfig } = getConfig();

console.log(serverRuntimeConfig.host) // 仅在服务端可用
console.log(publicRuntimeConfig.staticFolder) // 均可用
```

### less 支持
官方提供了 [next-less][next-less] 来对 `less` 提供支持。
```
yarn add @zeit/next-less less
```
配置的时候很有意思，它提供了一个方法来返回 `less` 的配置，我们需要把我们原有的配置传入方法
```js
// in next.config.js
const withLess = require('@zeit/next-less');
var path = require('path');
const config = require('./config');

const resolve = (dir) => {
    return path.join(__dirname, dir);
}

module.exports = withLess({
    webpack: (config) => {
        Object.assign(config.resolve.alias, {
            'components': resolve('components')
        });
        return config;
    },
    env: {
        ...config[process.env.NODE_ENV]
    }
});
```
创建一个 `style.less`:
```css
@font-size: 50px;
.example {
    font-size: @font-size;
}
```
我们就可以在页面里引入 `less` 文件了
```
import "./styles.less";
```
要注意的是这样引入是全局的，要做好命名空间的约定，每个文件用类名包裹。
也可以用 `css modules`：
```js
// next.config.js
const withLess = require('@zeit/next-less')
module.exports = withLess({
  cssModules: true
})
```
```js
mport css from "../styles.less"

export default () => <div className={css.example}>Hello World!</div>
```
不过说实话这样真不好用，个人感觉。

### 设置公共部分
假如页面有个公共的 `header` ，我总是希望有一个 `Layout` 里面有页面的公共部分，比如 `<header>`、`<footer>` 然后中间才是我们的页面，在 `Vue` 中我们可以用 `Router` 实现,但在Next.js中所有页面都是独立的 page。官方的教程里也是每个 page 引入一个 `<header>` 这显然不是我想要的。

#### Custom `<App>`
Next.js 用一个 `<App>` 组件去渲染每一个 page ,所以我可以自定义 App 组件来实现一个公共的 `<Layout>`。<br>
创建 `/pages/_app.tsx`:
```ts
import React from 'react'
import App from 'next/app'

class MyApp extends App {
    render() {
        const { Component, pageProps } = this.props
        return <Component {...pageProps} />
    }
}

export default MyApp
```
原始的 App 组件就长这样，现在我可以加一些自己的东西。先搞个 `Layout` 组件：
```ts
// in components/layout.tsx
import React from 'react';
import Head from 'next/head'
import Header from './header';

export default class Layout extends React.Component {
    render() {
        const { children, viewer } = this.props;
        const { snakeBar } = this.state;
        return (
            <>
                <Head>
                    <meta name="viewport" content="initial-scale=1.0, width=device-width" key="viewport" />
                </Head>
                <Header/>
                {children}
            </>
        )
    }
}
```
比如我设置一个统一的 `viewport` 或者 `title`,再给页面搞个统一的 `Header` 。
再改造下我们的 `_app.tsx` 就可以了：
```ts
import React from 'react'
import App from 'next/app'
import Layout from 'components/layout';

class MyApp extends App {
    render() {
        const { Component, pageProps } = this.props
        return (
            <Layout>
                <Component {...pageProps} />
            </Layout>  
        );
    }
}

export default MyApp
```
这样我们还可以在 `Layout` 里面设置全局样式
```ts
// in layout.tsx
import 'styles/global.less';
```
虽然可以直接在任意文件设置全局样式,但我并不喜欢这种方式：
```ts
export default () => {
    return (
        <>
            <p>hi</p>
            <style global jsx>{`somestyle`}</style>
        </>
    )
};
```
### 给每个页面提供数据
现在我有了公共的 `header` 现在我想给每个页面提供数据。我们甚至可以在 MyApp 中定义   `getInitialProps` 方法。
```ts
import App from 'next/app';
import Layout from 'components/layout';

export default class MyApp extends App {
    static async getInitialProps({ Component, ctx }) {
        /** 注意这里一定要调一把 Component.getInitialProps(ctx) 拿到 props */
        /** 否则每个页面会拿不到 getInitialProps 的数据 */
        const pageProps = await Component.getInitialProps(ctx);
        let data = await curl(...);
        return { pageProps: {...pageProps, data} };
    }
    render() {
        const { Component, pageProps } = this.props as any;
        return (
            // 也可以传给 layout
            <Layout data={pageProps.data}>
                <Component {...pageProps} />
            </Layout>
        );
    }
}
```
> note:在 `_app.tsx` 中使用 `getInitialProps` 有个弊端，这样每个页面都会被服务端渲染，next.js 有个功能是 [Automatic Static Optimization][static-optimization]
即会自动把没有阻塞数据要求的页面优化为静态页面，它是根据 `getInitialProps` 方法判断的。

这样每个页面的 props 里都能拿到 data 了。同样的我也可以把请求错误的 error 传到 layout 中做一些处理，这样即便我在服务端调的接口报错了，我也能在页面上提示出来。
```ts
import App from 'next/app';
import Layout from 'components/layout';

export default class MyApp extends App {
    static async getInitialProps({ Component, ctx }) {
        const pageProps = await Component.getInitialProps(ctx);
        try {
            let data = await curl(...);
            return { pageProps: {...pageProps, data} };
        } catch (error) {
            return { pageProps, error: error.message };
        }
    }
    render() {
        const { Component, pageProps, error } = this.props as any;
        return (
            <Layout error={error} data={pageProps.data}>
                <Component {...pageProps} />
            </Layout>
        );
    }
}
```
```ts
// in components/layout.tsx
import React from 'react';
import Head from 'next/head'
import Header from './header';

export default class Layout extends React.Component {
    render() {
        const { children, error } = this.props;
        const { snakeBar } = this.state;

        if (error) {
            // do something in this
        }

        return (
            <>
                <Head>
                    <meta name="viewport" content="initial-scale=1.0, width=device-width" key="viewport" />
                </Head>
                <Header/>
                {children}
            </>
        )
    }
}
```
> note: 提一嘴 `getInitialProps` 方法在第一次进入页面或者刷新的时候都是在服务端调用的，而路由跳转的情况是在客户端调用的。

### 构建优化
`next.js` 的构建部署就比较简单，构建运行就可以了。
```
npm run build
npm run start
```
试着访问了下，效果并不尽人意，加载速度非常慢，一方面服务端渲染请求的速度比较慢，另一方面打包完的资源比较大，因为我引入了几个比较大的模块，再加上提取了公共部分 `_app` ,有必要优化一下。

**包分析**

首先安装 next 的包分析插件。
```
yarn add @next/bundle-analyzer
```
在 `next.config.js` 中编辑，配合环境变量一起使用：
```js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
})
module.exports = withBundleAnalyzer(otherPlugins())
```
然后可以运行命令分析构建的大小
```
ANALYZE=true yarn build
```
完事会生成两个两个报告一个是 server 端一个是 client 端，这里主要看下 client 端，作为服务端渲染 `Next.js` 是个多页面的配置，每个 page 都会作为入口，其依赖都会被打包到单独的 chunk 中。

![analyze][chunk_img]

 `article.js` 太大了，其本身是展示文章的页面，需要依赖 markdown 相关的模块，其中的 `highlight.js` 占据了全部的大小。要对其优化下。

首先我希望 `highlight.js` 被放在单独的 chunk 中，不和 article 页面捆绑在一起，因为我希望有时内容并不是 markdown 时他能被按需加载。

现在我尝试将组件转化为动态组件，Next.js 的 [dynamic imports][dynamic] 提供了动态组件的支持:

```ts
// import Highlight from 'react-highlight';
import dynamic from 'next/dynamic';
const Highlight: any = dynamic(() => import('react-highlight'));
/** react-highlight 中依赖了 highlight.js */
```
ok,利用 `import()` 语法webpack 能很好的分割出 `react-highlight`, 现在它也能够被按需利用了：

![first_optimize][first_optimize]

现在 `react-highlight` 都被分割在了 chunks/10 中，但是它还是太大了，这么大是因为 `highlight.js` 里面包含了各种语言，而我会用到的可能就其中的一小部分，即便我尝试单独 import 其中部分语言，它还是会把所有语言都打包到一块去，现在我要手动处理下：

**webpack.ContextReplacementPlugin**

`ContextReplacementPlugin` 的功能很简单，允许覆盖查找规则。我们把 `highlight.js` 的查找路径改为其中部分语言的路径，这样构建的时候就会根据我们覆盖的路径进行打包：

```js
//  in next.config.js
module.exports = {
    webpack: (config, { webpack }) => {
        config.plugins.push(new webpack.ContextReplacementPlugin(
            /highlight\.js\/lib\/languages$/,
            new RegExp(`^./(${['javascript', 'typescript', 'bash', 'basic', 'json'].join('|')})$`)
        ));
        return config;
    },
}
```

这样 webpack 匹配 `highlight.js/lib/languages` 时会替换成其中几个语言 `highlight.js/lib/languages/{somelanguage}` , 大大减少了构建的大小：

![first_optimize_done][first_optimize_done]

ok, 它现在被压缩到非常能接受的范围了。

**SplitChunksPlugin**

再仔细看下报告图我还发现了一些问题,之前利用了 `_app.tsx` 提取了公共部分，然后项目又用到了 `@material-ui` ,在公共入口处引用了 `@material-ui` 的主题，这样的话每个页面都有 `@material-ui` 相关的代码，它应该在 `commons.xxx.js` 的块中，而现实是他出现在每个有引用 `@material-ui` 的地方。

![second_optimize][second_optimize]

去看一下 `Next.js` 默认配置：

```ts
/** in next/build/webpack-config.ts */
const splitChunksConfigs: {
    [propName: string]: webpack.Options.SplitChunksOptions
  } = {
    prod: {
      chunks: 'all',
      cacheGroups: {
        default: false,
        vendors: false,
        commons: {
          name: 'commons',
          chunks: 'all',
          minChunks: totalPages > 2 ? totalPages * 0.5 : 2,
        },
        react: {
          name: 'commons',
          chunks: 'all',
          test: /[\\/]node_modules[\\/](react|react-dom|scheduler|use-subscription)[\\/]/,
        },
      },
    }
}
```

官方并没有对 `_app` 做特殊的配置，分包的策略只是将 `react|react-dom|scheduler|use-subscription` 放在 commons 中，然后是引用次数大于页面数量的一半的依赖放也在 commons 中。这就不太合理了，我可不想每个页面都加载 `@material-ui`，我们来手动优化下：

```js
//  in next.config.js
module.exports = {
    webpack: (config, { webpack }) => {
        config.optimization.splitChunks &&
        (
            config.optimization.splitChunks.cacheGroups.react.test =
            /[\\/]node_modules[\\/](react|react-dom|scheduler|use-subscription|@material-ui\/styles\/esm|@material-ui\/core\/esm\/styles)[\\/]/
        );
        return config;
    },
}
```

我手动把 `@material-ui` 中关于主题的一部分一起放到了 `commons` ,其他部分还是让他按照规则自动分配：

![second_optimize_done][second_optimize_done]

嗯嗯，现在访问就快多了。

### 未完待续
[quick_start]:https://nextjs.org/learn/basics/getting-started/setup
[next-less]:https://github.com/zeit/next-plugins/tree/master/packages/next-less
[static-optimization]:https://nextjs.org/docs#automatic-static-optimization
[dynamic]:https://nextjs.org/docs/#dynamic-import

[chunk_img]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/nextJs/1577435701641.jpg
[first_optimize]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/nextJs/1577440273496.jpg
[first_optimize_done]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/nextJs/[first_optimize_done]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/nextJs/1577440273496.jpg.jpg
[second_optimize]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/nextJs/[first_optimize_done]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/nextJs/1577689958466.jpg.jpg
[second_optimize_done]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/nextJs/[first_optimize_done]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/nextJs/1577694466593.jpg.jpg
