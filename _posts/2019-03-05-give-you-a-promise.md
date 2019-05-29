---
title: FBLPromises 源码分析
date: 2019-03-05
category: programming
---

# Intro

Promise 是什么？根据维基百科的定义
> They describe an object that acts as a proxy for a result that is initially unknown, 
> usually because the computation of its value is not yet complete.

很多编程语言都有 Promise 的库，甚至有些语言，如 Javascript，甚至集成到了标准库中。作为 iOS 开发者，也避免不了写大量回调的代码，然而这样直接的写法在复杂的异步场景下，会带来可理解性和可维护性上的挑战。Promise 作为一个比较轻量级的抽象，在 OOP 的包装下，通过一些简单的扩展实现，可以很方便地处理一些异步场景，如`then:`(链式调用)，`all:`(等待多个异步结果)，`retry:`(重试)等。

# FBLPromise

[FBLPromises](https://github.com/google/promises) 就是在 Objective-C 下的一个 Promise 库。下面以它的源码作为分析，其实核心代码只有两个函数。

在 FBLPromise 对象中，有这些实例变量和静态变量。

```objc
static dispatch_queue_t gFBLPromiseDefaultDispatchQueue;

@implementation FBLPromise {
  /** Current state of the promise. */
  FBLPromiseState _state;
  /**
   Set of arbitrary objects to keep strongly while the promise is pending.
   Becomes nil after the promise has been resolved.
   */
  NSMutableSet *__nullable _pendingObjects;
  /**
   Value to fulfill the promise with.
   Can be nil if the promise is still pending, was resolved with nil or after it has been rejected.
   */
  id __nullable _value;
  /**
   Error to reject the promise with.
   Can be nil if the promise is still pending or after it has been fulfilled.
   */
  NSError *__nullable _error;
  /** List of observers to notify when the promise gets resolved. */
  NSMutableArray<FBLPromiseObserver> *_observers;
}


...

@implementation FBLPromise (TestingAdditions)

...

+ (dispatch_group_t)dispatchGroup {
  static dispatch_group_t gDispatchGroup;
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    gDispatchGroup = dispatch_group_create();
  });
  return gDispatchGroup;
}

@end


```

`gFBLPromiseDefaultDispatchQueue` 默认是 mainQueue。`dispatchGroup` 也是静态变量。`state` 就是一个 Promise 所处的状态。`value`, `error` 就是 Promise 最终给出的结果。`pendingObjects` 就不太清楚了，没怎么用到过。`_observers` 下面会讲到。


# State Change

Promise 的 State 是非常重要的属性，下面就看一下 State 是如何被改变的。
![promise_state_change](/static/fbl_promise/FBLPromise.002.jpeg)

首先有三个初始化方法，可以把 Promise 初始化到 Pending, Fulfilled, Rejected state。值得注意的是，在调用 `-fullfill:` 时，如果传入的是 `error` 也会进入到 Rejected State。

# DispatchGroup

一般碰到跟 `dispatch` 相关的字眼，都会特别留意，这说明 Promise 内部会用 GCD[^1] 来处理异步任务。

![promise_dispatch_group](/static/fbl_promise/FBLPromise.003.jpeg)

这里有一个 `static` 修饰的 `dispatchGroup`，通过 lazy initialization 的形式创建出来。因为 Promise 里异步任务会被提交到这个 `dispatchGroup` 里，下面会讲到。调用 `-initPending:` 初始化的时候，会调用 `dispatch_group_enter()`，在 Promise 被 `-fullfill:`, `-reject:`, `-dealloc` 的时候会调用 `dispatch_group_leave()`。

所以说这个 `dispatchGroup` 会管理所有 Promise 提交的异步任务。可以通过 `dispatch_group_wait()` 来得到所有异步任务被执行结束的回调。

```objc
BOOL FBLWaitForPromisesWithTimeout(NSTimeInterval timeout) {
  BOOL isTimedOut = NO;
  NSDate *timeoutDate = [NSDate dateWithTimeIntervalSinceNow:timeout];
  static NSTimeInterval const minimalTimeout = 0.01;
  static int64_t const minimalTimeToWait = (int64_t)(minimalTimeout * NSEC_PER_SEC);
  dispatch_time_t waitTime = dispatch_time(DISPATCH_TIME_NOW, minimalTimeToWait);
  dispatch_group_t dispatchGroup = FBLPromise.dispatchGroup;
  NSRunLoop *runLoop = NSRunLoop.currentRunLoop;
  while (dispatch_group_wait(dispatchGroup, waitTime)) {
    isTimedOut = timeoutDate.timeIntervalSinceNow < 0.0;
    if (isTimedOut) {
      break;
    }
    [runLoop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:minimalTimeout]];
  }
  return !isTimedOut;
}

```

这里指定一个超时时间 `timeout`，在 `while` 循环里通过不断唤醒 `runloop`来处理事件，除非所有任务完成结束（`dispatch_group_wait()` 返回 0 ）或者发生了超时，才跳出循环。

# 链式调用
链式调用涉及到两个最主要的方法，`-observeOnQueue:fullfill:reject:` 和 `-chainOnQueue:chainedFulfill:chainedReject:`。

前者是观察 Promise 的 state，来执行相应的回调（`onFulfill` 和 `onReject`）。如果处于 Pending state，那么会在 `_observers` 里添加一个观察回调，等到这个 promise 被 `-fullfill:` 或 `-reject:` 时，所有的 `_observers` 里的回调都会被执行，当初传递的 `onFulfill` 和 `onReject` 回调也会被调用。

```objc
- (void)observeOnQueue:(dispatch_queue_t)queue
               fulfill:(FBLPromiseOnFulfillBlock)onFulfill
                reject:(FBLPromiseOnRejectBlock)onReject {
...
  @synchronized(self) {
    switch (_state) {
      case FBLPromiseStatePending: {
        if (!_observers) {
          _observers = [[NSMutableArray alloc] init];
        }
        [_observers addObject:^(FBLPromiseState state, id __nullable resolution) {
          dispatch_group_async(FBLPromise.dispatchGroup, queue, ^{
            switch (state) {
              case FBLPromiseStatePending:
                break;
              case FBLPromiseStateFulfilled:
                onFulfill(resolution);
                break;
              case FBLPromiseStateRejected:
                onReject(resolution);
                break;
            }
          });
        }];
        break;
      }
      case FBLPromiseStateFulfilled: {
        dispatch_group_async(FBLPromise.dispatchGroup, queue, ^{
          onFulfill(self->_value);
        });
        break;
      }
      case FBLPromiseStateRejected: {
        dispatch_group_async(FBLPromise.dispatchGroup, queue, ^{
          onReject(self->_error);
        });
        break;
      }
    }
  }
}
```

调用 `-chainOnQueue:chainedFulfill:chainedReject:` 时，会调用 `-observeOnQueue:fullfill:reject` 对自身进行观察，在 fullfill 或 reject 的回调被调用时，如果回调返回值是 promise 对象，则对这个返回值进行观察，在 fullfill 或 reject 的回调被调用时，对新建的 promise 调用 `-fullfill:` 或 `-reject:` 方法。最后返回这个新建的 promise，从而实现链式调用。

相当于在调用 `then:` 方法时，如果观察的回调返回的是一个 promise，那么我们这里会新建一个 promiseA，对 promise 进行观察，在 fullfill 或 reject 被调用时，调用 promiseA 的 fullfill 或 reject 。如果提供了 `chainedFullfill` 或 `chainedReject` 参数，那么会对观察后的回调中的返回值进行处理，此时用户可以返回一个自己创建的 promise，那么会从这个 promise 开始观察，否则就是前面所描述的从调用方这个 promise 观察得到的回调返回值开始观察。这里 `reject:` 分支的思路是一样的，不再多做描述。

```objc
- (FBLPromise *)chainOnQueue:(dispatch_queue_t)queue
              chainedFulfill:(FBLPromiseChainedFulfillBlock)chainedFulfill
               chainedReject:(FBLPromiseChainedRejectBlock)chainedReject {
...

  FBLPromise *promise = [[FBLPromise alloc] initPending];
  __auto_type resolver = ^(id __nullable value) {
    if ([value isKindOfClass:[FBLPromise class]]) {
      [(FBLPromise *)value observeOnQueue:queue
          fulfill:^(id __nullable value) {
            [promise fulfill:value];
          }
          reject:^(NSError *error) {
            [promise reject:error];
          }];
    } else {
      [promise fulfill:value];
    }
  };
  [self observeOnQueue:queue
      fulfill:^(id __nullable value) {
        value = chainedFulfill ? chainedFulfill(value) : value;
        resolver(value);
      }
      reject:^(NSError *error) {
        id value = chainedReject ? chainedReject(error) : error;
        resolver(value);
      }];
  return promise;
}
```

# 小结

这里的 Promise 核心其实就两个函数，通过 `_observsers` 数组来保存回调的和延迟回调的调用，所有的回调都是添加到一个全局的 `dispatchGroup` 里执行，通过新建一个 promise 对象，观察上一个 promise 的回调返回值来实现链式调用。

[^1]: https://en.wikipedia.org/wiki/Grand_Central_Dispatch
