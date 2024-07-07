# 第 4 节：Fetch

在本书的前三节中，你开始探索 Core Data 的基础，包括在 Core Data Persistent Store 中保存和获取数据的非常基本的方法。

到目前为止，你执行的大多是简单的、未优化的 fetch，例如“fetch 所有 BowTie Entity”。有时这就是你需要做的全部。通常，你会希望对如何从 Core Data 检索信息施加更多控制。

基于你目前所学的内容，本章将深入探讨 Fetch 主题。Fetch 是 Core Data 中的一个大主题，你可以使用许多工具。到本节结束时，你将知道如何：

* 只 fetch 你需要的
* 使用谓词(predicate)优化 fetch 的结果
* 在后台 fetch 以避免阻塞 UI
* 通过直接在 persistent store 中更新对象来避免不必要的 fetch

本章是一个 toolbox sampler；它的目的是让你接触到许多 fetch 技术，所以到时候，你就会知道使用什么工具。

### NSFetchRequest

正如你在前面学到的，你可以通过创建一个 NSFetchRequest 的实例来从 Core Data 中获取记录，根据需要配置它并将它交给 NSManagedObjectContext 来完成繁重的工作。

看起来很简单，但实际上有五种不同的方法来获取 fetch 请求。有些比其他的更受欢迎，但作为 Core Data 开发人员，你可能会在某个时候遇到所有这些。

在跳转到本章的起始项目之前，这里有五种不同的方法来设置获取请求，这样你就不会感到惊讶了：

```swift
// 1
let fetchRequest1 = NSFetchRequest<Venue>()
let entity = 
  NSEntityDescription.entity(forEntityName: "Venue",
                             in: managedContext)!
fetchRequest1.entity = entity

// 2
let fetchRequest2 = NSFetchRequest<Venue>(entityName: "Venue")

// 3
let fetchRequest3: NSFetchRequest<Venue> = Venue.fetchRequest()

// 4
let fetchRequest4 = 
  managedObjectModel.fetchRequestTemplate(forName: "venueFR")

// 5
let fetchRequest5 =
  managedObjectModel.fetchRequestFromTemplate(
    withName: "venueFR",
    substitutionVariables: ["NAME" : "Vivi Bubble Tea"])
```

依次通过每个：

1. 将 NSFetchRequest 的实例初始化为通用类型：NSFetchRequest\<Venue>。至少，你必须为获取请求指定一个 NSEntityDescription。在这种情况下，实体是 Venue。你初始化 NSEntityDescription 的一个实例，并使用它来设置获取请求的 entity 属性。
2. 这里使用了 NSFetchRequest 的便利初始化。它初始化一个新的获取请求并一步设置其 entity 属性。你只需要为实体名称提供一个字符串，而不是完整的 NSEntityDescription。
3. 正如第二个例子是第一个的缩写，第三个是第二个的缩写。当你生成 NSManagedObject 子类时，此步骤还会生成一个类方法，该方法返回一个 NSFetchRequest 已经设置为获取相应的实体类型。这就是 Venue.fetchRequest() 的来源。此代码位于 Venue+CoreDataProperties.swift 中。
4. 在第四个示例中，你从 NSManagedObjectModel 中检索你的 fetch 请求。你可以在 Xcode 的数据模型编辑器中配置和存储常用的获取请求。你将在本章后面学习如何执行此操作。
5. 最后一种情况与第四种类似。从你的 managed object model 中检索一个提取请求，但是这次，你传递了一些额外的变量。这些“替代”变量在 predicate 中用于优化你获取的结果。

前三个示例是你已经见过的简单案例。除了存储的获取请求和 NSFetchRequest 的其他技巧之外，你将在本节的其余部分看到更多这些简单的案例！

> 注意：如果你还不熟悉它，NSFetchRequest 是一个通用类型。如果你检查 NSFetchRequest 的初始化程序，你会注意到它接受类型作为参数 \<ResultType : NSFetchRequestResult>。
>
> ResultType 指定你希望作为获取请求结果的对象类型。例如，如果你需要一个 Venue 对象数组，则获取请求的结果现在将是 \[Venue] 而不是 \[Any]。这很有用，因为你不必再转为 \[Venue]。

### 介绍 BubbleTea 应用程序

本章的示例项目是一个 BubbleTea 应用程序。使用该应用程序，你可以找到附近出售你最喜欢的饮品的地点。

对于本节，你将只使用来自 Foursquare 的静态场所数据：即纽约市大约 30 个出售珍珠奶茶的地点。你将使用此数据来构建 filter/sort，以按照你认为合适的方式排列场所列表。

转到本节的文件并打开 BubbleTeaFinder.xcodeproj。构建并运行起始项目。

你会看到以下内容：

![image-20221126151258664](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126151258664.png)

示例应用程序由许多具有静态信息的表视图单元格组成。虽然示例项目目前不是很令人兴奋，但已经为你完成了很多设置。

打开项目导航器并查看启动项目中的完整文件列表：

![image-20221126151407579](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126151407579.png)

事实证明，你在本书第一部分必须完成的大部分 Core Data 设置都已准备好供你使用。以下是你在入门项目中获得的组件的快速概述，分为几类：

* Seed Data：seed.json 是一个 JSON 文件，其中包含纽约市 bubble tea 的场所的真实场所数据。由于这是来自 Foursquare 的真实数据，因此结构比本书之前使用的 Seed Data 更复杂。
*   Data Model：点击 BubbleTeaFinder.xcdatamodeld 打开 Xcode 的模型编辑器。最重要的实体是 Venue。它包含场地名称、电话号码和目前提供的特价商品数量等属性。

    由于 JSON 数据相当复杂，Data Model 将场所的信息分解为其他 Entity。它们是 Category、 Location、 PriceInfo 和 Stats。
* ManagedObject 子类：数据模型中的所有实体也有相应的 NSManagedObject 子类。它们是 Venue+CoreDataClass.swift、Location+CoreDataClass.swift、PriceInfo+CoreDataClass.swift、Category+CoreDataClass.swift 和 Stats+CoreDataClass.swift。你可以在 NSManagedObject 组中找到它们及其随附的 EntityName+CoreDataProperties.swift 文件。
* CoreDataStack：与前面的章节一样，这个对象包装了一个 NSPersistentContainer 对象，它本身包含 Core Date 对象的核心，称为“stack”；the context、the model、the persistent store、the persistent store coordinator。无需设置 - 它随时可用。
* ViewController：显示场地列表的初始视图控制器是 ViewController.swift。首次启动时，初始视图控制器从 seed.json 中读取，创建相应的 Core Data object 并将它们保存到 persistent store 中。点击右上角的 Filter 按钮会弹出 FilterViewController.swift。目前这里没有发生太多事情。在本节中，你将向这两个文件添加代码。

当你第一次启动示例应用程序时，你只会看到静态信息。但是，你的应用委托已经从 seed.json 中读取了种子数据，将其解析为核心数据对象并将它们保存到持久存储中。

你的第一个任务是获取此数据并将其显示在表视图中。

### 存储 fetch 请求

如前所述，你可以在数据模型中存储经常使用的 fetch 请求。这不仅使它们更易于访问，而且你还可以获得使用基于 GUI 的工具来设置获取请求参数的好处。

打开 BubbleTeaFinder.xcdatamodeld 并长按“Add Entity”按钮：

![image-20221126153130417](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126153130417.png)

从菜单中选择 Add Fetch Request。这将在左侧栏上创建一个新的获取请求，并将你带到一个特殊的 fetch request editor：

![image-20221126153255744](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126153255744.png)

> 注意：你可以单击左侧边栏中新创建的 FetchRequest 来更改其名称。

使用 Xcode Data Model Editor 中的可视化工具，你可以根据需要将获取请求设置为 general 或者 specific。首先，创建一个 fetch request，从持久存储中检索所有 Venue 对象。

你只需在此处进行一项更改：单击 Fetch all 旁边的下拉菜单并选择 Venue。

![image-20221126153533282](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126153533282.png)

这就是你需要做的。如果你想使用额外的 predicate 来优化你的获取请求，你还可以从 fetch request editor 中添加条件。

是时候试用一下你新创建的获取请求了。打开 ViewController.swift 并在 coreDataStack 下面添加以下两个属性：

```swift
var fetchRequest: NSFetchRequest<Venue>?
var venues: [Venue] = []
```

第一个属性将保存你的 fetchRequest。 第二个属性是将用于填充表视图的 Venue 对象数组。

接下来，将以下内容添加到 viewDidLoad() 的末尾：

```swift
guard let model = 
  coreDataStack.managedContext
    .persistentStoreCoordinator?.managedObjectModel,
  let fetchRequest = model
    .fetchRequestTemplate(forName: "FetchRequest")
    as? NSFetchRequest<Venue> else {
      return
}
self.fetchRequest = fetchRequest
fetchAndReload()
```

这样做会将你刚刚设置的 fetchRequest 属性连接到你使用 Xcode 的 data model editor 创建的属性。 这里要记住三件事：

1. 与获取 fetch 请求的其他方式不同，这种方式涉及 ManagedObjectModel。 这就是为什么你必须通过 coreDataStack 属性来检索你的 fetch 请求的原因。
2. 正如你在上一章中看到的，你构建了 CoreDataStack，因此只有 Managed Context 是公共的。要检索 ManagedObjectModel，你必须通过 ManagedContext 的 PersistentStoreCoordinator。
3. NSManagedObjectModel 的 fetchRequestTemplate(forName:) 接受一个字符串标识符。此标识符必须与你在模型编辑器中为获取请求选择的名称完全匹配。否则，你的应用程序将抛出异常并崩溃。

最后一行调用了一个你还没有定义的方法，所以 Xcode 会报错。要解决这个问题，请在 UITableViewDataSource 扩展之上添加以下扩展：

```swift
// MARK: - Helper methods
extension ViewController {

  func fetchAndReload() {

    guard let fetchRequest = fetchRequest else {
      return
    }
    
    do {
      venues =
        try coreDataStack.managedContext.fetch(fetchRequest)
      tableView.reloadData()
    } catch let error as NSError {
      print("Could not fetch \(error), \(error.userInfo)")
    }

  }
}
```

顾名思义，fetchAndReload() 执行获取请求并重新加载表视图。此类中的其他方法需要查看获取的对象，因此你将获取的结果存储在之前定义的 venues 属性中。

在运行示例项目之前，你还需要做一件事：将表视图的数据源与获取的 Venue 对象连接起来。

在 UITableViewDataSource 扩展中，将 tableView(\_:numberOfRowsInSection:) 和 tableView(\_:cellForRowAt:) 的占位符实现替换为以下内容：

```swift
func tableView(_ tableView: UITableView,
               numberOfRowsInSection section: Int) -> Int {
  venues.count
}

func tableView(_ tableView: UITableView,
               cellForRowAt indexPath: IndexPath)
               -> UITableViewCell {

  let cell =
    tableView.dequeueReusableCell(
      withIdentifier: venueCellIdentifier, for: indexPath)

  let venue = venues[indexPath.row]
  cell.textLabel?.text = venue.name
  cell.detailTextLabel?.text = venue.priceInfo?.priceCategory  
  return cell
}
```

你已经在本书中多次实现了这些方法，所以你可能熟悉它们的作用。 第一个方法，tableView(\_:numberOfRowsInSection:)，将表格视图中的单元格数量与 venues 数组中获取的对象数量相匹配。

第二种方法，tableView(\_:cellForRowAt:)，将给定索引路径的单元格出队，并使用 venues 数组中相应 Venue 的信息填充它。 在这种情况下，主标签获取场地名称，详细标签获取价格类别，该价格类别是三个可能值之一：$、\$$ 或 \$$$。

构建并运行项目，你将看到以下内容：

![image-20221126154547069](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126154547069.png)

向下滚动列表。这些都是纽约市真正出售这种美味饮品的地方。

> 注意：什么时候应该在 Data Model 中存储 FetchRequest？
>
> 如果你知道你将在应用程序的不同部分反复进行相同的 fetch，则可以使用此功能来避免多次编写相同的代码。存储 fetch request 的一个缺点是无法指定结果的排序顺序。因此，你看到的场地列表可能与截图中的顺序不同。

### fetch 不同的 ResultType

一直以来，你可能一直认为 NSFetchRequest 是一个相当简单的工具。你给它一些指令，你会得到一些东西作为回报。还有什么呢？

如果是这样的话，你就低估了这个 class。 NSFetchRequest 是 Core Data 框架的多功能瑞士军刀！

你可以使用它来获取单个值，计算数据的统计信息，例如平均值、最小值、最大值等。

你问这怎么可能？ NSFetchRequest 有一个名为 resultType 的属性。到目前为止，你只使用了默认值 .managedObjectResultType。以下是获取请求的 resultType 的所有可能值：

* .managedObjectResultType：返回 managed objects（默认值）。
* .countResultType：返回匹配 fetch request 的对象的计数。
* .dictionaryResultType：这是一个包罗万象的返回类型，用于返回不同计算的结果。
* .managedObjectIDResultType：返回唯一标识符而不是完整的 managed objects。

让我们回到示例项目并在实践中应用这些概念。

在示例项目运行的情况下，点击右上角的 Filter 以调出过滤器屏幕的 UI。

你现在不会实施实际的过滤器或排序。相反，你将关注以下四个标签：

![image-20221126162730312](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126162730312.png)

筛选分为三个部分：价格、最受欢迎和排序方式。 最后一部分在技术上不是由“filters”组成的，但排序通常与过滤器密切相关，所以你可以这样保留它。

每个价格过滤器下方是属于该价格类别的场地总数的空间。 同样，所有场所的交易总数都有一个位置。 接下来你将实现这些。

#### Return Count

打开 FilterViewController.swift 并在 import UIKit 下方添加以下内容：

```swift
import CoreData
```

接下来，在最后一个 @IBOutlet 属性下面添加以下属性：

```swift
// MARK: - Properties
var coreDataStack: CoreDataStack!
```

这将保存对你在 ViewController.swift 中使用的 CoreDataStack 对象的引用。

接下来，打开 ViewController.swift 并将 prepare(for:sender:) 实现替换为以下内容：

```swift
override func prepare(for segue: UIStoryboardSegue,
                      sender: Any?) {

  guard segue.identifier == filterViewControllerSegueIdentifier,
    let navController = segue.destination
      as? UINavigationController,
    let filterVC = navController.topViewController
      as? FilterViewController else {
        return
  }

  filterVC.coreDataStack = coreDataStack
}
```

新的代码行将 CoreDataStack 对象从 ViewController 传到 FilterViewController。

打开 FilterViewController.swift 并在 coreDataStack 下面添加以下惰性属性：

```swift
lazy var cheapVenuePredicate: NSPredicate = {
  return NSPredicate(format: "%K == %@", 
    #keyPath(Venue.priceInfo.priceCategory), "$")
}()
```

你将使用这个延迟实例化的 NSPredicate 来计算最低价格类别中的场所数量。

> 注意：NSPredicate 支持基于字符串的 #keyPath。 这就是为什么你可以使用 priceInfo.priceCategory 从 Venue entity 取到 PriceInfo entity，并使用 #keyPath 关键字来获取关键路径的安全、编译时检查值。

接下来，在 UITableViewDelegate 扩展下面添加以下扩展：

```swift
// MARK: - Helper methods
extension FilterViewController {

  func populateCheapVenueCountLabel() {

    let fetchRequest =
      NSFetchRequest<NSNumber>(entityName: "Venue")
    fetchRequest.resultType = .countResultType
    fetchRequest.predicate = cheapVenuePredicate
      
    do {
      let countResult =
        try coreDataStack.managedContext.fetch(fetchRequest)
    
      let count = countResult.first?.intValue ?? 0
      let pluralized = count == 1 ? "place" : "places"
      firstPriceCategoryLabel.text = 
        "\(count) bubble tea \(pluralized)"
    } catch let error as NSError {
      print("count not fetched \(error), \(error.userInfo)")
    }

  }
}
```

此扩展提供 populateCheapVenueCountLabel()，它创建一个获取请求以获取 Venue 实体。 然后将结果类型设置为 .countResultType 并将获取请求的谓词设置为 cheapVenuePredicate。 请注意，为了使其正常工作，获取请求的类型参数必须是 NSNumber，而不是 Venue。

当你将获取结果的结果类型设置为 .countResultType 时，返回值将变为包含单个 NSNumber 的 Swift 数组。 NSNumber 中的整数是你要查找的总数。

再一次，你针对 CoreDataStack 的 NSManagedObjectContext 属性执行获取请求。 然后从结果 NSNumber 中提取整数并使用它来填充 firstPriceCategoryLabel。

在运行示例应用程序之前，将以下内容添加到 viewDidLoad() 的底部：

```swift
populateCheapVenueCountLabel()
```

现在构建并运行以测试这些更改是否生效：

![image-20221126163800181](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126163800181.png)

第一个价格过滤器下的标签现在显示为“27”。万岁！你已经成功地使用 NSFetchRequest 来计算计数。

> 注意：你可能认为你可以轻松获取实际的 Venue 对象并从数组的 count 属性中获取计数。确实如此。获取计数而不是对象主要是一种性能优化。例如，如果你有纽约市的人口普查数据并且想知道有多少人居住在其大都市区，你更愿意 Core Data 给你数字 8,300,000（整数）还是一个包含 8,300,000 条记录的数组？
>
> 显然，直接获取计数更节省内存。有一整章专门讨论 Core Data 性能。如果你想了解更多有关 Core Data 中性能优化的信息，请查看第 8 节“衡量和提升性能”。

现在你已经熟悉了计数结果类型，你可以快速实现第二个价格类别过滤器的计数。在 cheapVenuePredicate 下面添加以下惰性属性：

```swift
lazy var moderateVenuePredicate: NSPredicate = {
  return NSPredicate(format: "%K == %@", 
    #keyPath(Venue.priceInfo.priceCategory), "$$")
}()
```

这个 NSPredicate 几乎与 cheap venue predicate 相同，除了这个匹配 \$$ 而不是 $。同样，在 populateCheapVenueCountLabel() 下面添加如下方法：

```swift
func populateModerateVenueCountLabel() {

  let fetchRequest = 
    NSFetchRequest<NSNumber>(entityName: "Venue")
  fetchRequest.resultType = .countResultType
  fetchRequest.predicate = moderateVenuePredicate

  do {
    

    let countResult = 
      try coreDataStack.managedContext.fetch(fetchRequest)
    
    let count = countResult.first?.intValue ?? 0
    let pluralized = count == 1 ? "place" : "places"
    secondPriceCategoryLabel.text = 
      "\(count) bubble tea \(pluralized)"

  } catch let error as NSError {
    print("count not fetched \(error), \(error.userInfo)")
  }
}
```

最后，将以下行添加到 viewDidLoad() 的底部以调用你新定义的方法：

```swift
populateModerateVenueCountLabel()
```

构建并运行示例项目。 和以前一样，点击右上角的 Filter：

![image-20221126164318706](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126164318706.png)

珍珠奶茶爱好者的好消息！ 只有两个地方价格适中。 珍珠奶茶作为一个整体似乎很容易获得。

#### 另一种获取计数的方法

现在你已经熟悉了 .countResultType，是时候提一下有一个替代 API 可以直接从 Core Data 中获取计数。

由于还有一个价格类别计数需要实施，你现在将使用此备用 API。

在 moderateVenuePredicate 下面添加以下惰性属性：

```swift
lazy var expensiveVenuePredicate: NSPredicate = {
  return NSPredicate(format: "%K == %@", 
    #keyPath(Venue.priceInfo.priceCategory), "$$$")
}()
```

接下来，在 populateModerateVenueCountLabel() 下面实现以下方法：

```swift
func populateExpensiveVenueCountLabel() {

  let fetchRequest: NSFetchRequest<Venue> = Venue.fetchRequest()
  fetchRequest.predicate = expensiveVenuePredicate

  do {
    let count =
      try coreDataStack.managedContext.count(for: fetchRequest)
    let pluralized = count == 1 ? "place" : "places"
    thirdPriceCategoryLabel.text = 
      "\(count) bubble tea \(pluralized)"
  } catch let error as NSError {
    print("count not fetched \(error), \(error.userInfo)")
  }
}
```

与前两个场景一样，你创建一个获取请求以检索 Venue 对象。

接下来，设置你之前定义为惰性属性的谓词：expensiveVenuePredicate。

此场景与后两个场景的区别在于，你没有将结果类型设置为 .countResultType。而不是通常的 fetch(\_:)，而是使用 NSManagedObjectContext 的方法 count(for:)。

count(for:) 的返回值是一个整数，你可以直接使用它来填充第三个价格类别标签。最后，将以下行添加到 viewDidLoad() 的底部以调用你新定义的方法：

```swift
populateExpensiveVenueCountLabel()
```

构建并运行以查看你的最新更改是否生效：

![image-20221126164626494](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126164626494.png)

### 使用 fetch 请求执行计算

所有三个价格类别标签都填充了属于每个类别的场所数量。下一步是在“提供交易”下填充标签。它目前说“0 笔交易”。那是不对的！

这些信息究竟从何而来？ Venue 有一个 specialCount 属性，用于捕获该场所当前提供的交易数量。与价格类别下的标签不同，你现在需要了解所有场馆的交易总额，因为特别精明的场馆可能会同时进行多项交易。

天真的方法是将所有场所加载到内存中，然后使用 for 循环对它们的交易求和。如果你希望有更好的方法，那么你很幸运：Core Data 内置了对许多不同函数的支持，例如 average、sum、min 和 max。

打开 FilterViewController.swift，在 populateExpensiveVenueCountLabel() 下面添加如下方法：

```swift
func populateDealsCountLabel() {

  // 1
  let fetchRequest = 
    NSFetchRequest<NSDictionary>(entityName: "Venue")
  fetchRequest.resultType = .dictionaryResultType

  // 2
  let sumExpressionDesc = NSExpressionDescription()
  sumExpressionDesc.name = "sumDeals"

  // 3
  let specialCountExp = 
    NSExpression(forKeyPath: #keyPath(Venue.specialCount))
  sumExpressionDesc.expression = 
    NSExpression(forFunction: "sum:",
                 arguments: [specialCountExp])
  sumExpressionDesc.expressionResultType =
    .integer32AttributeType

  // 4
  fetchRequest.propertiesToFetch = [sumExpressionDesc]

  // 5
  do {

    let results = 
      try coreDataStack.managedContext.fetch(fetchRequest)
    
    let resultDict = results.first
    let numDeals = resultDict?["sumDeals"] as? Int ?? 0
    let pluralized = numDeals == 1 ?  "deal" : "deals"
    numDealsLabel.text = "\(numDeals) \(pluralized)"

  } catch let error as NSError {
    print("count not fetched \(error), \(error.userInfo)")
  }
}
```

这个方法包含了一些你之前在书中没有遇到过的类，所以这里依次解释一下：

1. 你首先创建用于检索 Venue 对象的典型提取请求。接下来，你将结果类型指定为 .dictionaryResultType。
2. 你创建一个 NSExpressionDescription 来请求总和，并将其命名为 sumDeals，这样你就可以从获取请求返回的结果字典中读取它的结果。
3. 你给表达式描述一个 NSExpression 来指定你想要求和函数。接下来，为该表达式提供另一个 NSExpression 以指定要求和的属性——在本例中为 specialCount。最后，你必须设置表达式描述的返回数据类型，因此将其设置为 integer32AttributeType。
4. 通过将其 propertiesToFetch 属性设置为你刚创建的表达式描述，你告诉原始获取请求获取总和。
5. 最后，在通常的 do-catch 语句中执行 fetch 请求。结果类型是一个 NSDictionary 数组，因此你可以使用表达式描述的名称 (sumDeals) 检索表达式的结果，这样就完成了！

> 注意：Core Data 还支持哪些其他功能？仅举几例：计数、最小值、最大值、平均值、中值、众数、绝对值等等。有关完整列表，请查看 Apple 的 NSExpression 文档。

从 Core Data 获取计算值需要你遵循许多通常不直观的步骤，因此请确保你有充分的理由使用此技术，例如性能方面的考虑。最后，将以下行添加到 viewDidLoad() 的底部：

```swift
populateDealsCountLabel()
```

构建示例项目并打开筛选/排序屏幕以验证你的更改：

![image-20221126165746284](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126165746284.png)

你现在已经使用了四种受支持的 NSFetchRequest 结果类型中的三种：.managedObjectResultType、.countResultType 和 .dictionaryResultType。

剩下的结果类型是 .managedObjectIDResultType。当你使用这种类型获取时，结果是一个 NSManagedObjectID 对象数组，而不是它们所代表的实际托管对象。 NSManagedObjectID 是托管对象的紧凑通用标识符。它就像数据库中的主键一样工作！

在 iOS 5 之前，通过 ID 获取很流行，因为 NSManagedObjectID 是线程安全的，使用它可以帮助开发人员实现线程限制并发模型。

现在线程限制已被弃用，取而代之的是更现代的并发模型，几乎没有理由再按对象 ID 获取了。

> 注意：你可以设置多个 ManagedObjectContext 来运行并发操作，并使长时间运行的操作远离主线程。有关详细信息，请查看第 9 节“多个 ManagedObjectContext”。

你已经体验了 fetch 请求可以为你做的所有事情。但与 fetch 请求返回的信息一样重要的是它没有返回的信息。出于实际原因，你必须在某个时候限制传入数据。

为什么？想象一个完美连接的对象图，其中每个 Core Data 对象都通过一系列关系连接到每个其他对象。如果 Core Data 没有对 fetch 请求返回的信息进行限制，那么你每次都会获取整个对象图！那不是内存有效的。

你可以手动限制从获取请求中获取的信息。例如，NSFetchRequest 支持批量获取。你可以使用属性 fetchBatchSize、fetchLimit 和 fetchOffset 来控制批处理行为。

Core Data 还尝试通过使用称为 faulting 的技术来为你最小化其内存消耗。fault 是一个占位符对象，表示尚未完全进入内存的 managed object。

另一种限制对象图的方法是使用 predicate，就像你在上面填充场地计数标签时所做的那样。让我们使用 predicate 将 Filter 添加到示例应用程序。

打开 FilterViewController.swift，并在类定义上方添加以下协议声明：

```swift
protocol FilterViewControllerDelegate: class {
  func filterViewController(
    filter: FilterViewController,
    didSelectPredicate predicate: NSPredicate?,
    sortDescriptor: NSSortDescriptor?)
}
```

该协议定义了一个委托方法，该方法将在用户选择新的排序/过滤组合时通知委托。

接下来，在 coreDataStack 下面添加以下三个属性：

```swift
weak var delegate: FilterViewControllerDelegate?
var selectedSortDescriptor: NSSortDescriptor?
var selectedPredicate: NSPredicate?
```

第一个属性将保存对 FilterViewController 委托的引用。 为了避免保留周期，它是一个 weak 属性。 第二个和第三个属性将分别保存对当前选择的 NSSortDescriptor 和 NSPredicate 的引用。

接下来，实现 search(\_:) 如下所示：

```swift
@IBAction func search(_ sender: UIBarButtonItem) {
  delegate?.filterViewController(
    filter: self,
    didSelectPredicate: selectedPredicate,
    sortDescriptor: selectedSortDescriptor)

  dismiss(animated: true)
}
```

这意味着每次你点右上角的搜索时，你都会通知 delegate 你的选择并关闭列表。

你需要在此文件中再做一项更改。 找到 tableView(\_:didSelectRowAt:) 并实现如下所示：

```swift
override func tableView(_ tableView: UITableView,
                        didSelectRowAt indexPath: IndexPath) {

  guard let cell = tableView.cellForRow(at: indexPath) else {
    return
  }

  // Price section
  switch cell {
  case cheapVenueCell:
    selectedPredicate = cheapVenuePredicate
  case moderateVenueCell:
    selectedPredicate = moderateVenuePredicate
  case expensiveVenueCell:
    selectedPredicate = expensiveVenuePredicate
  default: break
  }

  cell.accessoryType = .checkmark
}
```

当用户点击前三个价格类别单元格中的任何一个时，此方法会将所选单元格映射到适当的 predicate。 你将对此 predicate 的引用存储在 selectedPredicate 中，以便在你通知代理用户选择时准备就绪。

接下来，打开 ViewController.swift 并添加以下扩展以符合 FilterViewControllerDelegate 协议：

```swift
// MARK: - FilterViewControllerDelegate
extension ViewController: FilterViewControllerDelegate {

  func filterViewController(
    filter: FilterViewController,
    didSelectPredicate predicate: NSPredicate?,
    sortDescriptor: NSSortDescriptor?) {

    guard let fetchRequest = fetchRequest else {
      return
    }
    
    fetchRequest.predicate = nil
    fetchRequest.sortDescriptors = nil
    
    fetchRequest.predicate = predicate
    
    if let sort = sortDescriptor {
      fetchRequest.sortDescriptors = [sort]
    }
    
    fetchAndReload()

  }
}
```

添加 FilterViewControllerDelegate Swift 扩展告诉编译器这个类将符合这个协议。每次用户选择新的过滤器/排序组合时都会触发此委托方法。

在这里，你重置获取请求的谓词和排序描述符，然后设置传递给方法的谓词和排序描述符并重新加载数据。

在测试价格类别过滤器之前，你还需要做一件事。找到 prepare(for:sender:) 并将以下行添加到方法的末尾：

```swift
filterVC.delegate = self
```

这将 ViewController 设置为 FilterViewController 的委托。

构建并运行示例项目。转到过滤器屏幕，点击第一个价格类别单元格（$），然后点击右上角的搜索。

你的应用程序崩溃并在控制台中显示以下错误消息：

```
*** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'Can't modify a named fetch request in an immutable model.'
```

发生了什么？在本章的前面，你在数据模型中定义了获取请求。事实证明，如果你使用该技术，获取请求将变得不可变。你不能在运行时改变它的 predicate，否则你会崩溃。如果你想以任何方式修改获取请求，你必须提前在数据模型编辑器中进行。

打开 ViewController.swift，并将 viewDidLoad() 替换为以下内容：

```swift
override func viewDidLoad() {
  super.viewDidLoad()

  importJSONSeedDataIfNeeded()

  fetchRequest = Venue.fetchRequest()
  fetchAndReload()
}
```

你删除了从 managed object model 中的模板检索提取请求的行。 相反，你直接从 Venue 实体获取 NSFetchRequest 的实例。

再次构建并运行示例应用程序。 转到过滤器屏幕，点击第二个价格类别单元格（\$$），然后点击右上角的搜索。

这是结果：

![image-20221126185336623](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126185336623.png)

不出所料，该类别中只有两个 venue。你将练习为其余过滤器编写更多 predicates。 该过程与你已经完成的过程类似，因此这次你将进行较少的解释。

打开 FilterViewController.swift 并在 expensiveVenuePredicate 下面添加这三个 lazy 属性：

```swift
lazy var offeringDealPredicate: NSPredicate = {
  return NSPredicate(format: "%K > 0",
    #keyPath(Venue.specialCount))
}()

lazy var walkingDistancePredicate: NSPredicate = {
  return NSPredicate(format: "%K < 500",
    #keyPath(Venue.location.distance))
}()

lazy var hasUserTipsPredicate: NSPredicate = {
  return NSPredicate(format: "%K > 0",
    #keyPath(Venue.stats.tipCount))
}()
```

第一个 predicate 指定当前提供一个或多个 venue，第二个 predicate 指定距离你当前位置不到 500 米的场所，第三个 predicate 指定至少有一个用户提示的 venue。

> 注意：到目前为止，在本书中，你已经编写了具有单一条件的 predicate。你还应该知道，你可以使用复合 predicate 运算符（如 AND、OR 和 NOT）编写检查两个条件而不是一个条件的 predicate。
>
> 或者，你可以使用类 NSCompoundPredicate 将两个简单 predicate 串成一个 compound predicate。
>
> NSPredicate 在技术上不是 Core Data 的一部分（它是 Foundation 的一部分）所以本书不会深入介绍它，但是你可以通过学习这个漂亮课程的来龙去脉来认真提高你的 Core Data 技能。有关更多信息，请务必查看 Apple 的谓词编程指南：
>
> https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Predicates/Articles/pUsing.html

接下来，向下滚动到 tableView(\_:didSelectRowAt:)。你将向之前添加的 switch 语句中再添加三种情况：

```swift
override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {

  guard let cell = tableView.cellForRow(at: indexPath) else {
    return
  }

  switch cell {
  // Price section
  case cheapVenueCell:
    selectedPredicate = cheapVenuePredicate
  case moderateVenueCell:
    selectedPredicate = moderateVenuePredicate
  case expensiveVenueCell:
    selectedPredicate = expensiveVenuePredicate
    
  // Most Popular section
  case offeringDealCell:
    selectedPredicate = offeringDealPredicate
  case walkingDistanceCell:
    selectedPredicate = walkingDistancePredicate
  case userTipsCell:
    selectedPredicate = hasUserTipsPredicate
  default: break
  }

  cell.accessoryType = .checkmark
}
```

在上面，你添加了 offeringDealCell、walkingDistanceCell 和 userTipsCell 的案例。 这些是你现在为其添加支持的三个新过滤器。

这就是你需要做的。 构建并运行示例应用程序。 转到过滤器页面，选择提供交易过滤器并点击搜索：

![image-20221126190312434](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126190312434.png)

你会看到总共六个场地。 请注意，由于你没有指定排序描述符，你的场所列表可能与屏幕截图中的场所顺序不同。

### 对 fetch 的结果进行排序

NSFetchRequest 的另一个强大功能是它能够为你对获取的结果进行排序。它通过使用另一个方便的基础类 NSSortDescriptor 来做到这一点。这些排序发生在 SQLite 级别，而不是内存中。这使得 Core Data 中的排序变得快速高效。

在本节中，你将实现四种不同的排序来完成过滤/排序屏幕。

打开 FilterViewController.swift 并在 hasUserTipsPredicate 下面添加以下三个惰性属性：

```swift
lazy var nameSortDescriptor: NSSortDescriptor = {
  let compareSelector =
    #selector(NSString.localizedStandardCompare(_:))
  return NSSortDescriptor(key: #keyPath(Venue.name),
                          ascending: true,
                          selector: compareSelector)
}()

lazy var distanceSortDescriptor: NSSortDescriptor = {
  return NSSortDescriptor(
    key: #keyPath(Venue.location.distance),
    ascending: true)
}()

lazy var priceSortDescriptor: NSSortDescriptor = {
  return NSSortDescriptor(
    key: #keyPath(Venue.priceInfo.priceCategory),
    ascending: true)
}()
```

添加 sort descriptor 的方式与添加过滤器的方式非常相似。每个排序描述符都映射到这三个 lazy NSSortDescriptor 属性之一。

要初始化 NSSortDescriptor 的实例，你需要三样东西：指定要排序的属性的 key path、排序是升序还是降序的说明以及执行比较操作的可选 selector。

> 注意：如果你以前使用过 NSSortDescriptor，那么你可能知道有一个基于 block 的 API，它使用比较器而不是选择器。不幸的是，Core Data 不支持这种定义排序描述符的方法。
>
> 同样的事情也适用于定义 NSPredicate 的基于 block 的方法。 Core Data 也不支持这个。原因是过滤和排序发生在 SQLite 数据库中，因此 predicate/sort descriptor 必须很好地匹配可以编写为 SQL 语句的内容。

三个排序描述符将分别按名称、距离和价格类别升序排序。在继续之前，请仔细查看第一个排序描述符 nameSortDescriptor。初始化器接受一个可选的选择器，NSString.localizedStandardCompare(\_:)。那是什么？

每当你对面向用户的字符串进行排序时，Apple 建议你传入 NSString.localizedStandardCompare(\_:) 以根据当前语言环境的语言规则进行排序。这意味着 sort 将“正常工作”并为具有特殊字符的语言做正确的事情。

接下来，找到 tableView(\_:didSelectRowAt:) 并将以下 case 添加到 default case 上方的 switch 语句的末尾：

```swift
// Sort By section
case nameAZSortCell:
  selectedSortDescriptor = nameSortDescriptor
case nameZASortCell:
  selectedSortDescriptor =
    nameSortDescriptor.reversedSortDescriptor
    as? NSSortDescriptor
case distanceSortCell:
  selectedSortDescriptor = distanceSortDescriptor
case priceSortCell:
  selectedSortDescriptor = priceSortDescriptor
```

与之前一样，此 switch 语句将用户点击的单元格与适当的排序描述符相匹配，因此当用户点击搜索时它已准备好传递给委托。

唯一的问题是 nameZA sort descriptor。无需创建单独的 sort descriptor，你可以重用 A-Z 的排序描述符并简单地调用 reversedSortDescriptor 方法。多么方便！

其他一切都连接起来供你测试你刚刚实施的种类。构建并运行示例应用程序。点击名称 (Z-A) 排序，然后点击搜索。你会看到这样排序的搜索结果：

![image-20221126192503569](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126192503569.png)

当你向下滚动表格视图时，你会看到该应用程序确实按字母顺序从 Z 到 A 对场地进行了排序。

你现在已经完成了过滤器，设置它以便用户可以将任何一个过滤器与任何一种类型组合在一起。尝试不同的组合，看看你得到了什么。 venue cell显示的信息不多，所以如果需要验证排序，可以直接上源码查阅 seed.json。

### 异步 fetch

如果你已经达到这一点，那么既有好消息也有坏消息（然后是更多好消息）。好消息是你已经学到了很多关于你可以用普通的 NSFetchRequest 做什么的知识。坏消息是，到目前为止，你执行的每个获取请求都在等待结果返回时阻塞了主线程。

当你阻塞主线程时，它会使屏幕对传入的触摸没有响应并产生一系列其他问题。你没有感觉到主线程的这种阻塞，因为你发出了一次获取几个对象的简单获取请求。

从 Core Data 开始，该框架就为开发人员提供了多种在后台执行提取的技术。从 iOS 8 开始，Core Data 有一个 API，用于在后台执行长时间运行的提取请求，并在提取完成时获得完成回调。

让我们看看这个新 API 的实际应用。打开 ViewController.swift 并在 venues 下面添加以下属性：

```swift
var asyncFetchRequest: NSAsynchronousFetchRequest<Venue>?
```

你有它。负责这种异步魔法的类被恰当地称为 NSAsynchronousFetchRequest。不过，不要被它的名字所迷惑。它与 NSFetchRequest 没有直接关系；它实际上是 NSPersistentStoreRequest 的子类。

接下来，将 viewDidLoad() 的内容替换为以下内容：

```swift
override func viewDidLoad() {
  super.viewDidLoad()

  importJSONSeedDataIfNeeded()

  // 1
  let venueFetchRequest: NSFetchRequest<Venue> = 
    Venue.fetchRequest()
  fetchRequest = venueFetchRequest

  // 2
  asyncFetchRequest =
    NSAsynchronousFetchRequest<Venue>(
    fetchRequest: venueFetchRequest) {
      [unowned self] (result: NSAsynchronousFetchResult) in

      guard let venues = result.finalResult else {
        return
      }
    
      self.venues = venues
      self.tableView.reloadData()

  }

  // 3
  do {
    guard let asyncFetchRequest = asyncFetchRequest else {
      return
    }
    try coreDataStack.managedContext.execute(asyncFetchRequest)
    // Returns immediately, cancel here if you want
  } catch let error as NSError {
    print("Could not fetch \(error), \(error.userInfo)")
  }
}
```

有很多你以前没见过的，所以让我们一步一步地介绍它：

1. 请注意，异步获取请求不会取代常规 fetch 请求。相反，你可以将异步获取请求视为你已有的 fetch 请求的 wrapper。
2. 要创建 NSAsynchronousFetchRequest，你需要两件事：一个普通的旧 NSFetchRequest 和一个 completion handler。你获取的地点包含在 NSAsynchronousFetchResult 的 finalResult 属性中。在完成处理程序中，你更新 venues 属性并重新加载表视图。
3. 指定完成处理程序是不够的！你仍然必须执行异步获取请求。 CoreDataStack 的 managedContext 属性再次为你处理繁重的工作。但是，请注意你使用的方法不同——这次是 execute(\_:) 而不是通常的 fetch(\_:)。

execute(\_:) 立即返回。你不需要对返回值做任何事情，因为你将从 completion block 中更新表视图。返回类型是 NSAsynchronousFetchResult。

> 注意：作为此 API 的额外好处，你可以使用 NSAsynchronousFetchResult 的 cancel() 方法取消获取请求。

是时候看看你的异步提取是否按承诺交付了。如果一切顺利，你应该不会注意到用户界面有任何差异。

构建并运行示例应用程序，你应该会像以前一样看到场地列表：

![image-20221126194510012](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20221126194510012.png)

万岁！你已经掌握了异步抓取。过滤器和排序也将起作用，除了它们仍然使用普通的 NSFetchRequest 来重新加载表视图。

### Batch updates：无 fetch 请求

有时，你从 Core Data 获取 objiect 的唯一原因是更改单个属性。然后，在你进行更改后，你必须将 Core Data Object 提交回 Persistent Store 并收工。这是你一直遵循的正常流程。

但是如果你想一次更新十万条记录怎么办？获取所有这些对象只是为了更新一个属性将花费大量时间和大量内存。

幸运的是，从 iOS 8 开始，有一种新的方法可以更新 Core Data objects，而无需将任何内容提取到内存中：batch updates。这种新技术大大减少了进行这些大量更新所需的时间和内存量。

新技术绕过 NSManagedObjectContext 并直接进入 Persistent Store。batch updates 的经典用例是消息传递应用程序或电子邮件客户端中的“全部标记为已读”功能。对于这个示例应用程序，你将做一些更有趣的事情。由于你非常喜欢珍珠奶茶，因此你会将 Core Data 中的每个地点标记为你的最爱。

让我们在实践中看看。打开 ViewController.swift 并将以下内容添加到 importJSONSeedDataIfNeeded() 调用下方的 viewDidLoad() 中：

```swift
let batchUpdate = NSBatchUpdateRequest(entityName: "Venue")
batchUpdate.propertiesToUpdate = 
  [#keyPath(Venue.favorite): true]

batchUpdate.affectedStores = 
  coreDataStack.managedContext
    .persistentStoreCoordinator?.persistentStores

batchUpdate.resultType = .updatedObjectsCountResultType

do {
  let batchResult = 
    try coreDataStack.managedContext.execute(batchUpdate)
      as? NSBatchUpdateResult
  print("Records updated \(String(describing: batchResult?.result))")
} catch let error as NSError {
  print("Could not update \(error), \(error.userInfo)")
}
```

你使用要更新的实体创建 NSBatchUpdateRequest 实例，在本例中为 Venue。

接下来，通过将 propertiesToUpdate 设置为一个字典来设置批量更新请求，该字典包含你要更新的属性的键路径 favorite 及其新值 true。然后将 affectedStores 设置为持久存储协调器的 persistentStores 数组。

最后，你将结果类型返回一个计数并执行你的批量更新请求。

构建并运行你的示例应用程序。如果一切正常，你将在控制台日志中看到以下内容：

```
Records updated Optional(30)
```

伟大的！ 你已经偷偷将纽约市的每家珍珠奶茶店标记为你的最爱。

现在你知道了如何在不将它们加载到内存的情况下更新你的 Core Data 对象。 是否还有另一个用例，你可能希望绕过 ManagedContext 并直接在持久存储中更改核心数据对象？

当然有——batch deletion！

你不必为了删除对象而将对象加载到内存中，尤其是在处理大量对象时。 从 iOS 9 开始，你有 NSBatchDeleteRequest 用于此目的。

顾名思义，批量删除请求可以高效地一次性删除大量 Core Data 对象。

与 NSBatchUpdateRequest 一样，NSBatchDeleteRequest 也是 NSPersistentStoreRequest 的子类。 这两种批处理请求的行为相似，因为它们都直接在 persistent store 上操作。

> **注意**：由于你回避了 NSManagedObjectContext，因此如果你使用批量更新请求或批量删除请求，你将不会获得任何验证。你的更改也不会反映在你的 managed context。
>
> 在使用持久存储请求之前，请确保你正确地清理和验证了你的数据！

### 关键点

* NSFetchRequest 是一个**通用类型**。它采用一个类型参数，该参数指定你希望作为 fetch 请求的结果获得的对象类型。
* 如果你希望在应用程序的不同部分重复使用相同类型的提取，请考虑使用 Data Model Editor 将 不可变 fetch 请求直接存储在你的数据模型中。
* 使用 NSFetchRequest 的 **count** 结果类型来有效地计算和返回来自 SQLite 的计数。
* 使用 NSFetchRequest 的 **dictionary** 结果类型从 SQLite 中有效地计算和返回平均值、总和和其他常见计算。
* 获取请求使用不同的技术，例如使用 **batch sizes**, **batch limits** 和**faulting**来限制返回的信息量。
* 在你的抓取请求中添加 **sort description**，以高效地对 fetch 的结果进行排序。
* 获取大量信息会阻塞主线程。使用 NSAsynchronousFetchRequest 将部分工作卸载到后台线程。
* NSBatchUpdateRequest 和 NSBatchDeleteRequest 减少了更新或删除 Core Data 中大量记录所需的时间和内存量。
