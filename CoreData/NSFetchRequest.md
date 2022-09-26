# NSFetchRequest

通过创建 `NSFetchRequest` 实例，根据需要对其进行配置并将其移交给 `NSManagedObjectContext` 来完成繁重的工作，从 `Core Data` 中获取记录。

看起来很简单，但实际上有五种不同的方法来获取 `fetch` 请求。有些比其他的更受欢迎，但作为 `Core Data` 开发人员，您可能会在某个时候遇到所有这些。

以下是设置 `fetch` 请求的五种不同方法：

```
// 1
let fetchRequest1 = NSFetchRequest<Venue>()
let entity = NSEntityDescription.entity(forEntityName: "Venue", in: managedContext)!

// 2                         
let fetchRequest2 = NSFetchRequest<Venue>(entityName: "Venue")
fetchRequest1.entity = entity

// 3
let fetchRequest3: NSFetchRequest<Venue> = Venue.fetchRequest()

// 4
let fetchRequest4 = managedObjectModel.fetchRequestTemplate(forName: "venueFR")
  
// 5
let fetchRequest5 = managedObjectModel.fetchRequestFromTemplate(withName: "venueFR", substitutionVariables: ["NAME" : "Vivi Bubble Tea"])
```

1. 将 `NSFetchRequest` 的实例初始化为泛型类型：`NSFetchRequest<Venue>`。至少必须为读取请求指定 `NSEntityDescription`。在本例中，实体为 `Venue`。初始化 `NSEntityDescription`的实例，并使用它来设置读取请求的实体属性。
2. 这里你使用`NSFetchRequest`的方便初始化器。它在一个步骤中初始化了一个新的获取请求并设置其实体属性。你只需要为实体名称提供一个字符串，而不是一个完整的`NSEntityDescription`。
3. 正如第二个例子是第一个的缩写，第三个例子是第二个的缩写。当您生成`NSManagedObject`子类时，此步骤还会生成一个`class`方法，该方法返回一个已经设置好的`NSFetchRequest`，以获取相应的实体类型。这就是`Venue.fetchRequest()`的由来。这个代码存在于`Venue+CoreDataProperties.swift`中。
4. 在第四个例子中，你从你的`NSManagedObjectModel`中检索你的`fetch`请求。你可以在`Xcode`的数据模型编辑器中配置和存储常用的`fetch`请求。
5. 最后一种情况与第四种情况类似。从你的托管对象模型中获取一个获取请求，但是这一次，你传入了一些额外的变量。这些 "替换 "变量被用在一个谓词中，以完善你的获取的结果。

