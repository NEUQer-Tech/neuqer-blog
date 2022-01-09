---
title: "初识Three.js"
date: 2022-01-09T22:40:41+08:00
author: 黄一珂
authorLink: https://github.com/OREO-code
tags: ["技术分享","Three.js"]
categories: ["前端技术分享"]
draft: false
---

# 初识Three.js

## 1.引言

Three.js是基于原生WebGL封装运行的三维引擎，在所有WebGL引擎中，Three.js是国内文资料最多、使用最广泛的三维引擎。

PS:实现一个基于 WebGL 的 3D 应用或使用 CSS 实现一个复杂的 3D 特效对前端同学的要求相对高一些，除了熟练掌握 JavaScript、CSS 以外，还需要掌握图形学的相关内容、 着色器编程语言 `GLSL`。

在人与人之间联系的互联网时代，主要是满足人与人之间的交流，Web页面的交互界面主要呈现为2D的交互效果，比如按钮、输入框等。

随着物联网的发展,工业、建筑等各个领域与物联网相关Web项目网页交互界面都会呈现出3D化的趋势。

物联网粮仓3D可视化案例：http://www.yanhuangxueyuan.com/3D/liangcang/index.html

室内设计作品展示案例：http://www.yanhuangxueyuan.com/3D/houseDesign/index.html

## 2.资源

github链接：https://github.com/mrdoob/three.js

```
three.js-master
└───build——src目录下各个代码模块打包后的结果
    │───three.js——开发的时候.html文件中要引入的threejs引擎库，和引入jquery一样，可以辅助浏览器调试
    │───three.min.js——three.js压缩后的结构文件体积更小，可以部署项目的时候在.html中引入。
    │
└───docs——Three.js API文档文件
    │───index.html——打开该文件可以实现离线查看threejs API文档
    │
└───editor——Three.js的可视化编辑器，可以编辑3D场景
    │───index.html——打开应用程序
    │
└───examples——里面有大量的threejs案例，平时可以通过代码编辑全局查找某个API、方法或属性来定位到一个案例
    │
└───src——Three.js引擎的各个模块，可以通过阅读源码深度理解threejs引擎
    │───index.html——打开该文件可以实现离线查看threejs API文档
    │
└───utils——一些辅助工具
    │───\utils\exporters\blender——blender导出threejs文件的插件
```

Three.js官网：https://threejs.org/

Three.js中文网:[Three.js中文网 (webgl3d.cn)](http://www.webgl3d.cn/)

CDN引入three.js三维引擎

```javascript
<script src="http://www.yanhuangxueyuan.com/versions/threejsR92/build/three.js"></script>
```



#### [WebGL Report](http://webglreport.com/)

在线测试浏览器对WebGL1.0、WebGL2.0支持状况(IE.......)

#### [Physijs物理引擎](https://github.com/chandlerprall/Physijs)

可以协助基于原生WebGL或使用three.js创建模拟物理现象，比如重力下落、物体碰撞等物理现

#### [stats.js](https://github.com/mrdoob/stats.js)

JavaScript性能监控器，同样也可以测试webgl的渲染性能

#### [dat.gui](https://github.com/dataarts/dat.gui)

轻量级的icon形用户界面框架，可以用来控制Javascript的变量，比如WebGL中一个物体的尺寸、颜色

#### [tween.js](https://github.com/tweenjs/tween.js/)

借助tween.js快速创建补间动画，可以非常方便的控制机械、游戏角色运动

#### [ThreeBSP](https://github.com/sshirokov/ThreeBSP)

可以作为three.js的插件，完成几何模型的布尔，各类三维建模软件基本都有布尔的概念

## 3.基本架构

在正式开始学习Three.js之前，我们来先学习Three.js创建的3D场景的基本结构。

先举个例子，大家都生活在一个三维空间(3D场景之中)，我们把视线移除屏幕，就能看见许许多多的立体物品。

此时在一个小房间中，我正好看到了一瓶水。

那么我们是如何看见这些的呢？

房间中光线照射到物体的表面,经过物体（塑料瓶）在各个方向的反射,有些物体反射的光线进入人的眼睛，然后一路经过各种理化现象到达视网膜，并转变为光刺激，最终转变为神经冲动进入到我们的视觉中枢渲染，我们就看到了立体。

在刚才的例子中，我们可以归纳出三个要素：一个带有适当光照和实际立体的场景、一双正常的眼睛、还有正常的视觉中枢

那么这就对应上了我们three.js程序的三要素:场景(Scene)、相机(Camera)、渲染器(Renderer)

![](/images/three.png)

## 4.创建一个基本的平面（平面）

首先，引入js文件

其次，因为我们需要创建一个全屏的场景，所以要去除掉外边距



```
body{
			margin: 0;
			overflow: hidden;
		}
```

然后我们将会在div盒子中渲染我们的three.js（这里与canvas相似）

```
<div id="webGL-output">	
		</div>
```

我们在js部分，加入一个init函数（便于控制渲染进程）,用window.onload去触发它

```
<script>
		function init(){
		}
		window.onload=init
	</script>
```

那么接下来我们就在其中写作，首先three.js有三部分--场景，相机，渲染器

我们可以初始化一下

```
			//创建场景
			var scene=new THREE.Scene()
			//创建相机
			var camera= new THREE.PerspectiveCamera(45,window.innerWidth/window.innerHeight,0.1,2000)
			//创建渲染器
			var renderer=new THREE.WebGLRenderer()
```

相机中，45对应相机到渲染平面的两端直线连线举例的夹角，后一个参数为渲染平面的长宽比，最后两个为渲染平面到相机的最小和最大距离

对于渲染器，我们可以规定初始的渲染属性

```
			//确立基本的渲染颜色
			renderer.setClearColor(new THREE.Color(0xEEEEEE))
			//确立渲染高度和宽度
			renderer.setSize(window.innerWidth,window.innerHeight)
```

接下来，为了帮助我们能够查看各个物体的位置，我们可以加入一个长度为20的坐标轴并添加进我们的场景中

```
	var axes=new THREE.AxesHelper(20)
	scene.add(axes)//添加进场景中
```

现在我们可以创建一个长为60宽为20的浅灰色平面旋转可以使二维平面体现更多的立体感

首先我们确立一下基本属性

```
			var planeGeometry=new THREE.PlaneGeometry(60,20)//确定尺寸
			var planeMaterial=new THREE.MeshBasicMaterial({color:0xcccccc})//确定材质(颜色)
```

最后我们根据属性创建平面

```
var plane=new THREE.Mesh(planeGeometry,planeMaterial)//将两个属性传入到Mesh网格方法中创建
scene.add(plane)//加入到场景中
```

创建完之后，我们可以试着改变它的位置

```
plane.rotation.x= -0.5*Math.PI
			plane.position.x=15
			plane.position.y=0
			plane.position.z=0
```

当然，我们已经确定了这个平面在场景中的存在，如果我们想要看到他，那么就得设定我们的相机位置（注意xyz轴）

```
			camera.position.x=-30
			camera.position.y=40
			camera.position.z=30
```

设定它的朝向为我们创建的名字为scene的场景

```
camera.lookAt(scene.position)
```

之后通过添加子节点的形式，将我们创建的场景搭载到之前名为'webGL-output'的盒子中去

```
document.getElementById('webGL-output').appendChild(renderer.domElement)
```

添加过去就已经完事了吗？

我们还需要一次最终的渲染过程，通过调用我们之前创建的渲染器对象，传入对应的场景和相机来使用方法渲染。（技术分享时忘了这步，大家一定要记得加上渲染）

```
renderer.render(scene,camera)
```

那么应该你就可以看到一个平面和一个长为20单位的立体轴出现在你的屏幕上了

完整代码：

```
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8" />
		<title></title>
	</head>
	<style>
		body{
			margin: 0;
			overflow: hidden;
		}
	</style>
	<script src="js/three.js"></script>
	<body>
		<div id="webGL-output">
			
		</div>
	</body>
	<script>
		function init(){
			//创建场景
			var scene=new THREE.Scene()
			//创建相机
			var camera= new THREE.PerspectiveCamera(45,window.innerWidth/window.innerHeight,0.1,2000)
			//创建渲染器
			var renderer=new THREE.WebGLRenderer()
			renderer.setClearColor(new THREE.Color(0xEEEEEE))
			renderer.setSize(window.innerWidth,window.innerHeight)
			var axes=new THREE.AxesHelper(20)
			scene.add(axes)
			var planeGeometry=new THREE.PlaneGeometry(60,20)
			var planeMaterial=new THREE.MeshBasicMaterial({color:0xcccccc})
			//根据属性创建平面
			var plane=new THREE.Mesh(planeGeometry,planeMaterial)
			scene.add(plane)
			plane.rotation.x= -0.5*Math.PI
			plane.position.x=15
			plane.position.y=0
			plane.position.z=0
			camera.position.x=-30
			camera.position.y=40
			camera.position.z=30
			camera.lookAt(scene.position)
			document.getElementById('webGL-output').appendChild(renderer.domElement)
			renderer.render(scene,camera)
		}
		window.onload=init
	</script>
</html>

```

## 5.立方体和球体的添加

这两个立体图形的添加和之前平面的添加有许多相似之处

```
			//添加一个正方体
			var cubeGeometry= new THREE.BoxGeometry(4,4,4)//分别对应长宽高
			var cubeMaterial = new THREE.MeshLambertMaterial({color:0xFFFFFF})//添加材质
			var cube = new THREE.Mesh(cubeGeometry,cubeMaterial)//根据之前的尺寸和材质创建cube立方体
			//确定立方体位置
			cube.position.y=2
			cube.position.x=5
			cube.position.z=-10
			//最后添加立方体至scene场景
			scene.add(cube)
```

球体特别讲一下，第一个参数实际上是球体的半径，后面两个参数则规定的是球体的细分数，，球体你可以理解为正多面体，就像圆一样是正多边形，当分割的边足够多的时候，正多边形就会无限接近于圆，球体当精细度越大时越接近于球体

```
			var sphereGeometry = new THREE.SphereGeometry(4,20,20)//分别对应半径，水平细分数，垂直细分数
			var sphereMaterial = new THREE.MeshLambertMaterial({color:0xFFFFFF})//添加材质
			var sphere = new THREE.Mesh(sphereGeometry,sphereMaterial)//根据之前的尺寸和材质创建球体
			sphere.position.set(0,10,10)//设定位置，效果与之前设定方法一样
			//最后添加球体至scene场景
			scene.add(sphere)
```

## 6.聚光灯源的控制

```
			//确定点光源的颜色
			var spotLight = new THREE.SpotLight(0xFF0000)
			//确定位置
			spotLight.position.set(-30,30,-10)
			//添加光源至场景中
			scene.add(spotLight)
```

## 7.小结

以上部分只是简单介绍了Three.js的简单应用方法，真正实际应用场景中，能够运用Three.js实现动态渲染，动画，以及与使用者的交互才是Three.js广泛运用为前端开发增添光彩的重要部分，如果感兴趣的话，可以多多了解相关知识。