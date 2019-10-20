---
layout: post
title: "App 架构 -- MVVM 分析"
---

分析 [App 架构](https://objccn.io/products/app-architecture) 这本书“构建迷你播放器 - MVVM-C” 章节。完整源码在 [Recordings-MVVM-C](https://github.com/songzhou21/app-architecture/tree/master/Recordings-MVVM-C)。我做了些修改，以适配 RxSwift (5.0.0) 版本。

# Model

除了 `Item+Rx.swift` 文件外没有 import RXSwift，用最基本的比如 Notification 发出通知。添加/删除/修改一个 item，都会发出 `Store.changedNotification` 通知。

## Store
有一个 `static let changedNotification = Notification.Name("StoreChanged")` 通知。

## Item
修改名称时会发送 `Store.changedNotification`。

## Folder
继承自 Item，表示一个文件夹对象。

添加/删除里面的一个 item，会发送 `Store.changedNotification`。

## Recording
继承自 Item，表示一个录音对象

nothing special

## Item+Rx
这里 import 了 RxSwift，做了从 Notification 到 Observable 的转换。

changeObservable:
1. 修改 item，item 被添加时的 Observable。
2. folder 的 Observable，在添加/删除其中的 item 时。

如果自身是 Folder 类型，因为 Folder 是 Item 的子类，那么 1，2 情况下都会有 Observable。


```swift
var changeObservable: Observable<()> {
	return NotificationCenter.default.rx.notification(Store.changedNotification).filter { [weak self] (note) -> Bool in
		guard let s = self else { return false }
		/// 1
		if let item = note.object as? Item, item == s, !(note.userInfo?[Item.changeReasonKey] as? String == Item.removed) {
			return true
			/// 2
		} else if let userInfo = note.userInfo, userInfo[Item.parentFolderKey] as? Folder == s {
			return true
		}
		return false
	}.map { _ in () }
}

```

这里面 1，2 都会返回 true 是因为：
1. 在修改 item，item 被添加的时候，能从 item 得到这个 Observable 的这些事件。
2. 而且也能从 item 所在的 folder 里得到添加/删除 item 这些事件的 Observable。


deletedObservable:

item 被删除时的 Observable

```swift
var deletedObservable: Observable<()> {
	return NotificationCenter.default.rx.notification(Store.changedNotification).filter { [weak self] (note) -> Bool in
		guard let s = self else { return false }
		if let item = note.object as? Item, item == s, note.userInfo?[Item.changeReasonKey] as? String == Item.removed {
			return true
		}
		return false
	}.map { _ in () }
}
```

# ViewModel

## FolderViewModel

`let folder: BehaviorRelay<Folder>`

`BehaviorRelay` 真实世界和 RXSwift 的桥接，可以订阅值，也可以手动向它发送值，"Relay" 的意思是它不会产生 “error” 或 "completed" 事件。

这里有一个 `folderUntilDeleted: Observable<Folder?>` 的 Observable。

```swift
init(initialFolder: Folder = Store.shared.rootFolder) {
	folder = BehaviorRelay(value: initialFolder)
	
	folderUntilDeleted = folder.asObservable()
	/// 每次设置一个新的 folder 都会创建一个 observable，需要切换到最新的 observable，否则还会订阅之前 folder 的 observable
	.flatMapLatest { currentFolder in
		/// Start by emitting the initial value
		Observable.just(currentFolder)
		// Re-emit the folder every time a non-delete change occurs
		.concat(currentFolder.changeObservable.map { _ in currentFolder })
		// Stop when a delete occurs
		.takeUntil(currentFolder.deletedObservable)
		// After a delete, set the current folder back to `nil`
		.concat(Observable.just(nil))
	}.share(replay: 1) /// 观察者共享这个 observable，订阅时重发最新的一个元素，否则每次订阅都会创建一个 observable。
}
```

`var navigationTitle: Observable<String>`

对 `folderUntilDeleted` 做一个转换成导航栏标题

`var folderContents: Observable<[AnimatableSectionModel<Int, Item>]> `

对 `folderUntilDeleted` 做一个转换成 Folder tableView 的数据


## RecordViewModel

`let duration = BehaviorRelay<TimeInterval>(value: 0)`

播放时长

`var timeLabelText: Observable<String?> `

播放时长格式化时间

```swift
func recorderStateChanged(time: TimeInterval?) {
	if let t = time {
		duration.accept(t)
	} else {
		dismiss?()
	}

}
```
修改播放时长

## PlayVieModel

`private let recordingUntilDeleted: Observable<Recording?>`

录音对象的 observable。

```swift
init() {
	recordingUntilDeleted = recording.asObservable()
	// Every time the folder changes
	.flatMapLatest { recording -> Observable<Recording?> in
		guard let currentRecording = recording else { return Observable.just(nil) }
		// Start by emitting the current recording
		return Observable.just(currentRecording)
		// Re-emit the recording every time a non-delete change occurs
		.concat(currentRecording.changeObservable.map { _ in recording})
		// Stop when a delete occurs
		.takeUntil(currentRecording.deletedObservable)
		// After a delete, set the current recording back to `nil`
		.concat(Observable.just(nil))
	}.share(replay: 1)
	///...
}
```

`let playState: Observable<Player.State?>`

播放状态

```swift

/// 因为返回的是 Observable，需要切换到最新 Observable，所以用 `flatMapLatest`
playState = recordingUntilDeleted.flatMapLatest { [togglePlay, setProgress] recording throws -> Observable<Player.State?> in
	guard let r = recording else {
		return Observable<Player.State?>.just(nil)
	}
    /// 创建一个 Observable
	return Observable<Player.State?>.create { (o: AnyObserver<Player.State?>) -> Disposable in
		guard let url = r.fileURL, let p = Player(url: url, update: { playState in
			o.onNext(playState)
		}) else {
			o.onNext(nil)
			return Disposables.create {}
		}
        
        
		o.onNext(p.state)
		let disposables = [
		togglePlay.subscribe(onNext: {
			p.togglePlay()
		}),
		setProgress.subscribe(onNext: { progress in
			p.setProgress(progress)
		})
		]
		return Disposables.create {
			p.cancel()
			disposables.forEach { $0.dispose() }
		}
	}
}.share(replay: 1)
```

界面相关的 Observable

```swift

var navigationTitle: Observable<String> {
    return recordingUntilDeleted.map { $0?.name ?? "" }
}
var hasRecording: Observable<Bool> {
    return recordingUntilDeleted.map { $0 != nil }
}
var noRecording: Observable<Bool> {
    return hasRecording.map { !$0 }.delay(.seconds(0), scheduler: MainScheduler())
}
var timeLabelText: Observable<String?> {
    return progress.map { $0.map(timeString) }
}
var durationLabelText: Observable<String?> {
    return playState.map { $0.map { timeString($0.duration) } }
}

...
```
# ViewController
viewController 会持有 viewModel，并建立 viewModel 与 UI 的绑定。

view 的 action 通过 viewModel 进行。

## ViewController 持有 ViewModel
`let viewModel = FolderViewModel()`

## ViewModel 与 UI 绑定
`viewModel.navigationTitle.bind(to: rx.title).disposed(by: disposeBag)`

## View action 到 ViewModel
```swift
@IBAction func createNewFolder(_ sender: Any) {
		modalTextAlert(title: .createFolder, accept: .create, placeholder: .folderName) { string in
			self.viewModel.create(folderNamed: string)
			self.dismiss(animated: true)
		}
	}
```
			
View ->(发送 action) ViewController ->(更改 viewModel) ViewModel ->(更改 model) -> Model


Model ->(观察 model) ViewModel ->(发送展示变更) ViewController ->(更改 View) View
			

