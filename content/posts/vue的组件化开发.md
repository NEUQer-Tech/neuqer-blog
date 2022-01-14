---
title: "Vue的组件式开发"
date: 2021-12-19T18:28:34+08:00
author: 周卿玉
authorLink: https://github.com/JooKS-me
tags: ["技术分享","Vue"]
categories: ["前端技术分享"]
draft: false
---

# Vue的组件化开发

### 作用

将大的系统拆解成小的组件

整个页面逻辑整体进行处理不利于后期的维护和扩展，因此可以将多个小功能块放到一起组成大的页面

也就是将大组件分成小组件，小组件再抽像
其中小组件独立可复用

### 创建组件

（这里是最基本的创建组件的方式）

1.创建组件构造器    调用Vue.extend()

2.注册组件   Vue.component()(全局组件）

~~~javascript
const cpnC  = Vue.extend({
             template: '<div><h2>标题</h2><h4>内容1</h4></div>'
        
         })
         Vue.component('my-cpn',cpnC)
~~~

 3.使用组件    在Vue示例的作用范围内使用组件

~~~html
<div id="app">
     <my-cpn></my-cpn>    
 </div>
~~~

组件可以在实例中使用，不能拿出来单独使用

### 全局组件和局部组件

~~~javascript
const app = new Vue({
             el:'#app',
             data:{
                 message:'nihao',
             },
              components:{
                //  cpn：使用组件时的标签名
                cpn:cpnC1 //在这里注册的不是全局，只有一个能用
             }
         })
         const app2 = new Vue({
             el:'#app2'
         })
~~~

app2不能使用cpn

### 父组件和子组件

父组件里面可以有子组件

Vue实例也可以看作是一个组件（root组件），也可以将vue组件看作其他组件的父组件

~~~javascript
		//第一个组件构造器（子组件）
         const cpnC1  = Vue.extend({
             template: '<div><h2>标题</h2><h4>内容1</h4></div>',
         })
         //第二个组件构造器(父组件)
         const cpnC2  = Vue.extend({
             template: '<div> <h2>hhh</h2> <h4>lalal</h4>  <cpn1></cpn1> </div>',
            components:{
                cpn1:cpnC1
                //在这注册之后只能在这个组件里面使用
                //会优先在这找
            }//在第二个组件构造器中创建组件，
            //第一个组件可以放到第二个组件构造器中
         })
~~~

### 分离写法

全部在组件扩展器中写非常不方便，而且在js中写很多html语法会非常乱，所以使用分离写法

~~~html
 <!-- 第一种分离写法 -->
     <script type="text/x-template" id="cpn">
        <div> 
            <h2>hhh</h2> 
            <h4>lalal</h4>  
        </div>
     </script>

     <!-- 第二种 -->
     <template id='cpn2'>
        <div> 
            <h2>hhh</h2> 
            <h4>lalal</h4>  
        </div>
     </template>
~~~

然后在使用id为cpn的部分的时候直接使用template:'#cpn'即可

~~~javascript
Vue.component('cpn1',{
            template:'#cpn'
        })//直接传到extend()中
//这是以语法糖的形式注册全局组件
~~~

### 数据传递

类比vue实例中的存放方式，可以在定义组件的时候设定data和methods

data必须是一个函数，一定要有return，返回的是一个对象(多个组件示例使用的不是一个data对象，这样每次使用的时候调用一次data函数，在栈空间中产生一个新的对象)

~~~javascript
Vue.component('cpn1',{
            template:'#cpn',
            data(){
                return {
                    title: 'abc',
                    count:1
                }
            },
            methods:{
                add(){
                    this.count++;
                }
            }
            
        })
~~~

### 数据传递

#### 父传子props

~~~javascript
props:{
    cmovies:{
    type:Array,//限制了类型
    default(){
        return []
        }
    },
    cm:{
        type:String,
        default:'aaaaa',//提供默认值
        // require:true//要用这些属性必须传这个值，不然会报错
        }
},
~~~

type进行了类型限制，要求了传入的时候的必须类型
default提供了默认值（类型是对象或者数组的时候默认值必须是一个函数）
require:只要使用这个组件就一定要传这个值，否则就会报错

#### 子传递到父$emit

~~~html
<template id="cpn">   
    <div>       
        <button v-for="item in categories" @click="btnClick(item)">{{item.name}}</button>   </div>
</template>
~~~

(categories是子组件中的数组)

当父组件要拿到子组件中的item时

~~~javascript
btnClick(item){                   
    //子组件发射事件                          
    this.$emit('itemclick',item)                    
    //itemclick:事件名称,默认将item传过去               
}
~~~

然后在使用组件的时候v-on(也可写为@)事件

~~~html
<div id="app">              
    <cpn v-on:itemclick="cpnclick"></cpn>
    <!--@itemclick也可以，差别是@itemclick未传参时会传event，（itemClick不能写驼峰）-->
    <!-- cpnclick是在父组件中的 -->        
    <!-- 在子组件中通过$emit()来触发事件        
	在父组件中通过v-on来监听子组件事件 -->
</div>
~~~

#### $children 和 $ref

~~~javascript
btnClick(){    
    console.log(this.$children);//打印的是一个数组，可以通过这个方法直接拿到子组件    
    for(let c of this.$children){        
        console.log(c.name);        
        c.showMessage;//调用子组件中的方法    
    }    
    console.log(this.$refs.aaa.name);//拿到子组件    
    // $refs =>对象类型，默认是一个空的对象 必须在组件中加入ref=''
}
~~~

#### $parent 和 $root

~~~javascript
btnClick(){      
    // 访问父组件      
    console.log(this.$parent);      
    console.log(this.$parent.name);      
    //得到父组件中的元素   
    
    //  访问根组件root      
    console.log(this.$root);       
    console.log(this.$root.message);
}
~~~

### 插槽slot

组件在不同的地方放置时需要的内容不同，可以替换slot中的内容

可以给slot中加上默认值，没有其他内容将显示默认值

替换内容时可替换多个语句

~~~html
<div id="app">
        <cpn><button>按钮</button></cpn>
        <cpn><input type="text"></cpn>
        <cpn></cpn><!--使用默认值-->
        <cpn>
            <h2>hhh</h2>
            <h2>hehehe</h2>
        </cpn>
        <!-- 可以放多个 -->
        
    </div>

    <template id='cpn'>
        <div>我是组件
            <slot><button>按钮</button></slot>
        <!-- 组件在不同的地方放置时需要的东西不同 -->
        <!-- 要是在这里放置一个默认值，没有的也显示默认值 -->
        </div>
        
    </template>
~~~

#### 具名插槽

没有指定名字就只替换没有名字的插槽，有名字的要替换需要添加名字

~~~html
<div id="app">
        <cpn><span slot="c">标题</span></cpn>
        <cpn><span>我还是组件</span></cpn>
    </div>

    <template id='cpn'>
        <div>我是组件
            <slot name="l"><span>左边</span></slot>
            <slot name="c"><span>中间</span></slot>
            <slot name="r"><span>右边</span></slot>
            <slot>我不是组件</slot>
        </div>
        
    </template>
~~~

#### 作用域插槽

父组件替换插槽的标签，但是内容由子组件来提供 

~~~javascript
components:{
                cpn:{
                    template:'#cpn',
                    data(){
                        return {
                            pLanguages:['JS','C++','C#','Java']//这是子组件中的数据
                        }
                    },
                    created(){
                        this.pLanguages.join('-')
                    }
                }
            }
~~~

对数据有不同的展示需求时，可以先将数据放入插槽

然后在改变展示形式的时候，父组件不能直接拿到数据，因为数据不在vue实例中，需要子组件传过去

~~~html
<div id="app">
       <cpn></cpn>
       <cpn>
           <!-- 需要获取子组件中的数据 -->
           <template #default="slot">
                <span>{{slot.data.join(' - ')}}</span>
                <!-- join():将字符串用指定的字符连接 -->
           </template>
       </cpn>
    </div>

<template id='cpn'>
        <div>
           <slot :data="pLanguages">
               <ul>
                   <li v-for="item in pLanguages">{{item}}</li>
               </ul>
           </slot>
        </div>
        
    </template>
~~~

### 将组件抽取出来

其实vue中也是可以写template的，会替代el，但是如果在这里写很多行代码的话会很乱，所以要抽取出来变成组件（还有data、methods也要抽取出来)
然后只需要在vue中用components注册就可以了

我们可以将项目中的多个文件使用webpack进行打包，会自动寻找文件之间的依赖，只需要将main.js进行打包则所有的相关文件都会被打包生成一个文件，然后在html文件中自动引入

自己配置webpack非常复杂，需要使用脚手架（CLI）
