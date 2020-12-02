原文地址：https://github.com/apple/swift/blob/main/docs/DynamicCasting.md#dynamic-casting-behavior

# Dynamic Casting Behavior

* Author: [Tim Kientzle](https://github.com/tbkka)
* Implementation: [apple/swift#29658](https://github.com/apple/swift/pull/29658)

## Introduction

Swfit提供了三个转型操作符：`is` , `as?` 和 `as!`。每个操作符左侧放置实例，右侧放置类型表达式。

* 转型检查操作符`is`，用来检查特定实例是否可以转换为特定目的类型。它会返回布尔值结果。
* 带有条件的转型操作符as?会尝试转换类型并返回一个Optional结果：如果转换失败会返回nil，其他情况会返回一个包含结果的非空类型可选值。
* 强制转型操作符`as!`无条件的执行转型操作并返回结果。如果`as!`表达式失败了，这次操作可能会导致系统异常中止。

注意：静态强制操作符`as`扮演不同的角色，在本文中，它的行为不会具体讨论。

以下不变性（invariants）涉及三个转型操作符：

* 转型检查：`x is T == ((x as? T) != nil)`
* 条件转型：`(x as? T) == (x is T) ? .some(x as! T) : nil`
* 强制转型：`x as! T`与`(x as? T)!`相同

特别要注意的是，`is`和`as!`可以以`as?`的形式实现，反之亦然。

与其他名字中带有`!`的操作符一样，`as!`仅在程序员预先知道可以转换成功的情况中适用。如果转换失败了，之后的行为可能就完全无法确定了。有关这种不确定性可能很重要的具体案例，请参阅下文关于数据转型的讨论。

后续部分详细介绍了管理特定Swift类型转换操作规则。除非另有说明，在以下各部分描述的类型之间的转型总是失败的。比如将 struct 实例转换为函数类型，反之亦然。

在可能的情况下，每个部分都包含机器可验证的invariants，这些invariants可以用作为此功能开发健壮测试组建的基础。

## Identity Cast

将类型的实例转型为自己的类型总是成功的，并且返回没有改变的原始值。
```
let a: Int = 7
a is Int // true
a as? Int // Succeeds
a as! Int == a // true

```

## Class and Foreign Types

类类型通常遵循标准的面向对象的转型规则。 Objective-C和CoreFoundation（CF）类型遵循Objective-C预期的行为。

### Classes

类类型之间进行转型遵循标准的面向对象的编程规则：

* 类向上转型：如果`C`是`SuperC`的子类并且`c`是`C`的实例，那么`c is SuperC == true`并且`(c as? SuperC) != nil`。（注意：这里的“向上转型”不会改变内存中的表现（representation））。然而，当通过变量或`SuperC`类型的表达式访问`c`时，仅在`SuperC`上定义过的方法和实例变量是可用的。
* 类向下转型：如果`C`是`SuperC`的子类并且`sc`是`SuperC`的实例。仅当`sc`确实是`C`的实例时，`sc is C`返回true。（注意：同样，向下转换不会影响内存中的表现。）
* Objective-C 类转型：当使用`@objc`属性定义的类或继承自Objective-C类时，上述规则也适用。
* Class向AnyObject转型：任何类引用都可以转型为`AnyObject`，然后再转型回原始类型。详情见后面“AnyObject”。
* 如果结构体和枚举类型遵守`_ObjectiveCBridgeable `协议，然后，可以将关联的桥接类型的类强制转换为struct或enum类型，反之亦然。详情见后面“The `_ObjectiveCBridgeable` Protocol”。

Invariants:

* 对于任意两种类类型`C1`和`C2`：仅当`((c as? C1) as? C2) != nil`时，`c is C1 && c is C2`。
* 对于任意类类型`C`：仅当`(c as! AnyObject) is C`时，`c is C`。
* 对于任意类类型`C`：如果`c is C`，那么`(c as! AnyObject) as! C === c`

### CoreFoundation types

* 如果`CF`是CoreFoundation类型，`cf`是一个`CF`的实例，并且`NS`是对应的Objective-C类型，那么`cf is NS == true`。
* 再进一步，因为每个Objective-C都继承自`NSObject`，所以`cf is NSObject == true`。
* 基于此前的结论，如果`T`是某些其他类型并且`cf is NS == true`,仅当`cf is T`时，`cf as! NS is T`。

上述的意图是将CoreFoundation类的实例同时视为相对应的Objective-C类的实例，以此来保证这些类的二重性。特别指出，如果在Objective-C类型上声明了protocol conformance，则可以将CoreFoundation类型的实例直接转换为协议类型。
 
XXX TODO:如果相反呢?如果ObjC实例等价于CF并且CF类型被扩展，…？

### Objective-C types

针对以下三种情况进行接下来的讨论：

* 使用`is`，`as?`和`as!`操作符进行显式转换。
* Swift向Objective-C隐式转换：当Swift代码通过非Objective-C类型的参数调用Objective-C的函数或方法时，或者Swift函数将值返回给Objective-C调用者时，这些转换是自动进行的。
* Objective-C向Swift隐式转换：当参数从Objective-C调用者传递给Swift函数时，或者当Objective-C将值返回给Swifter调用者时，这些转换是自动进行的。除非另有说明，否则以下所有情况均同样适用于上述所有三个情况。

Swift和Objective-C之间的显式转型通常符合之前描述的类类型的通用规则。同样的，类实例向Objective-C协议类型的强制转型符合协议类型转型的通用规则。

XXX TODO：解释Objective-C类型向Swift类型的隐式转换。

如上所述，CoreFoundation 类型可以与Objective-C类型之间强制转型。

Objective-C类和协议：

* 仅当`T.self is NSObject.Type`时，`T`是Object-C的类类型。
* 仅当XXX TODO XXX时，`P`是Objective-C协议。

### The `_ObjectiveCBridgeable` Protocol

`_ObjectiveCBridgeable`协议允许某些类型选择自定义转型行为。请注意，尽管该机制是为简化Swift与Objective-C的交互而专门设计的，但它不一定与Objective-C绑定。

`_ObjectiveCBridgeable`协议定义了一个关联的引用类型`_ObjectiveCType`，并提供了与关联的`_ObjectiveCType`进行相互转型的一系列方法。这个协议允许库代码提供专有的机制来将Swift类型转型为引用类型。

注意:关联的`_ObjectiveCType `被约束为`AnyObject`的子类型。它并不局限于现在的Objective-c类型。特别是，该机制同样适用于非Apple平台上的Foundation的Swift实现和Apple平台上的Objective-C Foundation。

例子1:Foundation通过让NSArry遵守`_ObjectiveCBridgeable`扩展了标准库中的`Array`类型。这允许Swift数组与Foundation `NSArray`实例相互转型。

```
let a = [1, 2, 3] // Array<Int>
let b = a as? AnyObject // casts to NSArray
```

例子2:Foundation通过让NSNumbers遵守`_ObjectiveCBridgeable`也扩展了Swfit的数字类型

```
let a = 1 // Int
// After the next line, b is an Optional<AnyObject>
// holding a reference to an NSNumber
let b = a as? AnyObject
// NSNumber is bridgeable to Double
let c = b as? Double

```

## Other Concrete Types

除了前面描述的类类型外，Swift还有其他几种具体类型。
在下面描述的不同类型之间进行转型（例如将枚举转换为元组）总会失败。

### Structs and Enums

不可以在具体的结构体或枚举类型之间进行转型。具体来说：

* 如果`S`和`T`是结构体或枚举类型并且`s is S == true`，仅当`S.self == T.self`时，`s is T`。

如果结构体或枚举类实现了刚才提到的`_ObjectiveCBridgeable `协议，那么它们可以向类类型转型。

### Optionals

在可选类型之间进行转型将根据需要显式（transparently）解包可选选项，包括嵌套的可选类型。

* 如果`U`和`T`是两个任意的类型,`t`是`T`的实例。那么`.some(t) is Optional<U> == t is U`
* 可选项注入（Optional injection）：如果`T`和`U`是两个任意的类型，`t`是`T`的非空实例，那么`t is Optional<U> == t is U`。
* 可选项推测（Optional projection）：如果`T`和`U`是两个任意的类型，`t`是`T`的实例，那么`.some(t) is U == t is U`。
* nil转型（Nil Casting）：如果`T`和`U`是两个任意的类型，那么`Optional<T>.none is Optional<U> == true`。

注意：针对上面提到的“可选项注入”，`t`是`T`的非空实例这一前提也就暗示着`T`不是`Optional`或者`T`是`Optional`并且`t`支持`.some(u)`形式。

例子

```

// T和U是任意两种不同的类型（可能是可选类型）
// NO是任意非可选类型
let t: T
let u: U
let no: NO
// 转型自由添加/删除 optional wrappers
t is Optional<T> == true
t is Optional<Optional<T>> == true
Optional<T>.some(t) is T == true
Optional<Optional<T>>.some(.some(t)) is T == true
// 仅当他们可以转型成内部类型时，非可选可以转型成可选类型。
no is Optional<U> == no is U
Optional<NO>.some(no) is Optional<U> == no is U
// 仅当内部值可以转型成其他类型时，非空可选值可以转型成相应类型
Optional<T>.some(t) is U == t is U
Optional<Optional<T>>.some(.some(t)) is U == t is U
// Nil可选值总是可以转型成其他可选类型
Optional<T>.none is Optional<U> == true
// Nil可选值永远不能转型成非可选类型。
Optional<T>.none is NO == false

```

深度保护（Depth Preservation）：上面的规则暗示嵌套的可选内容可以显式解包到任意深度。例如，如果一个`T`类型的实例可以转型为`U`类型，那么`T????`可以转型成`U????`。如果前者包含`nil`值，结果是不确定的。将`nil`转型成`U????`可能会以`.none`，`.some(.none)`,`.some(.some(.none))`,`.some(.some(.some(.none)))`中的任意值结束。

为了解决这种不确定情况，实现应首先检查内部`.none`值的类型并计算可选项深度（optional depth）。请注意，这里的“可选项深度”是指从最内层的非可选项类型到`.none`类型所需要的可选项解包器的数量。

1. 如果目标允许，则该值应以与源相同的可选深度注入目标。
2. 否则，该值应以最大的可选深度注入。

例子

```
// 深度保护
// 这里的`.none`类型为`T？`，深度为1
let t1: T???? = .some(.some(.some(.none)))
// 这里的`.none`类型为`T????`，深度为4
let t4: T???? = .none
// 可选项深度2，匹配
t1 as! U???? // Produces .some(.some(.some(.none)))
t1 as! U??? // Produces .some(.some(.none))
t1 as! U?? // Produces .some(.none)
t1 as! U? // Produces .none
// 可选项深度2，这里不可能是4
t4 as! U?? // Produces .none
// 请记住`as?`添加了一层可选操作
// 因为外层是`.some`，这些可以转型成功。
t1 as? U???? // Produces .some(.some(.some(.some(.none))))
t1 as? U??? // Produces .some(.some(.some(.none)))
t1 as? U?? // Produces .some(.some(.none))
t1 as? U? // Produces .some(.none)
t4 as? U?? // Produces .some(.none)

```

### Array/Set/Dictionary Casts

对于Array，Set或Dictionary类型，可以使用转型操作符将其转换为具有不同元素类型的同一外部容器的另一个实例（分别为Array，Set或Dictionary）。注意，接下来的讨论只针对这几个特殊类型。这不适用任何其他类型，也没有任何机制将这种行为添加到其他类型上。

例如：存在一个`Array<Int>`类型的`arr`，你可以转型`arr as? Array<Any>`。将会得到一个新数组。其中原始数组中的每个`Int`都被单独转型为`Any`。

然而，如果任何一个元素不能转型，外层的转型会失败。例如，思考下面的例子：

```

let a: Array<Any> = [Int(7), "string"]
a as? Array<Int> // Fails because "string" cannot be cast to `Int`

```

具体来说，转型操作符对`Array`的操作与下面的实现类似。特别需要注意的是，一个空数组可以成功转型为任意一个目标数组类型。

```
func arrayCast<T,U>(source: Array<T>) -> Optional<Array<U>> {
  var result = Array<U>()
  for t in source {
    if let u = t as? U {
      result.append(u)
    } else {
      return nil
    }
  }
  return result
}

```

Invariants

* 仅当数组内容这样操作时，数组可以转型：如果`t`是`T`的实例并且`U`是任意类型，那么`t is U == [t] is [U]`。
* 空数组随意转型：如果`T`和`U`是任意类型，那么`Array<T>() is Array<U>`。

相同的逻辑也同样适用于`Set`和`Dictionary`的转型。注意，如果集合组成部分的转型操作包括将原集合中的不相等元素转换成目标中的相等元素，则结果`Set`或`Dictionay`中的元素数量可能减少。

具体来说，转型操作符对Set和Dictionary的操作与下面的代码类似：

```
func setCast<T,U>(source: Set<T>) -> Optional<Set<U>> {
  var result = Set<U>()
  for t in source {
    if let u = t as? U {
      result.append(u)
    } else {
      return nil
    }
  }
  return result
}

func dictionaryCast<K,V,K2,V2>(source: Dictionary<K,V>) -> Optional<Dictionary<K2,V2>> {
  var result = Dictionary<K2,V2>()
  for (k,v) in source {
    if let k2 = k as? K2, v2 = v as? V2 {
      result[k2] = v2
    } else {
      return nil
    }
  }
  return result
}

```

#### Collection Casting performance and `as!`

对于`as?`转型，之前提到的转型操作要求每个元素都可以单独转换。当尝试在Swift和Objective-C的代码之间共享大型容器时，这样的转换操作可能带来性能瓶颈。

然而，对于`as!`转型，程序员有责任在请求转换之前确保操作成功。可以通过在相关元素需要的时候再把内部组件转型来满足刚才的要求，尽量（但是不强制）这么做。在（程序员）知道数据安全且内部组件转换重要的前提下，这种懒转换（lazy conversion）可以大幅提高性能。然而，如果转换不能完成，则无法确定转型请求会立即失败还是程序在稍后的某个时刻失败。

### Tuples

仅在以下列举的情况成立时，元组类型T1可以转型成元组类型T2。

* T1和T2有相同数量的元素
* 如果元素在T1和T2中都有标签，则标签相同。
* T1的每个元素都可以单独转型为T2的相关类型。

### Functions

仅在以下列举的情况成立时，函数类型F1可以转型成函数类型F2。

* 两个类型有相同数量的参数
* 对应的参数有相同的类型
* 返回类型是相同的
* 如果F1是一个抛出异常的函数类型，那么F2必须是一个抛出异常的函数类型。如果F1不能抛出异常，那么F2可以抛出异常也可以不抛出异常。

注意，仅仅是参数和返回值是可转型，还不够，他们必须完全相同。

## Existential Types

从概念上讲，“existential type”是隐式包装器，持有一个类型和一个类型实例。各种existential type在他们可以持有的类型（例如，`AnyObject`只能携带引用类型）和容器对外暴露的能力（`AnyHashable`暴露相同性检测和哈希）上有所不同。

existential types `E`的关键不变式如下：

* Strong existential invariant：如果`t`是任意实例，`U`是任意类型，并且`t is E`那么`(t as! E) as? U`与`t as? U`返回相同结果。

直观上简单点说，如果你可以将实例`t`放入`E`中，那么你可以通过转型将其取出。对于Equatable类型，这意味着当两个操作成功时，他们的结果是相等的。这也就意味着如果其中任何一个`as?`转型失败，另一个也会失败。

`AnyObject`和`AnyHashable`没有完全满足刚才提到的strong invariant。相反，它们满足了以下weakder版本：
* Weak existential invariant：如果`t`是任意实例，`U`是任意类型，并且
`t is U`和`t is E`，那么`(t as! E) as? U`与`t as? U`返回相同的结果。

### Objective-C Interactions

strong existential invariants与weak existential invariants的情况不同是因为转型为`AnyObject `或`AnyHashable `会触发Objective-C桥接转换。这些转换结果可能扩展了原始类型的转型结果。因此，即使`t`不能单独直接转型，`(t as! E)`可能转型成`U`。

一个例子是Foundation的`NSNumber`类型，它有条件地（conditionally）与Swift数字类型桥接。因此，当Foundation在作用域中时。`Int(7) is Double == false`但是`(Int(7) as! AnyObject) is Double == true`。通常来说，添加从一种类型到几种不同类型的桥接行为的能力也就意味着Swift的转型是不能传递的。

### Any

任意Swift实例可以转型成`Any`类型。`Any`实例没有可用的方法或属性。为了使用其内容，必须要将其转型成其他类型。每个类型标识符(type identifier)都是元类型`Any.Type`的实例。

Invariants

* 如果`t`是实例，那么`t is Any == true`
* 如果`t`是实例，那么`t as! Any`总是成功的
* 对于任何类型（包括协议类型）,`T.self is Any.Type`
* Strong existential invariant:如果`t`是任意实例并且`U`是任意类型，那么`(t as! Any) as? U`会返回与`t as? U`相同的结果。

### AnyObject

任意类、枚举、结构体、元组、函数、元类型或者existential元类型实例都可以转型成`AnyObject`。

转型成其他类型可以访问`AnyObject`容器的内容：

* Weak existential invariant:如果`t`是任意实例，`U`是任意类型，`t is AnyObject`，并且`t is U`，那么`(t as! AnyObject) as? U`将返回与`t as? U`相同的结果

#### Implementation Notes

`AnyObject`在内存中表示为指向引用计数对象的指针。对象的动态类型可以从对象的“isa”字段中恢复。可选项形式`AnyObject?`除了允许null，其他都与`AnyObject `相同。

引用类型(类、元类型、或者existential元类型实例)可以不通过任何转换直接指向`AnyObject`。

对于非引用类型，包括结构体、枚举、元组类型，转型逻辑需要先检查`_ObjectiveCBridgeable `一致性，它可以将源转换为专门的引用类型。（可以在此前的“The _ObjectiveCBridgeable Protocol”了解更多细节。）

如果桥接失败，值将会被拷贝到一个与Objective-C内存管理兼容的不透明堆容器。这个容器的唯一作用是允许Swift类型可以与`AnyObject`相互转型，以便它们可以与需要调用Swift代码的Objective-C函数共享。Objective-C代码不能使用这个容器的内容；它只能存储引用并将其传递回Swift函数。

针对不支持Objective-C运行时的平台，内部容器类型有另一种实现来支持与`AnyObject`相同的转型操作。

XXX TODO：仅当协议类型与`__SwiftValue `兼容的情况下，运行时逻辑才会生成将它们转型成`AnyObject`的代码。此逻辑的实际效果如何？这是否意味着将协议类型转型成1AnyObject1有时会解包（如果协议不兼容），有时不会解包？哪些协议受此影响？

### Error (SE-0112)

`Error`类型的行为与用于转型的ordinary existential type类似。

（有关`Error`协议的更详细信息可以看之后的“Note: 'Self-conforming' protocols”）

### AnyHashable (SE-0131)

出于转型的目的，`AnyHashable`的表现与existential type类似。它满足之前提到的weak existential invariant。

然后注意，在其他方面`AnyHashable `的行为与existential type不同。例如，它的元类型名为`AnyHashable.Type`，并且它没有existential元类型。

### Protocol Witness types

任何协议定义（那些包含associatedtype属性或使用`Self`别名的协议定义除外）都具有以该协议命名的关联的existential type。

具体来说，，假设你定义了一个协议

```
protocol P {}
```

这个定义会产生一个existential type（也叫做`P`）。因为它详细地暴露了协议定义的功能，所以这种existential type也被称为“protocol witness type”。`P`实例不能访问类型`T`的其他功能。仅当`T`遵守协议`P`时，具体类型`T`的任意Swift实例可以转型为类型`P`。

转型后的其他适当的类型可以访问protocol witness的内容：

* Strong existential Invariant：对于任意协议`P`，实例`t`，类型`U`，如果`t is P`，那么`t as? U`会返回与`(t as! P) as? U`相同的结果。

除了the protocol witness type`P`之外，每个Swift协议`P`隐式定义了两个其他类型：`P.Protocol`是`protocol metatype`，`P.self`的类型。`P.Type`是“protocol existential metatyp“。后续有相关的详细描述。

关于协议转型和可选

当可选类型转型成协议类型时，如果可能，可选属性可能被保留。存在某个`Optional<T>`类型的实例`o`和某个协议`P`，转型请求`o as? P`将根据`Optional<T>`是否直接遵守协议`P`而返回不同的结果。

* 如果`Optional<T>`遵守协议`P`，那么转型结果将会是包装`o`实例的protocol witness。在这种情况下，后续对`Optional<T>`的转换将会恢复原始实例。特别指出，这种情况会保留nil实例。
* 如果`Optional<T>`不直接遵守协议`P`，那么`o`将会被解包，并尝试使用包含（contained）对象进行转型。如果`o == nil`，也就意味着失败了。一个嵌套的可选`T???`在这种情况下会导致内部的非可选完全解包。
* 如果以上所有方法均失败，也就意味着转型失败。

例如，`Optional `遵守协议`CustomDebugStringConvertible`,但是不遵守`CustomStringConvertible `。将一个可选实例转型成第一类协议，将会产生一个实例，它的`.debugDescription`属性描述可选实例。将一个可选实例转型成第一类协议，将会产生一个实例，它的`.description `属性描述内部非可选实例。

## Metatypes

Swift支持两种类型的元类型：除了许多其他语言支持的常规元类型外，它还具有可用于访问静态协议要求的 existential 元类型

### Metatypes

对于任何类型`T`，都有唯一实例`T.self`在运行时表示类型。与所有实例相同，`T.self`有一个类型。我们可以称这种类型为“`T`的元类型”。从技术上讲，类型的静态变量或方法归属于`T.self`实例，并由`T`的元类型决定：

例子：

```
struct S {
  let ivar = 2
  static let svar = 1
}
S.ivar // Error: only available on an instance
S().ivar // 2
type(of: S()) == S.self
S.self.svar // 1
S.svar // Shorthand for S.self.svar
```

针对大多数Swift类型，`T`的元类型可以用`T.Type`表示。然后，在以下情况中，元类型有不同的命名方法：

* 对于nomial协议类型（nominal protocol type）`P`，元类型用`P.Protocol`命名。 
* 对于无协议existential类型（non-protocol existential types）`E`，元类型也可以命名为“E.Protocol”。比如`Any.Protocol`，`AnyObject.Protocol`和`Error.Protocol`。
* 对于一个绑定到泛型变量`G`的类型，即使`G`被绑定到协议或者existential，其元数据也用`G.Type`命名。特别注意，如果`G`被绑定到nomial协议类型`P`，那么`G.Type`是元数据`P.Protocol`的另一个名称。因此`G.Type.self == P.Protocol.self`。
* 如上所述，尽管`AnyHashabl`在某些方面的行为类似于existential type，但其元类型称为`AnyHashable.Type`。

例子：

```
// Metatype of a struct type
struct S: P {}
S.self is S.Type // always true
S.Type.self is S.Type.Type // always true
let s = S()
type(of: s) == S.self // always true
type(of: S.self) == S.Type.self

// Metatype of a protocol (or other existential) type
protocol P {}
P.self is P.Protocol // always true
// P.Protocol is a metatype, not a protocol, so:
P.Protocol.self is P.Protocol.Type // always true
let p = s as! P
type(of: p) == P.self // always true

// Metatype for a type bound to a generic type variable
f(s) // Bind G to S
f(p) // Bind G to P
func f<G>(_ g: G) {
   G.self is G.Type // always true
}
```

Invariants

* 对于nominal non-protocol type `T`，`T.self is T.Type`
* 对于nominal protocol type `p`，`P.self is P.Protocol`
* `P.Protocol`是一个单例，仅当`T`确实是`P`时，`T.self is P.Protocol`。
* 仅当`T.self is P.Type`时，无协议类型`T`遵守协议`P`。（如果`T`是协议类型，可以在“Self conforming existential types”了解更多细节）
* 仅当`T.self is U.Type`时，`T`是无协议类型`U`的子类型。
* 子类型定义元类型子类型：如果`T`和`U`都是无协议类型，`T.self is U.Type == T.Type.self is U.Type.Type`
* 子类型定义元类型子类型：如果`T`是无协议类型并且`P`是协议类型，`T.self is P.Protocol == T.Type.self is P.Protocol.Type`

### Existential Metatypes

协议可以为类型的实例指定约束并提供默认实现。它们还可以为类型的静态成员指定约束和提供默认实现。如之前所述，将类型的常规实例转型为协议类型将产生一个protocol witness实例，该实例仅对外暴露协议所需或提供的那些功能。类似地，可以将类型标识符实例（`T.self`）转型为协议的“existential metatype”，仅对外暴露与协议的静态需求（static requirements）相对应的类型部分。

协议`P`的existential metatype被称为`P.Type`。（回想一下，对于无协议类型`T`，表达式`T.Type`引用元类型。无协议类型没有existential metatypes。对于泛型变量`G`，即使这个泛型变量绑定到一个协议，这个表达式也引用原始元类型。Swift中没有通过泛型变量引用existential metatype的机制。）

例子

```
protocol P {
   var ivar: Int { get }
   static svar: Int { get }
}
struct S: P {
   let ivar = 1
   static let svar = 2
}
S().ivar // 1
S.self.svar // 2
(S() as! P).ivar // 1
(S.self as! P.Type).svar // 2
```

Invariants

* 如果`T`遵守协议`P`并且`t`是`T`的一个实例，那么`t is P`并且`T.self is P.Type` 
* 如果`P`是`P1`的子协议并且`T`是任意类型，那么`T.self is P.Type`就意味着`T.self is P1.Type`
* 由于每个类型`T`都遵守协议`Any`，`Any.Type`始终为true。
* 由于每个类类型`C`都准守协议`AnyObject`，`C.self is AnyObject.Type`始终为true(这也包括Objective-C的类类型)

### Note: "Self conforming" existential types

如上所述，`P`的协议定义隐式定义了类型`P. type`(the existential metatype)和`P. protocol`(元类型)。它还定义了一个容器类型的关联类型`P`，它可以容纳任何遵守协议`P`的具体类型。

如果容器类型`P`遵守协议`P`，那么这个协议就是“self conforming”。换句话说`P.self`是`P.Type`的一个实例。（请记住，`P.self`永远是`P.Protocol`的一个实例）

这是Swift注意的问题，因为以下结构尝试在已有具体实例显示遵守协议`P`的情况下，以`P`类型调用泛型`f`:

```
func f<T:P>(t: T) { .. use t .. }
let a : P = something
f(a)
```
这个构造只有在`T = P`,`T`遵守`P`时才有效（This construct is valid only if `T` conforms to `P` when `T = P`）。也就是说，如果`P`self-conforms。

泛型类型也会出现类似情况：

```
struct MyGenericType<T: P> {
  init(_ value: T) { ... }
}
let a : P
let b : MyGenericType(a)
```

如上所述，由于`a`的类型是`P`，此代码使用`T = P`实例化`MyGenericType`,仅当`P`遵守`P`时才有效。请注意，任何指定静态方法、静态属性、关联类型或初始化器的协议都不可能是self-conforming。

尽管上述是专门针对协议的讨论，但它同样适用于其他existential types。在Swift 5.3中，唯一self-conforming existential types是Any、Error和Objective-C协议，它们没有静态需求（static requirements）。

Invariants

* `Any` self-conforms: `Any.self is Any.Type == true`
* `Error` self-conforms: `Error.self is Error.Type == true`
* 如果`P` self-conforms并且是`P1`的子协议, 那么`P.self is P1.Type == true`

例如，这里的最后一个Invariant意味着对于没有静态要求（static requirements）的任何Objective-C协议`OP`，`OP.self is AnyObject.Type`。这源于这样一个事实，即`OP`self-conforms ，而且每个Objective-C协议都将`AnyObject`作为隐式父协议。

## Implementation Notes

源代码中出现的转型运算符会根据所涉及类型的详细信息以多种方式转换为SIL。一个通用的路径体现在`as!`转换成一些`unconditional_checked_cast `指令的变体和`as`与`is`转换成条件分支指令如`checked_cast_br`。

SIL优化试图简化这些步骤。在某些情况下，转型的结果在编译期就可以完全确定。在某些情况下，编译器可以生成专门的代码来完成转型（例如，指派对`AnyObject`变量的引用）。在其他情况下，编译器可以确定转型是否会成功，但不一定能够计算结果。例如，类引用转型为父类将总是成功的，它可能允许将`checked_cast_br`指令简化为非分支的`unconditional_checked_cast`。同样的，一个静态证实总是会失败的`as?`转型，可以被简化为固定的`nil`值。注意，这些SIL函数也被用作其他目的，包括在调用Objective-C时隐式桥接环转。即使`-Onone`模式中优化被限制的情况下，这也是非常常见的。

当SIL被转换为LLVM IR后，其余的转型操作将以对恰当运行时函数调用的形式呈现。最常见的这类函数是`swift_dynamicCast`，接收一个指向输入指针，存储结果的地址和描述所涉及类型的元数据。可能的话，编译器更趋向于发起对该函数的特殊变体的调用，其名称如`swift_dynamicCastClass`和`swift_dynamicCastMetatypeToObject`（完整的列表在[运行时文档](https://github.com/apple/swift/blob/main/docs/Runtime.md)中）。这些定制版本需要更少的参数并执行更少的内部检查，这使它们使用起来开销更少。

`swift_dynamicCast`函数检查输入和输出类型以确定恰当的转换。此过程对输入类型进行递归，以检查existential容器或`可选`的内容。如果输出是existential容器或者`Optional`类型，它也会递归输出类型，以确定合适的基本转换，然后必须为内容配置恰当的容器。它也可能执行额外的元数据查找，以确定是否存在符合`Hashable `，`Error `或`ObjectiveCBridgeable `协议的类型。对于集合，可能最终以递归方式尝试转换每个元素。

## Compared to Swift 5.3

这是Swift 5.3与上面描述的行为的一些不同之处：

* 与以前的静态源类型相比，目标类型是“更可选（more Optional）”的转型操作会产生错误。这就禁止了下面的所有操作：将`Int`注入`Int?`，从某个不透明的`Any`容器中提取`Int?`，`Array<Int>`转型成`Array<Int?>`。本文允许所有这些操作。

```
let a = 7
// Swift 5.3: error: cannot downcast to a more optional type
// Specification: returns true
a is Int?
// Swift 5.3: error: cannot downcast to a more optional type
// Specification: returns false
a is Optional<Double>

let b: Int? = 7
let c: Any = b
// Swift 5.3: error: cannot downcast to a more optional type
// Specification: returns true
c is Int?

```

* CoreFoundation类型的实例有时可以被转型为对应Objc-C类型上定义的协议，有时不可以。为了保持行为一致，我们必须选择其中之一；转型成功似乎更吻合一般意义上Objc-C/CF类型的双重性。

```
import Foundation
protocol P {}
extension NSString: P {}
let a = CFStringCreateWithCString(nil, "hello, world",
                CFStringBuiltInEncodings.UTF8.rawValue)
// Swift 5.3: prints "true"
print(a is P)
let b: Any = a
// Swift 5.3: prints "false"
// Specification: prints "true"
print(b is P)
```

* Swift5.3编译器断言试图将结构体转型为AnyObject

```
struct S {}
let s = S()
// Swift 5.3: Compiler crash (in asserts build)
// Specification:  Succeeds via __SwiftValue boxing
s as? AnyObject
```

* 在未优化的版本中，`NSNumber()`不能通过`as?`转型成它自身

```
import Foundation
let a = NSNumber()
// true in 5.3 for optimized builds; false for unoptimized builds
print((a as? NSNumber) != nil)
```

* `Optional<NSNumber>`不能推测(project)

```
import Foundation
let a: Optional<NSNumber> = NSNumber()
// Swift 5.3: false
// Specification: true
print(a is NSNumber)
// Swift 5.3: nil
// Specification: .some(NSNumber())
print(a as? NSNumber)
```

* NSNumber()在运行时转型成`Any`会崩溃

```
import Foundation
let a = NSNumber()
// Swift 5.3: Runtime crash (both optimized and unoptimized builds)
// Specification: Succeeds
print(a is Any)
```

* SR-2289: CF类型不能被转型成协议existentials

```
import Foundation
protocol P {}
extension CFBitVector : P {
  static func makeImmutable(from values: Array<UInt8>) -> CFBitVector {
    return CFBitVectorCreate(nil, values, values.count * 8)
  }
}
// Swift 5.3: Crashes in unoptimized build, prints true in optimized build
// Specification: prints true
print(CFBitVector.makeImmutable(from: [10,20]) is P)
```

* `Optional<T> as Any`不能转型成协议类型。请注意，这对于weak字段的反射来说是一个特殊的问题，因为`Mirror`会将那些包含`Optional`的字段反射为`Any`。

```
protocol P {}
class C: P {}
let c: C? = C()
let a = c as? Any
// Swift 5.3: prints "false"
// Specification: prints "true"
print(a is P)
```

* SR-8964:  包含`Optional<Any>`的`Any`不能转型成`Error`

```
struct MyError: Error { }
let a: Any? = MyError()
let b: Any = a
// Swift 5.3: Prints false
// Specification: prints true
print(b is Error)
```

* SR-6126: 嵌套可选项结果导致不一致

```
// Note: SR-6126 includes many cases similar to the following
let x: Int? = nil
print(x as Int??) // ==> "Optional(nil)"
// Swift 5.3: prints "nil"
// Specification: should print "Optional(nil)" (same as above)
print((x as? Int??)!)
```

* `Error.self`不能完全self-conform

```
// Swift 5.3: Prints "false"
// Specification: prints "true"
print(Error.self is Error.Type)
```
* Objective-C协议元类型不能完全self-conform

```
import Foundation
let a = NSObjectProtocol.self
print(a is NSObjectProtocol.Type)
```
* SR-1999: `Any`内容不能转型为协议类型

```
protocol P {}
class Foo: P {}
let optionalFoo: Foo? = Foo()
let any: Any = optionalFoo
// Swift 5.3: Prints "false"
// Specification: prints "true"
print(any as? P)
```

* XXX TODO List others
