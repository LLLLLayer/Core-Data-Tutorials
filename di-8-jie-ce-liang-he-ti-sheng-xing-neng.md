# 第 8 节：测量和提升性能

在许多方面，这是一个显而易见的事情：你应该努力优化你开发的任何应用程序的性能。 性能不佳的应用充其量会收到差评，最坏的情况是变得无响应和崩溃。

使用 Core Data 的应用程序也是如此。 幸运的是，由于 Core Data 的内置优化，例如故障，Core Data 的大多数实现已经是快速和轻量级的。

然而，使 Core Data 成为出色工具的灵活性意味着你可以以对性能产生负面影响的方式使用它。 从设置数据模型的糟糕选择到低效的获取和搜索，Core Data 有很多机会减慢你的应用程序。

本章开始时，你会看到一个运行缓慢的内存消耗大的应用程序。 到本章结束时，你将拥有一个轻巧且快速的应用程序，如果你发现自己的应用程序笨重、缓慢，你将确切地知道该去哪里查看以及该怎么做——以及如何避免这种情况 第一名！

### 入门

与大多数事情一样，性能是内存和速度之间的平衡。 你的应用程序的 Core Data model objects 可以存在于两个地方：随机存取存储器（RAM）或磁盘上。

访问 RAM 中的数据比访问磁盘上的数据快得多，但设备的 RAM 比磁盘空间少得多。

![image-20230311212321980](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230311212321980.png)

特别是 iOS 设备，可用 RAM 较少，这会阻止你将大量数据加载到内存中。 由于 RAM 中的模型对象较少，你的应用程序的操作会由于频繁的缓慢磁盘访问而变慢。 当你将更多模型对象加载到 RAM 中时，你的应用程序可能会感觉响应更快，但你不能饿死其他应用程序，否则操作系统将终止你的应用程序！

### 入门项目

起始项目 EmployeeDirectory 是一个基于选项卡栏的应用程序，其中包含大量员工信息。 它就像联系人应用程序，但适用于一家虚构的公司。

在 Xcode 中打开本章的 EmployeeDirectory 起始项目，构建并运行它。

该应用程序将需要很长时间才能启动，一旦启动，它会感觉迟钝，甚至可能会在你使用时崩溃。 放心，这是设计使然！

> 注意：启动项目可能甚至无法在你的系统上启动。 该应用程序被设计为尽可能缓慢，同时仍然能够在大多数系统上运行，因此你所做的性能改进将很容易被注意到。 如果该应用程序拒绝在你的系统上运行，请继续执行。 你所做的第一组更改应该使该应用程序即使在最慢的设备上也能运行。

正如你在下面的屏幕截图中看到的，第一个选项卡包括一个表视图和一个自定义单元格，其中包含所有员工的基本信息，例如姓名和部门。

点击一个单元格以显示所选员工的更多详细信息，例如开始日期和剩余休假天数。

![image-20230311215504171](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230311215504171.png)

点击员工头像全屏显示； 点击全屏图片上的任意位置将其关闭。

该应用程序的启动时间相当长，初始员工列表的滚动性能可能需要一些工作。 该应用程序还使用大量内存，你将在下一节中自行测量。

### 测量、改变、验证

与其猜测你的性能瓶颈在哪里，你可以通过首先有针对性地测量你的应用程序的性能来节省你自己的时间和精力。 Xcode 专门为此目的提供了工具。

理想情况下，你应该测量性能，进行有针对性的更改，然后再次测量以验证你的更改是否产生了预期的影响。

你应该根据需要多次重复此测量-更改-验证过程，直到你的应用程序满足所有性能要求。

![image-20230311215558625](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230311215558625.png)

在本章中，你将这样做：

* 你将使用 Gauges、Instruments 和 XCTest 框架测量提供的入门项目中的性能问题。
* 接下来，你将更改代码以提高应用程序的性能。
* 最后，你将通过再次测量来验证更改是否达到了预期的结果。

然后你将重复这个循环，直到 EmployeeDirectory 表现得像 Core Data 冠军！

#### 测量问题

构建、运行并等待应用程序启动。 完成后，使用内存报告查看应用程序使用了多少 RAM。

要启动内存报告，首先验证应用程序正在运行，然后执行以下步骤：

1. 单击左侧导航器窗格中的调试导航器。
2. 要获取更多信息，请点击箭头展开正在运行的进程（在本例中为 EmployeeDirectory）。

![image-20230312024200937](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312024200937.png)

现在，单击“内存”行并查看内存量表的上半部分：

![image-20230312024141683](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312024141683.png)

上半部分包括一个内存使用量表，显示你的应用程序正在使用的内存量和百分比。 对于 EmployeeDirectory，你会看到 100 MB 到 400 MB 的内存正在使用中，或者 iPhone 6S 上可用 RAM 的大约 10%，这是仍在运行 iOS 14 的性能最低的设备。

Usage Comparison 饼图将此内存块描述为总可用内存的一小部分。 它还显示其他进程正在使用的 RAM 量，以及可用的空闲 RAM 量。

现在，查看内存报告的下半部分：

![image-20230312021451738](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312021451738.png)

下半部分包含一个图表，显示随时间变化的 RAM 使用情况。 对于 EmployeeDirectory，你会看到两个不同的区域。

首次启动时，EmployeeDirectory 在加载主要员工列表之前执行导入操作。 暂时忽略内存中的这些峰值。

下一个内存使用块发生在导入操作之后，此时员工列表可见。 应用程序完全加载列表后，你可以看到内存使用情况相当稳定。

> 注意：如果你使用 iPhone 6S 或 iPhone SE 以外的设备（包括 iOS 模拟器），你的内存条可能看起来与这些屏幕截图不完全相同。 利用率百分比将基于测试设备上的可用 RAM 量，这可能与 iPhone 6S 上的可用 RAM 不匹配。

考虑到应用程序中只有 50 条员工记录，RAM 使用率相当高。 Data Model 本身可能在这里有问题，所以这就是你开始调查的地方。

#### 探索数据源

在 Xcode 中，打开项目导航器并单击 EmployeeDirectory.xcdatamodeld 以查看 Data Model。 起始项目的模型由一个具有 11 个属性的 Employee 实体和一个具有两个属性的 Sale 实体组成。

在Employee实体上，about、address、department、email、guid、name、phone属性都是string类型； active 是一个布尔值； 图片是二进制数据； startDate 是一个日期，而 vacationDays 是一个整数。 Employee 与 Sale 具有一对多关系，其中包含一个 amount 整数属性和一个 date Date 属性。

首次启动时，该应用将从捆绑的 JSON 文件 seed.json 中导入样本数据。 这是 JSON 的摘录：

```
{
  "guid": "769adb89-82ad-4b39-be41-d02b89de7b94",
  "active": true,
  "picture": "face10.jpg",
  "name": "Kasey Mcfarland",
  "vacationDays": 2,
  "department": "Marketing",
  "startDate": "1979-09-05",
  "email": "kaseymcfarland@liquicom.com",
  "phone": "+1 (909) 561-2981",
  "address": "201 Lancaster Avenue, West Virginia, 2583",
  "about": "Dolore reprehenderit ... voluptate consectetur.\r\n"
},
```

> 注意：你可以通过修改位于 AppDelegate.swift 顶部的 amountToImport 和 addSalesRecords 常量来改变应用程序从 seed.json 文件导入的数据的数量和类型。 现在，将这些常量设置为默认值。

在性能方面，与个人资料图片的大小相比，显示员工姓名、部门、电子邮件地址和电话号码的文本是无关紧要的，后者大到足以潜在地影响列表的性能。

#### 进行更改以提高性能

高内存使用率的罪魁祸首可能是员工头像。 由于图片是作为二进制数据属性存储的，当你访问员工记录时，Core Data 将分配内存并加载整个图片——即使你只需要访问员工的姓名或电子邮件地址！

这里的解决方案是将图片拆分成单独的相关记录。 从理论上讲，你将能够高效地访问员工记录，然后仅在真正需要时才加载图片。

首先，通过单击 EmployeeDirectory.xcdatamodeld 打开可视化模型编辑器。 首先在你的模型中创建一个对象或实体。 在底部工具栏中，单击添加实体加号 (+) 按钮以添加新实体。

将实体命名为 EmployeePicture。 然后单击该实体并确保在“Utilities section”部分中选择了第四个选项卡。 将类更改为 EmployeePicture，将模块更改为当 Current Product Module，并将代码生成更改为手动/无。

确保通过单击左侧面板中的实体名称或图表视图中实体的图表来选择 EmployeePicture 实体。

接下来，单击并按住右下角的加号 (+) 按钮（在编辑器样式分段控件旁边），然后从弹出窗口中单击添加属性。 命名新的属性为 picture。

最后，在数据模型检查器中，将属性类型更改为二进制数据并选中允许外部存储选项。

你的编辑器应类似于以下内容：

![image-20230312022202118](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312022202118.png)

如前所述，二进制数据属性通常存储在数据库中。 如果选中允许外部存储选项，Core Data 会自动决定是将数据作为单独的文件保存到磁盘还是留在 SQLite 数据库中更好。

选择 Employee 实体并将 picture 属性重命名为 pictureThumbnail。 为此，在图表视图中选择 picture 属性，然后在 data model inspector 中编辑名称。

你已更新模型以将原始图片存储在单独的 Entity 中，并将缩略图版本存储在主 Employee 实体中。 当应用程序从 Core Data 获取 Employee 实体时，较小的缩略图不需要那么多 RAM。 完成项目其余部分的修改后，你将有机会对此进行测试并验证应用程序使用的 RAM 比以前少。

你可以通过关系将两个实体链接在一起。 这样，当应用程序需要更高质量、更大的图片时，它仍然可以通过关系检索它。

选择 Employee 实体并单击并按住右下角的加号 (+) 按钮。 这次，选择添加 relationship。 命名 relationship 为 picture，将目标设置为 EmployeePicture，最后将删除规则设置为级联。

Core Data relationship 应该总是双向的，所以现在添加一个对应关系。 选择 EmployeePicture 实体并添加新关系。 将新关系命名为 employee，将 Destination 设置为 Employee，最后将 Inverse 设置为 picture。

你的模型现在应该如下所示：

![image-20230312022639411](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312022639411.png)

现在你已经完成了对模型的更改，你需要为新的 EmployeePicture Entity 创建一个 NSManagedObject 子类。 这个子类将允许你从代码访问新 Entity。

右键单击 EmployeeDirectory 组文件夹并选择新建文件。 选择 Cocoa Touch Class 模板并单击 Next。 将类命名为 EmployeePicture 并使其成为 NSManagedObject 的子类。 确保选择 Swift 作为语言，单击下一步，最后单击创建。

从项目导航器中选择 EmployeePicture.swift 并将自动生成的代码替换为以下代码：

```swift
import Foundation
import CoreData

public class EmployeePicture: NSManagedObject {
}

extension EmployeePicture {
  @nonobjc 
  public class func fetchRequest() ->
    NSFetchRequest<EmployeePicture> {
    return NSFetchRequest<EmployeePicture>(
      entityName: "EmployeePicture")
  }

  @NSManaged public var picture: Data?
  @NSManaged public var employee: Employee?
}
```

这是一个非常简单的类，只有两个属性。 第一个是图片，与你刚刚在可视化数据模型编辑器中创建的 EmployeePicture 实体的单个属性相匹配。 第二个属性 employee 与你在 EmployeePicture Entity 上创建的关系相匹配。

> 注意：你也可以让 Xcode 自动创建 EmployeePicture 类。 要以这种方式添加新类，请打开 EmployeeDirectory.xcdatamodeld，转到编辑器 ▸ 创建 NSManagedObject 子类...，选择数据模型，然后在接下来的两个对话框中选择 EmployeePicture 实体。 在最后一个框中选择 Swift 作为语言选项。 如果有人问你，请拒绝创建 Objective-C 桥接头文件。 单击创建以保存文件。

接下来，从项目导航器中选择 Employee.swift 文件并更新代码以使用新的 pictureThumbnail 属性和图片关系。 将 picture 变量重命名为 pictureThumbnail 并添加一个名为 picture 的 EmployeePicture 类型的新变量。 你的变量现在将如下所示：

```swift
@NSManaged public var about: String?
@NSManaged public var active: NSNumber?
@NSManaged public var address: String?
@NSManaged public var department: String?
@NSManaged public var email: String?
@NSManaged public var guid: String?
@NSManaged public var name: String?
@NSManaged public var phone: String?
@NSManaged public var pictureThumbnail: Data?
@NSManaged public var picture: EmployeePicture?
@NSManaged public var startDate: Date?
@NSManaged public var vacationDays: NSNumber?
@NSManaged public var sales: NSSet?
```

接下来，你需要更新应用程序的其余部分以使用新的实体和属性。

打开 EmployeeListViewController.swift 并在 tableView(\_:cellForRowAt:) 中找到以下代码行。

它应该很容易找到，因为它会在下一行有一个错误标记！

```swift
if let picture = employee.picture {
```

这段代码从员工对象中获取 picture 数据。 现在完整的图片保存在一个单独的实体中，你应该使用新添加的 pictureThumbnail 属性。 更新文件以匹配以下代码：

```swift
if let picture = employee.pictureThumbnail {
```

接下来，打开 EmployeeDetailViewController.swift 并在 configureView() 中找到以下代码。 同样，它应该在错误旁边：

```swift
if let picture = employee.picture {
```

你需要更新设置的图片，就像你在 EmployeeListViewController.swift 中所做的那样。 与单元格图片一样，员工详细信息视图只有一张小图片，因此只需要缩略图版本。 将代码更新为如下所示：

```swift
if let picture = employee.pictureThumbnail {
```

接下来，打开EmployeePictureViewController.swift，在configureView()中找到如下代码：

```swift
guard let employeePicture = employee?.picture else {
  return
}
```

这一次，你要使用图片的高质量版本，因为图片将全屏显示。 更新文件以使用你在 Employee 实体上创建的图片关系来访问图片的高质量版本：

```swift
guard let employeePicture = employee?.picture?.picture else {
  return
}
```

在构建和运行之前还有一件事要做。 打开 AppDelegate.swift 并在 importJSONSeedData(\_:) 中找到以下代码行：

```swift
employee.picture = pictureData
```

现在你有了单独的实体来存储高质量图片，你需要更新这行代码以设置 pictureThumbnail 属性和图片关系。

将上面的行替换为以下内容：

```swift
employee.pictureThumbnail =
  imageDataScaledToHeight(pictureData, height: 120)

let pictureObject =
  EmployeePicture(context: coreDataStack.mainContext)

pictureObject.picture = pictureData

employee.picture = pictureObject
```

首先，你使用 imageDataScaledToHeight 将 pictureThumbnail 设置为原始图片的较小版本。 接下来，你创建一个新的 EmployeePicture 实体。

你将新 EmployeePicture 实体的图片属性设置为 pictureData 常量。 最后，将员工实体上的图片关系设置为新创建的图片实体。

> 注意：imageDataScaledToHeight 接收图像数据，将其调整为传入的高度并将质量设置为 80%，然后返回新的图像数据。

如果你的应用程序需要图片并通过网络调用检索数据，则应确保 API 尚未包含图片的较小缩略图版本。 像这样动态转换图像会产生很小的性能成本。

由于你更改了模型，请从你的测试设备中删除该应用程序。 构建并运行应用程序。 搏一搏！ 你应该准确地看到你之前看到的内容。

该应用程序应该像以前一样工作，你甚至可能会注意到缩略图导致的性能差异很小。 但进行此更改的主要原因是为了提高内存使用率。

#### 验证更改

现在你已经对项目进行了所有必要的更改，是时候看看你是否真的改进了应用程序。

在应用程序运行时，使用内存报告查看应用程序使用了多少 RAM。 这次它只消耗了大约 20 MB 到 60 MB 的 RAM，或者 iPhone 6S 总可用 RAM 的大约 2%。

![image-20230312024111651](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312024111651.png)

现在看报告的下半部分。 与上次一样，初始峰值来自导入操作，你可以忽略它。 这次平坦的面积要低得多。

![image-20230312024243401](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312024243401.png)

恭喜，你仅通过调整其数据模型就减少了该应用程序的 RAM 使用量！

首先，你使用内存报告工具测量了应用程序的性能。 接下来，你更改了 Core Data 存储和访问应用程序数据的方式。 最后，你验证了这些更改提高了应用程序的性能。

### 获取和性能

Core Data 是你应用程序数据的守护者。 任何时候你想要访问数据，你都必须通过 Fetch 请求来检索它。

例如，当应用程序加载员工列表时，它需要执行一次提取。 但是每次访问持久性存储都会产生开销。 你不想获取比你需要的更多的数据——只要足够你就不会经常返回到磁盘。 请记住，磁盘访问比 RAM 访问慢得多。

为了获得最佳性能，你需要在任何给定时间获取的对象数量与让许多记录占用 RAM 中宝贵空间的有用性之间取得平衡。

应用程序的启动时间有点慢，表明初始提取时出现问题。

#### 获取批量大小

核心数据获取请求包括 fetchBatchSize 属性，这使得获取足够的数据变得容易，但又不会太多。

如果你没有设置批处理大小，Core Data 会使用默认值 0，这会禁用批处理。

设置一个非零的正批量大小可以让你将返回的数据量限制为批量大小。 随着应用程序需要更多数据，Core Data 会自动执行更多批处理操作。 如果你搜索了 EmployeeDirectory 应用程序的源代码，你将看不到对 fetchBatchSize 的任何调用。 这表明另一个潜在的改进领域！

让我们看看是否有任何地方可以使用批量大小来提高应用程序的性能。

#### 测量问题

你将使用 Instruments 工具来分析获取操作在你的应用程序中的位置。

首先，选择一个 iPhone 模拟器目标，然后从 Xcode 的菜单栏中选择 Product，然后选择 Profile（或按 ⌘ + I）。 这将构建应用程序并启动 Instruments。

> 注意：你只能将 Instruments Core Data 模板与模拟器一起使用，因为该模板需要 DTrace 工具，而该工具在真实 iOS 设备上不可用。 你可能还需要为 Target 选择一个开发团队，以使 Instruments 能够运行。

你会看到以下选择窗口：

![image-20230312152247818](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312152247818.png)

选择 Core Data 模板并单击选择。 这将启动仪器窗口。 如果这是你第一次启动 Instruments，可能会要求你输入密码以授权 Instruments 分析正在运行的进程——别担心，在此对话框中输入密码是安全的。

Instruments 启动后，单击窗口左上角的 Record 按钮。

启动 EmployeeDirectory 后，上下滚动员工列表约 20 秒，然后单击出现在记录按钮位置的停止按钮。

单击 Fetches tool。 Instruments 窗口应类似于以下内容：

![image-20230312152429086](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312152429086.png)

默认的 Core Data 模板包括以下工具来帮助你调整和监控性能：

* Faults Instrument：捕获有关导致缓存未命中的故障事件的信息。 这有助于诊断低内存情况下的性能。
* Fetches Instrument：捕获获取次数和获取操作的持续时间。 这将帮助你平衡获取请求的数量与每个请求的大小。
* Saves Instrument：捕获有关 ManagedObjectContext 保存事件的信息。 将数据写入磁盘可能会影响性能和电池电量，因此该仪器可以帮助你确定是否应该将数据分批保存到一个大的保存中，而不是许多小的保存中。

由于你单击了 Fetches 工具，Instruments 窗口底部的详细信息部分会显示有关发生的每个提取的更多信息。

三行中的每一行都对应于应用程序中的同一行代码。 前两行是私有 Core Data，因此你可以忽略它们。

EmployeeDirectory 导入 50 名员工。 获取计数显示 50，这意味着该应用程序同时从 Core Data 获取所有员工。 那不是很有效！

Fetches 工具证实了你的体验，提取速度很慢并且很容易被注意到，正如你所看到的，它大约需要 5,000 微秒。 在使表视图可见并为用户交互做好准备之前，应用程序必须完成此提取。

> 注意：根据你的 Mac，屏幕上的数字（以及条的粗细）可能与这些屏幕截图中显示的不匹配。 更快的 Mac 将有更快的获取。 别担心——重要的是你在修改项目后看到的时间变化。

**改进性能的更改**

打开 EmployeeListViewController.swift 并在 employeeFetchRequest(\_:) 中找到以下代码行：

```swift
let fetchRequest: NSFetchRequest<Employee> =
  Employee.fetchRequest()
```

此代码使用 Employee 实体创建一个获取请求。 你还没有设置批处理大小，所以它默认为 0，这意味着不进行批处理。 将获取请求的批量大小设置为 10，将上面的内容替换为以下内容：

```swift
let fetchRequest: NSFetchRequest<Employee> =
  Employee.fetchRequest()
fetchRequest.fetchBatchSize = 10
```

你如何得出最佳批量大小？ 一个好的经验法则是将批量大小设置为在任何给定时间出现在屏幕上的项目数的两倍左右。 员工列表一次在屏幕上显示三到五名员工，因此 10 个是合理的批量大小。

**验证更改**

现在你已经对项目进行了必要的更改，是时候再次查看你是否真的改进了应用程序。

要测试此修复，首先构建并运行应用程序并确保它仍然有效。

接下来再次启动 Instruments：从 Xcode 中选择 Product，然后选择 Profile，或者按 ⌘ + I) 并重复之前执行的步骤。 请记住在单击 Instruments 中的停止按钮之前上下滚动员工列表约 20 秒。

> 注意：要使用最新代码，请确保从 Xcode 启动应用程序，这会触发构建，而不是仅仅点击 Instruments 中的红色按钮。

这一次，Core Data Instrument 应该是这样的：

![image-20230312154031343](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312154031343.png)

现在有多个fetch，初始fetch更快！

第一次 fetch 看起来与原始抓取类似，因为它抓取了所有 50 名员工。 然而，这一次，它只获取对象的数量，而不是完整的对象，这使得获取持续时间更短。 既然你已经根据请求设置了批处理大小，Core Data 会自动执行此操作。

第一次提取后，你可以看到大量提取，每批 10 个。当你滚动员工列表时，只有在需要时才提取新实体。

你已经将初始提取的时间减少到原来的近三分之一，并且后续提取的时间更小更快。 恭喜，你的应用程序速度再次提高了！

#### 高级 Fetch

Fetch 请求使用 predicates 来限制返回的数据量。 如上所述，为了获得最佳性能，你应该将获取的数据量限制在所需的最低限度：获取的数据越多，获取所需的时间就越长。

获取谓词性能：你可以使用谓词限制获取请求。 如果你的获取请求需要复合谓词，你可以通过将限制性更强的谓词放在首位来提高效率。 如果你的谓词包含字符串比较，则尤其如此。 例如，格式为“(active == YES) AND (name CONTAINS\[cd] %@)”的谓词可能比“(name CONTAINS\[cd] %@) AND (active == YES)”更有效 ”。

有关更多谓词性能优化，请参阅 Apple 的谓词编程指南：apple.co/2a1Rq2n。

构建并运行 EmployeeDirectory，然后选择标记为 Departments 的第二个选项卡。 此选项卡显示部门列表和每个部门的员工数。

点击部门单元格以查看所选部门的员工列表。

![image-20230312163945876](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312163945876.png)

点击每个部门单元格中的详细信息披露（也称为信息图标）以显示员工总数、在职员工和员工可用休假天数的明细。

第一个屏幕仅列出部门和每个部门的员工人数。 这里没有太多数据，但这里可能仍然存在性能问题。 让我们找出答案。

**衡量问题**

你将使用 XCTest 框架而不是 Instruments 来衡量部门列表屏幕的性能。 XCTest 通常用于单元测试，但它也包含用于测试性能的有用工具。

首先，熟悉应用程序如何创建部门列表屏幕。 打开 DepartmentListViewController.swift 并在 totalEmployeesPerDepartment() 中找到以下代码。

```swift
//1
let fetchRequest: NSFetchRequest<Employee> =
  Employee.fetchRequest()

let fetchResults: [Employee]
do {
  fetchResults = 
    try coreDataStack.mainContext.fetch(fetchRequest)
} catch let error as NSError {
  print("ERROR: \(error.localizedDescription)")
  return [[String: String]]()
}

//2
var uniqueDepartments: [String: Int] = [:]
for department in fetchResults.compactMap({ $0.department }) {
  uniqueDepartments[department, default: 0] += 1
}

//3
return uniqueDepartments.map { department, headCount in
  [
    "department": department,
    "headCount": String(headCount)
  ]
}
```

此代码执行以下操作：

1. 它使用 Employee Entuty 创建一个获取请求，然后获取所有员工。
2. 它遍历员工部门并构建一个字典，其中键是部门名称，值是该部门的员工人数。
3. 它构建了一个字典数组，其中包含部门列表屏幕所需的信息。

现在来衡量这段代码的性能。

打开 DepartmentListViewControllerTests.swift（注意文件名中的 Tests 后缀）并添加以下方法：

```swift
func testTotalEmployeesPerDepartment() {
  measureMetrics(
    [.wallClockTime],
    automaticallyStartMeasuring: false
  ) {
    let departmentList = DepartmentListViewController()
    departmentList.coreDataStack =
      CoreDataStack(modelName: "EmployeeDirectory")

    startMeasuring()
    _ = departmentList.totalEmployeesPerDepartment()
    stopMeasuring()
  }
}
```

此函数使用 measureMetrics 来查看代码执行所需的时间。

你每次都必须设置一个新的 Core Data Stack，这样你的结果就不会被 Core Data 出色的缓存能力所影响，这将使后续测试运行得非常快！

在该块内，你首先创建一个 DepartmentListViewController 并为其提供一个 CoreDataStack。 然后，你调用 totalEmployeesPerDepartment 来检索每个部门的员工数。

现在你需要运行这个测试。 从 Xcode 的菜单栏中，选择 Product，然后选择 Test，或按 ⌘ + U。这将构建应用程序并运行测试。

测试运行完成后，Xcode 将如下所示：

![image-20230312165840248](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312165840248.png)

注意两个新事物：

1. testTotalEmployeesPerDepartment 旁边有一个绿色复选标记。 这意味着测试运行并通过了。
2. 右侧有一条消息，其中包含测试所用的时间。

在低规格测试设备上，你可能会看到长达 0.252 秒的时间来执行 totalEmployeesPerDepartment 操作。 这些结果可能看起来不错，但仍有改进的余地。

> 注意：你可能会得到略有不同的测试结果，具体取决于你的测试设备。 别担心——重要的是你在修改项目后看到的时间变化。

**改进性能的更改**

totalEmployeesPerDepartment 的当前实现使用获取请求来遍历所有员工记录。 还记得本章中的第一个优化，将全尺寸照片拆分为一个单独的实体吗？ 这里有一个类似的问题：Core Data 加载了整个员工记录，但你真正需要的只是按部门统计员工数量。

以某种方式按部门对记录进行分组并对其进行计数会更有效。 你不需要员工姓名和照片缩略图等详细信息！

打开 DepartmentListViewController.swift 并在 totalEmployeesPerDepartment() 下面添加以下方法：

```swift
func totalEmployeesPerDepartmentFast() -> [[String: String]] {
  //1
  let expressionDescription = NSExpressionDescription()
  expressionDescription.name = "headCount"

  //2
  let arguments = [NSExpression(forKeyPath: "department")]
  expressionDescription.expression = NSExpression(
    forFunction: "count:",
    arguments: arguments
  )

  //3
  let fetchRequest: NSFetchRequest<NSDictionary> =
    NSFetchRequest(entityName: "Employee")
  fetchRequest.propertiesToFetch =
    ["department", expressionDescription]
  fetchRequest.propertiesToGroupBy = ["department"]
  fetchRequest.resultType = .dictionaryResultType

  //4
  var fetchResults: [NSDictionary] = []
  do {
    fetchResults =
      try coreDataStack.mainContext.fetch(fetchRequest)
  } catch let error as NSError {
    print("ERROR: \(error.localizedDescription)")
    return [[String: String]]()
  }
  return fetchResults as? [[String: String]] ?? []
}
```

这段代码仍然使用获取请求来填充部门列表屏幕，但它利用了 NSExpression。 它是这样工作的：

1. 首先，你创建一个 NSExpressionDescription 并将其命名为 headCount。
2. 接下来，你创建一个 NSExpression，其中包含 department 属性的 count: 函数。
3. 接下来，你使用 Employee 实体创建一个获取请求。 这一次，获取请求应该只使用 propertiesToFetch 获取所需的最少属性； 你只需要 department 属性和先前创建的表达式的计算属性。 提取请求还按 epartment 属性对结果进行分组。 你对托管对象不感兴趣，因此获取请求返回类型为 DictionaryResultType。 这将返回一个字典数组，每个字典包含一个部门名称和一个员工人数——正是你所需要的！
4. 最后，执行获取请求。

在viewDidLoad()中找到如下一行代码：

```swift
items = totalEmployeesPerDepartment()
```

这行代码使用旧的和缓慢的功能来填充部门列表屏幕。 通过调用你刚刚创建的函数替换它：

```swift
items = totalEmployeesPerDepartmentFast()
```

现在，应用程序使用更快的、支持 NSExpression 的获取请求填充部门列表屏幕的表视图数据源。

> 注意：NSExpression 是一个强大的 API，但很少使用，至少很少直接使用。 当你使用比较操作创建谓词时，你可能不知道，但你实际上是在使用表达式。 NSExpression 中有许多预置的统计和算术表达式，包括 average、sum、count、min、max、median、mode 和 stddev。

请查阅 NSExpression 文档以获得全面的概述。

**验证更改**

现在你已经对项目进行了所有必要的更改，现在是时候再次查看你是否提高了应用程序的性能。

打开 DepartmentListViewControllerTests.swift 并添加一个新函数来测试你刚刚创建的 totalEmployeesPerDepartmentFast 函数。

```swift
func testTotalEmployeesPerDepartmentFast() {
  measureMetrics(
    [.wallClockTime],
    automaticallyStartMeasuring: false
  ) {
    let departmentList = DepartmentListViewController()
    departmentList.coreDataStack =
      CoreDataStack(modelName: "EmployeeDirectory")

    startMeasuring()
    _ = departmentList.totalEmployeesPerDepartmentFast()
    stopMeasuring()

  }
}
```

和以前一样，此测试使用 measureMetrics 来查看特定功能花费的时间； 在这种情况下，totalEmployeesPerDepartmentFast。

现在你需要运行这个测试。 从 Xcode 的菜单栏中，选择 Product，然后选择 Test，或按 ⌘ + U。这将构建应用程序并运行测试。 测试运行完成后，Xcode 将类似于以下内容：

![image-20230312170701710](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312170701710.png)

这一次，你会看到两条包含总执行时间的消息，一条消息紧挨着每个测试函数。

> 注意：如果你没有看到时间消息，你可以在测试期间生成的日志中查看每个单独测试运行的详细信息。 从 Xcode 的菜单栏中，选择 View、Debug Area，然后选择 Show Debug Area。

根据你的测试设备，新函数 totalEmployeesPerDepartmentFast 大约需要 0.002 秒才能完成。 这比原始函数 totalEmployeesPerDepartment 使用的 0.1 到 0.3 秒快得多。 你的速度提高了 100% 以上！

#### 获取计数

正如你已经看到的，你的应用程序并不总是需要来自 Core Data 对象的所有信息； 一些屏幕只需要具有特定属性的对象的数量。

例如，员工详细信息屏幕显示员工自加入公司以来的销售总数。

就此应用而言，你不关心每笔销售的内容——例如，销售日期或购买者的姓名——只关心总共有多少销售。

衡量问题

你将再次使用 XCTest 来衡量员工详细信息屏幕的性能。

打开 EmployeeDetailViewController.swift 并找到 salesCountForEmployee(\_:)。

```swift
func salesCountForEmployee(_ employee: Employee) -> String {
  let fetchRequest: NSFetchRequest<Sale> = Sale.fetchRequest()
  fetchRequest.predicate = NSPredicate(
    format: "%K = %@",
    argumentArray: [#keyPath(Sale.employee), employee]
  )

  let context = employee.managedObjectContext
  do {
    let results = try context?.fetch(fetchRequest)
    return "\(results?.count ?? 0)"
  } catch let error as NSError {
    print("Error: \(error.localizedDescription)")
    return "0"
  }
}
```

获取完整的销售对象只是为了查看给定员工的销售额可能是浪费的。 这可能是另一个提升性能的机会！

在尝试解决问题之前，让我们衡量一下问题。

打开 EmployeeDetailViewControllerTests.swift 并找到 testCountSales()。

```swift
func testCountSales() {
  measureMetrics(
    [.wallClockTime],
    automaticallyStartMeasuring: false
  ) {
    let employee = getEmployee()
    let employeeDetails = EmployeeDetailViewController()
    startMeasuring()
    _ = employeeDetails.salesCountForEmployee(employee)
    stopMeasuring()
  }
}
```

与前面的示例一样，此函数使用 measureMetrics 来查看单个函数运行需要多长时间。 该测试从一个方便的方法中获取一个员工，创建一个 EmployeeDetailViewController，开始测量然后调用有问题的方法。

从 Xcode 的菜单栏中，选择 Product，然后选择 Test，或按 ⌘ + U。这将构建应用程序并运行测试。

![image-20230312172101779](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230312172101779.png)

性能还算不错——但仍有一些改进空间。

**改进性能的更改**

在前面的示例中，你使用 NSExpression 对数据进行分组并按部门提供员工计数，而不是返回实际记录本身。 你会在这里做同样的事情。

打开 EmployeeDetailViewController.swift 并将以下代码添加到 salesCountForEmployee(\_:) 方法下方的类中。

```swift
func salesCountForEmployeeFast(_ employee: Employee) -> String {

  let fetchRequest: NSFetchRequest<Sale> = Sale.fetchRequest()
  fetchRequest.predicate = NSPredicate(
    format: "%K = %@",
    argumentArray: [#keyPath(Sale.employee), employee]
  )

  let context = employee.managedObjectContext

  do {
    let results = try context?.count(for: fetchRequest)
    return "\(results ?? 0)"
  } catch let error as NSError {
    print("Error: \(error.localizedDescription)")
    return "0"
  }
}
```

此代码与你在上一节中查看的函数非常相似。 主要区别在于，你现在调用 count(for:)，而不是调用 execute(\_:)。 在configureView()中找到如下一行代码：

```swift
salesCountLabel.text = salesCountForEmployee(employee)
```

这行代码使用旧的销售计数功能来填充部门详细信息屏幕上的标签。 通过调用你刚刚创建的函数替换它：

```swift
salesCountLabel.text = salesCountForEmployeeFast(employee)
```

**验证更改**

现在你已经对项目进行了必要的更改，是时候再次查看你是否改进了应用程序。 打开 EmployeeDetailViewControllerTests.swift 并添加一个新函数来测试你刚刚创建的 salesCountForEmployeeFast 函数。

```swift
func testCountSalesFast() {
  measureMetrics(
    [.wallClockTime],
    automaticallyStartMeasuring: false
  ) {
    let employee = getEmployee()
    let employeeDetails = EmployeeDetailViewController()
    startMeasuring()
    _ = employeeDetails.salesCountForEmployeeFast(employee)
    stopMeasuring()
  }
}
```

这个测试与之前的测试相同，只是它使用了新的并且幸运的是速度更快的函数。

从 Xcode 的菜单栏中，选择 Product，然后选择 Test，或按 ⌘U。 这将构建应用程序并运行测试。

看起来不错——又一次性能提升！

**使用 relationship**

上面的代码很快，但更快的方法似乎仍然需要很多工作。 你必须创建提取请求、创建谓词、获取对上下文的引用、执行提取请求并获取结果。

Employee 实体有一个 sales 属性，它包含一个包含 Sale 类型对象的 Set。

打开 EmployeeDetailViewController.swift 并在 salesCountForEmployeeFast(\_:) 方法下方添加以下新方法：

```swift
func salesCountForEmployeeSimple(
  _ employee: Employee
) -> String {
  "\(employee.sales?.count ?? 0)"
}
```

这样看起来好多了。 通过在 Employee 实体上使用销售关系，代码变得更加简单 — 也更容易理解。

按照与上述相同的模式，更新视图控制器和测试以改为使用此方法。

立即查看性能变化。

### 关键点

由于 Core Data 的内置优化（例如故障），Core Data 的大多数实现已经快速且轻便。

* 在对 Core Data 性能进行改进时，你应该进行测量，进行有针对性的更改，然后再次进行测量以验证你的更改是否产生了预期的影响。
* 对数据模型的小改动，例如将大型二进制 blobs 移动到其他实体，可以提高性能。
* 为了获得最佳性能，你应该将获取的数据量限制在所需的最低限度：获取的数据越多，获取所需的时间就越长。
* 性能是内存和速度之间的平衡。 在你的应用程序中使用 Core Data 时，请始终牢记这种平衡。

### 挑战

使用你刚学到的技术，尝试提高 DepartmentDetailsViewController 类的性能。 不要忘记编写测试来测量执行前后的时间。 提示一下，有许多方法提供计数，而不是完整记录； 这些可能会以某种方式进行优化，以避免加载记录的内容。
