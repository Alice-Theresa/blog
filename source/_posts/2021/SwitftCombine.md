---
title: Combine，重新定义函数式编程
date: 2021-03-26
tags: [iOS]
---

## Combine

2019年，苹果推出了新一代UI框架SwiftUI，一并出场的还有Combine这个函数式编程库，在此之前，想在iOS平台上使用函数式编程有两个库，分别是RAC以及RxSwift。

Combine作为官方推出的产品，必然是走第三方库的路，让第三方库无路可走，所以项目没有历史负担的话，应优先使用官方库

Combine有三大核心，Publisher、Operator以及Subscriber，其实跟别的函数式编程库没什么大区别，毕竟思想都是一致的

Publisher主要有两种，一种是如Just、Future等一次性的“发布”，另外一种是Subject类型，源源不断的往外“发布”
Subscriber就是用来订阅Publisher的“发布”
而Operator介于两者之间，对“发布”的“内容”进行一系列的操作

## 盒子

为了更好的理解，我们打个比喻，Publisher其实就是一个“盒子”
有Just类型、Future类型等，里面可以装东西，如数字“1”

Just(1)，这就是一个类型为Just的盒子装着数字1，我们对其订阅，则会进行“拆盒子”的动作，拿到盒子里面的东西

**以下代码均忽略sink返回的Cancellable**

```swift
Just(1).sink { value in
    print(value)
}
```
这里会打印数字1

### map & flatMap

函数式编程里面用得最多的估计就是 map 和 flatMap
它们都属于Operator，均可对盒子内部的东西进行改动

```swift
Just(1)
    .map { _ in
        "A"
    }
    .sink { value in
        print(value)
    }
```

```swift
Just(1)
    .flatMap { _ in
        Just("A")
    }
    .sink { value in
        print(value)
    }
```

上面两段代码的最终结果都是一样的，打印字母A

我们可以看到 map 直接返回一个新的结果，因为 map 只会对盒子内部的东西进行操作，然后返回时放回原来的盒子里

而 flatMap 需要你操作完后重新“包装一个盒子”再返回，因此在flatMap中你可以对盒子进行替换，如换成Future等类型的盒子

如果我们在 map 返回的时候像flatMap 那样套多一个盒子会怎样？

```swift
Just(1)
    .map { _ in
        Just("A")
    }
    .sink { value in
        print(value)
    }
```
他会打印一个“盒子”，里面包含字母A

```swift
Just<String>(output: "A")
```

也就是说，经过这样子的 map 操作后，字母A外面会套了两层“盒子”，订阅的时候拆掉最外一层，最后得到一个里面的“盒子”

这里可以联想一下，我们所说的函数式编程，是有点数学的味道，就拿map 对比 flatMap 来说，map 其实是一个“升阶“函数

### switchToLatest

既然有“升阶“，就有”降阶“, switchToLatest 会额外帮你拆掉最外面的“盒子”

<img src="/images/2021/SwitftCombine/example.jpg">

```swift
Just(1)
    .map { _ in
        Just("A")
    }
    .switchToLatest()
    .sink { value in
        print(value)
    }
```

这样我们就能得到想要的结果了，打印出字母A

### eraseToAnyPublisher

由于swift是强静态类型语言，返回对象时需要明确类型，但在Combine中，经过了一系列Operator操作过后，所得到的类型会变得非常臃肿（嵌套多层），如：

```swift
Publishers.FlatMap<Publishers.SetFailureType<Just<Void>, Error>, AnyPublisher<Data, Error>>
```
这只进行了一次flatMap，但已经非常难阅读，使用eraseToAnyPublisher后则会变成

```swift
AnyPublisher<Void, Error>
```
这样子就舒服多了

### publisher & collect

Combine提供了publisher可以快速让数组变成“发布者”

但这个发布者会将数组内的元素“一个个”发出去，订阅时就会离散地接收

```swift
[0, 1, 2, 3, 4]
    .publisher
    .flatMap { value in
        Just(value + 1)
    }
    .collect()
    .sink { value in
        print(value)
    }
```

使用collect进行收集
这样子就会打印出数组：[1, 2, 3, 4, 5]


### error

```swift
enum CustomError: Error {
    case originError
    case mapError
}

let error: CustomError = CustomError.originError

let future = Future<Int, Error>() { promise in
    promise(.failure(error))
}

future
    .mapError { _ in CustomError.mapError }
    .tryCatch { e -> AnyPublisher<Int, Error> in
        print(e) // 此时错误已从 originError 变成 mapError
        return Just(1).setFailureType(to: Error.self).eraseToAnyPublisher()
    }
    .sink { _ in } receiveValue: { value in
        print(value)
    }
```

打印结果为1

Just是没有error的一个“盒子”，error类型为Never，但有时候转换需要为其定一个错误，可以使用setFailureType来达到这一目的

当发生错误时，我们可以这样处理：
1. 使用mapError来转换成别的错误
2. 使用tryCatch来转换成别的Publisher
3. 使用replaceError直接替换成别的值


## 每记

在《每记》项目中，我们使用了SwiftUI + MVVM + Combine，配合三层架构使用（UI层，中间层以及数据层）

由于SwiftUI的特性，让View变得更加纯粹，而不像UIKit中的ViewController那样臃肿，SwiftUI配合MVVM十分顺手
底层数据层以基建库为主，如使用Combine对数据库、网络库封装，让其作为“Publisher”向顶层发送数据和结果

自此我们自下而上地全方位使用Combine，将函数式编程完全融入到项目之中
