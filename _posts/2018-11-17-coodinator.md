---
layout: post
title: Coordinator Design pattern
date: 2018-11-17
categories: programming
---

# 传统的导航逻辑处理
在 app 开发中，处理逻辑是每天都在做的事情，但如何在一个合适的地方处理逻辑，却讨论的不多。很多人在 `ViewController` 中来处理逻辑，这听起来确实是合适的地方，在 MVC 架构中，也只有 V 这部分听起来像是该做这样事情的地方。但如果在实际开发中真的这样做，会发现很多问题。

一个问题是，`ViewController` 很难复用，这里不是指单个 `ViewController` 的复用问题。单个 `ViewController` 通过 Delegate 的形式暴露出行为，在不管哪个架构中，都是很方便可以达到复用的效果。我这里指的复用是多个 `ViewController` 当成整体的复用问题。

比如一个表单页面（VCA），有个二维码扫描按钮，点击进入二维码扫描页面（VCD），请求得到扫描后的结果，填充到表单。这样两个页面是当成一个整体来看待的，当 push 一个表单页面，点击二维码扫描按钮的行为已经和这个表单页面绑定了，它仅仅是用来服务这个表单页面。如果我们想复用这整个行为中的页面，但我们想在扫描出结果后换个接口请求，我们该怎么办。有一种办法是通过依赖注入的形式传入不同的参数，来控制整个行为，但这样的方式显然是不可维护的，状态多了之后，整体逻辑就会很混乱。

图中的虚线是 Delegate 传递方向。

![001](/static/coordinator/002.jpeg)
传统的导航逻辑



因为在导航这样的交互方式下，ViewController 的导航逻辑是以串形的方式连接在一起，想要拦截修改中间的逻辑就很困难。

# Coordinator 设计模式
![001](/static/coordinator/001.jpeg)

Coordinator 设计模式



Coordinator 设计模式最初在 [KHANLOU](http://khanlou.com/2015/01/the-coordinator/) 的博客上看到，通过引入 Coordinator 对象，各页面通过 Delegate 形式传递到 Coordinator，来统一处理导航逻辑。

我也把上面描述的二维码扫描问题用 Coordiantor 的形式实现了一遍，demo 在 [github][coordinator-demo] 上。可以在图中看到，引入了 `AppCoordinator` 和 `ScanCoordinator` 对象，前者是控制整个 app 的导航行为，后者是仅仅负责首页二维码扫描填充这一行为。`VCA`, `VCD` 等 `ViewController` 通过 Delegate 形式传递到了 `ScanCoordinator` 上，而 `ScanCoordinator` 也通过 Delegate 形式传递到了 `AppCoordinator`。



# 结语

Coordinaotr 所设计的思想就是中央集权的形式来管理各页面的导航行为，而不是各页面以串形的方式处理，大大加强了多个 `ViewController` 作为整个的复用。



[coordinator]: http://khanlou.com/2015/01/the-coordinator/
[coordinator-demo]: https://github.com/songzhou21/CoordinatorDesignPatternDemo