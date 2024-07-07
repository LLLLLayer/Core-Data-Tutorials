# 第 2 节：NSManagedObject 子类

## 第 2 节：NSManagedObject 子类

在第 1 节中，你接触了一个简单的 Core Data 应用程序；现在是探索 Core Data 提供的更多内容的时候了！

本章的核心是 NSManagedObject 的子类化，为每个 Data Entity 创建自己的类。这会在 Data Model Editor 中的 Entity 与代码中的类之间创建直接的一对一映射。这意味着在代码的某些部分，你可以使用对象和属性，而不必过多担心 Core Data 方面的事情。

在此过程中，你将了解 Core Data Entitiy 中可用的所有数据类型，包括一些常见的字符串和数字类型等其他的类型。使用所有可用的数据类型，你还将了解如何验证数据以在保存前自动检查值。

### 入门

转到本书随附的文件并打开起始文件夹中名为 BowTies 的示例项目。与 HitList 一样，该项目使用 Xcode 的支持 Core Data 的模板。和以前一样，这意味着 Xcode 生成了自己的即用型 Core Data stack，位于 AppDelegate.swift 中。

打开 Main.storyboard。你可以在此处找到示例项目的单页 UI：

![image-20221115014757542](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221115014757542.png)

正如你可能猜到的那样，BowTies 是一个轻量级领结管理应用程序。你可以使用最顶部的分段控件在你拥有的不同颜色的领结之间切换——该应用假定每一种颜色都有一个。点击“R”表示红色，“O”表示橙色，依此类推。

点击特定颜色会弹出领带图像，并在屏幕上显示有关领带的特定信息的多个标签。这包括：

* 领结的名字（所以你可以区分颜色相似的领结）
* 你戴领带的次数
* 你最后一次戴领带的日期
* 这条领带是否是你的最爱

左下角的 Wear 按钮会增加你佩戴该特定领带的次数，并将最后一次佩戴日期设置为今天。

橙色不是你的颜色？不用担心。右下角的评级按钮更改领结的评级。这个特定的评级系统使用从 0 到 5 的等级，允许十进制值。

这就是应用程序在其最终状态下应该做的事情。打开 ViewController.swift 查看它当前做了什么：

```swift
import UIKit

class ViewController: UIViewController {

  // MARK: - IBOutlets
  @IBOutlet weak var segmentedControl: UISegmentedControl!
  @IBOutlet weak var imageView: UIImageView!
  @IBOutlet weak var nameLabel: UILabel!
  @IBOutlet weak var ratingLabel: UILabel!
  @IBOutlet weak var timesWornLabel: UILabel!
  @IBOutlet weak var lastWornLabel: UILabel!
  @IBOutlet weak var favoriteLabel: UILabel!
  @IBOutlet weak var wearButton: UIButton!
  @IBOutlet weak var rateButton: UIButton!

  // MARK: - View Life Cycle
  override func viewDidLoad() {
    super.viewDidLoad()
  }

  // MARK: - IBActions
  @IBAction func segmentedControl(
    _ sender: UISegmentedControl) {

  }

  @IBAction func wear(_ sender: UIButton) {

  }

  @IBAction func rate(_ sender: UIButton) {

  }
}
```

坏消息是目前的状态，BowTies 什么也没做。好消息是你不需要进行任何 Ctrl 拖动！

用户界面上的分段控件和所有标签已经在代码中连接到 IBOutlets。 此外，分段控制、Wear 和 Rate 按钮都有对应的 IBActions。

看起来你已经拥有了开始添加一些 Core Data 所需的一切——但是等等，你要在屏幕上显示什么？ 没有输入法可言，所以应用程序必须附带示例数据。这完全正确。 BowTies 包含一个名为 SampleData.plist 的属性列表，其中包含七个示例领带的信息，每个领带代表彩虹的每种颜色。

![image-20221115015345644](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221115015345644.png)

此外，应用程序的 Assets.xcassets 包含与 SampleData.plist 中的七个领结对应的七个图像。

你现在要做的就是获取此示例数据，将其存储在 Core Data 中并使用它来实现领结管理功能。

### 构造 Data

在上一章中，你了解了在开始一个新的 Core Data 项目时首先要做的事情之一就是创建你的 data model。

打开 BowTies.xcdatamodeld 并单击左下角的 Add Entity 以创建一个新实体。双击新实体并将其名称更改为 BowTie，如下所示：

![image-20221115015702449](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221115015702449.png)

在上一节中，你创建了一个简单的 Person Entity，它有一个 String 属性来保存 name。 Core Data 支持其他几种数据类型，你将在新的 BowTie 实体中使用它们中的大部分。

属性的数据类型决定了你可以在其中存储什么样的数据以及它将在磁盘上占用多少空间。在 Core Data 中，属性的数据类型以 Undefined 开头，因此你必须将其更改为其他类型。

如果你还记得 SampleData.plist，每个领结都有十条相关信息。 这意味着 BowTie Entity 最终将在 model editor 中具有至少十个属性。

选择左侧的 BowTie，然后单击 Attributes 下的加号 (+)。 将新属性的名称更改为 name 并将其类型设置为 String：

![image-20221115020016223](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221115020016223.png)

再重复此过程七次以添加以下属性：

* 名为 isFavorite 的 Boolean
* 名为 lastWorn 的 Date
* 名为 rating 的 Double
* 名为 searchKey 的 String
* 名为 timesWorn 的 Integer 32
* 名为 id 的 UUID
* 名为 url 的 URI

这些数据类型中在日常编程中都很常见。如果你之前没有听说过 UUID，它是通用唯一标识符的缩写，通常用于唯一标识信息。

URI代表统一资源标识符，用于命名和标识不同的资源，如文件和网页。事实上，所有的 URL 都是 URI！

完成后，你的属性部分应类似于以下内容：

![image-20221115020511705](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221115020511705.png)

如果属性的顺序不同，请不要担心——重要的是属性名称和类型是正确的。

> 注意：你可能已经注意到 timesWorn 整数属性有三个选项：Integer 16、Integer 32 或 Integer 64。
>
> 16、32、64是指表示整数的位数。这一点很重要，原因有二：位数反映了一个整数在磁盘上占用的空间大小以及它可以表示多少个值，也称为它的范围。以下是三种整数的范围：
>
> 16 位整数的范围：-32768 到 32767
>
> 32 位整数的范围：–2147483648 到 2147483647
>
> 64 位整数的范围：–9223372036854775808 到 9223372036854775807

你如何选择？你的数据来源将决定最佳的整数类型。你假设你的用户真的很喜欢领结，所以一个 32 位整数应该提供足够的存储空间供领结佩戴一生。

每个领结都有一个关联的图像。你将如何将其存储在 Core Data 中？向 BowTie 实体添加一个属性，将其命名为 photoData 并将其数据类型更改为二进制数据：

![image-20221115020637045](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221115020637045.png)

Core Data 提供了直接在 Data Model 中存储任意二进制数据块的选项。这些可以是任何东西，从图像到 PDF 文件，再到任何可以序列化为 0 和 1 的东西。

正如你所想象的，这种便利可能会付出高昂的代价。在与其他属性相同的 SQLite 数据库中存储大量二进制数据可能会影响应用程序的性能。这意味着每次访问实体时都会将一个巨大的二进制加载到内存中，即使你只需要访问它的名称也是如此！

幸运的是，Core Data 预见到了这个问题。选择 photoData 属性后，打开 Attributes Inspector 并选中 Allows External Storage 选项。

![image-20221115020854593](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221115020854593.png)

当你启用 Allows External Storage 时，Core Data 会根据每个值试探性地决定是否应将数据直接保存在数据库中或存储指向单独文件的 URI。

> 注意：允许外部存储选项仅适用于二进制数据属性类型。此外，如果你打开它，你将无法使用该属性查询 Core Data。

综上所述，除了 Strings、Integers、Doubles、Booleans 和 Dates，Core Data 还可以保存 Binary Data，而且可以高效智能地保存。

### 在 Core Data 中存储非标准数据类型

不过，你可能还想保存许多其他类型的数据。 例如，如果必须存储 UIColor 的实例，你会怎么做？

使用目前提供的选项，你必须将颜色解构为其单独的组件并将它们保存为整数（例如，红色：255，绿色：101，蓝色：155）。 然后，在获取这些组件后，你必须在运行时重新构建颜色。

或者，你可以将 UIColor 实例序列化为 Data 并将其保存为二进制数据。 然后，你还必须在之后“加水”以将二进制数据重新构造回你最初想要的 UIColor 对象。

Core Data 再次为你提供支持。 如果你仔细查看 SampleData.plist，你可能会注意到每个领结都有一个相关联的颜色。 在 model editor 中选择 BowTie 实体并添加一个名为 tintColor 且类型为 Transformable 的新属性。

![image-20221116015322729](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221116015322729.png)

Transformable 属性持久化 Xcode 的 Data Model Inspector 中未列出的数据类型。这些包括 Apple 在其框架中提供的类型，例如 UIColor 和 CLLocationCoordinate2D 以及你自己的类型。

Transformable 属性非常强大和灵活，但你需要预先做一些工作来告诉 iOS 如何将这些类型与数据相互转换。你必须满足三个要求才能使用 Transformable 属性：

1. 将 NSSecureCoding 协议添加到将受支持数据类型。
2. 创建并注册一个 NSSecureUnarchiveFromDataTransformer 子类。
3. 将自定义 data transformer 子类与 Data Model Editor 中的 Transformable 属性相关联。

由于你正在处理 UIColor，好消息是它已经符合 NSSecureCoding。 Apple 框架中的大多数数据类型都可以。

要满足第二个要求，请单击 File\New\File... 并从 Cocoa Touch 模板创建一个文件。将文件命名为 ColorAttributeTransformer 并使其成为 NSSecureUnarchiveFromDataTransformer 的子类。单击下一步并将文件保存在你的项目中。

![image-20221116015733828](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221116015733828.png)

接下来，用以下实现替换新文件的内容。

```swift
import UIKit

class ColorAttributeTransformer:
  NSSecureUnarchiveFromDataTransformer {

  //1
  override static var allowedTopLevelClasses: [AnyClass] {
    [UIColor.self]
  }

  //2
  static func register() {
    let className =
      String(describing: ColorAttributeTransformer.self)
    let name = NSValueTransformerName(className)

    let transformer = ColorAttributeTransformer()
    ValueTransformer.setValueTransformer(
      transformer, forName: name)

  }
}
```

下面是这段代码的作用：

1. 覆盖 allowedTopLevelClasses 以返回此 DataTransformer 可以 decode 的类列表。我们想要持久化和检索 UIColor 的实例，因此在这里你返回一个仅包含该类的数组。
2. 顾名思义，静态函数 register() 帮助你使用 ValueTransformer 注册你的子类。但是为什么你需要这样做呢？ ValueTransformer 维护一个键值映射，其中键是你使用 NSValueTransformerName 提供的名称，值是相应转换器的实例。你稍后将在数据模型编辑器中需要此映射。

接下来，打开 AppDelegate.swift 并将 application(\_:didFinishLaunchingWithOptions:) 替换为以下实现：

```swift
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions
  launchOptions: [UIApplication.LaunchOptionsKey: Any]?)
                 -> Bool {

  ColorAttributeTransformer.register()

  return true
}
```

在这里，你使用之前实现的静态方法注册 DataTransformer。注册可以在你的应用程序设置 CoreDataStack 之前的任何时候进行。

接下来，回到 BowTies.xcdatamodeld，选择 tintColor 属性并打开 Data Model Inspector。将 Transformer 的值更改为 ColorAttributeTransformer，并将 Custom Class 设置为 UIColor。

![image-20221116020322263](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221116020322263.png)

你的数据模型现已完成。 BowTie 实体具有将所有信息存储在 SampleData.plist 中所需的十个属性。

### Managed Object 子类

在上一章的示例项目中，你使用 KVC 来访问 Person 实体的属性。它看起来类似于以下内容：

```swift
// Set the name
person.setValue(aName, forKeyPath: "name")

// Get the name
let name = person.value(forKeyPath: "name")
```

即使你可以使用 KVC 直接在 NSManagedObject 上做任何事情，但这并不意味着你应该这样做！

KVC 的最大问题是你使用字符串而不是强类型类访问数据。这通常被开玩笑地称为编写字符串类型的代码。

你可能从经验中知道，字符串类型的代码容易出现愚蠢的人为错误，例如打字错误和拼写错误。键值编码也没有充分利用 Swift 的类型检查和 Xcode 的自动完成功能。 “一定有别的办法！”你可能在想，你是对的。

KVC 的最佳替代方法是为 Data Model 中的每个 Entity 创建 NSManagedObject 子类。这意味着将有一个 BowTie 类，每个属性都有正确的类型。

Xcode 可以手动或自动为你生成子类。你为什么希望 Xcode 为你做这件事？如果你永远不必查看或更改它们，那么生成这些子类文件并让它们弄乱你的项目可能会有点麻烦。从 Xcode 8 开始，你可以在每个实体的基础上选择让 Xcode 自动生成和更新这些文件，并将它们存储在项目的派生数据文件夹中。

使用 Model Editor 时，此设置位于 Data Model Inspector 的 Codegen 中。因为你在本书中学习 Core Data，所以你不会使用自动代码生成。

确保你仍然打开 BowTies.xcdatamodeld，选择 BowTie 实体并打开数 Data Model Inspector。将 Codegen 下拉菜单设置为 Manual/None，如下所示：

![image-20221116020911913](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221116020911913.png)

> 注意：确保将 BowTie Entity 添加到 Model 后，第一次编译之前，更改此生成设置。
>
> 如果你在第一次编译后设置代码生成设置，你将拥有两个版本的托管对象子类：一个在派生数据中，另一个在源代码中。如果发生这种情况，当你再次尝试编译时会遇到问题。

接下来，转到 Editor\Create NSManagedObject Subclass...。在接下来的两个对话框中选择 Data Model，然后选择 BowTie entity。单击创建以保存文件。

Xcode 为你生成了两个 Swift 文件，一个名为 BowTie+CoreDataClass.swift，另一个名为 BowTie+CoreDataProperties.swift。打开 BowTie+CoreDataClass.swift。它应该类似于以下内容：

```swift
import Foundation
import CoreData

@objc(BowTie)
public class BowTie: NSManagedObject {

}
```

接下来，打开 BowTie+CoreDataProperties.swift。你生成的属性可能与此处显示的顺序不同，但该文件应类似于以下内容：

```swift
import Foundation
import CoreData

extension BowTie {

  @nonobjc public class func fetchRequest() 
    -> NSFetchRequest<BowTie> {
    

    return NSFetchRequest<BowTie>(entityName: "BowTie")

  }

  @NSManaged public var name: String?
  @NSManaged public var isFavorite: Bool
  @NSManaged public var lastWorn: Date?
  @NSManaged public var rating: Double
  @NSManaged public var searchKey: String?
  @NSManaged public var timesWorn: Int32
  @NSManaged public var id: UUID?
  @NSManaged public var url: URL?
  @NSManaged public var photoData: Data?
  @NSManaged public var tintColor: UIColor?
}

extension BowTie : Identifiable {

}
```

在面向对象的说法中，对象是一组值以及在这些值上定义的一组操作。在这种情况下，Xcode 将这两个东西分成两个单独的文件。值（即与 data model 中的 BowTie 属性对应的属性）在 BowTie+CoreDataProperties.swift 中，而操作在当前为空的 BowTie+CoreDataClass.swift 中。

> 注意：你可能想知道为什么会生成两个单独的文件。将自定义代码添加到 NSManagedObject 子类是很常见的。如果你在 Model Editor 中更新 BowTie Entity 并再次转到 Editor\Create NSManagedObject Subclass...，你不会希望丢失该代码。代码创建很智能 - 如果 BowTie+CoreDataClass.swift 已经存在，它只会创建 BowTie+CoreDataProperties.swift，保留所有自定义代码。这就是 Core Data 生成两个文件的主要原因，而不是像以前的 Xcode 版本那样生成一个文件。

Xcode 已经为 Data Model 中的每个属性创建了一个具有属性的类。由于它创建了 UIColor 的属性，因此你需要在 import CoreData 下添加以下 import UIKit 以修复错误。

Foundation 或 Swift 标准库中对于模型编辑器中的每个属性类型都有一个相应的类。这是属性类型到运行时类的完整映射：

* String maps to String?
* Integer 16 maps to Int16
* Integer 32 maps to Int32
* Integer 64 maps to Int64
* Float maps to Float
* Double maps to Double
* Boolean maps to Bool
* Decimal maps to NSDecimalNumber?
* Date maps to Date?
* URI maps to URL?
* UUID maps to UUID?
* Binary data maps to Data?
* Transformable maps to NSObject?

> 注意：与 Objective-C 中的 @dynamic 类似，@NSManaged 属性通知 Swift 编译器，属性的后备存储和实现将在运行时而不是编译时提供。

常规属性由内存中的实例变量支持。Managed Object 的属性不同：它由 ManagedObjectContext 提供，因此在编译时不知道数据的来源。

请注意，由于你为 tintColor 属性提供了 UIColor 的自定义类，Core Data 已经使用 UIColor? 生成了该属性，而不是 NSObject?。

恭喜，你刚刚在 Swift 中创建了你的第一个 Managed Object 子类！

与 KVC 相比，这是一种更好的处理 Core Data 实体的方法，并且有两个主要好处：

1. Managed object subclasses 释放了 Swift 属性的语法能力。通过使用属性而不是 KVC 访问属性，你可以与 Xcode 和编译器友好相处。
2. 你获得了覆盖现有方法或添加自己的方法的能力。注意有一些 NSManagedObject 方法你永远不能覆盖。查看 Apple 的 NSManagedObject 文档以获取完整列表。

为了确保数据模型和新的托管对象子类之间的所有内容都正确连接，你将执行一个小测试。

打开 AppDelegate.swift 并将 application(\_:didFinishLaunchingWithOptions:) 替换为以下实现：

```swift
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions
  launchOptions: [UIApplication.LaunchOptionsKey: Any]?)
                 -> Bool {

  ColorAttributeTransformer.register()

  // Save test bow tie
  let bowtie = NSEntityDescription.insertNewObject(
    forEntityName: "BowTie",
    into: self.persistentContainer.viewContext) as! BowTie
  bowtie.name = "My bow tie"
  bowtie.lastWorn = Date()
  saveContext()

  // Retrieve test bow tie
  let request: NSFetchRequest<BowTie> = BowTie.fetchRequest()

  if let ties =
    try? self.persistentContainer.viewContext.fetch(request),
    let testName = ties.first?.name,
    let testLastWorn = ties.first?.lastWorn {
    print("Name: \(testName), Worn: \(testLastWorn)")
  } else {
    print("Test failed.")
  }

  return true
}
```

在应用程序启动时，此测试会创建一个领结设置其名称和 lastWorn 属性，并在 ManagedObjectContext 保存。之后，它立即获取所有 BowTie 实体并将第一个实体的名称和 lastWorn 日期打印到控制台；此时应该只有一个。构建并运行应用程序并密切关注控制台：

```swift
Name: My bow tie, Worn: 2022-11-15 18:24:39 +0000
```

如果你一直在仔细遵循，请按预期将 name 和 lastWorn 打印到控制台。这意味着你能够成功保存和获取 BowTie 托管对象子类。掌握了这些新知识，是时候实现整个示例应用程序了。

### 传递 ManagedContext

打开 ViewController.swift 并在 import UIKit 下方添加以下内容：

```swift
import CoreData
```

接下来，在最后一个 IBOutlet 属性下面添加以下内容：

```swift
// MARK: - Properties
var managedContext: NSManagedObjectContext!
```

重申一下，在 Core Data 中执行任何操作之前，你首先必须获得一个 NSManagedObjectContext 才能使用。 了解如何将 ManagedObjectContext 传递到应用程序的不同部分是 Core Data 编程的一个重要方面。

打开 AppDelegate.swift 并将当前包含测试代码的 application(\_:didFinishLaunchingWithOptions:) 替换为之前的实现：

```swift
func application(_ application: UIApplication,
                  didFinishLaunchingWithOptions
  launchOptions: [UIApplication.LaunchOptionsKey: Any]?)
  -> Bool {
  ColorAttributeTransformer.register()
  return true
}
```

你有七个领结很想进入你的 Core Data 存储。 打开 ViewController.swift 并在 rate(\_:) 下面添加以下方法：

```swift
// Insert sample data
func insertSampleData() {

  let fetch: NSFetchRequest<BowTie> = BowTie.fetchRequest()
  fetch.predicate = NSPredicate(format: "searchKey != nil")

  let tieCount = (try? managedContext.count(for: fetch)) ?? 0

  if tieCount > 0 {
    // SampleData.plist data already in Core Data
    return
  }

  let path = Bundle.main.path(forResource: "SampleData",
                              ofType: "plist")
  let dataArray = NSArray(contentsOfFile: path!)!

  for dict in dataArray {
    let entity = NSEntityDescription.entity(
      forEntityName: "BowTie",
      in: managedContext)!
    let bowtie = BowTie(entity: entity,
                        insertInto: managedContext)
    let btDict = dict as! [String: Any]

    bowtie.id = UUID(uuidString: btDict["id"] as! String)
    bowtie.name = btDict["name"] as? String
    bowtie.searchKey = btDict["searchKey"] as? String
    bowtie.rating = btDict["rating"] as! Double
    let colorDict = btDict["tintColor"] as! [String: Any]
    bowtie.tintColor = UIColor.color(dict: colorDict)
    
    let imageName = btDict["imageName"] as? String
    let image = UIImage(named: imageName!)
    bowtie.photoData = image?.pngData()
    bowtie.lastWorn = btDict["lastWorn"] as? Date
    
    let timesNumber = btDict["timesWorn"] as! NSNumber
    bowtie.timesWorn = timesNumber.int32Value
    bowtie.isFavorite = btDict["isFavorite"] as! Bool
    bowtie.url = URL(string: btDict["url"] as! String)

  }
  try? managedContext.save()
}
```

Xcode 会抱怨缺少 UIColor 上的方法声明。 要解决此问题，请将以下私有 UIColor 扩展名添加到文件末尾最后一个大括号下方。

```swift
private extension UIColor {

  static func color(dict: [String: Any]) -> UIColor? {
    guard 
      let red = dict["red"] as? NSNumber,
      let green = dict["green"] as? NSNumber,
      let blue = dict["blue"] as? NSNumber else {
        return nil
    }
    

    return UIColor(
      red: CGFloat(truncating: red) / 255.0,
      green: CGFloat(truncating: green) / 255.0,
      blue: CGFloat(truncating: blue) / 255.0,
      alpha: 1)

  }
}
```

这是相当多的代码，但都相对简单。第一个方法，insertSampleData，检查任何 BowTie；稍后你将了解这是如何工作的。如果不存在，它会在 SampleData.plist 中获取领结信息，遍历每个领结字典并将新的 BowTie 实体插入到你的 Core Data Store 中。在此迭代结束时，它保存 ManagedObjectContext 属性以将这些更改提交到磁盘。

你通过私有扩展添加到 UIColor 的 color(dict:) 方法也很简单。 SampleData.plist 将颜色存储在包含三个键的字典中：红色、绿色和蓝色。这个静态方法接受这个字典并返回一个真正的 UIColor。

这里有两点需要特别注意：

1. 在 Core Data 中存储图像的方式。属性列表包含每个领结的文件名，而不是文件图像——实际图像在项目的资产目录中。使用此文件名，你可以实例化 UIImage 并立即通过 pngData() 将其转换为数据，然后再将其存储在 imageData 属性中。
2. 存储颜色的方式。尽管颜色存储在可转换属性中，但在将其存储在 tintColor 中之前不需要任何特殊处理。你只需设置属性即可。

前面的方法将你在 SampleData.plist 中的所有领结数据插入到 Core Data 中。现在你需要从某个地方访问数据！

接下来，将 viewDidLoad() 替换为以下实现：

```swift
// MARK: - View Life Cycle
override func viewDidLoad() {
  super.viewDidLoad()

  let appDelegate = 
    UIApplication.shared.delegate as? AppDelegate
  managedContext = appDelegate?.persistentContainer.viewContext

  //1
  insertSampleData()

  //2
  let request: NSFetchRequest<BowTie> = BowTie.fetchRequest()
  let firstTitle = segmentedControl.titleForSegment(at: 0) ?? ""
  request.predicate = NSPredicate(
    format: "%K = %@",
    argumentArray: [#keyPath(BowTie.searchKey), firstTitle])

  do {
    //3
    let results = try managedContext.fetch(request)

    //4
    if let tie = results.first {
      populate(bowtie: tie)
    }

  } catch let error as NSError {
    print("Could not fetch \(error), \(error.userInfo)")
  }
}
```

这是你从 Core Data 获取 BowTie 并填充 UI 的地方。

一步一步，这是你用这段代码做的事情：

1. 你调用之前实现的 insertSampleData()。由于每次启动应用程序时都可以调用 viewDidLoad() ，因此 insertSampleData() 执行一次提取以确保它不会多次将示例数据插入到 Core Data 中。
2.  你创建一个获取请求以获取新插入的 BowTie Entities。segmented control 有按颜色筛选的选项卡，因此 predicate 添加条件以查找与所选颜色匹配的领结。predicate 既非常灵活又非常强大——你将在后文中阅读更多关于它们的内容。

    现在，知道这个特定的 predicate 正在寻找领结，其 searchKey 属性设置为分段控件的第一个按钮标题：在本例中为 R。
3. 一如既往，ManagedObjectContext 为你完成繁重的工作。它执行你刚才制作的获取请求并返回一个 BowTie 对象数组。
4. 用结果数组中的第一个领结填充用户界面。如果有错误，将错误打印到控制台。

你尚未定义 populate 方法，因此 Xcode 会发出警告。在 insertSampleData() 下面添加以下实现：

```swift
func populate(bowtie: BowTie) {

  guard let imageData = bowtie.photoData as Data?,
    let lastWorn = bowtie.lastWorn as Date?,
    let tintColor = bowtie.tintColor else {
      return
  }

  imageView.image = UIImage(data: imageData)
  nameLabel.text = bowtie.name
  ratingLabel.text = "Rating: \(bowtie.rating)/5"

  timesWornLabel.text = "# times worn: \(bowtie.timesWorn)"

  let dateFormatter = DateFormatter()
  dateFormatter.dateStyle = .short
  dateFormatter.timeStyle = .none

  lastWornLabel.text =
    "Last worn: " + dateFormatter.string(from: lastWorn)

  favoriteLabel.isHidden = !bowtie.isFavorite
  view.tintColor = tintColor
}
```

领结中定义的大多数属性都有一个 UI 元素。 由于 Core Data 仅将图像存储为二进制数据块，因此你的工作是将其重新组合成图像，以便视图控制器的图像视图可以使用它。

同样，你不能直接使用 lastWorn 日期属性。 你首先需要创建一个日期格式化程序，将日期转换为人类可以理解的字符串。

最后，存储领结颜色的 tintColor 可转换属性不仅会更改屏幕上的一个元素的颜色，还会更改屏幕上所有元素的颜色。 只需在视图控制器的视图上设置色调颜色即可！ 现在一切都染上了相同的颜色。

> 注意：Xcode 生成一些 NSManagedObject 子类属性作为可选类型。 这就是为什么在 populate 方法中，你在方法开头使用 guard 语句解开 BowTie 上的一些 Core Data 属性。

构建并运行应用程序。 屏幕上出现红色领结，如下所示：

![image-20221117022623381](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221117022623381.png)

Wear 和 Rate 按钮目前没有任何作用。 点击分段控件的不同部分也没有任何作用。 你还有工作要做！

首先，你需要跟踪当前选择的领结，以便你可以在课堂上的任何地方引用它。 仍然在 ViewController.swift 中，在 managedContext 下面添加以下属性来执行此操作：

```swift
var currentBowTie: BowTie!
```

接下来，在viewDidLoad()中的do-catch语句中找到对populate(bowtie:)的调用，在其上方添加如下一行，设置currentBowTie的初始值：

```swift
currentBowTie = tie
```

跟踪当前选择的领结对于实现 Wear 和 Rate 按钮是必要的，因为这些操作只会影响当前的领结。

每次用户点击 Wear 时，按钮都会执行 wear(\_:) 操作方法。 但是 wear(\_:) 目前是空的。 将 wear(\_:) 实现替换为以下内容：

```swift
@IBAction func wear(_ sender: UIButton) {  
  currentBowTie.timesWorn += 1
  currentBowTie.lastWorn = Date()

  do {
    try managedContext.save()
    populate(bowtie: currentBowTie)    
  } catch let error as NSError {    
    print("Could not fetch \(error), \(error.userInfo)")
  }
}
```

此方法采用当前选定的领结并将其 timesWorn 属性递增 1。 接下来，将 lastWorn 日期更改为今天并保存托管对象上下文以将这些更改提交到磁盘。 最后，你填充用户界面以可视化这些更改。

构建并运行应用程序，然后根据需要多次点击 Wear。 看起来你非常喜欢红色领结的永恒优雅！

同样，每次用户点击 Rate 时，它都会执行代码中的 rate(\_:) 操作方法。 rate(\_:) 当前为空。 将 rate(\_:) 的实现替换为以下内容：

```swift
@IBAction func rate(_ sender: UIButton) {

  let alert = UIAlertController(title: "New Rating",
                                message: "Rate this bow tie",
                                preferredStyle: .alert)

  alert.addTextField { textField in
    textField.keyboardType = .decimalPad
  }

  let cancelAction = UIAlertAction(title: "Cancel",
                                   style: .cancel)

  let saveAction = UIAlertAction(
    title: "Save",
    style: .default
    ) { [unowned self] _ in
      if let textField = alert.textFields?.first {
        self.update(rating: textField.text)
      }
    }

  alert.addAction(cancelAction)
  alert.addAction(saveAction)

  present(alert, animated: true)
}
```

点击 Rate 现在会弹出一个带有单个文本字段、一个取消按钮和一个保存按钮的警报视图控制器。 点击保存按钮调用 update(rating:)，这...

糟糕，你还没有定义那个方法。 在 populate(bowtie:) 下面添加以下实现：

```swift
func update(rating: String?) {

  guard let ratingString = rating,
    let rating = Double(ratingString) else {
      return
  }

  do {
    currentBowTie.rating = rating
    try managedContext.save()
    populate(bowtie: currentBowTie)
  } catch let error as NSError {    
    print("Could not save \(error), \(error.userInfo)")
  }
}
```

你将 alert 的文本字段中的文本转换为 Double 并使用它来更新 currentBowTie rate 属性。 最后，你像往常一样通过 managedContext 来提交你的更改并刷新 UI 以实时查看你的更改。

试试看。 构建并运行应用程序并点击 Rate：

![image-20221117023518318](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221117023518318.png)

输入 0 到 5 之间的任何十进制数，然后点击保存。 如你所料，评级标签会更新为你输入的新值。 现在再点击一次评分。 还记得红色领结的永恒优雅吗？ 假设你非常喜欢它，你决定给它打 6 分（满分 5 分）。点击保存刷新用户界面：

![image-20221117023557464](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221117023557464.png)

虽然你可能绝对喜欢红色，但现在既不是夸张的时间也不是夸张的地方。 你的应用程序允许你将 6 保存为一个应该最多为 5 的值。你手上的数据无效。

### Core Data 中的数据验证

你的第一直觉可能是编写客户端验证——类似于“只有当值大于 0 且小于 5 时才保存新评级”。幸运的是，你不必自己编写这段代码。 Core Data 支持开箱即用的大多数属性类型的验证。

打开 BowTies.xcdatamodeld，选择评级属性并打开the data model inspector。

在 Validation 旁边，键入 0 表示最小值，键入 5 表示最大值。而已！无需编写任何 Swift 来拒绝无效数据。

![image-20221118021510440](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221118021510440.png)

> 注意：通常，如果你想在你的应用程序发布后更改它，你必须对你的数据模型进行版本控制。你将在第 6 节“版本控制和迁移”中了解更多相关信息。

属性验证是少数例外之一。如果你在发布后将其添加到你的应用程序中，则无需对你的数据模型进行版本控制。

但这究竟是做什么的？

在你对 ManagedObjectContext 调用 save() 后，验证立即开始。ManagedObjectContext 检查模型以查看是否有任何新值与你设置的验证规则冲突。

如果存在验证错误，则保存失败。还记得 do-catch 块中包装 save 方法的 NSError 吗？到目前为止，如果出现错误，除了将其记录到控制台之外，你没有理由做任何特殊的事情。验证改变了这一点。

再次构建并运行该应用程序。给红色领结打 6 分（满分 5 分）并保存。一个相当神秘的错误消息将显示到你的控制台：

```
Could not save Error Domain=NSCocoaErrorDomain Code=1610 "rating is too large." UserInfo={NSValidationErrorObject=<BowTie: 0x6000038c2da0> (entity: BowTie; id: 0xac775ff23f190c42 <x-coredata://67543601-368D-464C-B976-957548F4AF71/BowTie/p3>; data: {
    id = "800C3526-E83A-44AC-B718-D36934708921";
    isFavorite = 0;
    lastWorn = "2014-06-19 18:15:24 +0000";
    name = "Red Bow Tie";
    photoData = "{length = 50, bytes = 0x89504e47 0d0a1a0a 0000000d 49484452 ... aece1ce9 00000078 }";
    rating = 6;
    searchKey = R;
    timesWorn = 4;
    tintColor = "UIExtendedSRGBColorSpace 0.937255 0.188235 0.141176 1";
    url = "https://en.wikipedia.org/wiki/Bow_tie";
}), NSLocalizedDescription=rating is too large., NSValidationErrorKey=rating, NSValidationErrorValue=6}, ["NSValidationErrorObject": <BowTie: 0x6000038c2da0> (entity: BowTie; id: 0xac775ff23f190c42 <x-coredata://67543601-368D-464C-B976-957548F4AF71/BowTie/p3>; data: {
    id = "800C3526-E83A-44AC-B718-D36934708921";
    isFavorite = 0;
    lastWorn = "2014-06-19 18:15:24 +0000";
    name = "Red Bow Tie";
    photoData = "{length = 50, bytes = 0x89504e47 0d0a1a0a 0000000d 49484452 ... aece1ce9 00000078 }";
    rating = 6;
    searchKey = R;
    timesWorn = 4;
    tintColor = "UIExtendedSRGBColorSpace 0.937255 0.188235 0.141176 1";
    url = "https://en.wikipedia.org/wiki/Bow_tie";
}), "NSLocalizedDescription": rating is too large., "NSValidationErrorValue": 6, "NSValidationErrorKey": rating]
```

错误附带的 userInfo 字典包含有关 Core Data 为何中止你的保存操作的各种有用信息。 它甚至有一条本地化的错误消息，你可以在关键字 NSLocalizedDescription 下向你的用户显示：评级太大。

但是，你如何处理此错误完全取决于你。 打开 ViewController.swift 并用以下内容替换 update(rating:) 以适当地处理错误：

```swift
func update(rating: String?) {

  guard let ratingString = rating,
    let rating = Double(ratingString) else {
      return
  }

  do {

    currentBowTie.rating = rating
    try managedContext.save()
    populate(bowtie: currentBowTie)

  } catch let error as NSError {

    if error.domain == NSCocoaErrorDomain &&
      (error.code == NSValidationNumberTooLargeError ||
        error.code == NSValidationNumberTooSmallError) {
      rate(rateButton)
    } else {
      print("Could not save \(error), \(error.userInfo)")
    }

  }
}
```

如果由于新评级太大或太小而发生错误，则再次显示 alert view。

否则，你将像以前一样使用新评级填充用户界面。

但是等等…… NSValidationNumberTooLargeError 和 NSValidationNumberTooSmallError 是从哪里来的？回到之前的控制台阅读并仔细查看第一行：

```
Could not save Error Domain=NSCocoaErrorDomain Code=1610 "rating is too large."
```

NSValidationNumberTooLargeError 是一个映射到整数 1610 的错误代码。

如需 Core Data 错误和代码定义的完整列表，你可以通过 Cmd 键单击 NSValidationNumberTooLargeError 来查阅 Xcode 中的 CoreDataErrors.h。

> 注意：当涉及 NSError 时，标准做法是检查错误的域和代码以确定出了什么问题。你可以在 Apple 的错误处理编程指南中阅读更多相关信息：https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ErrorHandlingCocoa/CreateCustomizeNSError/CreateCustomizeNSError.html

构建并运行应用程序。通过再次向红色领带展示一些爱来验证新的验证规则是否正常工作。

如果你输入超过 5 的任何值并尝试保存，该应用程序会拒绝你的评分并要求你使用新的警报视图重试。 成功！

### 完成剩下的内容

Wear 和 Rate 按钮工作正常，但应用程序只能显示一条领带。 在分段控件上点击不同的值应该可以切换领带。 你将通过实现该功能来完成此示例项目。

每次用户点击分段控件时，它都会在你的代码中执行 segmentedControl(\_:) 操作方法。 将 segmentedControl(\_:) 的实现替换为以下内容：

```swift
@IBAction func segmentedControl(_ sender: UISegmentedControl) {
  guard let selectedValue = sender.titleForSegment(
    at: sender.selectedSegmentIndex) else {
      return
  }

  let request: NSFetchRequest<BowTie> = BowTie.fetchRequest()
  request.predicate = NSPredicate(
    format: "%K = %@",
    argumentArray: [#keyPath(BowTie.searchKey), selectedValue])

  do {
    let results = try managedContext.fetch(request)
    currentBowTie = results.first
    populate(bowtie: currentBowTie)
  } catch let error as NSError {
    print("Could not fetch \(error), \(error.userInfo)")
  }
}
```

分段控件中每个分段的标题对应于特定关系的 searchKey 属性。 获取当前所选片段的标题并使用精心设计的 NSPredicate 获取合适的领结。

然后，使用结果数组中的第一个领结（每个 searchKey 应该只有一个）来填充用户界面。

构建并运行应用程序。 点击分段控件上的不同字母以获得迷幻效果。

你做到了！ 有了这个应用程序，你就可以成为 Core Data 大师了。

### 关键点

* Core Data 支持不同的属性数据类型，这决定了你可以在实体中存储的数据类型以及它们将占用磁盘空间的大小。 一些常见的属性数据类型是 String、Date 和 Double。
* 二进制数据属性数据类型使你可以选择在数据模型中存储任意数量的二进制数据。
* Transformable 属性数据类型允许你在数据模型中存储任何符合 NSSecureCoding 的对象。
* 使用 NSManagedObject 子类是处理 Core Data Entity 的更好方法。 你可以手动生成子类，也可以让 Xcode 自动生成。
* 你可以使用 NSPredicate 细化 NSFetchRequest 获取的 entities。
* 你可以直接在数据模型编辑器中为大多数属性数据类型设置验证规则（例如最大值和最小值）。 如果你尝试保存无效数据，ManagedObjectContext 将抛出错误。

