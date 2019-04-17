---
title: "Webpack使用小结"
date: 2016-11-20
type: "notes"
draft: false
---

分而治之是软件工程领域的重要思想，对于复杂度日益增加的前端也同样适用。一般前端团队选择合适的框架之后就要开始考虑开发维护的效率问题。而模块化是目前前端领域比较流行的分而治之手段。
Javascript模块化已经有很多规范和工具，例如CommonJS/AMD/requireJS/CMD/ES6 Module，在上篇文章中有详细的介绍。CSS模块化基本要依靠Less, Sass以及Stylus等于处理器的import/minxin特性实现。而HTML以及HTML模版和其他资源比如图片的模块化怎么去处理呢？
这也正是webpack要解决的问题之一。严格来说，webpack是一个模块打包工具（module bundler），它既不像requireJS和seaJS这样的模块加载器，也不像grunt和gulp这样优化前端开发流程的构建工具，像是两类工具的集合却又远不止如此。

Webpack是一个模块打包工具，它将JS、CSS、HTML以及图片等都视为模块资源，这些模块资源必然存在某种依赖关系，webpack就是通过静态分析各种模块文件之间的依赖关系，通过不同种类的Loader将所有模块打包成起来。

![](https://webpack.github.io/assets/what-is-webpack.png)

### webpack VS Gulp

严格来说，Gulp与webpack并没有可比性。Gulp应该和Grunt属于同一类工具，能够优化前端工作流程，比如压缩合并JS、CSS ，预编译Typescript、Sass等。也就是说，我们可以根据需要配置插件，就可以将之前需要手动完成的任务自动化。webpack作为模块打包工具，可以和browserify相提并论。两者都是预编译模块化解决方案。相比requireJS、seaJS这类‘在线’模块化方案更加智能。因为是‘预编译’，不需要在浏览器中加载解释器。另外，你可以直接在本地编写JS，不管是 AMD / CMD / ES6 风格的模块化，都编译成浏览器认识的JS。

总之，Gulp只是一个流程构建工具，而webpack、browserify等是模块化解决方案。Gulp也可以配置seaJS、requireJS甚至webpack的插件。

### 避免多个配置文件

刚开始接触webpack的时候，不管是去浏览[GitHub](https://github.com/)上面star数较多的webpack项目，还是搜索[stack overflow](http://stackoverflow.com/)上面赞成数较多的回答，发现很多人提倡在一个项目中针对开发和产品发布提供不同的配置文件，比如`webpack.dev.config.js`和`webpack.prod.config.js`。看起来很清晰，也可以让新手迅速上手老项目，但仔细研究就会发现，不通环境的配置文件大部分配置项基本相同。这与工程领域一直提倡的DRY（Don't Repeat Yourself）原则相悖，于是就产生了另外一种做法，先生成一个common的`webpack.common.config.js`，然后再针对不同的环境去继承（其实就是require）common的配置文件。但是不管怎样，其实都是生成多个不同的配置文件。如果换个角度想想，这些配置文件虽然不同，但都遵循着node的逻辑，所以完全可以只维护一个配置文件，然后针对不同的环境传入不同的参数。如果你使用npm，则完全可以在package.json文件中这样写：
```javascript
"scripts": {
	"devs": "cross-env DEV=1 webpack-dev-server --hot --inline",
	"build": "cross-env PROD=1 rm -rf ./build && webpack --p"
}
```
其中[cross-env](https://github.com/kentcdodds/cross-env)是个跨平台的环境变量设置工具，可以允许Unix风格环境变量设置通用在window平台。
这样只维护一个webpack.config.js配置文件，然后在配置文件中处理自定义的参数。怎么处理自定义参数呢？这里我们使用webpack自带插件definePlugin提供魔力变量（magic globals）来处理：
```javascript
plugins: [
	new webpack.DefinePlugin ({
		__DEV__: JSON.stringify(JSON.parse(process.env.DEV || 'false')),
		__PROD__: JSON.stringify(JSON.parse(process.env.PROD || 'false'))
	})
]
```
然后在配置文件的其他地方就可以根据设定的环境变量更有针对性地配置不同插件等。甚至在业务逻辑中也可以这样针对不同环境做针对性地调试，比如在开发环境下可以AJAX可以调试本地mock数据，然后在发布的时候，可以正常访问服务端数据。
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

### 定位webpack打包性能

如何快速定位webpack打包速度慢的原因呢？webpack提供了方便的命令行工具让我们可以在命令行输出中看到每个打包步骤的耗时，并且对于耗时较长的步骤用特殊颜色标记，甚至配置显示或隐藏部分模块的打包输出。
下面介绍webpack命令行工具的三个参数：
* `colors` 输出的结果带上颜色，红色代表耗时较长的步骤
* `profile` 输出每个步骤的耗时
* `display-modules` 输出默认情况下隐藏的模块，默认在`["node_modules", "bower_components", "jam", "components"]`中的模块将不会显示

这样命令行输出就会包含对我们定位打包性能非常有用的信息，包括每个步骤打包耗时。
```
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

### 善用RESOLVE

先从解析模块路径和分析依赖讲起。但当项目应用依赖的模块越来越多，越来越重时，项目越来越大，文件和文件夹越来越多时，这个过程就变得越来越关乎性能。

webpack配置中`resolve`有个`alias`的配置项，作用是把对某个模块的依赖重定向到另一个路径。我们接下来看看怎么巧妙使用`resolve.alias`来提高编码效率的同时优化webpack打包性能：
```javascript
resolve: {
	alias: {
		moment: "moment/min/moment-with-locales.min.js"
	}
}
```
这样写的话我们就不用在每个引用`moment`的JS文件中显示书写require或者import后面一长串的路径，只需要简单在JS文件中require('monent')，其实就等价于require('moment/min/moment-with-locales.min.js')，这样既减少我们处理路径时不小心引起的错误，而且代码也更简洁。更重要的是，如果我们直接require('monent')，而没有在webpack中配置别名（alias），webpack就会从node_modules下面找所有`moment`相关的包并且打包，包括源码和烟锁后的代码，这样明显拖慢打包速度。

webpack默认会去寻找所有`resolve.root`下的模块以及模块的依赖，但是有些目录我们很确定它没有新的依赖，可以明确告知webpack不要扫描这个文件的依赖，从而减轻webpack的工作量。这时会用到`module.noParse`配置项。
```javascript
module: {
	noParse: [/moment-with-locales/]
}
```
这样，因为我们在resolve.alias中配置了`moment`重定向到`moment/min/moment-with-locales.min.js`，`module.noParse`配置中的`/moment-with-locales/`生效，所以webpack就直接把依赖打包。

### Babel Loader + ES6

webpack是个非常好的工具，和NPM搭配起来使用管理模块实在非常方便。好马配好鞍，像Babel这样神级的工具让我们在这个浏览器尚未全面普及 ES6 语法的时代可以先一步体验到新的语法带来的便利和效率上的提升。Babel团队也为我们提供了和webpack集成的babel-loader，这样我们只要在webpack中简单配置就可以体验ES6的新特性：
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

这样配置当然也没问题，但是对于很多的第三方包来说，完全没有经过babel-loader的必要（成熟的3rd-party包会在发布前es5化以兼容老版本的浏览器），所以让这些包经过babel-loader的处理无疑会带来巨大的性能负担，毕竟babel6要经过几十个插件的处理。这里我们可以在配置loader的时候使用exclude，去除不需要babel-loader处理的第三方包，从而使整包的构建效率飞速提高。
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

在CSS中或者JS代码中，都会涉及到require图片资源的情况，webpack可以内联图片地址到打包JS中并且通过require()返回图片路径。其实，不只是图片，还有css中用到的iconfont以及flash等，都可以相似处理。这里需要用到url-loader或file-loader。
> file-loader: 将匹配到的文件复制到输出文件夹，并根据`output.publicPath`的设置返回文件路径
> url-loader: 类似file-loader，但是它可以返回一个DataUrl(base64)如果文件小于设置的限制值limit
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
通过向url-loader传递参数，如果图片小于8kb，则使用base64内联在CSS或者JS文件中，大于8kb，则通过`output.publishPath`配置的前缀将图片路径写入代码，并提取图片到输出目录。

### webpack、Angular和jquery之谜

很多人都知道Angular其实封装了一个微型版本的jQuery叫[jQLite](https://docs.angularjs.org/api/ng/function/angular.element)，但是还是可以使用`jQuery`的，尤其是当使用到一些第三方库依赖于`jQuery`的时候。
> To use jQuery, simply ensure it is loaded before the angular.js.

Angular官方Doc的这句话其实是在我们未使用任何模块加载／打包工具的前提下，也就是说所有的JS文件是通过`script`标签引入到全局。确实如此，如果去看看Angular的源码就会发现：
```javascript
// bind to jQuery if present;
var jqName = jq();
jQuery = isUndefined(jqName) ? window.jQuery :    // use jQuery (if present)
	       !jqName ? undefined :    // use jqLite
	       window[jqName];    // use jQuery specified by `ngJq`
```

但是如果使用了webpack情况就稍微有所不同，毕竟webpack倡导模块化，默认是不允许讲模块暴露给全局的。那么这种情况下怎样让Angular使用jQuery呢，可能大部分人都和我一开始的想法一样：
```javascript
import "jquery";
import * as angular from "angular";
```
然并卵。
webpack提供了一种比较高效的方法，使用webpack内置的`ProvidePlugin`插件，只需要在webpack.config.js中简单配置：
```javascript
plugins: [
	new webpack.ProvidePlugin ({
		$: "jquery",
		jQuery: "jquery",
		"window.jQuery": "jquery"
	})
]
```
providePlugin其实是在处理模块的过程中替换字符串和变量。去看一下webpack打包后的code你就会发现真相：
```javascript
// bind to jQuery if present;
var jqName = jq();
jQuery = isUndefined(jqName) ? __webpack_provided_window_dot_jQuery :    // use jQuery (if present)
	       !jqName ? undefined :    // use jqLite
	       window[jqName];    // use jQuery specified by `ngJq`
```
这样，即使当webpack碰到require的第三方库中出现全局的`$`、`jQeury`和`window.jQuery`时，就会使用node_module下`jQuery`库了。

### 合并公公代码

当webpack打包的项目中有多个入口文件，而这些文件都require或者import了相同的模块，如果你不做任何事情，webpack会为每个入口文件引入一份相同的模块。当相同模块变化时，所有引入的entry都需要一次rebuild，造成了性能的损耗。`CommonsChunkPlugin`可以将相同的模块提取出来单独打包，进而减小rebuild时的性能消耗。
比如在entry中定义`app`和`vendor`两个模块入口，前者是我们自己编写的代码合并压缩而成，后者是使用的第三方库合并压缩后的结果：
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

然后在plugin部分使用`CommonsChunkPlugin`将公共部分提取出来：
```javascript
new webpack.optimize.CommonsChunkPlugin('vendor', __PROD__ ? 'vendor.[hash].js' : 'vendor.js')
```
这样，所有模块中只要有require到vendor数组中定义的这些第三方模块，那么这些第三方模块都会被统一提取出来，放入vendor.js中去。在插件的配置中我们还进行了判断，如果是生产环境则给最终生成的文件名加hash。

### 提取单独的样式文件

在处理CSS时webpack需要两个不同的loader：css-loader和style-loader，前者负责将CSS文件变成文本返回，并处理其中的url()和@import()，而后者将CSS以style标签的形式插入到页面中去。如果用到其他的预编译样式语言（如Less，Sass以及Stylus），还需要在css-loader处理之前经过对应loader的预编译，一般使用的时候这么写：
```javascript
{
	test: /\.scss$/,
	loader: 'style!css?sourceMap!sass?sourceMap'
}
```
这样配置的loader处理所有的scss文件，先用sass-loader将Sass变成CSS，然后使用css-loader和style-loader。但这样的问题就是所有CSS文件都是直接以style标签的形式插入页面，不利于缓存。如果我们想单独的.css文件，就需要webpack提供的插件`extract-text-webpack-plugin`将css从js代码中抽出并合并。在loader中这样定义：
```javascript
{
	test: /\.scss$/,
	loader: ExtractTextPlugin.extract('style', 'css?sourceMap!sass?sourceMap')
}
```

然后再plugin部分加入`extract-text-webpack-plugin`的配置项：
```javascript
var ExtractTextPlugin = require("extract-text-webpack-plugin");
// ...
new ExtractTextPlugin(__PROD__ ? '[name].[hash].css' : '[name].css')
```

### LazyLoad

使用webpack的[Code Splitting](http://webpack.github.io/docs/code-splitting.html)可以方便地实现LazyLoad。对于大型的应用来说，一次性将所有文件全部下载进浏览器显得很不划算，因为有些功能可能不太常用，不需要一开始就加载。Code Splitting功能可以在代码中定义“分割点”，即代码执行到这些点时才会去加载所需的模块，这样就实现了按需加载的LazyLoad。
那么，“分割点”设在哪？一个页面模块往往包括HTML、CSS、JS三种资源文件，怎么去加载这三种资源？
对于“分割点”的设置很简单，由于是在路由到某个页面时再去加载所需模块，所以当然把“分割点”放在路由的定义中。放在路由中还可以帮我们解决三种资源加载的问题，因为路由中通常需要定义路由到这个页面时的template和controller，这恰恰是刚才提到的HTML和JS资源，至于CSS资源，完全可以作为HTML和JS资源的依赖。

#### 动态加载模板

在定义路由的时候在template中去动态加载所需的模板，但是template和templateUrl参数是个string，显然不能满足我们的需求。我们自然可以想到使用ui-router的`templateProvider`参数了，它的值可以是一个函数，然后在这个函数中支持返回一个promise，由这个promise最终来返回HTML字符串。
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
这样只有在访问/preferences这个路由时再去加载`preferences.partial.html`这个template。来看看templateProvider的配置，首先它的函数返回一个promise（第5行），这个promise直接使用$q的构造函数，这个构造函数接受两个参数：`resolve`和`reject`，分别用来resolve和reject这个promise。然后在这个promise中我们要resolve的就是`preferences.partial.html`这个template的字符串内容。那么什么时候去resolve呢？这里我们重点来看：webpack的`require.ensure`函数，就是使用函数来实现“Code Splitting”的，它有三个参数：
1. 依赖的模块数组：模块名组成的数组，webpack会先加载这些模块再执行后面的回调函数。
2. 回调函数：在模块数组加载完以后才会执行这个。
3. chunk的名字：从这个“分割点”分割出去的模块会放入额外的模块，这个参数指定模块的名字。这个参数主要用于存在多个`require.ensure`的情况，可以将多个“分割点”分割出去的代码放入同一个模块。

#### 动态加载所需JS

接下来说说如何动态加载JS资源。这里的JS资源其实就是包含controller的JS文件。那么controller的JS模块怎么动态加载呢？每个路由定义都可以有一个`resolve`，只有promise被`resolve`了才会真正进入到那个路由中去，所以这正是动态加载JS和其他模块依赖的地方：
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
这个resolve取名loadPreferencesModule，因为它的作用就是加载preferences模块以及它所依赖的其它模块，并不需要我们真正的去return任何东西，所以第8行直接resolve了空值。另外注意第7行同样使用了preferences作为chunk名，这保证了这个“分割点”分割出去的代码和上面分割出去的template是在一起的。
要知道[ocLazyLoad](https://github.com/ocombe/ocLazyLoad)是一个适用于Angular的Lazyload的库，用它就可以实现Angular的LazyLoad。但它的作用并不是动态加载文件，而是使webpack动态加载进来的文件中的Angular的module名生效，因为Angular是不允许动态去声明一个module的。

至于如何加载CSS资源了，其实CSS只要用使用css-loader放在JS中或HTML中通过require(xxx.css)加载进来，因为通常CSS是与某个页面或某个模块绑定的。
