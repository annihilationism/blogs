### webpack-chain

#### 什么是 webpack-chain

`webpack-chain` 尝试通过可链式或顺流式的 API 创建和修改 webpack 配置。API 的 Key 部分可以由用户指定的名称引用，这有助于跨项目修改配置方式的标准化。应用一个链式 API 来生成和简化 2-4 版本的 webpack 的配置的修改。

#### 为什么使用 webpack-chain？

webpack 的核心配置的创建和修改基于有潜在难于处理的 Javascript 对象。虽然这对于配置单个项目来说还算 OK 的，但当你尝试跨项目共享这些对象并使其进行后续的修改就会变得混乱不堪。

使用 `webpack-chain` 共享配置很简单，仅仅在导出配置和在传递给 webpack 之前调用 `.toConfig()` 方法将配置导出给 webpack 使用。

#### sideEffects

基于我们对 fp 中的 side effect 的理解，我们可以认为，只要我们确定当前包里的模块不包含副作用，然后将发布到 npm 里的包标注为 sideEffects: false ，我们就能为使用方提供更好的打包体验。原理是 webpack 能将标记为 side-effects-free 的包由 import {a} from xx 转换为 import {a} from 'xx/a'，从而自动修剪掉不必要的 import，作用同 babel-plugin-import。

虽然 b 模块的导出是被忽略了，但是副作用代码被保留下来了。由于目前 transformer 转换后可能引入的各种奇怪操作引发的副作用（参考：你的 Tree-Shaking 并没什么卵用），很多时候我们会发现就算有了 tree shaking 我们的 bundle size 还是没有明显的减小。而通常我们期望的是 b 模块既然不被使用了，其中所有的代码应该不被引入才对。

这个时候 sideEffects 的作用就显现出来了：如果我们引入的 包/模块 被标记为 sideEffects: false 了，那么不管它是否真的有副作用，只要它没有被引用到，整个 模块/包 都会被完整的移除。以 mobx-react-devtool 为例，我们通常这样去用：

```jsx
import DevTools from "mobx-react-devtools";

class MyApp extends React.Component {
  render() {
    return (
      <div>
        ...
        {process.env.NODE_ENV === "production" ? null : <DevTools />}
      </div>
    );
  }
}
```

这是一个很常见的按需导入场景，然而在没有  sideEffects: false  配置时，即便  NODE_ENV  设为  production ，打包后的代码里依然会包含  mobx-react-devtools  包，虽然我们没使用过其导出成员，但是  mobx-react-devtools  还是会被 import，因为里面“可能”会有副作用。但当我们加上 sideEffects false 之后，tree shaking 就能安全的把它从 bundle 里完整的移除掉了。

上面也说到，通常我们发布到 npm 上的包很难保证其是否包含副作用（可能是代码的锅可能是 transformer 的锅），但是我们基本能确保这个包是否会对包以外的对象产生影响，比如是否修改了 window 上的属性，是否复写了原生对象方法等。如果我们能保证这一点，其实我们就能知道整个包是否能设置  sideEffects: false 了，至于是不是真的有副作用则并不重要，这对于 webpack 而言都是可以接受的。这也就能解释为什么能给 vue 这个本身充满副作用的包加上 sideEffects: false 了。

所以其实 webpack 里的 sideEffects: false 的意思并不是我这个模块真的没有副作用，而只是为了在摇树时告诉 webpack：我这个包在设计的时候就是期望没有副作用的，即使他打完包后是有副作用的，webpack 同学你摇树时放心的当成无副作用包摇就好啦！。

也就是说，只要你的包不是用来做 polyfill 或 shim 之类的事情，就尽管放心的给他加上 sideEffects: false 吧！

项目中使用到 less 或者 sass 的话不能直接 sideEffects: false，会被 tree shaking，导致 mini-css-extract-plugin 无法抽取出 css 文件，需要额外排除\*.css。

#### tree shaking

tree shaking 是一个术语，通常用于描述移除 JavaScript 上下文中的未引用代码(dead-code)。它依赖于 ES2015 模块语法的 静态结构 特性，例如 import 和 export。这个术语和概念实际上是由 ES2015 模块打包工具 rollup 普及起来的。

webpack 2 正式版本内置支持 ES2015 模块（也叫做 harmony modules）和未使用模块检测能力。新的 webpack 4 正式版本扩展了此检测能力，通过 package.json 的 "sideEffects" 属性作为标记，向 compiler 提供提示，表明项目中的哪些文件是 "pure(纯正 ES2015 模块)"，由此可以安全地删除文件中未使用的部分。

#### ReactDom

React 在 v0.14 之前是没有 ReactDOM 的，所有功能都包含在 React 里。从 v0.14(2015-10)开始，React 才被拆分成 React 和 ReactDOM。为什么要把 React 和 ReactDOM 分开呢？因为有了 ReactNative。React 只包含了 Web 和 Mobile 通用的核心部分，负责 Dom 操作的分到 ReactDOM 中，负责 Mobile 的包含在 ReactNative 中。具体参考 v0.14 的 release：React v0.14 - React Blog。

ReactDom 只做和浏览器或 DOM 相关的操作，例如：ReactDOM.render() 和 ReactDOM.findDOMNode()。如果是服务器端渲染，可以 ReactDOM.renderToString()。React 不仅能通过 ReactDOM 和 Web 页面打交道，还能用在服务器端 SSR，移动端 ReactNative 和桌面端 Electron。

#### DefinePlugin

DefinePlugin 用于定义全局变量，在 webpack 打包时会对这些变量做替换。

##### bundle

Bundle: bundle 通常是由多个不同的模块产生，它是已经加载完毕和被编译后的源代码的最终版本。

Bundle Splitting: 这是 webpack 优化代码的一种方法，即将一个单独的应用拆解为多个 bundle。通过将多个 bundle 中公共引入的模块拆解出来，我们可以减少项目的代码量，从而减小应用的体积，并且通过浏览器缓存，我们可以将代码进一步优化。

Chunk: 这个 webpack 中专用的术语用于管理 webpack 内部的打包进程。bundle 由许多 chunk 组成，chunk 有几种类型，比如说“入口”和“子块”。通常 chunk 和输出的 bundle 一一对应，但是，有些是一对多的关系。

Code Splitting: 它表示将你的代码拆分成多个 bundle 或 chunk，之后你可以按需加载它们，而不是简单地加载一个单独的文件。

##### webpack target

由于 JavaScript 即可以编写服务端代码也可以编写浏览器代码，所以 webpack 提供了多种部署 target，你可以在 webpack 的配置选项中进行设置。

`devtool`: 此选项控制是否生成，以及如何生成 source map。

#### 不同的 source map 模式的区别

- eval：eval 模式会把每个 module 封装到 eval 里包裹起来执行，并且会在末尾追加注释。
- source-map：该模式会把每个 module 封装到函数中执行，并且生成 `source-map` 文件，在 `output` 文件末尾添加对应的 `map` 注释。
- hidden-source-map：相比较 `source-map` 模式，缺少了末尾的注释。
- inline-source-map：在文件末尾处，sourceMap 作为 DataURL 的形式被内嵌进了 bundle 中，由于 sourceMap 的所有信息都被加到了 bundle 中，整个 bundle 文件变得硕大无比。
- eval-module-map：和 `eval` 类似，但是把注释里的 `sourceMap` 都转换为了 `DataURL`。
- cheap-source-map: 和 source-map 生成结果差不多。output 目录下的 index.js 内容一样。
  但是 cheap-source-map 生成的 index.js.map 的内容却比 source-map 生成的 index.js.map 要少很多代码，我们对比一下上文 source-map 生成的 index.js.map 的结果，发现 source 属性里面少了列信息，只剩一个"webpack:///js/index.js"。
- cheap-module-source-map：在 cheap-module-source-map 下 sourceMap 的内容更少了，sourceMap 的列信息减少了，可以看到 sourcesContent 也没有了。

开发环境推荐：

- eval-source-map：打包速度快，可追溯代码

生产环境推荐：

- cheap-module-source-map （这也是下版本 webpack 使用-d 命令启动 debug 模式时的默认选项）

生产环境使用 sourcemap，可以把 `.map` 文件上传到内网，不发布到外网。

#### webpack optimization

从 `webpack 4` 开始，会根据你选择的 `mode` 来执行不同的优化，不过所有的优化还是可以通过 `optimization` 来手动配置的。

##### optimization.minimize

告知 `webpack` 使用 `TerserPlugin` 或其他在 `optimization.minimizer` 定义的插件来压缩 `bundle`。

##### optimization.minimizer

允许你通过提供一个或多个定制过的 `TerserPlugin` 实例，覆盖默认压缩工具（minimizer）。

##### optimization.splitChunks

对于动态导入模块，默认使用 webpack v4+ 提供的全新的通用分块策略。

###### cacheGroups

至于其他属性会通过另外一篇文章做一个总结。这个主要是了解 cacheGroups 的用法，cacheGroup 的作用是将 chunks 按照 cacheGroup 中给定的条件分组输出，cache 会集成 splitchunks 的基础配置，自己内部也可以自定义这些基础配置自定义的话会覆盖掉 splitchunks 中的配置。webpack 内置了一份默认配置包含了根据 test：node_modules 以及 default 属性的代码分割，这样即使我们不配置 cacheGroups 的情况下只要开启了 chunks:all 也会默认把公用库和业务代码分割。

通过 cacheGroup 能做的是更加细化的对 chunks 进行分割，在代码变和不变之间做更好的分离，比如我们的工具库可能很少更改就可以考虑一个 chunk，这样就和业务分离了。

再就是经过上面的验证，会发现多个 chunk 如果存在规则重叠，实际上不会将同样的代码构建到每个 chunk 中去，而是复用其中的一个 chunk，这样的话我们在配置的过程当中更需要明确代码的边界，尽量避免这种优化的发生，保证每个 test 之间不重叠。

#### optimization.runtimeChunk

将 `optimization.runtimeChunk` 设置为 `true` 或 `multiple`，会为每个只含有 `runtime` 的入口添加一个额外 `chunk`。

#### optimization.emitOnErrors

使用 optimization.emitOnErrors 在编译时每当有错误时，就会 emit asset。这样可以确保出错的 asset 被 emit 出来。关键错误会被 emit 到生成的代码中，并会在运行时报错。

#### optimization.moduleIds

告知 webpack 当选择模块 id 时需要使用哪种算法。将 optimization.moduleIds 设置为 false 会告知 webpack 没有任何内置的算法会被使用，但自定义的算法会由插件提供。optimization.moduleIds 的默认值是 false。

#### optimization.nodeEnv

告知 webpack 将 process.env.NODE_ENV 设置为一个给定字符串。


## Loader

Loader：模块转换器，用于把模块原内容按照需求转换成新内容。

## Plugin

Plugin：扩展插件，在 Webpack 构建流程中的特定时机注入扩展逻辑来改变构建结果或做你想要的事情。

插件的用处，对开发者来说就是可以接触到 `webpack` 构建流程中的各个阶段并劫持做一些代码处理，对使用者来说则是我们可以通过各类插件实现自动生成 HTML 模板（html-webpack-plugin）、自动压缩图片（imagemin-webpack-plugin）等功能。

插件向第三方开发者提供了 `webpack` 引擎中完整的能力。使用阶段式的构建回调，开发者可以引入它们自己的行为到 `webpack` 构建流程中。

一个 `webpack` 插件应该包含：

- 一个 `Javascript` 函数或 `Javascript` 类，用于承接这个插件模块的所有逻辑；
- 在它原型上定义的 `apply` 方法，会在安装插件时被调用，并被 `webpack compiler` 调用一次；
- 指定一个触及到 `webpack` 本身的事件钩子，即 `hooks`，用于特定时机处理额外的逻辑；
- 对 `webpack` 实例内部做一些操作处理；
- 在功能流程完成后可以调用 `webpack` 提供的回调函数；

在 `apply` 函数中，`webpack` 会传入一个 `compiler` 参数，这个对象包含了 `webpack` 环境所有的配置信息，包含 `options`、`loaders`、`plugins` 这些信息。这个对象在 `webpack` 启动时被实例化，它是全局为的，可以简单地把它理解为 `webpack` 实例。

为了在指定生命周期做自定义的一些逻辑处理，我们需要在 compiler 暴露的钩子上指明我们的 tap 配置，一般这由一个字符串命名和一个回调函数组成。一般来说，compile 过程中会触发如下几个钩子：

1. beforeRun
2. run
3. beforeCompile
4. compile
5. make
6. seal

假设我们想在 compiler.run() 之前处理逻辑，那么就要调用 beforeRun 钩子来处理：

```js
compiler.hooks.beforeRun.tap('testPlugin', (comp) => {
  // ...
});
```

而钩子 entryOption 表示在 webpack 选项中的 entry 配置项处理过之后，执行该插件，钩子 compilation 表示在编译创建之后，执行插件，更详细的 compiler 钩子列表可参见官方文档。

说完 complier 我们再来看看 compilation。compilation 对象包含了当前的模块资源、编译生成资源、变化的文件等。当 webpack 以开发模式运行时，每当检测到一个文件变化，一次新的 compilation 将被创建。compilation 对象也提供了很多事件回调供插件做扩展。通过 compilation 也能读取到 compiler 对象。两者的区别在于，前者代表了整个 webpack 从启动到关闭的生命周期，而 compilation 只代表一次单独的编译。

同样的，compilation 也对应有不同的钩子给开发者调用，具体可参见官方文档。

不论是 compiler 还是 compilation 阶段，从上述举例的几个事件钩子中都可以看出，貌似是存在不同的类型。所以最后，我们再来看看这一块。

> 根据插件所能触及到的 event hook(事件钩子)，对其进行分类。每个 event hook 都被预先定义为 synchronous hook(同步), asynchronous hook(异步), waterfall hook(瀑布), parallel hook(并行)，而在 webpack 内部会使用 call/callAsync 方法调用这些 hook。 —— webpack 中文文档

其中同步钩子有以下几种，你在查询文档的时候可以在钩子名称后面找到对应的类型：

- SyncHook(同步钩子) - SyncHook
- Bail Hooks(保释钩子) - SyncBailHook
- Waterfall Hooks(瀑布钩子) - SyncWaterfallHook

异步钩子如下：

- Async Series Hook(异步串行钩子) - AsyncSeriesHook
- Async waterfall(异步瀑布钩子) - AsyncWaterfallHook
- Async Series Bail - AsyncSeriesBailHook
- Async Parallel - AsyncParallelHook
- Async Series Bail - AsyncSeriesBailHook

如果你不进一步追究，那么按照如下所示的方式对不同钩子进行 tap 处理即可，其中 tap 方法用于同步处理，异步方式则可以调用 tapAsync 方法或 tapPromise 方法。

## splitChunks

首先搞明白： webpack里的代码分割是个什么鬼？ 它允许你将一个文件分割成多个文件。如果使用的好，它能大幅提升你的应用的性能。其原因是基于浏览器会缓存你的代码这一事实。每当你对某一文件做点改变，访问你站点的人们就要重新下载它。然而依赖却很少变动。如果你将（这些依赖）分离成单独的文件，访问者就无需多次重复下载它们了。

## compilation

Compilation 在解析过程中，会将解析后的模块记录在 modules 属性中，那么每一个模块实例又是什么呢？

首先我们先回顾一下最开始的类图，我们会发现跟模块相关的类非常多，看起来类之间的关系也十分复杂，但其实只要记住下面的公式就很好理解：

这个公式的解读是： 一个依赖对象（Dependency）经过对应的工厂对象（Factory）创建之后，就能够生成对应的模块实例（Module）。

### 首先什么是 Dependency？

我个人的理解是，还未被解析成模块实例的依赖对象。比如我们运行 webpack 时传入的入口模块，或者一个模块依赖的其他模块，都会先生成一个 Dependency 对象。作为基类的 Dependency 十分简单，内部只有一个 module 属性来记录最终生成的模块实例。但是它的派生类非常多，webpack 中有单独的文件夹（webpack/lib/dependencies）来存放所有的派生类，这里的每一个派生类都对应着一种依赖的场景。比如从 CommonJS 中require一个模块，那么会先生成 CommonJSRequireDependency。

### 有了 Dependency 之后，如何找到对应的工厂对象呢？

Dependecy 的每一个派生类在使用前，都会先确定对应的工厂对象，比如 SingleEntryDependency 对应的工厂对象是 NormalModuleFactory。这些信息全部是记录在 Compilation 对象的 dependencyFactories 属性中，这个属性是 ES6 中的 Map 对象。直接看下面的代码可能更容易理解：

```js
// https://github.com/webpack/webpack/blob/master/lib/SingleEntryPlugin.js
class SingleEntryPlugin {
    apply(compiler) {
        compiler.plugin("compilation", (compilation, params) => {
            const normalModuleFactory = params.normalModuleFactory;
            // 这里记录了 SingleEntryDependency 对应的工厂对象是 NormalModuleFactory
            compilation.dependencyFactories.set(SingleEntryDependency, normalModuleFactory);
        });
        compiler.plugin("make", (compilation, callback) => {
            // 入口的模块会先变成一个 Dependency 对象
            const dep = SingleEntryPlugin.createDependency(this.entry, this.name);
            compilation.addEntry(this.context, dep, this.name, callback);
        });
    }
}
// https://github.com/webpack/webpack/blob/master/lib/Compilation.js
class Compilation extends Tapable {
    _addModuleChain(context, dependency, onModule, callback) {
        // 其他代码..
        // 开始构建时，通过 Compilation 的 dependenciesFactories 属性找到对应的工厂对象
        const moduleFactory = this.dependencyFactories.get(dependency.constructor);
        if(!moduleFactory) {
            throw new Error(`No dependency factory available for this dependency type: ${dependency.constructor.name}`);
        }
        this.semaphore.acquire(() => {
            // 调用工厂对象的 create 方法，dependency作为参数传入，最终生成模块实例
            moduleFactory.create({
                contextInfo: {
                    issuer: "",
                    compiler: this.compiler.name
                },
                context: context,
                dependencies: [dependency] // 作为参数传入
            }, (err, module) => {
                // module就是生成的模块实例
                // 其他代码..
            })
        })
    }
}
```

一种工厂对象只会生成一种模块，所以不同的模块实例都会有不同的工厂对象来生成。模块的生成过程我们在第一篇文章有讨论过，无非就是解析模块的 request，loaders等信息然后实例化。

## webpack 文件编译流程

webpack 编译流程（文件） -> plugin -> dependency -> module -> chunk -> template -> source code


### 模块对象有哪些特性呢？

同样在第一篇文章中，我们知道一个模块在实例化之后并不意味着构建就结束了，它还有一个内部构建的过程。所有的模块实例都有一个 build 方法，这个方法的作用是开始加载模块源码（并应用loaders），并且通过 js 解析器来完成依赖解析。这里要两个点要注意：

  - 模块源码最终是保存在 _source 属性中，可以通过 _source.source() 来得到。注意在 build 之前 _source 是不存在的。
  - js 解析器解析之后会记录所有的模块依赖，这些依赖其实会分为三种，分别记录在 variables，dependencies， blocks属性。模块构建之后的递归构建过程，其实就是读取这三个属性来重复上面的过程：依赖 => 工厂 => 模块

我们再来看看这些模块类，从前面的类图看，它们是继承于 Module 类。这个类实际上才是我们平常用来跟 chunk 打交道的类对象，它内部有 _chunks 属性来记录后续所在的 chunk 信息，并且提供了很多相关的方法来操作这个对象：addChunk，removeChunk，isInChunk，mapChunks等。后面我们也会看到，Chunk 类与之对应。

Module 类往上还会继承于 DependenciesBlock，这个是所有模块的基类，它包含了处理依赖所需要的属性和方法。上面所说的 variables，dependencies，blocks 也是这个基类拥有的三个属性。它们分别是：

  - variables 对应需要对应的外部变量，比如 __filename，__dirname，process 等node环境下特有的变量
  - dependencies 对应需要解析的其他普通模块，比如 require("./a") 中的 a 模块会先生成一个 CommonJSRequireDependency
  - blocks 对应需要解析的代码块（最终会对应成一个 chunk），比如 require.ensure("./b")，这里的 b 会生成一个 DependenciesBlock 对象

## Chunk

讨论完 webpack 的模块之后，下面需要说明的是 Chunk 对象。关于 chunk 的生成，在第一篇文章中有涉及，这里不再赘述。

chunk 只有一个相关类，而且并不复杂。Chunk 类内部的主要属性是 _modules，用来记录包含的所有模块对象，并且提供了很多方法来操作：addModule，removeModule，mapModules 等。

另外有几个方法可能比较实用，这里也列出来：

  - integrate 用来合并其他chunk
  - split 用来生成新的子 chunk
  - hasRuntime 判断是否是入口 chunk

## Template

Compilation 实例在生成最终文件时，需要将所有的 chunk 渲染（生成代码）出来，这个时候需要用到下面几个属性：

  - mainTemplate 对应 MainTemplate 类，用来渲染入口 chunk
  - chunkTemplate 对应 ChunkTemplate 类，用来传染非入口 chunk
  - moduleTemplate 对应 ModuleTemplate，用来渲染 chunk 中的模块
  - dependencyTemplates 记录每一个依赖类对应的模板

首先 chunk 的渲染入口是 mainTemplate 和 chunkTemplate 的 render 方法。根据 chunk 是否是入口 chunk 来区分使用哪一个：

```js
// https://github.com/webpack/webpack/blob/master/lib/Compilation.js
if(chunk.hasRuntime()) { // 入口chunk
    source = this.mainTemplate.render(this.hash, chunk, this.moduleTemplate, this.dependencyTemplates);
} else {
    source = this.chunkTemplate.render(chunk, this.moduleTemplate, this.dependencyTemplates);
}
```

两个类的 render 方法将生成不同的”包装代码”，MainTemplate 对应的入口 chunk 需要带有 webpack 的启动代码，所以会有一些函数的声明和启动。

这两个类都只负责这些”包装代码”的生成，包装代码中间的每个模块代码，是通过调用 renderChunkModules 方法来生成的。这里的 renderChunkModules 是由他们的基类 Template 类提供，方法会遍历 chunk 中的模块，然后使用 ModuleTemplate 来渲染。

```js
// https://github.com/webpack/webpack/blob/master/lib/MainTemplate.js
// MainTemplate的部分render方法：
const source = new ConcatSource();
source.add("/******/ (function(modules) { // webpackBootstrap\n");
source.add(new PrefixSource("/******/", bootstrapSource));
source.add("/******/ })\n");
source.add("/************************************************************************/\n");
source.add("/******/ (");
// 调用 renderChunkModules 方法，遍历这个 chunk 的所有模块生成对应的代码
const modules = this.renderChunkModules(chunk, moduleTemplate, dependencyTemplates, "/******/ ");
source.add(this.applyPluginsWaterfall("modules", modules, chunk, hash, moduleTemplate, dependencyTemplates));
source.add(")");
// https://github.com/webpack/webpack/blob/master/lib/Template.js
module.exports = class Template extends Tapable {
    // 其他代码..
    renderChunkModules(chunk, moduleTemplate, dependencyTemplates, prefix) {
        // 其他代码..
        var allModules = chunk.mapModules(function(module) {
            return {
                id: module.id,
                // 调用 moduleTemplate.render 方法
                source: moduleTemplate.render(module, dependencyTemplates, chunk)
            };
        });
        // 其他代码..
    }
}
```

ModuleTemplate 做的事情跟 MainTemplate 类似，它同样只是生成”包装代码”来封装真正的模块代码，而真正的模块代码，是通过模块实例的 source 方法来提供。该方法会先读取 _source 属性，即模块内部构建时应用loaders之后生成的代码，然后使用 dependencyTemplates 来更新模块源码。
