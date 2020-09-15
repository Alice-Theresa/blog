---
title: React Native 简易路由 续
date: 2020-09-26
tags: React Native
---

<img src="/images/2020/reactNativeRouter2/cover.jpg">

对简易路由进行升级，加入参数回传功能，同样的，将支持原生、RN之间的回传

### 重构

将Navigator单独抽离，新增构造方法，每一个页面都将拥有独自的navigator，screenID为唯一字符串

```javascript
constructor(screenID, moduleName) {
  this.screenID = screenID
  this.moduleName = moduleName
  this.resultListeners = undefined
}
```

异步的push方法

```javascript
push = async (component, options) => {
  NavigationBridge.push(component, options)
  return await this.waitResult()
}
```

waitResult返回一个Promise，暂存并等待pop回来的时候调用

```javascript
waitResult() {
  return new Promise((resolve) => {
    const listener = (data) => {
      resolve(['ok', data])
      this.resultListener = undefined
    }
    listener.cancel = () => {
      resolve(['cancel', null])
      this.resultListener = undefined
    }
    this.resultListener = listener
  })
}
```

pop回来的时候根据结果来执行或取消

```javascript
execute(data) {
  this.resultListener(data)
}

unmount = () => {
  this.resultListener.cancel()
}
```

为了实现参数回传，需要增加一个setResult的桥接方法，将数据写入页面所对应栈的Model

```javascript
setResult(data) {
  NavigationBridge.setResult(data)
}
```

### 监听回调通知

原生路由调用pop后将会把结果发送给RN，这里为每一个页面加入监听，根据screenID来找到需要回调结果的页面，result_type来判断用否有传值

另外将navigator注入，可直接从props中获取

```javascript
const withNavigator = (moduleName) => {
  return (WrappedComponent) => {
    const component = (props, ref) => {
      const { screenID } = props
      const navigator = new Navigator(screenID, moduleName)
      useEffect(() => {
        const subscription = EventEmitter.addListener('NavigationEvent', (data) => {
          if (data['screen_id'] === screenID && data['event'] === 'component_result') {
            if (data['result_type'] === 'cancel') {
              navigator.unmount()
            } else {
              navigator.execute(data['result_data'])
            }
          }
        })
        return () => {
          subscription.remove()
        }
      }, [])
      const injected = {
        navigator
      }
      return <WrappedComponent ref={ref} {...props} {...injected} />
    }
    return React.forwardRef(component)
  }
}
```

注册前调用

```javascript
let withComponent = withNavigator(appKey)(component)
AppRegistry.registerComponent(appKey, () => withComponent)
```

### 页面栈

为UIViewController新增一个分类

每次新建页面的时候都会随机生成一个screenID，这里用iOS的NSUUID方法来生成

```objc
@interface UIViewController (ALC)

@property (nonatomic, copy, readonly) NSString *screenID;

- (void)didReceiveResultData:(NSDictionary *)data type:(NSString *)type;

@end
```

每个tab都有独立的栈，为ALCNavigationManager新增栈以及对应的方法

```objc
@property (nonatomic, strong, readonly) NSMutableDictionary<NSString *, NSMutableArray<ALCStackModel *> *> *stacks;

- (void)push:(UINavigationController *)nav vc:(UIViewController *)vc;
- (void)clear;
```

其中出入栈的数据模型

```objc
@interface ALCStackModel : NSObject

@property (nonatomic, copy) NSString *screenID;
@property (nonatomic, copy) NSDictionary *data;

- (instancetype)initWithScreenID:(NSString *)screenID;

@end
```

自定义UINavigationController，用didShowViewController监听页面的变化，将每一次变化的viewController入栈

```objc
- (void)navigationController:(UINavigationController *)navigationController
       didShowViewController:(UIViewController *)viewController
                    animated:(BOOL)animated {
    [[ALCNavigationManager shared] push:navigationController vc:viewController];
}
```

如果入栈的viewController其screenID在栈中不存在，为push操作，否则为pop

如果pop的时候栈顶没有设置数据，即为取消返回，否则传值

```objc
- (void)push:(UINavigationController *)nav vc:(UIViewController *)vc {
    ALCStackModel *model = [[ALCStackModel alloc] initWithScreenID:vc.screenID];
    NSMutableArray *stack = [self.stacks valueForKey:nav.screenID];
    if (![stack containsObject:model]) {
        [stack addObject:model];
    } else if (stack.count > 1) {
       NSUInteger index = [stack indexOfObject:model];
       ALCStackModel *last = stack.lastObject;
       [stack removeObjectsInRange:NSMakeRange(index + 1, self.stacks.count - 1)];
       if (last.data) {
           [vc didReceiveResultData:last.data type:@"ok"];
       } else {
           [vc didReceiveResultData:@{} type:@"cancel"];
       }
    }
}
```

在ALCNativeViewController中实现 setResult: 方法，设置参数到栈顶
在ALCReactViewController中实现 didReceiveResultData:type: 方法，向React Native发送参数

### 使用

<font size=4>**从RN页返回传参，不管上一页是什么**</font>

```javascript
props.navigator.setResult({ rn_key: "rn_value" })
props.navigator.pop()
```

<font size=4>**从原生页返回传参，不管上一页是什么**</font>

```objc
[self setResult:@{@"native_key": @"native_value"}];
[self.navigationController popViewControllerAnimated:YES];
```

<font size=4>**RN页获取结果**</font>

```javascript
const resp = await props.navigator.push('Detail')
```

<font size=4>**原生页获取结果**</font>

重写方法

```objc
- (void)didReceiveResultData:(NSDictionary *)data type:(NSString *)type
```
