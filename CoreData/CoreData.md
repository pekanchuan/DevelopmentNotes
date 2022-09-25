# Core Data

自定义由四个 `Core Data`类构成：

* `NSManagedObjectModel`
* `NSPersistentStore`
* `NSPersistentStoreCoordinator`
* `NSManagedObjectContext`

在这四个类中，到目前为止，您只在本书中遇到过`NSManagedObjectContext`。但其他三个一直在幕后，支持您的托管上下文。

在这一章中，您将详细了解这四个类的操作。您将构建自己的Core Data堆栈，而不是依赖默认入门模板；围绕这些类的可自定义包装器。

## NSManagedObjectModel

`NSManagedObjectModel`表示你的应用程序的数据模型中的每个对象类型，它们可以拥有的属性，以及它们之间的关系。Core Data栈的其他部分使用该模型来创建对象、存储属性和保存数据。

正如本书前面提到的，把`NSManagedObjectModel`看成是一个数据库模式可能会有帮助。如果你的`Core Data`堆栈在后台使用`SQLite`，`NSManagedObjectModel`代表了数据库的模式。

然而，`SQLite`只是你可以在`Core Data`中使用的众多持久化存储类型之一（后面会有更多介绍），所以最好用更普遍的术语来考虑托管对象模型。

> 注意：您可能想知道`NSManagedObjectModel`与您一直使用的数据模型编辑器有何关系。好问题！

> 可视化编辑器创建和编辑`xcdatamodel`文件。有一个特殊的编译器`momc`，将模型文件编译成momd文件夹中的一组文件。

> 正如您的`Swift`代码经过编译和优化，以便在设备上运行一样，编译的模型可以在运行时高效访问。`Core Data`使用`mond`文件夹的编译内容在运行时初始化`NSManagedObjectModel`。

## NSPersistentStore

`NSPersistentStore`将数据读写到你决定使用的任何存储方式。`Core Data`提供了四种类型的`NSPersistentStore`：三种原子型和一种非原子型。

一个原子持久性存储需要完全反序列化并加载到内存中，然后才能进行任何读或写操作。相比之下，非原子持久性存储可以根据需要将自己的大块内容加载到内存中。

下面是对四种内置Core Data存储类型的简要概述：

1. `NSSQLiteStoreType`由一个`SQLite`数据库支持。它是Core Data开箱即支持的唯一非原子存储类型，使其具有轻量级和高效的内存占用率。这使得它成为大多数iOS项目的最佳选择。Xcode的Core Data模板默认使用这种存储类型。
2. `NSXMLStoreType`由一个XML文件支持，使其成为所有存储类型中最容易被人阅读的。这种存储类型是原子性的，所以它可能有很大的内存占用。`NSXMLStoreType`仅在OS X上可用。
3. `NSBinaryStoreType`是由一个二进制数据文件支持的。和`NSXMLStoreType`一样，它也是一个原子存储，所以整个二进制文件必须在你对它做任何事情之前加载到内存中。在现实世界的应用中，你很少会发现这种类型的持久化存储。
4. `NSInMemoryStoreType`是内存中的持久化存储类型。在某种程度上，这种存储类型并不是真正的持久性。终止应用程序或关闭手机，存储在内存存储类型中的数据就会消失在空气中。虽然这似乎违背了Core Data的目的，但内存持久化存储对于单元测试和某些类型的缓存是有帮助的。

## NSPersistentStoreCoordinator

`NSPersistentStoreCoordinator`是管理对象模型和持久化存储之间的桥梁。它负责使用模型和持久化存储来完成Core Data中的大部分艰苦工作。它理解`NSManagedObjectModel`，知道如何向`NSPersistentStore`发送信息，以及从`NSPersistentStore`获取信息。

`NSPersistentStoreCoordinator`还隐藏了如何配置你的持久化存储的实现细节。这有两个原因是很有用的：

1. `NSManagedObjectContext`（接下来会出现！）不必知道它是否保存到SQLite数据库、XML文件甚至是自定义增量存储。
2. 如果你有多个持久化存储，持久化存储协调器会向受管上下文提供一个统一的接口。就被管理的上下文而言，它总是与一个单一的、聚合的持久性存储进行交互。

## NSManagedObjectContext

在日常工作中，你会在四个堆栈组件中最多地使用`NSManagedObjectContext`。你可能只有在需要用Core Data做一些更高级的事情时才会看到其他三个组件。

由于使用`NSManagedObjectContext`的工作非常普遍，所以理解上下文的工作方式是非常重要的! 下面是到目前为止你可能已经从书中得到的一些东西：

* 上下文是一个内存中的抓取板，用于处理你的管理对象。
* 你在托管对象上下文中对你的核心数据对象做所有的工作。
* 在你对上下文调用`save()`之前，你所做的任何改变都不会影响磁盘上的底层数据。

现在，这里有五件关于上下文的事情以前没有提到。其中有几件对后面的章节非常重要，所以要密切注意：

1. 上下文管理着它所创建或获取的对象的生命周期。这种生命周期管理包括强大的功能，如故障、反转关系处理和验证。
2. 一个被管理的对象不可能没有相关的上下文而存在。事实上，一个被管理的对象和它的上下文是如此紧密地结合在一起，以至于每个被管理的对象都保持着对它的上下文的引用，它可以像这样被访问：`let managedContext = employee.managedObjectContext`
3. 上下文具有很强的作用域性；一旦一个被管理的对象被与一个特定的上下文相关联，它将在其生命周期内保持与同一个上下文相关联。
4. 一个应用程序可以使用一个以上的上下文--大多数非琐碎的Core Data应用程序都属于这一类。由于一个上下文是磁盘上内容的内存划痕，你实际上可以同时将同一个Core Data对象加载到两个不同的上下文中。
5. 上下文不是线程安全的。管理对象也是如此。你只能在创建上下文和托管对象的同一线程中与之交互。

## NSPersistentContainer

如果你认为Core Data堆栈只有四块，那你可就大吃一惊了! 从**iOS 10**开始，有一个新的类来协调所有四个核心数据栈类：管理模型、存储协调器、持久化存储和管理上下文。

这个类的名字是`NSPersistentContainer`，正如它的名字所暗示的，它是一个把所有东西都放在一起的容器。你可以简单地初始化一个`NSPersistentContainer`，加载它的持久化存储，然后你就可以开始工作了，而不是浪费你的时间来编写模板代码，把所有四个堆栈组件连接在一起。