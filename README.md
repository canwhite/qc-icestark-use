# qc-icestark-use
icestark微服务框架使用心得和部分内容的讲解  
以vue为例，注意基座和微应用都可以有多种选择  

### 基座的创建
```
npm init ice icestark-layout @vue-materials/icestark-layout-app
cd icestark-layout
npm install
npm start
-----------------
上边是使用了脚手架，如果要对已有项目进行改造
npm i --save @ice/stark
npm i --save @ice/stark-app
-----------------
至于在基座中的使用是一致的，在App.vue中

(1)引入
import { registerMicroApps, start } from '@ice/stark';
import { setBasename } from '@ice/stark-app';
import Layout from './layouts/BasicLayout'//这个是侧边栏，一会将路由的时候分析

(2)注册微应用
一般放在mounted钩子里,
name：字段为微应用唯一标识，可以自己定义，但是必须唯一
activePath：微应用激活的路由规则，也是自己定义

微应用项目加载有url和entry两种，
url分别对应js和css文件为一个数组，http和https加载均支持

//微应用挂载的DOM节点，通常情况下所有微应用的 container 都是同一个。
const container = document.getElementById('container');
registerMicroApps([
    {
        name: 'seller',
        activePath: '/seller',
        title: '商家平台',
        sandbox: true,
        url: [
            'https:////iceworks.oss-cn-hangzhou.aliyuncs.com/icestark/child-seller-react/build/js/index.js',
            'https:////iceworks.oss-cn-hangzhou.aliyuncs.com/icestark/child-seller-react/build/css/index.css',
        ],
        container,
    },
    {
        // TODO: Angular
        name: 'angular',
        activePath: '/angular',
        title: 'Angular',
        sandbox: true,
        entry: 'https://iceworks.oss-cn-hangzhou.aliyuncs.com/icestark/child-common-angular/index.html',//也可以直接给html入口
    }
]);

(3)微应用的生命周期监听
同样在mounted中实现，相当于一个内置监听
start({
    //预加载
    prefetch:true,
    //沙箱，通过 with + new Function 的形式，为微应用脚本创建沙箱运行环境
    //并通过 Proxy 代理阻断沙箱内对 window 全局变量的访问和修改。
    sandbox:true,
    onAppEnter: (appConfig) => {
        const { activePath } = appConfig;//activePath 基座自定义路由
        setBasename(activePath)//和微应用的路由衔接
    },
    onLoadingApp: () => {
        this.loading = true;
    },
    onFinishLoading: () => {
        this.loading = false;
    },
    onRouteChange: (_, pathname) => {
    // 处理微应用间跳转无法触发 Vue Router 响应
        this
            .$router
            .push(pathname)
            .catch(() => {})
    },
    onActiveApps: (activeApps) => {
        this.microAppsActive = (activeApps || []).length;
    }
});
*****************************************************
(4)这里有个setBasename是为了将自定义路由微应用的路由作拼接的
onAppEnter: (appConfig) => {
    const { activePath } = appConfig;//activePath 基座自定义路由
    setBasename(activePath)//和微应用的路由衔接
},

```
### 微应用的创建
```
npm init ice icestark-child @vue-materials/icestark-child-app
yarn install
yarn start
----------------------
改造的话注意引入
npm i --save @ice/stark-app
----------------------

(1)在main.js中导出微应用相关生命周期

*****************************************************
vue2.x
// 应用入口文件 src/main.js
import Vue from 'vue';
import isInIcestark from '@ice/stark-app/lib/isInIcestark';
import setLibraryName from '@ice/stark-app/lib/setLibraryName';

let vue;

// 注意：`setLibraryName` 的入参需要与 webpack 工程配置的 output.library 保持一致
setLibraryName('microApp');

export function mount(props) {
  const { container } = props;
  vue = new Vue(...).$mount();
  // for vue don't replace mountNode
  container.innerHTML = '';
  container.appendChild(vue.$el);
}

export function unmount() {
  vue && vue.$destroy();
}

if (!isInIcestark()) {
  new Vue(...);
}

*****************************************************
vue3.x

import Vue from 'vue';
import { isInIcestark, setLibraryName } from '@ice/stark-app';
import ElementUI from 'element-ui';
import 'element-ui/lib/theme-chalk/index.css';
import './styles/global.scss';
import App from './App.vue';
import router from './router';

Vue.use(ElementUI);

Vue.config.productionTip = false;

let vue = null;

// set in vue.config.js
setLibraryName('icestark-vue-demo');

export function mount(props) {
  const { container } = props;
  vue = new Vue({
    router,
    mounted: () => {
      console.log('App mounted');
    },
    destroyed: () => {
      console.log('App destroyed');
    },
    render: h => h(App),
  }).$mount();

  // for vue don't replace mountNode
  container.innerHTML = '';
  container.appendChild(vue.$el);
}

export function unmount() {
  if (vue) vue.$destroy();
  vue.$el.innerHTML = '';
  vue = null;
}

if (!isInIcestark()) {
  // 初始化 vue 项目
  // eslint-disable-next-line no-new
  new Vue({
    router,
    el: '#app',
    mounted: () => {
      console.log('App mounted');
    },
    destroyed: () => {
      console.log('App destroyed');
    },
    render: h => h(App),
  });
}


(2)这里边setLibraryName需要和webpack中output.library保持一致
并在webpack中构建为umd文件适配前后端
vue.config.js

output: {
    library: 'icestark-vue-demo',
    libraryTarget: 'umd',
}
这部分主要是为了服务于基座注册信息和app映射的

(3)路由拼接
基座里有 setBasename(activePath)是把自定义路由暴露出来
我们微应用定义路由的时候需要把自定义路由拿来作拼接

import Vue from 'vue';
import Router from 'vue-router';
import { getBasename } from '@ice/stark-app';
import routes from '@/config/routes';

Vue.use(Router);

export default new Router({
  routes,
  mode: 'history',
  base: getBasename(),
});
这里的的getBasename就是为了拿到自定义路由
如果微应用单独跑的话，得到的是空字符串


```

### 微应用往基座的挂载

```
我按照上一节思路定义了一个微应用
如果往基座上挂载呢
(1)打包
npm run build 
打完包之后正常情况要把文件放到服务器，我们这里直接本地起了服务
其中js文件的地址是
http://127.0.0.1:5500/dist/js/app.js
css文件的地址是
http://127.0.0.1:5500/dist/css/app.css

(2)在基座中注册

registerMicroApps([
    {
    name: 'custom',//这个名字随便起，但是是唯一的
    activePath: '/custom',//这个路由也是唯一的，再layout中使用
    title: '自定义平台',
    sandbox: true,
    url: [
        'http:////127.0.0.1:5500/dist/js/app.js',
        'http:////127.0.0.1:5500/dist/css/app.css',
    ],
    container,
    },
    ...
]);

(3)在基座App.vue提供了layout侧边栏和容器
<template>
  <div id="app">
    <div>
      <layout />
    </div>
    <div class="content" v-loading="loading">
      <div id="container"></div>
      <router-view v-if="!microAppsActive" />
    </div>
  </div>
</template>
其中microAppsActive是指在微应用没有活跃的情况下
使用基座路由
data () {
    return {
        ...
        microAppsActive: false,
    }
},
start({
    ...
    onActiveApps: (activeApps) => {
        this.microAppsActive = (activeApps || []).length;
    }
});

最后一定要记得，在src/layouts/BasicLayout/index.vue中去配置对应侧边栏
<el-menu>
    <el-submenu index="/custom">
    <template slot="title">custom</template>
    <el-menu-item index="/custom/">自定义首页</el-menu-item>
    <el-menu-item index="/custom/list">自定义列表</el-menu-item>
    <el-menu-item index="/custom/detail">自定义详情</el-menu-item>
    </el-submenu>
</el-menu>

```

### 通信
```
npm install --save @ice/stark-data
------------------------------------

(1)基座通过store和微应用通信

--基座主要是set
import { store } from '@ice/stark-data';

const userInfo = { name: 'Tom', age: 18 };
store.set('language', 'CH'); // 设置语言
store.set('user', userInfo); // 设置登录后当前用户信息

setTimeout(() => {
  store.set('language', 'EN');
}, 3000);

--微应用通过on监听或者get方法获取数据
异步的时候用监听，同步的时候直接get
// 微应用
import { store } from '@ice/stark-data';

// 监听语言变化，因为language异步设置了，所以需要监听
store.on('language', (lang) => {
  console.log(`current language is ${lang}`);
}, true);

// 获取当前用户，同步直接get
const userInfo = store.get('user');


--------------------------
(2)微应用通过event和基座通信
--基座进行监听
// 主应用
import { event } from '@ice/stark-data';

event.on('freshMessage', () => {
  // 重新获取消息数
});

--微应用触发事件
// 微应用
import { event } from '@ice/stark-data';

event.emit('freshMessage');

--todo
带参可以类比eventBus，有待验证，但是八九不离十
this.$EventBus.$emit("updateMsg", this.msg);
this.$EventBus.$on('updateMsg', value=>{this.title= value})

--done
确实和上边eventBus的方法一致，应该就是原有的封装
微应用：
event.emit('freshMessage',"hello");
基座上：
event.on('freshMessage', (value) => {
  // 重新获取消息数
  console.log(value);
});
注意两边event的引入


```

### 样式和脚本隔离
```
样式隔离主要依赖css modules
脚本隔离主要通过沙箱实现
沙箱，通过 with + new Function 的形式，为微应用脚本创建沙箱运行环境
并通过 Proxy 代理阻断沙箱内对 window 全局变量的访问和修改。

使用的时候，单个AppRoute中
{
    sandbox: true,//沙箱的使用
    name: 'custom',
    activePath: '/custom',
    title: '...',
    url: [...],
    container,
},

```

### 性能优化
```
微应用资源的预加载
start({
    prefetch:true,//预加载
    ...
})

```
