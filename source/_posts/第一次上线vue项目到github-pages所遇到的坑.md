---
title: 第一次上线vue项目到github-pages所遇到的坑
date: 2019-02-16 16:11:06
tags:
---
<font size="5">[项目地址](http://s1mple.site/travel-project/#/)</font>
上线vue项目到gh-pages其实很简单，只需要将项目打包好的dist目录上传到仓库的gh-pages上即可
```bash
# 强制添加dist文件夹，因为.gitignore文件中定义了忽略该文件
git add -f dist

# 提交到本地暂存区
git commit -m 'Initial the page of project'

# 部署dist目录下的代码
git subtree push --prefix dist origin gh-pages
```
没想到遇到这么多小问题。

### 坑一
这个项目是很早之前在atom编辑器上写的，今天用vs-code的控制台打包，没想到竟然报了这个错
```bash
There are multiple modules with names that only differ in casing.
# 有多个模块同名仅大小写不同
This can lead to unexpected behavior when compiling on a filesystem with other case-semantic.
# 这可能导致在一些文件系统中产生不是预期的行为
Use equal casing. 
# 使用唯一的写法
```
就是说我的组件名或文件名和引用不一致，我寻思也妹错啊，改了之后还是报错，然后重启了编辑器，再打包。。好了。。。真是无语

### 坑二
这个坑也是因为坑一才踩上的，因为一直坑一调不好，我就把`import Vue from 'vue'` 改成了 `import Vue form 'Vue'`
然后运行dev的时候报了这个错
```bash
[Vue warn]: You are using the runtime-only build of Vue where the template compiler is not available. Either pre-compile the templates into render functions, or use the compiler-included build.
```
因为vue有两种形式的代码 compiler（模板）模式和runtime模式（运行时），vue模块的package.json的main字段默认为runtime模式， 指向了"dist/vue.runtime.common.js"位置。当为runtime模式时，是不允许在vue实例中写template的
```js
// compiler
new Vue({
  el: '#app',
  router: router,
  store: store,
  template: '<App/>',
  components: { App }
})

//runtime
new Vue({
  router,
  store,
  render: h => h(App)
}).$mount("#app")
```
默认vue-cli在webpack中写了import compiler模式的vue的alias
```js
 resolve: {
    extensions: ['.js', '.vue', '.json'],
    alias: {
      'vue$': 'vue/dist/vue.esm.js',
      '@': resolve('src'),
      'styles': resolve('src/assets/styles'),
      'common': resolve('src/common')
    }
  }
```
当`import Vue from 'vue'`时就引入的是compiler模式的vue文件，而我乱改把'vue'改成了'Vue'，所以就报错

### 坑三之坑中之坑
其实上面两个都是小问题，没一会就解决了，这个坑才是大问题。
>在将Vue应用部署到gh-pages分支后，可能会出现部分资源无法加载的问题，原因就在于vue中的webpack配置在打包时其publicPath为根路径，如果该静态页在服务器中被访问则不会出现以上问题。在github解析时如果按照根路径解析会出错，因此在github上部署静态页时可以考虑将publicPath设置为当前目录，即 publicPath: './' 。

当我将dist上传之后，打开网站，发现是一片空白，检查network发现打包的js和css文件都正常加载，但我的json数据文件没有加载，所以页面是一片空白。
最初做项目时使用的是本地假数据文件，为了上线我将数据文件也上传了github，并将axios获取的文件地址改成了线上地址：
`axios.get(https://github.com/459605823/travel-project/blob/gh-pages/static/mock/city.json)`，但还是一片空白。
然后在mounted钩子中加入调试信息，发现mounted根本没有执行，也就是根本没有访问项目对应的页面，之后就是百度。。
最后发现是vue-router的问题，因为我使用了[vue-router的history模式](https://router.vuejs.org/zh/guide/essentials/history-mode.html#%E5%90%8E%E7%AB%AF%E9%85%8D%E7%BD%AE%E4%BE%8B%E5%AD%90)
```js
export default new Router({
  mode: 'history',
  routes: [
    {
      path: '/',
      name: 'Home',
      component: Home
    },
    {
      path: '/city',
      name: 'City',
      component: City
    },
    {
      path: '/detail/:id',
      name: 'Detail',
      component: Detail
    }
  ],
  // 对于所有路由导航，简单地让页面滚动到顶部。
  scrollBehavior (to, from, savedPosition) {
    return { x: 0, y: 0 }
  }
})
```
当使用这种模式，因为应用是单页面客户端应用，如果后台没有正确的配置，在浏览器访问就会返回404。所以要么使用默认的hash模式，要么就需要在服务端增加一个覆盖所有情况的候选资源：如果URL匹配不到任何静态资源，则应该返回同一个index.html页面，这个页面就是app所依赖的页面。
因为是上传github-pages，无法修改服务器端，所以就只能乖乖改回hash了，改了之后，项目就可以正常访问了。

