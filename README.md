# 第 1 节：你的第一个 Core Data App

欢迎来到 Core Data！

在本节中，你将编写你的第一个 Core Data 应用程序。你将看到开始使用 Xcode 中提供的所有资源是多么容易，从入门代码模板到 Data Model 编辑器。

到本节结束时，你将知道如何：

* 使用 Xcode 的 Model Editor 建模数据
* 向 Core Data 添加新记录
* 从 Core Data 中获取一组记录
* 使用 TableView 显示获取的记录

你还将了解 Core Data 在幕后做了什么，以及你如何与各种移动的部分进行交互。这将使你很好地理解接下来的两节，他们会继续介绍 Core Data 以及更高级的模型和数据验证。

### 入门

打开 Xcode 并根据 App 模板创建一个新的 iOS 项目。将应用程序命名为 HitList，并将界面从 SwiftUI 更改为 Storyboard。本书中的所有示例项目都使用了 Interface Builder 和 UIKit。最后选中使用 Core Data，但未选中 Host in CloudKit 和 Include Tests。

选中 Use Core Data， Xcode 为 AppDelegate.swift 中的 NSPersistentContainer 生成模版代码。

NSPersistentContainer 由一组对象组成，这些对象有助于从 Core Data 中保存和检索信息。在这个容器中是一个管理 Core Data 状态的对象，一个表示 Data Model 的对象，等等。

你将在前几节中了解其中的每一部分。稍后，你有机会编写自己的 Core Data Stack！标准 Stack 适用于大多数应用程序，但根据你的应用程序及其数据要求，你可以自定义 Stack 以提高效率。

这个示例应用程序的想法很简单：将有一个 TableView，其中包含你自己的“hit list”的列表。你将能够向此列表添加 name，并最终使用 Core Data 确保数据在 session 之间存储。

单击 Main.storyboard 在 Interface Builder 中打开它。在画布上选择 view controller 并将其嵌入到 navigation controller 中。从 Xcode 的编辑器菜单中，选择 Embed In… ▸ Navigation Controller。

![image-20221113213514900](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113213514900.png)

接下来，将 Table View 拖到 ViewController 中，调整它的大小，使其覆盖整个视图。

按住 Ctrl 键从 document outline 中的 Table View 拖动到其父视图，然后选择 Leading Space to Safe Area 约束：

![image-20221113213349114](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113213349114.png)

再重复三次，选择约束 Trailing Space to Safe Area、Top Space to Safe Area，最后选择 Bottom Space to Safe Area。 添加这四个约束将使 Table View 填充其父视图。

> 注意：确保 Interface Builder 使用 0 点的常量创建约束，这意味着与屏幕的每一侧齐平。 如果不是这种情况，你可以打开右侧的 Size Inspector 并更正间距。

接下来，拖动一个 Bar Button Item 并将其放置在 view controller 的 navigation bar 上。 最后，选择栏按钮项并将其 system item 更改为 Add。

你的画布应类似于以下屏幕截图：

![image-20221113213305659](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113213305659.png)

每次点击 Add button 时，都会出现一个包含文本字段的 AlertController。在那里，你将能够在文本字段中输入某人的姓名。点击 Save 将保存名称，关闭 AlertController 并刷新 TableView，显示你输入的所有名称。

但首先，你需要使 ViewController 成为 TableView 的 DataSource。在画布中，从 TableView 中按住 Ctrl 键拖动到导航栏上方的黄色 ViewController 器图标，如下图，然后点击dataSource：

![image-20221113213432557](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113213432557.png)

你不需要设置表视图的委托，因为点击单元格不会触发任何操作。 没有比这更简单的了！

通过按下 Control-Command-Option-Enter 或选择 Storyboard scene 右上角的调整编辑器按钮并选择 Assistant 来打开 assistant editor，如下所示。

![image-20221113213629466](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113213629466.png)

按住 Ctrl 从表视图拖动到类定义内的 `ViewController.swift` 上，以创建一个 IBOutlet。

接下来，将新的 IBOutlet 属性命名为 tableView，产生以下行：

```swift
@IBOutlet weak var tableView: UITableView!
```

接下来，按住 Ctrl 键从添加按钮拖动到 ViewController.swift 中，就在你的 viewDidLoad() 定义下方。 这一次，创建一个 Action 而不是 Outlet，将方法命名为 addName，类型为 UIBarButtonItem：

```swift
@IBAction func addName(_ sender: UIBarButtonItem) {

}
```

你现在可以在代码中引用表视图和栏按钮项的操作。

接下来，你将为 RableView 设置 model。 将以下属性添加到 TableView IBOutlet 下方的 ViewController.swift 中：

```swift
var names: [String] = []
```

names 是一个可变数组，其中包含 table view 显示的字符串值。 接下来，将 viewDidLoad() 的实现替换为以下内容：

```swift
override func viewDidLoad() {
  super.viewDidLoad()

  title = "The List"
  tableView.register(UITableViewCell.self,
                     forCellReuseIdentifier: "Cell")
}
```

这将在 navigation bar 上设置一个标题，并将 UITableViewCell 类注册到表视图。

> 注意：register(\_:forCellReuseIdentifier:) 保证在将 Cell reuseIdentifier 提供给 dequeue 方法时，你的表视图将返回正确类型的单元格。

接下来，仍然在 ViewController.swift 中，在 ViewController 的类定义下方添加以下 UITableViewDataSource 扩展：

```swift
// MARK: - UITableViewDataSource
extension ViewController: UITableViewDataSource {

  func tableView(_ tableView: UITableView,
                 numberOfRowsInSection section: Int) -> Int {
    return names.count
  }

  func tableView(_ tableView: UITableView,
                 cellForRowAt indexPath: IndexPath)
                 -> UITableViewCell {
    let cell =
      tableView.dequeueReusableCell(withIdentifier: "Cell",
                                    for: indexPath)
    cell.textLabel?.text = names[indexPath.row]
    return cell

  }
}
```

如果你曾经使用过 UITableView，这段代码应该看起来很熟悉。 首先，你将表中的行数作为名称数组中的项目数返回。

接下来，tableView(\_:cellForRowAt:) 将表视图单元格出队，并用名称数组中的相应字符串填充它们。

接下来，你需要一种添加新 name 的方法，以便表视图可以显示它们。 实现你之前按 Ctrl 键拖动到代码中的 addName IBAction 方法：

```swift
// Implement the addName IBAction
@IBAction func addName(_ sender: UIBarButtonItem) {

  let alert = UIAlertController(title: "New Name",
                                message: "Add a new name",
                                preferredStyle: .alert)

  let saveAction = UIAlertAction(title: "Save",
                                 style: .default) {
    [unowned self] action in
    
    guard let textField = alert.textFields?.first,
      let nameToSave = textField.text else {
        return
    }
    
    self.names.append(nameToSave)
    self.tableView.reloadData()

  }

  let cancelAction = UIAlertAction(title: "Cancel",
                                   style: .cancel)

  alert.addTextField()

  alert.addAction(saveAction)
  alert.addAction(cancelAction)

  present(alert, animated: true)
}
```

每次点击 Add 按钮时，此方法都会显示一个带有文本字段和两个按钮的 UIAlertController：Save 和 Cancel。

Save 将文本字段当前文本插入到名称数组中，然后重新加载 TableView。由于 name 数组是 TableView 的 model，因此你在文本字段中输入的任何内容都会出现在表格视图中。

最后，首次构建并运行你的应用程序。接下来，点击添加按钮。 alert controller 将如下所示：

![image-20221113212534515](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113212534515.png)

将四五个 name 添加到列表中。你应该会看到类似于下面的内容：

![image-20221113212615021](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113212615021.png)

你的 TableView 将显示数据，而你的数组将存储 name，但这里缺少的重要内容是持久化。该数组在内存中，但如果你强制退出应用程序或重新启动设备，你的列表将被清除。 Core Data 提供持久化，这意味着它可以以更持久的状态存储数据，因此它可以比应用程序重新启动或设备重新启动更长寿。

你还没有添加任何 Core Data 元素，因此在你离开应用程序后应该不会保留任何内容。 让我们测试一下。 如果你使用的是物理设备，请按 Home 按钮；如果你使用的是模拟器，请按 Shift + ⌘ + H。这将带你回到主屏幕上。

在主屏幕上，点击 HitList 图标以将应用程序带回前台。名字还在屏幕上。发生了什么？

当你点击主页按钮时，当前位于前台的应用程序会转到后台。操作系统会闪存冻结当前内存中的所有内容，包括 names 数组中的字符串。同样，当需要醒来并返回前台时，操作系统会恢复内存中曾经存在的内容，就好像你从未离开过一样。

Apple 在 iOS 4 中引入了多任务处理方面的这些进步。它们为 iOS 用户创造了无缝体验，但增加了 iOS 开发人员对持久性的定义。names 真的保留了吗？

不，不是真的。如果你在快速应用程序切换器中完全杀死了该应用程序或关闭了手机，那么这些名称就会消失。你也可以验证这一点。将应用程序置于前台，进入快速应用程序切换器。

如果你的设备有主屏幕按钮，你可以双击主屏幕按钮；如果你使用的是 iPhone X 或更高版本，你可以从屏幕底部慢慢向上拖动。

从这里，向上轻弹 HitList 应用程序快照以终止应用程序。从应用程序切换器中删除应用程序后，内存中没有了 HitList 的痕迹。通过返回主屏幕并点击 HitList 图标以触发新的启动，验证名称已消失。

如果你已经使用 iOS 一段时间并且熟悉多任务处理的工作方式，那么快速冻结和持久性之间的区别可能会很明显。但是，在用户看来，没有区别。用户不关心为什么名称仍然存在，无论应用程序进入后台并返回，还是因为应用程序保存并重新加载它们。重要的是当应用程序返回时名称仍然存在！

因此，持久化的真正考验是在新的应用程序启动后你的数据是否仍然存在。

### 构造 Data

现在你知道了如何检查持久性，你可以深入研究 Core Data。 HitList 应用程序的目标很简单：保留你输入的 name，以便在新的应用程序启动后可以查看它们。

到目前为止，你一直在使用普通的旧 Swift 字符串将 names 存储在内存中。在本节中，你将用 Core Data 对象替换这些字符串。第一步是创建一个 managed object model，它描述了 Core Data 在磁盘上表示数据的方式。

默认情况下，Core Data 使用 SQLite 数据库作为持久存储，因此你可以将 Data Model 视为 database schema。

> 注意：你会在本书中多次遇到 managed 这个词。如果你在一个类的名字中看到“managed”，比如在 NSManagedObjectContext 中，你很可能正在处理一个 Core Data 类。 “managed”是指 Core Data 对 Core Data object 生命周期的管理。
>
> 但是，不要假设所有 Core Data 类都包含“managed”一词。大多数没有。有关 Core Data 类的完整列表，请查看 Apple 文档中的 Core Data framework。

由于你已选择使用 Core Data，Xcode 会自动为你创建一个 Data Model 文件并将其命名为 HitList.xcdatamodeld。

打开 HitList.xcdatamodeld。如你所见，Xcode 有一个强大的 Data Model editor：

![image-20221113212739793](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113212739793.png)

Data Model editor 有很多特性，你将在后面的章节中探索。现在，让我们专注于创建单个 Core Data entity。

单击左下角的添加实体以创建新实体。双击新实体并将其名称更改为 Person，如下所示：

![image-20221113220341799](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113220341799.png)

你可能想知道为什么 model editor 使用术语 Entity。你不是简单地定义了一个新类吗？正如你很快就会看到的，Core Data 有自己的词汇表。以下是你经常遇到的一些术语的简要说明：

* Entity 是 Core Data 中的类定义。典型的例子是 Employee 或 Company。在关系数据库中，一个 Entity 对应一个表。
* Attribute 是附加到特定 Entity 的一条信息。例如，Employee 实体可以具有 name、position 和 salary 的属性。在数据库中，属性对应于表中的特定字段。
* Relationship 是多个 Entity 之间的链接。在 Core Data 中，两个 Entity 之间的关系称为一对一关系，而一个和多个 Entity 之间的关系称为一对多关系。

现在你知道 Attribute 是什么了，你可以给之前创建的 Person 对象添加一个属性。还是在 HitList.xcdatamodeld 中，选择左侧的 Person 并单击 Attributes 下的加号 (+)。

将新属性的名称设置为 name 并将其类型更改为 String：

![image-20221113212208722](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113212208722.png)

在 Core Data 中，Attribute 可以是多种数据类型之一。你将在接下来的几节中了解这些内容。

### save 到 Core Data

打开 ViewController.swift，在 import UIKit 下添加以下 Core Data 模块导入：

```swift
import CoreData
```

接下来，将 names 属性定义替换为以下内容：

```swift
var people: [NSManagedObject] = []
```

你将存储 Person Entity 而不是 String names，因此将 TableView 的 Data Model 的数组重命名为 people。它现在包含 NSManagedObject 的实例而不是简单的字符串。

NSManagedObject 表示存储在 Core Data 中的单个对象；你必须使用它在 Core Data Persistent Store 中创建、编辑、保存和删除。你很快就会看到，NSManagedObject 是一个“变形器(shape-shifter)”。它可以是 Data Model Entity 的形式，用做你定义的任何属性和关系。

由于你正在更改表格视图的模型，因此你还必须替换之前实现的两个数据源方法。将你的 UITableViewDataSource 扩展替换为以下内容：

```swift
// MARK: - UITableViewDataSource
extension ViewController: UITableViewDataSource {
  func tableView(_ tableView: UITableView,
                 numberOfRowsInSection section: Int) -> Int {
    return people.count
  }

  func tableView(_ tableView: UITableView,
                 cellForRowAt indexPath: IndexPath)
                 -> UITableViewCell {

    let person = people[indexPath.row]
    let cell =
      tableView.dequeueReusableCell(withIdentifier: "Cell",
                                    for: indexPath)
    cell.textLabel?.text =
      person.value(forKeyPath: "name") as? String
    return cell

  }
}
```

这些方法最显着的变化发生在 tableView(\_:cellForRowAt:) 中。现在，你不再将单元格与模型数组中的相应字符串匹配，而是将单元格与相应的 NSManagedObject 匹配。请注意如何从 NSManagedObject 中获取名称属性。它发生在这里：

```swift
cell.textLabel?.text =
  person.value(forKeyPath: "name") as? String
```

为什么你必须这样做？事实证明，NSManagedObject 不知道你在 Data Model 中定义的name 属性，因此无法通过属性直接访问它。 Core Data 提供读取值的唯一方法是 key-value coding(KVC)。

> 注意：KVC 是 Foundation 中使用字符串间接访问对象属性的一种机制。在这种情况下，KVC 使 NSMangedObject 在运行时表现得有点像字典。
>
> KVC 可用于从 NSObject 继承的所有类，包括 NSManagedObject。你不能使用 KVC 访问不是从 NSObject 继承的 Swift 对象上的属性。

接下来，找到 addName(\_:) 并将 save UIAlertAction 替换为以下内容：

```swift
let saveAction = UIAlertAction(title: "Save", style: .default) {
  [unowned self] action in

  guard let textField = alert.textFields?.first,
    let nameToSave = textField.text else {
      return
  }

  self.save(name: nameToSave)
  self.tableView.reloadData()
}
```

这将获取文本字段中的文本并将其传递给名为 save(name:) 的新方法。save(name:) 还不存在。在 addName(\_:) 下添加以下实现：

```swift
func save(name: String) {

  guard let appDelegate =
    UIApplication.shared.delegate as? AppDelegate else {
    return
  }

  // 1
  let managedContext =
    appDelegate.persistentContainer.viewContext

  // 2
  let entity =
    NSEntityDescription.entity(forEntityName: "Person",
                               in: managedContext)!

  let person = NSManagedObject(entity: entity,
                               insertInto: managedContext)

  // 3
  person.setValue(name, forKeyPath: "name")

  // 4
  do {
    try managedContext.save()
    people.append(person)
  } catch let error as NSError {
    print("Could not save. \(error), \(error.userInfo)")
  }
}
```

这就是 Core Data 发挥作用的地方！下面是代码的作用：

1.  在你可以从你的 Core Data Store 中保存或检索任何内容之前，你首先需要获得一个 NSManagedObjectContext。 你可以将 ManagedObjectContext 视为用于处理 managed objects 的内存中“便签本”。

    将一个新的 managed object 保存到 Core Data 中作为一个两步过程：首先，将一个新的 managed object 插入到 ManagedObjectContext 中；接着你就可以“commit” ManagedObjectContext 的更改以将其保存到磁盘。

    Xcode 已经生成了一个 ManagedObjectContext 作为新项目模板的一部分。请记住，只有在开始时选中 Use Core Data 复选框时才会发生这种情况。此默认 ManagedObjectContext 为 NSPersistentContainer 的属性，存在于 application delegate 中。要访问它，你首先要获得对 application delegate 引用。
2.  你创建一个新的 managed object 并将其插入到 ManagedObjectContext。你可以使用 NSManagedObject 的静态方法一步完成：entity(forEntityName:in:)。

    你可能想知道 NSEntityDescription 到底是什么。回想一下，NSManagedObject 被称为 shape-shifter 类，因为它可以表示任何实体。Entity description 将 Data Model 中的 Entity 定义与运行时的 NSManagedObject 实例联系起来。
3. 有了 NSManagedObject，你可以使用 KVO 设置名称属性。你必须准确拼写 KVC key，否则你的应用程序将在运行时崩溃。
4. 你可以通过在 ManagedObjectContext 上调用 save 来将更改提交给 person 并保存到磁盘。注意 save 可能会引发错误，这就是为什么你在 do-catch 块中使用 try 关键字调用它的原因。最后，将新的 managed object 插入到 people 数组中，以便在重新加载 table view 时显示出来。

这比使用字符串数组要复杂一些，但也不算太糟。这里的一些代码，例如 managed object context 和 entity，可以在你自己的 init() 或 viewDidLoad() 中只完成一次，然后再重用。为简单起见，你使用相同的方法完成所有操作。

构建并运行应用程序，并在 table view 中添加一些 name：

![image-20221113220427070](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113220427070.png)

如果 name 实际上存储在 Core Data 中，则 HitList 应用程序应该通过持久性测试。将应用程序置于前台，转到快速应用程序切换器，然后终止它。点击 HitList 应用程序以触发新启动。 等等，表视图是空的：

![image-20221113220621108](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113220621108.png)

你已保存到 Core Data，但在应用启动后，people 数组为空！ 那是因为数据在磁盘上等着你，但你还没有显示它。

### 从 Core Data 中 fetch

要从持久存储中获取数据到托管对象上下文中，你必须获取它。打开 ViewController.swift 并在 viewDidLoad() 下面添加以下内容：

```swift
override func viewWillAppear(_ animated: Bool) {
  super.viewWillAppear(animated)

  //1
  guard let appDelegate =
    UIApplication.shared.delegate as? AppDelegate else {
      return
  }

  let managedContext =
    appDelegate.persistentContainer.viewContext

  //2
  let fetchRequest =
    NSFetchRequest<NSManagedObject>(entityName: "Person")

  //3
  do {
    people = try managedContext.fetch(fetchRequest)
  } catch let error as NSError {
    print("Could not fetch. \(error), \(error.userInfo)")
  }
}
```

这就是代码的作用：

1. 在你可以使用 Core Data 做任何事情之前，你需要一个 ManagedObjectContext。像以前一样，你获取 application delegate 并获取对其 persistentContainer 的引用以获取其 ManagedObjectContext。
2.  顾名思义，NSFetchRequest 是负责从 Core Data 中获取数据的类。获取请求既强大又灵活。你可以使用提取请求来提取一组满足所提供标准的对象。

    NSFetchRequest 有几个限定符用于优化返回的结果集。你将后文了解有关这些限定符的更多信息；现在，你应该知道 NSEntityDescription 是这些必需的限定符之一。

    设置 fetchRequest 的 entity 属性，或者使用 init(entityName:) 对其进行初始化，获取特定实体的所有对象。这就是你在此处获取所有 Person entity 所执行的操作。另请注意 NSFetchRequest 是一种通用类型。这种泛型的使用指定了获取请求的预期返回类型，在本例中为 NSManagedObject。
3. 你将提取请求交给 ManagedObjectContext 来完成繁重的工作。 fetch(\_:) 返回满足获取请求指定条件的托管对象数组。

> 注意：与 save() 一样，fetch(\_:) 也可能抛出错误，因此你必须在 do 块中使用它。如果在获取过程中发生错误，你可以在 catch 块中检查错误并做出适当的响应。

构建并运行应用程序。你应该立即看到你之前添加的名称列表：

![image-20221113221734118](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221113221734118.png)

向列表中添加更多名称并重新启动应用程序以验证保存和提取是否正常。只要不删除应用程序、重置模拟器或将手机从高楼上扔下去，无论如何，名字都会出现在表格视图中。

> 注意：此示例应用程序中有一些粗糙的边缘：你每次都必须从应用程序委托中获取托管对象上下文，并且你使用 KVC 来访问实体的属性，而不是更自然的对象样式的 person.name。

有更好的方法来保存和从 Core Data 中获取数据，你将在以后的章节中探讨这些方法。

### 关键点

* Core Data 提供磁盘持久化，这意味着即使在终止你的应用程序或关闭你的设备后，你的数据仍可访问。这与内存中持久性不同，后者只会在你的应用程序在内存中时保存你的数据，无论是在前台还是在后台。
* Xcode 带有一个强大的 Data Model Editor，你可以使用它来创建你的 ManagedObjectModel。
* ManagedObjectModel 由 entities、 attributes 和 relationships 组成。
* Entity 是 Core Data 中的类定义。
* Attribute 是附加到 Entity 的一条信息。
* Relationship 是多个 Entity 之间的链接。

NSManagedObject 是 Core Data Entity 的运行时表示。你可以使用 KVC 读取和写入其属性。

* 你需要一个 NSManagedObjectContext 来 save() 或 fetch(\_:) Core Data 的数据。
