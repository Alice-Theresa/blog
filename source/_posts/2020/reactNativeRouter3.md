---
title: React Native 简易路由 完
date: 2020-10-24
tags: React Native
---

<img src="/images/2020/reactNativeRouter3/cover.jpg">

### 封装

我们将之前的路由封装成一个模块，这样子我们就可以通过npm包的方式在多个项目中重复使用，另外模块采用TypeScript

### 全局样式

定义一个全局样式的Model

```ts
interface GlobalStyle {
  backIcon?: {uri: string} // 设置返回图标
  hideNavigationBarShadow?: boolean // 隐藏导航栏底部线
  hideBackTitle?: boolean // 是否隐藏返回按钮旁边的文字
  navigationBarColor?: string // 导航栏背景颜色
  navigationBarItemColor?: string // 导航栏item颜色

  tabBarColor?: string // tabbar背景颜色
  tabBarItemColor?: string // tabbar选中颜色
  tabBarDotColor?: string // tabbar圆点颜色
}
```

通过桥传给原生，原生使用该配置来设定全局样式，同时需要一些辅助函数，例如解析16进制字符串到对应的颜色

```ts
export const setStyle = (style: GlobalStyle) => {
  NavigationBridge.setStyle(style)
}
```

### 路由

声明一个路由，包含一个字典用于存储注册的页面以及对应的路径名

```ts
let active = false
let hasHandleInitialURL = false
const configs = new Map<string, string>()

class Router {
  static uriPrefix?: string

  static open = async (path: string) => {
    if (!path) {
      return
    }
    path = path.replace(Router.uriPrefix!, '')
    if (!path.startsWith('/')) {
      path = `/${path}`
    }
    const [pathName, queryString] = path.split('?')
    const moduleName = configs.get(pathName)
    if (!moduleName) {
      return
    }
    const queryParams = (queryString || '').split('&').reduce((result: any, item: string) => {
      if (item !== '') {
        const nextResult = result || {}
        const [key, value] = item.split('=')
        nextResult[key] = value
        return nextResult
      }
      return result
    }, {})
    const navigator = await Navigator.current()
    if (navigator) {
      navigator.push(moduleName, queryParams)
    }
  }

  addRoutePath(moduleName: string, routePath: string) {
    configs.set(routePath, moduleName)
  }

  clear = () => {
    this.inactivate()
    configs.clear()
  }

  activate = (uriPrefix: string) => {
    if (!uriPrefix) {
      throw new Error('must pass `uriPrefix` when activate router.')
    }
    if (!active) {
      Router.uriPrefix = uriPrefix
      if (!hasHandleInitialURL) {
        hasHandleInitialURL = true
        Linking.getInitialURL()
          .then(url => {
            if (url) {
              Router.open(url)
            }
          })
          .catch(err => console.error('An error occurred', err))
      }
      Linking.addEventListener('url', this.routeEventHandler)
      active = true
    }
  }

  inactivate = () => {
    if (active) {
      Router.uriPrefix = undefined
      Linking.removeEventListener('url', this.routeEventHandler)
      active = false
    }
  }

  private readonly routeEventHandler = (event: {url: string}) => {
    Router.open(event.url)
  }
}
```

路由支持Deep Link，使用前需要在Xcode中设置好对应的URL Schemes，并激活对应的scheme

```ts
router.activate('alc://')
```

可以通过一个queryString来打开特定的页面并传参，例如

```ts
Router.open('/path/to/somewhere?key=value')
```

会解析成路径 /path/to/somewhere 以及参数 {key: value}

根据路径找到对应的页面并push出来

在终端中运行该命令，让模拟器进行Deep Link跳转

```
xcrun simctl openurl booted "alc://Detail"
```
