> 文章首发于我的博客 https://github.com/mcuking/blog/issues/80

在开始分析源码之前，笔者先把之前收集到的 webpack 构建流程图贴在下面。
后面的分析过程读者可以对照着这张图来进行理解。

![image](https://user-images.githubusercontent.com/22924912/72984163-04482780-3e1e-11ea-9b8c-2047a46c5383.png)

### 构建准备阶段

回顾前面的文章，在 webpack-cli 重新调用 webpack 包时，首先执行的就是 node_module\webpack\lib\webpack.js 中的函数。如下：

```js
const webpack = (options, callback) => {
	...
	let compiler;
	if (Array.isArray(options)) {
		compiler = new MultiCompiler(
			Array.from(options).map(options => webpack(options))
		);
	} else if (typeof options === "object") {
		// 检查传入的 options 并设置默认项
		options = new WebpackOptionsDefaulter().process(options);

		// 初始化一个 compiler 对象实例
		compiler = new Compiler(options.context);
		// 将 options 挂在到这个实例对象上
		compiler.options = options;

		// 清理构建的缓存
		new NodeEnvironmentPlugin({
			infrastructureLogging: options.infrastructureLogging
		}).apply(compiler);

		// 遍历 options.plugins 数组，将用户配置的 plugins 全部初始化
		// 并将插件内部业务逻辑绑定到 Compiler 实例对象上，等待实例对象触发对应钩子后执行
		// 请参考上篇分析文章
		if (options.plugins && Array.isArray(options.plugins)) {
			for (const plugin of options.plugins) {
				if (typeof plugin === "function") {
					plugin.call(compiler, compiler);
				} else {
					plugin.apply(compiler);
				}
			}
		}
		compiler.hooks.environment.call();
		compiler.hooks.afterEnvironment.call();

		// 根据 options 中配置的参数，实例化内部插件并绑定到 compiler 实例对象上
		// 例如 externals 有配置，则需要配置 ExternalsPlugins
		compiler.options = new WebpackOptionsApply().process(options, compiler);
	} else {
		throw new Error("Invalid argument: options");
	}
	...
        if (callback) {
		...
		compiler.run(callback);
	}
	return compiler;
};
```

其中 WebpackOptionsDefaulter 这个类的作用就是检测 options 并设置默认配置项，其关键代码如下：

```js
class WebpackOptionsDefaulter extends OptionsDefaulter {
	constructor() {
		super();

		this.set("entry", "./src");

		this.set("devtool", "make", options =>
			options.mode === "development" ? "eval" : false
		);
		this.set("cache", "make", options => options.mode === "development");

		this.set("context", process.cwd());
		this.set("target", "web");
		...
	}
}
```

set 方法和 process 方法都是继承自父类 OptionsDefaulter，这里就不赘述了。

接着是初始化了一个 compiler 对象，并将处理好的 options 挂载到实例上。然后开始初始化 options.plugins 上的插件，将插件绑定到 compiler 实例对象上，如果 plugin 是函数，则直接调用；如果是对象，则调用对象上的 apply 方法。这也就为什么 webpack 的插件配置一般都是对象实例数组的原因，如下：

```js
{
  plugins: [new HtmlWebpackPlugin()];
}
```

注：具体插件原理可参考上篇文章

最后调用了一个名为 WebpackOptionsApply 的类，我们看下其实现的部分代码：

```js
class WebpackOptionsApply extends OptionsApply {
	constructor() {
		super();
	}

	process(options, compiler) {
		...
		new EntryOptionPlugin().apply(compiler);
		compiler.hooks.entryOption.call(options.context, options.entry);
		...
		if (options.externals) {
			ExternalsPlugin = require("webpack/lib/ExternalsPlugin");
			new ExternalsPlugin(
				options.output.libraryTarget,
				options.externals
			).apply(compiler);
		}
		...
	}
}
```

从中不难发现，WebpackOptionsApply 主要作用就是根据 options 中的设置，来挂载对应的插件到 compiler 实例对象上，例如如果设置了 externals，则挂载 ExternalsPlugin 插件。需要注意的是，有些插件是默认必须要挂载的，而不由 options 中的设置决定，例如 EntryOptionPlugin 插件。

这里我们正好可以到 EntryOptionPlugin 看下我们在 options 里经常设置的参数 entry 到底支持几种类型，主要代码如下：

```js
const itemToPlugin = (context, item, name) => {
  if (Array.isArray(item)) {
    return new MultiEntryPlugin(context, item, name);
  }
  return new SingleEntryPlugin(context, item, name);
};

module.exports = class EntryOptionPlugin {
  apply(compiler) {
    compiler.hooks.entryOption.tap('EntryOptionPlugin', (context, entry) => {
      if (typeof entry === 'string' || Array.isArray(entry)) {
        itemToPlugin(context, entry, 'main').apply(compiler);
      } else if (typeof entry === 'object') {
        for (const name of Object.keys(entry)) {
          itemToPlugin(context, entry[name], name).apply(compiler);
        }
      } else if (typeof entry === 'function') {
        new DynamicEntryPlugin(context, entry).apply(compiler);
      }
      return true;
    });
  }
};
```

有上面代码我们可以知道 entry 可以是字符串、数组、对象和函数，其中当是数组时，则挂载 MultiEntryPlugin 插件，也就是说 webpack 会将多个文件打包成一个文件。而当是对象时，则遍历每个键值对，然后执行 itemToPlugin 方法，也就是说 webpack 会将对象中的每一项入口对应的文件分别打包成不同的文件，这个就对应到了我们常说的多页面打包场景。

到这里是不是发现当看懂了源码，则会对之前死记硬背的 webpack 配置有了更深入的理解了呢？其实这就是阅读源码的一个非常棒的好处。

接下来则是调用了 compiler 的 run 方法，那么我们就回到 Compiler 文件中，进一步分析 Compiler 中到底做了哪些事情。

### 模块构建和 chunk 生成阶段

下面就是 Compiler 类的关键代码：

```js
class Compiler extends Tapable {
	constructor(context) {
		super();
		this.hooks = {
			// 总共有 26 个钩子，下面列举的是比较常见的
			run: new AsyncSeriesHook(["compiler"]),
			emit: new AsyncSeriesHook(["compilation"]),
			compilation: new SyncHook(["compilation", "params"]),
			compile: new SyncHook(["params"]),
			make: new AsyncParallelHook(["compilation"]),
			...
		},
		...
	}

	watch(watchOptions, handler) {

	}

	run(callback) {
		const onCompiled = (err, compilation) => {
			...
		};

		this.hooks.beforeRun.callAsync(this, err => {
			if (err) return finalCallback(err);

			this.hooks.run.callAsync(this, err => {
				if (err) return finalCallback(err);

				this.readRecords(err => {
					if (err) return finalCallback(err);

					this.compile(onCompiled);
				});
			});
		});
	}

	...

	emitAssets(compilation, callback) {

	}

	...

	createCompilation() {
		return new Compilation(this);
	}

	newCompilation(params) {
		const compilation = this.createCompilation();
		compilation.fileTimestamps = this.fileTimestamps;
		compilation.contextTimestamps = this.contextTimestamps;
		compilation.name = this.name;
		compilation.records = this.records;
		compilation.compilationDependencies = params.compilationDependencies;
		this.hooks.thisCompilation.call(compilation, params);
		this.hooks.compilation.call(compilation, params);
		return compilation;
	}

	createNormalModuleFactory() {
		const normalModuleFactory = new NormalModuleFactory(
			this.options.context,
			this.resolverFactory,
			this.options.module || {}
		);
		this.hooks.normalModuleFactory.call(normalModuleFactory);
		return normalModuleFactory;
	}

	newCompilationParams() {
		const params = {
			normalModuleFactory: this.createNormalModuleFactory(),
			contextModuleFactory: this.createContextModuleFactory(),
			compilationDependencies: new Set()
		};
		return params;
	}

	...

	compile(callback) {
		const params = this.newCompilationParams();
		this.hooks.beforeCompile.callAsync(params, err => {
			if (err) return callback(err);

			this.hooks.compile.call(params);

			const compilation = this.newCompilation(params);

			this.hooks.make.callAsync(compilation, err => {
				if (err) return callback(err);

				compilation.finish(err => {
					if (err) return callback(err);

					compilation.seal(err => {
						if (err) return callback(err);

						this.hooks.afterCompile.callAsync(compilation, err => {
							if (err) return callback(err);

							return callback(null, compilation);
						});
					});
				});
			});
		});
	}
}
```

在分析这块源码前，我们先明确下 webpack 两个核心概念： Compiler 和 Compilation。

- Compiler 类（`./lib/Compiler.js`）： webpack 的主要引擎，在 compiler 对象记录了完整的 webpack 环境信息，在 webpack 从启动到结束，compiler 只会生成一次。你可以在 compiler 对象上读取到 webpack config 信息，outputPath 等；

- Compilation 类（`./lib/Compilation.js`）：代表了一次单一的版本构建和生成资源。compilation 编译作业可以多次执行，比如 webpack 工作在 watch 模式下，每次监测到源文件发生变化时，都会重新实例化一个 compilation 对象。一个 compilation 对象表现了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息。

#### Compiler 和 Compilation 区别？

compiler 代表的是不变的 webpack 环境； compilation 代表的是一次编译作业，每一次的编译都可能不同。

#### compiler.run()

单独截取 run 方法如下：

```js
run(callback) {
    const onCompiled = (err, compilation) => {
        ...
       if (this.hooks.shouldEmit.call(compilation) === false) {
		const stats = new Stats(compilation);
		stats.startTime = startTime;
		stats.endTime = Date.now();
		this.hooks.done.callAsync(stats, err => {
			if (err) return finalCallback(err);
			return finalCallback(null, stats);
		});
		return;
	}
        ...
    };

    this.hooks.beforeRun.callAsync(this, err => {
        if (err) return finalCallback(err);

        this.hooks.run.callAsync(this, err => {
            if (err) return finalCallback(err);

            this.readRecords(err => {
                if (err) return finalCallback(err);

                this.compile(onCompiled);
            });
        });
    });
}
```

在 run 函数里，首先触发了一些钩子：`beforeRun --> run --> done`，并在触发 run 钩子的时候，执行了 this.compile 方法。那么我们就去看下这个 compile 方法具体做了些什么。

#### compiler.compile()

首先截取 compile() 方法关键代码：

```js
compile(callback) {
    const params = this.newCompilationParams();
    this.hooks.beforeCompile.callAsync(params, err => {
        if (err) return callback(err);

        this.hooks.compile.call(params);

        const compilation = this.newCompilation(params);

        this.hooks.make.callAsync(compilation, err => {
            if (err) return callback(err);

            compilation.finish(err => {
                if (err) return callback(err);

                compilation.seal(err => {
                    if (err) return callback(err);

                    this.hooks.afterCompile.callAsync(compilation, err => {
                        if (err) return callback(err);

                        return callback(null, compilation);
                    });
                });
            });
        });
    });
}
```

代码中初始化了一个 compilation 实例对象，另外和 run 方法一些，compile 也触发一系列钩子：`beforeCompile --> compile --> make --> afterCompile`。

其中根据最上面的流程图，在 make 钩子阶段，webpack 开始了真正的对模块的编译。那么我们看下到底什么逻辑订阅了 make 钩子。通过全局搜索 `hooks.make.tapAsync`，我们可以看到 SingleEntryPlugin、MultiEntryPlugin、DllEntryPlugin、DynamicEntryPlugin 等插件中都订阅了 make 钩子。

那么我们先进入 SingleEntryPlugin 文件（`./lib/SingleEntryPlugin.js`）中查看，关键代码如下：

```js
class SingleEntryPlugin {
	constructor(context, entry, name) {
		this.context = context;
		this.entry = entry;
		this.name = name;
	}

	apply(compiler) {
		...
		compiler.hooks.make.tapAsync(
			"SingleEntryPlugin",
			(compilation, callback) => {
				const { entry, name, context } = this;

				const dep = SingleEntryPlugin.createDependency(entry, name);
				compilation.addEntry(context, dep, name, callback);
			}
		);
	}

	...
}
```

其中主要调用了 compilation.addEntry 方法，继续查看 compilation.addEntry（`./lib/Compilation.js`）。

```js
addEntry(context, entry, name, callback) {
	this.hooks.addEntry.call(entry, name);

	...
	this._addModuleChain(
		context,
		entry,
		module => {
			this.entries.push(module);
		},
		(err, module) => {
			...

			if (module) {
				slot.module = module;
			} else {
				const idx = this._preparedEntrypoints.indexOf(slot);
				if (idx >= 0) {
					this._preparedEntrypoints.splice(idx, 1);
				}
			}
			this.hooks.succeedEntry.call(entry, name, module);
			return callback(null, module);
		}
	);
}
```

通过上面代码我们可以看到 addEntry 有调用了 _addModuleChain，后面调用我就不在这里展示代码了，直接把调用栈列出来，感兴趣的同学可以自行查看源码。调用栈如下：

`this.addEntry --> this._addModuleChain --> this.addModule --> this.buidModule --> module.build`

addEntry 的作用是将模块的入口信息传递给模块链中，即 addModuleChain，随后继续调用 compiliation.factorizeModule，这些调用最后会将 entry 的入口信息”翻译“成一个模块(严格上说，模块一般是 NormalModule 实例化后的对象)。

下面是 buidModule 方法的关键代码，当 module.build 构建成功后，会调用 succeedModule 钩子，如果失败则调用 failedModule。

```js
buildModule(module, optional, origin, dependencies, thisCallback) {
	...

	this.hooks.buildModule.call(module);
	module.build(
		this.options,
		this,
		this.resolverFactory.get("normal", module.resolveOptions),
		this.inputFileSystem,
		error => {
			...
			const originalMap = module.dependencies.reduce((map, v, i) => {
				map.set(v, i);
				return map;
			}, new Map());
			module.dependencies.sort((a, b) => {
				const cmp = compareLocations(a.loc, b.loc);
				if (cmp) return cmp;
				return originalMap.get(a) - originalMap.get(b);
			});
			if (error) {
				this.hooks.failedModule.call(module, error);
				return callback(error);
			}
			this.hooks.succeedModule.call(module);
			return callback();
		}
	);
}
```

那么 module 又是从哪里来的？从 Compilation.js 代码中我们可以知道这个是 Module 类的实例，其中又具体分为 NormalModule、ExternalModule、MutiModule、DelegatedModule 等。

我们先进入到常见的 NormalModule 中查看源码（./lib/NormalModule.js）。nomalModule.build 又调用了自身的 nomalModule.doBuild 方法

```js
doBuild(options, compilation, resolver, fs, callback) {
	...

	runLoaders(
		{
			resource: this.resource,
			loaders: this.loaders,
			context: loaderContext,
			readResource: fs.readFile.bind(fs)
		},
		(err, result) => {...}
	)
}
```

nomalModule.doBuild 方法又调用了 runLoaders 方法来调用对应的 loader 对模块进行编译，写过 webpack loader 童鞋应该对 runLoader 比较熟悉，这个可以独立运行 webpack loader，而无需安装整个 webpack，对于调试 webpack loader 很方便，最终会通过 loader 的组合将所有模块（css，less，jpg 等）编译成标准的 js 模块。

得到标准 js 模块后，在 normalModule.doBuild 方法的最后一个参数即回调函数中，又对模块进行 paser。

```js
const result = this.parser.parse(source);
```

### 文件生成阶段
