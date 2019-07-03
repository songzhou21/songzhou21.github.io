---
layout: post
title: "App 架构"
---

一个录音 app。

# Model

## Item
`Item` 对象是 `Recording` 和 `Folder` 的 superClass。

Item:
- store
- parent: Folder

## 修改

修改 item 名称

```swift
	func setName(_ newName: String) {
		name = newName
		if let p = parent {
			// 修改之前和之后当前 item 的索引
			let (oldIndex, newIndex) = p.reSort(changedItem: self)
			
			// 调用 store save 方法，发送通知
			// - changeReasonKey: 指定原因
			// - oldValueKey：旧索引
			// - newValueKey: 新索引
			// - parentFolderKey: parent folder 对象
			store?.save(self, userInfo: [Item.changeReasonKey: Item.renamed, Item.oldValueKey: oldIndex, Item.newValueKey: newIndex, Item.parentFolderKey: p])
		}
	}
```

## Folder

每个 folder 持有多个子项 `contents: [Item]`。

### 添加

添加 item

```swift
	func add(_ item: Item) {
		// 不能重复添加
		assert(contents.contains { $0 === item } == false)
		
		// contents 中添加 item 
		contents.append(item)
		
		// sort
		contents.sort(by: { $0.name < $1.name })
		
		let newIndex = contents.index { $0 === item }!
		
		// 建立 parent 引用
		item.parent = self
		
		// 调用 store save 方法，发送通知
		// - changeReasonKey: 指定原因
		// - newValueKey: 新索引
		// - parentFolderKey: parent folder 对象
		store?.save(item, userInfo: [Item.changeReasonKey: Item.added, Item.newValueKey: newIndex, Item.parentFolderKey: self])
	}
```

### 删除

删除 item

```swift
func remove(_ item: Item) {
		// 检查 item 是否存在
		guard let index = contents.index(where: { $0 === item }) else { return }
		
		// 去除 parent 引用
		item.deleted() 
		
		// contents 中移除 item
		contents.remove(at: index)
		
		// 调用 store save 方法，发送通知
		// - changeReasonKey: 指定原因
		// - oldValueKey: 旧索引
		// - parentFolderKey: parent folder 对象
		store?.save(item, userInfo: [
			Item.changeReasonKey: Item.removed,
			Item.oldValueKey: index,
			Item.parentFolderKey: self
		])
	}
````


# Store

单例，持有 rootFolder。

```swift
static let shared = Store(url: documentDirectory)

private(set) var rootFolder: Folder
```

`save(_ notifying: Item, userInfo: [AnyHashable: Any])` 方法

负责发送 `Notification`。先执行文件系统写入操作，再发送 `Notification`，指定发送对象 `object` 为 `notifying`，`userInfo` 为 `userInfo`。

# Notification

传入 `Notification` userinfo 的 key
```swift
extension Item {
	static let changeReasonKey = "reason"
	static let newValueKey = "newValue"
	static let oldValueKey = "oldValue"
	static let parentFolderKey = "parentFolder"
	static let renamed = "renamed"
	static let added = "added"
	static let removed = "removed"
}

```

# ViewController

## FolderViewController

持有一个 `folder` 对象，默认是 root folder，被修改后进行刷新列表等 UI 操作。

```swift
var folder: Folder = Store.shared.rootFolder {
		didSet {
			tableView.reloadData()
			if folder === folder.store?.rootFolder {
				title = .recordings
			} else {
				title = folder.name
			}
		}
	}
```

`viewDidLoad()` 注册通知 `Store.changedNotification`。

```swift
NotificationCenter.default.addObserver(self, selector: #selector(handleChangeNotification(_:)), name: Store.changedNotification, object: nil)
```


`func handleChangeNotification(_ notification: Notification)` 处理通知回调。

检查通知发送的 object 为自己持有的 `folder` 对象。
