# 第 9 节：多 ManagedObjectContext

ManagedObjectContext 是用于处理 managed objects 的内存暂存器。

大多数应用程序只需要一个 ManagedObjectContext。 大多数 Core Data 应用程序中的默认配置是与主队列关联的单个 ManagedObjectContext。 多个 ManagedObjectContext 使你的应用程序更难调试； 它不是你会在每种情况下在每个应用程序中使用的东西。

也就是说，某些情况确实需要使用多个托管对象上下文。 例如，长时间运行的任务（如导出数据）将阻塞仅使用单个主队列托管对象上下文的应用程序的主线程，并导致 UI 卡顿。

在其他情况下，例如在对用户数据进行编辑时，将 ManagedObjectContext 视为一组更改是有帮助的，应用程序可以在不再需要它们时丢弃这些更改。 使用子上下文使这成为可能。

在本章中，你将通过为 Surfer 制作一个日记应用程序并通过添加多个 Context 以多种方式改进它来了解多 ManagedObjectContext。

### 入门

本章的入门项目是一个面向 Surfer 的简单日记应用程序。 每次冲浪后，Surfer 可以使用该应用程序创建一个新的日记条目，记录海浪参数，例如涌浪高度或周期，并从 1 到 5 对冲浪进行评分。

### 介绍 SurfJournal

转到本章的文件并找到 SurfJournal 启动项目。 打开项目，然后构建并运行应用程序。

在启动时，应用程序会列出所有以前的冲浪会话日志条目。 点击一行会显示冲浪会话的详细视图，并可以进行编辑。

如你所见，示例应用程序可以正常运行并具有数据。 点击左上角的导出按钮将数据导出到逗号分隔值 (CSV) 文件。 点击右上角的加号 (+) 按钮可添加新的日记条目。 点击列表中的一行会在编辑模式下打开条目，你可以在其中更改或查看冲浪会话的详细信息。

虽然示例项目看起来很简单，但它实际上做了很多事情，可以作为添加多上下文支持的良好基础。 首先，让我们确保你对项目中的各个类有很好的理解。

打开项目导航器并查看启动项目中的完整文件列表：

![image-20230312210229092](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312210229092.png)

在开始编写代码之前，请花点时间回顾一下每个类开箱即用的功能。 如果你已经完成了前面的章节，你应该会发现这些类中的大部分都很熟悉：

* AppDelegate：首次启动时，app delegate 创建 Core Data Stack 并在主视图控制器 JournalListViewController 上设置 coreDataStack 属性。
* CoreDataStack：与前面的章节一样，这个对象包含 Core Data 对象的核心，称为 Stack。 与之前的章节不同，这次 Stack 安装了一个数据库，该数据库在首次启动时已经包含数据。 现在还不用担心这个； 你很快就会看到它是如何工作的。
* JournalListViewController：示例项目是一个单页的、基于表格的应用程序。 该文件代表该表。 如果你对其 UI 元素感到好奇，请前往 Main.storyboard。 导航控制器中嵌入了一个表视图控制器和一个类型为 SurfEntryTableViewCell 的原型单元格。
* JournalEntryViewController：此类处理创建和编辑冲浪日记条目。 你可以在 Main.storyboard 中看到它的 UI。
* JournalEntry：此类代表冲浪日记条目。 它是一个 NSManagedObject 子类，具有六个属性属性：日期、高度、位置、周期、评级和风。 如果你对此类的实体定义感到好奇，请查看 SurfJournalModel.xcdatamodel。
* JournalEntry+Helper：这是对 JournalEntry 对象的扩展。 它包括 CSV 导出方法 csv() 和 stringForDate() 辅助方法。 这些方法在扩展中实现，以避免在你更改 CoreDataModel 时被破坏。

当你第一次启动该应用程序时，已经有大量数据。

虽然前面一些章节中的项目从 JSON 文件导入种子数据，但这个示例项目带有一个种子核心数据数据库。

#### Core Data stack

打开CoreDataStack.swift，在 seedCoreDataContainerIfFirstLaunch() 中找到如下代码：

```swift
// 1
let previouslyLaunched =
  UserDefaults.standard.bool(forKey: "previouslyLaunched")
if !previouslyLaunched {
  UserDefaults.standard.set(true, forKey: "previouslyLaunched")

  // Default directory where the CoreDataStack will store its files
  let directory = NSPersistentContainer.defaultDirectoryURL()
  let url = directory.appendingPathComponent(
    modelName + ".sqlite")

  // 2: Copying the SQLite file
  let seededDatabaseURL = Bundle.main.url(
    forResource: modelName,
    withExtension: "sqlite")!

  _ = try? FileManager.default.removeItem(at: url)

  do {
    try FileManager.default.copyItem(at: seededDatabaseURL,
                                     to: url)
  } catch let nserror as NSError {
    fatalError("Error: \(nserror.localizedDescription)")
  }
```

1. 首先检查 UserDefaults 中是否有 previouslyLaunched 布尔值。 如果当前执行确实是应用程序的第一次启动，则 Bool 将为假，使 if 语句为真。 首次启动时，你要做的第一件事是将 previouslyLaunched 设置为 true，这样播种操作就不会再发生。
2. 然后，将包含在应用程序包中的 SQLite 种子文件 SurfJournalModel.sqlite 复制到 CoreData 提供的方法 NSPersistentContainer.defaultDirectoryURL() 返回的目录中。

现在查看 seedCoreDataContainerIfFirstLaunch() 的其余部分：

```swift
  // 3: Copying the SHM file
  let seededSHMURL = Bundle.main.url(forResource: modelName,
    withExtension: "sqlite-shm")!
  let shmURL = directory.appendingPathComponent(
    modelName + ".sqlite-shm")

  _ = try? FileManager.default.removeItem(at: shmURL)

  do {
    try FileManager.default.copyItem(at: seededSHMURL,
                                     to: shmURL)
  } catch let nserror as NSError {
    fatalError("Error: \(nserror.localizedDescription)")
  }

  // 4: Copying the WAL file
  let seededWALURL = Bundle.main.url(forResource: modelName,
    withExtension: "sqlite-wal")!
  let walURL = directory.appendingPathComponent(
    modelName + ".sqlite-wal")

  _ = try? FileManager.default.removeItem(at: walURL)

  do {
    try FileManager.default.copyItem(at: seededWALURL,
                                     to: walURL)
  } catch let nserror as NSError {
    fatalError("Error: \(nserror.localizedDescription)")
  }

  print("Seeded Core Data")
}
```

1. SurfJournalModel.sqlite 复制成功后，复制支持文件 SurfJournalModel.sqlite-shm。
2. 最后，复制剩余的支持文件 SurfJournalModel.sqlite-wal。

SurfJournalModel.sqlite、SurfJournalModel.sqlite-shm 或 SurfJournalModel.sqlite-wal 在首次启动时无法复制的唯一原因是发生了一些非常糟糕的事情，例如宇宙辐射造成的磁盘损坏。 在这种情况下，设备（包括任何应用程序）也可能会失败。 如果文件复制失败，则继续没有意义，因此 catch 块调用 fatalError。

> 注意：开发人员通常不赞成使用 abort 和 fatalError，因为它会导致应用程序突然退出且没有任何解释，从而使用户感到困惑。 这是可以接受 fatalError 的一种情况，因为应用程序需要 Core Data 才能工作。 如果一个应用程序需要 Core Data 而 Core Data 不工作，那么让应用程序继续运行是没有意义的，只会在稍后以不确定的方式失败。

调用 fatalError 至少会生成堆栈跟踪，这在尝试解决问题时会很有帮助。 如果你的应用程序支持远程日志记录或崩溃报告，你应该在调用 fatalError 之前记录任何可能有助于调试的相关信息。

为了支持并发读取和写入，此示例应用程序中的持久性 SQLite 存储使用 SHM（共享内存文件）和 WAL（预写日志记录）文件。 你不需要知道这些额外的文件是如何工作的，但你需要知道它们的存在，并且你需要在为数据库做种时复制它们。 如果你未能复制这些文件，该应用程序将运行，但它可能会丢失数据。

现在你已经了解了从种子数据库开始的一些知识，你将通过在临时私有上下文中工作来了解多个 ManagedObjectContext。

### 在后台工作

如果你还没有这样做，请点击左上角的“导出”按钮，然后立即尝试滚动浏览会话日志条目列表。 注意到什么了吗？ 导出操作需要几秒钟，它会阻止 UI 响应滚动等触摸事件。

UI 在导出操作期间被阻塞，因为导出操作和 UI 都在使用主队列来执行它们的工作。 这是默认行为。

解决此问题的传统方法是使用 Grand Central Dispatch 在后台队列上运行导出操作。 然而，Core Data 管理的ManagedObjectContext 不是线程安全的。 这意味着你不能只派发到后台队列并使用相同的 Core Data stack。

解决方案很简单：使用私有后台队列而不是导出操作的主队列。 这将使主队列保持空闲以供 UI 使用。 但在你着手解决问题之前，你需要了解导出操作的工作原理。

#### 导出数据

首先查看应用程序如何为 JournalEntry 实体创建 CSV 字符串。 打开 JournalEntry+Helper.swift 找到 csv()：

```swift
func csv() -> String {
  let coalescedHeight = height ?? ""
  let coalescedPeriod = period ?? ""
  let coalescedWind = wind ?? ""
  let coalescedLocation = location ?? ""
  let coalescedRating: String
  if let rating = rating?.int16Value {
    coalescedRating = String(rating)
  } else {
    coalescedRating = ""
  }

  return [
    stringForDate(),
    coalescedHeight,
    coalescedPeriod,
    coalescedWind,
    coalescedLocation,
    coalescedRating,
    "\n"
  ].joined(separator: ",")
}
```

如你所见，JournalEntry 返回以逗号分隔的实体属性字符串。 因为允许 JournalEntry 属性为 nil，该函数使用 nil 合并运算符 (??) 来导出空字符串，而不是无用的调试消息，即属性为 nil。

> 注意：nil 合并运算符 (??) 解包一个包含值的可选； 否则返回默认值。 例如，以下内容： let coalescedHeight = height != nil ? height! "" 可以使用 nil 合并运算符缩短为：let coalescedHeight = height ?? ""。

这就是应用程序如何为单个日记条目创建 CSV 字符串，但应用程序如何将 CSV 文件保存到磁盘？ 打开 JournalListViewController.swift，在 exportCSVFile() 中找到如下代码：

```swift
// 1
let context = coreDataStack.mainContext
var results: [JournalEntry] = []
do {
  results = try context.fetch(self.surfJournalFetchRequest())
} catch let error as NSError {
  print("ERROR: \(error.localizedDescription)")
}

// 2
let exportFilePath = NSTemporaryDirectory() + "export.csv"
let exportFileURL = URL(fileURLWithPath: exportFilePath)
FileManager.default.createFile(
  atPath: exportFilePath,
  contents: Data(),
  attributes: nil
)
Going through the CSV export code step-by-step:
```

首先，通过执行提取请求检索所有 JournalEntry Entity。

1. 获取请求与获取结果控制器使用的请求相同。因此，你重用 surfJournalFetchRequest 方法来创建请求以避免重复。
2. 接下来，通过将文件名 export.csv 附加到 NSTemporaryDirectory 方法的输出来为导出的 CSV 文件创建 URL。

NSTemporaryDirectory 返回的路径是临时文件存储的唯一目录。 这是存放可以轻松再次生成且不需要通过 iTunes 或 iCloud 备份的文件的好地方。

创建导出 URL 后，调用 createFile(atPath:contents:attributes:) 创建一个空文件，你将在其中存储导出的数据。 如果一个文件已经存在于指定的文件路径，这个方法将首先删除它。

一旦应用程序有了空文件，它就可以将 CSV 数据写入磁盘：

```swift
// 3
let fileHandle: FileHandle?
do {
  fileHandle = try FileHandle(forWritingTo: exportFileURL)
} catch let error as NSError {
  print("ERROR: \(error.localizedDescription)")
  fileHandle = nil
}

if let fileHandle = fileHandle {
  // 4
  for journalEntry in results {
    fileHandle.seekToEndOfFile()
    guard let csvData = journalEntry
      .csv()
      .data(using: .utf8, allowLossyConversion: false) else {
        continue
    }

    fileHandle.write(csvData)

  }

  // 5
  fileHandle.closeFile()

  print("Export Path: \(exportFilePath)")
  self.navigationItem.leftBarButtonItem =
    self.exportBarButtonItem()
  self.showExportFinishedAlertView(exportFilePath)

} else {
  self.navigationItem.leftBarButtonItem =
    self.exportBarButtonItem()
}
```

以下是文件处理的工作原理：

3. 首先，应用程序需要创建一个用于写入的文件处理程序，它只是一个处理写入数据所需的低级磁盘操作的对象。 要创建用于写入的文件处理程序，请使用 FileHandle(forWritingTo:) 初始化程序。
4. 接下来，遍历所有 JournalEntry Entity。在每次迭代期间，你尝试在 JournalEntry 上使用 csv() 并在 String 上使用 data(using:allowLossyConversion:) 创建一个 UTF8 编码的字符串。如果成功，则使用文件处理程序 write() 方法将 UTF8 字符串写入磁盘。
5. 最后，关闭导出文件写入文件处理程序，因为它不再需要了。

一旦应用程序将所有数据写入磁盘，它就会显示一个带有导出文件路径的警告对话框。

![image-20230313014932786](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230313014932786.png)

> 注意：这个带有导出路径的警报控制器非常适合学习目的，但对于真正的应用程序，你需要为用户提供一种检索导出的 CSV 文件的方法，例如使用 UIActivityViewController。

要打开导出的 CSV 文件，请使用 Excel、Numbers 或你最喜欢的文本编辑器导航到并打开在警报对话框中指定的文件。 如果你在 Numbers 中打开文件，你将看到以下内容：

![image-20230313014948169](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230313014948169.png)

现在你已经了解了该应用程序当前如何导出数据，是时候进行一些改进了。

#### 在后台导出

你希望 UI 在导出过程中继续工作。 要修复 UI 问题，你将在私有后台上下文而不是主上下文中执行导出操作。

打开 JournalListViewController.swift，在 exportCSVFile() 中找到如下代码：

```swift
// 1
let context = coreDataStack.mainContext
var results: [JournalEntry] = []
do {
  results = try context.fetch(self.surfJournalFetchRequest())
} catch let error as NSError {
  print("ERROR: \(error.localizedDescription)")
}
```

正如你之前看到的，此代码通过在托管对象上下文中调用 fetch() 来检索所有日志条目。

接下来，将上面的代码替换为以下代码：

```swift
// 1
coreDataStack.storeContainer.performBackgroundTask { context in
  var results: [JournalEntry] = []
  do {
    results = try context.fetch(self.surfJournalFetchRequest())
  } catch let error as NSError {
    print("ERROR: \(error.localizedDescription)")
  }
```

你现在不使用 UI 也使用的 main ManagedObjectContext，而是在 Stack 的 PersistentStoreContainer上调用 performBackgroundTask(\_:) 。 这将创建一个新的 ManagedObjectContext 将其传递到闭包中。

performBackgroundTask(\_:) 创建的 context 位于私有队列中，不会阻塞主 UI 队列。 闭包中的代码在该私有队列上运行。

你还可以手动创建一个新的临时私有上下文，其并发类型为 .privateQueueConcurrencyType，而不是使用 performBackgroundTask(\_:)。

> 注意：托管对象上下文可以使用两种并发类型：
>
> Private Queue 指定将与专用调度队列而不是 Main Queue 相关联的 context。 这是你刚刚用于将导出操作从主队列中移出的队列类型，因此它不会再干扰 UI。
>
> Main Queue，默认类型，指定上下文将与主队列相关联。 这种类型是主要上下文 (coreDataStack.mainContext) 使用的类型。 任何 UI 操作，例如为表视图创建获取的结果控制器，都必须使用这种类型的上下文。

上下文及其托管对象只能从正确的队列中访问。 NSManagedObjectContext 具有 perform(_:) 和 performAndWait(_:) 以将工作定向到正确的队列。 你可以将启动参数 -com.apple.CoreData.ConcurrencyDebug 1 添加到应用程序的方案中，以捕获调试器中的错误。

接下来，在同一个方法中找到如下代码：

```swift
  print("Export Path: \(exportFilePath)")
  self.navigationItem.leftBarButtonItem =
    self.exportBarButtonItem()
  self.showExportFinishedAlertView(exportFilePath)
} else {
  self.navigationItem.leftBarButtonItem =
    self.exportBarButtonItem()
}
```

替换为

```swift
  print("Export Path: \(exportFilePath)")
  self.navigationItem.leftBarButtonItem =
    self.exportBarButtonItem()
  self.showExportFinishedAlertView(exportFilePath)
} else {
  self.navigationItem.leftBarButtonItem =
    self.exportBarButtonItem()
}
Replace the code with the following:
    print("Export Path: \(exportFilePath)")
    // 6
    DispatchQueue.main.async {
      self.navigationItem.leftBarButtonItem =
        self.exportBarButtonItem()
      self.showExportFinishedAlertView(exportFilePath)
    }
  } else {
    DispatchQueue.main.async {
      self.navigationItem.leftBarButtonItem =
        self.exportBarButtonItem()
    }
  }
} // 7 Closing brace for performBackgroundTask
```

完成任务：

6. 你应该始终在主队列上执行与 UI 相关的所有操作，例如在导出操作完成时显示警报视图； 否则，可能会发生不可预测的事情。 使用 DispatchQueue.main.async 显示主队列上的最终警报视图消息。
7. 最后，添加一个右花括号以关闭你之前在步骤 1 中通过 performBackgroundTask(\_:) 调用打开的块。

现在你已经将导出操作移动到具有专用队列的新上下文，构建并运行以查看它是否有效！

你应该准确地看到你之前看到的内容：

![image-20230313015826983](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230313015826983.png)

点击左上角的“导出”按钮，然后立即尝试滚动浏览会话日志条目列表。 这次有什么不同吗？ 导出操作仍需要几秒钟才能完成，但表视图在此期间继续滚动。 导出操作不再阻塞 UI。

Cowabunga，伙计！ 粗糙的工作使 UI 更具响应性。

你刚刚见证了在私有后台队列上工作如何改善用户对你的应用程序的体验。 现在，你将通过检查 child context 来扩展对多个 context 的使用。

### 在暂存器上编辑

现在，SurfJournal 在创建新日记条目或查看现有日记条目时使用主上下文 (coreDataStack.mainContext)。 这种方法没有错； 入门项目按原样工作。

对于像这样的日志式应用程序，你可以通过将编辑或新条目视为一组更改来简化应用程序架构，就像便签本一样。 当用户编辑日记条目时，你更新了被管理对象的属性。

更改完成后，你可以保存它们或丢弃它们，具体取决于用户想要做什么。

你可以将 child context 视为临时便笺本，你可以完全丢弃它，或者保存更改并将更改发送到父上下文。

但从技术上讲，什么是 child context？

所有 ManagedObjectContext 都有一个父存储，你可以从中检索和更改托管对象形式的数据，例如本项目中的 JournalEntry 对象。 通常，父存储是 PersistentStoreCoordinator，CoreDataStack 类提供的 main Context 就是这种情况。 或者，你可以将给定 context 的父存储设置为另一个 ManagedObjectContext，使其成为 child context。

![image-20230313020234315](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230313020234315.png)

当你保存 child context 时，更改只会转到 parent context。 在保存 parent context 之前，不会将对 parent context 的更改发送到 PersistentStoreCoordinator。

在你开始添加 child context 之前，你需要了解当前的查看和编辑操作是如何工作的。

#### 查看和编辑

操作的第一部分需要从主列表视图转到日志详细信息视图。

打开 JournalListViewController.swift 并找到 prepare(for:sender:)：

```swift
// 1
if segue.identifier == "SegueListToDetail" {
  // 2
  guard let navigationController =
    segue.destination as? UINavigationController,
    let detailViewController =
      navigationController.topViewController
        as? JournalEntryViewController,
    let indexPath = tableView.indexPathForSelectedRow else {
      fatalError("Application storyboard mis-configuration")
  }
  // 3
  let surfJournalEntry =
    fetchedResultsController.object(at: indexPath)
  // 4
  detailViewController.journalEntry = surfJournalEntry
  detailViewController.context =
    surfJournalEntry.managedObjectContext
  detailViewController.delegate = self
```

逐步执行 segue 代码：

1. 有两个 segue：SegueListToDetail 和 SegueListToDetailAdd。 第一个显示在前面的代码块中，当用户点击表视图中的一行以查看或编辑以前的日记条目时运行。
2. 接下来，你将获得对用户将最终看到的 JournalEntryViewController 的引用。 它显示在导航控制器中，因此需要进行一些拆包。 此代码还验证表视图中是否存在选定的索引路径。
3. 接下来，使用获取结果控制器的 object(at:) 方法获取用户选择的 JournalEntry。
4. 最后，你在 JournalEntryViewController 实例上设置所有必需的变量。 surfJournalEntry 变量对应于步骤 3 中解析的 JournalEntry Entity。上下文变量是用于任何操作的托管对象上下文； 目前，它只使用主上下文。 JournalListViewController 将自己设置为 JournalEntryViewController 的委托，以便在用户完成编辑操作时通知它。

SegueListToDetailAdd 类似于 SegueListToDetail，不同之处在于应用程序创建一个新的 JournalEntry Entity 而不是检索现有 Entity。

当用户点击右上角的加号 (+) 按钮创建新日记条目时，该应用会执行 SegueListToDetailAdd。

现在你知道这两个 segues 是如何工作的，打开 JournalEntryViewController.swift 并查看文件顶部的 JournalEntryDelegate 协议：

```swift
protocol JournalEntryDelegate: AnyObject {
  func didFinish(
    viewController: JournalEntryViewController,
    didSave: Bool
  )
}
```

JournalEntryDelegate 协议非常短，只包含一个方法：didFinish(viewController:didSave:)。 协议要求委托人实施的此方法指示用户是否已完成编辑或查看日记条目以及是否应保存任何更改。

要了解 didFinish(viewController:didSave:) 是如何工作的，请切换回 JournalListViewController.swift 并找到该方法：

```swift
func didFinish(
  viewController: JournalEntryViewController,
  didSave: Bool
) {
  // 1
  guard didSave,
    let context = viewController.context,
    context.hasChanges else {
      dismiss(animated: true)
      return
  }
  // 2
  context.perform {
    do {
      try context.save()
    } catch let error as NSError {
      fatalError("Error: \(error.localizedDescription)")
    }
    // 3
    self.coreDataStack.saveContext()
  }
  // 4
  dismiss(animated: true)
}
```

依次获取每个编号的评论：

1. 首先，使用 guard 语句检查 didSave 参数。 如果用户点击保存按钮而不是取消按钮，这将是正确的，因此应用程序应该保存用户的数据。 guard 语句还使用 hasChanges 属性来检查是否有任何更改； 如果什么都没有改变，就没有必要浪费时间做更多的工作。
2. 接下来，将 JournalEntryViewController 上下文保存在 perform(\_:) 闭包中。 该代码将此context 设置为 main Context； 在这种情况下，它有点多余，因为只有一个 Context，但这不会改变行为。稍后将 child context 添加到工作流后，JournalEntryViewController context 将不同于 main Context，因此需要此代码。 如果保存失败，则调用 fatalError 中止应用程序并提供相关错误信息。
3. 接下来，通过在 CoreDataStack.swift 中定义的 saveContext 保存 mainContext，将任何编辑保存到磁盘。
4. 最后，关闭 JournalEntryViewController。

> 注意：如果托管对象上下文是 MainQueueConcurrencyType 类型，则不必将代码包装在 perform(\_:) 中，但使用它也无妨。

如果你不知道 context 是什么类型，就像 didFinish(viewController:didSave:) 中的情况一样，使用 perform(\_:) 是最安全的，因为它可以同时用于 parent context 和 child context。

上面的实现有一个问题——你发现了吗？

当应用程序添加新的日记条目时，它会创建一个新对象并将其添加到 ManagedObjectContext 中。 如果用户点击取消按钮，应用程序不会保存 context，但新对象仍然存在。 如果用户随后添加并保存另一个条目，取消的对象将仍然存在！ 除非你有耐心一直滚动到最后，否则你不会在 UI 中看到它，但它会显示在 CSV 导出的底部。

你可以通过在用户取消视图控制器时删除对象来解决此问题。 但是，如果更改很复杂，涉及多个对象，或者需要你在编辑工作流程中更改对象的属性怎么办？ 使用子 chind context 将帮助你轻松处理这些复杂情况。

#### 使用 chind context 进行编辑

现在你已经知道应用程序当前如何编辑和创建 JournalEntry Entity ，你将修改实现以使用 child context 作为临时草稿本。

这很容易做到——你只需要修改 segues。 打开 JournalListViewController.swift 并在 prepare(for:sender:) 中找到以下 SegueListToDetail 代码：

```swift
detailViewController.journalEntry = surfJournalEntry
detailViewController.context =
  surfJournalEntry.managedObjectContext
detailViewController.delegate = self
```

接下来，将该代码替换为以下内容：

```swift
// 1
let childContext = NSManagedObjectContext(
  concurrencyType: .mainQueueConcurrencyType)
childContext.parent = coreDataStack.mainContext

// 2
let childEntry = childContext.object(
  with: surfJournalEntry.objectID) as? JournalEntry

// 3
detailViewController.journalEntry = childEntry
detailViewController.context = childContext
detailViewController.delegate = self
```

这是逐个播放：

1. 首先，你使用 .mainQueueConcurrencyType 创建一个名为 childContext 的新 ManagedObjectContext。 在这里，你设置 parent Context 而不是 PersistentStoreCoordinator，就像你在创建 ManagedObjectContext 时通常所做的那样。 在这里，你将 parent 设置为 CoreDataStack 的 mainContext。
2. 接下来，你使用子上下文的 object(with:) 方法检索相关的日记条目。 你必须使用 object(with:) 来检索日记条目，因为 ManagedObject 特定于创建它们的 context。 但是，objectID 值并不特定于单个 context，因此你可以在需要访问多个 context 中的对象时使用它们。
3. 最后，你在 JournalEntryViewController 实例上设置所有必需的变量。 这一次，你使用 childEntry 和 childContext 而不是原来的 surfJournalEntry 和 surfJournalEntry.managedObjectContext。

> 注意：你可能想知道为什么需要将 ManagedObject 和 ManagedObjectContext 都传递给 detailViewController，因为 ManagedObject 已经有一个 context 变量。 这是因为托管对象对 context 只有弱引用。 如果你不传递上下文，ARC 将从内存中删除 context（因为没有其他东西保留它）并且应用程序将不会像你期望的那样运行。

构建并运行你的应用程序； 它应该像以前一样工作。 在这种情况下，应用程序没有明显的变化是件好事； 用户仍然可以点击一行来查看和编辑冲浪会话日记条目。

![image-20230313021512742](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230313021512742.png)

通过使用 child context 作为日志编辑的容器，你已经降低了应用架构的复杂性。 通过在单独的 context 中进行编辑，取消或保存 ManagedObject 更改是微不足道的。

干得好，伙计！ 当涉及到多个 ManagedObjectContext 时，你不再是个怪人。 大胆！

### 关键点

* ManagedObjectContext 是用于处理托管对象的内存暂存器。
* Private background context 可用于防止阻塞主 UI。
* context 与特定队列相关联，只能在这些队列上访问。
* child context 可以通过使保存或丢弃编辑变得容易来简化应用程序的架构。
* ManagedObject 与其 context 紧密绑定，不能与其他 context 一起使用。

### 挑战

使用你的新知识，尝试更新 SegueListToDetailAdd 以在添加新日记条目时使用子上下文。

就像以前一样，你需要创建一个以主上下文为父上下文的子上下文。 你还需要记住在正确的上下文中创建新条目。

如果你遇到困难，请查看本章文件夹中包含挑战解决方案的项目——但请先尽力而为！
