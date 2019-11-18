# 运行demo
```cli
    sudo npm run install:all
    sudo npm run start
```

访问`http://localhost:5000/`

# 基于single-spa的vue微前端项目

微前端的概念是从后端的微服务的迁移过来的。将 Web 应用由单一的单体应用转变为多个小型前端应用聚合为一的应用。各个前端应用还可以独立运行、独立开发、独立部署。
> 注意：这里的前端应用指的是前后端分离的单页面应用

## single-spa
single-spa是一个可以将多个前端应用聚合到同一个页面展示的框架。换句话说: single-spa会监听路由的变化，它会在特定的路由下将相应的应用挂载到指定的DOM节点上。

## 设计理念
* 中心化路由

    在前端应用中路由是中心，因为有了路由才能展示相应的界面。在基于single-spa的微前端项目中我们需要一个地方去管理我们的应用，即：发现存在哪些应用，这些应用都对应了哪个路由，在特定的路由先去加载这个应用对应的资源

* 标识化应用

    给每个应用都起一个唯一的名字。

* 生命周期

    single-spa设计了一个基本的生命周期，有五个状态：
    1. load: 决定加载哪个应用，并绑定生命周期
    2. bootstrap: 获取静态资源
    3. mount: 安装应用，如创建 DOM 节点
    4. unload: 删除应用的生命周期
    5. unmount: 卸载应用，如删除 DOM 节点

* 独立部署与配置自动化

    现在的前端项目的部署很大程度都是围绕这配置进行的，如果应用的配置能自动化，那么整个系统就自动化。如果我们要在微前端项目中添加或者删除一个应用，我们就更新微前端项目的配置，而这个配置应该自动生成。

## 项目结构图

![基于single-spa的vue微前端项目结构](./img/project-construction.png)

### 前端入口项目
前端入口项目不写业务代码，只是用于获取业务项目的配置(即：存在哪些业务项目，业务项目的入口)，注册各个业务项目以及加载各个业务项目的公共资源，入口项目有一个html文件，在业务项目处于激活状态时，将业务项目的DOM树挂载到入口项目的html中。

### 业务项目
业务项目的路由由自己定义，在整个微前端项目中，业务项目是按需加载。

## 实现方案
> 补充：我是使用systemJs加载静态资源
### 各个项目的配置
```
{
        name:'main-project',
        // 需要一直在页面中显示
        base:true,
        path:'/',
        // 项目的入口
        projectIndex:'http://localhost:9100',
        // 项目的入口js文件的路径。
        // 从项目入口html文件中用正则匹配到入口js文件的路径，将得到的路径保存到main字段中
        main:'',
        // 在html文件中的js脚本路径，这里的js脚本路径是过滤掉入口js之后的路径
        scripts:[],
        // 在html文件中用link引入的外部样式
        outerStyles:[],
        // html文件中style标签内嵌样式
        innerStyles:[]
    },
    {
        name:'customers',
        base:false,
        path:'/customers',
        domID:'main',
        projectIndex:'http://localhost:5100',
        main:'',
        scripts:[],
        outerStyles:[],
        innerStyles:[]
    },
    {
        name:'goods',
        base:false,
        path:'/goods',
        domID:'main',
        projectIndex:'http://localhost:9010',
        main:'',
        scripts:[],
        outerStyles:[],
        innerStyles:[]
    }
```

goods，customers和main-project是三个独立的项目。通过webpack打包之后js，css等静态资源的文件名是不固定的，但是各个项目的html访问入口是固定，所以为了获得每个项目的入口js的路径，先获取各个项目的html文件的内容，然后从html内容中匹配到入口js文件路径和其他的脚本路径

### 获取子项目html中插入的js路径
```js
function importHTML(projects) {
    const fetchPromises = [];
    projects.forEach(project => {
        const promise = window.fetch(project.projectIndex)
                            .then(response => response.text())
                            .then(html => {
                                // 从html中匹配出所有的脚本
                                const { entry,scripts } = processTpl(html,getDomain(project.projectIndex));
                                // 项目入口js路径
                                project.main = entry;
                                scripts.forEach(script => {
                                    if(script !== entry) {
                                        // 除入口js之外的其他js脚本
                                        project.scripts.push(script)
                                    }
                                });
                                return project;
                            });
        fetchPromises.push(promise);
    })

    return Promise.all(fetchPromises)
}
```
以项目中js资源为例，js资源分为两种，一种是一边运行一边加载的js脚本，另一种资源是通过打包工具（例如webpack）打包之后直接插入到html文件中的js资源（如：入口js），这部分js资源是保证项目正常运行的基础，在生产环境中为了资源加载的性能，不会只打包出一个包，所以在html文件中我们除了找到入口js的路径还需要找到其他的js路径，并且需要将其他的js脚本在bootstrap生命周期中插入到主项目的html中

importHTML函数接受的参数是项目的配置数组

### 注册应用
```js
function isActive(location,page) {
    let isShow = false;
    if(location.hash.startsWith(`#${page}`)){
        isShow = true
    }
    return isShow;
}
function activeFns(app) {
    return function (location) {
        return isActive(location,app.path)
    }
}
function registerApp(singleSpa,projects) {
    projects.forEach(function (project) {
        function start(app) {
            // 确保应用挂载点在页面中存在
            if(!app.domID || document.getElementById(app.domID)) {
                singleSpa.registerApplication(app.name,
                    () => {
                        return System.import(app.main).then(resData => {
                            return {
                                bootstrap:[resData.bootstrap,insertScriptsBootstrap(app.scripts)],
                                mount:resData.mount,
                                unmount:resData.unmount
                            }
                        })
                    },
                    project.base ? (function () { return true }) : activeFns(project))
            } else {
                setTimeout(function () {
                    start(app);
                },50)
            }
        }

        start(project);
    })
}
```

在获取到各个项目的入口js路径之后才注册应用，`insertScriptsBootstrap`函数的作用是：将除入口js之外的其他js脚本插入到html文档中
### 启动single-spa
```js
    singleSpa.start();
```

### 给各个应用注册生命周期函数
single-spa-vue是一个在vue项目中注册single-spa生命周期的工具库。

安装single-spa-vue
```cli
vue add single-spa

// 或者

npm install --save single-spa-vue
```

改写入口文件
```js
import vue from 'vue';
import App from './App.vue';
import router from './router';
import store from './store/base';
import singleSpaVue from 'single-spa-vue';
import elementUI from 'element-ui';
vue.use(elementUI)
vue.config.productionTip = false;
const vueLifecycles = singleSpaVue({
  Vue:vue,
  appOptions: {
    el:'#main',
    render: (h) => h(App),
    router,
    store
  },
});

export const bootstrap = vueLifecycles.bootstrap;
export const mount = vueLifecycles.mount;
export const unmount = vueLifecycles.unmount;
```

### 各个应用统一路由前缀
以goods项目为例，在定义路由的时候所有路由都以`/goods`开头
### 抽离公共资源

配置webpack的externals字段使webpack在打包的时候不打包公共库如(vue,vue-router,私有npm包等),如下：
```js
{
    externals:['vue',{'vue-router':'vueRouter'},{'element-ui':'elementUI'}]
}
```
### 用systemJS定义import map
import map 与webpack的externals配合使用能够让应用不打包公共库的代码，并且在应用运行的时候才加载公共库。
```
<script type="systemjs-importmap">
      {
        "imports": {
           "single-spa": "https://cdnjs.cloudflare.com/ajax/libs/single-spa/4.3.7/system/single-spa.min.js",
            "vue": "https://cdn.jsdelivr.net/npm/vue@2.6.10/dist/vue.js",
          "vueRouter": "https://cdn.jsdelivr.net/npm/vue-router@3.0.7/dist/vue-router.min.js",
          "elementUI":"https://cdn.jsdelivr.net/npm/element-ui@2.12.0/lib/index.js",
          "Vuex":"https://cdn.jsdelivr.net/npm/vuex@3.1.1/dist/vuex.min.js",
          "axios":"https://cdn.jsdelivr.net/npm/axios@0.19.0/dist/axios.min.js",
        }
      }
 </script>
```

这样代码在运行的时候遇到import、require时，会找到库在systemJs import map 中对应的路径，并进行动态外部加载，加载完成之后将库暴露出的对象赋值给代码中的变量。

*** 由于使用systemJs 加载资源，所有要将项目打包成umd格式的
```
output: {
    ...
    libraryTarget: 'umd',
    library: xxx,
}
```
### 配置跨域访问
```cli
    devServer:{
        port:9010,
        headers: {
            "Access-Control-Allow-Origin": "*",
        },
    },
```
由于子项目的资源需要在入口项目中访问，所以需要在子项目中配置跨域访问
### 子项目的publicPath
由于子项目的资源是在入口项目(入口项目和子项目在不同的域)中访问，所以需要将子项目的publicPath设置为完整的路径（即：包括协议和域名），这样才能保证子项目的资源能够正确加载。[output.publicPath](https://www.webpackjs.com/configuration/output/#output-publicpath)
### 各个应用间进行通信
使用浏览器自定义事件来实现各个应用间的通讯
```js
// customers
window.dispatchEvent(new CustomEvent('logout'));

// main-project
 window.addEventListener('logout',handler);
```
> 注意：各个应用之间应该尽可能少的进行通信，如果两个应用之间频繁的进行通信，那么它们两个应该合并成一个
### 隔离css样式
使用webpack，postcss在构建阶段为业务的所有CSS都加上自己的作用域
```
postcss:{
    plugins:[require('postcss-plugin-namespace')('.main-project',{ ignore: [ '*'] })]
}
```

## 参考文章
* [single-spa](https://single-spa.js.org/)
* [systemjs](https://github.com/systemjs/systemjs)
* [微前端那些事儿](https://github.com/phodal/microfrontends)
* [每日优鲜供应链前端团队微前端改造](https://juejin.im/post/5d7f702ce51d4561f777e258)
* [微前端项目](https://segmentfault.com/a/1190000019957162)
* [用微前端的方式搭建类单页应用-美团技术团队](https://tech.meituan.com/2018/09/06/fe-tiny-spa.html)
