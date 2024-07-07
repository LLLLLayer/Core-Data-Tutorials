# 第 7 节：单元测试

单元测试是将项目分解为小的、可测试的软件部分或单元的过程。 与其测试“点击按钮时应用程序创建新记录”场景，不如将其分解为测试更小的操作，例如按钮触摸事件、创建实体以及测试保存是否成功。

在本章中，你将学习如何使用 Xcode 中的 XCTest 框架来测试你的 Core Data 应用程序。 单元测试 Core Data 应用程序并不像它应该的那样简单，因为大多数测试将依赖于有效的 Core Data Stack。 你可能不希望单元测试中的大量测试数据干扰你在模拟器或设备上完成的手动测试，因此你将学习如何将测试数据分开。

为什么要关心对应用程序进行单元测试？ 原因有很多：

* 你可以在很早的阶段就改变你的应用程序的架构和行为。 你可以测试应用程序的大部分功能，而无需担心 UI。
* 你将在本章中很好地介绍 XCTest，但你应该已经对它有一个基本的了解才能从本章中获得最大收益。
* 你可以防止由多个开发人员组成的团队相互争吵，因为每个开发人员都可以独立于其他人进行更改并进行测试。
* 你可以节省测试时间。 无需点击三个不同的屏幕并在字段中输入测试数据，你可以对应用程序代码的任何部分运行一个小测试，而不是通过 UI 手动操作。
* 你将在本章中很好地介绍 XCTest，但你应该已经对它有一个基本的了解才能从本章中获得最大收益。

有关更多信息，请查看 Apple 文档 (https://developer.apple.com/documentation/xctest)、我们的 iOS 单元测试和 UI 测试教程 (https://www.raywenderlich.com/960290-ios-unit-testing-and-ui-testing-tutorial），或我们的 iOS 测试驱动开发教程一书，它深入探讨了单元测试和编写可测试代码。

### 入门

你将在本章中使用的示例项目 CampgroundManager 是一个预订系统，用于跟踪露营地、每个地点的便利设施和露营者本身。

该应用程序正在进行中。 基本概念：小型露营地可以使用此应用程序来管理他们的露营地和预订，包括时间表和付款。 用户界面非常基础； 它具有功能性，但没有提供太多价值。 没关系——在本教程中，你永远不会构建和运行应用程序！

该应用程序的业务逻辑已分解为小块。 你将编写单元测试来帮助设计。 当你开发单元测试并充实业务逻辑时，很容易看出用户界面还有哪些工作要做。

业务逻辑按主题分为三个不同的类。 一种用于露营地，一种用于露营者，一种用于预订。 所有类都有后缀 Service，你的测试将集中在这些服务类上。

#### 访问控制

默认情况下，Swift 中的类具有“internal”访问级别。 这意味着你只能从它们自己的模块中访问它们。 由于应用程序和测试位于不同的目标和不同的模块中，你通常无法在测试中从应用程序访问类。

解决此问题的方法有以下三种：

1. 你可以将你的应用程序中的类和方法标记为 public 的，以使它们在测试中可见（或打开以允许子类化）。
2. 你可以将类添加到 File Inspector 中的测试目标，以便它们将在测试中编译并可从测试中访问。
3. 你可以在单元测试中的任何导入前面添加 Swift 关键字@testable，以访问类中导入的所有内容。

在 CampgroundManager 示例项目中，app 目标中的必要类和方法已标记为 public 或 open。 这意味着你只需要从测试中 import CampgroundManager，你就可以访问任何你需要的东西。

> 注意：使用 @testable 将是最简单的方法，但它在语言中的存在有些值得商榷。 理论上，只有公共方法才应该进行单元测试； 任何不公开的东西都是不可测试的，因为没有公共接口或合同。 使用 @testable 绝对比盲目地将 public 添加到所有类和函数更容易被接受。

### 用于测试的 Core Data stack

由于你将测试应用程序的 Core Data 部分，因此首要任务是设置 Core Data stack 以进行测试。

好的单元测试遵循首字母缩写词 FIRST：

* Fast：如果你的测试运行时间太长，你将不会费心运行它们。
* Isolated：任何测试在单独运行时或在任何其他测试之前或之后运行时都应该正常运行。
* Repeatable：每次针对相同的代码库运行测试时，你都应该得到相同的结果。
* Self-verifying：测试本身应该报告成功或失败； 你不必检查文件或控制台日志的内容。
* Timely：在你已经编写代码之后编写测试有一些好处，特别是如果你正在编写新测试来覆盖新错误。 不过，理想情况下，测试首先作为你正在开发的功能的规范。

执行单元测试时，应用程序会启动，测试会在正在运行的应用程序的环境中运行。 实际上，如果应用程序的状态受到正在运行的测试的影响，反之亦然，这可能会导致问题。 CampgroundManager 已设置为允许单元测试执行覆盖 AppDelegate。 这可以防止应用程序干扰单元测试。 查看 main.swift 和 TestingAppDelegate.swift 了解更多细节。

CampgroundManager 使用 Core Data 将数据存储在磁盘上的数据库文件中。这听起来不是很孤立，因为来自一个测试的数据可能会被写入数据库并可能影响其他测试。 这听起来也不太可重复，因为每次运行测试时数据都会在数据库文件中累积。 你可以在运行每个测试之前手动删除并重新创建数据库文件，但这不会很快。

解决方案是一个修改过的 Core Data Stack，它使用内存中的 SQLite 存储，而不是一个由磁盘上的文件支持的存储。 这将很快并且每次都提供一个干净的状态。

你在本书的大部分内容中使用的 CoreDataStack 默认为磁盘上的 SQLite 存储。 当你使用 CoreDataStack 进行测试时，你希望它改为使用内存存储。

首先，创建一个子类化 CoreDataStack 的新类，以便你可以更改存储。

1. 右击 CampgroundManagerTests 组下的 Services，点击New File。
2. 在 iOS ▸ Source 下选择 Swift File。 点击下一步。
3. 将文件命名为 TestCoreDataStack.swift。 确保仅选择了 CampgroundManagerTests Target。
4. 单击创建。
5. 如果提示添加 Objective-C 桥接标头，请选择不创建。

最后，用以下内容替换文件的内容：

```swift
import CampgroundManager
import Foundation
import CoreData

class TestCoreDataStack: CoreDataStack {
  override init() {
    super.init()

    let container = NSPersistentContainer(
      name: CoreDataStack.modelName,
      managedObjectModel: CoreDataStack.model)
    container.persistentStoreDescriptions[0].url =
      URL(fileURLWithPath: "/dev/null")
    
    container.loadPersistentStores { _, error in
      if let error = error as NSError? {
        fatalError(
          "Unresolved error \(error), \(error.userInfo)")
      }
    }
    
    self.storeContainer = container

  }
}
```

这个类是 CoreDataStack 的子类，并且只覆盖单个属性的默认值：storeContainer。 由于你要覆盖 init() 中的值，因此不会使用来自 CoreDataStack 的持久容器，甚至不会对其进行实例化。 TestCoreDataStack 中的持久容器使用 /dev/null 的文件位置，这是空设备。 这是一个特殊的文件系统位置，所有数据都被丢弃，这导致 SQLite 创建一个内存存储而不是磁盘上的存储。 你可以实例化堆栈并在测试中写入尽可能多的数据。 当测试结束时——内存中的存储会自动清除。 堆栈就位后，就该创建你的第一个测试了！

> 注意：Core Data 也有一个内存存储类型 NSInMemoryStoreType，你可以将其用于测试，但是 SQLite 存储和内存存储在幕后的操作方式之间存在根本差异。 使用内存支持的 SQLite 存储可以最接近应用程序的真实行为。

如果你的测试需要大量数据，内存中的 SQLite 存储可能不是最佳方法。 在这些情况下，请遵循上述方法，但使用特定于测试的 URL 来创建仅用于测试的磁盘存储。 然后，你的 tearDown() 方法将关闭测试存储并删除每个测试的文件。

#### 你的第一个测试

当你将应用程序设计为小模块的集合时，单元测试最有效。你不必将所有业务逻辑都放入一个巨大的视图控制器中，而是创建一个（或多个）类来封装该逻辑。

在大多数情况下，你可能会向部分完成的应用程序添加单元测试。 对于 CampgroundManager，已经创建了 CamperService、CampSiteService 和 ReservationService 类，但它们的功能还不完整。 你将首先测试最简单的类 CamperService。

首先创建一个新的测试类：

1. 右击 CampgroundManagerTests 组下的 Services 组，点击New File。
2. 选择 iOS ▸ Source ▸ Unit Test Case Class。 点击下一步。
3. 将类命名为 CamperServiceTests； 应该已经选择了 XCTestCase 的子类。 选择 Swift 作为语言，然后单击 Next。
4. 确保 CampgroundManagerTests 目标复选框是唯一选中的目标。 单击创建。

在 CamperServiceTests.swift 中，添加以下导入语句：

```swift
import CampgroundManager
import CoreData
```

接下来，将以下两个属性添加到类中：

```swift
// MARK: Properties
var camperService: CamperService!
var coreDataStack: CoreDataStack!
```

这些属性将保存对被测 CamperService 实例和 CoreDataStack 的引用。 这些属性是隐式展开的可选值，因为它们将在 setUp() 而不是 init() 中初始化。

接下来，将所有已删除的方法替换为以下内容：

```swift
override func setUp() {
  super.setUp()

  coreDataStack = TestCoreDataStack()
  camperService = CamperService(
    managedObjectContext: coreDataStack.mainContext,
    coreDataStack: coreDataStack)
}
```

setUp 在每次测试运行之前被调用。 这是你创建课程中所有单元测试所需的任何资源的机会。 在本例中，你初始化 camperService 和 coreDataStack 属性。

每次测试后重置数据是明智的——你的测试是独立的和可重复的，还记得吗？使用内存存储并在 setUp() 中创建新上下文可以为你完成此重置。

请注意，CoreDataStack 实例实际上是一个 TestCoreDataStack 实例。 CamperService 初始化方法采用它需要的上下文以及 CoreDataStack 的实例，因为上下文保存方法是该类的一部分。 你还可以使用 setUp() 将标准测试数据插入到上下文中以备后用。

接下来，直接在 setUp() 下面添加以下实现：

```swift
override func tearDown() {
  super.tearDown()

  camperService = nil
  coreDataStack = nil
}
```

tearDown() 与 setUp() 相反，在每次测试执行后调用。 在这里，该方法将简单地使所有属性为 nil，并在每次测试后重置 CoreDataStack。

此时 CamperService 上只有一个方法：addCamper(\_:phonenumber:)。 仍然在 CamperServiceTests.swift 中，在 tearDown() 下面添加以下方法来测试 addCamper：

```swift
func testAddCamper() {
  let camper = camperService.addCamper(
    "Bacon Lover",
    phoneNumber: "910-543-9000")

  XCTAssertNotNil(camper, "Camper should not be nil")
  XCTAssertTrue(camper?.fullName == "Bacon Lover")
  XCTAssertTrue(camper?.phoneNumber == "910-543-9000")
}
```

你创建具有特定属性的 camper，然后检查以确认存在具有你期望的属性的 camper。

这是一个简单的测试，但它确保如果 addCamper 内部的任何逻辑被修改，基本操作不会改变。 例如，如果你添加一些新的数据验证逻辑以防止喜欢培根的人预订露营地，addCamper 可能会返回 nil。 然后该测试将失败，提醒你你在验证中犯了错误或需要更新测试。

> 注意：为了在真实的开发环境中完善这个测试用例，你需要为奇怪的场景编写单元测试，例如 nil 参数、空参数或重复的 camper names。

通过单击“Product”菜单，然后选择“Test”（或键入 Command+U）来运行单元测试。 你应该会在 Xcode 中看到一个绿色的复选标记。

![image-20230311180115025](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230311180115025.png)

这是你的第一个测试！ 这种类型的测试对于利用你的数据模型和检查属性是否正确存储很有用。

还要注意测试如何作为使用 API 的人的微型文档。 它是如何调用 addCamper 并描述预期行为的示例。 在这种情况下，该方法应该返回一个有效的对象而不是 nil。

请注意，此测试创建了对象并检查了属性，但没有将任何内容保存到存储中。 这个项目使用一个单独的队列上下文，所以它可以在后台持久化数据。 然而，测试直接通过； 这意味着你无法使用 XCTAssert 检查保存结果，因为你无法确定后台操作何时完成。 保存数据是 Core Data 的重要组成部分——那么你如何测试应用程序的这一部分呢？

### 异步测试

在 Core Data 中使用单个 ManagedObjectContext 时，一切都在主 UI 线程上运行。 然而，创建 background context 是一种常见的模式，它们是 main context 的 children，用于在不阻塞 UI 的情况下完成工作。

在给定上下文的正确线程上执行工作很容易：你只需将工作包装在 performBlockAndWait() 或 performBlock() 中，以确保它在与上下文关联的线程上执行。 performBlockAndWait() 将在继续之前等待完成块的执行，而 performBlock() 将立即返回并在上下文中排队执行。

测试 performBlock() 执行可能很棘手，因为你需要某种方式从块内部向外界发送有关测试状态的信号。 幸运的是，XCTestCase 中有一个称为期望的特性可以帮助解决这个问题。

下面的示例显示了如何使用期望在完成测试之前等待异步方法完成：

```swift
let expectation = expectation(withDescription: "Done!")

someService.callMethodWithCompletionHandler() {
  expectation.fulfill()
}

waitForExpectations(timeout: 2.0, handler: nil)
```

这里的关键是某些东西必须满足或触发 expectation，以便测试继续进行。 最后的 wait 方法采用一个时间参数（以秒为单位），因此测试不会永远等待，并且可能会超时（并失败），以防期望永远不会实现。

在提供的示例中， fulfill() 在传递给测试方法的完成处理程序中被显式调用。 使用 Core Data 保存操作，更容易监听 NSManagedObjectContextDidSave 通知，因为它发生在一个你不能显式调用 fulfill() 的地方。

向 CamperServiceTests.swift 添加一个新方法，以测试在添加新 camper 时是否保存了根上下文：

```swift
func testRootContextIsSavedAfterAddingCamper() {
  //1
  let derivedContext = coreDataStack.newDerivedContext()
  camperService = CamperService(
    managedObjectContext: derivedContext,
    coreDataStack: coreDataStack)

  //2
  expectation(
    forNotification: .NSManagedObjectContextDidSave,
    object: coreDataStack.mainContext) { _ in
      return true
  }

  //3
  derivedContext.perform {
    let camper = self.camperService.addCamper(
      "Bacon Lover",
      phoneNumber: "910-543-9000")
    XCTAssertNotNil(camper)
  }

  //4
  waitForExpectations(timeout: 2.0) { error in
    XCTAssertNil(error, "Save did not occur")
  }
}
```

这是代码的细分：

1. 对于这个测试，你创建一个 background context 工作。 使用此 context 而不是 main context 重新创建 CamperService 实例。
2. 你创建链接到通知的 text expectation。 在这种情况下，期望链接到来自 Core Data Stack 的 root context 的 NSManagedObjectContextDidSave 通知。 通知的处理程序很简单：它返回 true，因为你只关心通知是否已触发。
3. 添加 camper，与之前完全相同，但这次是在 derived context 的执行块内，因为这是一个后台上下文，需要在其自己的线程上运行操作。
4. 测试最多等待两秒钟的期望值。 如果出现错误或超时，处理程序块的错误参数将包含一个值。

运行单元测试，你应该会在这个新方法旁边看到一个绿色的复选标记。 将 UI 阻塞操作（例如 Core Data 保存操作）保持在主线程之外非常重要，这样你的应用程序才能保持响应。

测试 expectation 对于确保单元测试涵盖这些异步操作非常重要。

你已经为应用程序中的现有功能添加了测试； 现在是时候添加一些功能并自己测试了。 或者为了更有趣——也许先写测试？

### 第一个测试

CampgroundManager 的一个重要功能是它能够为 camper 预留场地。 在接受预订之前，系统必须了解露营地的所有露营地。 CampSiteService 的创建是为了帮助添加、删除和查找露营地。

打开 CampSiteService.swift，你会注意到唯一实现的方法是 addCampSite。 这个方法没有单元测试，所以为服务创建一个测试用例：

1. 右击CampgroundManagerTests 组下的 Services，点击New File。
2. 选择 iOS\Source\Unit Test Case Class。 点击下一步。
3. 将类命名为 CampSiteServiceTests； 应该已经选择了 XCTestCase 的子类。 选择 Swift 作为语言，然后单击 Next。
4. 确保 CampgroundManagerTests 目标复选框是唯一选中的目标。 单击创建。

用以下内容替换文件的内容：

```swift
import UIKit
import XCTest
import CampgroundManager
import CoreData

class CampSiteServiceTests: XCTestCase {

  // MARK: Properties
  var campSiteService: CampSiteService!
  var coreDataStack: CoreDataStack!

  override func setUp() {
    super.setUp()

    coreDataStack = TestCoreDataStack()
    campSiteService = CampSiteService(
      managedObjectContext: coreDataStack.mainContext,
      coreDataStack: coreDataStack)

  }

  override func tearDown() {
    super.tearDown()

    campSiteService = nil
    coreDataStack = nil

  }
}
```

这看起来与之前的测试类非常相似。 随着你的测试套件的扩展并且你注意到常见或重复的代码，你可以重构你的测试以及你的应用程序代码。 你可以放心地这样做，因为如果你搞砸了任何事情，单元测试将会失败！

添加以下新方法来测试添加 campsite。 这看起来和工作起来都像测试新 camper 创建的方法：

```swift
func testAddCampSite() {
  let campSite = campSiteService.addCampSite(
    1,
    electricity: true,
    water: true)

  XCTAssertTrue(
    campSite.siteNumber == 1,
    "Site number should be 1")
  XCTAssertTrue(
    campSite.electricity!.boolValue,
    "Site should have electricity")
  XCTAssertTrue(
    campSite.water!.boolValue,
    "Site should have water")
}
```

为确保在此方法期间保存 context，请将以下内容添加到测试中：

```swift
func testRootContextIsSavedAfterAddingCampsite() {
  let derivedContext = coreDataStack.newDerivedContext()

  campSiteService = CampSiteService(
    managedObjectContext: derivedContext,
    coreDataStack: coreDataStack)

  expectation(
    forNotification: .NSManagedObjectContextDidSave,
    object: coreDataStack.mainContext) { _ in
      return true
  }

  derivedContext.perform {
    let campSite = self.campSiteService.addCampSite(
      1,
      electricity: true,
      water: true)
    XCTAssertNotNil(campSite)
  }

  waitForExpectations(timeout: 2.0) { error in
    XCTAssertNil(error, "Save did not occur")
  }
}
```

此方法对于你之前创建的方法应该也很熟悉。 运行单元测试； 一切都应该过去。 在这一点上，你应该感到有点偏执。 如果测试被破坏并且它们总是通过怎么办？ 是时候进行一些测试驱动的开发，并获得将红色测试变成绿色测试的嗡嗡声了！

> 注意：测试驱动开发 (TDD) 是一种开发应用程序的方法，首先编写测试，然后逐步实现该功能，直到测试通过。 然后为下一个功能或改进重构代码。 涵盖 TDD 方法超出了本章的范围，但如果你决定遵循 TDD，那么此处介绍的步骤将帮助你使用 TDD。

将以下方法添加到 CampSiteServiceTests.swift 的末尾以测试 getCampSite()：

```swift
func testGetCampSiteWithMatchingSiteNumber() {
  _ = campSiteService.addCampSite(
    1,
    electricity: true,
    water: true)

  let campSite = campSiteService.getCampSite(1)
  XCTAssertNotNil(campSite, "A campsite should be returned")
}

func testGetCampSiteNoMatchingSiteNumber() {
  _ = campSiteService.addCampSite(
    1,
    electricity: true,
    water: true)

  let campSite = campSiteService.getCampSite(2)
  XCTAssertNil(campSite, "No campsite should be returned")
}
```

两个测试都使用 addCampSite 方法来创建新的 CampSite。 你从之前的测试中知道此方法有效，因此无需再次测试。 实际测试包括通过 ID 检索 CampSite 并测试结果是否为 nil。

想想用空数据库开始每个测试有多可靠。 如果你不使用内存存储，则很容易有一个露营地与第二次测试的 ID 匹配，然后就会失败！

运行单元测试。 期望 CampSite 的测试失败，因为你还没有实现 getCampSite。

另一个单元测试——不需要站点的单元测试——通过了。 这是一个误报的例子，因为该方法总是返回 nil。 为每个方法添加针对多个场景的测试以尽可能多地执行代码路径非常重要。

使用以下代码在 CampSiteService.swift 中实现 getCampSite：

```swift
public func getCampSite(_ siteNumber: NSNumber) -> CampSite? {
  let fetchRequest: NSFetchRequest<CampSite> =
    CampSite.fetchRequest()
  fetchRequest.predicate = NSPredicate(
    format: "%K = %@",
    argumentArray: [#keyPath(CampSite.siteNumber), siteNumber])

  let results: [CampSite]?
  do {
    results = try managedObjectContext.fetch(fetchRequest)
  } catch {
    return nil
  }
  return results?.first
}
```

现在重新运行单元测试，你应该会看到绿色的复选标记。 啊，成功的甜蜜满足！

注意：本书捆绑资源中包含的本章最终项目包括涵盖每种方法的多个场景的单元测试。 你可以浏览该代码以获取更多示例。

### 验证和重构

ReservationService 将包含一些相当复杂的逻辑来确定露营者是否能够预订站点。 ReservationService 的单元测试将要求目前创建的每个服务都测试其操作。

像以前一样创建一个新的测试类：

1. 右击CampgroundManagerTests组下的Services，点击New File。
2. 选择 iOS ▸ Source ▸ Unit Test Case Class。 点击下一步。
3. 将类命名为 ReservationServiceTests； 应该已经选择了 XCTestCase 的子类。 选择 Swift 作为语言。 点击下一步。
4. 确保 CampgroundManagerTests 目标复选框是唯一选中的目标。 单击创建。

用以下内容替换文件的内容：

```swift
import Foundation
import CoreData
import XCTest
import CampgroundManager

class ReservationServiceTests: XCTestCase {

  // MARK: Properties
  var campSiteService: CampSiteService!
  var camperService: CamperService!
  var reservationService: ReservationService!
  var coreDataStack: CoreDataStack!

  override func setUp() {
    super.setUp()
    coreDataStack = TestCoreDataStack()
    camperService = CamperService(
      managedObjectContext: coreDataStack.mainContext,
      coreDataStack: coreDataStack)
    campSiteService = CampSiteService(
      managedObjectContext: coreDataStack.mainContext,
      coreDataStack: coreDataStack)
    reservationService = ReservationService(
      managedObjectContext: coreDataStack.mainContext,
      coreDataStack: coreDataStack)
  }

  override func tearDown() {
    super.tearDown()

    camperService = nil
    campSiteService = nil
    reservationService = nil
    coreDataStack = nil

  }
}
```

这是你在之前的测试用例类中使用的设置和拆卸代码的稍长版本。 除了像往常一样设置 Core Data Stack 外，你还为每个测试在 setUp 中创建了每个服务的新实例。

添加以下方法来测试创建预订：

```swift
func testReserveCampSitePositiveNumberOfDays() {
  let camper = camperService.addCamper(
    "Johnny Appleseed",
    phoneNumber: "408-555-1234")!
  let campSite = campSiteService.addCampSite(
    15,
    electricity: false,
    water: false)

  let result = reservationService.reserveCampSite(
    campSite,
    camper: camper,
    date: Date(),
    numberOfNights: 5)

  XCTAssertNotNil(
    result.reservation,
    "Reservation should not be nil")
  XCTAssertNil(
    result.error,
    "No error should be present")
  XCTAssertTrue(
    result.reservation?.status == "Reserved",
    "Status should be Reserved")
}
```

单元测试创建一个 camper 和 campsite，两者都需要保留一个地点。 这里的新部分是你使用预订服务来预订露营地，将露营者和露营地与日期联系在一起。

单元测试验证 Reservation 对象是否已创建并且 NSError 对象不在返回的元组中。 查看 reserveCampSite 调用，你可能已经意识到过夜数应该至少大于零。 添加以下单元测试以测试该条件：

```swift
func testReserveCampSiteNegativeNumberOfDays() {
  let camper = camperService.addCamper(
    "Johnny Appleseed",
    phoneNumber: "408-555-1234")!
  let campSite = campSiteService.addCampSite(
    15,
    electricity: false,
    water: false)

  let result = reservationService!.reserveCampSite(
    campSite,
    camper: camper,
    date: Date(),
    numberOfNights: -1)

  XCTAssertNotNil(
    result.reservation,
    "Reservation should not be nil")
  XCTAssertNotNil(
    result.error,
    "An error should be present")
  XCTAssertTrue(
    result.error?.userInfo["Problem"] as? String == "Invalid number of days",
    "Error problem should be present")
  XCTAssertTrue(
    result.reservation?.status == "Invalid",
    "Status should be Invalid")
}
```

运行单元测试，你会发现测试失败了。 显然，编写 ReservationService 的人并没有考虑检查这个！ 在它进入世界之前，你在测试中发现了这个错误是一件好事——也许预订负数的夜晚会导致退款！

测试是探测系统和发现其行为漏洞的好地方。 该测试也作为准规范； 测试表明你仍然期待一个有效的、非零的结果，但是一个错误条件设置。

首先，打开 ReservationService.swift，找到 reserveCampSite(\_:camper:\date:numberOfNights:) 并替换：

```swift
reservation.status = "Reserved"
```

为：

```swift
if numberOfNights <= 0 {
  reservation.status = "Invalid"
  registrationError = NSError(
    domain: "CampingManager",
    code: 5,
    userInfo: ["Problem": "Invalid number of days"])
} else {
  reservation.status = "Reserved"
}
```

最后，通过将 let 替换为 var，将 registrationError 从常量更改为变量。

现在重新运行测试并检查负天数测试是否通过。 当你想要添加额外的功能或验证规则时，你可以看到流程如何继续进行重构。

无论你了解所测试代码的详细信息还是将其视为黑盒，你都可以针对 API 编写这些类型的测试，以查看其行为是否符合你的预期。 如果是这样，太好了！ 这意味着测试将确保你的代码按预期运行。 如果不是，你要么需要更改测试以匹配代码，要么更改代码以匹配测试。

### 关键点

* 单元测试应遵循 FIRST 原则：Fast, Isolated, Repeatable, Self-verifying, and Timely.
* 创建一个特定于单元测试的 Persistent Store，并在每次测试时重置其内容。使用内存中的 SQLite 存储是最简单的方法。
* Core Data 可以异步使用，并且很容易用 XCTestExpectation 类进行测试。

### 接下来去哪？

你可能已经多次听说，对你的工作进行单元测试是维护稳定软件产品的关键。 虽然 Core Data 可以帮助从你的项目中消除大量容易出错的持久性代码，但如果使用不当，它可能会成为逻辑错误的来源。

编写可以使用 Core Data 的单元测试将有助于在你的代码到达你的用户之前稳定你的代码。 XCTestExpectation 是一个简单但功能强大的工具，可帮助你以异步方式测试核心数据。 明智地使用它！

作为挑战，CampSiteService 有许多尚未实现的方法，标有 TODO 注释。 使用 TDD 方法，编写单元测试，然后实现使测试通过的方法。 如果你遇到困难，请查看本章资源中包含的挑战项目以获取示例解决方案。
