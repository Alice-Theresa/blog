---
title: React Native 简易路由
date: 2020-08-23
tags: React Native
---

实现以下功能:
push、pop、popToRoot、present、dismiss、switchTab

且同时兼容原生以及RN之间的切换

### React Native部分

由于路由在原生端控制，因此项目中的所有页面都需要单独注册，且注册的同时需要将每个页面的样式配置记录下来
```

```

React Native各页面中的样式在navigationItem中设定，如导航栏标题、是否隐藏导航栏等
```

```

注册完毕后调用setRoot设置根页面，这里设定为2个TabBar， 各自拥有独立的Navigation
```

```

### 原生部分

```

```

ALCNativeViewController和ALCReactViewController继承UIViewController，用于创建原生、RN的页面。
Bridge实现具体桥接的方法，即setRoot、push、pop等。
Manager管理路由，这里包含两个字典，分别记录原生、RN的页面，同时提供注册、查找页面的方法。
```

```

在路由跳转时，会查找该页面是否注册过，根据该页面是RN还是原生来创建对应的控制器
```

```

### 使用

如果有原生页面，启动时需要在AppDelegate里注册，该原生页面需要继承ALCNativeViewController，路由将自动支持原生页面的导航，这里注册了一个名为NativeViewController的原生页面
```

```

和跳转RN页面一样，跳转原生时只需要传入原生页面的注册名字
```

```

同样的，原生跳转RN，根据RN页面注册的名称来创建即可
```

```

这样就实现了一个简单的由原生控制的路由，且能在RN和原生之间无缝切换
