---
layout: post
title: "Morden Collection View：DataSource 篇 -- 如何保证 model 与 view 的一致展现"
---

# Morden Collection View：DataSource 篇 -- 如何保证 model 与 view 的一致展现

## 0 前言

`UICollectionView` 从 iOS 6 引入 iOS 系统开始，一直是展现大量相同类型数据的首选 view。然而随着系统和应用的复杂度越来越高，`UICollectionView` 也暴露出了一系列的问题，包括：

- `UICollectionViewDataSource` 协议虽然保证了数据源的灵活性，但是缺乏机制保证 model 和 view 的一致展现
- 基于 `UICollectionFlowLayout` 的布局方式实现简单的网格化布局容易，但对于复杂的布局样式显得捉襟见肘
- `UICollectionViewCell` 的展现样式与 Cell 本身的类型强相关，复用困难
- ...

苹果也意识到了上述问题，于是从 iOS 13 系统开始，针对 `UICollectionView` 以及 `UITableView` 做了一系列优化与 API 的改动。本文先从 DataSource 的角度入手，观察新 API 究竟带来了哪些变化。

## 1 现状

在介绍新 API 之前，先来看看”传统“方式如何实现一个 CollectionView。

`UICollectionViewDataSource` 是系统提供的 `UICollectionView` 的数据源协议，包括如下几个方法：

```swift
// Getting Item and Section Metrics
// 每个 section 中 item 的数量
func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int
// section 的数量，默认为 1
optional func numberOfSections(in collectionView: UICollectionView) -> Int

// Getting Views for Items
// 根据 indexPath 返回对应的 cell
func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell
// 根据 kind 和 indexPath 返回对应的 supplementary view
optional func collectionView(_ collectionView: UICollectionView, viewForSupplementaryElementOfKind kind: String, at indexPath: IndexPath) -> UICollectionReusableView

// Reordering Items
// indexPath 位置的对应元素能否移动
optional func collectionView(_ collectionView: UICollectionView, canMoveItemAt indexPath: IndexPath) -> Bool
// 元素移动后的 callback，通常需要对应调整数据源
optional func collectionView(_ collectionView: UICollectionView, moveItemAt sourceIndexPath: IndexPath, to destinationIndexPath: IndexPath)

// Configuring an Index（iOS 14+）
// 返回元素的快速索引（参见系统通讯录）, 与 UITableView 效果对应
optional func indexTitles(for collectionView: UICollectionView) -> [String]?
// 根据索引返回定位
optional func collectionView(_ collectionView: UICollectionView, indexPathForIndexTitle title: String, at index: Int) -> IndexPath
```

实现其中未被标记为 `optional` 的两个：

```swift
// 每个 section 中 item 的数量
func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int
// 根据 indexPath 返回对应的 cell
func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell
```

即可完成一个最基本的 `UICollectionView` 实现，例如：

```swift
class Cell: UICollectionViewCell {
    let label: UILabel
    
    override init(frame: CGRect) {
        label = UILabel()
        super.init(frame: frame)
        
        contentView.addSubview(label)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
}

class LegacyCollectionViewController: UICollectionViewController {
    var people = Person.allPeople
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        collectionView.register(Cell.self, forCellWithReuseIdentifier: "cell")
    }

    override func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        people.count
    }
    
    override func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "cell", for: indexPath) as! Cell
        
        cell.label.text = people[indexPath.row].name
        cell.label.sizeToFit()
        
        return cell
    }
}
```

## 2 一切看起来都 OK，所以，问题在哪...

仅仅展示一个静态的 CollectionView 当前毫无问题，然而在现实中通常不会如此顺利。也许是新的网络请求返回需要修改当前展现在屏幕上的数据，也许是用户需要手动编辑数据，这里就以一个简单的交换两条数据为例，方法的实现同样非常简单：

```swift
func randomSwap() {
    let i = Int.random(in: 0..<people.count)
    let j = Int.random(in: 0..<people.count)
    people.swapAt(i, j)

    collectionView.reloadData()
}
```

这样的方式当然可以实现需求，但是未免显得有些过于粗暴了。因为少量数据的变动导致所有元素重新绘制一遍，实在不是一个好主意。`UICollectionView` 当然提供了更精细化的 `reloadItems(at: [IndexPath])` 方法：

```swift
func randomSwap() {
    let i = Int.random(in: 0..<people.count)
    let j = Int.random(in: 0..<people.count)
    people.swapAt(i, j)

    collectionView.reloadItems(at: [IndexPath(row: i, section: 0),
                                    IndexPath(row: j, section: 0)])
}
```

然而这种实现也并非完美，首先这种实现无法使用动画直观反馈交换过程，其次当进行复杂的修改，例如同时有添加/修改/删除操作时，计算出所有需要变动的 IndexPath 也是一件麻烦事。

先解决动画的问题，`UICollectionView` 提供了

```swift
func performBatchUpdates(_ updates: (() -> Void)?, completion: ((Bool) -> Void)? = nil)
```

方法专门用于实现类似批量调整需求，这里要做的是交换对应位置的元素，实现如下：

```swift
func randomSwap() {
    let i = Int.random(in: 0..<people.count)
    let j = Int.random(in: 0..<people.count)
    people.swapAt(i, j)

    collectionView.performBatchUpdates {
        collectionView.moveItem(at: IndexPath(row: i, section: 0), to: IndexPath(row: j, section: 0))
        collectionView.moveItem(at: IndexPath(row: j, section: 0), to: IndexPath(row: i, section: 0))
    }
}
```

移动 `i` 到 `j` 再移动 `j` 到 `i` 的操作可能会让人感到迷惑，实际上在 `performBatchUpdates` block 中执行的移动操作并无执行先后顺序之分，可以认为每次移动操作都是作用于未被修改过的 CollectionView 上的。

## 3  无须计算 IndexPath 的新 API

虽然解决了动画的问题，然而找出所有变动的 IndexPath 仍是一个不得不处理的问题。幸运的是，苹果意识到了这个问题，并在 iOS 13 版本中推出了全新的 API：`UICollectionViewDiffableDataSource`。

### 3.1 如何构造 UICollectionViewDiffableDataSource

`UICollectionViewDiffableDataSource` 提供了唯一的构造方法：

```swift
@MainActor init(collectionView: UICollectionView, cellProvider: @escaping UICollectionViewDiffableDataSource<SectionIdentifierType, ItemIdentifierType>.CellProvider)

typealias UICollectionViewDiffableDataSource<SectionIdentifierType, ItemIdentifierType>.CellProvider = (_ collectionView: UICollectionView, _ indexPath: IndexPath, _ itemIdentifier: ItemIdentifierType) -> UICollectionViewCell?
```



构造 `UICollectionViewDiffableDataSource` 需要两个参数，一个是与 DataSource 关联的 CollectionView，另一个是返回 `UICollectionViewCell` 的名为 `CellProvider` 的 block。`CellProvider` 的参数和返回值看上去似乎很眼熟：

```swift
func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell
```

正是之前 `UICollectionViewDataSource` 协议中获取 Cell 的方法，只是多了一个 `itemIdentifier` 参数。

### 3.2 灵活选择 ItemIdentifier

`ItemIdentifierType` 作为 `UICollectionViewDiffableDataSource` 的泛型类型，只要求满足 `Hashable` 协议，这给了其在选择上的一定灵活性。可以考虑的方案是：1. 使用 model 中合适的属性作为 `ItemIdentifier`；2. 直接令 model 的类型满足 `Hashable` 协议。

对于常见的通过网络从服务端获取 model 的 App 来说，通常 model 中会带有其在数据库中的 ID，这种情况下可以直接使用这个属性作为 ItemIdentifier。需要注意的地方是，`CellProvider` 提供的参数的类型是 `ItemIdentifierType`，因此在使用前需要建立 `itemIdentifier` 与 model 的映射关系：

```swift
UICollectionViewDiffableDataSource<Section, Int>(collectionView: collectionView, cellProvider: { collectionView, indexPath, identifier in
            let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "cell", for: indexPath) as! Cell
            
            let person = dataStore.person(with: identifier)
            cell.configure(with: person)
            
            return cell
        })
```

当然对一些相对简单的 model 来说，可以考虑直接使用 model 自身作为 `itemIdentifier`，只是在 model 实现`Hashable` 协议时有一点需要注意。Swift 语言会通过编译器给 `struct ` 类型的 model 自动实现 `Hashable` 协议，具体的实现是让类型的所有属性参与 Hash 过程，这种实现可能未必符合需要，因此需要手动重写  `func hash(into hasher: inout Hasher)`  方法，使得只有必要的属性参与 hash 过程：

```swift
struct Person: Hashable {
    var name: String
    var id: Int
    
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

### 3.3 不要忘了 SectionIdentifier

除了用于标记 item 的 `itemIdentifier`，对于 Section 来说也有同样的 `SectionIdentifier`。对于复杂的 CollectionView，可能有多个不同的 Section 同时/非同时展现，这种情况参照 itemIdentifier 的处理方式即可。

对于简单的 CollectionView，可能是固定的数个 Section，也可能简单到只有一个 Section。得益于 Swift 能为一些值类型提供默认的 `Hashable` 实现，这种情况可以直接使用 enum 作为 `SectionIdentifier`。

```swift
enum Section: Hashable {
    case main, secondary
}
```

### 3.4 NSDiffableDataSourceSnapshot：状态的抽象

说完了如何构造数据与 Cell 的关系，接下来的一步就是给 `UICollectionViewDiffableDataSource` 提供数据用于展示。`UICollectionViewDiffableDataSource` 提供了

```swift
func apply(_ snapshot: NSDiffableDataSourceSnapshot<SectionIdentifierType, ItemIdentifierType>, animatingDifferences: Bool = true, completion: (() -> Void)? = nil)
```

接口，其中 `snapshot` 参数便是承载数据的关键类型。

snapshot，译为“快照”，最先是摄影中的概念，后来被计算机理论借用，代表系统在某个时间点上的状态。苹果再度将这个概念拓展到了 iOS，用于表示 CollectionView 的某个状态。虽然承载着这个 CollectionView 的数据，但构造一个 snapshot 却相当简单：

```swift
var snapshot = NSDiffableDataSourceSnapshot<Section, Person>()
```

有了 snapshot 之后，便是调用 append 方法为其添加数据：

```swift
mutating func appendSections(_ identifiers: [SectionIdentifierType])
mutating func appendItems(_ identifiers: [ItemIdentifierType], toSection sectionIdentifier: SectionIdentifierType? = nil)
```

最后，使用 `apply` 方法将 snapshot 提供给 dataSource。完整的过程类似：

```swift
var snapshot = NSDiffableDataSourceSnapshot<Section, Person>()

snapshot.appendSections([.main])
snapshot.appendItems(Person.allPeople, toSection: .main)

dataSource.apply(snapshot)
```

从变量的 `var` 关键词，`append` 方法的 `mutating` 关键词可以很明显看出，`NSDiffableDataSourceSnapshot` 是一个 struct，这意味着每一个 snapshot 只是其内部数据的载体，保存某个具体的 snapshot 并无意义。换句话说，当数据发生变化时，没有机制能自动感知变化的发生，而是需要开发者向 dataSource 提供新的 snapshot 用于更新 UI。

### 3.5 获取当前 snapshot 用于增删改查

既然 snapshot 对应了当前的 UI 状态，那么自然可以通过获取当前 snapshot，对其进行修改，再 apply 的步骤更新 collectionView 的显示。

以一个简单的插入排序为例：

```swift
func sort(step: Int) {
    // 获取 snapshot 以及当前数据
    var snapshot = dataSource.snapshot()
    var items = snapshot.itemIdentifiers(inSection: .main)

    guard step < items.count else { return }

    // 修改数据
    var target = items.startIndex
    while items[target].id < items[step].id {
        target += 1
    }
    items.insert(items.remove(at: step), at: target)

    // 更新 snapshot
    snapshot.deleteAllItems()
    snapshot.appendSections([.main])
    snapshot.appendItems(items)

    // apply snapshot
    dataSource.apply(snapshot)

    DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
        self.sort(step: step + 1)
    }
}
```

即可实现 collectionView 实时展示排序进度

![result](/asserts/DiffableDataSource/01.gif)
