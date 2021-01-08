---
title: iOS基础-Push&Pop
date: 2017-12-25 14:25:56
categories:
  - iOS
---

# Push & Pop

在研究自定义导航栏时，发现的这一系列问题。

- 项目中一些不太好的写法改正
- 侧滑返回手势失效的问题
- Push & Pop 动画效果异常的问题

<!-- more -->

## 项目中一些代码的改正

1.通常在一个项目中，是需要自定义一个导航栏的基类来管理导航栏的。所以：继承 UINavigationController 创建 HDNavigationViewController，几个 TabbarVC 的导航栏都使用 HDNavigationViewController。重写 UINavigationController 的 push 方法

```
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated {
	if (self.childViewControllers.count==1) {
		viewController.hidesBottomBarWhenPushed = YES; //viewController是将要被push的控制器
	}
	[super pushViewController:viewController animated:animated];
}
```

这样写保证从 RootVCpush 的时候，将 tabbar 隐藏掉。不必再每次要 push 到下一个页面的时候都写

```
	self.tabBarController.tabBar.hidden = YES;
```

或者

```
	viewController.hidesBottomBarWhenPushed = YES; //viewController是将要被push的控制器
```

## 侧滑返回手势失效的问题

在导航栏中，如果自定义导航栏并且修改了 leftBarButtonItem，或者将导航栏隐藏，那么将会导致侧滑返回失效，为了解决这一问题，可在基类控制器的- viewWillAppear 方法里面中将侧滑手势的代理重新设置为当前 VC 即可。

```
self.navigationController.interactivePopGestureRecognizer.delegate = (id)self;
```

但这样会带来一些问题，稍后再说这个问题，先说如果不采用这种方式，该如何恢复侧滑返回手势。

- 为了不让 pop 代理失效，如果导航栏左侧的只是一个返回按钮，那么可以修改导航栏的 backBarButtonItem 来修改。

```
[self.navigationController.navigationBar setBackIndicatorImage:[UIImage imageNamed:@"nav_back"]];
[self.navigationController.navigationBar setBackIndicatorTransitionMaskImage:[UIImage imageNamed:@"nav_back"]];
UIBarButtonItem *backItem = [[UIBarButtonItem alloc] initWithTitle:@"" style:UIBarButtonItemStylePlain target:nil action:nil];
self.navigationItem.backBarButtonItem = backItem;
```

- 某些需要导航栏隐藏的界面可以选择使用，只将 navigationBar 隐藏，而不是整个导航栏。

```
self.navigationController.navigationBar.hidden = YES;
```

上面两种方式，虽然可以实现侧滑返回，但都有一些局限性或者无法实现需求。下面将介绍，如何使用 leftBarButtonItem 或者隐藏导航栏同时不让侧滑手势失效，即解决设置导航栏手势代理所带来的一些问题。

### 设置手势代理的问题

```
self.navigationController.interactivePopGestureRecognizer.delegate = (id)self;
```

如果在基类 VC 中设置这个导航栏，那么处于根控制器下的 VC 也具有右滑手势，但因为是根控制器，所以没有手势无效，但如果在改控制器视图下，在边缘区域右滑，那么将造成页面卡死状态，无法再点击其他内容，按下 home 键以后再重新打开，会出现各种奇怪的 bug。为了解决这一问题，就需要将根视图的手势给去掉。

```
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    if (self == [self.navigationController.viewControllers firstObject]) {
        self.navigationController.interactivePopGestureRecognizer.enabled = NO;
    } else {
        self.navigationController.interactivePopGestureRecognizer.enabled = YES;
    }
}
```

在基类中，判断是否是根控制器，如果是，那么禁止该手势否则打开。

## Push & Pop 动画效果异常的问题

1.因为有一些界面的导航栏差异性较大，我的通常做法是，将导航栏隐藏，然后自定义一个符合要求的 view 来作为导航栏的 navigationBar。如果使用

```
	self.navigationController.navigationBarHidden = YES;
```

这样的效果能够满足需求，但是如果加入了侧滑手势之后，在侧滑返回的时候，就会出现一个黑条
为了解决这个问题，可在基类的- viewWillDisappear 方法中，添加下面这句代码即可

```
[self.navigationController setNavigationBarHidden:YES animated:animated];
```

保证返回时隐藏。 同时由于页面较为复杂可能存在多种 push 情况，例如
![image.png](http://upload-images.jianshu.io/upload_images/4039772-481066596c6e4a80.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

A->B 需要在 AB 的- viewWillAppear 方法设置
[self.navigationController setNavigationBarHidden:YES animated:YES];
B->D 同 A-B
B->E 需要在 E 的- viewWillAppear 方法中设置
[self.navigationController setNavigationBarHidden:NO animated:YES];

A’->B 需要在 A’的 viewWillAppear 方法中设置
[self.navigationController setNavigationBarHidden:NO animated:YES];
在 B 的- viewWillAppear 方法设置
[self.navigationController setNavigationBarHidden:YES animated:YES];

A’-C 则什么都不需要设置，或者设置为
self.navigationController setNavigationBarHidden:NO animated:YES];

所以我想的是在基类中，添加一个 BOOL 属性为是否显示导航栏，如果需要隐藏某个 VC，那么在该 VC 的 viewdidload 方法中将此 BOOL 属性置为 YES。在基类中的- viewWillAppear 方法中可以这样写，\_isHideNav 就是该 BOOL 属性。同时为了解决 pop 手势失效的问题，可在隐藏导航栏的地方将代理设置一下。

```
if (_isHideNav) {
	[self.navigationController setNavigationBarHidden:YES animated:animated];
self.navigationController.interactivePopGestureRecognizer.delegate = (id)self;
} else {
	[self.navigationController setNavigationBarHidden:NO animated:animated];
}
```

其中 animated 涉及到 push 与 pop 时的过渡动画效果是否显示。
如果
[self.navigationController setNavigationBarHidden:YES animated:NO]; 那么从
显示 pop 隐藏 ，显示 push 隐藏 时会有黑色的条。即提前消失。 其他情况无异常。

如果
[self.navigationController setNavigationBarHidden:NO animated:NO]; 那么从
隐藏 pop 显示 ，隐藏 push 显示 时导航条会提前显示出来。

### 透明导航栏的处理（未完待续）

还有一个问题：如果将某一个页面的导航条设置为透明，就会有一些奇怪的东西。push 到透明的条时，会突然变黑。或 Pop 时突然出现，为了解决这个问题，可以利用方法交换，交换滑动的进度方法，在此拦截进度，将透明度变换设置为渐变的效果。

### 一些发现

研究的过程中，发现的问题。

```
[self.navigationController setNavigationBarHidden:YES animated:animated];
self.navigationController.navigationBar.hidden = YES;
```

这两句话什么区别？

一个是针对导航栏的，一个是针对 NavigationBar 的。隐藏导航栏会导致侧滑手势失效，隐藏 NavigationBar 则不会导致侧滑手势失效。另外，
self.navigationController setNavigationBarHidden:NO animated:YES]; 这个方法还做了一些更多的操作。

## 总结

如何做到兼顾隐藏导航栏保持侧滑返回手势，以及良好的切换动画效果。 在基类控制器中写上如下代码

```
- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    if (_isHideNav) {
        [self.navigationController setNavigationBarHidden:YES animated:animated];
        self.navigationController.interactivePopGestureRecognizer.delegate = (id)self;
    } else {
        [self.navigationController setNavigationBarHidden:NO animated:animated];
    }
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    if (self == [self.navigationController.viewControllers firstObject]) {
        self.navigationController.interactivePopGestureRecognizer.enabled = NO;
    } else {
        self.navigationController.interactivePopGestureRecognizer.enabled = YES;
    }
}
```

如果某个 VC 需要隐藏导航栏，只需要将设置\_isHideNav 属性为 YES 即可。

## 思考 🤔

- 利用方法交换，更改手势，无需重新设置

self.navigationController.interactivePopGestureRecognizer.delegate = (id)self;

- 透明度渐变式的过渡动画
