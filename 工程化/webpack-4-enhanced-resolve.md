# 剖析 webpack 4 模块解析

## 需要具备的基础

- webpack tapable「看看git仓库的README基本就差不多了」.
- 简单会用webpack.

## 解析原理

webpack使用的是enhanced-resolve这个模块在进行解析，解析的原理可以简单理解成是一个管道「pipeline」 进行解析，从最初的地方传入要解析的路径，并且也要传回调函数，经过一个个插件解析，最终有了结果后返回 or 报错.

哈哈哈，是不是感觉似曾相识，有点类似koa的洋葱模型，但是当然更加复杂.

### 初始化剖析

整体的设计原理和webpack一脉相承，基于tapable的插件化机制进行设计，所有的功能都在各个插件中.

```
// enhanced-resolve/lib/ResolverFactory.js
// 这个文件是用来生Resolver的，以及注册相关的插件.

exports.createResolver = function(options) {
	//// OPTIONS ////

  // 初始化参数

	//// options processing ////

	if (!resolver) {
		resolver = new Resolver(
			useSyncFileSystemCalls
				? new SyncAsyncFileSystemDecorator(fileSystem)
				: fileSystem
		);
	}

	//// pipeline ////

	resolver.ensureHook("resolve");
	resolver.ensureHook("parsedResolve");
	resolver.ensureHook("describedResolve");
	resolver.ensureHook("rawModule");
	resolver.ensureHook("module");
	resolver.ensureHook("relative");
	resolver.ensureHook("describedRelative");
	resolver.ensureHook("directory");
	resolver.ensureHook("existingDirectory");
	resolver.ensureHook("undescribedRawFile");
	resolver.ensureHook("rawFile");
	resolver.ensureHook("file");
	resolver.ensureHook("existingFile");
	resolver.ensureHook("resolved");

  // 根据参数注入一系列插件

	//// RESOLVER ////

  // 注入插件
	plugins.forEach(plugin => {
		plugin.apply(resolver);
	});

	return resolver;
};
```

### 路径解析

绝对路径和模块解析原理类似，就不解释了.

解析是从enhanced-resolve/lib/Resolver.js中的resolve方法开始的，开始一个接一个的启动tapable的事件，最初开始的是resolve. 下面通过具体的例子看看整体流程怎么走的.

注意
- 下面的每个插件下面有两个事件名字，是初始化的时候输入的，第一个是监听的事件，第二个是将要启动的事件「这也是插件一个个顺序执行的原因」.
- 插件按照嵌套关系执行，省略了一些无关插件.

```
// 解析 './index.js'
// 主要解析发生在JoinRequestPlugin，将相对路径和context路径拼接起来，然后判断下是否存在文件，就解析结束了.
* the whole pipeline & 每个plugin作用.
    * this.hooks.resolve 启动.
    * 
    * UnsafeCachePlugin.「缓存」
        * resolve.
        * new-resolve.
    * 
    * ParsePlugin.「请求解析」
        * new-resolve.
        * parsed-resolve.
        *  
        * 对请求解析的数据做一层解析，e.g. 判断下文件/目录/模块?.
    * 
    * DescriptionFilePlugin「获取package.json文件」
        * parsed-resolve.
        * described-resolve.
        *  
        * ….一堆其他注入的插件.  「影响不大，可以无视」
        * 
        * JoinRequestPlugin.「相对路径解析join，就是发生在这里」
            * after-described-resolve.
            * relative.
            *  
            * DescriptionFilePlugin.
                * relative.
                * described-relative.
                *   
                * - - - 文件和目录解析 - - -  
                * 
                * FileKindPlugin.「判断下是否是文件」
                    * described-relative.
                    * raw-file.
                    *  
                    * TryNextPlugin.「过渡到下一个插件，相当于一个中转站」
                        * raw-file.
                        * file.
                        * 
                        * FileExistsPlugin. 「判断文件是否存在」
                            * file.
                            * existing-file.
                            *  
                            * NextPlugin. 「过渡到下一个插件，相当于一个中转站」
                                * existing-file.
                                * resolved. 
                                *  
                                * ResultPlugin.「处理结果，最终的处理插件」
                                    * resolved.

```

```
// 解析 '@/index.js' # alias @=path.resolve(__dirname, src)
// 主要解析发生在AliasPlugin，这个插件将 alias & 路径拼接起来，然后继续走下面的插件判断文件是否存在.
* the whole pipeline & 每个plugin作用.
    * this.hooks.resolve 启动.
    * 
    * 
    * UnsafeCachePlugin.「缓存」
        * resolve.
        * new-resolve.
    * 
    * ParsePlugin.「请求解析」
        * new-resolve.
        * parsed-resolve.
        *  
        * 对请求解析的数据做一层解析，e.g. 判断下文件/目录/模块?.
    * 
    * DescriptionFilePlugin「获取package.json文件」
        * parsed-resolve.
        * described-resolve.
        *  
        *  
        * AliasPlugin.「别名解析」
            * described-resolve.
            * resolve. 
            *  
            * UnsafeCachePlugin.「缓存」
                * resolve.
                * new-resolve.
            * 
            * ParsePlugin.「请求解析」
                * new-resolve.
                * parsed-resolve.
                *  
                * 对请求解析的数据做一层解析，e.g. 判断下文件/目录/模块?.
            * 
            * DescriptionFilePlugin「获取package.json文件」
                * parsed-resolve.
                * described-resolve.
                * 
                * ….一堆其他注入的插件.  「影响不大，可以无视」
                * 
                * JoinRequestPlugin.「相对路径解析join，就是发生在这里」
                    * after-described-resolve.
                    * relative.
￼
                    *  
                    * DescriptionFilePlugin.
                        * relative.
                        * described-relative.
                        *   
                        * 
                        * - - - 文件和目录解析 - - -  
                        * 
                        * FileKindPlugin.「判断下是否是文件」
                            * described-relative.
                            * raw-file.
                            *  
                            * TryNextPlugin.「过渡到下一个插件，相当于一个中转站」
                                * raw-file.
                                * file.
                                * 
                                * FileExistsPlugin. 「判断文件是否存在」
                                    * file.
                                    * existing-file.
                                    *  
                                    * NextPlugin. 「过渡到下一个插件，相当于一个中转站」
                                        * existing-file.
                                        * resolved. 
                                        *  
                                        * ResultPlugin.「处理结果，最终的处理插件」
                                            * resolved.
```

## 自定义改造解析方式

相信总是有一些奇怪的业务场景，需要进行自定义改造，这也是源码阅读的必要吧. 自定义路径解析其实整体思路挺简单的，加一个插件，如果对于想继续走原来逻辑的，就`callback()`，如果想要自定义解析的就返回`callback(null, result)`.

我之所以进行剖析是因为项目需要拆分，将一个项目按照业务 or 功能进行拆分，用monorepo的方式进行管理，所以之前项目中 `@` 方式的alias路径解析，会需要在多个项目路径进行判断解析。

你或许会说为啥不将相关代码拆到一个公共项目，当然是要考虑成本的，项目拆分改造意味着后续的代码都要copy-paste上来，所以当然得越快越好，并且对业务代码也要尽可能少地改动，避免对业务造成了影响.

webpack5已经可以支持array alias解析了，但是目前beta阶段如果升4->5，对项目的风险来说更大，所以只能寻求在webpack4的基础上支持array解析，从前面的源码分析应该知道webpack进行模块解析的功能都是插件支持的，所以只要将webpack5中使用的插件搬过来就好了，前提当然是要对相关的插件源码进行审核了，整体的流程如下.

```
// 解析 '@/index.js'
// 主要解析发生在AliasPluginNew，这个插件会遍历alias数组，将 alias & 路径拼接起来，然后继续走下面的插件判断文件是否存在，一个个校验.

// webpack.config.js
const path = require('path');
const AliasPluginNew = require('webpack/node_modules/enhanced-resolve/lib/AliasPluginNew');

module.exports = {
  entry: '@/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
  },
  resolve: {
    plugins: [
      new AliasPluginNew('described-resolve', [
        {
          name: '@',
          alias: [
            path.resolve(__dirname, 'src'),
            path.resolve(__dirname, 'otherSrc'),
          ]
        },
      ], 'resolve')
    ],
  }
};

* the whole pipeline & 每个plugin作用.
    * this.hooks.resolve 启动.
    * 
    * 
    * UnsafeCachePlugin.「缓存」
        * resolve.
        * new-resolve.
    * 
    * ParsePlugin.「请求解析」
        * new-resolve.
        * parsed-resolve.
        *  
        * 对请求解析的数据做一层解析，e.g. 判断下文件/目录/模块?.
    * 
    * DescriptionFilePlugin「获取package.json文件」
        * parsed-resolve.
        * described-resolve.
        *  
        *  
        * AliasPluginNew.「多别名解析」
            * described-resolve.
            * resolve. 
            *  
            * UnsafeCachePlugin.「缓存」
                * resolve.
                * new-resolve.
            * 
            * ParsePlugin.「请求解析」
                * new-resolve.
                * parsed-resolve.
                *  
                * 对请求解析的数据做一层解析，e.g. 判断下文件/目录/模块?.
            * 
            * DescriptionFilePlugin「获取package.json文件」
                * parsed-resolve.
                * described-resolve.
                * 
                * ….一堆其他注入的插件.  「影响不大，可以无视」
                * 
                * JoinRequestPlugin.「相对路径解析join，就是发生在这里」
                    * after-described-resolve.
                    * relative.
                    *  
                    * DescriptionFilePlugin.
                        * relative.
                        * described-relative.
                        *   
                        * 
                        * - - - 文件和目录解析 - - -  
                        * 
                        * FileKindPlugin.「判断下是否是文件」
                            * described-relative.
                            * raw-file.
                            *  
                            * TryNextPlugin.「过渡到下一个插件，相当于一个中转站」
                                * raw-file.
                                * file.
                                * 
                                * FileExistsPlugin. 「判断文件是否存在」
                                    * file.
                                    * existing-file.
                                    *  
                                    * NextPlugin. 「过渡到下一个插件，相当于一个中转站」
                                        * existing-file.
                                        * resolved. 
                                        *  
                                        * ResultPlugin.「处理结果，最终的处理插件」
                                            * resolved.
```

## 谈谈源码阅读

对于开发人员来说，阅读源码的重要性不言而喻了吧，常规来说就是下面几种情况吧：
- 要解决一个bug，但是google搜不到.
- 要引入一个框架 or 库，但是要根据业务需要改造.
- 要设计一个公司内部的框架.

那么如何阅读源码呢？
1. 理解整体的设计流程.
2. 其实很简单，源码阅读换句话来说就是 *逆向工程* ，就是从具体的实际使用场景出发，去看作者怎么实现的，并且对于稍微复杂点的项目而言，如果你不了解作者要解决的问题，你是很难看懂代码干了些啥的.「谨记，如果实在看不懂，但不影响解决问题，就不要死磕了，因为可能使用场景你不知道」

在阅读的过程中一定要记得借助相关的调试工具进行断点，不然的话，看懂是很难的.

## 最后

欢迎各位大佬评论交流学习.
