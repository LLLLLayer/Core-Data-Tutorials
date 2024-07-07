# 第 10 节：NSPersistentCloudKitContainer

NSPersistentCloudKitContainer 在 WWDC 2019 上首次亮相，它是 Core Data 的便捷包装器，可为你的应用程序提供数据同步服务。 CloudKit 是 Apple 的后端即服务 (BaaS，backend as a service)，基于 iCloud 服务。

Apple 在 iOS 8 中引入了 CloudKit 以简化应用程序和设备之间的数据同步。 应用程序开发人员可以利用 iCloud 的强大功能与一组简单的 API 同步以访问数据。CloudKit 具有适用于 iOS、macOS 和 JavaScript 的 API。

然而，Core Data 对其存储数据的方式有很多要求，并且它们与 CloudKit 同步的方式不兼容。 如果开发人员想将他们应用程序的数据同步到 iCloud 以供其他设备使用，他们必须自己完成这项工作，以在 Core Data 和 CloudKit 模型之间转换用户数据。

NSPersistentCloudKitContainer 是 CloudKit 和 Core Data 之间的桥梁。 它抽象出了跨设备无缝同步数据的许多困难部分。

### 了解 CloudKit 的优势

如你所见，Core Data 是一个强大的数据持久性框架。 开始使用它并将其与你的应用程序集成起来很容易。

Core Data 不仅仅对用户在应用程序中输入的数据有用。 缓存从 Web 服务和服务器后端获取的数据也很棒。

然而，并不是每个应用程序都有一个复杂的后端来支持它。 如果你的用户想在一台设备上开始记笔记，然后在他们的计算机或 iPad 上从他们离开的地方继续，如果你的应用程序没有强大的后端，你就会遇到麻烦。 在这种情况下，Core Data 是有限的，因为它将数据保存在单个应用程序的沙箱中。

CloudKit 解决了这个问题。 它还使诸如冲突解决和身份验证之类的事情变得更加容易，因为它与 iCloud 的工作方式密切相关。 最后，它为开发人员提供了访问综合仪表板的权限，以帮助他们管理数据、模式和用户活动。

如果你的应用程序使用 Core Data 并且你需要跨设备同步数据，那么 NSPersistentCloudKitContainer 就是答案。

### 准备使用 NSPersistentCloudKitContainer

CloudKit 需要付费的 Apple Developer 帐户才能运行。 CloudKit 仪表板和相关工具不适用于个人免费帐户。

你还需要在模拟器上登录 iCloud，以使用你用于登录 Apple Developer 帐户的 Apple ID 测试设备。

如果你的测试设备是你的主要设备并且你没有登录 iCloud 的 Apple ID，请将你的常规 Apple ID 添加到你的 Apple Developer 帐户作为额外的开发者。

CloudKit 支持的 Core Data Model 有一些特定的要求。 例如，它们不支持唯一约束、未定义和 objectID 属性或拒绝删除规则。

实体之间的任何关系都必须具有反向关系。 此外，模型配置不得具有与其他配置中的实体相关的实体。

如果你的模型具有任何这些不受支持的功能，请通过创建一个新的单独模型来解决问题。 使用该新模型来存储你要同步的特定数据。

你熟悉应用程序中的数据模型，并且了解应用程序中的更改如何影响模型版本之间的迁移。 你的数据模型也存在于 iCloud 的 CloudKit 中，而不仅仅是应用程序中。

在开发过程中，你在对应用程序进行更改时将最新的架构上传到 CloudKit。 当你准备好发布你的应用程序时，你将最终架构提升为 production 状态。 模式 production 后，你永远无法重命名 CloudKit 记录名称或类型，但你可以添加字段和实体。

Apple 在 CloudKit 文档中介绍了处理模式更改管理的建议方法。

NSPersistentCloudKitContainer 严重依赖 push notification 在数据更改时通知设备。虽然模拟器用于开发，但它不会实时响应在另一个模拟器中所做的更改。 你需要一个真实的设备来进行测试，并需要一个 Apple 的付费开发者帐户来配置推送通知。

### Core Data 和 CloudKit 入门

要了解将 CloudKit 与 Core Data 一起使用有多么容易，你将创建一个快速演示应用程序。 在此过程中，你将了解如何设置 CloudKit 以及如何与 CloudKit 仪表板交互。

#### 设置你的项目

首先在 Xcode 中创建一个新项目。 在模板选择器中选择 iOS 平台和 App。

![image-20230315013604507](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230315013604507.png)

输入产品名称，然后选择团队和组织标识符。 界面选择 SwiftUI，生命周期选择 SwiftUI App。 确保选中在 CloudKit 中使用核心数据和主机。 点击下一步。 为项目选择一个目的地，然后单击“Create”。

在创建项目时选中“使用 CloudKit”复选框会将应用委托设置为使用 NSPersistentCloudKitContainer，但它不会配置任何 entitlement。 你需要自己做这件事。

在项目导航器中，单击根项目节点，然后单击应用程序的主目标。 接下来，选择 Signing & Capabilities 选项卡并单击标题中的 + Capability 按钮。 将出现一个搜索框。 搜索 iCloud 并双击它以将其添加到项目中。

![image-20230315014502083](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230315014502083.png)

在新添加的 iCloud 部分，选中服务下的 CloudKit 框，然后单击容器下的 + 号。 出现提示时，输入应用程序的完整捆绑包 ID。 在这种情况下，该值是 com.raywenderlich.CloudKitExample。

![image-20230315015042415](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230315015042415.png)

第一次添加时容器名称会是红色的，不用担心。 稍等片刻，然后单击刷新按钮——文本将变为黑色。

![image-20230315015059460](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230315015059460.png)

接下来，单击标题中的 + 功能按钮。 搜索背景模式并双击它以将其添加到项目中。

![image-20230315015140230](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230315015140230.png)

在新添加的后台模式部分中，选中 Remote notifications。

![image-20230315015309078](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230315015309078.png)

现在，打开 Persistence.swift 并在 init(inMemory:) 方法的末尾添加以下行，就在调用 loadPersistentStores 方法之后：

```swift
container.viewContext.automaticallyMergesChangesFromParent
  = true
```

通过此更改，当 CloudKit 通过互联网进行更改时，你的 UI 将立即更新。

为确保没有构建错误并确保应用程序在模拟器或设备中启动、构建和运行，然后在应用程序启动后，再次从 Xcode 停止它。

CloudKit 知道你的应用程序，因为你在上一步中创建了一个容器。 要访问它，请单击 Signing & Capabilities 中 iCloud 下的 CloudKit Dashboard 按钮。 使用 Apple Developer Apple ID 登录后，你会看到一个容器列表。 单击你之前创建的容器名称。

单击 Schema，你会注意到没有列出自定义类型。 自定义类型是 CloudKit 用来表示你的数据模型的，因此它们非常重要。

#### 添加自定义类型

接下来，你将告诉 CloudKit 你的数据模型。

无法将你的 Core Data Model 文件直接上传到 CloudKit 仪表板。 相反，你可以通过将你的应用程序启动到模拟器或设备中来通知 CloudKit 你当前的模式，该模拟器或设备已登录与你的 Apple Developer ID 关联的 iCloud 帐户。 当应用程序启动时，它将连接到 CloudKit 并上传架构。

如果你使用的是模拟器，请转到“系统偏好设置”，点击“登录你的 iPhone”并输入你的 Apple Developer Apple ID 凭据。

现在你已经登录到 iCloud，回到 Xcode。 构建并运行应用程序。 加载后，点击 + 按钮向表中添加一个新条目。 你应该会看到类似于以下内容的行条目：

![image-20230318151054660](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318151054660.png)

当你的应用程序正在运行时，你可能会看到大量调试消息。 他们中的大多数对你还没有用。 但是，如果你仔细向后滚动，你应该能够在你刚刚创建的新条目向上游发送时挑选出 CloudKit 消息。 这意味着它正在工作！

![image-20230318151157547](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318151157547.png)

请注意，名为 Event 的 Core Data Entity在进入 CloudKit 时变为 CD\_Event。 发生这种情况是因为 NSPersistentCloudKitContainer 自动为实体名称和字段名称添加前缀 CD\_。

返回 CloudKit 仪表板并重新加载页面。 单击你应用程序的容器，然后单击架构。 你会看到 CD\_Event 列为自定义类型。 哇哦！

![image-20230318151551932](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318151551932.png)

#### 查看你的数据

仪表板还使你能够查看数据，而不仅仅是模式。 单击当前选择架构的页面上的下拉菜单，然后选择数据。 你将看到可让你在 CloudKit 中查找数据的查询工具。

![image-20230318151621374](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318151621374.png)

在类型下拉列表中，找到并选择 CD\_Event（如果尚未选择）。 确保你在上面选择了 com.apple.coredata.cloudkit.zone。 你想要查看所有数据，而不是任何特定记录，因此请继续并单击查询记录。

呃哦！ 一个错误？ 别担心——你没有做错任何事。

![image-20230318151820751](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318151820751.png)

#### 修复错误

你收到此错误是因为当你首次上传架构时，没有任 Core Data 字段是可查询的，CloudKit 字段 recordName 也不是。 接下来你会改变它。

返回模式编辑器。 单击 CD\_Event Custom Type，然后单击右侧的 Edit Indexes，位于 Custom Fields 部分下方。

![image-20230318151908570](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318151908570.png)

接下来，单击添加索引。

![image-20230318152043650](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318152043650.png)

默认情况下，你会在下拉列表中看到字段 recordName 以及索引类型下的 QUERYABLE。 这正是你需要添加的索引和类型，因此单击 Save Changes。

![image-20230318152214199](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318152214199.png)

返回数据屏幕并尝试再次单击查询记录并选择 CD\_Event 类型。 你现在将看到你在模拟器中添加的一条记录。

单击 Name 下的蓝色文本，你会看到一个面板弹出，其中包含整个 CloudKit 记录。 请注意每个 Core Data 模型字段是如何列为自定义字段的。

![image-20230318152334755](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318152334755.png)

#### 编辑和同步你的记录

现在，尝试通过 CloudKit 仪表板编辑记录——你将测试 CloudKit 的神奇之处。

记录中应用程序中唯一可见的字段是 CD\_timestamp 字段。 单击该字段并将日期更改为你会记住的其他日期，然后单击保存。

> 注意：CloudKit 数据编辑器相当简单——它不会遵守应用程序 Core Data Model 或底层代码中的任何业务逻辑或格式规则。 编辑原始数据时要非常小心。 如果你引入格式错误的值，它可能会使你的应用程序崩溃或给你的用户带来问题。

![image-20230318152811029](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318152811029.png)

切换回你的模拟器。 请注意，尽管它仍在运行，但日期和时间尚未更新为你更改的值。

这是因为，如前所述，推送和后台通知在模拟器中不起作用。 在设备上，只要有新记录或编辑，CloudKit 就会向应用程序发送静默通知，但模拟器没有此功能。

你可以通过暂时将应用程序置于后台来强制模拟器使用 CloudKit 签入。 通过按 Command+Shift+H 将应用程序置于后台，然后通过点击应用程序的图标将其带回前台。 与 iCloud 同步后，日期会发生变化。

![image-20230318153031143](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318153031143.png)

魔术，对吧？

在本节中，你创建了一个小型测试应用程序来演示如何让 CloudKit 与 Core Data 一起工作。 你必须尝试使用 CloudKit 仪表板并查看你的 Core Data 模型作为 CloudKit 模式的样子。 你还编辑了一条记录以证明同步有效。

这对于全新的应用程序来说非常棒——但是如何将现有应用程序转换为使用 CloudKit？

在下一节中，你将使用即将发布的 RayWenderlich.com 应用程序 Dog Doodies 来做到这一点。

### 介绍 Dog Doodies 应用程序

当有人养了一只新狗时，他们想确保自己了解宠物的需求。 没有什么比忘记你的狗何时需要外出“履行职责”并发现屋子里有惊喜等着你更糟糕的了。

![image-20230318153633320](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318153633320.png)

这就是 Dog Doodies 的用武之地！

你可以跟踪你的爱犬的每项重要职责，查看它们的历史记录，并查看一个计时器，告诉你距离上一次完成任务还有多长时间。

在我们最初的用户测试期间，我们很快发现在单个设备上拥有历史记录是不够的。 人们希望数据存储在他们的手表、iPad 甚至计算机上。 这是 CloudKit 可以提供帮助的一个很好的例子。

在 projects\starter\DogDoodies 下找到并打开此部分的起始项目。 在 Model/DogDoodies.xcdatamodeld 打开 Core Data Model。

![image-20230318153817921](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318153817921.png)

Core Data 模型有两个实体：Pet 和 Activity。 最初，设计师希望该应用程序适用于你家中的任何宠物。 用户研究得出结论，猫更聪明，不需要（或不想）被追踪。

宠物有几个属性，最重要的是它的名字。

每个宠物都可以有零到多个 Activity 对象。 一个 Activity 有一个类型，比如 pee、poop 或 walk。 它有一个用于记录活动发生时间的字段和一个用于记录任何特殊注释的未来字段。

Activity 也有一个指向 Pet 对象的链接，这很棒，因为 NSPersistentCloudKitContainer 需要反向关系。 值得庆幸的是，数据模型中的所有内容都与 CloudKit 完全兼容，因此你无需进行太多更改即可将其集成到应用程序中。

现在，是时候将 CloudKit 添加到应用程序了。

#### 配置 CloudKit

为此项目配置 CloudKit 类似于你为简单演示应用程序所做的配置。

Xcode 应该考虑一下，然后设置 Xcode 以正确地对项目进行代码签名。

接下来，单击标题中的 + 功能按钮。 搜索 iCloud 并将其添加到项目中。

在新添加的 iCloud 部分中，选中服务下的 CloudKit 框，然后单击容器下的 + 号。

出现提示时，输入你在上一步中修改的应用程序的完整捆绑包 ID。

和以前一样，第一次添加时容器名称将是红色的。 稍等片刻，然后单击刷新按钮。 文本将变为黑色。

接下来，单击标题中的 + 功能按钮。 搜索背景模式并将其添加到项目中。 在新添加的后台模式部分中，选中远程通知框。

最后，在 Xcode 中构建并运行应用程序以确保没有配置问题。

伟大的！ 现在你已准备好转换你的应用程序！

#### 将容器转换为 CloudKit

将 Dog Doodies 转换为 CloudKit 的第一步也是唯一的一步是用 NSPersistentCloudKitContainer 替换持久容器类。

打开 CoreDataStack.swift 并将 storeContainer 属性定义替换为以下内容：

```swift
private lazy var storeContainer: NSPersistentContainer = {
  // 1
  let container = 
    NSPersistentCloudKitContainer(name: self.modelName)
  container.loadPersistentStores { _, error in
    if let error = error as NSError? {
      print("Unresolved error \(error), \(error.userInfo)")
    }
  }

  // 2
  container.viewContext.automaticallyMergesChangesFromParent
    = true

  // 3
  do {
    try container.viewContext.setQueryGenerationFrom(.current)
  } catch {
    fatalError("###\(#function): Failed to pin viewContext to the current generation:\(error)")
  }

  return container
}()
```

在上面的代码中，你：

1. 将 NSPersistentContainer 替换为 NSPersistentCloudKitContainer。 你可以在这里停下来，该应用程序将与 CloudKit 一起运行。
2. 开启了自动合并更改的选项，和之前的app一样。 在某些情况下，自动合并对于你的应用来说并不是最佳选择——例如，如果你有很多自定义验证逻辑，或者如果你有棘手的合并冲突需要通过用户干预来管理。 对于你的简单应用程序，此选项会很有效。
3. 将用于主线程上所有 UI 工作的 ManagedObjectContext 固定到来自 CloudKit 的当前一代数据。

固定 Context 可防止 UI 在用户与其交互时发生更改。 这避免了当用户尝试点击一行并且传入的更改添加另一行时导致显示错误记录的情况。

每当你保存、合并或重置上下文时，固定的生成都会自动更新。 由于你在上一步中启用了自动合并，因此固定 UI 不会产生太大的可见效果。

如果你有一个应该“暂停”更新的特定屏幕，请为该 UI 创建一个新的 context 并将其固定，但关闭它的自动合并。

#### 记录一些小玩意儿！

构建并运行应用程序。 启动后，你会看到“未选择狗”消息。

![image-20230318155226281](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318155226281.png)

点击“选择狗”按钮以调出应用程序中当前为空的狗列表。 点击 + 按钮并输入 Applesauce 作为你的狗的名字。 点击添加，然后点击表格中的行以选择该狗。

![image-20230318155253664](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318155253664.png)

假设你刚刚带着 Applesauce 去外面上厕所。 那天早上她喝了很多咖啡，所以她只需要小便。 点击水滴按钮。 你会看到时钟开始计时，这会告诉你她上次小便已经过去了多长时间。 点击“历史记录”按钮可查看有关你爱犬的更多详细信息。

![image-20230318155321737](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318155321737.png)

对于下一步，你将在不同的模拟器或实际的 iOS 设备上运行该应用程序。 在启动应用程序之前，请确保在该目标上登录相同的 Apple ID/iCloud 帐户。 配置完成后，在 Xcode 中选择该模拟器或设备，然后在保持原始模拟器运行的同时构建并运行应用程序。

请注意，当应用程序启动时，倒数计时器已经在运行。 当你点击历史记录时，你创建的活动已经列出！ 太棒了，对吧？

点击其他两个按钮之一以记录另一项活动。 在另一个模拟器中，通过按 Shift+Command+H 将应用置于后台，然后将其带回前台。 新创建的活动的计时器应该开始倒计时并与其他设备中的计时器紧密匹配。

你还可以通过在活动行上从右向左滑动，然后点击删除来删除活动。 CloudKit 将跟踪已删除的记录，以便连接到该容器的所有客户端都可以重播该操作。

处理已删除的数据特别棘手，这让许多开发人员彻夜难眠以在他们的应用程序中解决这个问题。 它与 Core Data 和 CloudKit 开箱即用的事实为你节省了很多精力。 谢谢，苹果！

#### 查看 CloudKit 数据

CloudKit 中的所有这些数据是什么样的？

单击 Xcode 中 iCloud 配置部分中的 CloudKit 仪表板按钮返回仪表板。 找到你为 Dog Doodies 应用程序创建的容器并选择它。 单击架构，然后单击编辑索引。 为 CD\_Pet 和 CD\_Activity 自定义记录类型添加 QUERYABLE 类型的 recordName 索引。 最后，在转到下一个记录类型之前单击保存更改。

切换到数据查看器并查询所有宠物记录。 单击详细信息，你将看到所有你希望看到的字段：宠物名称、动物类型和实体名称。 你可能还会注意到可见的布尔值。 这是在 Core Data 模型中，但尚未在应用程序中使用。 它在 CloudKit 中填充，因为该属性配置为具有默认值 true。

![image-20230318155513420](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318155513420.png)

现在，查询所有 Activity 记录并单击一个活动的详细信息。 你会看到有一个带有 GUID 的 CD\_Pet 引用。 这就是 CloudKit 保持引用完整性的方式，只需在其中保存相关记录的唯一 ID。

![image-20230318155614014](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230318155614014.png)

在 Activity 中有一个字段是看不到的，它是备注字段。 如果你在 Core Data 中的字段是可选字段并且你没有为其提供任何数据，则该字段可能不会显示在 CloudKit 中。

#### 了解 CloudKit 的弱点

就像生活中的大多数事情一样，如果某件事看起来太容易了，那么你可能做错了。 你在本教程中没有做错任何事情，但 CloudKit 看似简单且配置要求低。

权利和代码签名是棘手的，特别是如果你不让 Xcode 自动管理这些东西。 复杂性在于细节，而这些细节最初很容易被忽视。

使用 CloudKit 时，请记住以下几个问题。

**模型版本控制**

随着时间的推移，你的数据模型将会改变。 在大多数情况下，你会在这里或那里添加一个属性，也许是一个新实体，并创建新的关系。

Core Data 的模型版本控制和迁移框架功能强大，可以处理许多复杂的场景。 CloudKit 的一个缺点是它不直接支持 Core Data 的内置模型版本控制系统。

正如本章前面提到的，CloudKit 允许你添加字段和实体，但不能重命名这些字段或删除那些实体。 随着你的应用程序随着时间的推移而增长，如果你决定在应用程序中保留单个模型，则需要在模型的变化中考虑到这一点。

请记住，你的应用程序可能会在各种设备、操作系统版本和用户场景中运行。 当你将应用发布到 App Store 时，不要假设用户会升级他们的应用到最新版本。 如果你的生产模式随着时间的推移不断变化，旧版本的应用程序将不知道新的字段或实体。 你的用户可能还会在他们的设备上运行多种应用程序版本，他们希望这些应用程序“正常工作”并表现良好。

Apple 提供了一些解决方案，例如在每个实体的数据模型中包含一个版本号。 这使你可以在应用程序中执行条件逻辑。

如果你的应用程序很复杂，请考虑为你的 CloudKit 商店提供单独的 Core Data 配置或整个模型/堆栈。 这让你的同步数据模型更改更少，并让你定位需要同步的特定数据，而不是同步整个 Store。

这种情况当然也有缺点：你需要将数据从一个堆栈传输到另一个堆栈，并且有可能发生合并冲突，你需要忽略或让用户处理。 你的实体名称也需要不同，除非你将它们放入不同的 Swift 模块中。

关于什么适合你的应用程序，没有明确的答案——你需要发现什么最适合你的用户。

**自动合并**

我们通过在演示应用程序和 Dog Doodies 中启用自动合并来作弊。 这可以更轻松地展示事情神奇地发生的方式，但会剥夺你更加注意 UI 如何以及何时响应数据存储中的更新的能力。

当合并发生时，你的整个应用程序是否应该更新 UI？ 如果数据更改不影响当前显示的任何内容怎么办？

自动合并也会影响用户的感知性能。 你可能需要考虑让用户决定如何解决合并冲突，而不是仅仅覆盖数据。

如果你需要更加注意处理和合并传入数据，请查看 Apple 在 Core Data 中的持久历史跟踪机制。 通过打开此选项，你可以检查上下文中的传入更改，以查看它们是否与当前视图相关。 如果不是，请忽略它们。

持久的历史记录跟踪不是一个开始使用的简单功能。 该机制周围有很多边缘案例，这会使测试变得更加棘手。 这将需要本书的全新一章来涵盖它……请发送你的请求：]

**调试 CloudKit**

不可避免地，数据同步会出现问题，你需要深入研究同步机制。 在设备上调试 CloudKit 的唯一方法是打开调试日志消息并通过仔细阅读来解释正在发生的事情。

通过在目标的运行方案配置中添加 com.apple.CoreData.CloudKitDebug 启动参数，打开日志记录并定义 CloudKit 显示多少日志记录。

添加 -com.apple.CoreData.CloudKitDebug 1 作为参数可为你提供最少的调试信息。 参数名称后面的数字表示日志记录的数量，1 是最低值，3 或 4 是最高值——Apple 并不清楚上限。

**在 iCloud 上测试**

你需要在将应用发布到生产环境之前对其进行测试，这在 iCloud 上并不容易，尤其是当你不是该项目的唯一开发人员时。

一般来说，iCloud 的设置不适合与测试设备和测试场景一起使用。 Apple 开发者帐户要求你在 Apple ID 上启用双因素身份验证。 由于开发人员需要在测试设备上登录和注销帐户的方式，设置测试 Apple ID 通常会触发欺诈锁定。

### 关键点

* NSPersistentCloudKitContainer 是对 Core Data 的强大补充，可为你的应用程序提供多设备同步功能。
* CloudKit 对 Core Data 数据模型有限制，不直接支持 Core Data 模型版本控制。
* CloudKit Dashboard 具有架构和数据检查工具，可帮助调试和维护应用程序的数据。
* iOS 模拟器不支持推送通知，这意味着你必须采取额外的步骤才能看到自动合并。
* NSPersistentCloudKitContainer 很容易引入你的项目，但随着时间的推移会增加你的应用程序的复杂性。 请注意面向未来的数据模型更改，并注意性能注意事项。

### 从这往哪儿走？

Apple 关于 NSPersistentCloudKitContainer 的文档随着时间的推移而增加。 如果你想深入了解其更高级的功能，或更深入地了解 CloudKit，请查看 Apple 开发者网站上的这些页面：

将 Core Data 存储与 CloudKit 同步：https://apple.co/2Um8dkf

使用 CloudKit 设置核心数据：https://apple.co/33vqK1y

NSPersistentCloudKitContainer：https://apple.co/2QuJrgM

在 CloudKit WWDC 2019 中使用核心数据：https://apple.co/3djNruv

将 Core Data 存储与 CloudKit 公共数据库 WWDC 2020 同步：https://apple.co/30dC5mI
