---
title: "Webpack实现概述"
date: 2023-04-01T20:31:50+08:00
draft: false
author: 柯昌鹏
tags: ["技术分享", "前端工程化"]
authorLink: "https://github.com/kcfuler"
---

背景
虽然现在前端工程化发展迅猛，对应框架的各种脚手架工具已经覆盖了我们开发时的基本需求，大多数时候代码的构建、打包已经不需要我们自己手动去做。但如果各种脚手架工具提供的默认设置不能满足我们的实际需求，这时对工程化本身的理解就能给我们带来更多解决问题的思路和方法。
vite、snowpack 等新工具已经很好用，但是 webpack 凭借它优秀的拓展能力和强大的生态，依然是开发的主流选择
和 vite 的区别
vite 的底层是基于 rollup 和 esbuild

- rollup 也是一个打包工具，定位和 webpack 类似。
- esbuild 则是一个和 babel 定位类似的代码转换工具，它通过 go 语言的多线程能力和优秀的算法，得到了非常惊人的转换速度， 但它在目前不能做到代码转 es5 语法，也就意味着兼容性的问题。
  所以，vite 在开发时使用 esbuild 带来了极快的编译，大大提高了大型项目的开发体验，而在打包时使用了 rollup，保证了兼容性和基于 rollup 的可拓展性。
  webpack 可以看成是一个打包流程的控制工具，它提供一个打包的流程，官方和社区根据大家实际的开发需求实现功能插件。
  它的大多数功能通过配置 plugin 和 loader 的方式实现，babel 就是一个典型的例子，通过配置 babel，我们能解析 jsx 等其它类型的文件，通过配置其它插件，我们就能在打包流程上面实现我们对于打包的各种需求。
  webpack 的优点在于对 commonJS, AMD,等模块化方案的支持，它几乎可以满足所有的兼容性需求，而开发方面也可以通过接入 esbuild，swc 等方式提高开发体验，是一个大而全的选择。
  具体的操作可以参考这篇文章为什么他的 Webpack 这么快? - 掘金
  原理
  核心概念
  Entry
  入口起点(entry point)指示 webpack 应该使用哪个模块,来作为构建其内部依赖图的开始。
  进入入口起点后,webpack 会找出有哪些模块和库是入口起点（直接和间接）依赖的。
  每个依赖项随即被处理,最后输出到称之为 bundles 的文件中。
  Output
  output 属性告诉 webpack 在哪里输出它所创建的 bundles,以及如何命名这些文件,默认值为 ./dist。
  基本上,整个应用程序结构,都会被编译到你指定的输出路径的文件夹中。
  Module
  模块,在 Webpack 里一切皆模块,一个模块对应着一个文件。Webpack 会从配置的 Entry 开始递归找出所有依赖的模块。
  Chunk
  代码块,一个 Chunk 由多个模块组合而成,用于代码合并与分割。
  Loader
  loader 让 webpack 能够去处理那些非 JavaScript 文件（webpack 自身只理解 JavaScript 和 JSON）。
  loader 可以将所有类型的文件转换为 webpack 能够处理的有效模块,然后你就可以利用 webpack 的打包能力,对它们进行处理。
  本质上,webpack loader 将所有类型的文件,转换为应用程序的依赖图（和最终的 bundle）可以直接引用的模块。
  Plugin
  loader 被用于转换某些类型的模块,而插件则可以用于执行范围更广的任务。
  插件的范围包括,从打包优化和压缩,一直到重新定义环境中的变量。插件接口功能极其强大,可以用来处理各种各样的任务。
  打包流程
  webpack 整体就是一个文件处理的流水线，webpack 提供流水线的架构，通过 loader 和各种插件来实现文件的打包处理

1. 首先，根据配置信息（webpack.config.js）找到入口文件(src/index.js)
2. 找到入口文件所依赖的模块，并收集关键信息：比如路径、源代码、它所依赖的模块
3. 根据上一步得到的信息，生成最终输出到硬盘中的文件（dist）：包括 modules 对象、require 模板代码、入口执行文件等
   在这个过程中，由于 webpack 只认识 js 和 json 文件，所以就需要 loader 来对各种文件进行转换。
   在整体的流程中，也有很多时机需要特定的处理，比如：

- 在打包前需要校验用户传过来的参数，判断格式是否符合要求
- 在打包过程中，需要知道哪些模块可以忽略编译，直接引用 cdn 链接
- 在编译完成后，需要将输出的内容插入到 html 文件中
- 在输出到硬盘前，需要先清空 dist 文件夹
  而 plugin 机制就是在这一条整体的打包流水线上提供出来的接口，类似于 vue、react 中的生命周期函数，让我们有针对流程来对打包结果进行调整的能力
  具体实现

1. 搭建结构，读取配置参数
   class Compiler {
   constructor() {}

run(callback) {}
}

//第一步：搭建结构，读取配置参数，这里接受的是 webpack.config.js 中的参数
function webpack(webpackOptions) {
const compiler = new Compiler()
return compiler;
} 2. 用配置参数对象初始化 Compiler 对象
//Compiler 其实是一个类，它是整个编译过程的大管家，而且是单例模式
class Compiler {

- constructor(webpackOptions) {
- this.options = webpackOptions; //存储配置信息
- //它内部提供了很多钩子
- this.hooks = {
-     run: new SyncHook(), //会在编译刚开始的时候触发此run钩子
-     done: new SyncHook(), //会在编译结束的时候触发此done钩子
- };
- }
  }

//第一步：搭建结构，读取配置参数，这里接受的是 webpack.config.js 中的参数
function webpack(webpackOptions) {
//第二步：用配置参数对象初始化 `Compiler` 对象

- const compiler = new Compiler(webpackOptions)
  return compiler;
  }

3. 挂载配置文件中的插件
   //自定义插件 WebpackRunPlugin
   class WebpackRunPlugin {
   apply(compiler) {
   compiler.hooks.run.tap("WebpackRunPlugin", () => {
   console.log("开始编译");
   });
   }
   }

//自定义插件 WebpackDonePlugin
class WebpackDonePlugin {
apply(compiler) {
compiler.hooks.done.tap("WebpackDonePlugin", () => {
console.log("结束编译");
});
}
}

- const { WebpackRunPlugin, WebpackDonePlugin } = require("./webpack");
  module.exports = {
  //其他省略
- plugins: [new WebpackRunPlugin(), new WebpackDonePlugin()],
  };
  //第一步：搭建结构，读取配置参数，这里接受的是 webpack.config.js 中的参数
  function webpack(webpackOptions) {
  //第二步：用配置参数对象初始化 `Compiler` 对象
  const compiler = new Compiler(webpackOptions);
  //第三步：挂载配置文件中的插件
- const { plugins } = webpackOptions;
- for (let plugin of plugins) {
- plugin.apply(compiler); // 挂载插件
- }
  return compiler;
  }

4. 执行 Compiler 对象的 run 方法开始执行编译
   这一步是关键，周期的钩子在这个部分触发
   //Compiler 其实是一个类，它是整个编译过程的大管家，而且是单例模式
   class Compiler {
   constructor(webpackOptions) {
   //省略
   }

- compile(callback){
- //
- }

- //第四步：执行`Compiler`对象的`run`方法开始执行编译
- run(callback) {
- this.hooks.run.call(); //在编译前触发 run 钩子执行，表示开始启动编译了
- const onCompiled = () => {
-     this.hooks.done.call(); //当编译成功后会触发done这个钩子执行
- };
- this.compile(onCompiled); //开始编译，成功之后调用 onCompiled
  }
  }

5. 根据配置文件中的 entry 配置项找到所有的入口
6. 从入口文件出发，调用配置的 loader 规则，对各模块进行编译
   Loader 本质上就是一个函数，接收资源文件或者上一个 Loader 产生的结果作为入参，最终输出转换后的结果。
   实现的一个关键点
   class Compilation {
   constructor(webpackOptions) {
   this.options = webpackOptions;
   this.modules = []; //本次编译所有生成出来的模块
   this.chunks = []; //本次编译产出的所有代码块，入口模块和依赖的模块打包在一起为代码块
   this.assets = {}; //本次编译产出的资源文件
   this.fileDependencies = []; //本次打包涉及到的文件，这里主要是为了实现 watch 模式下监听文件的变化，文件发生变化后会重新编译
   }

buildModule(name, modulePath) {
//省略其他
}

build(callback) {
//第五步：根据配置文件中的`entry`配置项找到所有的入口
//省略其他
//第六步：从入口文件出发，调用配置的 `loader` 规则，对各模块进行编译
for (let entryName in entry) {
//entryName="main" entryName 就是 entry 的属性名，也将会成为代码块的名称
let entryFilePath = path.posix.join(baseDir, entry[entryName]); //path.posix 为了解决不同操作系统的路径分隔符,这里拿到的就是入口文件的绝对路径
//6.1 把入口文件的绝对路径添加到依赖数组（`this.fileDependencies`）中，记录此次编译依赖的模块
this.fileDependencies.push(entryFilePath);
//6.2 得到入口模块的的 `module` 对象 （里面放着该模块的路径、依赖模块、源代码等）
let entryModule = this.buildModule(entryName, entryFilePath);

-     //6.3 将生成的入口文件 `module` 对象 push 进 `this.modules` 中
-     this.modules.push(entryModule);
      }
      //编译成功执行callback
      callback()
  }
  }

7. 找出此模块所依赖的模块，再对依赖模块进行编译
   这部分代码比较多，实现的部分就是通过代码的依赖分析，找到各个模块之间的关系。

- loader 对代码进行转换
  sourceCode = loaders.reduceRight((code, loader) => {
  return loader(code);
  }, sourceCode);

8. 等所有模块都编译完成后，根据模块之间的依赖关系，组装代码块 chunk
   class Compilation {
   constructor(webpackOptions) {
   this.options = webpackOptions;
   this.modules = []; //本次编译所有生成出来的模块
   this.chunks = []; //本次编译产出的所有代码块，入口模块和依赖的模块打包在一起为代码块
   this.assets = {}; //本次编译产出的资源文件
   this.fileDependencies = []; //本次打包涉及到的文件，这里主要是为了实现 watch 模式下监听文件的变化，文件发生变化后会重新编译
   }

buildModule(name, modulePath) {
//省略其他
}

build(callback) {
//第五步：根据配置文件中的`entry`配置项找到所有的入口
//省略其他
//第六步：从入口文件出发，调用配置的 `loader` 规则，对各模块进行编译
for (let entryName in entry) {
//entryName="main" entryName 就是 entry 的属性名，也将会成为代码块的名称
let entryFilePath = path.posix.join(baseDir, entry[entryName]); //path.posix 为了解决不同操作系统的路径分隔符,这里拿到的就是入口文件的绝对路径
//6.1 把入口文件的绝对路径添加到依赖数组（`this.fileDependencies`）中，记录此次编译依赖的模块
this.fileDependencies.push(entryFilePath);
//6.2 得到入口模块的的 `module` 对象 （里面放着该模块的路径、依赖模块、源代码等）
let entryModule = this.buildModule(entryName, entryFilePath);
//6.3 将生成的入口文件 `module` 对象 push 进 `this.modules` 中
this.modules.push(entryModule);
//第八步：等所有模块都编译完成后，根据模块之间的依赖关系，组装代码块 `chunk`（一般来说，每个入口文件会对应一个代码块`chunk`，每个代码块`chunk`里面会放着本入口模块和它依赖的模块）

-     let chunk = {
-       name: entryName, //entryName="main" 代码块的名称
-       entryModule, //此代码块对应的module的对象,这里就是src/index.js 的module对象
-       modules: this.modules.filter((item) => item.names.includes(entryName)), //找出属于该代码块的模块
-     };
-     this.chunks.push(chunk);
      }
      //编译成功执行callback
      callback()
  }
  }

9. 把各个代码块 chunk 转换成一个一个文件加入到输出列表
10. 确定好输出内容之后，根据配置的输出路径和文件名，将文件内容写入到文件系统

实践
下面通过一个小例子来解释上述的各个过程在实际开发中的体现
https://github.com/kcfuler/frontend-interview/tree/main/%E5%89%8D%E7%AB%AF%E5%B7%A5%E7%A8%8B%E5%8C%96/webpack/06_webpack%E5%88%86%E5%8C%85-%E5%8A%A8%E6%80%81%E5%AF%BC%E5%85%A5
参考文章 & 推荐阅读
webpack
二十张图片彻底讲明白 Webpack 设计理念，以看懂为目的 - 掘金
