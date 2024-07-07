# 第 5 节：NSFetchedResultsController

如果你仔细阅读了前面的章节，你会注意到大多数示例项目都使用 UITableView。那是因为 Core Data 非常适合 UITableView。设置你的 fetch 请求，获取一组 ManagedObjects 并将结果插入 UITableView 的数据源。这是一个常见场景。

Apple 的 Core Data 框架的作者也是这么想的！事实上，他们看到了 UITableView 和 Core Data 之间紧密联系的巨大潜力，他们编写了一个类来正式化这种联系：NSFetchedResultsController。

顾名思义，NSFetchedResultsController 是一个 controller，但它不是 view controller。它没有用户界面。它的目的是通过抽象出将 UITableView 与 Core Data 支持的数据源同步所需的大部分代码，从而使开发人员的工作更轻松。

正确设置 NSFetchedResultsController，你的 table 将“神奇地”模仿其数据源，而你无需编写多行代码。在本节中，你将了解本课程的来龙去脉。你还将了解何时使用它以及何时不使用它。你准备好了吗？

### World Cup App

本章的示例项目是适用于 iOS 的世界杯记分牌应用程序。启动时，应用程序将列出所有参加世界杯的球队。点击一个国家的 cell 将使该国的胜利加一。在这个简化版的世界杯中，点击次数最多的国家将赢得比赛。这个排名大大简化了真正的淘汰规则，但对于演示目的来说已经足够了。

转到本章的文件并找到启动文件夹。打开 WorldCup.xcodeproj。构建并运行启动项目：

![image-20221128021131544](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221128021131544.png)

示例应用程序由表视图中的 20 个静态单元格组成。 那些明亮的蓝色方框是球队旗帜所在的地方。 你看到的不是真实 name，而是“Team name”。虽然示例项目不太令人兴奋，但它实际上为你做了很多设置。

打开项目导航器并查看启动项目中的完整文件列表：

![image-20221128021308502](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221128021308502.png)

在进入代码之前，让我们简要回顾一下每个开箱即用的类的功能。你会发现你在前面章节中手动完成的很多设置已经为你实现了。万岁！

* CoreDataStack：与前面的章节一样，这个对象包装了一个 NSPersistentContainer 的实例，它又包含核心数据对象的核心，称为“Stack”；the context、the model、the persistent store、the persistent store coordinator。不需要额外设置这个，它随时可用。
* ViewController：示例项目是一个单页应用程序，该文件代表该页面。第一次启动时，视图控制器从 seed.json 中读取，创建相应的 Core Data Objects 并将它们保存到 Persistent Store 中。如果你对其 UI 元素感到好奇，请前往 Main.storyboard。有一个 table、一个 navigation 和一个 prototype cell。
* Team+CoreDataClass & Team+CoreDataProperties：这些文件代表一个国家的球队。它是一个 NSManagedObject 子类，其四个属性中的每一个都有属性：teamName、qualifyingZone、imageName 和 wins。如果你对其实体定义感到好奇，请前往 WorldCup.xcdatamodel。
* Assets.xcassets：示例项目的资产目录包含 seed.json 中每个国家/地区的国旗图像。

本书的前三章涵盖了上述 Core Data 概念。如果“managed object subclass”没有引起你的注意，或者如果你不确定 Core Data Stack 应该做什么，你可能需要返回并重新阅读相关章节。如果你准备好继续，你将开始实施世界杯应用程序。你可能已经知道上次谁赢得了世界杯，但这是你为你选择的国家/地区改写历史的机会，只需轻点几下！

### 这一切都始于一个获取请求……

NSFetchedResultsController 的核心是 NSFetchRequest 的结果的包装器。现在，示例项目包含静态信息。你将创建一个fetched results controller 以在 tableview 中显示来自 Core Data 的 teams 列表。

打开 ViewController.swift 并添加一个 lazy 属性以将获取的结果控制器保存在 coreDataStack 下方：

```swift
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

与 NSFetchRequest 一样，NSFetchedResultsController 需要一个通用类型参数，在本例中为 Team，以指定你希望使用的 entity 类型。让我们一步一步地完成这个过程：

1.  fetchedResultsController 处理 Core Data 和你的 TableView 之间的协调，但它仍然需要你提供一个 NSFetchRequest。请记住 NSFetchRequest 类是高度可定制的。它可以采用 sort descriptors, predicates 等。

    在此示例中，你直接从 Team 类获取 NSFetchRequest，因为你想要获取所有 Team 对象。
2.  fetchedResultsController 的初始化方法有四个参数：首先是你刚刚创建的 fetch 请求。

    第二个参数是 NSManagedObjectContext 的实例。与 NSFetchRequest 一样，获取结果控制器类需要一个 managed object context 来执行 fetch。它实际上不能自己获取任何东西。

    其他两个参数是可选的：sectionNameKeyPath 和 cacheName。现在将它们留空；你将在本节后面阅读更多关于它们的信息。

接下来，将以下代码添加到 viewDidLoad() 的末尾以实际执行 fetch：

```swift
do {
  try fetchedResultsController.performFetch()
} catch let error as NSError {
  print("Fetching error: \(error), \(error.userInfo)")
}
```

在这里执行 fetch 请求。如果有错误，你将错误记录到控制台。

但是等一下……你获取的结果在哪里？使用 NSFetchRequest 获取返回结果数组，使用 NSFetchedResultsController 获取不返回任何内容。

NSFetchedResultsController 既是获取请求的包装器，也是获取结果的容器。你可以使用 fetchedObjects 属性或 object(at:) 方法获取它们。

接下来，你将把获取的 ResultsController 连接到常用的 tableView 数据源方法。获取的结果决定了部分的数量和每个部分的行数。

考虑到这一点，重新实现 numberOfSections(in:) 和 tableView(\_:numberOfRowsInSection:)，如下所示：

```swift
func numberOfSections(in tableView: UITableView) -> Int {
  fetchedResultsController.sections?.count ?? 0
}

func tableView(_ tableView: UITableView,
               numberOfRowsInSection section: Int)
               -> Int {
  guard let sectionInfo = 
    fetchedResultsController.sections?[section] else {
      return 0
  }

  return sectionInfo.numberOfObjects
}
```

tableView 中的 sections 数对应于 fetchedResultsController 中的 sections 数。你可能想知道这个 tableView 怎么可以有多个 section。你不是简单地获取并显示所有 Team 吗？

没错。这次你将只有一个部分，但请记住 NSFetchedResultsController 可以将你的数据分成多个 section。你将在本章后面看到这样的示例。

此外，每个 tableView section 中的 rows 对应于每个 fetchedResultsController section 中的 numberOfObjects。你可以通过其 sections 属性查询有关 fetchedResultsController section 的信息。

> 注意：sections 数组包含实现 NSFetchedResultsSectionInfo 协议的不透明对象。此轻量级协议提供有关 section 的信息，例如其 title 和 object 数。

实现 tableView(\_:cellForRowAt:) 通常是下一步。

然而，快速浏览一下该方法就会发现它已经在根据 TeamCell 需要调用 configure。你需要更改的是填充单元格的辅助方法。找到 configure(cell:for:) 并将其替换为以下内容：

```swift
func configure(cell: UITableViewCell,
               for indexPath: IndexPath) {
  guard let cell = cell as? TeamCell else {
      return
  }

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

此方法采用 cell 和 indexPath。你使用此 indexPath 从 fetchedResultsController 中获取相应的 Team 对象。

接下来，你使用 Team 对象填充 cell 的旗 teamLabel、scoreLabel 和 flagImageView。

再次注意没有数组变量保存你的 Team。它们都存储在 fetched results controller 中，你可以通过 object(at:) 访问它们。

是时候测试你的创作了。构建并运行应用程序。准备好，设置和......崩溃？

```
Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: 'An instance of NSFetchedResultsController requires a fetch request with sort descriptors'
```

发生了什么？ NSFetchedResultsController 正在帮助你解决这个问题，尽管你可能并不喜欢它！

如果你想用它来填充 tableview 并让它知道哪个 managed object 应该出现在哪个 indexPath，你不能只给它一个基本的获取请求。

它的最低要求是你设置一个 entity description，它将获取该 entity 类型的所有对象。然而，NSFetchedResultsController 至少需要一个排序描述符。否则，它怎么知道你的表视图的正确顺序？

回到 fetchedResultsController 惰性属性，在 let fetchRequest 之后添加如下几行：

```swift
let sort = NSSortDescriptor(
  key: #keyPath(Team.teamName),
  ascending: true)
fetchRequest.sortDescriptors = [sort]
```

添加此排序描述符将按字母顺序从 A 到 Z 显示团队并修复较早的崩溃。构建并运行应用程序。

![image-20221129021547911](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221129021547911.png)

成功！ 世界杯参赛者的完整名单在你的设备或 iOS 模拟器上。 但是请注意，每个国家的胜利都是零，并且没有办法增加分数。

#### 修改数据

让我们修复每个人的零分并添加一些代码来增加获胜次数。 仍然在 ViewController.swift 中，将表视图委托方法 tableView(\_:didSelectRowAt:) 的当前空实现替换为以下内容：

```swift
func tableView(_ tableView: UITableView,
               didSelectRowAt indexPath: IndexPath) {

  let team = fetchedResultsController.object(at: indexPath)
  team.wins += 1
  coreDataStack.saveContext()
}
```

当用户点击一行时，你将获取与 indexPath 对应的 Team，增加其获胜次数并将更改提交到 Core Data 的持久存储。

你可能认为 fetchedResultsController 仅适用于从 Core Data 获取结果，但你返回的 Team 对象是相同的 managed object 子类。 你可以像往常一样更新它们的值并保存。

再次构建并运行，并点击列表中的第一个国家（Algeria）几次：

![image-20221129021925232](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221129021925232.png)

这里发生了什么？ 你正在点击，但获胜的次数并没有增加。 你正在 Core Data 的底层持久存储中更新 Algeria 的获胜次数，但你没有触发 UI 刷新。 返回 Xcode，停止应用程序，然后再次构建并运行。

![image-20221129022028922](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221129022028922.png)

正如你所怀疑的那样，从头开始重新启动应用程序会强制刷新 UI，显示阿尔及利亚的真实得分为 6。NSFetchedResultsController 有一个很好的解决这个问题的方法，但现在，让我们使用暴力解决方案。

将以下行添加到 tableView(\_:didSelectRowAt:) 的末尾：

```swift
tableView.reloadData()
```

除了增加 Team 的获胜次数外，点击一个单元格现在会重新加载整个表格视图。 这种方法是笨拙的，但它现在可以完成工作。 再次构建并运行该应用程序。

点击任意多个国家/地区，点击次数不限。 验证 UI 是否始终是最新的。

![image-20221129022157077](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221129022157077.png)

你已经启动并运行了一个 fetchedResultsController。

如果这就是 NSFetchedResultsController 所能做的全部，你可能会感到有点失望。毕竟，你可以使用 NSFetchRequest 和一个简单的数组来完成同样的事情。

真正的魔力出现在本节的其余部分。 NSFetchedResultsController 在 Cocoa Touch 框架中占有一席之地，具有 section 处理和更改监控等功能，你将在接下来介绍这些功能。

### 将结果分为 sections

世界杯有六个资格赛区：非洲、亚洲、大洋洲、欧洲、南美洲和北美/中美洲。 Team 实体有一个名为 qualifyingZone 的字符串属性，用于存储此信息。

在本节中，你将把国家列表分成各自的区。 NSFetchedResultsController 使这变得非常简单。

让我们看看它的实际效果。返回到实例化 NSFetchedResultsController 的 lazy 属性，并对 fetchedResultsController 的初始化程序进行以下更改：

```swift
let fetchedResultsController = NSFetchedResultsController(
  fetchRequest: fetchRequest,
  managedObjectContext: coreDataStack.managedContext,
  sectionNameKeyPath: #keyPath(Team.qualifyingZone),
  cacheName: nil)
```

此处的不同之处在于你为可选的 sectionNameKeyPath 参数传递了一个值。你可以使用此参数来指定 fetchedResultsController 应该用于对结果进行分组和生成部分的属性。

这些部分究竟是如何生成的？每个唯一的属性值成为一个 section。 NSFetchedResultsController 然后将其获取的结果分组到这些部分中。在这种情况下，它将为 qualifyingZone 的每个唯一值生成 section，例如“Africa”、“Asia”、“Oceania”等。这正是你想要的！

> 注意：sectionNameKeyPath 采用 keyPath 字符串。它可以采用属性名称的形式，例如 qualifyingZone 或 teamName，也可以深入到 Core Data 关系中，例如 employee.address.street。使用#keyPath 语法来防止拼写错误和字符串类型的代码。

sectionNameKeyPath 现在将向表视图报告部分和行，但当前的 UI 看起来没有任何不同。要解决此问题，请将以下方法添加到 UITableViewDataSource 扩展：

```swift
func tableView(_ tableView: UITableView,
               titleForHeaderInSection section: Int)
               -> String? {
  let sectionInfo = fetchedResultsController.sections?[section]
  return sectionInfo?.name
}
```

实现此数据源方法会将 section 标题添加到 tableView 中，从而可以轻松查看一个 section 的结束位置和另一个 section 的开始位置。在这种情况下，该部分的标题来自区。和以前一样，这个信息直接来自 NSFetchedResultsSectionInfo 协议。

构建并运行应用程序。你的应用程序将如下所示：

![image-20221129023050358](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221129023050358.png)

向下滚动页面。 有好消息也有坏消息。 好消息是该应用程序占所有六个 section。 万岁！ 坏消息是世界颠倒了。

仔细看看这些部分。 你会在非洲看到阿根廷，在亚洲看到喀麦隆，在南美洲看到俄罗斯。 这怎么发生的？ 这不是数据的问题； 你可以打开 seed.json 并验证每个团队是否列出了正确的区。

你想出来了吗？ 国家列表仍然按字母顺序显示，fetchedResultsController 只是将 tableView 分成几个 section，就好像同一排位赛区的所有球队都分组在一起一样。

返回延迟实例化的 NSFetchedResultsController 属性并进行以下更改以解决问题。

用以下代码替换在获取请求上创建和设置排序描述符的现有代码：

```swift
let zoneSort = NSSortDescriptor(
  key: #keyPath(Team.qualifyingZone), 
  ascending: true)
let scoreSort = NSSortDescriptor(
  key: #keyPath(Team.wins), 
  ascending: false)
let nameSort = NSSortDescriptor(
  key: #keyPath(Team.teamName), 
  ascending: true)

fetchRequest.sortDescriptors = [zoneSort, scoreSort, nameSort]
```

问题是 sort descriptor。 这是另一个需要记住的 NSFetchedResultsController “陷阱”。 如果要使用 section keyPath 分隔获取的结果，则第一个排序描述符的属性必须与 keyPath 的属性相匹配。

NSFetchedResultsController 的文档强调了这一点，并且有充分的理由！ 你看到了当排序描述符与键路径不匹配时发生的情况——你的数据没有任何意义。

再构建并运行一次以验证此更改是否解决了问题：

![image-20221129023715430](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221129023715430.png)

确实如此。 更改排序描述符恢复了示例应用程序中的地缘政治平衡。 非洲球队在非洲，欧洲球队在欧洲等等。

> 请注意，在每个排位赛区内，球队按获胜次数从高到低排序，然后按名称排序。这是因为在前面的代码片段中，你添加了三个排序描述符：首先按排位赛区排序，然后按获胜次数排序，最后按名称排序。

在继续之前，花点时间想一想如果没有 fetchedResultsSontroller，你需要做些什么来按排位赛区分开团队。首先，你必须创建一个字典并遍历团队以找到唯一的区。

当你遍历团队数组时，你必须将每个团队与正确的排位赛区相关联。一旦你有了按区域划分的团队列表，你就必须对数据进行排序。

当然自己做也不是不可以，但是很繁琐。这就是 NSFetchedResultsController 让你免于做的事情。你可以在今天剩下的时间里休息，去海滩观看世界杯比赛。谢谢你，NSFetchedResultsController！

### 缓存 Cell

正如你可能想象的那样，将 Team 分组并不是一项成本低廉的操作。没有办法避免遍历每个 Team。

在这种情况下这不是性能问题，因为只有 32 个团队需要考虑。但是想象一下，如果你的数据集更大，会发生什么。如果你的任务是迭代超过 300 万个人口普查记录并按州或省将它们分开怎么办？

“我它放在后台线程上！”可能是你的第一个想法。但是，在所有 section 都可用之前，tableView 无法自行填充。你可能会避免阻塞主线程，但你仍然需要查看 spinner。不可否认，这成本很高。至少，你应该只支付一次成本：一次找出 section 分组，然后每次都重复使用你的结果。

NSFetchedResultsController 的作者思考了这个问题，想出了一个解决方案：缓存。你无需执行太多操作即可将其打开。

回到延迟实例化的 NSFetchedResultsController 并对获取的结果控制器初始化进行以下修改，向 cacheName 参数添加一个值：

```swift
let fetchedResultsController = NSFetchedResultsController(
  fetchRequest: fetchRequest,
  managedObjectContext: coreDataStack.managedContext,
  sectionNameKeyPath: #keyPath(Team.qualifyingZone),
  cacheName: "worldCup")
```

你指定一个缓存名称以打开 NSFetchedResultsController 的磁盘部分缓存。这就是你需要做的一切！请记住，这部分缓存完全独立于 Core Data 的 persistent store,，你在其中持久存储 Team。

> 注意：NSFetchedResultsController 的 section 缓存对其 fetch 请求的变化非常敏感。可以想象，任何更改（例如不同的实体描述或不同的排序描述符）都会为你提供一组完全不同的已提取对象，从而使缓存完全无效。如果你进行这样的更改，则必须使用 deleteCache(withName:) 删除现有缓存或使用不同的缓存名称。

构建并运行应用程序几次。第二次应该比第一次快一点，这是 NSFetchedResultsController 的缓存系统在工作！

在第二次启动时，NSFetchedResultsController 直接从缓存中读取。这节省了到 Core Data 持久存储的往返，以及计算这些部分所需的时间。万岁！

在第 8 节“衡量和提升性能”中，你将了解如何衡量性能并了解你的代码更改是否真的让事情变得更快。

在你自己的应用程序中，如果你将结果分组到多个 section 并且拥有非常大的数据集或针对较旧的设备，请考虑使用 NSFetchedResultsController 的缓存。

### 监控变化

本节已经涵盖了使用 NSFetchedResultsController 的三个主要好处中的两个：section 和 cache。第三个也是最后一个好处有点像一把双刃剑：它很强大但也很容易被误用。

在本章的前面，当你实现点击以增加获胜次数时，你添加了一行代码来重新加载表视图以显示更新后的分数。这是一个蛮力解决方案，但它奏效了。

当然，你可以通过巧妙地使用 UITableView API 来仅重新加载选定的 cell，但这并不能解决根本问题。

不要过于哲学化，但根本问题是改变。底层数据发生了一些变化，你必须明确重新加载用户界面。

想象一下世界杯应用程序的第二个版本会是什么样子。也许每个团队都有一个详细信息屏幕，你可以在其中更改分数。

也许应用程序调用 API 端点并从网络服务获取新的分数信息。你的工作是为每个更新基础数据的代码路径刷新 tableView。

明确地这样做很容易出错，更不用说有点无聊了。有没有更好的办法？就在这里 fetched results controller 来救援。

NSFetchedResultsController 可以侦听其结果集中的变化并通知其委托 NSFetchedResultsControllerDelegate。每当基础数据发生变化时，你都可以根据需要使用此委托来刷新 tableView。

fetched results controller 可以监控其“结果集”的变化是什么意思？这意味着除了已经获取的对象之外，它还可以监控所有对象的变化，无论是旧的还是新的。这种区别将在本节后面变得更加清楚。

让我们在实践中看看。还是在 ViewController.swift 中，将以下扩展名添加到文件底部：

```swift
// MARK: - NSFetchedResultsControllerDelegate
extension ViewController: NSFetchedResultsControllerDelegate {

}
```

这只是告诉编译器 ViewController 类将实现一些获取结果控制器的委托方法。

接下来，回到你的 lazy NSFetchedResultsController 属性并在返回之前将 viewController 设置为 fetched results controller 的委托。在初始化 fetched results controller 后添加以下代码行：

```swift
fetchedResultsController.delegate = self
```

这就是开始监视更改所需的全部内容！当然，下一步是在收到这些更改报告时做一些事情。接下来你会做的。

> 注意： fetchedResultsController 只能监视通过其初始化程序中指定的 ManagedObjectContext 所做的更改。如果你在应用程序的其他地方创建一个单独的 NSManagedObjectContext 并开始在那里进行更改，那么你的委托方法将不会运行，直到这些更改被保存并与 fetchedResultsController 的 context 合并。

### 应对变化

首先，从 tableView(\_:didSelectRowAt:) 中删除 reloadData() 调用。如前所述，这是你现在要替换的蛮力方法。

NSFetchedResultsControllerDelegate 有四种不同粒度的方法。首先，实施最广泛的委托方法，即说：“嘿，有些东西刚刚改变了！”

在 NSFetchedResultsControllerDelegate 扩展中添加以下方法：

```swift
func controllerDidChangeContent(_ controller: 
  NSFetchedResultsController<NSFetchRequestResult>) {
    tableView.reloadData()
}
```

更改可能看起来很小，但实施此方法意味着任何更改，无论来源如何，都会刷新表视图。构建并运行应用程序。通过点击几个单元格来验证表视图的单元格是否仍然正确更新：

![image-20221130020320664](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221130020320664.png)

分数标签像以前一样更新，但还有其他事情发生。 当一个国家的积分多于同一排位赛区的另一个国家时，该国家将“跳”上一个级别。 这是 fetchedResultsController，它注意到抓取结果的排序顺序发生了变化，并相应地重新调整表视图的数据源。

当单元格确实四处移动时，它会非常紧张……几乎就像每次发生变化时你都在完全重新加载表格一样。

接下来，你将从重新加载整个表格到仅刷新需要更改的内容。 获取结果控制器委托可以告诉你是否需要移动、插入或删除特定索引路径，因为获取结果控制器的结果集发生了变化。

将 NSFetchedResultsControllerDelegate 扩展的内容替换为以下三个委托方法以查看实际效果：

```swift
func controllerWillChangeContent(_ controller:
  NSFetchedResultsController<NSFetchRequestResult>) {
    tableView.beginUpdates()
}

func controller(_ controller:
  NSFetchedResultsController<NSFetchRequestResult>,
  didChange anObject: Any,
  at indexPath: IndexPath?,
  for type: NSFetchedResultsChangeType,
  newIndexPath: IndexPath?) {

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

让我们简要回顾一下你刚刚添加或修改的所有三种方法。

* controllerWillChangeContent(\_:)：此委托方法通知你即将发生更改。你使用 beginUpdates() 准备好你的 tableView。
*   controller(\_:didChange:at:for:newIndexPath:): 这个方法比较啰嗦。并且有充分的理由——它准确地告诉你哪些对象发生了变化，发生了什么类型的变化（插入、删除、更新或重新排序）以及受影响的 indexPath 是什么。

    这个中间方法是众所周知的将表格视图与 Core Data 同步的粘合剂。无论底层数据发生多少变化，你的 tableView 都将与 persistent store 中发生的事情保持一致。
* controllerDidChangeContent(\_:)：你最初实现的用于刷新 UI 的委托方法结果是通知你更改的三个委托方法中的第三个。无需刷新整个表视图，你只需调用 endUpdates() 来应用更改。

> 注意：你最终如何处理更改通知取决于你的个人应用程序。你在上面看到的实现是 Apple 在 NSFetchedResultsControllerDelegate 文档中提供的示例。

请注意，这些方法的顺序和性质非常巧妙地与用于更新表视图的“开始更新、进行更改、结束更新”模式相关联。这不是巧合！

构建并运行以查看你的工作成果。马上，每个排位赛区都会根据获胜次数列出球队。点击不同的国家几次。你会看到 cell 平滑地动画以维持此顺序。

本节还有一个 NSFetchedResultsControllerDelegate 方法需要探索。将其添加到扩展中：

```swift
func controller(_ controller: 
  NSFetchedResultsController<NSFetchRequestResult>,
  didChange sectionInfo: NSFetchedResultsSectionInfo,
  atSectionIndex sectionIndex: Int,
  for type: NSFetchedResultsChangeType) {

  let indexSet = IndexSet(integer: sectionIndex)

  switch type {
  case .insert:
    tableView.insertSections(indexSet, with: .automatic)
  case .delete:
    tableView.deleteSections(indexSet, with: .automatic)
  default: break
  }
}
```

此委托方法与 controllerDidChangeContent(\_:) 类似，但会通知你对部分而不是单个对象的更改。在这里，你将处理基础数据的更改触发整个部分的创建或删除的情况。

花点时间想想什么样的变化会触发这些通知。也许如果一支新球队从一个全新的资格赛区进入世界杯，则 fetchedResultsController 会利用该值的唯一性并通知其 delegate 有关新 section 的信息。

### 插入一个 section

为了演示在结果集中插入时表视图会发生什么，让我们假设有一种添加新团队的方法。

如果你仔细观察，你可能会注意到右上角的 + 栏按钮项。 它一直被禁用。

让我们现在来实现它。 在 ViewController.swift 中，在 viewDidLoad() 下面添加以下方法：

```swift
override func motionEnded(
  _ motion: UIEvent.EventSubtype,
  with event: UIEvent?) {
  if motion == .motionShake {
    addButton.isEnabled = true
  }
}
```

你覆盖 motionEnded(\_:with:) 以便摇动设备启用 + 条按钮项。 这将是你进入的秘密方式。 addButton 属性一直持有对该栏按钮项的引用！

接下来，在标有 // MARK 的扩展名上方添加以下扩展名： - Internal：

```swift
// MARK: - IBActions
extension ViewController {
  @IBAction func addTeam(_ sender: Any) {
    let alertController = UIAlertController(
      title: "Secret Team",
      message: "Add a new team",
      preferredStyle: .alert)

    alertController.addTextField { textField in
      textField.placeholder = "Team Name"
    }
    
    alertController.addTextField { textField in
      textField.placeholder = "Qualifying Zone"
    }
    
    let saveAction = UIAlertAction(
      title: "Save",
      style: .default
    ) { [unowned self] _ in
    
      guard 
        let nameTextField = alertController.textFields?.first,
        let zoneTextField = alertController.textFields?.last
        else {
          return
      }
    
      let team = Team(
        context: self.coreDataStack.managedContext)
    
      team.teamName = nameTextField.text
      team.qualifyingZone = zoneTextField.text
      team.imageName = "wenderland-flag"
      self.coreDataStack.saveContext()
    }
    
    alertController.addAction(saveAction)
    alertController.addAction(UIAlertAction(title: "Cancel",
                                            style: .cancel))
    
    present(alertController, animated: true)

  }
}
```

这**是**一个相当长但易于理解的方法。 当用户点击添加按钮时，它会显示一个 alert controller，提示用户输入一个 Team。

alert controller 有两个文本字段：一个用于输入球队名称，另一个用于进入排位赛区。点击保存提交更改并将新团队插入到 Core Data 的 persistent store 中。再次构建并运行该应用程序。

如果你在设备上跑步，请摇晃它。如果你在模拟器上运行，请按 Command + Control + Z 来模拟摇晃事件。

![image-20221130021723849](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221130021723849.png)

芝麻开门！“摇一摇”，添加按钮现在处于活动状态！

世界杯正式接受一支新球队。向下滚动表格至欧洲资格区的末尾和北美、中美洲和加勒比地区资格区的开头。你马上就会明白为什么。

在继续之前，请花几秒钟时间接受这一点。你将通过将另一支球队加入世界杯来改变历史。你准备好了吗？

点击右上角的 + 按钮。你会看到一个警报视图，询问新 Team 的详细信息。

![image-20221130021758676](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221130021758676.png)

作为新 Team 进入虚构（但繁荣）的 Wenderland 合条件的区域键入 Internets，然后点击保存。 快速动画后，你的用户界面应如下所示：

![image-20221130021957655](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221130021957655.png)

由于“Internets”是获取结果控制器的 sectionNameKeyPath 的新值，因此此操作创建了一个新部分并将一个新 Team 添加到 fetched results controller 结果集中。

它处理事物的数据方面。此外，由于你适当地实现了 fetched results controller 的委托方法，因此 tableView 通过插入一个包含一个新行的新 section 来响应。

这就是 NSFetchedResultsControllerDelegate 的美妙之处。你可以设置一次并忘记它。底层数据源和你的 tableView 将始终同步。

至于 Wenderland 旗帜是如何进入应用程序的：嘿，我们是开发者！我们需要为各种可能性做好计划。

### Diffable data sources

在 iOS 13 中，Apple 引入了一种实现 tableView 和 collectionView 的新方法：Diffable data sources。无需实现常用的数据源方法，如 numberOfSections(in:) 和 tableView(\_:cellForRowAt:) 来提供部分信息和单元格，使用 Diffable data sources，你可以使用快照提前设置 section 和 row。

除了 Diffable data sources，还有一种使用 NSFetchedResultsController 来监控获取请求结果集变化的新方法。

让我们从示例项目中删除现有的数据源实现开始。继续并删除符合 UITableViewDataSource 的整个 ViewController 扩展。添加 // MARK: - UITableViewDataSource。

然后，滚动到 ViewController 的顶部并添加以下属性：

```swift
var dataSource: UITableViewDiffableDataSource<String, NSManagedObjectID>?
```

UITableViewDiffableDataSource 对于两种类型是通用的——String 表示部分标识符，NSManagedObjectID 表示不同 Team 的 managed object 标识符。

接下来，在 configure(cell:for) 下面添加以下新方法：

```swift
func setupDataSource()
  -> UITableViewDiffableDataSource<String, NSManagedObjectID> {
    UITableViewDiffableDataSource(
    tableView: tableView
    ) { [unowned self] (tableView, indexPath, managedObjectID) 
      -> UITableViewCell? in

      let cell = tableView.dequeueReusableCell(
        withIdentifier: self.teamCellIdentifier,
        for: indexPath)
    
      if let team =
          try? coreDataStack.managedContext.existingObject(
            with: managedObjectID) as? Team {
        self.configure(cell: cell, for: team)
      }
      return cell
    }

}
```

此方法创建你的 diffable data source。 当像这样创建数据源时，它会自动将自己添加为 tableView 的数据源。 请注意，你传入了一个用于配置 cell 的闭包，而不是一个单独的方法。

由于数据源对于 NSManagedObjectID 是通用的，因此你使用 existingObject(with:) 将标识符转换为相应的 Team 对象以配置每个 cell。

由于你在数据源闭包中解析了 Team 对象，因此你需要重新实现 configure(cell:for)。 用以下内容替换其实现：

```swift
  func configure(cell: UITableViewCell,
                 for team: Team) {

    guard let cell = cell as? TeamCell else {
        return
    }
    
    cell.teamLabel.text = team.teamName
    cell.scoreLabel.text = "Wins: \(team.wins)"
    
    if let imageName = team.imageName {
      cell.flagImageView.image = UIImage(named: imageName)
    } else {
      cell.flagImageView.image = nil
    }

  }
```

接下来，将以下内容添加到 importJSONSeedDataIfNeeded() 之后的 viewDidLoad() 中

```swift
dataSource = setupDataSource()
```

在之前的设置中，表视图的数据源是视图控制器。表视图数据源现在是你之前设置的可比较数据源对象。

现在找到 NSFetchedResultsControllerDelegate 实现并删除你在上一节中设置的所有四个委托方法：

* controllerWillChangeContent(_:)_
* controller(\_:didChangeContentWith:)
* controllerDidChangeContent(\_:)
* controller(didChange:atSectionIndex:for:)

取而代之的是，实现以下委托方法：

```swift
func controller(
  _ controller: NSFetchedResultsController<NSFetchRequestResult>,
  didChangeContentWith
  snapshot: NSDiffableDataSourceSnapshotReference) {

  let snapshot = snapshot
    as NSDiffableDataSourceSnapshot<String, NSManagedObjectID>
  dataSource?.apply(snapshot)
}
```

之前委托调用与 UITableView 中的方法很好地对齐，例如 beginUpdates() 和 endUpdates()，你不再需要调用它们，因为你已切换到 diffable data source。

相反，新的委托方法为你提供了对获取的结果集的任何更改的摘要，并向你传递了一个预先计算的快照，你可以将其直接应用于你的 tableView。简单多了！

构建并运行以查看新的可差异快照所处的位置：

![image-20221201021232931](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221201021232931.png)

伟大的！似乎大多数事情都有效，但有两个问题。首先，控制台警告你 tableView 在显示在屏幕上之前正在布置其单元格，其次是 team 似乎按区分组，但 section 标题不见了。

控制台警告正在发生，因为现在事情以不同的顺序发生。当 viewController 是表的数据源，并且你正在实现旧的获取结果控制器委托方法时，table 在加载并添加到屏幕之前不会请求任何信息。现在你正在使用一个 diffable data source，当你在 results controller 上调用 performFetch() 时会发生第一个更改， results controller 又调用 controller(\_: didChangeContentWith:)，它从第一行开始“添加”所有行拿来。你在 viewDidLoad() 中调用 performFetch()，这发生在将视图添加到窗口之前。

要解决此问题，你需要稍后执行第一次提取。从 viewDidLoad() 中删除 do / catch 语句，因为它现在在生命周期中发生得太早了。实现 viewDidAppear(\_:)，在视图添加到窗口后调用：

```swift
override func viewDidAppear(_ animated: Bool) {
  super.viewDidAppear(animated)
  UIView.performWithoutAnimation {
    do {
      try fetchedResultsController.performFetch()
    } catch let error as NSError {
      print("Fetching error: \(error), \(error.userInfo)")
    }
  }
}
```

构建并运行，控制台警告消失了。 现在修复 section 标题。

他们为什么失踪了？ 早些时候，当你移除 UITableViewDataSource 的实现时，你也移除了 tableView(\_:titleForHeaderInSection:)。 此方法提供了填充节标题的字符串，如果没有这些字符串，标题就会消失。

无法使用 UITableViewDiffableDataSource 重新打开这些标头，因此你将采用替代方法。 找到实现 UITableViewDelegate 方法的部分并实现这两个：

```swift
func tableView(_ tableView: UITableView,
               viewForHeaderInSection section: Int) -> UIView? {

  let sectionInfo = fetchedResultsController.sections?[section]

  let titleLabel = UILabel()
  titleLabel.backgroundColor = .white
  titleLabel.text = sectionInfo?.name

  return titleLabel
}

func tableView(_ tableView: UITableView,
               heightForHeaderInSection section: Int)
  -> CGFloat {
  20
}
```

这两个委托方法不是只返回填充 setion header 的标题，而是创建并返回 UILabel 以与节标题的高度一起显示。

构建并运行以查看是否带回了丢失的 header：

![image-20221201021840421](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221201021840421.png)

section header 回来了，但如果你点击任何 team cell，你会注意到获胜次数不再增加。 diffable 数据源只考虑哪些对象 ID 在哪个部分中的顺序。即使你的团队因为胜利增加而在该部分中上升，数据源也只会移动现有单元而不是重新配置它。只有当单元格移出屏幕并再次打开时，你才会看到新分数。

要解决此问题，请在 UITableViewDelegate 部分找到 tableView(\_:didSelectRowAt:) 并在调用 saveContext() 之前添加以下代码：

```swift
if var snapshot = dataSource?.snapshot() {
  snapshot.reloadItems([team.objectID])
  dataSource?.apply(snapshot, animatingDifferences: false)
}
```

在这里，你获取现有快照，告诉它你的团队需要重新加载，然后将更新后的快照应用回数据源。然后数据源将为你的 team 重新加载 cell。当你保存上下文时，这将触发获取的结果控制器的委托方法，该方法将应用任何需要发生的重新排序。再次构建并运行并确认一切都像宣传的那样工作。

如果你走到这一步，请拍拍自己的背。你不仅使用不同的数据源重新实现示例项目，而且还使用新的获取结果控制器委托方法对监控更改的方式进行了现代化改造。在此过程中，你还删除了许多以前需要的样板文件。

> 注意：如果你正在更改不支持 diffable data sources 的视图的状态，你应该记住还有另一个 NSFetchedResultsControllerDelegate 方法可以一次性为你提供对获取结果的所有更改的摘要，但使用CollectionDifference 返回结果。

### 关键点

* NSFetchedResultsController 抽象掉了将 tableView 与 CoreData store 同步所需的大部分代码。
* NSFetchedResultsController 的核心是 NSFetchRequest 的包装器和获取结果的容器。
* fetched results controller 需要在其获取请求上设置至少一个排序描述符。如果你忘记了排序描述符，你的应用程序将会崩溃。
* 你可以设置获取结果的控制器 sectionNameKeyPath 以指定一个属性以将结果分组到 section Section。每个唯一值对应于不同的表视图部分。
* 将一组获取的结果分组为多个 section 是一项昂贵的操作。通过在获取的结果控制器上指定缓存名称，避免多次计算部分。
* FetchedResultsController可以侦听其结果集中的变化并通知其代理 NSFetchedResultsControllerDelegate 响应这些变化。
* NSFetchedResultsControllerDelegate 监视单个 Core Data 记录中的更改（无论是插入、删除还是修改）以及对整个部分的更改。
* DiffableDataSource 使使用 FetchedResultsController 和 tableView 更容易。

### 然后去哪儿？

你已经看到了 NSFetchedResultsController 是多么强大和有用，并且你已经了解了它与 table view 一起工作的效果。table view 在 iOS 应用程序中非常常见，你已经亲眼目睹了 fetched results 控制器如何为你节省大量时间和代码！

通过对委托方法进行一些调整，你还可以使用 fetchedResultsController 来驱动 collectionView——主要区别在于 collectionView 不将其更新包含在开始和结束调用中，因此有必要存储更改和最后批量应用它们。

在其他上下文中使用 fetched results controller 之前，你应该记住一些事情。请注意如何实现 fetched results controller 委托方法。即使基础数据发生最轻微的变化也会触发这些变化通知，因此请避免执行任何你不愿意反复执行的昂贵操作。

NSFetchedResultsController 之所以重要还有另一个原因：它填补了 iOS 开发人员与 macOS 开发人员相比所面临的空白。与 iOS 不同，macOS 具有 Cocoa 绑定，它提供了一种将视图与其底层数据模型紧密耦合的方法。听起来有点熟？

如果你发现自己编写了复杂的逻辑来计算 section，或者为了让你的 tableView 与 Core Data 很好地配合而大费周章，请回想一下本章！
