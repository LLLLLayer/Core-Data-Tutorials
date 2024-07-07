# 第 6 节：版本控制和迁移

你已经了解了如何在 Core Data 应用程序中设计 Data Model 和 NSManagedObject 子类。 在应用程序开发期间，在发布日期之前，彻底的测试可以帮助确定 Data Model。 然而，应用发布后应用使用、设计或功能的变化将不可避免地导致 Data Model 的变化。 那你怎么办呢？

你无法预测未来，但有了 Core Data，你可以随着应用程序的每个新版本进行迁移。迁移过程将更新使用以前版本的 Data Model 创建的 Data 以匹配当前 Data Model。

本节通过带你了解 note-taking 应用程序数据模型的演变，讨论 Core Data 迁移的许多方面。

你将从一个简单的应用程序开始，其 data model 中只有一个 entity。 随着你向应用程序添加更多功能和数据，你在本节中执行的迁移将变得越来越复杂。

### 何时迁移

什么时候需要进行迁移？ 需要更改 Data Model 时。

但是，在某些情况下你可以避免迁移。 如果应用仅将 Core Data 用作离线缓存，那么当你更新应用时，你可以简单地删除并重建数据存储。 这只有在用户数据的真实来源不在 data store 中时才有可能。 在所有其他情况下，你需要保护用户的数据。

也就是说，任何时候如果不更改 Data Store 就无法实现设计更改或功能请求，你将需要创建新版本的 Data Store 并提供迁移路径。

### 迁移过程

初始化 Core Data Stack 时，涉及的步骤之一是将 Store 添加到 Persistent Store Coordinator。 当你遇到这个步骤时，Core Data 在将 Store 添加到 Coordinator 之前会做一些事情。 首先，Core Data 分析 Store 的 Model 版本。 接下来，它将此版本与 Coordinator 配置的 Data Model 进行比较。 如果 Store 的 Model 版本和 Coordinator 的 Model 版本不匹配，Core Data 将在启用时执行迁移。

> 注意：如果未启用迁移，并且 Store 与 Model 不兼容，Core Data 不会将 Store 附加到 Coordinator，并指定带有适当原因代码的错误。

要开始迁移过程，Core Data 需要原始 Data Model 和目标 Data Model。 它使用这两个版本来加载或创建用于迁移的映射 Model，用于将原始 Store 中的 Data 转换为可以存储在新 Store 中的 Data。 一旦 Core Data 确定了映射 Model，迁移过程就可以正式开始了。

迁移分三步进行：

1. 首先，Core Data 将所有对象从一个 Data Store 复制到下一个。
2. 接下来，Core Data 根据关系映射将所有对象 Relationship Map 关联起来。
3. 最后，在目标 Model 中执行任何 Data 验证。 Core Data 在数据复制期间禁用目标 Model 验证。

你可能会问，“如果出现问题，原始源 Data Store 会发生什么情况？” 对于几乎所有类型的 Core Data 迁移，原始 store 都不会发生任何变化，除非迁移无误地完成。 只有迁移成功后，Core Data 才会删除原来的 Data Store。

### 迁移类型

除了 Apple 提出的 lightweight 和 heavyweight 迁移之间的简单区别之外，还有更多的迁移变体。这里提供了更微妙的迁移名称变体，但这些名称不是官方类别。 你将从最不复杂的迁移形式开始，并以最复杂的形式结束。

**Lightweight 迁移**

Lightweight 迁移是 Apple 的术语，指的是你参与的工作量最少的迁移。 当你使用 NSPersistentContainer 时，这会自动发生，或者你必须在构建自己的 Core Data Stack 时设置一些标志。 你可以更改 Data Model 的程度有一些限制，但由于启用此选项所需的工作量很小，因此它是理想的设置。

**Manual 迁移**

Manual 迁移需要你做更多的工作。 你需要指定如何将旧数据集映射到新数据集，但你可以配置更明确的映射 model 文件。 在 Xcode 中设置映射模型与设置数据模型非常相似，具有类似的 GUI 工具和一些自动化功能。

**Custom manual 迁移**

这是迁移复杂性指数的第 3 级。 你仍将使用映射模型，但使用自定义代码对其进行补充以指定数据的自定义转换逻辑。 自定义实体转换逻辑涉及创建 NSEntityMigrationPolicy 子类并在那里执行自定义转换。

**Fully manual 迁移**

完全手动迁移适用于即使指定自定义转换逻辑也不足以将数据从一个模型版本完全迁移到另一个模型版本的时候。 自定义版本检测逻辑和迁移过程的自定义处理是必要的。 在本章中，你将设置一个完全手动的迁移来跨非顺序版本更新数据，例如从版本 1 跳到版本 4。

在本章中，你将了解每种迁移类型以及何时使用它们。 让我们开始吧！

### 入门

本书的资源中包含一个名为 UnCloudNotes 的入门项目。 找到起始项目并在 Xcode 中打开它。

在 iPhone 模拟器中构建并运行应用程序。 你会看到一个空的笔记列表：

![image-20230225150719112](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225150719112.png)

点击右上角的加号 (+) 按钮添加新 Note。 添加 Title，并点击创建以将新 Note 保存到数据存储中。 重复此操作几次，以便你有一些要迁移的样本数据。

返回 Xcode，打开 UnCloudNotesDatamodel.xcdatamodeld 文件以显示 Xcode 中的实体建模工具。 数据模型很简单——只有一个 Eneity：Note，带有一些属性。

![image-20230225151701295](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225151701295.png)

你将向该应用程序添加一项新功能：将照片附加到便笺的功能。 数据模型没有任何位置来保存此类信息，因此你需要在数据模型中添加一个位置来保存照片。 但是你已经在应用程序中添加了一些测试笔记。 如何在不破坏现有 Note 的情况下更改模型？是时候进行第一次迁移了！

### 轻量级迁移

在 Xcode 中，如果你还没有选择 UnCloudNotes Data Model 文件。 这将在主工作区中显示 Entity Modeler。 接下来，打开 Editor 菜单并选择 Add Model Version...。将新版本命名为 UnCloudNotesDataModel v2 并确保在 Based on model 字段中选择 UnCloudNotesDataModel。 Xcode 现在将创建数据模型的副本。

> 注意：你可以给这个文件任何你想要的名字。 连续的 v2、v3、v4 等命名可帮助你轻松区分版本。

此步骤将创建第二个版本的数据模型，但你仍然需要告诉 Xcode 使用新版本作为当前模型。 如果你忘记了这一步，选择顶级 UnCloudNotesDataModel.xcdatamodeld 文件将执行你对原始模型文件所做的任何更改。 你可以通过选择一个单独的模型版本来覆盖此行为，但确保你不会意外修改原始文件仍然是一个好主意。

为了执行任何迁移，你希望保持原始模型文件不变，并对全新的模型文件进行更改。

在右侧的“File Inspector”窗格中，底部有一个名为“模型版本”的选择菜单。

更改该选择以匹配新数据模型的名称 UnCloudNotesDataModel v2。

![image-20230225152424701](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225152424701.png)

完成更改后，请注意项目导航器中的绿色小复选标记图标已从以前的数据模型移动到 v2 DataModel：

![image-20230225152439500](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225152439500.png)

在设置堆栈时，Core Data 将首先尝试将 Persistent Store 与勾选的模型版本连接起来。 如果找到一个 Store File，并且它与该 Model Dile 不兼容，则会触发迁移。旧版本在这里支持迁移。 当前 Model 是 Core Data 将确保在附加 stack 的其余部分供你使用之前加载的模型。

确保选择了 v2 数据模型并将图像属性添加到 Note Entity。 将属性的名称设置为 image，将属性的类型设置为 Transformable。

由于此属性将包含图像的实际二进制位，因此你将使用自定义 NSValueTransformer 将二进制位转换为 UIImage 并再次转换回来。 ImageTransformer 中就为你提供了这样一个转换器。 在屏幕右侧的 Data Model Inspector 中，查找 Value Transformer 字段，然后输入 ImageTransformer。 接下来，在 Module 字段中，选择 Current Product Module。

![image-20230225155647315](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225155647315.png)

> 注意：当从模型文件中引用代码时，就像在 Xib 和 Storyboard 文件中一样，你需要指定一个模块（UnCloudNotes 或 Current Product Module，具体取决于你的下拉菜单提供的内容）以允许类加载器找到确切的代码。

新模型现在可以编写一些代码了！ 打开 Note.swift 并在 displayIndex 下方添加以下属性：

```swift
@NSManaged var image: UIImage?
```

构建并运行应用程序。你会看到你的 Note 仍然神奇地显示出来！ 事实证明，轻量级迁移是默认启用的。 这意味着每次你创建新的数据模型版本时，它都会自动迁移。 多么节省时间！

#### 推断映射模型

当你在 NSPersistentStoreDescription 上启用 shouldInferMappingModelAutomatically 标志时，Core Data 可以在许多情况下推断映射模型。Core Data 可以自动查看两个数据模型的差异，并在它们之间创建映射模型。

对于模型版本之间相同的 Entity 和 Attributes，这是一种直接的数据传递映射。对于其他更改，只需遵循 Core Data 的一些简单规则即可创建映射模型。

在新模型中，更改必须符合明显的迁移模式，例如：

* 删除 entities、attributes 或 relationships
* 使用 renamingIdentifier 重命名实体、属性或关系
* 添加一个新的可选属性
* 添加具有默认值的新的必需属性
* 将可选属性更改为非可选属性并指定默认值
* 将非可选属性更改为可选
* 更改实体层次结构
* 添加新的父实体并在层次结构中向上或向下移动属性
* 将关系从一对一变为一对多
* 将关系从非有序到多更改为有序到多（反之亦然）

> 注意：查看 Apple 的文档以获取有关 Core Data 如何推断轻量级迁移映射的更多信息：https://developer.apple.com/documentation/coredata/using\_lightweight\_migration。

正如你从这个列表中看到的，Core Data 可以检测，更重要的是，自动对数据模型之间的各种常见变化做出反应。

根据经验，所有迁移（如有必要）都应从轻量级迁移开始，只有在需要时才转向更复杂的映射。

至于从 UnCloudNotes 到 UnCloudNotes v2 的迁移，image 属性的默认值为 nil，因为它是一个可选属性。 这意味着 Core Data 可以轻松地将旧数据存储迁移到新数据存储，因为此更改遵循轻量级迁移模式列表中的第 3 项。

#### 图片附件

现在数据已迁移，你需要更新 UI 以允许将图像附件添加到新笔记中。 幸运的是，大部分工作已经为你完成。

打开 Main.storyboard 并找到 Create Note 场景。 在下方，你将看到包含用于附加图像的界面的“使用图像创建笔记”场景。

Create Note 场景附加到具有根视图控制器关系的导航控制器。 按住 Control 从导航控制器拖动到 Create Note With Images 场景并选择根视图控制器关系 segue。

![image-20230225161218098](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225161218098.png)

这将断开旧的 Create Note 场景并连接新的图像驱动的场景。

接下来，打开 AttachPhotoViewController.swift 并将以下方法添加到 UIImagePickerControllerDelegate 扩展：

```swift
func imagePickerController(
  _ picker: UIImagePickerController,
  didFinishPickingMediaWithInfo info:
  [UIImagePickerController.InfoKey: Any]
) {
  guard let note = note else { return }

  note.image =
    info[UIImagePickerController.InfoKey.originalImage]
    as? UIImage

  _ = navigationController?.popViewController(
    animated: true)
}
```

一旦用户从标准图像选择器中选择图像，这将填充笔记的新图像属性。

接下来，打开 CreateNoteViewController.swift 并将 viewDidAppear(\_:) 替换为以下内容：

```swift
override func viewDidAppear(_ animated: Bool) {
  super.viewDidAppear(animated)

  guard let image = note?.image else {
    titleField.becomeFirstResponder()
    return
  }

  attachedPhoto.image = image
  view.endEditing(true)
}
```

如果用户在笔记中添加了一个图像，这将显示新图像。

接下来，打开 NotesListViewController.swift 并使用以下内容更新 tableView(\_:cellForRowAt):：

```swift
override func tableView(
  _ tableView: UITableView,
  cellForRowAt indexPath: IndexPath
) -> UITableViewCell {
  let note = notes.object(at: indexPath)
  let cell: NoteTableViewCell
  if note.image == nil {
    cell = tableView.dequeueReusableCell(
      withIdentifier: "NoteCell",
      for: indexPath
    ) as! NoteTableViewCell
  } else {
    cell = tableView.dequeueReusableCell(
      withIdentifier: "NoteCellWithImage",
      for: indexPath
    ) as! NoteImageTableViewCell
  }

  cell.note = note
  return cell
}
```

这将根据是否存在图像的注释使正确的 UITableViewCell 子类出列。 最后，打开 NoteImageTableViewCell.swift 并将以下内容添加到 updateNoteInfo(note:) 的末尾：

```swift
noteImage.image = note.image
```

这将使用笔记中的图像更新 NoteImageTableViewCell 中的 UIImageView。 构建并运行，并选择添加一个新的 note ：

![image-20230225161931021](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225161931021.png)

点击 Attach Image 按钮将图像添加到 note 中。 从模拟照片库中选择一张图片，你将在新笔记中看到它：

![image-20230225162424438](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225162424438.png)

该应用程序使用标准的 UIImagePickerController 将照片作为附件添加到笔记中。

> 注意：要将你自己的图像添加到模拟器的相册中，请将图像文件拖到打开的模拟器窗口中。 值得庆幸的是，iOS 模拟器附带了一个可供你使用的照片库。

如果你使用的是设备，请打开 AttachPhotoViewController.swift 并将图像选择器控制器上的 sourceType 属性设置为 .camera 以使用设备相机拍照。 现有代码使用相册，因为模拟器中没有相机。

添加一些带照片的示例 Note ，因为在下一节中，你将使用示例数据继续进行稍微复杂的迁移。

如果你使用的是设备，请打开 AttachPhotoViewController.swift 并将图像选择器控制器上的 sourceType 属性设置为 .camera 以使用设备相机拍照。 现有代码使用相册，因为模拟器中没有相机。

添加一些带照片的示例笔记，因为在下一节中，你将使用示例数据继续进行稍微复杂的迁移。

注意：此时，你可能希望将 v2 源代码复制到另一个文件夹中，以便稍后返回。 或者，如果你使用的是源代码管理，请在此处设置一个标记，以便你可以回到这一点。 你可能还想保存数据存储文件的副本，并为此版本的应用程序的名称附加“v2”，因为稍后你将使用它进行更复杂的迁移。

恭喜； 你已成功迁移数据并基于迁移的数据添加了新功能。

### 手动迁移

该数据模型的下一步发展是从将单个图像附加到便条上转变为附加多个图像。 Note Entity 将保留，你需要一个新的图像实体。 因为一张便条可以有很多张图片，所以会有一对多的关系。

将一个实体一分为二并不完全在轻量级迁移可以支持的事情列表中。 是时候升级到自定义手动迁移了！

每次迁移的第一步都是创建一个新的模型版本。 和以前一样，选择 UnCloudNotesDataModel.xcdatamodeld 文件并从 Editor 菜单项中选择 Add Model Version...。将此模型命名为 UnCloudNotesDataModel v3 并将其基于 v2 数据模型。 使用文件检查器窗格中的选项将新模型版本设置为默认模型。

接下来，你将向新数据模型添加一个新 entity。 在左下角，单击“添加实体”按钮。 重命名为 Attachment。 选择 entity 并在 Data Model Inspector 中，将类名称设置为 Attachment，将模块设置为 Current Product Module。

![image-20230225164143455](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225164143455.png)

在附件实体中创建两个属性。 添加一个名为 image 的非可选属性，类型为 Transformable，其中 Custom Class transformer 字段设置为 ImageTransformer，Module 字段设置为 Current Product Module。 这与你之前添加到 Note 实体的图像属性相同。 添加第二个名为 dateCreated 的非可选属性，并使其成为 Date 类型。

接下来，从 Attachment Entity 添加到 Note Entity 的关系。 将关系名称设置为 note，并将其目标设置为 Note。

选择 Note 实体并删除图像属性。 最后，创建从 Note 实体到 Attachment 实体的一对多关系。 将其标记为可选。 命名关系为 attachments，将目标设置为附件并选择你刚刚创建的 note 关系作为反向。

data model 现在可以迁移了！ 当 Core Data model 准备就绪时，你的应用程序中的代码将需要一些更新以使用对数据 entity 的更改。 请记住，你不再使用 Note 上的图像属性，而是使用多个附件。

创建一个名为 Attachment.swift 的新文件并将其内容替换为以下内容：

```swift
import Foundation
import UIKit
import CoreData

class Attachment: NSManagedObject {
  @NSManaged var dateCreated: Date
  @NSManaged var image: UIImage?
  @NSManaged var note: Note?
}
```

接下来，打开 Note.swift 并将 image 属性替换为以下内容：

```swift
@NSManaged var attachments: Set<Attachment>?
```

你的应用程序的其余部分仍然依赖于图像属性，因此如果你尝试构建该应用程序，将会出现编译错误。 将以下内容添加到附件下方的 Note 类中：

```swift
var image: UIImage? {
  return latestAttachment?.image
}

var latestAttachment: Attachment? {
  guard let attachments = attachments,
    let startingAttachment = attachments.first else {
      return nil
  }

  return Array(attachments).reduce(startingAttachment) {
    $0.dateCreated.compare($1.dateCreated)
      == .orderedAscending ? $0 : $1
  }
}
```

此实现使用计算属性，它从最新的附件中获取图像。

如果有多个附件，顾名思义，latestAttachment 将获取最新的一个并将其返回。

接下来，打开 AttachPhotoViewController.swift。 当用户选择图像时更新它以创建一个新的 Attachment 对象。 将导入添加到文件顶部：

```swift
import CoreData
```

接下来，将 imagePickerController(\_:didFinishPickingMediaWithInfo:) 替换为：

```swift
func imagePickerController(
  _ picker: UIImagePickerController,
  didFinishPickingMediaWithInfo info:
  [UIImagePickerController.InfoKey: Any]
) {
  guard let note = note,
    let context = note.managedObjectContext else {
      return
  }

  let attachment = Attachment(context: context)
  attachment.dateCreated = Date()
  attachment.image =
    info[UIImagePickerController.InfoKey.originalImage] 
    as? UIImage
  attachment.note = note

  _ = navigationController?.popViewController(animated: true)
}
```

此实现创建一个新的 Attachment entity，将来自 UIImagePickerController 的图像添加为 image 属性，然后将 Attachment 的 note 属性设置为当前 note。

#### 映射模型

通过轻量级迁移，Core Data 可以自动创建一个映射模型，以便在更改很简单时将数据从一个模型版本迁移到另一个模型版本。 当更改不那么简单时，你可以手动设置步骤以使用映射模型从一个模型版本迁移到另一个模型版本。

重要的是要知道，在创建映射模型之前，你必须完成并最终确定你的目标模型。

在创建新映射模型的过程中，你实际上会将源模型和目标模型版本锁定到映射模型文件中。

这意味着你在创建映射模型后对实际数据模型所做的任何更改都不会被映射模型看到。

现在你已经完成了对 v3 数据模型的更改，你知道轻量级迁移无法完成这项工作。 要创建映射模型，请在 Xcode 中打开文件菜单并选择 New ▸ File。

导航到 iOS\Core Data 部分并选择映射模型：

![image-20230225172101637](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225172101637.png)

点击Next，选择v2数据模型作为源模型，选择v3数据模型作为目标模型。

将新文件命名为 UnCloudNotesMappingModel\_v2\_to\_v3。 我通常使用的文件命名约定是数据模型名称以及源版本和目标版本。 随着应用程序随着时间的推移收集越来越多的映射模型，这种文件命名约定使得更容易区分文件以及它们随时间变化的顺序。

打开 UnCloudNotesMappingModel\_v2\_to\_v3.xcmappingmodel。 幸运的是，映射模型并不是完全从头开始的； Xcode 检查源模型和目标模型并尽可能多地进行推断，因此你从一个包含基础知识的映射模型开始。

#### 属性映射

有两个映射，一个名为 NoteToNote，另一个名为 Attachment。 NoteToNote 描述了如何将 v2 Note 实体迁移到 v3 Note 实体。

选择 NoteToNote，你将看到两个部分：Attribute Mappings 和 Relationship Mappings。

![image-20230225172331060](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225172331060.png)

这里的属性映射非常简单。 注意带有模式 $source 的值表达式。 $source 是映射模型编辑器的特殊标记，表示对源实例的引用。 请记住，使用 Core Data，你不是在处理数据库中的行和列。 相反，你正在处理对象、它们的属性和类。

在这种情况下，body、dateCreated、displayIndex 和 title 的值将直接从源中传输。 这些都是简单的案例！

attachments 关系是新的，所以 Xcode 无法从源中填写任何内容。 但是，事实证明你不会使用这个特定的关系映射，所以删除这个映射。 你很快就会得到正确的关系映射。

选择 Attachment 映射并确保右侧的实用程序面板已打开。

选择 Utilities 面板中的最后一个选项卡以打开 Entity Mapping inspector：

![image-20230225172823304](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225172823304.png)

在下拉列表中选择注释作为源实体。 选择源实体后，Xcode 将尝试根据源实体和目标实体的属性名称自动解析映射。 在这种情况下，Xcode 将为你填写 dateCreated 和图像映射：

![image-20230225172849738](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225172849738.png)

Xcode 还会将实体映射从 Attachment 重命名为 NoteToAttachment。 Xcode 再次提供帮助； 它只需要你轻轻一推即可指定源实体。 由于属性名称匹配，Xcode 将为你填充值表达式。 将数据从 Note entity 映射到 Attachment entity 是什么意思？ 把这想象成是说，“对于每个 Note，制作一个 Attachment 并复制 image 和 dateCreated 属性。”

此映射将为每个 Note 创建一个 Attachment，但如果 Note 附有图像，你实际上只需要一个 Attachment。 确保选中 NoteToAttachment 实体映射，并在检查器中将 Filter Predicate 字段设置为 image != nil。 这将确保附件映射仅在源中存在图像时发生。

#### 关系映射

迁移能够将 image 从 Note 复制到 Attachment，但到目前为止，还没有将 Note 链接到 Attachment 的关系。 获得该行为的下一步是添加关系映射。

在 NoteToAttachment 映射中，你会看到一个名为 note 的关系映射。 就像你在 NoteToNote 中看到的关系映射一样，值表达式为空，因为 Xcode 不知道如何自动迁移关系。

选择 NoteToAttachment 映射。 选择关系列表中的 note 关系行，以便检查器更改以反映关系映射的属性。 在 Source Fetch 字段中，选择 Auto Generate Value Expression。 在 Key Path 字段中输入 $source 并从 Mapping Name 字段中选择 NoteToNote。

这应该生成一个如下所示的值表达式：

```
FUNCTION($manager,
  "destinationInstancesForEntityMappingNamed:sourceInstances:",
  "NoteToNote", $source)
```

FUNCTION 语句类似于 objc\_msgSend 语法； 也就是说，第一个参数是对象实例，第二个参数是选择器，任何其他参数都作为参数传递给该方法。

因此，映射模型正在调用 $manager 对象上的方法。 $manager token 是对处理迁移过程的 NSMigrationManager 对象的特殊引用。

> 注意：使用 FUNCTION 表达式仍然依赖于一些 Objective-C 语法知识。 Apple 开始将 Core Data 100% 移植到 Swift 可能还需要一段时间！

Core Data 在迁移过程中创建迁移管理器。 迁移管理器跟踪哪些源对象与哪些目标对象相关联。 destinationInstancesForEntityMappingNamed:sourceInstances: 方法将查找源对象的目标实例。

上一页中的表达式表示“将 note 关系设置为通过 NoteToNote 映射迁移到此映射的任何 $source 对象”，在本例中将是新数据存储中的 Note 实体。 你已经完成了自定义映射！ 你现在拥有一个配置为将单个实体拆分为两个并将适当的数据对象关联在一起的映射。

#### 最后一件事

在运行此迁移之前，你需要更新 Core Data 设置代码以使用此映射模型，而不是尝试自行推断。

打开 CoreDataStack.swift 并查找 storeDescription 属性，你首先在其上设置了启用迁移的标志。 将标志更改为以下内容：

```swift
description.shouldMigrateStoreAutomatically = true
description.shouldInferMappingModelAutomatically = false
```

通过将 shouldInferMappingModelAutomatically 设置为 false，你已确保 persistent store coordinator 现在将使用新的映射模型来迁移存储。 是的，这就是你需要更改的所有代码； 没有新代码！

当 Core Data 被告知不要推断或生成映射模型时，它将在默认包或主包中查找映射模型文件。 映射模型包含模型的源版本和目标版本。 Core Data 将使用该信息来确定使用哪个映射模型（如果有的话）来执行迁移。 它真的就像更改单个选项以使用自定义映射模型一样简单。

严格来说，这里没有必要将 shouldMigrateStoreAutomatically 设置为 true，因为 true 是默认值。 但是，我们只是说，我们稍后会再次需要它。

构建并运行应用程序。 你会发现表面上并没有发生太大变化！ 但是，如果你仍然像以前一样看到你的笔记和图像，则映射模型有效。 Core Data 更新了 SQLite 存储的底层模式以反映 v3 数据模型的变化。

> 注意：同样，你可能希望将 v3 源代码复制到不同的文件夹中，以便稍后返回。 或者，如果你使用的是源代码管理，请在此处设置一个标记，以便你可以回到这一点。 同样，你可能还想保存数据存储文件的副本，并为此版本的应用程序的名称附加“v3”，因为稍后你将使用它进行更复杂的迁移。

### 复杂的映射模型

想到了 UnCloudNotes 的一个新功能，是时候再次迁移数据模型了！ 这一次，仅支持图像附件是不够的，希望该应用程序的未来版本能够支持视频、音频文件或真正添加任何类型的有意义的附件。

你决定拥有一个名为 Attachment 的基础实体和一个名为 ImageAttachment 的子类。 这将使每个附件类型都有自己的有用信息。 图像可以具有标题、图像大小、压缩级别、文件大小等属性。 稍后，你可以为其他附件类型添加更多子类。

虽然新图像会在保存之前获取此信息，但你需要在迁移过程中从当前图像中提取该信息。 你需要使用 CoreImage 或 ImageIO 库。 这些是 Core Data 绝对不支持开箱即用的数据转换，这使得自定义手动迁移成为完成这项工作的合适工具。

像往常一样，任何数据迁移的第一步都是在 Xcode 中选择数据模型文件，然后选择编辑器 ▸ 添加模型版本...。这一次，创建名为 UnCloudNotesDataModel v4 的数据模型的版本 4。 不要忘记在 Xcode Inspector 中将数据模型的当前版本设置为 v4。

打开 v4 数据模型并添加一个名为 ImageAttachment 的新实体。 将类设置为 ImageAttachment，将模块设置为当前 Current Product Module。 对 ImageAttachment 进行以下更改：

1. 将父实体设置为 Attachment。
2. 添加名为 caption 的必需字符串属性。
3. 添加一个必需的名为 width 的 Float 属性。
4. 添加一个需要的 Float 属性名称 height。
5. 添加一个名为 image 的可选 Transformable 属性。
6. 将 ValueTransformer 设置为 ImageTransformer，并将 Module 值设置为 Current Product Module。

接下来，在 Attachment 实体中：

1. 删除 image 属性。
2. 如果已自动创建新关系，则将其删除。

parent entity 类似于拥有父类，这意味着 ImageAttachment 将继承 Attachment 的属性。 稍后设置托管对象子类时，你会在代码中看到这种继承。

在为映射模型创建自定义代码之前，如果现在创建 ImageAttachment 源文件会更容易。 创建一个名为 ImageAttachment 的新 Swift 文件并将其内容替换为以下内容：

```swift
import UIKit
import CoreData

class ImageAttachment: Attachment {
  @NSManaged var image: UIImage?
  @NSManaged var width: Float
  @NSManaged var height: Float
  @NSManaged var caption: String
}
```

接下来，打开 Attachment.swift 并删除 image 属性。 由于它已移至 ImageAttachment，并从 v4 数据模型中的附件实体中删除，因此应从代码中删除它。 这应该适用于新的数据模型。

#### 映射模型

在 Xcode 菜单中，选择文件 ▸ 新建文件并选择 iOS ▸ CoreData▸ Mapping Model。 选择版本 3 作为源模型，选择版本 4 作为目标。 将文件命名为 UnCloudNotesMappingModel\_v3\_to\_v4。

在 Xcode 中打开新的映射模型，你会看到 Xcode 再次帮助你填充了一些映射。

从 NoteToNote 映射开始，Xcode 直接将源 entoty 从源存储复制到目标，没有任何转换或转换。 这个简单的数据迁移的默认 Xcode 值是很好的，按原样！

选择 AttachmentToAttachment 映射。 Xcode 还检测了源实体和目标实体中的一些公共属性并生成了映射。 但是，你想将 Attachment 实体转换为 ImageAttachment 实体。 Xcode 在此处创建的内容会将旧的 Attachment 实体映射到新的 Attachment 实体，这不是此迁移的目标。 删除此映射。

接下来，选择 ImageAttachment 映射。 此映射没有源实体，因为它是一个全新的实体。 在检查器中，将源实体更改为附件。 现在 Xcode 知道了来源，它会为你填充一些值表达式。 Xcode 还会将映射重命名为更合适的名称 AttachmentToImageAttachment。

![image-20230225211107881](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230225211107881.png)

对于剩余的未填充属性，你需要编写一些代码。 这是你需要图像处理和自定义代码的地方，而不是简单的 FUNCTION 表达式！ 但首先，删除那些额外的映射、标题、高度和宽度。 这些值将使用自定义迁移策略计算，这恰好是下一节！

#### 自定义迁移策略

要超越映射模型中的 FUNCTION 表达式，你可以直接子类化 NSEntityMigrationPolicy。 这让你可以编写 Swift 代码来逐个实例地处理迁移，这样你就可以调用可供应用程序其余部分使用的任何框架或库。

将一个新的 Swift 文件添加到名为 AttachmentToImageAttachmentMigrationPolicyV3toV4.swift 的项目中，并将其内容替换为以下起始代码：

```swift
import CoreData
import UIKit

let errorDomain = "Migration"

class AttachmentToImageAttachmentMigrationPolicyV3toV4: NSEntityMigrationPolicy {

}
```

这个命名约定对你来说应该很熟悉； 它注意到这是一个自定义迁移策略，用于将数据从模型版本 3 中的附件转换为模型版本 4 中的 ImageAttachments。

在你忘记它之前，你需要将这个新的映射类连接到你新创建的映射文件。 回到 v3-to-v4 映射模型文件，选择 AttachmentToImageAttachment 实体映射。 在 Entity Mapping Inspector 中，使用你刚刚创建的完全命名空间的类名（包括模块）填写 Custom Policy 字段：

* UnCloudNotes.AttachmentToImageAttachmentMigrationPolicyV3toV4

当你按 Enter 确认此更改时，自定义策略上方的类型应更改为读取自定义。

当 Core Data 运行此迁移时，它会在需要为该特定数据集执行数据迁移时创建自定义迁移策略的实例。 这是你运行任何自定义转换代码以在迁移期间提取图像信息的机会！ 现在，是时候向自定义实体映射策略添加一些自定义逻辑了。

打开 AttachmentToImageAttachmentMigrationPolicyV3toV4.swift 并添加执行迁移的方法：

```swift
override func createDestinationInstances(
  forSource sInstance: NSManagedObject,
  in mapping: NSEntityMapping,
  manager: NSMigrationManager
) throws {
  // 1
  let description = NSEntityDescription.entity(
    forEntityName: "ImageAttachment",
    in: manager.destinationContext)
    let newAttachment = ImageAttachment(
    entity: description!,
    insertInto: manager.destinationContext
  )

  // 2
  func traversePropertyMappings(
    block: (NSPropertyMapping, String) -> Void
  ) throws {
    if let attributeMappings = mapping.attributeMappings {
      for propertyMapping in attributeMappings {
        if let destinationName = propertyMapping.name {
          block(propertyMapping, destinationName)
        } else {
          // 3
          let message =
            "Attribute destination not configured properly"
          let userInfo =
            [NSLocalizedFailureReasonErrorKey: message]
          throw NSError(domain: errorDomain,
                        code: 0, userInfo: userInfo)
        }
      }
    } else {
      let message = "No Attribute Mappings found!"
      let userInfo = [NSLocalizedFailureReasonErrorKey: message]
      throw NSError(domain: errorDomain,
                    code: 0, userInfo: userInfo)
    }
  }

  // 4
  try traversePropertyMappings { propertyMapping, destinationName in
    if let valueExpression = propertyMapping.valueExpression {
      let context: NSMutableDictionary = ["source": sInstance]
      guard let destinationValue =
        valueExpression.expressionValue(
        with: sInstance,
        context: context
      ) else {
          return
      }

      newAttachment.setValue(destinationValue,
                             forKey: destinationName)
    }
  }

  // 5
  if let image = sInstance.value(forKey: "image") as? UIImage {
    newAttachment.setValue(image.size.width, forKey: "width")
    newAttachment.setValue(image.size.height, forKey: "height")
  }

  // 6
  let body = sInstance.value(
    forKeyPath: "note.body"
  ) as? NSString ?? ""
  newAttachment.setValue(
    body.substring(to: 80),
    forKey: "caption"
  )

  // 7
  manager.associate(
    sourceInstance: sInstance,
    withDestinationInstance: newAttachment,
    for: mapping
  )
}
```

此方法是对默认 NSEntityMigrationPolicy 实现的覆盖。 迁移管理器使用它来创建目标实体的实例。 源对象的实例是第一个参数； 覆盖后，由开发人员创建目标实例并将其正确关联到迁移管理器。

这是正在发生的事情，一步一步：

1. 首先，你创建新目标对象的一个实例。 迁移管理器有两个 Core Data stacks——一个从 source 读取，一个写入 destination ——所以你需要确保在这里使用 destination context。 现在，你可能会注意到本节没有使用新的花哨的短 ImageAttachment(context: NSManagedObjectContext) 初始化程序。 好吧，事实证明，使用新语法此迁移只会崩溃，因为它取决于已加载和最终确定的模型，而这并没有发生在迁移的中途。
2. 接下来，创建一个 traversePropertyMappings 函数，该函数执行迭代属性映射的任务（如果它们存在于迁移中）。 此函数将控制遍历，而下一节将执行每个属性映射所需的操作。
3. 如果出于某种原因，entityMapping 对象的 attributeMappings 属性没有返回任何映射，这意味着你的映射文件指定不正确。 发生这种情况时，该方法将抛出错误并提供一些有用的信息。
4. 尽管是自定义手动迁移，但大多数属性迁移应该使用你在映射模型中定义的表达式来执行。 为此，使用上一步中的遍历函数并将值表达式应用于源实例并将结果设置为新的目标对象。
5. 接下来，尝试获取图像的实例。 如果存在，则获取其 width 和 height 以填充新对象中的数据。
6. 对于 caption，只需抓住 note 的 body 文本并取前 80 个字符。
7. 迁移管理器需要知道源对象、新创建的目标对象和映射之间的联系。 在自定义迁移结束时未能调用此方法将导致目标存储中的数据丢失。

这就是自定义迁移代码！ Core Data 在启动时检测到 v3 数据存储时将选择映射模型，并将其应用于迁移到新的数据模型版本。 由于你添加了自定义 NSEntityMigrationPolicy 子类并在映射模型中链接到它，Core Data 将自动调用你的代码。

最后，是时候回到主 UI 代码并更新数据模型使用以考虑新的 ImageAttachment 实体了。 打开 AttachPhotoViewController.swift 并找到 imagePickerController(\_:didFinishPickingMediaWithInfo:)。

更改设置附件的行，使其改为使用 ImageAttachment：

```swift
let attachment = ImageAttachment(context: context)
```

当你在这里时，你还应该为 caption 属性添加一个值。 caption 属性是必需的字符串值，因此如果创建的 ImageAttachment 没有值（即 nil 值），则保存将失败。

理想情况下，会有一个额外的字段用于输入值，但现在添加以下行：

```swift
attachment.caption = "New Photo"
```

接下来，打开 Note.swift 并将 image 属性替换为以下内容：

```swift
var image: UIImage? {
  let imageAttachment = latestAttachment as? ImageAttachment
  return imageAttachment?.image
}
```

现在所有代码更改都已就位，你需要更新主数据模型以使用 v4 作为主数据模型。

在项目导航器中选择 UnCloudNotesDataModel.xcdatamodeld。 在身份窗格中的模型版本下，选择 UnCloudNotesDataModel v4。

构建并运行应用程序。 数据应正确迁移。 同样，你的 note、image 和所有内容都将在那里，但你现在拥有面向未来的 UnCloudNotes，可以添加视频、音频和其他任何内容！

### 迁移非顺序版本

到目前为止，你已经按顺序完成了一系列数据迁移。 你已按顺序将数据从版本 1 迁移到版本 2、版本 3 到版本 4。 不可避免的是，在 App Store 发布的现实世界中，用户可能会跳过更新，例如需要从版本 2 升级到版本 4。 那么会发生什么？

当 Core Data 执行迁移时，它的意图是只执行一次迁移。 在这个假设的场景中，Core Data 会寻找一个从版本 2 到版本 4 的映射模型； 如果一个不存在，如果你告诉它，Core Data 会推断出一个。 否则迁移将失败，并且 Core Data 将在尝试将存储附加到持久存储协调器时报告错误。

你如何处理这种情况，以便你请求的迁移成功？ 你可以提供多个映射模型，但随着应用程序的增长，你需要提供的映射模型的数量会过多：从 v1 到 v4、v1 到 v3、v2 到 v4 等等。 你花在映射模型上的时间会比花在应用程序本身上的时间更多！

解决方案是实施完全自定义的迁移序列。 你知道从版本 2 到版本 3 的迁移是有效的； 从 2 到 4，如果你手动将商店从 2 迁移到 3 以及从 3 迁移到 4，它会很好地工作。这种逐步迁移意味着你将阻止 Core Data 寻找直接的 2 到 4 或 甚至是 1 到 4 次迁移。

### 自迁移 Stack

要开始实施此解决方案，你需要创建一个单独的迁移管理器类。 此类的责任是在被要求时提供正确迁移的 Core Data stack。 此类将具有 stack 属性并将返回 CoreDataStack 的实例，正如 UnCloudNotes 在整个过程中使用的那样，它已经完成了对应用程序有用的所有必要迁移。

首先，创建一个名为 DataMigrationManager 的新 Swift 文件。 打开文件并将其内容替换为以下内容：

```swift
import Foundation
import CoreData

class DataMigrationManager {
  let enableMigrations: Bool
  let modelName: String
  let storeName: String = "UnCloudNotesDataModel"
  var stack: CoreDataStack

  init(modelNamed: String, enableMigrations: Bool = false) {
    self.modelName = modelNamed
    self.enableMigrations = enableMigrations
  }
}
```

你会注意到我们开始时看起来像当前的 CoreDataStack 初始化程序。 这是为了让下一步更容易理解。

接下来，打开NotesListViewController.swift，替换栈惰性初始化代码：

```swift
private lazy var stack: CoreDataStack =
  CoreDataStack(modelName: "UnCloudNotesDataModel")
```

为：

```swift
private lazy var stack: CoreDataStack = {
  let manager = DataMigrationManager(
    modelNamed: "UnCloudNotesDataModel",
    enableMigrations: true)
  return manager.stack
}()
```

你将使用 lazy 属性来保证堆栈只被初始化一次。 其次，初始化实际上由 DataMigrationManager 处理，因此使用的堆栈将是从 DataMigrationManager 返回回的堆栈。 如前所述，新的 DataMigrationManager 初始化程序的签名类似于 CoreDataStack。 那是因为你有大量的迁移代码，将迁移的责任与保存数据的责任分开是个好主意。

> 注意：项目不会立即构建，因为你尚未在 DataMigrationManager 中初始化 stack 属性的值。 放心，这很快就会出现。

现在是更难的部分：你如何确定商店是否需要迁移？ 如果是这样，你如何确定从哪里开始？ 为了进行完全自定义的迁移，你需要一些支持。 首先，找出模型是否匹配并不明显。 你还需要一种方法来检查持久存储文件与模型的兼容性。 让我们先开始使用所有支持功能！

在 DataMigrationManager.swift 的底部，在 NSManagedObjectModel 上添加一个扩展：

```swift
extension NSManagedObjectModel {
  private class func modelURLs(
    in modelFolder: String
  ) -> [URL] {
    Bundle.main
      .urls(forResourcesWithExtension: "mom",
      subdirectory: "\(modelFolder).momd") ?? []
  }

  class func modelVersionsFor(
    modelNamed modelName: String
  ) -> [NSManagedObjectModel] {
    modelURLs(in: modelName)
      .compactMap(NSManagedObjectModel.init)
  }

  class func uncloudNotesModel(
    named modelName: String
  ) -> NSManagedObjectModel {
    let model = modelURLs(in: "UnCloudNotesDataModel")
      .first { $0.lastPathComponent == "\(modelName).mom" }
      .flatMap(NSManagedObjectModel.init)
    return model ?? NSManagedObjectModel()
  }
}
```

第一个方法返回给定名称的所有模型版本。 第二种方法返回一个名为 UnCloudNotesDataModel 的 NSManagedObjectModel 的特定实例。 通常，Core Data 会给你最新的数据模型版本，但这种方法会让你深入了解特定版本。

> 注意：当 Xcode 将你的应用程序编译到其应用程序包中时，它还会编译你的数据模型。 应用程序包将在其根目录下有一个包含 .mom 文件的 .momd 文件夹。 MOM 或托管对象模型文件是 .xcdatamodel 文件的编译版本。 每个数据模型版本都有一个 .mom。

要使用此方法，请在 NSManagedObjectModel 类扩展中添加以下方法：

```swift
class var version1: NSManagedObjectModel {
  uncloudNotesModel(named: "UnCloudNotesDataModel")
}
```

此方法返回数据模型的第一个版本。 这负责获取模型，但是如何检查模型的版本呢？ 将以下属性添加到类扩展：

```swift
var isVersion1: Bool {
  self == Self.version1
}
```

NSManagedObjectModel 的比较运算符对于正确检查模型相等性不是很有帮助。 要使 == 比较在两个 NSManagedObjectModel 对象上起作用，请将以下运算符函数添加到文件中。 你需要在类扩展之外添加它，就在全局范围内：

```swift
func == (
  firstModel: NSManagedObjectModel,
  otherModel: NSManagedObjectModel
) -> Bool {
  firstModel.entitiesByName == otherModel.entitiesByName
}
```

这里的想法很简单：如果两个 NSManagedObjectModel 对象具有相同的实体集合和相同的版本散列，则它们是相同的。

现在一切都已设置，你可以为接下来的 3 个版本重复 version 和 isVersion 模式。 继续将以下版本 2 到 4 的方法添加到类扩展中：

```swift
class var version2: NSManagedObjectModel {
  uncloudNotesModel(named: "UnCloudNotesDataModel v2")
}

var isVersion2: Bool {
  self == Self.version2
}

class var version3: NSManagedObjectModel {
  uncloudNotesModel(named: "UnCloudNotesDataModel v3")
}

var isVersion3: Bool {
  self == Self.version3
}

class var version4: NSManagedObjectModel {
  uncloudNotesModel(named: "UnCloudNotesDataModel v4")
}

var isVersion4: Bool {
  self == Self.version4
}
```

现在你已经有了比较模型版本的方法，你将需要一种方法来检查特定的持久存储是否与模型版本兼容。 将这两个辅助方法添加到 DataMigrationManager 类：

```swift
private func store(
  at storeURL: URL,
  isCompatibleWithModel model: NSManagedObjectModel
) -> Bool {
  let storeMetadata = metadataForStoreAtURL(storeURL: storeURL)

  return model.isConfiguration(
    withName: nil,
    compatibleWithStoreMetadata:storeMetadata
  )
}

private func metadataForStoreAtURL(
  storeURL: URL
) -> [String: Any] {
  let metadata: [String: Any]
  do {
    metadata = try NSPersistentStoreCoordinator
    .metadataForPersistentStore(
      ofType: NSSQLiteStoreType,
      at: storeURL, 
      options: nil
    )
  } catch {
    metadata = [:]
    print("Error retrieving metadata for store at URL:
      \(storeURL): \(error)")
  }
  return metadata
}
```

第一种方法是一个简单的便利包装器，用于确定持久存储是否与给定模型兼容。 第二种方法有助于安全地检索 Store 的元数据。

接下来，将以下计算属性添加到 DataMigrationManager 类：

```swift
private var applicationSupportURL: URL {
  let path = NSSearchPathForDirectoriesInDomains(
    .applicationSupportDirectory,
    .userDomainMask,
    true
  ).first
  return URL(fileURLWithPath: path!)
}

private lazy var storeURL: URL = {
  let storeFileName = "\(self.storeName).sqlite"
  return URL(
    fileURLWithPath: storeFileName,
    relativeTo: self.applicationSupportURL
  )
}()

private var storeModel: NSManagedObjectModel? {
  NSManagedObjectModel.modelVersionsFor(modelNamed: modelName)
    .first {
        self.store(at: storeURL, isCompatibleWithModel: $0)
    }
}
```

这些属性允许你访问当前 Store URL 和模型。 事实证明，CoreData API 中没有方法可以向 Store 询问其模型版本。 相反，最简单的解决方案是蛮力。 由于你已经创建了辅助方法来检查 Store 是否与特定模型兼容，因此你只需遍历所有可用模型，直到找到适用于该 Store 的模型。

接下来，你需要你的迁移管理器记住当前的模型版本。 为此，你将首先创建一个从包中获取模型的通用方法，然后你将简单地使用该通用方法来查找模型。

首先，将以下方法添加到 NSManagedObjectModel 类扩展中：

```swift
class func model(
  named modelName: String,
  in bundle: Bundle = .main
) -> NSManagedObjectModel {
  bundle
    .url(forResource: modelName, withExtension: "momd")
    .flatMap(NSManagedObjectModel.init)
      ?? NSManagedObjectModel()
}
```

这个方便的方法用于使用顶级文件夹初始化托管对象模型。 Core Data 将自动查找当前模型版本并将该模型加载到 NSManagedObjectModel 中以供使用。 重要的是要注意，此方法仅适用于已版本化的 Core Data 模型。

接下来，向 DataMigrationManager 类添加一个属性，如下所示：

```swift
private lazy var currentModel: NSManagedObjectModel =
    .model(named: self.modelName)
```

currentModel 属性是惰性的，所以它只加载一次，因为它每次都应该返回相同的东西。 .model 是调用刚刚添加的函数的简写方式，该函数将从顶级 momd 文件夹中查找模型。

当然，如果你拥有的模型不是当前模型，那就是运行迁移的时候了！ 将以下起始方法添加到 DataMigrationManager 类（你将在稍后填写）：

```swift
func performMigration() {
}
```

接下来，将你之前添加的 stack 属性定义替换为以下内容：

```swift
var stack: CoreDataStack {
  guard enableMigrations,
    !store(
      at: storeURL,
      isCompatibleWithModel: currentModel
    )
  else { return CoreDataStack(modelName: modelName) }

  performMigration()
  return CoreDataStack(modelName: modelName)
}
```

最后，计算属性将返回一个 CoreDataStack 实例。 如果设置了迁移标志，则检查初始化中指定的存储是否与 Core Data 确定为数据模型的当前版本兼容。 如果 Store 无法加载当前模型，则需要进行迁移。 否则，你可以使用具有当前设置的模型的任何版本的堆栈对象。

你现在拥有一个自迁移的 Core Data stack，可以始终保证与最新的模型版本保持同步！ 构建项目以确保所有内容都可以编译。 下一步是添加自定义迁移逻辑。

#### 自迁移 stack

现在是时候开始构建迁移逻辑了。 将以下方法添加到 DataMigrationManager 类：

```swift
private func migrateStoreAt(
  URL storeURL: URL,
  fromModel from: NSManagedObjectModel,
  toModel to: NSManagedObjectModel,
  mappingModel: NSMappingModel? = nil
) {
  // 1
  let migrationManager =
    NSMigrationManager(sourceModel: from, destinationModel: to)

  // 2
	var migrationMappingModel: NSMappingModel
  if let mappingModel = mappingModel {
    migrationMappingModel = mappingModel
  } else {
    migrationMappingModel = try! NSMappingModel
    .inferredMappingModel(
        forSourceModel: from, destinationModel: to)
  }

  // 3
  let targetURL = storeURL.deletingLastPathComponent()
  let destinationName = storeURL.lastPathComponent + "~1"
  let destinationURL = targetURL
    .appendingPathComponent(destinationName)

  print("From Model: \(from.entityVersionHashesByName)")
  print("To Model: \(to.entityVersionHashesByName)")
  print("Migrating store \(storeURL) to \(destinationURL)")
  print("Mapping model: \(String(describing: mappingModel))")

  // 4
  let success: Bool
  do {
    try migrationManager.migrateStore(
      from: storeURL,
      sourceType: NSSQLiteStoreType,
      options: nil,
      with: migrationMappingModel,
      toDestinationURL: destinationURL,
      destinationType: NSSQLiteStoreType,
      destinationOptions: nil
    )
    success = true
  } catch {
    success = false
    print("Migration failed: \(error)")
  }
  // 5
  if success {
    print("Migration Completed Successfully")

    let fileManager = FileManager.default
    do {
        try fileManager.removeItem(at: storeURL)
        try fileManager.moveItem(
          at: destinationURL,
          to: storeURL
        )
    } catch {
      print("Error migrating \(error)")
    }
  }
}
```

这种方法完成了所有繁重的工作。 如果你需要做一个轻量级的迁移，你可以传递 nil 或者直接跳过最后一个参数。

这是正在发生的事情，一步一步：

1. 首先，你创建一个迁移管理器的实例。
2. 如果将映射模型传递给该方法，则使用它。 否则，创建一个推断映射模型。
3. 由于迁移将创建第二个数据存储并逐个实例地从原始文件迁移数据到新文件，因此目标 URL 必须是不同的文件。 现在，本节中的示例代码将创建一个与原始文件夹相同的 destinationURL 和一个用“\~1”连接的文件。 目标 URL 可以位于临时文件夹中，也可以位于你的应用有权写入文件的任何位置。
4. 这是你让迁移管理器发挥作用的地方！ 你已经使用源模型和目标模型对其进行了设置，因此你只需将映射模型和两个 URL 添加到组合中。
5. 鉴于结果，你可以将成功或错误消息打印到控制台。 在成功案例中，你也执行了一些清理工作。 在这种情况下，删除旧商店并用新商店替换它就足够了。

现在只需使用正确的参数调用该方法即可。 还记得 performMigration 的空实现吗？ 是时候填写了。

将以下行添加到该方法：

```swift
if !currentModel.isVersion4 {
  fatalError("Can only handle migrations to version 4!")
}
```

此代码将仅检查当前模型是否是该模型的最新版本。 如果当前模型不是第 4 版，这段代码会退出并终止应用程序。这有点极端——在你自己的应用程序中，你可能想要继续迁移——但这样做肯定会提醒你思考 关于迁移，如果你曾经向你的应用程序添加另一个数据模型版本！ 值得庆幸的是，即使这是 performMigration 方法中的第一次检查，它也不应该运行，因为下一部分在应用最后一个可用迁移后停止。

可以改进 performMigration 方法以处理所有已知模型版本。 为此，请在先前添加的 if 语句下方添加以下内容：

```swift
if let storeModel = self.storeModel {
  if storeModel.isVersion1 {
    let destinationModel = NSManagedObjectModel.version2

    migrateStoreAt(
      URL: storeURL,
      fromModel: storeModel,
      toModel: destinationModel
    )

    performMigration()
  } else if storeModel.isVersion2 {
    let destinationModel = NSManagedObjectModel.version3
    let mappingModel = NSMappingModel(
      from: nil,
      forSourceModel: storeModel,
      destinationModel: destinationModel
    )

    migrateStoreAt(
      URL: storeURL,
      fromModel: storeModel,
      toModel: destinationModel,
      mappingModel: mappingModel
    )

    performMigration()
  } else if storeModel.isVersion3 {
    let destinationModel = NSManagedObjectModel.version4
    let mappingModel = NSMappingModel(
      from: nil,
      forSourceModel: storeModel,
      destinationModel: destinationModel
    )

    migrateStoreAt(
      URL: storeURL,
      fromModel: storeModel,
      toModel: destinationModel,
      mappingModel: mappingModel
    )
  }
}
```

无论你从哪个版本开始，步骤都是类似的：

* 轻量级迁移使用简单标志 1) 启用迁移，以及 2) 推断映射模型。 由于如果缺少一个映射模型，migrateStoreAt 方法将推断出一个映射模型，因此你已成功替换该功能。 通过运行 performMigration，你已经启用了迁移。
* 将目标模型设置为正确的模型版本。 请记住，你一次只能“升级”一个版本，因此从 1 到 2 以及从 2 到 3。
* 对于版本 2 及更高版本，还加载映射模型。
* 最后，调用你在本节开头编写的 migrateStoreAt(URL:fromModel:toModel:mappingModel:)。

这个解决方案的优点在于，尽管有所有比较支持辅助函数，DataMigrationManager 类实际上是在使用已经为每个迁移定义的映射模型和代码。

该解决方案是按顺序手动应用每个迁移，而不是让 Core Data 尝试自动执行操作。

> 注意：如果你从版本 1 或 2 开始，最后会递归调用 performMigration()。 这将触发另一次运行以继续序列； 一旦你处于版本 3 并运行迁移以到达版本 4。随着你添加更多数据模型版本以继续自动迁移序列，你将继续添加此方法。

### 测试顺序迁移

测试这种类型的迁移可能有点复杂，因为你需要回到过去并运行以前版本的应用程序以生成要迁移的数据。 如果你在此过程中保存了应用程序项目的副本，那就太好了！

否则，你将在本书捆绑的资源中找到该项目的以前版本。

首先，确保你按原样复制了项目——这是最终项目！

以下是测试每次迁移所需的一般步骤：

1. 从模拟器中删除应用程序以清除数据存储。
2. 打开应用程序的版本 2（这样你至少可以看到一些图片！），然后构建并运行。
3. 创建一些测试笔记。
4. 从 Xcode 退出应用程序并关闭项目。
5. 打开应用程序的最终版本，构建并运行。

此时，你应该会看到一些带有迁移状态的控制台输出。 请注意，迁移将在应用出现在屏幕上之前发生。

![image-20230226025710065](https://raw.githubusercontent.com/LLLLLayer/Galaxy/main/resources/images/core\_data/image-20230226025710065.png)

你现在拥有一个可以在旧数据版本的任意组合之间成功迁移到最新版本的应用程序。

### 关键点

* 当你需要对数据模型进行更改时，迁移是必要的。
* 尽可能使用最简单的迁移方法。
* 轻量级迁移是 Apple 的术语，指的是你参与的工作量最少的迁移。
* 重量级迁移，如 Apple 所述，可以包含多种不同类型的自定义迁移。
* 自定义迁移让你可以创建一个映射模型来指导核心数据进行轻量级无法自动完成的更复杂的更改。
* 创建映射模型后，请勿更改目标模型。
* 自定义手动迁移比映射模型更进一步，让你可以从代码更改模型。
* 完全手动迁移让你的应用程序从一个版本顺序迁移到下一个版本，如果用户跳过将他们的设备更新到两者之间的版本，可以防止出现问题。
* 迁移测试很棘手，因为它依赖于源存储中的数据。 在将你的应用程序发布到 App Store 之前，请务必测试多个场景。
