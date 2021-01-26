---
title: "Webpack 使用小结"
date: 2016-11-20
categories: ['note', 'tech', 'thought']
draft: false
---

分而治之是软件工程领域的重要思想，对于复杂度日益增加的前端也同样适用。一般前端团队选择合适的框架之后就要开始考虑开发维护的效率问题。而模块化是目前前端领域比较流行的分而治之手段。

javascript 模块化已经有很多规范和工具，例如 `CommonJS/AMD/requireJS/CMD/ES6 Module` ，在上篇文章中有详细的介绍。 CSS 模块化基本要依靠 `Less` , `Sass` 以及 `Stylus` 等于处理器的 `import/minxin` 特性实现。而 html 以及 html 模版和其他资源比如图片的模块化怎么去处理呢？

这也正是 webpack 要解决的问题之一。严格来说， webpack 是一个模块打包工具，它既不像 requireJS 和 seaJS 这样的模块加载器，也不像 grunt 和 gulp 这样优化前端开发流程的构建工具，像是两类工具的集合却又远不止如此。

总的来说， Webpack 是一个模块打包工具，它将 js 、 css 、 html 以及图片等都视为模块资源，这些模块资源必然存在某种依赖关系， webpack 就是通过静态分析各种模块文件之间的依赖关系，通过不同种类的 Loader 将所有模块打包成起来。

![](https://webpack.github.io/assets/what-is-webpack.png)

## webpack VS gulp

严格来说， gulp 与 webpack 并没有可比性。 gulp 应该和 grunt 属于同一类工具，能够优化前端工作流程，比如压缩合并 js 、css ，预编译 typescript 、 sass 等。也就是说，我们可以根据需要配置插件，就可以将之前需要手动完成的任务自动化。 webpack 作为模块打包工具，可以和 browserify 相提并论，两者都是预编译模块化解决方案。相比 requireJS 、 seaJS 这类“在线”模块化方案更加智能。因为是“预编译”，所以不需要在浏览器中加载解释器。另外，你可以直接在本地编写 js ，不管是 AMD/CMD/ES6 风格的模块化，都编译成浏览器认识的 js 。

总之， gulp 只是一个流程构建工具，而 webpack 、 browserify 等是模块化解决方案， gulp 也可以配置 seaJS 、 requireJS 甚至 webpack 的插件。

## 避免多个配置文件

刚开始接触 webpack 的时候，不管是去浏览 GitHub 上面 star 数较多的 webpack 项目，还是搜索 stack overflow 上面赞成数较多的回答，发现很多人提倡在一个项目中针对开发和产品发布提供不同的配置文件，比如 `webpack.dev.config.js` 和 `webpack.prod.config.js` 。这样看起来很清晰，也可以让新手迅速上手老项目，但仔细研究就会发现，不通环境的配置文件大部分配置项基本相同。这与工程领域一直提倡的 DRY(Don't Repeat Yourself) 原则相悖，于是就产生了另外一种做法，先生成一个通用的 `webpack.common.config.js` ，然后再针对不同的环境去继承（其实就是 `require`）通用配置文件。但是不管怎样，其实都是生成多个不同的配置文件。如果换个角度想想，这些配置文件虽然不同，但都遵循着 nodejs 的逻辑，所以完全可以只维护一个配置文件，然后针对不同的环境传入不同的参数。如果你使用 npm ，则完全可以在 `package.json` 文件中这样写：

```javascript
"scripts": {
	"devs": "cross-env DEV=1 webpack-dev-server --hot --inline",
	"build": "cross-env PROD=1 rm -rf ./build && webpack --p"
}
```

其中 [cross-env](https://github.com/kentcdodds/cross-env) 是个跨平台的环境变量设置工具，可以允许 unix 风格环境变量设置通用在 window 平台。

这样只维护一个 `webpack.config.js` 配置文件，然后在配置文件中处理自定义的参数。怎么处理自定义参数呢？这里我们使用 webpack 自带插件 `definePlugin` 提供魔力变量（magic globals）来处理：

```javascript
plugins: [
	new webpack.DefinePlugin ({
		__DEV__: JSON.stringify(JSON.parse(process.env.DEV || 'false')),
		__PROD__: JSON.stringify(JSON.parse(process.env.PROD || 'false'))
	})
]
```

然后在配置文件的其他地方就可以根据设定的环境变量更有针对性地配置不同插件等。甚至在业务逻辑中也可以这样针对不同环境做针对性地调试，比如在开发环境下可以 ajax 可以调试本地 mock 数据，然后在发布的时候，可以正常访问服务端数据。

```javascript
if (__DEV__) {
	// code for dev
	//...
}

if (__PROD__) {
	// code for production
	//...
}
```

## 定位 webpack 打包性能

如何快速定位 webpack 打包速度慢的原因呢？ webpack 提供了方便的命令行工具让我们可以在命令行输出中看到每个打包步骤的耗时，并且对于耗时较长的步骤用特殊颜色标记，甚至配置显示或隐藏部分模块的打包输出。
下面介绍 webpack 命令行工具的三个参数：

* `colors` 输出的结果带上颜色，红色代表耗时较长的步骤
* `profile` 输出每个步骤的耗时
* `display-modules` 输出默认情况下隐藏的模块，默认在 `["node_modules", "bower_components", "jam", "components"]` 中的模块将不会显示

这样命令行输出就会包含对我们定位打包性能非常有用的信息，包括每个步骤打包耗时。

```bash
Hash: 0818c9f97693e1a7706f
Version: webpack 1.13.1
Time: 516ms
    Asset    Size  Chunks             Chunk Names
bundle.js  454 kB       0  [emitted]  null
   [0] ./index.js 89 bytes {0} [built]
       factory:9ms building:11ms = 20ms
   [1] ./~/moment/moment.js 123 kB {0} [built]
       [0] 20ms -> factory:5ms building:168ms = 193ms
   [2] (webpack)/buildin/module.js 251 bytes {0} [built]
       [0] 20ms -> [1] 173ms -> factory:111ms building:154ms = 458ms
   [3] ./~/moment/locale ^\.\/.*$ 2.63 kB {0} [optional] [built]
       [0] 20ms -> [1] 173ms -> factory:3ms building:9ms dependencies:95ms = 300ms
   [4] ./~/moment/locale/af.js 2.39 kB {0} [optional] [built]
       [0] 20ms -> [1] 173ms -> [3] 12ms -> factory:52ms building:50ms dependencies:148ms = 455ms
```

### 善用 resolve

先从解析模块路径和分析依赖讲起。但当项目应用依赖的模块越来越多，越来越重时，项目越来越大，文件和文件夹越来越多时，这个过程就变得越来越关乎性能。

webpack 配置中 `resolve` 有个 `alias` 的配置项，作用是把对某个模块的依赖重定向到另一个路径。我们接下来看看怎么巧妙使用 `resolve.alias` 来提高编码效率的同时优化 webpack 打包性能：

```javascript
resolve: {
	alias: {
		moment: "moment/min/moment-with-locales.min.js"
	}
}
```

这样写的话我们就不用在每个引用 `moment` 的 js 文件中显示书写 `require`或者 `import` 后面一长串的路径，只需要简单在 js 文件中 `require('monent')` ，其实就等价于 `require('moment/min/moment-with-locales.min.js')` ，这样既减少我们处理路径时不小心引起的错误，而且代码也更简洁。更重要的是，如果我们直接 `require('monent')` ，而没有在 webpack 中配置别名， webpack 就会从 node_modules 下面找所有 `moment` 相关的包并且打包，包括源码和烟锁后的代码，这样明显拖慢打包速度。

webpack 默认会去寻找所有 `resolve.root` 下的模块以及模块的依赖，但是有些目录我们很确定它没有新的依赖，可以明确告知 webpack 不要扫描这个文件的依赖，从而减轻 webpack 的工作量。这时会用到 `module.noParse` 配置项。

```javascript
module: {
	noParse: [/moment-with-locales/]
}
```

这样，因为我们在 `resolve.alias` 中配置了 `moment` 重定向到 `moment/min/moment-with-locales.min.js` ， `module.noParse` 配置中的 `/moment-with-locales/` 生效，所以 webpack 就直接把依赖打包。

## babel loader + ES6

webpack 是个非常好的工具，和 npm 搭配起来使用管理模块实在非常方便。好马配好鞍，像 Babel 这样神级的工具让我们在这个浏览器尚未全面普及 ES6 语法的时代可以先一步体验到新的语法带来的便利和效率上的提升。 Babel 团队也为我们提供了和 webpack 集成的 `babel-loader`，这样我们只要在 webpack 中简单配置就可以体验 ES6 的新特性：

```javascript
module: {
	loaders: [
		{
			test: /\.js$/,
			loader: 'babel-loader',
			query: {
				presets: ['es2015', 'stage-1']
			}
		}
	]
}
```

这样配置当然也没问题，但是对于很多的第三方包来说，完全没有经过 `babel-loader`的必要（成熟的第三方包会在发布前 ES5 化以兼容老版本的浏览器），所以让这些包经过 `babel-loader` 的处理无疑会带来巨大的性能负担，毕竟 babel6 要经过几十个插件的处理。这里我们可以在配置 loader 的时候使用 `exclude` ，去除不需要 `babel-loader` 处理的第三方包，从而使整包的构建效率飞速提高。

```javascript
module: {
	loaders: [
		{
			test: /\.js$/,
			loader: 'babel-loader',
			exclude: [/node_modules/, /bower_components/],
			query: {
				presets: ['es2015', 'stage-1']
			}
		}
	]
}
```

### 处理图片、字体等文件

在 css 中或者 js 代码中，都会涉及到 `require` 图片资源的情况， webpack 可以内联图片地址到打包 js 中并且通过 `require()` 返回图片路径。其实，不只是图片，还有 css 中用到的 iconfont 以及 flash 等，都可以相似处理。这里需要用到 `url-loader` 或 `file-loader` 。

- file-loader: 将匹配到的文件复制到输出文件夹，并根据 `output.publicPath` 的设置返回文件路径
- url-loader: 类似 `file-loader`，但是它可以返回一个 DataUrl(base64) 如果文件小于设置的限制值 limit

```javascript
module:{
	loaders:[
		{
			test: /\.(png|jpg|jpeg|gif|ico)$/,
			loader: 'url-loader?limit=8192'    //  <= 8kb inline base64
		},
		{
			test: /\.woff(\?v=\d+\.\d+\.\d+)?$/,
			loader: 'url?limit=10000&minetype=application/font-woff'
		},
		{
			test: /\.woff2(\?v=\d+\.\d+\.\d+)?$/,
			loader: 'url?limit=10&minetype=application/font-woff'
		},
		{
			test: /\.ttf(\?v=\d+\.\d+\.\d+)?$/,
			loader: 'url?limit=10&minetype=application/octet-stream'
		},
		{
			test: /\.eot(\?v=\d+\.\d+\.\d+)?$/,
			loader: 'file'
		},
		{
			test: /\.svg(\?v=\d+\.\d+\.\d+)?$/,
			loader: 'url?limit=10&minetype=image/svg+xml'
		}
	]
}

```
通过向 `url-loader` 传递参数，如果图片小于8kb，则使用 base64 内联在 css 或者 js 文件中，大于8kb，则通过 `output.publishPath` 配置的前缀将图片路径写入代码，并提取图片到输出目录。

### webpack、angular和jquery之谜

很多人都知道 angular 其实封装了一个微型版本的 jQuery 叫 [jQLite](https://docs.angularjs.org/api/ng/function/angular.element) ，但是还是可以使用 `jQuery` 的，尤其是当使用到一些第三方库依赖于 `jQuery` 的时候。

> To use jQuery, simply ensure it is loaded before the angular.js.

angular 官方文档中的这句话其实是在我们未使用任何模块加载/打包工具的前提下，也就是说所有的 js 文件是通过 `script` 标签引入到全局。确实如此，如果去看看 angular 的源码就会发现：

```javascript
// bind to jQuery if present;
var jqName = jq();
jQuery = isUndefined(jqName) ? window.jQuery :    // use jQuery (if present)
	       !jqName ? undefined :    // use jqLite
	       window[jqName];    // use jQuery specified by `ngJq`
```

但是如果使用了 webpack 情况就稍微有所不同，毕竟 webpack 倡导模块化，默认是不允许讲模块暴露给全局的。那么这种情况下怎样让 angular 使用 jQuery 呢，可能大部分人都和我一开始的想法一样：

```javascript
import "jquery";
import * as angular from "angular";
```

然并卵。

webpack 提供了一种比较高效的方法，使用 webpack 内置的 `ProvidePlugin` 插件，只需要在 `webpack.config.js` 中简单配置：

```javascript
plugins: [
	new webpack.ProvidePlugin ({
		$: "jquery",
		jQuery: "jquery",
		"window.jQuery": "jquery"
	})
]
```

`providePlugin` 其实是在处理模块的过程中替换字符串和变量。去看一下 webpack 打包后的代码你就会发现真相：

```javascript
// bind to jQuery if present;
var jqName = jq();
jQuery = isUndefined(jqName) ? __webpack_provided_window_dot_jQuery :    // use jQuery (if present)
	       !jqName ? undefined :    // use jqLite
	       window[jqName];    // use jQuery specified by `ngJq`
```

这样，即使当 webpack 碰到 require 的第三方库中出现全局的 `$` 、 `jQeury` 和 `window.jQuery` 时，就会使用 node_module 下 `jQuery` 库了。

### 合并公公代码

当 webpack 打包的项目中有多个入口文件，而这些文件都 `require` 或者 `import` 了相同的模块，如果你不做任何事情， webpack 会为每个入口文件引入一份相同的模块。当相同模块变化时，所有引入的 entry 都需要一次重新构建打包 ，造成了性能的损耗。 `CommonsChunkPlugin` 可以将相同的模块提取出来单独打包，进而减小重新构建打包时的性能消耗。

比如在 entry 中定义 `app` 和 `vendor` 两个模块入口，前者是我们自己编写的代码合并压缩而成，后者是使用的第三方库合并压缩后的结果：

```javascript
entry: {
	app: [
		'source/app/index.js'
	], 
	vendor: [
		'angular',
		'angular-ui-router',
		'angular-animate'
		// ...
	]
}
```

然后在 `plugin` 部分使用 `CommonsChunkPlugin` 将公共部分提取出来：

```javascript
new webpack.optimize.CommonsChunkPlugin('vendor', __PROD__ ? 'vendor.[hash].js' : 'vendor.js')
```

这样，所有模块中只要有 `require` 到 vendor 数组中定义的这些第三方模块，那么这些第三方模块都会被统一提取出来，放入 `vendor.js` 中去。在插件的配置中我们还进行了判断，如果是生产环境则给最终生成的文件名加hash。

### 提取单独的样式文件

在处理 css 时 webpack 需要两个不同的 loader：`css-loader` 和 `style-loader` ，前者负责将 css 文件变成文本返回，并处理其中的 `url()` 和 `@import()` ，而后者将 css 以 style 标签的形式插入到页面中去。如果用到其他的预编译样式语言（如 less ， sass 以及 stylus ），还需要在 css-loader 处理之前经过对应 loader 的预编译，一般使用的时候这么写：

```javascript
{
	test: /\.scss$/,
	loader: 'style!css?sourceMap!sass?sourceMap'
}
```

这样配置的 loader 处理所有的 scss 文件，先用 sass-loader 将 sass 变成 css ，然后使用 css-loader 和 style-loader 。但这样的问题就是所有 css 文件都是直接以 style 标签的形式插入页面，不利于缓存。如果我们想单独的 css 文件，就需要 webpack 提供的插件 `extract-text-webpack-plugin` 将 css 从 js 代码中抽出并合并。在 loader 中这样定义：

```javascript
{
	test: /\.scss$/,
	loader: ExtractTextPlugin.extract('style', 'css?sourceMap!sass?sourceMap')
}
```

然后再 plugin 部分加入 `extract-text-webpack-plugin` 的配置项：

```javascript
var ExtractTextPlugin = require("extract-text-webpack-plugin");
// ...
new ExtractTextPlugin(__PROD__ ? '[name].[hash].css' : '[name].css')
```

### 懒加载

使用 webpack 的 [Code Splitting](http://webpack.github.io/docs/code-splitting.html) 可以方便地实现懒加载 。对于大型的应用来说，一次性将所有文件全部下载进浏览器显得很不划算，因为有些功能可能不太常用，不需要一开始就加载。 Code Splitting 功能可以在代码中定义“分割点”，即代码执行到这些点时才会去加载所需的模块，这样就实现了按需加载的懒加载。

那么，“分割点”设在哪？一个页面模块往往包括 html 、 css 、 js 三种资源文件，怎么去加载这三种资源？

对于“分割点”的设置很简单，由于是在路由到某个页面时再去加载所需模块，所以当然把“分割点”放在路由的定义中。放在路由中还可以帮我们解决三种资源加载的问题，因为路由中通常需要定义路由到这个页面时的 template 和 controller ，这恰恰是刚才提到的 html 和 js 资源，至于 css 资源，完全可以作为 html 和 js 资源的依赖。

#### 动态加载模板

在定义路由的时候在 template 中去动态加载所需的模板，但是 template 和 templateUrl 参数是个字符串，显然不能满足我们的需求。我们自然可以想到使用 ui-router 的 `templateProvider` 参数了，它的值可以是一个函数，然后在这个函数中支持返回一个 promise ，由这个 promise 最终来返回 html 字符串。

```javascript
$stateProvider
	.state('framework.preferences', {
		url: '/preferences',
		templateProvider: ($q) => {
			return $q((resolve) => {
				require.ensure([], () => resolve(require('./preferences.partial.html')), 'preferences');
		        });
		},
		controller: 'PreferencesController as pc',
	});
```

这样只有在访问 `/preferences` 这个路由时才去加载 `preferences.partial.html` 这个 template 。来看看 templateProvider 的配置，首先它的函数返回一个 promise ，这个 promise 直接使用 `$q` 的构造函数，这个构造函数接受两个参数： `resolve` 和 `reject` ，分别用来 resolve 和 reject 这个 promise 。然后在这个 promise 中我们要 resolve 的就是 `preferences.partial.html` 这个 template 的字符串内容。那么什么时候去 resolve 呢？这里我们重点来看： webpack 的 `require.ensure` 函数，就是使用函数来实现 “Code Splitting” 的，它有三个参数：

1. 依赖的模块数组：模块名组成的数组， webpack 会先加载这些模块再执行后面的回调函数；
2. 回调函数：在模块数组加载完以后才会执行这个；
3. chunk 的名字：从这个“分割点”分割出去的模块会放入额外的模块，这个参数指定模块的名字。这个参数主要用于存在多个 `require.ensure` 的情况，可以将多个“分割点”分割出去的代码放入同一个模块；

#### 动态加载

接下来说说如何动态加载 js 资源。这里的 js 资源其实就是包含 controller 的 js 文件。那么 controller 的 js 模块怎么动态加载呢？每个路由定义都可以有一个 `resolve` ，只有 promise 被 `resolve` 了才会真正进入到那个路由中去，所以这正是动态加载 js 和其他模块依赖的地方：

```javascript
resolve: {
	loadPreferencesModule: ($q, $ocLazyLoad) => {
		return $q((resolve) => {
			require.ensure([], () => {
				// load whole module
				let module = require('./preferences.js');
				$ocLazyLoad.load({name: module.name});
				resolve();
			}, 'preferences');
		});
	}
}
```

这个 resolve 取名 loadPreferencesModule ，因为它的作用就是加载 preferences 模块以及它所依赖的其它模块，并不需要我们真正的去 return 任何东西，所以第8行直接 resolve 了空值。另外注意第7行同样使用了 preferences 作为 chunk 名，这保证了这个“分割点”分割出去的代码和上面分割出去的 template 是在一起的。
要知道 [ocLazyLoad](https://github.com/ocombe/ocLazyLoad) 是一个适用于 angular 的 懒加载库，用它就可以实现 angular 的懒加载。但它的作用并不是动态加载文件，而是使 webpack 动态加载进来的文件中的 angular 的模块名生效，因为 angular 是不允许动态去声明一个模块的。

至于如何加载 css 资源了，其实 css 只要用使用 css-loader 放在 js 中或 html 中通过 `require(xxx.css)` 加载进来，因为通常 css 是与某个页面或某个模块绑定的。
