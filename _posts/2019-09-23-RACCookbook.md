---
layout: post
title: "ReactiveCocoa 实战"
---

分享 ReactiveCocoa 在实际使用过程中的一些经验，用案例来展示。

假设读者对 ReactiveCocoa 基本概念都比较熟悉，比如 signal，sequence，一些基本的对 signal 的操作等。
不太熟悉的可以阅读 [halfrost](https://halfrost.com/reactivecocoa_racsignal) 作者的系列文章，从源码层面分析，讲解的很全面。

# 数组操作

## 数据元素中的 signal

一个 viewModels 数组，任意一个 viewModel number 字段值大于 0，则发送 1，否则 0

参考 https://stackoverflow.com/a/19711002/1911562

声明 ViewModel
```objc
@interface ViewModel : NSObject

@property (nonatomic) NSNumber *number;

@end

@implementation ViewModel

@end

@interface ExampleTest : XCTestCase

@property (nonatomic, copy) NSArray *viewModels;

@end
```

创建信号
 ```objc
 @weakify(self)
 /// 初始化 signal
 /// 每次设置 viewModels 的 signal
 ///============================
 RACSignal *enabled = RACObserve(self, viewModels);
 
 enabled = [enabled map:^id _Nullable(NSArray * _Nullable viewModels) {
     /// 这是 sequence，里面的元素是各个 viewModel 的 number 字段返回的 bool 类型 signal，
     /// `startWith` 默认开始的 signal 发送的值是 @(NO)
     /// 如果不设置默认的 signal，在 viewModels 为空时是不会发送任何东西，因为是 `emptySignal` 会直接 completed
     RACSequence *inputSignals = [[viewModels.rac_sequence map:^id _Nullable(ViewModel * _Nullable viewModel) {
         /// RACObserve() 对 self 有引用
         @strongify(self)
         return [RACObserve(viewModel, number) map:^id _Nullable(id  _Nullable number) {
             return @([number integerValue] > 0);
         }];
     }] startWith:[RACSignal return:@(NO)]];
     
     /// 任意一个 viewModel 修改，都需要发送值，而且需要合并其他 viewModel 最新的值
     /// 所以需要用 `combineLatest`
     /// 对 viewModels signal sequence 进行合并成一个 signal
     /// 合并的 signal 发送的值是形如 (0,1) 的 tuple
     RACSignal *ret = [RACSignal combineLatest:inputSignals];
     
     /// 对 tuple 进行 or 操作
     /// (0,1) => 0
     return [ret or];
 }];
 
 /// 因为每次设置 viewModels 都会返回一个 signal，我们要切换到最新的一个
 /// 这里转换之前的 enabled 是高阶信号（信号里面元素是信号）
 enabled = [enabled switchToLatest];
```

测试
```objc
 /// 测试
 ///=========================
 ViewModel *vm1 = [ViewModel new];
 ViewModel *vm2 = [ViewModel new];

 self.viewModels = @[vm1, vm2];
 
 /// 初始化两个 vm
 [[enabled subscribeNext:^(id  _Nullable x) {
     NSLog(@"sub-1 %@", x);
     XCTAssert([x isEqual:@(NO)]);
 }] dispose];
 
 /// 设置其中一个 vm 的值
 vm2.number = @2;
 [[enabled subscribeNext:^(id  _Nullable x) {
     NSLog(@"sub-2 %@", x);
     XCTAssert([x isEqual:@(YES)]);
 }] dispose];
 
 /// 清空
  vm2.number = nil;
  [[enabled subscribeNext:^(id  _Nullable x) {
      NSLog(@"sub-3 %@", x);
      XCTAssert([x isEqual:@(NO)]);
  }] dispose];
 
 /// 设置两个 vm 的值
 vm1.number = @1;
 vm2.number = @2;
 [[enabled subscribeNext:^(id  _Nullable x) {
     NSLog(@"sub-4 %@", x);
     XCTAssert([x isEqual:@(YES)]);
 }] dispose];
```

结果
```
Test Case '-[ExampleTest testViewModelsOnePropertyObservation]' started.
2019-09-23 13:05:14.958947+0800 ReactiveCocoaDemo[21180:1246792] sub-1 0
2019-09-23 13:05:14.961012+0800 ReactiveCocoaDemo[21180:1246792] sub-2 1
2019-09-23 13:05:14.963358+0800 ReactiveCocoaDemo[21180:1246792] sub-3 0
2019-09-23 13:05:14.965047+0800 ReactiveCocoaDemo[21180:1246792] sub-4 1
```
可以看到，初始化的时候发送 `0`，是因为手动在最前面拼接了 `[RACSignal return:@(NO)]` 信号。
