---
layout: post
title: Swift 学习笔记（三）
category: Tech
---

## 与 Objective-C API 交互

### 初始化

* 使用 Swift 语法调用 Objective-C 的构造器时，方法名中的 `init` 和 `initWith` 前缀会被截去，其余各部分依次变为构造器的参数名。同时不再需要调用 `alloc` 方法。
* 为了一致性和便捷性，Objective-C 中的工厂方法（factory methods）也被映射成为 Swift 中的构造器。
* 在 Objective-C 中可能返回 `nil` 的构造器，引入 Swift 时被定义为了可失败构造器。

```objective-c
// Objective-C
UITableView *myTableView = [[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStyleGrouped];
UIColor *color = [UIColor colorWithRed:0.5 green:0.0 blue:0.5 alpha:1.0];
```

```swift
// Swift
let myTableView = UITableView(frame: CGRectZero, style: .Grouped)
let color = UIColor(red: 0.5, green: 0.0, blue: 0.5, alpha: 1.0)
```

### 方法和属性

* 在 Swift 中调用 Objective-C 对象的方法和属性时，使用点语法。
* Objective-C 的方法移植到 Swift 中时，原方法名的第一部分作为新的方法名，其余部分依次作为第二个及以后参数的外部名称，而第一个参数不需要名称。

<!--more-->

### 闭包

* Swift 中的闭包和 Objective-C 中的代码块是互相兼容的，所以可以将闭包直接传入一个以代码块为参数的函数。

```objective-c
// Objective-C
void (^completionBlock)(NSData *, NSError *) = ^(NSData *data, NSError *error) {/* ... */}
```

```swift
// Swift
let completionBlock: (NSData, NSError) -> Void = {data, error in /* ... */}
```

* 然而闭包和代码块有一个关键性的不同，闭包中的变量是可修改的，而不像代码块那样使用变量值的拷贝。即 Swift 中变量的默认行为相当于加了 Objective-C 中的 `__block` 关键字。

### `id` 兼容性

* Objective-C 中的 `id` 可以指向任何类型的对象，这相当于 Swift 中的 `AnyObject` 协议类型。
* `AnyObject` 允许在不进行类型转换的情况下调用任何 Objective-C 的方法和属性，这个行为类似于隐式解析可选类型。因此与 Objective-C 不同，若方法或属性不存在将触发运行时错误，这可以使用可选链来避免。

### Swift 类型兼容性

* 如果希望 Swift API 在 Objective-C 中可用，可以使用 `@objc` 属性。
* 如果在 Swift 中定义的类继承自 `NSObject` 或者其他 Objective-C 类，编译器将自动添加 `@objc` 属性，同时类中所有的属性和方法也会加上该属性，除非其权限级别为 `private`。另外当使用 `@IBOutlet` / `@IBAction` / `@NSManaged` 属性时，`@objc` 也会被自动加上。
* 当在 Objective-C 中使用 Swift API 时，编译器会作直接的翻译，如 `func playSong(name: String)` 会被翻译为 `- (void)playSong:(NSString *)name`。
* 而对于构造器，其第一个参数名会在前面加上 `initWith` 成为方法名，如 `init(songName: String, artist: String)` 将被翻译为 `- (instancetype)initWithSongName:(NSString *)songName artist:(NSString *)artist`。
* 亦可以使用 `@objc(name)` 指定在 Ojective-C 中的标识符，方法有参数时需要加上 `:`。

### Objective-C Selector

* Selector 是一个指向 Objective-C 方法名的类型，在 Swift 中它被表示为 `Selector` 结构体。而字符串能够自动转换为 selector，因此可以传递给一个以 selector 为参数的方法。


## 以 Objective-C 的行为编写 Swift 类

### Interface Builder

* 在 Swift 中使用 outlet 和 action 时，需要在属性和方法前加上 `@IBOutlet` 和 `@IBAction`，并将它们声明为隐式解析可选类型。
* 对于一个继承自 `UIView` 或 `NSView` 的自定义视图，可以在类定义前加上 `@IBDesignable`，则该视图将在 IB 的画布（canvas）上实时渲染。
* 同时可以为类的属性添加 `@IBInspectable`，这样便能在 IB 的监视器面板（inspector）中编辑这些属性。

### Core Data Managed Object

* Core Data 为 `NSManagedObject` 子类的属性提供了底层存储和实现，所以需要在 Core Data 数据模型相应的属性前面加上 `@NSManaged`，以告诉编译器该属性的存储和实现将在运行时提供，类似于 Objective-C 中的 `@dynamic`。

### 在 Objective-C API 中使用 Swift 类名

* Swift 的类以其属于的模块为命名空间，因此在 Objective-C API 中引用 Swift 类名时需要写出其全名，如 `MyFramework.DataManager`。


## 使用 Cocoa 数据类型

### 字符串

* `import Foundation` 后，Swift 自动为 `String` 和 `NSString` 进行桥接，因此几乎不必在代码中使用 `NSString` 对象。例如可以在 Swift 字符串中访问 `NSString` 类的 `capitalizedString` 属性，Swift 会自动将 `String` 桥接到 `NSString` 对象并访问这个属性，最后经转换它仍然返回 `String`。
* 在 Objective-C 中，会使用 `NSLocalizedString` 一系列不同的宏来对字符串进行本地化，而在 Swift 中这被简化为一个单独的函数 `NSLocalizedString(key:tableName:bundle:value:comment:)`，其中 `tableName` / `bundle` / `value` 参数已提供默认值。

### 数字

* `Int` / `UInt` / `Float` / `Double` / `Bool` 均被自动桥接到 `NSNumber`，对于以 `NSNumber` 为参数的 API 可以直接使用它们，但反过来不行，因为 `NSNumber` 可能代表多种不同的类型。

### 集合类型

* 当把 `NSArray` 桥接到 Swift 数组时，最终得到的结果是 `[AnyObject]` 类型，所有 Objective-C 对象和 Swift 类实例都与其兼容；而 `Int` 虽然不是一个类的实例，但因为它可以和 `NSNumber` 桥接，所以也是和 `AnyObject` 兼容的。
* 而把 `NSDictionary` 桥接到 Swift 字典时会返回 `[NSObject: AnyObject]`。


## 采用 Cocoa 设计模式

### 委托（Delegation）

* 在 Swift 和 Objective-C 中，委托通常都是以协议的形式来表示。委托的过程通常包括：检查委托者是否非空、检查方法是否被实现、若满足上述条件则调用这个方法，这些可以通过 Swift 中的可选链和可选绑定来简便地实现。

### 错误报告（Eroor Reporting）

* Swift 的错误报告沿用了 Objective-C 传参的模式。在 Swift 中可以定义一个 `NSError?` 类型的可选变量，并在变量前加上 `&` 运算符使其成为 `NSErrorPointer` 对象作为 error 参数。
* 当自定义一个以 `NSErrorPointer` 为参数的函数时，为对象的 `memory` 属性赋予 `NSError` 对象即可。

### Target-Action

* Target-action 设计模式用于当特定事件发生时，一个对象向另一个对象发送消息。在 Swift 中该模式与 Objective-C 中基本相似，并且可以使用 Swift 中的 `Selector` 类型。

### 内省（Introspection）

* 在 Objective-C 中，可以使用 `isKindOfClass:` 来检测对象的类型，使用 `conformsToProtocol:` 来检测对象是否遵守协议。而在 Swift 中可以通过 `is` 运算符实现以上功能，且能通过 `as?` / `as!` 向下转型。


## 与 C API 交互

### 原始类型（Primitive Types）

* Swift 为 C 语言的原始类型提供了等价的 Swift 类型，但它们不会隐式转换为 Swift 原有的数字类型。

| C 类型 | Swift 类型 |
| ---- | ---- |
| bool | CBool |
| char, signed char | CChar |
| unsigned char | CUnsignedChar |
| short | CShort |
| unsigned short | CUnsignedShort |
| int | CInt |
| unsigned int | CUnsignedInt |
| long | CLong |
| unsigned long | CUnsignedLong |
| long long | CLongLong |
| unsigned long long | CUnsignedLongLong |
| wchar_t | CWideChar |
| char16_t | CChar16 |
| char32_t | CChar32 |
| float | CFloat |
| double | CDouble |

### 枚举

* Swift 会导入所有用 `NS_ENUM` / `NS_OPTIONS` 宏标记的 C 样式枚举，同时枚举成员的名称前缀会被自动截断，例如 `UITableViewCellStyleDefault` 在 Swift 中将是 `UITableViewCellStyle` 的成员 `.Default`。

### 指针

* Swift 尽可能避免了对指针的直接访问，但当需要直接操作内存时仍有多种指针类型可用。

| C 语法 | Swift 语法 |
| ---- | ---- |
| const Type * | UnsafePointer\<Type\> |
| Type * | UnsafeMutablePointer\<Type\> |
| Type * const * | UnsafePointer\<Type\> |
| Type * __strong * | UnsafeMutablePointer\<Type\> |
| Type ** | AutoreleasingUnsafeMutablePointer\<Type\> |
| RetType (*)(ArgType) | CFunctionPointer\<ArgType -> RetType\> |

* 若指针类型中 `<Type>` 为 `<Void>`，则它可以指向任何类型。
* 以**常量指针**为参数的函数可以接受以下值：空指针 `nil`，常量指针、变量指针或自动释放指针，相应类型的值前加 `&` 取地址，相应类型的数组。
* 以**变量指针**为参数的函数可以接受以下值：空指针 `nil`，变量指针，相应类型的值前加 `&` 取地址，相应类型的数组前加 `&`。
* 以**自动释放指针**为参数的函数可以接受以下值：空指针 `nil`，自动释放指针，原始类型的拷贝前加 `&`。

### 全局变量

* C 和 Objective-C 源文件中的全局变量将被自动导入 Swift。

### 预处理指令

1. Swift 编译器中不包含预处理器，因此宏（macros）没有被引入 Swift。
2. 在 Swift 中可以使用构建配置（build configurations）进行条件编译，构建配置包括字面值 `true` 和 `false`，命令行标志（flags），以及平台测试函数。其中命令行标志可通过 ` -D <#flag#>` 指定。

| 平台测试函数 | 有效参数 |
| ---- | ---- |
| os() | OSX, iOS |
| arch() | x86_64, arm, arm64, i386 |

```swift
#if buildConfiguration && !buildConfiguration
    statements
#elseif buildConfiguration
    statements
#else
    statements
#endif
```


## 在同一个工程中使用 Swift 和 Objective-C

* Swift 对 Objective-C 的兼容性支持在同一个工程中同时使用两种语言，因此可以用这种叫做 mix and match 的特性来开发基于混合语言的应用。

### 在同一个 App Target 中导入

* 将 Objective-C 代码导入到 Swift 时，需要依赖 Objective-C 的桥接头文件（bridging header）。当添加 Swift 文件到现有的 Objective-C 应用（或反之）时，Xcode 会自动创建这些头文件，名为 `ProductModuleName-Bridging-Header.h`。在桥接头文件中，可以 `#import` 任何想暴露给 Swift 的头文件。
* 将 Swift 代码导入 Objective-C 时，需要 `#import` Xcode 自动生成的头文件，它声明了所有 Swift 中定义的接口。该文件可以看作 Swift 代码的 umbrella header，名为 `ProductModuleName-Swift.h`。

| - | 导入到 Swift | 导入到 Objective-C  |
| ---- | ---- | ---- |
| Swift 代码 | 不需要 import | `#import "ProductModuleName-Swift.h"` |
| Objective-C 代码 | 不需要 import，但需要 Objective-C bridging header | `#import "Header.h"` |

### 在同一个 Framework Target 中导入

* 首先确认已经将 `Build Settings > Packaging > Defines Module` 设置为 `Yes`。
* 要将 Objective-C 代码导入到 Swift 中，需要在 Objective-C 的 umbrella header（即 `ProductModuleName.h`）中 `#import` 相应的头文件。
* 要将 Swift 代码导入到 Objective-C 中，只需 `#import` Xcode 自动生成的头文件，名为 `<ProductName/ProductModuleName-Swift.h>`。

| - | 导入到 Swift | 导入到 Objective-C  |
| ---- | ---- | ---- |
| Swift 代码 | 不需要 import | `#import <ProductName/ProductModuleName-Swift.h>` |
| Objective-C 代码 | 不需要 import，但需要 Objective-C umbrella header | `#import "Header.h"` |

### 导入外部框架

* 导入到 Swift 时使用 `import FrameworkName`
* 导入到 Objective-C 时使用 `@import FrameworkName;`

### 在 Objective-C 中使用 Swift

* 将 Swift 导入 Objective-C 之后，便可以用 Objective-C 的语法访问 `@objc` 修饰的类和协议。
* Swift 的独有特性无法在 Objective-C 中使用，包括：泛型、元组、枚举、结构体、顶层函数、全局变量、类型别名、可变参数、嵌套类型、柯里化函数（curried functions）。
* 为避免循环引用，不要把 Swift 导入到 Objective-C 头文件中，但可以用 `@class SwiftClass;` 前向声明（forward declare）一个 Swift 类来使用它。
* Objective-C 的类不可以继承自 Swift。

### Product Module Name

* 默认的 Product Module Name 跟 Product Name 相同，不过如果其中如果有非字母数字的字符，它们将会被 `_` 替代。
* 也可以在 `Build Settings > Packaging > Product Module Name` 中自定义名称。

---

[\<Prev\> Swift 学习笔记（二）](/swift-notes-2/) | n/a \<Next\>