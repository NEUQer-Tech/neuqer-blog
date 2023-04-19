---
title: "Odin低代码平台文档"
date: 2023-04-19T14:54:57+08:00
author: 卞仁平
aurthorLink: https://github.com/Juns-g
tags: ["文档","项目"]
categories: ["项目文档"]
draft: false
---

# Odin低代码平台文档

odin站群系统

## odin-server odin服务端

### 架构

#### 代码分包

![1681885207159.png](/images/19-643f8a2c8d5c1.png)

#### COLA架构

![1681885224762.png](/images/19-643f8a2f8e04c.png)

#### 领域驱动模型图

![1681885229542.png](/images/19-643f8a2e961e3.png)

### 接口文档

https://www.apifox.cn/apidoc/shared-41e5b884-c128-4f53-9599-f097f08b91fc

### 技术栈

| 整体架构 | COLA                  | [官方文档](https://blog.csdn.net/significantfrank/article/details/110934799) | [Github](https://github.com/alibaba/COLA)                   |
| -------- | --------------------- | ------------------------------------------------------------ | ----------------------------------------------------------- |
| 权限控制 | jCasbin               | [官方文档](https://casbin.org/docs/overview)                 | [Github](https://github.com/casbin/jcasbin)                 |
| 文件存储 | Spring-file-storage   | [官方文档](https://spring-file-storage.xuyanwu.cn/)          | [Github](https://github.com/1171736840/spring-file-storage) |
| 数据库   | MySQL、Redis、MongoDB | /                                                            | /                                                           |

### 开发中遇到的问题及解决方案

| 姓名    | 遇到的问题                                                   | 解决方案                                                     |
| ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| @吴亚洲 | amis写后台管理系统的登录逻辑困难                             | 把cookie的检验放在server.js中，拦截所有请求，在所有请求之前查看是否存在cookie，不存在则跳转至登录界面，同时在发送所有请求时携带cookie里的登录信息，后端拦截并检验 |
| @吴亚洲 | amis要求的数据返回格式和项目之前使用的统一返回类格式不同     | 编写符合amis要求的返回类AmisDto.class                        |
| @吴亚洲 | amis请求后端接口的跨域问题                                   | 在后端请求拦截器中书写设置请求头的逻辑                       |
| @吴亚洲 | 统计每张图片访问量时每次访问一张图片就要进行mysql表的更新    | 把每张图片的访问量暂存到redis中，监听redis过期键，redis图片访问数据过期时取出这段时间的访问量再更新到mysql |
| @梁绍涵 | 混用stringredisTemplate和redisTemplate                       | 当redis中存入的数据是可读形式而非字节数组时，使用redisTemplate取值的时候会无法获取导出数据，要使用 StringRedisTemplate 试试。 |
| @梁绍涵 | 对redis键设置过期时间出错                                    | redis键在持久化后如果想再设置过期时间，需要先将这个键删除再重新设置，否则redis会将这次操作设置忽略 |
| @高昆   | 实现爬虫自动化                                               | 利用python的selenium实现模拟点击操作以实现自动化爬取         |
| @卢晨昕 | json数据的转换，amis返回后端的数据的格式有时是json类型，有时则是把json类型全部包装成一个字符串 | 后端使用object类型进行接受和转换，一种方法是写工具类进行转化，另一种是直接使用google的json类（alibaba的做不到） |
| @卢晨昕 | 爬取数据的自动转换                                           | 在python爬虫中对数据进行转换，增加一些必要字段，删除一些造成预览错误的字段 |
| @卢晨昕 | 数据备份，odin的数据库没有备份文件，同时也没有日志文件，所以无法进行数据的恢复 | 数据库开启日志功能和备份功能                                 |

### 个人总结

@吴亚洲 

在odin项目中，学会了amis的基本使用，熟悉了如何用amis快速构建一个后台管理系统；学习了一些鉴权和运维相关的知识

@高昆 

在odin项目过后,可以熟练使用amis搭建一个后台管理系统;加深了对非关系型数据库以及COLA架构的了解,学习了一些爬虫知识.

@梁绍涵 

通过odin项目，学习了amis使用，可以通过amis搭建一下简单的项目管理系统，提高了对项目开发的理解。

@卢晨昕 

odin项目深深的加固了我对于后端整体架构的记忆，并且对于项目的开发流程有了深刻的认识。同时amis毕业，已经可以满足以后的快速使用。对数据库的数据还原和js爬虫、python的selenium都有学习。

## odin-web odin前端

### 项目简介

#### 一、技术栈

- 编程语言：JavaScript+TypeScript
- 构建工具：Vite 3.x
- 前端框架：Vue 3.x
- 路由工具：Vue Router 4.x
- 状态管理：Pinia 2.x
- UI 框架：Element Plus
- CSS 预编译：Scss
- CSS 动画处理：Animate.css
- HTTP 工具：Axios
- 代码规范：Prettier + ESLint
- 提交规范：Commitizen + Commitlint

#### 二、项目说明

[Odin低代码平台](http://odin.neuqer.com/index)是一款基于Vue3+Element Plus实现的低代码静态网页站群制作方案。它的主要目标用户是那些没有开发经验的人员，他们可以使用拖拉拽组件的形式自行搭建所需的网站页面，无需编写任何代码。该平台的理念是简洁、轻松，旨在帮助用户快速构建网站。在编辑器中，用户可以通过切换路径来制作站群，以构建多个页面。对于需要精细调整的需求，Odin还提供了代码编辑功能。此外，用户可以从模板中心获取模板来提高网站搭建效率。如果用户在使用过程中遇到任何问题，Odin提供帮助中心，以解答问题。此外，Odin还提供了针对不同需求的客户制定的不同会员方案。

#### 三、功能

- 实时预览功能：用户可以在编辑页面时即时预览页面效果。
- 拖拽流程功能：通过拖拽不同的组件，用户可以快速构建页面流程。
- 可视化页面功能：页面编辑过程中，用户可以直观地看到页面结构和样式，并进行实时编辑。
- 多样模板使用功能：提供多个不同风格和用途的模板，方便用户根据需求选择和使用。
- 多页面站群系统功能：支持创建和管理多个页面，方便用户对站点进行维护和管理。
- 页面组件化搭建功能：通过将页面划分为多个组件，用户可以更灵活地搭建页面结构。
- 多样组件任意搭配功能：提供多种组件供用户选择和搭配，方便用户根据需要自由组合。
- 提供在线部署服务功能：提供在线部署服务，方便用户快速将页面上线并访问。
- 导出文件自行部署功能：支持将页面导出为文件，用户可以将其部署在自己的服务器上。

### 下载运行

1. 将[odin仓库(私有)](https://github.com/NEUQer-Tech/odin)fork到自己的仓库
2. 下载项目至本地，例如`git clone ``git@github.com``:L-H-X/odin.git`
3. 在编辑器中打开odin-web文件夹
4. 打开终端，请提前准备好最新版本的node环境
5. 下载依赖包`npm install`
6. 运行`npm run dev`

### 项目结构

```Plain
└─src ---------------------- // 源码目录
  ├─api -------------------- // 接口请求
  │ └─handle --------------- // axios封装以及error处理
  │   ├─axios.js 
  │   └─errorHandle.js 
  ├─App.vue 
  ├─component -------------- // 组件封装
  │ ├─BackStage 
  │ ├─IndexPage 
  │ ├─packages 
  │ └─pageEditor 
  │   ├─components --------- // 编辑器组件
  │   ├─editor.vue 
  │   ├─hook --------------- // 封装函数
  │   └─style -------------- // 编辑器样式
  ├─main.js 
  ├─pages ------------------ // 页面组件
  │ ├─Article -------------- // 文章页面
  │ ├─BackSatge ------------ // 管理后台
  │ ├─Dev ------------------ // 编辑器
  │ ├─Error ---------------- // 404界面
  │ ├─Index ---------------- // 首页等页面
  │ ├─Login ---------------- // 登录页面
  │ ├─Preview -------------- // 页面预览
  │ ├─SitePage ------------- // 在线部署的页面1
  │ └─WorkBench ------------ // 工作台
  ├─router ----------------- // 路由配置
  ├─stores ----------------- // 状态管理
  ├─style ------------------ // 通用样式
  └─utils ------------------ // 工具函数
    ├─editor.config.jsx ---- // 物料区
    ├─Materiel ------------- // 封装的物料区组件
    ├─message.js ----------- // ElMessage封装
    └─myCloudPop.js -------- // 滚动到视口触发动画的轮子
```

### 技术架构

![1681885241402.png](/images/19-643f8a309e2ba.png)

### 功能实现

#### 编辑区

![1681885245581.png](/images/19-643f8a306cfbc.png)

编辑区主要由这三个区域组成，最主要的交互区域是物料区和画布

#### 画布

- 文件位置：`src\component\pageEditor\editor.vue`的底部，类名是`editor-container`
- 使用的组件文件夹位置：`src\component\pageEditor`
- 编辑器的核心功能都是基于jsx实现的，有些是直接用的.jsx，有些是.vue里面的script中使用jsx
- 其中外层都是一些使用到的组件，例如标尺，缩放，网格，json编辑器等,通过参数控制显示隐藏
- 最里层的`<EditorBlock></EditorBlock>`是画布的内容组件
- `src\component\pageEditor\components\EditorBlock.vue`是画布组件
- 画布的数据使用pinia进行存储，位置在`src\stores\data.js`，EditorBlock中使用props来接收传来的数据，然后进行渲染，从物料区托组件到画布的过程中也会将数据传入pinia然后画布进行响应式渲染。
- 详情可以去看对应位置的代码

#### 物料区

没有点击组件时的物料区就是如图所示的样子，点击后下方会出现组件编辑区域

![1681885250053.png](/images/19-643f8a2dbc956.png)

物料区使用了Element Plus的标签页tabs组件，tabs里面的`el-tab-pane`是根据组件列表渲染的

```JavaScript
<el-tabs stretch style="margin:0 15px">
    {componentList.map(item => (
        <el-tab-pane label={item.label} lazy className={`editor-component-${item.Classkey}`}>
            {item.list.map(component => (
                <div>
                    <div className={`editor-component-${item.Classkey}-item component`}>
                        {component.preview()}
                    </div>
                    {item.tag ? (
                        <el-tag type="info" style="margin-bottom: 35px;">
                            {component.key}
                        </el-tag>
                    ) : null}
                </div>
            ))}
        </el-tab-pane>
    ))}
</el-tabs>
```

使用map遍历componentList，生成标签页，每一个标签页中里面根据componentList数组里面的list生成内容

```JavaScript
// 物料区列表
// label: 导航描述 | ClassKey：物料区CSS样式关键字 | list：物料 | tag：是否展示徽章描述
const componentList = [
    {
        label: '常用组件',
        Classkey: 'common',
        list: config.componentCommonList,
        tag: true
    },
    {
        label: '展示组件',
        Classkey: 'data',
        list: config.componentDataShowList,
        tag: true
    },
    { label: '表单组件', Classkey: 'form', list: config.componentFormList, tag: true },
    {
        label: '按钮组件',
        Classkey: 'button',
        list: config.componentButtonList,
        tag: false
    },
    { label: '图标组件', Classkey: 'icon', list: config.componentIconList, tag: false }
]
```

list的数据源于`src\utils\editor.config.jsx`，这里面是物料区的组件内容，可以通过import的路径寻找对应的组件，下面以`src\utils\Materiel\Button.jsx`为例讲解部分

```JavaScript
import { registerConfig } from '../editor.config.jsx'

export function buttonRegister() {
    registerConfig.register(
        {
            type: 'button',
            preview: () => <el-Button style="width:130px">Default</el-Button>,
            render: (blockStyles, block) => <el-Button style={blockStyles.value}>{block.text}</el-Button>,
            key: '默认按钮'
        })
}
```

preview是展示在物料区时组件的样子，render则是拖动进入画布时的内容，组建的样式由传入的blockStyles决定，内容等取决于传入的block参数

组件编辑区由一层`<KeepAlive></KeepAlive>`包裹，`<KeepAlive>` 包裹动态组件时，会缓存不活跃的组件实例，而不是销毁它们。

```JavaScript
<KeepAlive>
    <div
        className={
            data.popoverShow ? ["enter editor-attribute"] : ["leave editor-attribute"]
        }
    >
        {!judge.value ? (
            <div style="display:none"></div>
        ) : (
            <el-scrollbar>
                <el-collapse
                    style="margin: 0 15px"
                    accordion
                >
                    {!Object.keys(data.canvas.focusData.componentProps).length ? (
                        <div style="display:none"></div>
                    ) : (
                        <el-collapse-item
                            title="组件特有样式"
                            name="1"
                        >
                            <RightCptStyle />
                        </el-collapse-item>
                    )}
                    <el-collapse-item
                        title="组件位置"
                        name="2"
                    >
                        <RightPosition />
                    </el-collapse-item>
                </el-collapse>
            </el-scrollbar>
        )}
    </div>
</KeepAlive>;
```

核心区域使用的el-collapse组件处理，我只复制了前两条item，每一个item里面都写了一个vue组件，

里面的数据都是和pinia存储的双向绑定的，通过修改里面的属性进而实现画布对应的组件的样式修改。

#### 工具栏

组件位置在`src\component\pageEditor\components\EditorHeader.vue`

工具栏主要集成了一些全局功能，路径功能的代码实现也放在里面了，其他的还有撤销，恢复，保存，自动保存，标尺网格的显示隐藏，快捷键查看，图库，新手引导，页面下载，预览等功能。

#### 其他功能

- 组件拖拽`src\component\pageEditor\hook\useBlockDragger.js`
- 画布缩放`src\component\pageEditor\hook\useContainerZoom.js`
- 右键功能`src\component\pageEditor\hook\useRightMenu.js`

#### 物料区与画布交互

1. 用户可以通过双击或者拖拽物料区中的组件来将其添加至画布。这个功能由 `src\component\pageEditor\hook\useMenuDragger.js` 提供拖拽支持。
2. 当用户将组件拖拽至画布后，程序会使用 `nanoid` 生成一个唯一的组件 `id`，然后通过计算鼠标落点位置，确定组件的位置，并将组件及其通用样式和特有样式存储到 `data.canvas.blocks` 中。
3. 渲染组件
	1. 在 `editor-container` 内部，程序通过遍历 `data.canvas.blocks` 中的内容，将每个组件及其 `index` 传递给 `EditorBlock` 组件进行渲染。
	2. `EditorBlock` 组件使用 `pinia` 存储的 `data` 以及接受的 `props` 内容来创建组件。其中，使用 `blockStyles` 作为组件内核样式，该样式的值来自于 `props.block`；`blockStylesDiv` 是组件内核外层的一个 `div` 盒子的样式，其值同样来自于 `props.block`。
	3. 对于跳转事件，程序使用 `vue-router` 执行跳转操作，创建了 `linkEvent` 函数来处理跳转。同时，将该函数绑定至组件的 `onClick` 事件。
	4. 此外，针对新加的需求，程序为文本框等组件添加了双击编辑功能。当用户双击组件时，程序会改变 `isEditText` 的值。当其值为 `true` 时，程序不会返回组件，而是返回一个 `BlockEditText` 组件。该组件内部是一个 `textarea`，实现了文本内容的修改功能。

##### 画布内交互

1. 部分组件的双击编辑文本功能见上文3.d
2. 画布内拖拽功能
	1. `src\component\pageEditor\hook\useBlockDragger.js`提供支持
	2. 定义了一个名为 useBlockDragger 的函数，该函数接受两个参数：focusData 和 canvasRef。函数内部定义了一些变量和函数，包括 target、data、dragState、mousedown、mousemove、mouseup 和 moveMouseUp。mousedown 函数用于处理鼠标按下事件，mousemove 函数用于处理鼠标移动事件，mouseup 函数用于处理鼠标松开事件，moveMouseUp 函数用于处理鼠标移动松开事件。函数返回一个对象，其中包含 mousedown 函数。
3. 组件大小调整功能
	1. 点击组件之后，在组件四周会出现虚线和圆点，拖拽圆点可以修改组件的宽高
		1. ![1681885255909.png](/images/19-643f8a300646b.png)
	2. `src\component\pageEditor\components\Zoom.vue`提供该功能的支持
	3. 先在`template`中定义了八个控制点的`div`元素，样式见`style`
	4. 接下来，在 `script setup` 使用了 `useData` 来获取了一个全局的 `store`，并将 `data.canvas.focusData` 传递给 `getInitStyle` 函数来初始化这八个控制点的位置。并且使用了 `watchEffect` 来监控 `focusData` 的变化，以更新控制点的位置。
	5. 然后在主函数`resizeMain`里面修改`data.canvas.focusData`的值来实现组件宽高的调整
	6. 组件内还定义了`removeAll`函数，用于移除所有监听器，避免在松开鼠标后继续拖动或缩放。它在 mouseup 事件被触发时被调用，这个事件被添加在 onMounted 函数中，并在 onBeforeUnmount 函数中被移除。
	7. 
4. 对齐辅助线功能
	1. `src\component\pageEditor\components\MarkLine.vue`提供支持
	2. 文件的`template`部分定义了一个`div`盒子，里面使用`v-for`遍历生成多个辅助线，辅助线的显示隐藏通过`v-show`来控制
	3. 最主要的函数是`showLine`，它从`dataStore.canvas.blocks`里面获取了所有的`components`，然后进行遍历寻找到所需要显示辅助线的`component`，更加详细的可以去看代码，有注释
5. 画布缩放
	1. `src\component\pageEditor\hook\useContainerZoom.js`提供支持
	2. 使用`stores`里面的数据，然后，定义了一个名为 `wheelNow` 的事件处理函数，用于响应鼠标滚轮事件，实现缩放功能。
	3. 在该函数中，首先调用了 `preventDefault()` 方法来阻止默认的滚轮事件，然后根据滚轮方向调整 `data.canvas.zoom` 的值，并使用 `Number()` 函数将其转换为数字类型。
	4. 接着，使用 `toFixed()` 方法保留一位小数，并将结果再次转换为数字类型，最后将结果存储回 `data.canvas.zoom`。
	5. 在事件处理函数的最后，使用 `block.value.style.zoom` 将块元素的缩放比例设置为 `data.canvas.zoom`，从而实现了缩放功能。

### 接口文档

[postman文档链接](https://documenter.getpostman.com/view/22693886/2s84DmwP6F#/65cc54d9-fc73-46f9-a880-ee7ab890a9d5)

[apifox文档链接](https://www.apifox.com/apidoc/shared-41e5b884-c128-4f53-9599-f097f08b91fc/api-66450723)

### 参考资料

**https://blog.csdn.net/qq_47876165/category_11694140.html**

**https://www.bilibili.com/video/BV1y3411z7ST**

https://vcc3.sahadev.tech/

[阿里巴巴低代码引擎](https://lowcode-engine.cn/) 

[国内低代码平台从业者交流](https://github.com/taowen/awesome-lowcode)

[基于 Vite2.x + Vue3.x + TypeScript H5 低代码平台](https://github.com/buqiyuan/vite-vue3-lowcode)

https://github.com/woai3c/visual-drag-demo

https://github.com/daybrush/moveable 

[国内排名前 5 的低代码开发平台对比](https://zhuanlan.zhihu.com/p/559966882)

