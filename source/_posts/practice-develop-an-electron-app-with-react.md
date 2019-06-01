title: 开发一个React + Electron应用
date: 2019-02-16 10:50:06
tags: 
    - react
    - electron
    - practice
categories:
    - 实践
---

最近用React + Electron开发了一个RSS阅读器，开源在：[https://github.com/breeze2/breader](https://github.com/breeze2/breader)，这里记录一下大致的开发过程。

![Breader](https://breeze2.github.io/breader/screenshot.jpg)

<!--more-->

## 初始化

### 创建项目
以普通的React应用做基础，一步步初始化项目。预先安装[yarn](https://yarnpkg.com/)工具，用`yarn`来创建一个React应用项目，假设名字叫`demo`，再引入Electron依赖。
```cmd
$ cd /PATH/TO/PROJECTS
$ yarn create react-app YOUR_APP_NAME
$ cd /PATH/TO/PROJECTS/YOUR_APP_NAME
$ yarn add electron --dev
```

### 配置入口文件

创建项目后，大致的目录结构如下：
```
|-->demo
    |-->node_modules
        |--...
    |-->public
        |--index.html
        |--...
    |-->src
    |--package.json
    |--...
```


一般来说，React应用（测试环境下）的入口文件就是`public/index.html`，而Electron应用的入口文件最好也放在`public`目录下，并命名为`electron.js`（这样[electron-builder](https://github.com/electron-userland/electron-builder)可以自动识别）。

先在`package.json`中lectron应用的入口文件，添加`main`配置：

```json
{
  ...
  "main": "public/electron.js",
  ...
}
```

`public/electron.js`代码：

```js
const { app, BrowserWindow } = require('electron')
const path = require('path')

let mainWindow

function createWindow() {
    mainWindow = new BrowserWindow({
        width: 960,
        height: 600,
    })
    mainWindow.loadFile('index.html')
    mainWindow.webContents.openDevTools()
    mainWindow.on('closed', function () {
        mainWindow = null
    })
}

app.on('ready', createWindow)

app.on('window-all-closed', function () {
    if (process.platform !== 'darwin') {
        app.quit()
    }
})

app.on('activate', function () {
    if (mainWindow === null) {
        createWindow()
    }
})
```

在执行执行`yarn electron ./`命令就可以启动Electron应用了，不过React应用还没启动起来。

### 启动应用
一般用`yarn react-scripts start`命令，就可以将React应用挂载在本地3000端口上（ http://localhost:3000 ），用于开发调试。要用React结合Electron一起开发调试，先安装[electron-is-dev](https://github.com/sindresorhus/electron-is-dev)来识别当前是否开发环境。

```cmd
$ yarn add electron-is-dev
```

一般开发环境是加载`http://localhost:3000/`，正式环境是加载`../build/index.html`文件，所以修改`public/electron.js`代码（`createWindow`函数内）：

```js
const isDev = require('electron-is-dev')

...
--    // mainWindow.loadFile('index.html')
++    if (isDev) {
++        mainWindow.loadURL('http://localhost:3000/')
++    } else {
++        mainWindow.loadFile(path.join(__dirname, '/../build/index.html'))
++    }
...
```

然后，在项目目录下，打开两个终端，一个执行`yarn react-scripts start`，一个执行`yarn electron ./`，就可以开发调试React + Electron应用了。
不用羡慕[electron-vue](https://github.com/SimulatedGREG/electron-vue)，可以一句命令可以直接启动，其实原理是一样的。
要实现一句命令启动也不难，只要把两句命令合到一起执行就可以了。先安装工具库：

```js
$ yarn add concurrently --dev
$ yarn add wait-on --dev
```

修改`package.json`，添加一个`electron-dev`脚本命令：

```json
{
  ...
  "scripts": {
    ...
    "electron-dev": "concurrently \"BROWSER=none react-scripts start\" \"wait-on http://localhost:3000 && electron .\"",
    ...
  }
  ...
}
```

这样，执行一句`yarn electron-dev`就可以启动React + Electron应用了。

## JS运行时
一般来说，浏览器提供一种JS运行时，NodeJS提供另一种JS运行时，Electron则是将这两种运行时结合到一起来提供。不过，这两种运行时并不是完美地和睦相处，比如说使用[webpack](https://webpack.js.org/)时，通过webpack打包的JS代码中，不能直接通过`import`关键字或者`require`函数来引入NodeJS提供的功能接口，因为webpack覆盖了NodeJS自带的`require`函数。
不过Electron中，`window`对象的`require`属性会映射到NodeJS自带的`require`函数上，比如要调用NodeJS提供的[http](https://nodejs.org/dist/latest-v10.x/docs/api/http.html)接口，可以这样写：

```js
const http = window.require('http')
```

## 重新编译NodeJS扩展

通常，纯JS代码实现的工具库都可以在Electron环境中运行。不过有些NodeJS的工具库并不是纯JS代码实现的，比如[node-sqlite3](https://github.com/mapbox/node-sqlite3)。`node-sqlite3`是C++编写，算是NodeJS的扩展，而不是单纯的工具库。`node-sqlite3`需要针对Electron环境重新编译，才能在Electron环境中运行。

### electron-rebuild
[electron-rebuild](https://github.com/electron/electron-rebuild)是专门Electron环境针对重新编译NodeJS扩展的一个工具。
在项目目录下，安装electron-rebuild：

```cmd
$ yarn add electron-rebuild --dev
```

比如，重新编译node-sqlite3，只需要：

```cmd
$ yarn electron-rebuild -f -w sqlite3
```

在MacOS系统上就是这么简单，在Windows系统就复杂一些。

### Windows系统上重新编译NodeJS扩展

在Windows系统上也一样可以使用electron-rebuild工具，但必须预先配置好编译环境。

首先，安装window编译工具：

```cmd
$ npm install --global --production windows-build-tools
$ npm config set msvs_version 2017
```

然后，下载并安装[Python2](https://www.python.org/downloads/release/python-2715/)，必须Python2不能Python3，如果同时安装了Python2和Python3，须给`npm`指定Python2可执行文件路径，避免调用了Python3：

```cmd
$ npm config set python /path/to/executable/python2
```

这样，才能在Windows系统上调用electron-rebuild。如果electron-rebuild执行过程中，遇上`CONNECT ERROR`可能是网络问题，可以换成淘宝源再试试。

所以尽量少用NodeJS扩展，可以免去跨平台时重新编译的麻烦。

## 应用打包

应用开发完成后，需要打包成`dmg`或`exe`等安装文件，可以使用[electron-builder](https://www.electron.build/)

### electron-builder
electron-builder有很多配置选项，来调用各式各样的程序打包。这里只简单介绍一下MacOS系统安装程序和Windows系统NSIS安装程序的打包配置（[NSIS](https://sourceforge.net/projects/nsis/)是一个开源的 Windows 系统下安装程序制作工具）。

修改`package.json`，添加`build`配置：

```json
{
  ...
  "homepage": "./", // 因为最后React应用引用的JS、CSS等资源都是本地的，只要用当前的相对路径即可
  "build": {
    "appId": "com.xxx.app", // 应用ID
    "npmRebuild": true, // 打包前是否重新编译NodeJS扩展
    "mac": {
      "category": "news", // 应用分类
      "icon": "build/icon.png" // 应用icon路径
    },
    "win": {
      "icon": "build/icon.png", // 应用icon路径
      "target": "nsis" // Windows安装文件的目标类型
    },
    "nsis": {
      "allowToChangeInstallationDirectory": true, // 是否允许修改安装路径
      "allowElevation": false, // 是否允许提升权限
      "createDesktopShortcut": true, // 是否创建桌面快捷方式
      "menuCategory": true, //是否在菜单栏创建分类
      "oneClick": false // 是否一键安装
    },
    "files": [
      "build/**/*" // 引入的文件
    ]
  }
  ...
}
```

然后，就可以打包Electron应用了：

```cmd
$ yarn react-scripts build // 先用webpack打包React应用到`build`目录下
$ yarn eletron-builder // 再用eletron-builder打包Electron应用
```


当然，正式打包还需要[代码签名](https://electronjs.org/docs/tutorial/code-signing)，还有更多配置，请查看[electron-builder配置说明](https://www.electron.build/configuration)。

## 参考
* [From React to an Electron app ready for production](https://medium.com/@kitze/%EF%B8%8F-from-react-to-an-electron-app-ready-for-production-a0468ecb1da3)