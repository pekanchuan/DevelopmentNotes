# Creating your stack object

创建 `CoreDataStack.swift` 文件

```
import Foundation
import CoreData

class CoreDataStack {
  private let modelName: String
  
  init(modelName: String) {
    self.modelName = modelName
  }
  
  private lazy var storeContainer: NSPersistentContainer = {
  
    let container = NSPersistentContainer(name: self.modelName)
    container.loadPersistentStores { _, error in
      if let error = error as NSError? {
        print("Unresolved error \(error), \(error.userInfo)")
      }
    }
    return container
  }()
}
```

首先导入`CoreData`模块，然后创建一个私有属性来存储 `modelName`。再添加一个初始化方法，将`modelName`保存到私有属性中。

然后，懒加载 `NSPersistentContainer`，传递初始化后的`modelName`。唯一需要做的事情就是在持久化容器上调用`loadPersistentStores(completionalHandler:)`（这个方法默认不会异步运行）。

最后在`modelName`下面添加一个懒加载的属性：

```
  lazy var managedContext: NSManagedObjectContext = {
    self.storeContainer.viewContext
  }()
```

尽管`NSPersistentContainer`为托管上下文（managed context）、托管模型（managed model）、存储协调器（store coordinator）和持久存储（persistent stores）（通过`NSPersistentStoreDescription`），但是自定义的`Core Data`类的工作方式略有不同。

在自定义的`CoreDataStack`中只有`NSManagedObjectContext`是公开访问的。`managed context`是访问自定义类中其他部分的唯一入口。持久性存储协调器（`NSPersistentStoreCoordinator`）是`NSManagedObjectContext`的一个公共属性，同样，托管对象模型和持久化存储阵列都是`NSPersistentStoreCoordinator`的公共属性。

最后，在下面添加一个方法：

```
  func saveContext() {
    guard managedContext.hasChanges else { return }
    
    do {
      try managedContext.save()
    } catch let error as NSError {
      print("Unresolved error \(error), \(error.userInfo)")
    }
  }
```

这是保存堆栈的托管对象上下文并处理任何由此产生的错误的便捷方法。

# Modeling your data

创建数据模型。如果在项目的文件夹中没有看到`.xcdatamodeld`文件，可以新建一个。

> 注意：模型文件个名字一定要和项目的名字一样，否则之后会遇到问题

