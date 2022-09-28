# NSFetchedResultsController

`NSFetchedResultsController`抽象出了将表视图与`Core Data`存储同步所需的大部分代码。

`NSFetchedResultsController`的核心是对`NSFetchRequest`的封装，以及对其获取结果的容器。

```
lazy var fetchedResultsController:
  NSFetchedResultsController<Team> = {
  // 1
  let fetchRequest: NSFetchRequest<Team> = Team.fetchRequest()
  // 2
  let fetchedResultsController = NSFetchedResultsController(
    fetchRequest: fetchRequest,
    managedObjectContext: coreDataStack.managedContext,
    sectionNameKeyPath: nil,
    cacheName: nil)
  return fetchedResultsController
}()
```

和`NSFetchRequest`一样，`NSFetchedResultsController`需要一个泛型参数。

接着，使用这个属性，例如在`viewDidLoad`中使用：

```
do {
  try fetchedResultsController.performFetch()
} catch let error as NSError {
  print("Fetching error: \(error), \(error.userInfo)")
}
```

`NSFetchedResultsController`既是一个`fetch request`的封装，也是一个`fetch result`的容器。你可以通过`fetchedObjects`属性或`object(at:)`方法获得它们。

将获取的结果与列表（`tableView`，`collectionView`，等等）进行绑定

```
func numberOfSections(in tableView: UITableView) -> Int {
  fetchedResultsController.sections?.count ?? 0
}

func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
  guard let sectionInfo = fetchedResultsController.sections[section] else {
  return 0
}
  return sectionInfo.numberOfObjects
}

func configure(cell: UITableViewCell, for indexPath: IndexPath) {
  guard let cell = cell as? TeamCell else { return }
  
  let team = fetchedResultsController.object(at: indexPath)
  cell.teamLabel.text = team.teamName
  cell.scoreLabel.text = "Wins: \(team.wins)"
  
  if let imageName = team.imageName {
    cell.flagImageView.image = UIImage(named: imageName)
  } else {
    cell.flagImageView.image = nil
  }
}
```

这个时候如果运行项目，程序会崩溃……🙃

在控制台的信息中会看到这段文字

>  'An instance of NSFetchedResultsController requires a fetch
request with sort descriptors'

`NSFetchedResultsController`需要至少一个排序描述符，在声明那里添加一个排序描述符

```
let sort = NSSortDescriptor(key: #keyPath(Team.teamName), ascending: true)
fetchRequest.sortDescriptors = [sort]
```

# Modifying data 修改数据

在`tableView(_:didSelectRowAt:)`方法中

```
func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
  let team = fetchedResultsController.object(at: indexPath)
  team.wins += 1
  coreDataStack.saveContext()
  
  tableView.reloadData()
}
```

# Grouping results into sections

对列表数据进行分组显示，在`fetchedResultsController`的初始化方法中给`sectionNameKeyPath`参数一个值就可以

```
let fetchedResultsController = NSFetchedResultsController(
  fetchRequest: fetchRequest,
  managedObjectContext: coreDataStack.managedContext,
  sectionNameKeyPath: #keyPath(Team.qualifyingZone),
  cacheName: nil)
```

使用`Team`中的`qualifyingZone`属性来进行分组

与列表进行关联

```
func tableView(_ tableView: UITableView, titleForHeaderInSection section: Int) -> String? {
  let sectionInfo = fetchedResultsController.sections?[section]
  return sectionInfo?.name
}
```

在属性声明中用`Team`模型的`teamName`声明单个排序描述符排序，还可以添加多个排序描述符来排序

```
let fetchRequest = Team.fetchRequest()

let zoneSort = NSSortDescriptor(key: #keyPath(Team.qualifyingZone), ascending: true)
let scoreSort = NSSortDescriptor(key: #keyPath(Team.wins), ascending: false)
let nameSort = NSSortDescriptor(key: #keyPath(Team.teamName), ascending: true)

fetchRequest.sortDescriptors = [zoneSort, scoreSort, nameSort]
```

上面的代码中的数据排序顺序，首先按照`zoneSort`排序，然后按照`scoreSort`排序，最后按照`nameSort`排序。

# Cache

对`NSFetchedResultsController`的实例进行修改：

```
let fetchedResultsController = NSFetchedResultsController(
  fetchRequest: fetchRequest,
  managedObjectContext: coreDataStack.managedContext,
  sectionNameKeyPath: #keyPath(Team.qualifyingZone),
  cacheName: "worldCup")
```

只为`cacheName`参数赋一个值，就这么简单。

# Monitoring changes

用`NSFetchedResultsControllerDelegate`来检测变化

### Responding to changes

```
func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
  tableView.reloadData()
}
```

实现这个方法意味着任何变化，无论来源如何，都会刷新表视图。

实现上面的代理方法，当数据变化时，是刷新整个列表，动画看起来很生硬。接下来只刷新变化的行，而不是整个列表。

```
func controllerWillChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
    tableView.beginUpdates()
}
func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>, didChange anObject: Any, at indexPath: IndexPath?, for type: NSFetchedResultsChangeType, newIndexPath: IndexPath?) {
  switch type {
  case .insert:
    tableView.insertRows(at: [newIndexPath!], with: .automatic)
  case .delete:
    tableView.deleteRows(at: [indexPath!], with: .automatic)
  case .update:
    let cell = tableView.cellForRow(at: indexPath!) as! TeamCell
    configure(cell: cell, for: indexPath!)
  case .move:
    tableView.deleteRows(at: [indexPath!], with: .automatic)
    tableView.insertRows(at: [newIndexPath!], with: .automatic)
  @unknown default:
    print("Unexpected NSFetchedResultsChangeType")
  }
}
func controllerDidChangeContent(_ controller:
  NSFetchedResultsController<NSFetchRequestResult>) {
    tableView.endUpdates()
}
```

* `controllerWillChangeContent(_:)`这个委托方法会通知你即将发生的变化。你用`beginUpdates()`来准备你的表视图。
* `controller(_:didChange:at:for:newIndexPath:)`这个方法很拗口。它准确地告诉你哪些对象发生了变化，发生了什么类型的变化（插入、删除、更新或重新排序）以及受影响的索引路径是什么。这个中间方法是胶水方法，它使你的表视图与核心数据同步。无论底层数据如何变化，你的表视图都会与持久化存储中的情况保持一致。
* `controllerDidChangeContent(_:)`你最初实现的刷新用户界面的委托方法，原来是通知你变化的三个委托方法中的第三个。你不需要刷新整个表视图，而只需要调用`endUpdates()`来应用这些变化。

另一个`NSFetchedResultsControllerDelegate`方法

```
func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>, didChange sectionInfo: NSFetchedResultsSectionInfo, atSectionIndex sectionIndex: Int, for type: NSFetchedResultsChangeType) {
  let indexSet = IndexSet(integer: sectionIndex)

  switch type {
  case .insert:
    tableView.insertSections(indexSet, with: .automatic)
  case .delete:
    tableView.deleteSections(indexSet, with: .automatic)
  default:
    break
  }
}
```

这个方法类似于`controllerDidChangeContent(_:)`，但它会通知你某个部分的变化，而不是单个对象的变化。在这里，你要处理底层数据的变化触发了整个部分的创建或删除的情况。

# Diffable data sources

在iOS 13中，苹果引入了一种新的方式来实现表视图和集合视图：可差异的数据源（**diffable data sources**）。

```
@preconcurrency @MainActor class UITableViewDiffableDataSource<SectionIdentifierType, ItemIdentifierType> : NSObject where SectionIdentifierType : Hashable, SectionIdentifierType : Sendable, ItemIdentifierType : Hashable, ItemIdentifierType : Sendable
```

[UITableViewDiffableDataSource](https://developer.apple.com/documentation/uikit/uitableviewdiffabledatasource)

除了可差异的数据源，还有一种使用`NSFetchedResultsController`的新方法，以监测获取请求的结果集的变化。

首先注释掉`UITableViewDataSource`所有的代理方法。

然后添加一个属性：

```
var dataSource: UITableViewDiffableDataSource<String, NSManagedObjectID>?
```

在这个属性中`UITableViewDiffableDataSource`的两个泛型类型中，字符串代表章节标识符，`NSManagedObjectID`代表不同团队的管理对象标识符。

接着添加一个新的方法：

```
func setupDataSource() -> UITableViewDiffableDataSource<String, NSManagedObjectID> {
  UITableViewDiffableDataSource(tableView: tableView) { [unowned self] tableView, indexPath, itemIdentifier -> UITableViewCell? in
    let cell = tableView.dequeueReusableCell(withIdentifier: self.teamCellIdentifier, for: indexPath)
    
    if let team = try? coreDataStack.managedContext.existingObject(with: itemIdentifier) as? Team {
      self.configure(cell: cell, for: team)
    }
    return cell
  }
}
```

这个方法创建了你的`UITableViewDiffableDataSource`。当创建一个这样的数据源时，它会自动将自己添加为表视图的数据源。注意，你传递了一个用于配置单元格的闭包，而不是单独的方法。
由于数据源对`NSManagedObjectID`来说是通用的，你使用`existingObject(with:)`将标识符变成相应的`Team`对象来配置每个单元。

实现`configure(cell:for:)`方法：

```
func configure(cell: UITableViewCell, for team: Team) {
  guard let cell = cell as? TeamCell else { return }
  
  cell.teamLabel.text = team.teamName
  cell.scoreLabel.text = "Wins: \(team.wins)"
  
  if let imageName = team.imageName {
    cell.flagImageView.image = UIImage(named: imageName)
  } else {
    cell.flagImageView.image = nil
  }
}
```

接着在`viewDidLoad()`中添加

```
dataSource = setupDataSource()
```

然后可以注释或者删除掉`NSFetchedResultsControllerDelegate`上面的四个方法，替换成一个新的方法

```
func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>, didChangeContentWith snapshot: NSDiffableDataSourceSnapshotReference) {
  let snapshot = snapshot as NSDiffableDataSourceSnapshot<String, NSManagedObjectID>
  dataSource?.apply(snapshot)
}
```

旧的委托方法告诉你什么时候会发生变化，变化是什么，以及变化何时完成。
这些委托方法与`UITableView`中的`beginUpdates()`和`endUpdates()`等方法非常吻合，由于你切换到了不同的数据源，你不再需要调用这些方法了。
取而代之的是，新的委托方法给你一个对获取的结果集的任何变化的总结，并传递给你一个预先计算的快照，你可以直接应用到你的表视图。这样就简单多了!

因为删除了`UITableViewDataSource`的所有代理方法，所以看不到上面设置的`section header title`。但是可以使用`UITableViewDelegate`中的代理方法来设置`viewForHeaderInSection`

```
func tableView(_ tableView: UITableView, viewForHeaderInSection section: Int) -> UIView? {
  let sectionInfo = fetchedResultsController.sections?[section]
  
  let titleLabel = UILabel()
  titleLabel.backgroundColor = .white
  titleLabel.text = sectionInfo?.name
  
  return titleLabel
}

func tableView(_ tableView: UITableView, heightForHeaderInSection section: Int) -> CGFloat {
  20
}
```

在`tableView(_:didSelectRowAt:)`代理方法中添加下面的方法，在`saveContext()`之前

```
if var snapshot = dataSource?.snapshot() {
  snapshot.reloadItems([team.objectID])
  dataSource?.apply(snapshot, animatingDifferences: false, completion: nil)
}
```

# Key points

* `NSFetchedResultsController`抽象出了将表视图与`Core Data`存储同步所需的大部分代码。
* `NSFetchedResultsController`的核心是一个`NSFetchRequest`的包装器和一个获取结果的容器。
* 获取的结果控制器要求在其获取请求上至少设置一个排序描述符。如果您忘记了排序描述符，您的应用程序将会崩溃。
* 您可以设置获取结果的控制器`sectionNameKeyPath`，以指定将结果分组到表格视图部分的属性。每个唯一值对应于不同的表视图部分。
* 将一组获取的结果分组到部分是一个昂贵的操作。通过在你的获取的结果控制器上指定一个缓存名称来避免多次计算部分。
* 一个获取的结果控制器可以监听其结果集的变化，并通知其代理`NSFetchedResultsControllerDelegate`，以响应这些变化。
* `NSFetchedResultsControllerDelegate`监视单个`Core Data`记录的变化（是否被插入、删除或修改），以及整个部分的变化。
* 差异化的数据源使得与获取的结果控制器和表视图的工作更加容易。