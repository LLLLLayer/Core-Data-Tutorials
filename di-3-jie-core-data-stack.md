# 第 3 节：Core Data Stack

到目前为止，你一直依赖 Xcode 的 Core Data 模板。 从 Xcode 获得帮助并没有错。 但是如果你真的想知道 Core Data 是如何工作的，那么构建你自己的 Core Data Stack 是必须的。

该 Stack 由四个 Core Data 类组成：

* NSManagedObjectModel
* NSPersistentStore
* NSPersistentStoreCoordinator
* NSManagedObjectContext

在这四个类中，到目前为止，你在本书中只遇到过 NSManagedObjectContext。 但其他三个一直在幕后支持你的 ManagedObjectContext。

在本章中，你将详细了解这四个类的作用。 你将构建自己的 Core Data Stack，而不是依赖默认的模板； 围绕这些类的可定制的 Wrapper。

### 入门

本章的示例项目是一个简单的遛狗应用程序。此应用程序可让你在简单的表格视图中保存遛狗的日期和时间。定期使用此应用程序，你的狗狗就会爱上你。

你将在本书随附的资源中找到示例项目 DogWalk。打开 DogWalk.xcodeproj，然后构建并运行起始项目。

![image-20221122015353234](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221122015353234.png)

如你所见，示例应用程序已经是一个完全可用（虽然简单）的原型。点击右上角的加号 (+) 按钮会在散步列表中添加一个新条目。该图像代表你当前正在遛狗，但除此之外什么都不做。

该应用程序具有它需要的所有功能，除了一项重要功能：散步列表不会持续存在。如果你终止 DogWalk 并重新启动，你的整个历史记录都将消失。如果你今天早上遛过狗，你会怎么记得？

本章的任务是在 Core Data 中保存散步列表。如果这听起来像是你在第 1 节和第 2 节中已经做过的事情，那么这里就是转折点；你将编写自己的 Core Data Stack 以了解幕后真正发生的事情！

### 了解 Core Data Stack

知道 Core Data Stack 是如何工作的不仅仅是一件好事。如果你正在使用更高级的设置，例如从旧的持久存储中迁移数据，那么深入研 Core Data Stack 是必不可少的。

在开始编写代码之前，让我们仔细考虑一下 Core Data 堆栈中的四个类——NSManagedObjectModel、NSPersistentStore、NSPersistentStoreCoordinator 和 NSManagedObjectContext——分别做了什么。

#### ManagedObjectModel

NSManagedObjectModel 代表应用程序 Data Model 中的每个对象类型，它们可以拥有的属性以及它们之间的关系。 Core Data Stack 的其他部分使用该 Model 来创建 objects、存储 properties 和保存 data。

正如本书前面提到的，将 NSManagedObjectModel 视为 Database Schema 可能会有所帮助。如果你的 Core Data Stack 在底层使用 SQLite，则 NSManagedObjectModel 表示 Database Schema。

然而，SQLite 只是你可以在 Core Data 中使用的众多持久存储类型中的一种（稍后会详细介绍），因此最好从更一般的角度来考虑 ManagedObjectModel。

> 注意：你可能想知道 NSManagedObjectModel 与你一直使用的 Data Model Editor 有什么关系。好问题！
>
> 可视化 Editor 创建并编辑 xcdatamodel 文件。有一个特殊的编译器 momc，它将 Model 文件编译成 momd 文件夹中的一组文件。
>
> 正如你的 Swift 代码经过编译和优化以便可以在设备上运行一样，编译后的模型也可以在运行时高效访问。 Core Data 使用 momd 文件夹的编译内容在运行时初始化 NSManagedObjectModel。

#### Persistent Store

NSPersistentStore 是将数据读取和写入的存储方法。 Core Data 提供了四种开箱即用的 NSPersistentStore：三种原子(atomic)的和一种非原子(non-atomic)的。

原子 Persistent Store 需要完全反序列化并加载到内存中，然后才能进行任何读取或写入操作。相比之下，非原子 Persistent Store 可以根据需要将自身的块(Chunks)加载到内存中。

以下是四种内置 Core Data store 类型的简要概述：

1. NSSQLiteStoreType 由 SQLite 数据库支持。它是 Core Data 唯一支持的开箱即用的非原子存储类型，使其具有轻量级和高效的内存占用。这使它成为大多数 iOS 项目的最佳选择。 Xcode 的 Core Data 模板默认使用这种存储类型。
2. NSXMLStoreType 由 XML 文件支持，使其成为所有存储类型中最易读的。这种存储类型是原子的，因此它可能占用大量内存。 NSXMLStoreType 仅在 OS X 上可用。
3. NSBinaryStoreType 由二进制数据文件支持。与 NSXMLStoreType 一样，它也是一个原子存储，因此必须先将整个二进制文件加载到内存中，然后才能对其进行任何操作。在现实世界的应用程序中，你很少会发现这种类型的持久性存储。
4. NSInMemoryStoreType 是内存中持久存储类型。在某种程度上，这种存储类型并不是真正持久的。终止应用程序或关闭手机，存储在内存存储类型中的数据就会消失得无影无踪。尽管这似乎违背了 Core Data 的目的，但内存中的持久存储对于单元测试和某些类型的缓存可能很有帮助。

> 注意：你是否对由 JSON 文件或 CSV 文件支持的持久存储类型感兴趣？好消息是你可以通过子类化 NSIncrementalStore 创建你自己的持久存储类型。
>
> 如果你对此选项感到好奇，请参阅 Apple 文档：https://developer.apple.com/library/archive/documentation/DataManagement/Conceptual/IncrementalStorePG/Introduction/Introduction.html

#### Persistent Store Coordinator

NSPersistentStoreCoordinator 是 ManagedObjectModel 和 Persistent Store 之间的桥梁。它负责使用 Model 和 Persistent Store 来完成 Core Data 中的大部分繁重工作。它了解 NSManagedObjectModel 并知道如何向 NSPersistentStore 发送信息和从中获取信息。

NSPersistentStoreCoordinator 还隐藏了如何配置 Persistent Store 的实现细节。这很有用，原因有二：

1. NSManagedObjectContext 不必知道它是保存到 SQLite 数据库、XML 文件还是自定义增量存储。
2. 如果你有多个 Persistent Store，则 Prsistent Store Coordinator 会为 ManagedObjectContext 提供一个统一的接口。就 ManagedObjectContext 而言，它始终与单个聚合 Persistent Store 交互。

#### ManagedObjectContext

在日常工作中，你大部分时间将使用 NSManagedObjectContext。当你需要使用 Core Data 做一些更高级的事情时，你才可能会看到其他三个组件。

由于使用 NSManagedObjectContext 非常普遍，了解 context 的工作原理非常重要！到目前为止，你可能已经从本书中学到了一些东西：

* context 是用于处理 managed objects 的内存暂存器。
* 你在 ManagedObjectContext 中使用 Core Data Objects 完成所有工作。
* 在调用 context 中的 save() 之前，你所做的任何更改都不会影响磁盘上的基础数据。

现在这里有五件关于 context 的事情，之前没有提到。其中一些对后面的章节非常重要，因此请密切注意：

1. context 管理它创建或获取的对象的 lifecycle。这种 lifecycle 管理包括强大的功能，例如错误处理、反向关系处理和验证。
2. 没有关联的 context， managed object 就无法存在。事实上，一个 managed object 和它的 context 是紧密耦合的，以至于每个 managed object 都保留一个对其 context 的引用，可以像这样访问它：

```swift
let managedContext = employee.managedObjectContext
```

3. context 是非常地域性的；一旦 managed object 与特定 context 相关联，它将在其生命周期期间保持与同一 context 相关联。
4. 一个应用程序可以使用多个 context——大多数重要的 Core Data 应用程序都属于这一类。由于 context 是磁盘上内容的内存暂存器，你实际上可以同时将同一个 Core Data Object 加载到两个不同的 context 中。
5. context 不是线程安全的。 managed object 也是如此：你只能在创建它们的同一线程上与 context 和 managed object 进行交互。

Apple 提供了许多在多线程应用程序中处理 context 的方法。你将在第 9 节“Multiple Managed Object Contexts”中阅读有关不同并发模型的所有内容。

#### Persistent Store Container

如果你认为 Core Data Stack 只有四个部分，你会大吃一惊！从 iOS 10 开始，有一个新类来编排所有四个 Core Data stack 类：the managed model, the store coordinator, the persistent store and the managed context。

这个类的名字是 NSPersistentContainer ，顾名思义，它是一个容器，将所有东西都放在一起。与其浪费时间编写样板代码将所有四个堆栈组件连接在一起，你可以简单地初始化一个 NSPersistentContainer，加载它的持久存储，然后就可以开始了。

### 创建 Core Data stack

现在你知道了每个组件的作用，是时候返回 DogWalk 并实现你自己的 Core Data stack 了。

正如你从前面的章节中了解到的，Xcode 在 app delegate 中创建了它的 Core Data stack。你会以不同的方式去做。你将创建一个单独的类来封装 stack，而不是将应用程序委托代码与 Core Data 代码混合。

转到 File ▸ New ▸ File…，选择 iOS ▸ Source ▸ Swift File 模板并单击 Next。将文件命名为 CoreDataStack 并单击创建以保存文件。

转到新创建的 CoreDataStack.swift。你将逐个创建此文件。首先用以下内容替换文件的内容：

```swift
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

你首先导入 Foundation 和 CoreData。接下来，创建一个私有属性来存储 modelName。接下来，创建一个 init 以将 modelName 保存到私有属性中。

接下来，你设置一个 lazy 实例化的 NSPersistentContainer，传递你在初始化期间存储的 modelName。你唯一需要做的另一件事是在持久容器上调用 `loadPersistentStores(completionHandler:)`（尽管出现了 completionHandler，但默认情况下此方法不会异步运行）。最后，在 modelName 下面添加以下延迟实例化的属性：

```swift
lazy var managedContext: NSManagedObjectContext = {
  return self.storeContainer.viewContext
}()
```

尽管 NSPersistentContainer 为 managed context、managed model、store coordinator 和 persistent stores（通过 \[NSPersistentStoreDescription]）提供公共访问器，但 CoreDataStack 的工作方式略有不同。

例如，CoreDataStack 唯一可公开访问的部分是 NSManagedObjectContext，因为你刚刚添加了 lazy 属性。其他所有内容都标记为私有。为什么是这样？

managed context 是访问 stack 其余部分所需的唯一入口点。persistent store coordinator 是 NSManagedObjectContext 的 public 属性。同样，managed object model 和持 persistent store 数组都是 NSPersistentStoreCoordinator 的 public 属性。

最后，在 storeContainer 属性下面添加以下方法：

```swift
func saveContext () {
  guard managedContext.hasChanges else { return }

  do {
    try managedContext.save()
  } catch let error as NSError {
    print("Unresolved error \(error), \(error.userInfo)")
  }
}
```

这是保存 stack 的 managed object context 并处理任何由此产生的错误的便捷方法。

打开 ViewController.swift 并进行以下更改。首先，导入 CoreData。在 import UIKit 下方添加以下内容：

```swift
import CoreData
```

接下来，在 dateFormatter 下面添加以下属性以保存 Core Data Stack：

```swift
lazy var coreDataStack = CoreDataStack(modelName: "DogWalk")	
```

### 构造 Data

现在你闪亮的新 Core Data Stack 已经安全地固定到 MainViewController 上了，是时候创建你的 Data Model 了。

转到你的项目导航器并...等一下。没有数据模型文件！这是正确的。由于我生成此示例应用程序时未启用使用 Core Data 的选项，因此没有 .xcdatamodeld 文件。

不用担心。转到 File ▸ New ▸ File..., select the iOS ▸ Core Data ▸ Data Model template，然后单击下一步。

将文件命名为 DogWalk.xcdatamodeld 并单击创建以保存文件。

> 注意：如果你没有准确命名你的数据模型文件 DogWalk.xcdatamodeld，你以后会遇到问题。 这是因为 CoreDataStack.swift 期望在 DogWalk.momd 找到编译后的版本。

打开数据模型文件并创建一个名为 Dog 的新 Entity。 你现在应该能够自己完成此操作，但如果你忘记了如何操作，请单击左下角的“ Add Entity 体”按钮。

添加一个名为 name 的 String 类型的属性。 你的 Data Model 应如下所示：

![image-20221125014748620](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221125014748620.png)

你还想跟踪特定狗的散步情况。 毕竟，这就是应用程序的全部意义所在！

定义另一个实体并将其命名为 Walk。 然后添加一个名为 date 的属性并将其类型设置为 Date。

![image-20221125014936290](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221125014936290.png)

回到 Dog Entity。 你可能认为你需要添加一个 Array 类型的新属性来保存 walks，但 Core Data 中没有数组类型。 相反，这样做的方法是将其建模为一种关系。 添加一个新的关系并将其命名为 walks。 将 destination 设置为 Walk：

![image-20221125015036471](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221125015036471.png)

你可以将 destination 视为一段关系的接收端。 默认情况下，每段关系都以一对一关系开始，这意味着你目前只能跟踪每只狗的一次散步。 除非你不打算长期饲养你的狗，否则你可能想要跟踪不止一次散步。

要解决此问题，请选择 walks 关系，打开 Data Model Inspector：

![image-20221125015232563](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221125015232563.png)

单击“Type”下拉菜单，选择“To Many”并选中“Ordered”。这意味着一只 Dog 可以有很多次 Walk，并且 Walk 的顺序很重要，因为你将显示按日期排序的 Walk。

选择 Walk 实体并创建返回到 Dog 的反向关系。将 destination 设置为 Dog，将 inverse 设置为 Walk。

![image-20221125015704013](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221125015704013.png)

将这种关系保留为一对一关系是可以的。一只 Dog 可以有很多次 Walk，但一次 Walk 只能属于一只 Dog——至少对于这个应用程序来说是这样。

逆向(inverse)让 model 知道如何找到返回的方式，可以这么说。鉴于 Walk 记录，你可以跟踪与狗的关系。多亏了 inverse， model 知道遵循 walks 关系以返回 walk 记录。

这是让你知道数据模型编辑器有另一种视图样式的好时机。你一直在关注表编辑器样式。

切换右下角的分段控件以切换到图形编辑器样式：

![image-20221125020546608](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221125020546608.png)

> 图形编辑器已经在 Xcode14 种删除。https://developer.apple.com/forums/thread/710008

图形编辑器是可视化 Core Data 实体之间关系的好工具。这里从 Dog 到 Walk 的对多关系用双箭头表示。 Walk 用一个箭头指向 Dog，表示一对一的关系。

随意在两种编辑器风格之间来回切换。你可能会发现使用表格样式添加和删除实体和属性更容易，使用图表样式可以更轻松地查看 Data Model 的大图。

### 添加 ManagedObject 子类

在上一章中，你学习了如何为你的 Core Data Entity 创建自定义 ManagedObject 子类。以这种方式工作更方便，因此这也是你将对 Dog 和 Walk 执行的操作。

与上一章一样，你将手动生成自定义 ManagedObject 子类，而不是让 Xcode 为你完成，这样你就可以看到幕后发生的事情。打开 DogWalk.xcdatamodeld，选择 Dog Entity 并将 Data Model inspector 中的代码生成下拉列表设置为 Manual/None。对 Walk 实体重复相同的过程。

然后，转到 Editor ▸ Create NSManagedObject Subclass... 并选择 DogWalk 模型，然后选择 Dog 和 Walk 实体。单击下一个屏幕上的创建以创建文件。

正如你在第 2 章中看到的那样，这样做会为每个实体创建两个文件：一个用于你在模型编辑器中定义的核心数据属性，另一个用于你可能添加到托管对象子类的任何未来功能。

Dog+CoreDataProperties.swift 应该是这样的：

```swift
import Foundation
import CoreData

extension Dog {

  @objc(insertObject:inWalksAtIndex:)
  @NSManaged public func insertIntoWalks(_ value: Walk,
                                         at idx: Int)

  @objc(removeObjectFromWalksAtIndex:)
  @NSManaged public func removeFromWalks(at idx: Int)

  @objc(insertWalks:atIndexes:)
  @NSManaged public func insertIntoWalks(_ values: [Walk],
                                         at indexes: NSIndexSet)

  @objc(removeWalksAtIndexes:)
  @NSManaged public func removeFromWalks(at indexes: NSIndexSet)
  @objc(replaceObjectInWalksAtIndex:withObject:)
  @NSManaged public func replaceWalks(at idx: Int,
                                      with value: Walk)

  @objc(replaceWalksAtIndexes:withWalks:)
  @NSManaged public func replaceWalks(at indexes: NSIndexSet,
                                      with values: [Walk])

  @objc(addWalksObject:)
  @NSManaged public func addToWalks(_ value: Walk)

  @objc(removeWalksObject:)
  @NSManaged public func removeFromWalks(_ value: Walk)

  @objc(addWalks:)
  @NSManaged public func addToWalks(_ values: NSOrderedSet)

  @objc(removeWalks:)
  @NSManaged public func removeFromWalks(_ values: NSOrderedSet)
}

extension Dog : Identifiable {

}
```

和以前一样，name 属性是一个 String 可选的。但是 Walk 的关系呢？ Core Data 使用 set 而不是数 array 来表示对多关系。因为你让 walks 关系有序，所以你有一个 NSOrderedSet。

> 注意：NSSet 似乎是一个奇怪的选择，不是吗？与 array 不同，set 不允许通过索引访问其成员。事实上，根本没有顺序！ Core Data 使用 NSSet 是因为 set 强制其成员之间的唯一性。在一对多关系中，同一个对象不能出现多次。

如果你需要按索引访问单个对象，你可以在可视化编辑器中选中 Ordered 复选框，就像你在此处所做的那样。然后，Core Data 会将关系表示为 NSOrderedSet。

同样，Walk+CoreDataProperties.swift 应该是这样的：

```swift
import Foundation
import CoreData

extension Walk {

  @nonobjc public class func fetchRequest()
    -> NSFetchRequest<Walk> {
    return NSFetchRequest<Walk>(entityName: "Walk")
  }

  @NSManaged public var date: Date?
  @NSManaged public var dog: Dog?
}

extension Walk : Identifiable {

}
```

回到 Dog 的反向关系只是 Dog 类型的一个属性。非常简单。

> 注意：有时 Xcode 会使用通用 NSManagedObject 类型而不是特定类创建关系属性，尤其是当你同时创建大量子类时。如果发生这种情况，只需自己更正类型或重新生成特定文件即可。

### 使用 Core Data

现在你的设置已经完成；你的 Core Data Stack，你的 Data Model 和你的 ManagedObject 子类。是时候将 DogWalk 转换为使用 Core Data 了。你之前已经做过几次，所以这对你来说应该是一个简单的部分。

假设此应用程序将在某个时候支持跟踪多只 Dog。第一步是跟踪当前选择的 Dog。

打开 ViewController.swift 并将 walks 数组替换为以下属性。现在忽略错误，你将在一分钟内修复这些错误：

```swift
var currentDog: Dog?
```

接下来，将以下代码添加到 viewDidLoad() 的末尾：

```swift
let dogName = "Fido"
let dogFetch: NSFetchRequest<Dog> = Dog.fetchRequest()
dogFetch.predicate = NSPredicate(format: "%K == %@",
                                 #keyPath(Dog.name), dogName)

do {
  let results = try coreDataStack.managedContext.fetch(dogFetch)
  if results.isEmpty {
    // Fido not found, create Fido
    currentDog = Dog(context: coreDataStack.managedContext)
    currentDog?.name = dogName
    coreDataStack.saveContext()
  } else {
    // Fido found, use Fido
    currentDog = results.first
  }
} catch let error as NSError {
  print("Fetch error: \(error) description: \(error.userInfo)")
}
```

首先，你从 Core Data 中获取所有名称为“Fido”的 Dog Entity。如果获取请求返回结果，则将第一个 Entity（应该只有一个）设置为当前选择的 Dog。

如果获取请求返回空结果，这可能意味着这是用户第一次打开应用程序。如果是这种情况，则插入一只新狗，将其命名为“Fido”，并将其设置为当前选择的狗。

> 注意：你刚刚实现了通常称为“查找”或“创建”模式的模式。此模式的目的是操作存储在 Core Data 中的对象，而不会冒在过程中添加重复对象的风险。在 iOS 9 中，Apple 引入了在 Core Data Entity 上指定唯一约束的功能。使用唯一约束，你可以在数据模型中指定哪些属性在实体上必须始终是唯一的，以避免添加重复项。

接下来，将 tableView(\_:numberOfRowsInSection:) 的实现替换为以下内容：

```swift
func tableView(_ tableView: UITableView,
               numberOfRowsInSection section: Int) -> Int {
  currentDog?.walks?.count ?? 0
}
```

正如你可能猜到的那样，这将表视图中的行数与当前所选狗中设置的行走次数相关联。如果当前没有选中的狗，则返回 0。

接下来，将 tableView(\_:cellForRowAt:) 替换为以下内容：

```swift
func tableView(
  _ tableView: UITableView,
  cellForRowAt indexPath: IndexPath
  ) -> UITableViewCell {
  let cell = tableView.dequeueReusableCell(
    withIdentifier: "Cell", for: indexPath)

  guard let walk = currentDog?.walks?[indexPath.row] as? Walk,
    let walkDate = walk.date as Date? else {
      return cell
  }

  cell.textLabel?.text = dateFormatter.string(from: walkDate)
  return cell
}
```

只有两行代码发生了变化。 现在，你获取每次步行的日期并将其显示在相应的表格视图单元格中。

add(\_:) 方法仍然引用旧的 walks 数组。 暂时将其注释掉； 你将在下一步中重新实现此方法：

```swift
@IBAction func add(_ sender: UIBarButtonItem) {
  // walks.append(Date())
  tableView.reloadData()
}
```

构建并运行以确保你已正确连接所有内容。

![image-20221126024711004](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126024711004.png)

万岁！如果你已经走到这一步，你刚刚将一只狗插入到 Core Data 中，并且当前正在用他的散步列表填充表格视图。这个列表目前没有任何走动，所以这个表看起来不是很令人兴奋。

点击加号 (+) 按钮，它什么也不做是可以理解的。你还没有在这个控件下实现任何东西！在转换到 Core Data 之前，add(\_:) 只是将 Date 添加到数组并重新加载表视图。重新实现如下图：

```swift
@IBAction func add(_ sender: UIBarButtonItem) {
  // Insert a new Walk entity into Core Data
  let walk = Walk(context: coreDataStack.managedContext)
  walk.date = Date()

  // Insert the new Walk into the Dog's walks set
  if let dog = currentDog,
    let walks = dog.walks?.mutableCopy()
      as? NSMutableOrderedSet {
      walks.add(walk)
      dog.walks = walks
  }

  // Save the managed object context
  coreDataStack.saveContext()

  // Reload table view
  tableView.reloadData()
}
```

这种方法的 Core Data 版本要复杂得多。首先，你必须创建一个新的 Walk Entiey 并将其日期属性设置为现在。接下来，你必须将此步行插入到当前选定的狗的步行列表中。

但是，walks 属性是 NSOrderedSet 类型。 NSOrderedSet 是不可变的，因此你首先必须创建一个可变副本 (NSMutableOrderedSet)，插入新的 walk，然后将这个可变有序集的不可变副本重置回狗身上。

> 注意：将新对象添加到一对多关系中是否会让你头晕目眩？许多人对此表示同情，这就是为什么 Dog+CoreDataProperties 包含生成的遍历有序集访问器的原因，这些访问器将为你处理所有这些。

例如，你可以将最后一个代码片段中的整个 if-let 语句替换为以下内容：

```swift
currentDog?.addToWalks(walk)
```

试试看！

不过，Core Data 可以让事情变得更容易。如果关系没有排序，你只能设置关系的一侧（例如，walk.dog = currentDog）而不是多侧，Core Data 将使用模型编辑器中定义的反向关系来将 Walk 添加到 Dog 的 Walk 中。

最后，你通过调用 Core Data 堆栈上的 saveContext() 将更改提交到持久存储，然后重新加载表视图。

构建并运行应用程序，然后点击加号 (+) 按钮几次。

walk 列表现在应该保存在 Core Data 中。通过在快速应用程序切换器中终止应用程序并从头开始重新启动来验证这一点。

![image-20221126025215464](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126025215464.png)

### 从 Core Data 中删除对象

假设你对触发器过于友好，并且在你无意的时候点击了加号 (+) 按钮。你实际上并没有遛狗，所以你想删除刚刚添加的散步。

你已经将对象添加到 Core Data，你已经获取它们，修改它们并再次保存它们。你还没有做的是删除它们——但你接下来要做的就是删除它们。

首先，打开 ViewController.swift 并将以下方法添加到 UITableViewDataSource 扩展：

```swift
func tableView(_ tableView: UITableView,
               canEditRowAt indexPath: IndexPath) -> Bool {
  true
}
```

你将使用 UITableView 的默认行为来删除项目：向左滑动以显示红色的删除按钮，然后点击它进行删除。

表格视图调用此 UITableViewDataSource 方法来询问特定单元格是否可编辑，返回 true 意味着所有单元格都应该是可编辑的。

接下来，将以下方法添加到相同的 UITableViewDataSource 扩展：

```swift
func tableView(
  _ tableView: UITableView,
  commit editingStyle: UITableViewCell.EditingStyle,
  forRowAt indexPath: IndexPath
) {

  //1
  guard let walkToRemove =
    currentDog?.walks?[indexPath.row] as? Walk,
    editingStyle == .delete else {
      return
  }

  //2
  coreDataStack.managedContext.delete(walkToRemove)

  //3
  coreDataStack.saveContext()

  //4
  tableView.deleteRows(at: [indexPath], with: .automatic)
}
```

当你点击红色的删除按钮时，将调用此表视图数据源方法。让我们逐步查看代码：

1. 首先，你获得对要删除的 walk 的引用。
2. 通过调用 NSManagedObjectContext 的 delete() 方法从 Core Data 中移除 walk。 Core Data 还负责从当前 dog 的 walks 关系中删除已删除的 walk。
3. 在你保存 managed object context 之前，任何更改都是最终的——甚至删除也不行！
4. 最后，如果保存操作成功，则为表视图设置动画以告知用户删除操作。

再次构建并运行该应用程序。你应该从之前的 walk 中选择任意一个并向左滑动。

![image-20221126025659781](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126025659781.png)

点击删除按钮以删除 Walk。通过终止应用程序并从头开始重新启动来验证步行是否真的消失了。你刚刚删除的 Walk 已永久消失。

> 注意：删除曾经是最“危险”的 Core Data 操作之一。为什么是这样？当你从 Core Data 中删除某些内容时，你必须同时删除磁盘上的记录以及代码中任何未完成的引用。

尝试访问没有 Core Data 后备存储的 NSManagedObject 会导致令人担忧的无法访问故障 Core Data 崩溃。

从 iOS 9 开始，删除比以往更安全。 Apple 在 NSManagedObjectContext 上引入了属性 shouldDeleteInaccessibleFaults，默认开启。这会标记数据删除，并将丢失的数据视为 NULL/nil/0。

### 关键点

* Core Data Stack 由五个类组成：NSManagedObjectModel、NSPersistentStore、NSPersistentStoreCoordinator、NSManagedObjectContext 和将所有内容放在一起的 NSPersistentContainer。
* managed object model 代表应用程序数据模型中的每个 data model、它们可以拥有的属性以及它们之间的关系。
* 持久存储可以由 SQLite 数据库（默认）、XML、二进制文件或内存存储支持。你还可以使用增量存储 API 提供你自己的后备存储。
* persistent store coordinator 隐藏了 persistent stores 如何配置的实现细节，并为你的 managed object context 提供了一个简单的接口。
* managed object context 管理它创建或获取的 managed object 的生命周期。它们负责获取、编辑和删除 managed object，以及更强大的功能，如验证、错误和反向关系处理。
