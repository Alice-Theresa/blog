---
title: React Native 简易路由
date: 2020-08-23
tags: React Native
---

<img src="/images/2020/reactNativeRouter/cover.jpg">

React Native有两大路由框架，react-navigation和react-native-navigation
第三方的路由使用起来并没有什么大问题，但对于原生和 RN 页面混合开发的项目来说，并不太友好
因此面对高度复杂的场景，自定义一个的路由会是更好的选择

以iOS为例，定制一个简易的原生路由

将拥有以下功能:
push、pop、popToRoot、present、dismiss、switchTab

同时兼容原生以及RN页面之间的切换

### React Native部分

导航的桥接方法

```javascript
export class Navigatior {
  static push = (component, options) => {
    NavigationBridge.push(component, options)
  }

  static pop = () => {
    NavigationBridge.pop()
  }

  static popToRoot = () => {
    NavigationBridge.popToRoot()
  }

  static present = (component, options) => {
    NavigationBridge.present(component, options)
  }

  static dismiss = () => {
    NavigationBridge.dismiss()
  }

  static switchTab = (index) => {
    NavigationBridge.switchTab(index)
  }
}
```

由于路由在原生端控制，因此项目中的所有页面都需要单独注册，且注册的同时需要将每个页面的样式配置记录下来

```javascript
const registerComponent = (appKey, component) => {
  let options = component.navigationItem || {}
  NavigationBridge.registerReactComponent(appKey, options)
  AppRegistry.registerComponent(appKey, () => component)
}

registerComponent('Home', Home)
registerComponent('Setting', Setting)
registerComponent('Detail', Detail)
registerComponent('Present', Present)
```

React Native各页面中的样式在navigationItem中设定，如导航栏标题、是否隐藏导航栏等

```javascript
Detail.navigationItem = {
  title: 'Detail',
  hideNavigationBar: false
}
```

注册完毕后调用setRoot设置根页面，这里设定为2个TabBar， 各自拥有独立的Navigation

```javascript
NavigationBridge.setRoot({
  root: {
    tabs: {
      children: [
        {
          component: 'Home',
          title: '主页',
          icon: Image.resolveAssetSource(require('./src/image/Home.png'))
        },
        {
          component: 'Setting',
          title: '设置',
          icon: Image.resolveAssetSource(require('./src/image/Profile.png'))
        }
      ]
    }
  }
})
```

### 原生部分

主要有以下4个类
* ALCNativeViewController
* ALCReactViewController
* ALCNavigationBridge
* ALCNavigationManager

ALCNativeViewController和ALCReactViewController继承UIViewController，用于创建原生、RN的页面。

ALCNavigationBridge实现具体桥接的方法，即registerReactComponent、setRoot、push、pop等。

ALCNavigationManager管理路由，这里包含两个字典，分别记录原生、RN的页面，同时提供注册、查找页面的方法。

```objc
@interface ALCNavigationManager : NSObject

@property (nonatomic, strong) RCTBridge *bridge;

@property (nonatomic, strong, readonly) NSMutableDictionary *nativeModules;
@property (nonatomic, strong, readonly) NSMutableDictionary *reactModules;

+ (instancetype)shared;

- (void)registerNativeModule:(NSString *)moduleName forController:(Class)clazz;
- (BOOL)hasNativeModule:(NSString *)moduleName;
- (Class)nativeModuleClassFromName:(NSString *)moduleName;

- (void)registerReactModule:(NSString *)moduleName options:(NSDictionary *)options;
- (BOOL)hasReactModuleForName:(NSString *)moduleName;
- (NSDictionary *)reactModuleOptionsForKey:(NSString *)moduleName;

- (UIViewController *)fetchViewController:(NSString *)pageName params:(NSDictionary *)params;
- (UIImage *)fetchImage:(NSDictionary *)json;

@end
```

在路由跳转时，会查找该页面是否注册过，根据该页面是RN还是原生来创建对应的控制器

```objc
- (UIViewController *)fetchViewController:(NSString *)pageName params:(NSDictionary * __nullable)params {
    BOOL hasNativeVC = [self hasNativeModule:pageName];
    UIViewController *vc;
    if (hasNativeVC) {
        Class clazz = [self nativeModuleClassFromName:pageName];
        vc = [[clazz alloc] initWithModuleName:pageName props:params];
    } else {
        NSDictionary *options = [self reactModuleOptionsForKey:pageName];
        RCTRootView *rootView = [[RCTRootView alloc] initWithBridge:self.bridge moduleName:pageName initialProperties:params];
        vc = [[ALCReactViewController alloc] initWithModuleName:pageName options:options];
        vc.view = rootView;
    }
    return vc;
}
```

### 使用

如果有原生页面，启动时需要在AppDelegate里注册，该原生页面需要继承ALCNativeViewController，路由将自动支持原生页面的导航，这里注册了一个名为NativeViewController的原生页面

```objc
[ALCNavigationManager shared].bridge = [[RCTBridge alloc] initWithDelegate:self launchOptions:launchOptions];
[[ALCNavigationManager shared] registerNativeModule:@"NativeViewController" forController:[ThisIsViewController class]];
```

和跳转RN页面一样，跳转原生时只需要传入原生页面的注册名字

```javascript
<Button
  title="push native"
  onPress={() => {
    Navigatior.push('NativeViewController')
  }}
/>
```

同样的，原生跳转RN，根据RN页面注册的名称来创建即可

```objc
- (void)goToDetail {
    UIViewController *vc = [[ALCNavigationManager shared] fetchViewController:@"Detail" params:nil];
    [self.navigationController pushViewController:vc animated:YES];
}
```
