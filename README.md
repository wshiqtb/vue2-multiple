# 自动构建多入口页面
本项目有vue-cli构建，稍微新增部分文件和代码，以最精简的方式展现自动构建多入口页面的原理
___

## 需要的包
 `glob`
____

## 理论原理
通过glob匹配对应目录的入口文件，动态创建entry入口的json数据和htmlwebpackplugin的数据，以达到多入口的目的
____
+ 创建入口目录（src/views/**）
+ `build/utils.js` 中添加辅助函数（匹配入口文件，构建json数据）
+ `build/webpack.base.conf.js` 中修改entry和htmlwebpackplugin的配置
+ `build/webpack.dev.conf.js, build/webpack.prod.conf.js`  中注释掉原来的htmlwebpackplugin配置

## 关键代码
主要文件和修改的代码

### 新增入口页面
注意项，js和html的名字要一样，因为入口和页面在htmlwebpackplugin中会进行动态匹配，过滤非当前入口的chunks
```
|--- src
    |--- views // 入口页面和入口js
      |--- admin 
          |--- App.vue #vue根挂载点
          |--- admin.js #入口js
          |--- admin.html #入口html模版
      	|--- index
          |--- App.vue
            |--- index.js
            |--- index.html
```
______

### `build/utils.js` 
```
...
const HtmlWebpackPlugin = require('html-webpack-plugin')
const glob = require('glob')
...
// 返回匹配文件的map, {filename: filepath, ...}
exports.getEntries = (globPath, options)=>{
  var entries = {};

  glob.sync(globPath,options).forEach(entry=>{
    var basename, extname;
    extname = path.extname(entry);
    basename = path.basename(entry,extname);
    entries[basename] = entry;
  })

  return entries;
}

// 返回htmlwebpackplugin插件配置
exports.generHtmlWebpackPlugins = (globPath, options)=>{
  var htmlwp = [],
      pages = exports.getEntries(globPath,options);

  Object.keys(pages).forEach(name=>{
    htmlwp.push(new HtmlWebpackPlugin({
      filename: path.resolve(__dirname, `../dist/${name}.html`),
      template: pages[name],
      inject: true,
      excludeChunks: Object.keys(pages).filter(item => {
        return (item != name)
      })
    }))
  })

  return htmlwp;
}
```
___

### `build/webpack.base.conf.js`
>这里如要注意 `./src/views/**/*.js` 和 `./src/views/**/*.html` 的路径，它们都是以执行脚本命令的目录（一般为项目根目录）为基准，而不是以build目录为基准。
```
...
const webpackConfig = {
  ...
  entry: utils.getEntries('./src/views/**/*.js'),
  ...
}
...
// 自定义多入口
webpackConfig.plugins = webpackConfig.plugins || []
webpackConfig.plugins = webpackConfig.plugins.concat(utils.generHtmlWebpackPlugins('./src/views/**/*.html'))
...
```
____

### `build/webpack.dev.conf.js, build/webpack.prod.conf.js`
> 这两个文件只需要注释掉htmlwebpackplugins的插件部分代码即可
> 如果想给prod的htmlwebpackplugins指定不同的配置，则可修改utils的generHtmlWebpackPlugins为只返回配置参数，将`build/webpack.base.conf.js`中的htmlwebpackplugins配置在`build/webpack.prod.conf.js,build/webpack.dev.conf.js`中进行特定参数合并，再new。

### 创建不同页面的路由
在router中创建不同入口页面的路由，并在views的入口js中引入对应的路由
```
|--- src
  |--- router
	  |--- admin.js
	  |--- index.js
```
**`views/admin/admin.js`**
```
...
import router from '../../router/admin'
...
```
**`views/index/index.js`**
```
...
import router from '../../router/index'
...
```

-----
最后 `npm start` 即可查看效果，祝您工作顺利，生活愉快！