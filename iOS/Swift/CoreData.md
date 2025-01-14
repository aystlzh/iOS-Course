# CoreData

* 要把 `Core Data` 作为一个对象图管理系统来使用，而不是关系型数据库

* `Core Data` 的架构图：
![](https://i.loli.net/2019/06/28/5d14eb260c2f935863.png)

    - **托管对象**：我们创建的数据模型。
    - **托管对象上下文**：托管对象上下文记录了它管理的对象，以及对这些对象的 CRUD，每个被托管的对象都知道自己属于哪个上下文。
    - `Core Data` 支持多个上下文。
    - 数据底层实际上会被存储在 `SQLite` 数据库里。

* `Core Data` 存储结构化的数据。所以为了使用 `Core Data` 需要先创建一个数据模型来描述我们的数据结构。

* **实体**：一个实体应该代表你的应用程序里有意义的一个部分数据。实体名称以大写字母开头。

* **属性**：属性名称应该以小写字母开头。因为 `Array` 已经遵循里 `NSCoding` 协议，所以可以直接把这样的数组直接存入一个可转换类型的属性 `Transformable` 里。

* 属性选项：**必选属性** `non-optional` 必须要赋给它们恰当的值，才能保存这些数据。把属性标记为可索引时 `indexed`，`Core Data` 会在底层 `SQLite` 数据库表里创建一个索引，可以加速这个属性的搜索和排序，但代价是插入数据时的性能下降和额外的存储空间。

* **托管对象子类**：实体只是面熟了那些数据属于某个对象，为了能够在代码中使用这个数据，还需要一个具有和实体里定义的属性们相对应的属性的类。
    - 实体和类都叫同一个名字。
    - 不建议使用 `Xcode` 的代码生成工程工具，而是直接手写。
    - 代码通常会如下所示：
        ```swift
        final class Mood: NSManagedObject { @NSManaged 􏰀leprivate(set) var date: Date @NSManaged 􏰀leprivate(set) var colors: [UIColor]
}
        ```
    -  `@NSManaged` 标签告诉编译器这些属性由 `Core Data` 来实现。
    - 在模型编辑器中选中这个实体，然后在 `data model inspector` 里输入它的类名完成实体和类的关联。

* **设置 `Core Data` 栈**

* **获取请求**
每次执行一个获取请求，都会直达文件系统，是一个相对昂贵的操作，可能是一个潜在的性能瓶颈。

* **Fetched Results Controller**
    - 与 `tableView` 的交互：
        ![](https://i.loli.net/2019/06/28/5d14f273b221324819.png)
    

* 始终把 `Core Data` 对象交互的代码封装进类似的一个 `block` 里。
