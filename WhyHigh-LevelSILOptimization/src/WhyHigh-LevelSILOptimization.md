
```
原标题：High-Level SIL Optimization in the Swift Compiler
地址：https://benng.me/2017/08/27/high-level-sil-optimization-in-the-swift-compiler/
```

去年Matt Rajca[发现](https://www.mattrajca.com/2016/10/22/benchmarking-swift-array-append-vs-concatentation.html)Swift的 `Array.append(element)`速度比`Array operator+=(collection)`快6倍。有点可惜，因为在我们看来后者不仅与前者语义相同，而且输入更加方便，也更顺眼。

我很喜欢最近关于“如何构建一个超级简单的编译器”博客文章，但是这对于那些想要深入编程语言和编译器世界的人来说，文章并没有提供太多帮助。这就是我花了这么长时间来写这篇关于Swift编译器文章的主要动机，尽管我已经有了一些使用LLVM的经验。

这篇文章的受众是那些已经了解编译器设计和术语的基础知识，但还没有为某个流行语言的编译器优化做出贡献的人。针对那些研究Swift高阶优化的人，这篇文章会发挥它最大的价值。

这篇文章更像参考指引而不是事无巨细的教程。因为如果你想写优化器，其中有太多的细节了。[这可能要花你一辈子的时间，你可能永远也做不完](https://en.wikipedia.org/wiki/The_Art_of_Computer_Programming)。

我将讲以下内容：

1. 如何阅读SIL（本文的主要内容，因为这非常重要，并且你会大量阅读SIL）
2. 如何写一个SIL测试用例
3. 如何写一个optimization pass
4. 添加optimization pass是多么低效的

如果你想知道我对Swift的贡献是什么，那就直接跳到结尾。这是一篇很长的文章，不是能一口气看完的。

## Why High-Level SIL Optimization?

我们的目标是将`arr += [5]`替换为`arr.append(5)`。

首先，我们必须知道在哪里运行此优化项以及为什么。此前，我写过的唯一优化是在LLVIM IR上运行的LLVM optimization passes。

这种方案的问题就是函数调用已经被 lower into LLVM IR后很难被替换。在如此低的级别上，很难分辨哪些指令属于原始函数调用，这大大复杂化了optimization pass的实现，并使其非常脆弱。

接下来执行此优化项最合理的地方是在AST阶段，但是因为Swift AST只有语法分析，没有语义分析，因此我们无法安全地对其进行转换。

为了解决这个问题，Swift编译器在AST和LLVM IR之间有一个中间代码，称为SIL（Swift Intermediate Language）。
SIL，类似LLVM，是一个Static-Single-Assignment (SSA) IR。区别于LLVM IR，SIL拥有更丰富的类型系统，使用基本的block参数而不是phi node，并且对我们来说很有帮助的是，它允许使用语义属性注释函数。

## Reading SIL
让我们看一下待优化的Swift如何生成SIL。编译Swift，然后输出canonical SIL

```
// input.swift
var list = [Int]();

for _ in 0..<1_000_000 {
  list += [5] // we'll swap this to list.append(5) to compare later
}
```

```
swift-sources/build/Ninja-DebugAssert/swift-macosx-x86_64/bin/swiftc\
-frontend\
-target x86_64-apple-macosx10.9\
-sdk /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.12.sdk\
-Onone\
-emit-sil\
-o out.sil\
input.swift
```

我们只看main方法中发生了什么。我会用T, T1, T2等等来表示类型，因此Array是指由类型为T的元素组成的数组。

### The Setup

```
// main
sil @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional>) -> Int32 {
```

- `//main` 这行注释包含了接下来方法的名称，SIL的注释辅助我们理解。
- `@main` 这个方法名字是"main"，SIL通过@前缀标识名称。
- `@convention(c)` 应该使用C调用约定。调用约定指定在调用函数时如何处理参数和返回值。
- `(Int32,`方法的第一个参数是32位整数。根据协议，我们知道这个整数是给程序的参数数（we know that this integer is the number of arguments given to the program）与此等价的C语言是int argc。
- `UnsafeMutablePointer<Optional>)`这里需要展开说一下
  - `UnsafeMutablePointer`是一个指向某些T类型内容的原始指针（内存地址）。
  - `Optional`意味着要么是Some要么是None。
  - `Int8`是8-bit整数。在上下文中，这是一个字符(char)。
  - `UnsafeMutablePointer`是一个指向字符的指针。在上下文中，这是一个字符数组，也可以理解为字符串。
  - `Optional`指针要么指向一个字符串，要么指向空。
  - 将它们放在一起，这是一个以空字符结尾的字符串指针数组。它在C语言中的等价形式是char*[]。
- `-> Int32`这个方法返回一个32位的整数。根据协议，我们知道这是程序的退出代码。

```
// %0       // user: %7
// %1       // user: %14
bb0(%0 : $Int32, %1 : $UnsafeMutablePointer\<Optional\\>):
```

- `// %0 // user: %7`注释，标记寄存器%7依赖寄存器%0。SIL使用%整数表示寄存器。因为SIL，和LLVM IR一样，是一个 [Static Single Assignment(SSA)](https://en.wikipedia.org/wiki/Static_single_assignment_form)IR，你可以将这些寄存器视为典型编程语言中的常量。它们持有一个值，并且一旦设置，就不能更改。每当编译器需要一个新的寄存器时，它就直接增加寄存器编号。SSA中的不可变寄存器使优化更容易编写。严格来讲，这些都是虚拟寄存器。之后的pass通过使用最终平台（target architecture）上的一个真实寄存器来“lower”它们。如果我们进一步查看代码，我们会看到使用%0的指令确实会把返回的结果存储在%7中。寄存器编号的作用域是它们所在basic blocks的方法。
- `bb0` [basic block](https://en.wikipedia.org/wiki/Basic_block) 0。basic block是一个直线代码序列，这意味着除了入口和出口，它没有分支。
- `(%0 : $Int32`basic block的第一个参数是32位整数。假设basic block bb3具有直接的前导bb1和bb2。bb3需要引用bb1中的寄存器%7，或者bb2中的寄存器%11，这取决于是从哪个前导basic block跳转到bb3。在LLVM IR中，我们将在bb3中使用Φ（Phi）函数在％7或％11之间“选择”并将选定的值分配给新的寄存器。在SIL中，前导的basic block bb1和bb2通过在分支（br）指令中将参数传递给bb3来进行选择。
- `%1 : $UnsafeMutablePointer<Optional>)`第二个参数，上面已经解释过了。我不确定波浪号（〜）是什么意思。

```
%2 = alloc\_stack $IndexingIterator, var, name "$i$generator", loc "input.swift":3:7, scope 2 // users: %78, %53, %59
```

- `alloc_stack T`  在栈上申请（未初始化）包含T的内存，并返回申请内存的地址。
- `$IndexingIterator`我们为其申请内存控件的迭代器的类型。SIL中的此类型以$开头。

```
%3 = metatype $@thin CommandLine.Type, scope 1
// function\_ref CommandLine.\_argc.unsafeMutableAddressor
%4 = function\_ref @\_TFOs11CommandLineau5\_argcVs5Int32 : $@convention(thin) () -> Builtin.RawPointer, scope 1 // user: %5
%5 = apply %4() : $@convention(thin) () -> Builtin.RawPointer, scope 1 // user: %6
%6 = pointer\_to\_address %5 : $Builtin.RawPointer to \[strict] $\*Int32, scope 1 // user: %9
%7 = struct\_extract %0 : $Int32, #Int32.\_value, scope 1 // user: %8
%8 = struct $Int32 (%7 : $Builtin.Int32), scope 1 // user: %9
store %8 to %6 : $\*Int32, scope 1 // id: %9
%10 = metatype $@thin CommandLine.Type, scope 1
// function\_ref CommandLine.\_unsafeArgv.unsafeMutableAddressor
%11 = function\_ref @\_TFOs11CommandLineau11\_unsafeArgvGSpGSqGSpVs4Int8\_\_\_ : $@convention(thin) () -> Builtin.RawPointer, scope 1 // user: %12
%12 = apply %11() : $@convention(thin) () -> Builtin.RawPointer, scope 1 // user: %13
%13 = pointer\_to\_address %12 : $Builtin.RawPointer to \[strict] $\*UnsafeMutablePointer<Optional>, scope 1 // user: %14
store %1 to %13 : $\*UnsafeMutablePointer<Optional>, scope 1 // id: %14
%15 = tuple (), scope 1
```

这些指令会处理我们程序的命令行参数。由于我们从来没有使用这些参数，所以之后的的optimizing pass将删除这些不必要的指令。为了简明扼要，我会跳过本节。

```
alloc\_global @\_Tv3out4listGSaSi\_, loc "input.swift":1:5, scope 1 // id: %16
%17 = global\_addr @\_Tv3out4listGSaSi\_ : $\*Array~~, loc "input.swift":1:5, scope 1 // users: %21, %73
```

- `alloc_global @foo` 为全局变量@foo初始化内存。
- `global_addr @foo` 获取全局变量@foo的地址。
- `@_Tv3out4listGSaSi_`这是mangle之后的整数数组，我们之后会用到这个数组添加元素

```
// function\_ref Array.init() -> \[A]
%18 = function\_ref @\_TFSaCfT\_GSax\_ : $@convention(method)  (@thin Array.Type) -> @owned Array, loc "input.swift":1:16, scope 1 // user: %20
%19 = metatype $@thin Array~~.Type, loc "input.swift":1:12, scope 1 // user: %20
%20 = apply %18~~(%19) : $@convention(method)  (@thin Array.Type) -> @owned Array, loc "input.swift":1:18, scope 1 // user: %21
store %20 to %17 : $\*Array~~, loc "input.swift":1:18, scope 1 // id: %21  // function\_ref Array.init() -> \[A]
```

- 寄存器%18中有很多操作
  - `function_ref @foo : $T`创建一个类型为T的对函数@foo的引用。
  - `@convention(method)`特指对Swift方法调用协议。这意味着SIL函数最后将使用“ self”参数进行调用，因为它是一个实例方法。
  - `(@thin Array.Type) -> @owned Array`这个函数类型有一个元祖类型参数并且返回类型。元祖类型是类型的类型。`τ\ 0\ 0`是这个泛型函数的占位符类型。@thin表示该元祖类型不需要存储，因为它是精确类型（exact type）。@owned表示接收者负责销毁该值。
  - 将以上放在一起意味着，创建了对泛型数组的引用。初始化函数，并将其存储在寄存器%18中。
- `metatype $T.Typ`创建了一个T类型的元祖对象引用。这里我们将会得到Array type的类型的引用。请注意，这是实际类型，因为它没有任何占位符类型。
- `apply %0(%1, %2, ...) : $(A, B, ...) -> R`调用函数%0，并传入参数%1，%2等等，参数对应的类型A，B等等，并提供R类型的返回值。

```
// function\_ref Collection<A>.makeIterator() -> IndexingIterator~<A>~
%22 = function\_ref @\_TFesRxs10Collectionwx8IteratorzGVs16IndexingIteratorx\_wx8\_ElementzWxS0\_7Element\_rS\_12makeIteratorfT\_GS1\_x\_ : $@convention(method)  (@in\_guaranteed τ\_0\_0) -> @out IndexingIterator, loc "input.swift":3:14, scope 2 // user: %53
```

- 创建`Collection.makeIterator（`）函数的引用。我们在bb13才会使用它。

```
%23 = integer\_literal $Builtin.Int64, 0, loc "input.swift":3:10, scope 2 // user: %24
%24 = struct $Int (%23 : $Builtin.Int64), loc "input.swift":3:10, scope 2 // user: %43
%25 = integer\_literal $Builtin.Int64, 1000000, loc "input.swift":3:14, scope 2 // user: %26
%26 = struct $Int (%25 : $Builtin.Int64), loc "input.swift":3:14, scope 2 // user: %45
%27 = alloc\_stack $CountableRange~~, loc "input.swift":3:11, scope 2 // users: %46, %55, %50
br bb1, loc "input.swift":3:11, scope 2 // id: %28
```

- 创建整数常量0和1000000，紧接着用这些字面量创建struct类型$Int。
- 在栈上为`$CountableRange`分配空间
- 跳转到basic block 1（bb1）

```
bb1: // Preds: bb0
 br bb2, loc "input.swift":3:11, scope 2 // id: %29
bb2: // Preds: bb1
 br bb3, loc "input.swift":3:11, scope 2 // id: %30
bb3: // Preds: bb2
 br bb4, loc "input.swift":3:11, scope 2 // id: %31
bb4: // Preds: bb3
 br bb5, loc "input.swift":3:11, scope 2 // id: %32
bb5: // Preds: bb4
 br bb6, loc "input.swift":3:11, scope 2 // id: %33
bb6: // Preds: bb5
 br bb7, loc "input.swift":3:11, scope 2 // id: %34
bb7: // Preds: bb6
 br bb8, loc "input.swift":3:11, scope 2 // id: %35
bb8: // Preds: bb7
 br bb9, loc "input.swift":3:11, scope 2 // id: %36
bb9: // Preds: bb8
 br bb10, loc "input.swift":3:11, scope 2 // id: %37
bb10: // Preds: bb9
 br bb11, loc "input.swift":3:11, scope 2 // id: %38
bb11: // Preds: bb10
```

这些basic blocks不执行任何操作，并立即跳转到之后的basic block。它们将在优化过程中删除。这看起来有点浪费，但是它简化了编译器的实现，因为初始代码生成与优化过程是分离的。

```
bb12:                                             // Preds: bb11
  // function\_ref CountableRange.init(uncheckedBounds : (lower : A, upper : A)) -> CountableRange~<A>~
  %40 = function\_ref @\_TFVs14CountableRangeCfT15uncheckedBoundsT5lowerx5upperx\_\_GS\_x\_ : $@convention(method)  (@in τ\_0\_0, @in τ\_0\_0, @thin CountableRange.Type) -> @out CountableRange, loc "input.swift":3:11, scope 2 // user: %46
  %41 = metatype $@thin CountableRange~~.Type, loc "input.swift":3:11, scope 2 // user: %46
```

- 创建`CountableRange.init(uncheckedBounds : (lower : A, upper : A)) -> CountableRange<A>`的引用
- 创建对元祖类型`CountableRange.Type`的引用。

```
%42 = alloc\_stack $Int, loc "input.swift":3:11, scope 2 // users: %43, %48, %46
  store %24 to %42 : $\*Int, loc "input.swift":3:11, scope 2 // id: %43
  %44 = alloc\_stack $Int, loc "input.swift":3:11, scope 2 // users: %45, %47, %46
  store %26 to %44 : $\*Int, loc "input.swift":3:11, scope 2 // id: %45
```

- `store %0 to %1`将%0的值存储到内存地址%1。
- 在basic block 0中，我们创建了两个$Int结构体，分别持有0和100000。现在我们申请栈空间，并将这些值存储在这些空间中。因为其他方法需要使用这些值，所以我们需要将它们放在栈上。

```
 %46 = apply %40(%27, %42, %44, %41) : $@convention(method)  (@in τ\_0\_0, @in τ\_0\_0, @thin CountableRange.Type) -> @out CountableRange, loc "input.swift":3:11, scope 2
```

- `%46`我们通过apply指令初始化一个`CountableRange`。为了方便理解，讲下几个参数：
  - `%27`指向我们为`CountableRange`分配的空间。
  - `%42` 指向我们为包含0的`$Int`结构体分配的空间。
  - `%44` 指向我们为包含1000000的`$Int`结构体分配的空间。
  - `%41` `CountableRange~~.Type`的引用

```
  metatype.dealloc_stack %44 : $*Int, loc "input.swift":3:11, scope 2 // id: %47
  dealloc_stack %42 : $*Int, loc "input.swift":3:11, scope 2 // id: %48
  br bb13, loc "input.swift":3:11, scope 2 // id: %49
```

- 我们不再需要那些存储在栈上的$Int结构体，所以我们在这里释放它们。
- 之后我们跳转到basic block13，这是另一个不必要的分支，因为bb13只有一个前导（as bb13 only has one predecessor）。

```
bb13:                                             // Preds: bb12
  %50 = load %27 : $\*CountableRange~~, loc "input.swift":3:11, scope 2 // user: %52
  %51 = alloc\_stack $CountableRange~~, loc "input.swift":3:11, scope 2 // users: %52, %54, %53
  store %50 to %51 : $\*CountableRange~~, loc "input.swift":3:11, scope 2 // id: %52
```

- `load`从内存地址%27读出`CountableRange`并将其存储在寄存器%50
- 为`CountableRange`在栈上申请空间，将地址存储在寄存器%51。
- `store the CountableRange`我们将其加载到％51处新分配的空间中，从而有效地进行了复制。

```
 %53 = apply %22<CountableRange~~, Int, Int, CountableRange~~, CountableRange~~, Int, Int, Int, Int, Int, Int, IndexingIterator, CountableRange~~, Int, Int, IndexingIterator, CountableRange~~, Int, Int, Int, Int, Int, Int, Int, Int>(%2, %51) : $@convention(method)  (@in\_guaranteed τ\_0\_0) -> @out IndexingIterator, loc "input.swift":3:14, scope 2
  dealloc\_stack %51 : $\*CountableRange~~, loc "input.swift":3:14, scope 2 // id: %54
  dealloc\_stack %27 : $\*CountableRange~~, loc "input.swift":3:14, scope 2 // id: %55
  br bb14, loc "input.swift":3:1, scope 2         // id: %56
```  

- 我们调用%22，这是一个`Collection.makeIterator()`的函数引用，函数有两个参数：
  - `%2`早前在basic block 0中为`IndexingIterator`分配的未初始化内存。
  - `%51`我们存储`CountableRange`副本的地址。
  - 将所有这些放在一起，我们将为`CountableRange`创建一个迭代器。
- 我们释放掉`CountableRange`和它副本的内存，因为我们刚才生成的`IndexingIterator`包含了countable range内所需的所有信息。
- 注意，我们不再需要`CountableRange`的副本，它可能在优化过程中被删除。
- 之后我们跳转到basic block14，不想之前的跳转，这是必须的，因为`bb14`是第一个（不是唯一）有两个前导（predecessors）的basic block。

### The Loop Header

这个basic block是loop Header，因为它控制了loop中的其他所有basic block。这意味着到其他basic block的跳转都必须经过该basic block。

```
bb14:                                             // Preds: bb15 bb13
  // function\_ref IndexingIterator.next() -> A.\_Element?
  %57 = function\_ref @\_TFVs16IndexingIterator4nextfT\_GSqwx8\_Element\_ : $@convention(method)  (@inout IndexingIterator) -> @out Optional, loc "input.swift":3:7, scope 2 // user: %59
  %58 = alloc\_stack $Optional~~, loc "input.swift":3:7, scope 2 // users: %61, %60, %59
  %59 = apply %57(%58, %2) : $@convention(method)  (@inout IndexingIterator) -> @out Optional, loc "input.swift":3:7, scope 2
  %60 = load %58 : $\*Optional~~, loc "input.swift":3:7, scope 2 // users: %66, %64
  dealloc\_stack %58 : $\*Optional~~, loc "input.swift":3:7, scope 2 // id: %61
```

- 获取`IndexingIterator.next()`的引用。
- 在栈上为`Optional`申请空间。
- 调用`IndexingIterator.next()`，并传入两个参数：
  - %58我们刚才为`Optional`申请的空间的地址。
  - %2早前在basic block 0中为`IndexingIterator`分配的未初始化内存
- 有个很有趣的事情：与之前的方法调用不同，%59中`IndexingIterator.next()`的返回值被忽略了。相反，返回值`optional`从%58加载到%60。

```
 %62 = integer\_literal $Builtin.Int1, -1, loc "input.swift":3:1, scope 2 // user: %64
  %63 = integer\_literal $Builtin.Int1, 0, loc "input.swift":3:1, scope 2 // user: %64
  %64 = select\_enum %60 : $Optional~~, case #Optional.some!enumelt.1: %62, default %63 : $Builtin.Int1, loc "input.swift":3:1, scope 2 // user: %65
  cond\_br %64, bb15, bb16, loc "input.swift":3:1, scope 2 // id: %65
```

- `select_enum %0 : $E case #foo: %1, case #bar: %2, ... default %3`这句话的意思是，如果类型为$E的枚举变量%0是foo，则返回%1，对于其他情况，依此类推，默认返回%3。在这段代码中，如果`Optional`的值是`.Some`，%64会返回-1。其他情况返回0。
- `cond_br %64, bb15, bb16`这是有条件的跳转，如果%64等于1，跳转到`bb15`，否则通过指定参数跳转到`bb16`。
- 善于观察的读者已经注意到这里使用%62也就是-1来初始化`integer_literal`。我不太确定为什么要这么做。因为关于[cond_br](https://github.com/apple/swift/blob/main/docs/SIL.rst#id205)文档中只提到了0和1的行为。

### The Loop Body

```
bb15:                                             // Preds: bb14
  %66 = unchecked\_enum\_data %60 : $Optional~~, #Optional.some!enumelt.1, loc "input.swift":3:1, scope 2 // user: %67
  debug\_value %66 : $Int, let, name "i", loc "input.swift":3:5, scope 2 // id: %67
```

`unchecked_enum_data %60 : $E`，`#E.foo`意味着根据给定的#E.foo提供不保证安全的枚举%60的值。
此时，生成的代码会根据我们使用的是.append(5)还是+=[5]而发生差异。

#### The .append(5) Case

```
  // function\_ref Array.append(A) -> ()
  %68 = function\_ref @\_TFSa6appendfxT\_ : $@convention(method)  (@in τ\_0\_0, @inout Array) -> (), loc "input.swift":4:10, scope 3 // user: %73
  %69 = integer\_literal $Builtin.Int64, 5, loc "input.swift":4:17, scope 3 // user: %70
  %70 = struct $Int (%69 : $Builtin.Int64), loc "input.swift":4:17, scope 3 // user: %72
  %71 = alloc\_stack $Int, loc "input.swift":4:17, scope 3 // users: %72, %74, %73
  store %70 to %71 : $\*Int, loc "input.swift":4:17, scope 3 // id: %72
  %73 = apply %68~~(%71, %17) : $@convention(method)  (@in τ\_0\_0, @inout Array) -> (), loc "input.swift":4:18, scope 3
  dealloc\_stack %71 : $\*Int, loc "input.swift":4:18, scope 3 // id: %74
  br bb14, loc "input.swift":5:1, scope 2         // id: %75
```

- 创建`Array.append()`的函数引用。
- 创建值为5的整数常量，然后创建携带这个常量的$Int结构体。
- 在栈上为这个$Int结构体申请内存空间并存储。
- 通过以下参数调用`Array.append()`
  - %71持有整数5的$Int结构体
  - %17全局数组变量的地址
- 释放$Int结构体的内存
- 跳转回 loop header bb14。

#### The +=[5] Case

```
// function_ref += infix<A> (inout [A.Iterator.Element], A) -> ()
  %68 = function_ref @_TFsoi2peuRxs10CollectionrFTRGSaWx8Iterator7Element__x_T_ : $@convention(thin)  (@inout Array, @in τ_0_0) -> (), loc "input.swift":5:10, scope 3 // user: %83
```

- 运算符实际上是函数。这会创建一个对+=函数的引用

```
// function_ref Array.init(arrayLiteral : [A]...) -> [A]
  %69 = function_ref @_TFSaCft12arrayLiteralGSax__GSax_ : $@convention(method)  (@owned Array, @thin Array.Type) -> @owned Array, loc "input.swift":5:13, scope 3 // user: %80
  %70 = metatype $@thin Array.Type, loc "input.swift":5:13, scope 3 // user: %80
  %71 = integer_literal $Builtin.Word, 1, loc "input.swift":5:14, scope 3 // user: %73
```

- 创建一个`Array.init(arrayLiteral : [A]...) -> [A]`函数的引用
- 创建元祖类型`Array.Type`的引用
- 创建一个值为1的整数常量。这是我们要分配内存的数组中元素的数量。

```
 // function_ref _allocateUninitializedArray<A> (Builtin.Word) -> ([A], Builtin.RawPointer)
  %72 = function_ref @_TFs27_allocateUninitializedArrayurFBwTGSax_Bp_ : $@convention(thin)  (Builtin.Word) -> (@owned Array, Builtin.RawPointer), loc "input.swift":5:14, scope 3 // user: %73
  %73 = apply %72(%71) : $@convention(thin)  (Builtin.Word) -> (@owned Array, Builtin.RawPointer), loc "input.swift":5:14, scope 3 // users: %75, %74
```

- 创建一个`_allocateUninitializedArray(count: Builtin.Word)`函数的引用，返回一个元组，其中包含一个未初始化元素的数组和一个指向第一个元素的指针。
- 调用`_allocateUninitializedArray`，并传入整数常量1这个唯一参数。

```
 %74 = tuple_extract %73 : $(Array, Builtin.RawPointer), 0, loc "input.swift":5:14, scope 3 // user: %80
  %75 = tuple_extract %73 : $(Array, Builtin.RawPointer), 1, loc "input.swift":5:14, scope 3 // user: %76
  %76 = pointer_to_address %75 : $Builtin.RawPointer to [strict] $*Int, loc "input.swift":5:14, scope 3 // user: %79
```

- %73的返回值是一个元组，所以`tuple_extract`取出第一个元素（未初始化的数组）到%74和第二个元素（指针）到%75.
- `pointer_to_address %75`是指针%75的未检查寻址（an unchecked conversion of the pointer `%75` into an address）。

```
%77 = integer_literal $Builtin.Int64, 5, loc "input.swift":5:14, scope 3 // user: %78
  %78 = struct $Int (%77 : $Builtin.Int64), loc "input.swift":5:14, scope 3 // user: %79
  store %78 to %76 : $*Int, loc "input.swift":5:14, scope 3 // id: %79
```

- 创建一个值为5的整数常量，然后创建一个$Int结构体携带该值。
- 将$Int结构体存储在未初始化数组的开头
- 数组%74现在已初始化！

```
%80 = apply %69(%74, %70) : $@convention(method)  (@owned Array, @thin Array.Type) -> @owned Array, loc "input.swift":5:14, scope 3 // user: %82
```

- %69是一个`_allocateUninitializedArray(count: Builtin.Word)`函数的引用。我们调用它并传入一下参数：
  - %74是我们刚才初始化的数组
  - %70是元祖类型`Array.Type`的引用
- %80现在是使用整数5初始化的数组

```
  %81 = alloc_stack $Array, loc "input.swift":5:13, scope 3  // users: %82, %84, %83
  store %80 to %81 : $*Array, loc "input.swift":5:13, scope 3 // id: %82
```

- 在栈上为数组申请内存，并将这个包含整数5的数组存储在刚申请的内存空间中。

```
 %83 = apply %68<[Int], Int, Int, CountableRange, IndexingIterator, ArraySlice, Int, Int, Int, Int, Int, Int, IndexingIterator, CountableRange, Int, Int, Int, IndexingIterator, ArraySlice, Int, Int, Int, Int, Int, Int, Int, Int>(%17, %81) : $@convention(thin)  (@inout Array, @in τ_0_0) -> (), loc "input.swift":5:10, scope 3
```

- %68是+=函数的引用，我们调用它并传入一下参数：
  - %17，全局数组变量的地址
  - %81。我们存储包含整数5的数组的栈地址

```
dealloc_stack %81 : $*Array, loc “input.swift”:5:15, scope 3 // id: %84br bb14, loc “input.swift”:6:1, scope 2 // id: %85
```

- 最后，我们释放临时数组的内存，并且跳转回loop header

### The Shared Exit Block

当`IndexingIterator`执行结束时，loop header中的`IndexingIterator.next()`将会返回一个空的`Optional`。这会导致`cond_br`跳转到这个basic block。

```
bb16:                                             // Preds: bb14
  %76 = integer\_literal $Builtin.Int32, 0, scope 2 // user: %77
  %77 = struct $Int32 (%76 : $Builtin.Int32), scope 2 // user: %79
  dealloc\_stack %2 : $\*IndexingIterator, loc "input.swift":3:7, scope 2 // id: %78
  return %77 : $Int32, scope 2                    // id: %79
}
```

- 创建一个$Int结构体包含值为0的整数常量。
- 释放`IndexingIterator`的内存，我们不再需要它了。
- 返回寄存器%77，这一个值为0的整数。告诉操作系统，程序安全退出了。


## The Test Case

现在你已经对SIL有足够的了解了。可以写一个针对我们实现的优化过程的测试用例了。

```
// CHECK-LABEL: sil @append_contentsOf
// CHECK:   [[ACFUN:%.*]] = function_ref @arrayAppendContentsOf
// CHECK-NOT: apply [[ACFUN]]
// CHECK:   [[AEFUN:%.*]] = function_ref @_TFSa6appendfxT_
// CHECK:   apply [[AEFUN]]
// CHECK: return
sil @append_contentsOf : $@convention(thin) () -> () {
```

这里`// CHECK`指导测试程序对指定的匹配文（或不匹配）进行断言。这些指令断言旧的`Array.append(contentsOf:)`函数调用已经被`Array.append(element:)`调用所取代。因为后者由标准库提供，因此它的名字被mangle为`@_TFSa6appendfxT_`。在真实的代码中，前一个调用的名称也会被mangle，但是因为我们的优化过程将只关注函数的语义属性，所以我们可以在测试中使用一个便于理解的函数名称。

```
  %0 = function_ref @swift_bufferAllocate : $@convention(thin) () -> @owned AnyObject
  %1 = integer_literal $Builtin.Int64, 1
  %2 = struct $MyInt (%1 : $Builtin.Int64)
  %3 = apply %0() : $@convention(thin) () -> @owned AnyObject
  %4 = metatype $@thin Array.Type
  %5 = function_ref @arrayAdoptStorage : $@convention(thin) (@owned AnyObject, MyInt, @thin Array.Type) -> @owned (Array, UnsafeMutablePointer)
  %6 = apply %5(%3, %2, %4) : $@convention(thin) (@owned AnyObject, MyInt, @thin Array.Type) -> @owned (Array, UnsafeMutablePointer)
  %7 = tuple_extract %6 : $(Array, UnsafeMutablePointer), 0
  %8 = tuple_extract %6 : $(Array, UnsafeMutablePointer), 1
  %9 = struct_extract %8 : $UnsafeMutablePointer, #UnsafeMutablePointer._rawValue
  %10 = pointer_to_address %9 : $Builtin.RawPointer to [strict] $*MyInt
  %11 = integer_literal $Builtin.Int64, 27
  %12 = struct $MyInt (%11 : $Builtin.Int64)
  store %12 to %10 : $*MyInt
```

创建仅含一个元素的数组。我们并没有按照在macOS上生成它的方式来编写SIL，因为在没有Objective-C桥接的Linux上测试会失败。

```
  %13 = alloc_stack $Array
  %14 = metatype $@thin Array.Type
  %15 = function_ref @arrayInit : $@convention(method) (@thin Array.Type) -> @owned Array
  %16 = apply %15(%14) : $@convention(method) (@thin Array.Type) -> @owned Array
  store %16 to %13 : $*Array
```

创建一个空数组，并将其存储在栈中。它需要在栈上，因为它是Array.append(contentsOf:)的self形参。

```
 %17 = function_ref @arrayAppendContentsOf : $@convention(method) (@owned Array, @inout Array) -> ()
  %18 = apply %17(%7, %13) : $@convention(method) (@owned Array, @inout Array) -> ()
```

获取对`Array.append(contentsOf:)`的引用，并以两个数组作为参数调用它。

```
  dealloc_stack %13 : $*Array
  %19 = tuple ()
  return %19 : $()
}
```

清空栈，并返回`void`，这在SIL中代表空元组。

## The Optimization Pass

SIL的类型有两种：Raw SIL 和 Canonical SIL。Swift编译器在AST之后生成Raw SIL。Raw SIL可能不是语义上有效的（semantically valid）。一系列确定的optimization passes作用于Raw SIL，并生成Canonical SIL，这保证在语义上是有效的。在此转换过程中，函数是专门化（specialized）和内联的，但是我们可以通过向函数添加语义属性来延迟函数的内联：

```
@semantic(“sometext”)
```

Semantic属性提示Swift编译器以不同的方式优化代码。它们可以用来禁用优化(如内联)，或强制优化。它们在Swift标准库中被广泛使用，在该库中，手动调试代码的优化方式非常重要。

`Array`方法被定义在**stdlib/public/core/Arrays.swift.gyb，gyb**是“Generate Your Boilerplate”的缩写。我们将在`Array.append(contentsOf:)`中添加一个Semantic属性，因为在Array +=函数调用之后存在的函数调用是内联的。之后我们将修改**include/swift/SILOptimizer/Analysis/ArraySemantic.h**，它的执行文件将通过新的Semantic属性更容易运行。我们将会把新的Semantic属性添加到**lib/SILOptimizer/LoopTransforms/COWArrayOpt.cpp**来屏蔽一些有关non-exhaustive switch的警告。

接下来，我们需要编写代码来查找数组字面量的初始化，查找数组中这些数组字面量的使用。追加`Array.append(contentsOf:`指令。然后验证数组在这两者之间没有被修改或转义。幸运的是，有一个pass已经做了类似的事情：**ArrayElementValuePropagation**。这种优化将像这样转换代码：

```
let a = [1, 2, 3];
let b = a[0] + a[2];
```

转换成这样的代码

```
let a = [1, 2, 3];
let b = 1 + 3;
```

这已经很接近我们想要的优化效果了，所以我们将修改这个pass增加一些额外的工作，而不是创建一个全新的pass。

一个优化pass的“主函数”是`void run()`,我们在SILFunctionTransform编写代码因为我们在转换函数体。下面是修改后的run()方法的演示。

```
void run() override {
    auto &Fn = *getFunction(); // Get a reference to the function we're in

    // Store information about the calls that we want to replace
    llvm::SmallVector
      GetElementReplacements;
    llvm::SmallVector
      AppendContentsOfReplacements;

    // Iterate through the basic blocks in the function
    for (auto &BB :Fn) {
      // Iterate through the instructions in the basic blocks
      for (auto &Inst : BB) {
        // Filter for only apply instructions
        if (auto *Apply = dyn_cast(&Inst)) {
          // This is a helper class that tells us if an apply instruction is an array literal allocation
          // and simplifies getting the elements in that literal
          ArrayAllocation ALit;
          if (ALit.analyze(Apply)) {
            ALit.getGetElementReplacements(GetElementReplacements);
            // We call out helper method that extracts all the elements of an array literal
            ALit.getAppendContentOfReplacements(AppendContentsOfReplacements);
          }
        }
      }
    }

    bool Changed = false;
    
    // This was already in this optimization.
    for (ArrayAllocation::GetElementReplacement &Repl : GetElementReplacements) {
      ArraySemanticsCall GetElement(Repl.GetElementCall);
      Changed |= GetElement.replaceByValue(Repl.Replacement);
    }

    // We'll just add this call to our helper function here
    Changed |= replaceAppendCalls(AppendContentsOfReplacements);

    // You need to invalidate the analysis if you've changed something in an optimization
    if (Changed) {
      PM->invalidateAnalysis(
          &Fn, SILAnalysis::InvalidationKind::CallsAndInstructions);
    }
  }
```

现在让我们看一下replacement helper方法。

```
bool replaceAppendCalls(
                  ArrayRef Repls) {
    auto &Fn = *getFunction();
    auto &M = Fn.getModule();
    auto &Ctx = M.getASTContext();

    if (Repls.empty())
      return false;

    DEBUG(llvm::dbgs() << "Array append contentsOf calls replaced in "
                       << Fn.getName() << " (" << Repls.size() getAnyNominal();
      SubstitutionMap ArraySubMap = ArrayType.getSwiftRValueType()
        ->getContextSubstitutionMap(M.getSwiftModule(), NTD);
      
      GenericSignature *Sig = NTD->getGenericSignature();
      assert(Sig && "Array type must have generic signature");
      SmallVector Subs;
      Sig->getSubstitutions(ArraySubMap, Subs);
      
      // Finally we call the helper function we added to ArraySemanticsCall
      // to perform the actual replacement.
      AppendContentsOf.replaceByAppendingValues(M, AppendFn,
                                                Repl.ReplacementValues, Subs);
    }
    return true;
  }
```

## A Wild Benchmark Regression Appears

我认为我们已经做到了这点，但是尽管我的优化是有效的，但这导致了其他基准的回退。这不太好。我必须通过比较从tip-of-tree swiftc生成的基准SIL和我修改过的分支来隔离问题。我注意到我修改过的分支中的函数调用是未专门化的，这导致了速度的减慢。

此处的问题由于为**array.append_element**和**array.append_contentsOf**添加了semantic属性，我们延后了内联。这些函数包含对其他泛型函数的调用，泛型专门化（specializer）器依赖于内联父函数，这样它就可以专门化（specialize）它们内部的函数调用。

一种解决办法，我认为是添加额外的内联流程来提高内联[转换的性能](https://github.com/apple/swift/blob/main/lib/SILOptimizer/Transforms/PerformanceInliner.cpp)。然而，仅仅为两个函数添加一个完整的内联流程似乎有些愚蠢。

另一种解决办法，我考虑的是添加一种不同类型的semantic标记，它只用于标识函数而不会改变优化器的行为。

最终，[eeckstein](https://github.com/apple/swift/blob/main/lib/SILOptimizer/Transforms/PerformanceInliner.cpp)通过了这个pr，我通过为两个数组函数添加semantic属性来完成特殊的内联转换。

## Lessons Learned

为swiftc做贡献需要耗费很大精力。我的PR经过了两个月才合入，当我做的时候感觉这像是一份兼职工作。我想我不会再做出这样的贡献了，除非有人付钱给我。并且使用一种比邮件对话更容易与有帮助的苹果员工沟通的方式。

在我开始写我的swiftc提交前我使用一台12寸的MacBook。它太慢了，以至于我每天只能迭代（iterate）一次代码，因为一次简单的编译和测试需要跑一整晚。我最终还是买了一台顶配的15寸MacBook Pro，因为这是解决每天只能迭代一次代码的唯一办法。

即使是新的macbook Pro，一次完整的集成测试也需要跑4个小时。这是个问题，只有Swift repo的committers有权限在它们的CI上运行测试用例。Swiftc可能会开源，但是如果贡献者需要数千（美元）购买机器才能工作，那也不会有太多的贡献者。

在提出我卑微的PR后，我成为了Swift的前100名贡献者之一。我工作相关的大部分文件也只有十来个人接触过。这对一个流程的语言来说不是好事，我希望会有改变。

Swift开发者的工作流程真的是又慢又死板。如果你切换分支，你必须等待大部分文件被重新编译，即使是一台顶配的电脑，这会花费15分钟或者更长时间。我最终保留了多个Swift repositories(每个20gb !)，这样我就可以更容易地切换分支，因为我要将SIL与编译器的不同版本区分开来。我不得不编写一些bash脚本来管理这种混乱情况。

Swiftc的学习曲线非常陡。文档非常少，代码注释写的也不好。这是一个非常大的工程，所以，即使是像我这样的小改变，也需要我与多位主题专家交谈。甚至有一群苹果的人和你一起工作

swiftc很容易被破坏，因为它太复杂了。我最早的PR在一个月内被approve并merge。尽管只有200多行的改变，我也收到来自6名reviewer的125条comment。即使经过如此多的审查，它也几乎差点被revert，因为它引入了一个内存泄漏，第七个人在运行了四个小时的标准库集成测试后发现了这个泄漏。

这个测试捕捉到内存泄漏完全是偶然。测试用例是用Swift写的，我的优化破坏了该测试中的内存泄漏断言。换而言之，我的优化造成了测试用例本身的内存泄漏，而不是测试用例正在测试的代码的内存泄漏。如果将其投入生产，这将是一个严重的bug，因为这种优化会将内存泄漏引入使用有缺陷的swiftc编译的Swift程序。

优化的pipeline也是很容易被破坏的。pass的运行顺序以及它们的配置方式是非常重要的，但是没有很好的文档记录。这块有太多模糊的内容，在我看来，一个解决方案什么时候是取巧的，什么时候是可接受的，很不明确。在代码中内联这两个方法在我看来可能是取巧的，但是我没有像苹果的人那样有那么多的上下文信息。

如果你想要为swfitc作贡献，确保运行所有测试和基准测试。标准库的集成测试在README中是没有文档的，但是你也应该运行它。有时这是很困难的。当我在做这个PR的时候，基准 runner没有工作，所以我不得不自己编写基准测试脚本。苹果CI上的基准测试是可以运行的，但这对我没有帮助，因为只有提交者可以访问它。

## Three Ideas For Improvment

我认为Swift作为一个开源项目有很大的潜力，但我不知道大多数人现在能如何为它做出贡献。

- 任何人，不仅仅是贡献者，都必须能够在CI上运行测试和基准测试。
- 要想从苹果员工那里获得帮助，需要一种比邮件交流更便捷的方式。
- 开发人员的工作流程需要改进。分支切换和重新编译应该是低成本的。像基准测试运行程序这样的脚本应该得到很好的维护。

我省略了一些优化工作的部分内容，但这篇文章已经太长了。我可能稍后会回来填补空白，但在此期间，你可以在[这里](https://github.com/apple/swift/pull/8464/files)找到这个pull请求的最终版本。
