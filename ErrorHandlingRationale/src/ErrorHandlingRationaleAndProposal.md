原文地址：https://github.com/apple/swift/blob/main/docs/ErrorHandlingRationale.rst#typed-propagation

本文调查了错误处理领域，分析了在其他语言中已经提出或正在实践中的各种思想，并最终提出了Swift的错误处理方案并在我们的API中贯彻这些规则。

# 基本原则

我需要先声明一些术语

## Kinds of propagation

我了解到人们讨论传播方式时会提到显示和隐式，但是我不打算用这些术语，因为在错误处理领域这样的区分是没有意义的，主要表现在：至少有三种不同的错误处理方式或多或少都带有显式传播的特点，并且其他维度的区分也是很重要的。

差别主要集中在一下几个方面：

* 该语言是否允许将函数被设计成用来生成错误； 这种语言就具备了**typed propagation**。
* 在**typed propagation**语言中，默认规则是函数可以产生错误或不产生错误;这是该语言的**默认传递规则**。
*  在**typed propagation**语言中，该语言是否静态地强制执行上述操作，*以使不产生错误的函数不调用无法处理错误的函数；* 这种语言具有静态强制类型化传播。（除了静态的方式，语言可以动态来执行上述操作，具体的操作，自动插入代码来断言是否将错误抛出。C++就是这么做的。）
*  该语言是否要求将所有潜在的错误点识别为potential error sites； 这种语言具有**marked propagation**。
*  是否使用该语言的常规数据流和控制流的方式显式完成传播；这种语言具有**manual propagation**。相反，可以控制程序从原始错误点自动跳转到适当处理程序的语言具有自动传播功能。

## Kinds of error
什么是错误?程序中可能出现许多不同的错误情况，我们希望程序员怎么应对这些错误呢？可以依据这个思路将错误进行分类。由于我们期望程序员做出不同的应对，同时语言是程序员做出应对错误的方式，因此在语言层面，针对不同类型的错误做出不同的反应是有意义的。

需要说明的是，一方面，在许多情况下，错误的类型却是报错代码作者的有意而为之。而一方面，根据上下文的不同，不一样的选择可能也是有作用的。我将使用“simple domain error”的示例可以轻松地代替“recoverable error”（如果作者认为自动传播比立即恢复更有用）乃至“logic failure”（如果作者希望防止试探性地使用，例如检查先决条件是非常昂贵）。

### Simple domain errors
simple domain error类似于非整数的字符串调用`String.toInt()`。尽管调用的方法参数有明显的限制，但是能够传递其他值（非法的值）来测试方法是否可以运行，这也很有用。客户端通常会立即处理该错误。

类似这种情况最好使用可选项来处理。更复杂的错误处理模型不能带来更多的好处，并且使用错误处理模型会使公共代码变的冗余。例如在JAVA中，`try`中尝试将`String`解析成整数，`catch`中会捕获解析异常，该异常在语法上是重量级的（syntactically heavyweight）(如果没有优化，效率会很低)。
由于Swift已经对可选项提供了很好的支持，所以这些情况没必要成为proposal的重点。

### Recoverable errors

Recoverable errors包括找不到文件，网络超时以及类似情况。这操作有多种可能的错误情况。应鼓励客户端认识到可能的错误条件，并考虑正确的处理方法。通常，这将通过中止当前操作、按需自行清理，并将错误传播到可以更合理地处理它的地方，例如，将错误通知给用户。
这些是我们大多数api使用NSError和CFError的错误条件。大多数库都有一些类似的概念。这是本文的重点。

### Universal errors

**Universal errors**和普通Recoverable errors之间的区别是错误情况的种类较少，而语言中错误的潜在来源则更多。如果错误可能是由多种不同的情况引起，那么程序员直接处理全部的错误根源几乎是不切实际的，这类错误就是**Universal errors**。

如果想将某些情况纳入错误处理范围，则只能将其视为Universal errors。 这些包括：

*	异步场景，例如进程收到`SIGINT`，或者当前线程被另一个线程取消。原则上，这些条件可以在任意instruction boundary，并且适当地处理它们需要程序员，编译器和运行时格外谨慎。 
* 无处不在的错误，例如内存不足或堆栈溢出，基本上可以假定任何操作都可能会引发该类错误。

但是，随着**抽象层**的引入，其他类型的错误情况实际上会变成Universal errors。程序员不希望某些操作报错，如读取集合size或读取对象的属性。但是，如果集合数据实际上是由数据库提供的呢？数据库查询操作可能会失败。如果用户必须编写代码就像任何晦涩的**抽象层**可能会产生错误一样，那么他们会被Universal errors困住。（tag）

Universal errors要求使用某些语言方法。除了可以保证不产生错误的特殊情况外，Universal errors的**类型传播（Typed propagation）**是不可能的。**Marked propagation**会让用户反感。传播必须是自动的，并且实现必须是“零成本”或尽可能接近零，因为在每个操作后检查错误的开销是非常大的。

由于这些原因，在我们的API中，通用错误条件通常是使用Objective-C异常实现的，尽管并非所有Objective-C异常都属于此类。

所有的要求总结成一点就是，几乎从所有的起点开始，所有操作都必须隐式地“不可撤销”。**This combination of requirements means that all operations must be implicitly "unwindable" starting from almost any call site it makes.**。为了系统的稳定性，unwinding过程必须复原所有可能已经被破坏的约束条件，编译器在这方面不会帮助程序员。程序员必须自觉地认识到约束条件被破坏时可能会发生错误，并且他们必须主动避免约束条件被破坏，万一忘记了主动预防导致了错误产生，也应该追查错误。这就需要对一个人的代码进行谨慎严格的考量，以预见所有的错误点并认识到重要的约束条件在不断变化。

很多问题的产生都是由代码编写不当造成的，以下有一些编程风格会尽力减少代码产生的问题。例如，一个功能强大的程序会着重地对其最外层的循环保持 mutation 和 side-effects，因此自然不会有任何约束条件在不断变化（tag）。从操作中的任意位置传播错误只会放弃到此为止所做的所有工作。然而，随着mutation 和 side-effects的出现，这种**平和的状态happy state**很快就会消失。复杂的mutation不能轻易逆转。数据包不能被发送。assert表示代码不应该这样写，但也不能让人从其他方面理解它，以上会让人很困惑。只要程序员确实面对这些问题，该语言就有责任帮助他们。

因此，在我看来，提倡使用universal errors是有很大问题的。它们破坏了代码的易读性，并且削弱了语言帮助程序员推理错误的能力。相反，此设计将重点放在这类显式可跟踪错误上，如同现在Apple平台上的`NSError`。

但是，基于一些重要的原因，我们不能完全禁止universal errors的使用：

* 如上所述，他们依然是将某些错误情况引入错误处理模型的唯一可行手段。然而，大部分的使用都遭到了各种各样的反对。余下最重要的使用场景就是“escaping”，如果API意外发现自己需要抛出错误，但是在设计阶段没有涉及相关的功能，这种情况就被称为“escaping”。 
* Swift的目标是，在任何一个可能的平台上，让Objective-C和C++的异常都是合法的交互（tag）。Swift必须为他们提供某种长期的支持。

这些原因并不能解决universal errors的问题（tag）。隐式地要求函数从任意点unwinding本身就是有风险的。我们不想推广这种模式。但是，当然可以编写能够正确处理universal errors的代码。从实际出发来看，unwinding大部分代码通常是可行的。Swift可以使用“零成本”exceptions支持辅助、无类型传播机制。对开发者来说，尽量仔细的编写代码，以最大程度的减少代码unwinding的范围。例如 通过在调用“escaping” API之后立即捕获universal errors，然后使用常规类型的传播将其重新抛出。

但是，这项工作不在Swift 2.0的范围之内。 我们可以轻松地做出此决定，因为这样做并不会阻止我们将来执行该决定：

* 我们目前不支持通过Swift函数传播异常，因此更改catch来捕获它们也不会对破坏兼容性。
* 尽管有些尴尬，但外部异常可以通过捕获机制自动反射到类似`Error`的模型中。
* 同时，必须处理Objective-C异常的开发人员始终可以通过在Objective-C中编写stub以将异常显式“桥接”到`NSError out`参数中来实现。这种处理方式不是理想的方式，但是目前是可以接受的。
### Logic failures
最后一类是逻辑故障，包括数组访问越界、强制展开`nil`可选值和其他类型的断言。程序员失误造成的错误，应该在开发时通过代码进行修正，而不是等到运行时再进行动态修复。
即使在断言失败之后，高可靠性系统也可能需要某种方式来运行。**可以将减少该过程视为拒绝服务攻击的媒介（ Tearing down the process can be viewed as a vector for a denial-of-service attack.）**。但是，断言错误可能表明进程已被破坏并受到攻击，无论如何，如果继续运行系统，可能会导致系统出现其他更严重的安全破坏。

如何正确处理这些错误情况是一个开放的问题，这不是此建议的重点。 如果我们决定让它们具备可恢复的特点，它们可能会遵循与universal errors相同的实现机制，即使不一定是相同的语言规则。

## Analysis

让我们更深入地了解上面列出的几种不同的错误处理。

### Propagation methods

从语言层面上来讲，有以下两种基本的方法将错误从错误出现的位置（error site ）传递到错误处理的位置。

第一种是**manual propagation**，它可以通过语言的normal evaluation、data flow和control flow processes来完成。以下是命令式语言C语言中一个手动传递错误的例子，其中用到了特殊返回值。

```
struct object *read_object(void) {
  char buffer[1024];
  ssize_t numRead = read(0, buffer, sizeof(buffer));
  if (numRead < 0) return NULL;
  ...
}
```


下面是另一种命令式语言Objective-C通过out-parameters手动传递错误值的示例。

```
- (BOOL) readKeys: (NSArray<NSString*>**) strings error: (NSError**) err {
  while (1) {
    NSString *key;
    if ([self readKey: &key error: err]) {
      return TRUE;
    }
    ...
  }
  ...
}
```

这是一个使用impure functional language SML(tag)的ADT进行手动传递的示例； 这有点武断，因为SML库实际上是这样使用异常的：

```
fun read_next_cmd () =
  case readline(stdin) of
    NONE => NONE
  | SOME line => if ...
```

以上的处理流程可以抽象成两个步骤。
第一，通过语言的data flow标准工具主动检查错误。
第二，通过语言的control flow标准工具，主动绕过函数的其他流程。

第二种是**automatic propagation**，通过一些隐蔽的、更内在的方式传递错误，而不会直接在普通的control flow规则中体现。以下是命令式语言Java中的异常自动传播的典型事例：

```
String next = readline();
```

如果`readline`遇到错误，会抛出一个异常。然后，该语言终止作用域（tag），直到该异常被有能力处理该异常的try语句动态捕获。请注意，这里没有任何代码暗指上述处理正在进行。

manual propagation的主要缺点是代码写起来复杂，并且需要大量重复样板。这些可能听起来没什么大不了的，但是确实是很严重的问题。

* 复杂的代码会让程序员注意力不集中，使他们写代码的时候不严谨。这种情况下写出的错误处理代码不仅没用甚至会带来更严重的问题。
* 重复的样板不仅会降低代码的可读性还会破坏代码的可维护性。这会占用程序员的时间，导致成本的增加。这会使错误处理变得麻烦，错误就不能被正确的处理。这种处理方式鼓励使用捷径（如宏定义），这可能会破坏其他优势和目的（tag）。

automatic propagation的主要缺点是，它会使代码的control flow变得难以理解。 我将在下一节中进一步讨论。

请注意，automatic propagation不一定只存在于语言内部。如果传播与源中对外暴露的结构体解耦，则传播是自动的。可以使用任何允许代码重构的语言工具（例如，使用宏或其他术语重写工具）或基本语法的重载（例如，Haskell使用do-notation操作monads）将automatic propagation复制为任何语言工具的库。

还应注意，对于任何特定程序，多种传播策略在“发挥作用”。 例如，Java通常在其标准库中使用异常，但是为了提高效率，某些特定的API可能会选择在出错时返回null。Objective-C提供了一个功能相当齐全的异常模型，但是标准的API（有一些重要的异常）仅将它们用于处理unrecoverable errors，标准的API更倾向于使用NSError输出参数进行manual propagation。Haskell有大量的核心库函数使用返回值`Maybe`代表成功或错误，但它也提供了至少两个类似传统的、automatically-propagating的异常(ErrorT单子转换和IO单子中的异常)。

因此，尽管我要说的好像是语言只需要实现一种错误传播策略，但是应该明白，实际情况总是更加复杂的。如果程序员就是想使用manual propagation，我们是没办法阻止他们的。提案的一部分将讨论同时使用多种策略。

### Marked propagation

传播是否被标记与传播是否是自动的问题密切相关。如果在call site上标明从该点开始传播是可能的，那么我们可以说这种语言使用了**marked propagation**。

在某种程度上，每种使用manual propagation的语言都使用**marked propagation**，因为手动传播错误的代码大致标记了产生错误的调用者。 但是，传播逻辑有可能与调用逻辑是解耦的。

marked propagation与语言设计中的一个核心思想是不一致的：如果一门语言执行隐式操作可能会产生错误，marked propagation就不能作为唯一的传播方门。例如，如果一种语言期望将out-of-memory认定为recoverable errors，则必须考虑内存分配的所有情况。在高级语言中，这将包含大量的隐式操作。这种语言不能要求使用marked propagation。

unmarked propagation让人无法接受的点在于，它使得程序员无法直接看到哪些操作会产生错误，因此无法直接了解函数的control flow。如果必须使用unmarked propagation，程序员就要考虑以下两项：

* 你要仔细考虑函数你主动调用的每个函数的实际动态行为。
* 你要仔细地封装方法，保证关键部分不会产生universal error，从而导致程序处于异常状态的情况。

第二项是可以借用一些手段实现的。 首先，这部分代码与control flow代码解耦，以保持稳定性并在操作后进行清理。例如，始终在C ++中使用构造函数和析构函数来管理资源。 因为存在可能抛出错误的隐式代码，所以C++中强制要求这么做，并且使用了exceptions。当时理论上这也是可以应用在其他语言中的。即便如此，它仍然需要相当谨慎和严格的编程风格。

可以想象automatic propagation的标记形式，其中传播本身是隐式的，只是必须明确标记（本地）始发点。 这是我们建议的一部分，我将在下面讨论。

### Typed propagation

下一个主要问题是语言是否明确跟踪并限制了错误传播。也就是说，函数声明中是否有明确的内容告诉程序员是否会产生错误？ 我们称之为**Typed propagation**。

#### Typed manual propagation

传播的类型与它是手动的还是标记的没有太大关系，但是有一些常见的模式。manual propagation的最主要形式都是类型化的，因​​为它们将失败作为直接结果或out参数传递给调用方。

这是out参数的另一个示例：

```
- (instancetype)initWithContentsOfURL:(NSURL *)url encoding:(NSStringEncoding)enc error:(NSError **)error;
```

out参数有一些不错的优点。首先，它们是可靠的标记来源；即使实际传播与调用分离，只要out参数具有可识别的名称，你也可以一直监听可能产生错误的调用。其次，可以共享一些模版，因为可以多次使用相同的变量作为out参数。不幸的是，您不能使用它来“欺骗”，只检查一次错误，除非有通俗约定确保以后的调用不会故意地复写该变量（tag）。

函数式语言中的一种常见替代方法是返回Either类型：

```
trait Writer {
  fn write_line(&mut self, s: &str) -> Result<(), IoError>;
}
```

如果调用者想使用结果，他们就必须处理错误。除非调用没有真正产生有意义的结果（`write_line`就没有），否则此方法将运行的很好； 之后工作量的多少就取决于本门语言是否易于忽略结果。这也会产生很多难处理的嵌套：

```
fn parse_two_ints_and_add_them() {
  match parse_int() {
    Err e => Err e
    Ok x => match parse_int() {
      Err e => Err e
      Ok y => Ok (x + y)
    }
  }
}
```

在此，对于每个可能失败的顺序计算，都需要另一级嵌套。重载的计算语法，如Haskell的`do`，将有助于解决这些问题，但只有通过切换到一种automatic propagation。

如果手动传播发生在非主干逻辑，则可以是非类型化的。例如，一个对象，在遇到错误时会在其自身上设置标志，而不是直接返回错误； 或考虑使用POSIX的变体，该变体期望单独检查errno以查看特定的系统调用是否失败。

#### Typed automatic propagation

具有automatic propagation类型的语言有以下几个特点。

##### The default typing rule

最重要的问题是，你是选择产生错误还是选择不产生错误。也就是说，没有特定注解的函数是否会产生错误？

通常的弹性准则是，希望使用lazy选项来为实现保留更多的灵活性。可以产生错误的函数肯定更灵活，因为它能做更多的事情。相反，将不能产生错误的函数更改为可以产生错误，也就意味着需要根据调用者响应错误的方式改变function contract。不幸的是，这会带来一些糟糕的后果：

* Marked propagation会变得很繁重。每个调用都需要一个annotation，要么在函数上(表示它不能生成错误)，要么在call site上(用于标记传播)。用户可能会反感这么多的记录方式。
* 大多数函数无法按照我定义的方式生成recoverable errors。也就是说，忽略universal errors的来源，可以合理地预期大多数功能将不会产生错误。但是，如果这不是默认状态，则意味着大多数功能都需要annotation。 再次，这是很多繁琐的记录工作。 API中也很混乱。
* 假设你注意到一个函数错误地缺少annotation。你修复了它，同时又不得不为它调用的所有函数都添加annotation，这工作是不可能完成的； 就像C ++中的const正确性一样，最终的结果是惩罚那些试图改进代码的认真的用户。
* 假装每个功能都是错误源的模型会把人搞崩溃掉。程序员对待代码应该有严谨的思考，但是要求他们对自己代码涉及到的其他代码也要严谨思考，这可能有点过分了。更糟的是，如果没有marked propagation，编译器就不能真正帮助程序员专注于已知的可能的错误来源。
* 编译器对代码生成的分析必须假设所有的事情都可能产生错误，即便实际上它们是不会产生错误的。这个过程中会创建很多隐式传播路径，这些路径之后全部都会失效，这严重增加了代码体积。

换句话说，默认情况下，函数无法生成错误。这与我假设的最常见的情况一致。就弹性而言，这意味着希望用户在发布API之前更加仔细地考虑哪些功能会产生错误。但这与Swift已经要求他们仔细考虑类型的做法类似。同样，他们至少会为初始化实现添加正确的annotations集合。 因此，我认为这是一个合理的选择。

##### Enforcement

接下来的问题是如何强制禁止automatic propagation的类型规则。它应该是静态的还是动态的？也就是说，如果一个函数声明不会产生错误，并且调用了一个会报错但是不处理错误的函数，那应该是编译器错误还是运行时断言？

动态强制执行的唯一真正好处是，如果函数被误标记为可以产生错误，调用这样的函数会变得简单。 如果默认情况下假定所有函数都产生错误，那就会出现问题，因为错误可能只是由于疏忽造成的。但是，如果假设函数不会产生错误，那么报错一定是某人故意为之。基于这种情况，大力推进静态检查是很有必要的。同时，动态强制执行完全破坏了类型传播的大多数优势，因此不需要在动态强制执行上浪费太多精力。真正保留下的唯一好处是，annotation可以用作有意义的文档。因此，对于本文的其余部分，除非另有说明，否则假定类型传播是静态强制的。

同时，动态强制很大程度上破坏了typed propagation带来的优势，因此不需要在这上面花太多心思。真正保留下来的唯一好处是，annotation可以用作有意义的文档。因此，对于本文的其余部分，除非另有说明，否则就默认类型传播是静态强制的。

##### Enforcement

最后一个问题是异常类型应该有多具体？函数应该能够声明其产生的特定错误类，还是应该仅仅以布尔值作为annotation？

从java上的经验来看，异常类型过于具体并不是最好的处理方式。能够识别错误的具体类型是很有用的，但是库通常希望能保留其产生错误的灵活性，因此许多错误最终变成了通用类型的错误。不同的库最终都有library-specific的常规错误类，而异常列表最终只是重述了库自身的依赖关系，或者以丢失关键信息的方式包装了底层错误。

#### Tradeoffs of typed propagation

Typed propagation具有许多优点和缺点，主要取决于传播是否是自动的。

主要优点是更安全。它迫使员做一些事情来处理或传播错误。 这将带来一些缺点，我将在后面进行讨论，但我认为这是一个相当核心的静态安全保证。 这在线程通信很普遍的环境中尤其重要，它指出了一种常见的情况，即错误需要以某种方式传播回原始线程。。

即使我们决定使用typed propagation，我们也不能忽视它的缺点，并研究如何改善这些缺点：

* 任何类型的多态都会变得更加复杂，尤其是高阶函数。 原则上，不会产生错误的函数是会产生错误的函数的子类型。但是：
	* Composability suffers。高阶函数必须确定是否允许它的函数参数产生错误。 如果不允许产生错误，则需要尽可能的限制它的使用场景，或者尽量避免它与能产生错误的函数一起使用。如果是这样，即使传递的函数参数不会产生错误也需要转换(如果使用手动传播，则需要显式转换)，调用的结果可能会声明可以生成错误，而实际上却不能的。这可以通过重载来解决，但这是大量的样板文件和冗余，特别是对于采用多个函数的调用(如函数复合操作符)。
	* 如果允许隐式转换，则可能需要引入thunks（tag）。在某些情况下，这些thunks是内联的——除非，对于代码来说，能够逆转这种转换并动态检测不能实际生成错误的函数是非常有用的。例如，如果算法知道其函数参数永不报错，则它可能能够避免不必要的记录。这带来了一些描述上的困难

* 它倾向于使用分散式错误处理，而不是让错误传播到实际知道如何处理错误的层级。
	* 有些程序员总是在代码中错误的使用错误处理程序（handler），这些错误处理程序handler只会吞掉错误，而不是将错误传播到正确的位置。这样处理反而会导致矫枉过正。如果错误只是静默传播，通常会更好。这样做可能会导致系统处于异常状态并没有任何相关记录（tag）。好的语言和可用于传播错误库工具可以帮助避免这种情况，尤其是在线程间通信的时候。
	* 在许多情况下，由于程序员对输入进行了仔细的限制，实际上不可能出现错误。例如，匹配`/ [0-9] {4} /`，然后将结果解析结果为整数。在不能传播错误的环境中这样做需要方便。但是有这个功能的库（可以传播错误）在设计的时候就要很小心，以防止误吞实际错误。如果库没有真正吞掉错误，而是正常的抛出错误引起程序异常，那么这就足够了。
	* swift中的高阶函数允许用户自定义错误处理combinators，这个改进点可能会有助于解决这些问题。也就是说，当懒惰的JAVA程序员还在为`try/catch`吞掉错误而发愁时，Swift已经为他们提供了更加优雅的处理方式。

标记的、静态强制类型传播的另一个小优点是：这对某些种类的重构是一个福音。具体来说，如果一个操作此前是不产生错误的，经过重构后，这个操作会产生错误。如果缺少了刚才提到的特性，那么重构会变得很危险，并且很容易引入意想不到的BUG。如果传播是非类型的，或者类型不是静态强制执行的，那么编译器根本无法帮助你找到需要有错误检查代码的call site。即使使用静态类型的传播，如果未在call site上专门标记该传播，则编译器也不会警告您有关可以处理或隐式传播错误的上下文进行的调用。但是如果所有的特性都具备，编译器就会为查找所有存在的call site提供有力的帮助。

### Error Types

错误有很多种。 能够以编程方式识别并响应特定种类的错误原因是非常重要的。 Swift应该为此支持简单的模式匹配。

但是我从未见过比这更粗糙的分类;例如，我不确定你应该如何应对一个任意的、未知的IO错误。而且，如果存在有效的错误类别，则可以使用谓词而不是公共子类来表示它们。在此我想从单一类型开始逐渐让人们明白为什么需要更多、更准确的错误分类。

#### Implementation design

实现自动错误传播有几种不同的常见策略（在实现设计中不需要过多的关注manual propagation）。

在实现阶段，大部分语言都有以下两个基本任务：

* 通过作用域和函数将控制权转移到相应的错误处理程序。
* 为突然终止的作用域执行各种意义的“清理”任务：
    * 销毁局部变量，例如带有析构函数的C ++变量或类似ARC的语言中的强/弱引用；
    * 释放分配在堆上的局部变量，比如Swift中捕获的变量或ObjC中的`__block`变量；
    * 执行scope-specific的终止代码，例如C＃的`using`或Java / ObjC的`synchronized`语句；
    * 执行特别的清理块，如Java中的`finally`块或Swift中的`defer`操作。

针对栈中需要清理或者处理错误再或者二者皆有的特定调用frame，我们将其称之为**interesting frames**

##### Implicit manual propagation

这种策略是隐式生成检查错误的代码并将它们插入到堆栈中，这些代码是模仿程序员在手动传播下编写的代码。例如，函数调用可以在一个特殊的结果寄存器中返回一个可选的错误;调用者将检查这个寄存器，如果返回错误，unwind stack，并向上抛出同样的错误值。

由于生成的代码要显式的表现出传播和unwind的过程，因此与其他方法相比，此策略对非错误路径的运行时性能会造成严重影响，并且需要更多代码来进行显式unwind。检测错误所涉及的代码逻辑分支通常很容易判断，因此在热代码中，对性能的直接影响很小，但是减少code locality会很大程度上影响总体性能。毕竟大部分代码都不是热代码。

除非出现在尾部位置，否则即使uninteresting frames也会经历这个检查。（也不需要真正的尾部调用；错误传播不会跳过任何事情。）函数必须在返回之前做一些附加的设置工作。

好的方面是，除了代码大小的影响之外，错误路径不会受到任何严重的影响。但是，代码大小的影响可能是巨大的：沿着错误路径传播有时需要大量重复代码。

因此，此方法对错误路径与非错误路径的的影响是差不多相同的，尽管它需要谨慎优化两条错误路径的代码大小。

##### `setjmp`/`longjmp`

另一个策略是动态维护interesting frames的线程本地堆栈。interesting frame内的函数必须像`setjmp`这样将有关其上下文的信息保存在缓冲区中，然后在运行时注册该缓冲区。如果作用域正常返回，则对应的缓冲区会被释放。传播从恢复 interesting-frames堆栈顶部的上下文开始。 捕获并处理异常的地方称为“landing pad”。

这样做的好处是uninteresting frames什么都不需要做。恢复上下文时会略过这些区域。对于错误路径和非错误路径这都是更快捷的方式。当前策略还可以做进一步优化，以便隐式地消除对错误的检测（与`setjmp `不同）：给landing pad使用一个更恰当的地址，以便将传播的错误直接恢复到目标位置。

这种策略的缺点是保存上下文和注册frame是有开销的：

* 注册frame需要访问线程本地状态，这在我们的平台上意味着函数调用，因为我们不愿意在ABI中提交更具体的内容。
* 跳过任意frame会使callee-save register失效，所以注册frame时也有必要同时保存它们。在有很多callee-save register的地方调用协议会引起很大的开销。但是，只有在可能从 landing pad恢复正常执行时，才需要这样做：如果landing pad仅进行清理然后重新传播，则这些寄存器将被保存并进一步还原。
* 像c++、ObjC ARC和Swift这样对大量局部变量必须（non-trivial）清理的语言，这也就意味着interesting frame中有很多的函数。这意味着保存上下文的成本更高，即使跳过uninteresting frames也不是很好的优化。
* 同样，这些语言中的函数通常具有许多不同的清理或处理程序再或者两个都有。例如，每个新的不可忽视的（non-trivial）变量都可能引入新的清除。该函数必须为每次清理(成本非常高)注册一个新的landing pad，或者跟踪它的当前位置，使函数内的landing pad能够知道它的作用域。

这种方法可以与下面的展开方法混合使用，可以让interesting-frames堆栈能够将帧中的清理抽象描述，而不仅仅是在某个地方恢复控制并期望帧将其清理。对于大部分只需要清理然后继续抛出错误的帧，这可以大大减少代码大小的影响。它甚至可以完全消除对landing pad的依赖。

在iOS/ARM32上平台上的ObjC/C++异常系统有点像刚才提到的混合方法。传播和清除代码在函数中是显式存在的，但是“personality”信息的已注册的上下文包含来自unwinding tables，上下文信息决定了是否需要在 landing pad中处理异常。它优化了`setjmp`实现，减少了一些上下文缓存和如上所述的分支线程。

在老一代的平台上（如PPC和i386），ObjC使用了标准的`setjmp`/`longjmp`函数。每个受保护的域单独保存上下文。所有这些都是以非常不安全的方式实现的，存在内联时无法正常运行。

总的来说，这种方法需要在interesting frames中的函数的无错误路径上做很多工作。鉴于我们期望在Swift中大范围使用interesting frames的函数，因此理论上来说，我们不会考虑这种实现方案。但是，它是iOS/ARM32上平台上的ObjC/C++异常的实现方法，因此我们至少需要和它交互。	

##### Table-based unwinding

最后一种方法是side-table stack unwinding。这就要求在仅依靠到调用的返回地址和当时的堆栈指针就可以在系统中准确找unwind任意函数的方法。

在我们的系统中，有如下过程。通过指令指针，the system unwinder查找加载该函数的链接快照（可执行文件或dylib）。链接快照包含一个特殊部分，即包含多个unwind table的表，这个特殊的表按照unwind table在链接快照中的偏移进行索引。每个non-leaf函数都应该在这个表中有一个entry，这个entry提供了足够的信息来从任意调用站点unwind函数。


此查找过程的开销是非常大的，主要是因为它必须遍历栈内所有数据，直到找到确实可以处理当前错误的对象。这样做的效率是非常低的。但是，由于在非错误路径上不需要显式的设置代码，因此某些场景下这种方法是“零开销”的。这有点用词不当，因为它确实会影响无错误时的性能。第一，为了将重定位数据解析为unwind table可以使用的符号，需要有少量的装载工作。第二，错误路径需要在函数中添加一些可能永远不会被执行的代码，这会降低code locality。第三，错误路径可能使用非错误路径将丢弃的信息。第四，最后，unwind table可能会占用很大内存，尽管这通常只是一个二进制大小的问题，因为它们被精心设计处理过的，所以除非抛出异常，否则不需要从磁盘上加载。但是不管怎么说，这已经接近“零开销”了。

基于以上来说，unwind frame的具体步骤是：

* 确定函数是否能处理错误
* 清除任何需要销毁的interesting scope(进入处理程序或离开函数)。
* 如果函数被完全展开，恢复函数可能改变的所有的callee-save register。

具体步骤是语言自定义的，因此该表还包含了语言自定义的personality信息，其中包括对解释它的函数的引用。这种设计意味着unwinder非常灵活的。它不仅可以支持任意语言，还可以为同一语言支持不同语言自定义的展开表布局。

我们当前的C ++和Objective-C自定义记录仅包含足够的信息来决定（1）当前帧是否可以处理异常，以及（2）如果不能处理异常，是否进行清理。如果以上两条都可以满足，那么它会恢复lading pad的上下文，langding pad不但会执行清理还会找到处理错误的帧。这种方法通常在函数中需要与隐式手动传播一样多的代码。但是，我们可以对许多常见情况进行优化，简单来说就是让析构函数自动调用清理。以下是一个landing pad的伪代码：

```
void *exception = /*...*/;
SomeCXXType::~SomeCXXType(&foo);
objc_release(bar);
objc_release(baz);
_Unwind_Resume(exception);

```

unwind table会有一条类似这样的记录：

```
CALL_WITH_FRAME_ADDRESS(&SomeCXXType::~SomeCXXType, FRAME_OFFSET_OF(foo))
CALL_WITH_FRAME_VALUE(&objc_release, FRAME_OFFSET_OF(bar))
CALL_WITH_FRAME_VALUE(&objc_release, FRAME_OFFSET_OF(baz))
RESUME

```

并且这个函数其实不需要代码。这通常会降低错误路径的速度，因为析构函数将不得不解释这种mini-language，但是它将所有开销从函数中移出并移到错误表中，从而使错误表更紧凑。

C++中也同样采用类似的方式。

#### Clean-up actions

许多语言都有内置的语言工具，可以在退出作用域时执行全部的清理工作。这样做有两个好处。 首先，即使不考虑错误传播，它也能起到“作用域保护”的作用，确保在由于`return `，`break `或`continue `语句而提前退出作用域时进行清理。否则，程序员必须在所有这些地方小心翼翼地手动清理内容。第二个好处是，在面对自动传播时，清理变得容易了。自动传播会在作用域之外会创建许多隐式的control flow路径，这会迫使程序员必须显式使用catch-and-rethrow代码块来全面覆盖这些路径，这也太扯了。

在这些语言特性中，有一种内在的约束（tension），也就是将显式清理代码按执行顺序放置，并将它放在待清理的代码附近。前者意味着通过从上到下顺序阅读代码，你可以了解什么时候会执行怎样的操作，同时你不必担心在作用域末尾会隐式插入代码。后者使得确定最终是否会进行清理变得很容易，你并不需要再找到finally代码块，并分析里面要执行的操作。


##### `finally`

Java、Objective-C和许多其他语言允许`try`语句带有一个`finally`子句。这个子句属于普通作用域（ordinary scope），并且可以执行任意操作。当前控制（controlled）域（包括全部catch子句）无论以哪种方式终止，都会执行`finally`子句：执行到尾部直接结束，直接从某个分支或者return结束，异常退出。

`finally`是相当笨拙和冗长的语言功能。它将清理代码解耦出来(尽管这样做有好处，如上所述)。它添加了很多大括号和缩进，所以添加新的清理编辑可能需要重新格式化大量代码。当同一个作用域需要多次清理时，程序员要么将清理代码放在同一个`finally`块中(这样清理操作可能会提前终止该块)，要么将它们堆在单独的块中(这让原本简单易懂的代码流变得很晦涩)。

##### `defer`

Go提供了一个defer语句，该语句中可以加入需要在函数结束前需要执行的代码。 （有关更多详细信息，请参见Go的总结。）

这将把语句内部的代码延迟执行，这使得读者可以立即看到将要结束的清除操作（但是，如上所述，这样做有缺点）。它的体积很小，这是个不错的优点，因为大多数的延迟操作都很短。它还允许添加多个操作，而不需要增加多余的嵌套。但是，函数退出语义（semantic）会使搜索介入清除操作的问题进一步恶化，并且它们在捕获局部变量的值时引入了语义（semantic）和性能问题。

##### Destructors

C ++允许类型定义析构函数，当函数离开作用域时将调用这些析构函数。

它们通常直接用于清除该类型值的所有权或其他常量。例如owning-point类型会在析构函数中释放它的值，而哈希表类型将销毁它的全部entry并释放其缓冲区。

但是它们通常也只用于隐式析构函数调用，扮演“作用域保护”的角色，以确保在当前操作完成之前完成某些操作。举个我关心的例子，编译器在解析局部作用域时可能使用这样的保护，以确保即使函数由于解析错误而提前退出，也可以从作用域链中删除期间新增的数据。不太好的是，由于类型析构函数是C ++进行此类清理的唯一工具，因此引入临时清理代码需要每次都定义一个新的类型。

与上面的选项相比，析构函数的独特优势在于，析构函数可以与表达式计算期间创建的临时值绑定。

通常，当表达式中执行某些“acquire”操作时，需要进行清理操作。`defer`和`finally`直到下一条语句执行后才生效，如果在acquire后可以注入代码，则会产生原子性问题（针对`finally`，假设在`try`之前获取，如果相反，在`try`中acquire，那么一定有一些东西触发清理，这也会有相同的原子性问题）。

相反，如果acquire操作创建临时对象的同时为它创建了带有清除功能的析构函数，则该语言会自动保证这种原子性。这种模式称为“resource acquisition is initialization”或“ RAII”。在RAII中，所有需要清理的资源都使用用户自定义的析构函数小心地封装在类中，而构造此类对象的过程等同于是获取基础资源的过程。

Swift不支持在值类型上使用用户定义的析构函数，但是它却为类和`deinit`方法提供了类似RAII操作的支持。即便如此，在保证对象生命周期上用户也应该格外小心，因为Swift通常不确保对象的销毁顺序。

如果存在一个可自定义的“资源”，并且有人已经封装了能够返回析构对象的acquisition API，这种情况下，使用RAII是很方便的。对于其他场景，理性的程序员可能不愿意定义新类型并且只为资源申请这个单一目的封装API，因此可能要采取其他特殊手段。

## Survey

### C

C语言并没有一个公认的错误处理方案。`setjmp`和`longjmp`中有内联展开机制，这不是一个靠谱的解决方案。在实际应用中，主流的做法是，使用一些异常值如空指针或越界数字来编码失败结果。异常值通常是基于函数的，有时甚至是基于参数或状态的。

对于调用者来说，在函数最后使用通用标签传播失败（就是`goto fail`）是很别扭的做法（在某些代码库中）。这是因为没有内在的语言支持来确保在抛出作用域前完成必要的清理工作。

### C++

C++有异常。异常可以是语言中的任何类型。传播类型仅与声明相关；间接函数指针通常被假定能够抛出异常。传播类型用于允许函数具体说明它们可能抛出的异常类型（`throws (std::exception)`），但是不提倡这样做，只要标示出函数是否可以抛出错误就好了（`noexcept（false）`）。

C++希望使内存不足成为可恢复的条件，这也就意味着需要内存分配可以抛出异常。因此，语言必须假定构造函数可能抛出异常。由于构造函数的调用是普遍的、隐式的，这也就意味着默认情况下所有函数都可以抛出异常。由于许多报错点（error site）是隐性的，因此不得不使用自动无标记传播（automatic unmarked propagation）。在这样的背景下，离开作用域前执行清理的唯一手段是让编译器自动执行。C++程序员常用的清理方式是将作用域内的所有清理操作转移到局部变量的析构函数中。甚至有时创建此类局部变量就是为了以这种方式设置清理操作。

不同的错误点（error site），会引发一系列不同的清理操作，同时错误点（error site）是非常多的。实际上，在c++11之前，默认情况下编译器不得不假定析构函数也是可以抛出的异常的，因此清理实际上会导致产生更多的错误。这会导致异常机制的代码量异常庞大，即便是那些不直接使用它们并且不想从内存不足的异常中恢复的系统也会遇到同样的问题。由于这个原因，许多C ++项目都明确禁用了异常并采用其他错误传播机制，对此尚无广泛共识。

### Objective-C

Objective-C具有一流的异常机制，有与java相似的类型集合：`@throw` / `@try` / `@catch` / `@finally`。异常值必须是Objective-C类的实例。ObjC在异常传播过程中对少量的隐式frame做了清理：释放`@synchronized`持有的锁，删除`__block`变量的栈副本，销毁ARC`__weak`变量。但是，ObjC不会释放保存在局部变量中的对象指针，即使在ARC（默认情况下）下也是如此。

过去，Objective-C异常机制是通过运行时管理的`setjmp`，`longjmp`和线程本地状态实现的，但是现在我们只在i386平台上支持这种异常机制，所有其他平台都使用`零成本`的方案，即使用C++异常 。

Objective-C异常通常仅用于不可恢复的环境，类似于上面我所说的“故障（failures）”。 这种环境下有几类主要的异常，api使用异常来报告错误。

相反，Objective-C主要使用手动传播，主要使用`NSError **`类型的输出参数。是否调用失败与是否在参数中写入非nil的错误无关。允许调用成功，并将错误对象写入参数（应将其忽略），并在不创建实际错误对象的情况下报告错误。相反，调用是否失败会在真实的返回值中体现。最常见的规则是`false` 或空对象结果表示错误。但是灵活的程序员也会使用很多其他规则，因为确实存在一些api，其中返回空对象也表示有效。

同时，CF api也有自己的一套单独的约定。

因此，我们可以遇见，逐步改善CF / Objective-C交互将是一个漫长而痛苦的过程。

### Java

Java具有一流的异常机制,具备unmarked automatic propagation的特点：`throw` / `try` / `catch` / `finally`。异常值必须是`Throwable `的子类实例化的值。传播通常采用静态强制的类型，默认情况下，调用不能抛出异常，`Error`和`RuntimeException`的子类除外。采用这种方案的最初目的是为了将这些类用于严重的运行时错误（`Error`）和在运行时被捕获的异常(`RuntimeException`)。在我们的方案中，这两种我们都将其归类为不可恢复的故障（failure）；从本质上讲，Java想推广一种完全静态执行的模型，在这种模型中，必要时仍然可以处理真正灾难性的问题。不幸的是，这些想法似乎没有很好地传达给开发者，结果就是一团糟。

Java允许方法自定义抛出的异常类型。在我看来，异常通常分为两类：

* 一些非常明确的异常类型，针对这些异常类型，调用者知道如何通过特定的操作来查找和处理。通常，这些是显而易见的、可预测的错误情况，例如主机名无法解析或字符串格式不正确。
* 还有很多非常模糊、黑盒的异常类型，无法真正有效地响应。例如，如果某个方法抛出`IOException`，则调用者只能传播它并中止当前操作外。

因此，如果你有能力处理少量的特定失败，使用特定类型错误是有帮助的。一旦异常列表中包含任意一种黑盒类型，这就是一个开放性的话题了。

### c# 

C＃的异常模型几乎与Java的完全一样，只是它没有类型：假设所有方法都可以抛出异常。因此，它会有一个比Java更简单的继承关系，所有的异常都继承自`Exception`。

继承关系的其他部分对我来说没有任何意义。许多类直接继承自`Exception`，但其他许多类继承自`SystemException`的子类。`SystemException`似乎没有任何逻辑分组:它不但包括所有运行时断言异常，也包括核心库中抛出的所有异常，包括XML和IO异常。

C＃有`using`语句，该语句对于在明确的作用域内绑定某些内容然后将其自动放置在所有路径上很有用。它只是建立在`try` / `finally`之上。

### Haskell

Haskell提供了三种不同的常见错误传播机制。

第一种方式，与其他很多函数式语言一样，它支持`Maybe`类型的manual propagation。函数通过返回`Nothing`来表示不会有有效的结果。这对函数式子集库来说是函数最常见的失败方法。

第二种方式，`IO` monad也提供了unmarked automatic propagation的纯异常。这些异常只能作为`IO`操作处理，但是没有类型：没有方法表明`IO`操作是否可以抛出。异常可以作为`IO`操作或者普通的延迟函数计算被抛出。后面一种情况里，只有某些特定条件下的计算才会抛出异常。		

第三种方式，`ErrorT`monad转换提供了typed automatic propagation。有趣的是，由于`ErrorT`的唯一原生计算指令是`throwError`，只有在将要抛出`ErrorT`时才会使用。其他的计算必须要显式的添加到monad中。`ErrorT`使用了marked propagation，因此所有不能被显式抛出的异常都需要用`left`标记：

```
prettyPrintShiftJIS :: ShiftJISString -> ErrorT TranscodeError IO ()
prettyPrintShiftJIS str = do
  lift $ putChar '"'     -- lift将IO计算转换为Error计算
  case transcodeShiftJISToUTF8 str of
    Left error -> throwError error
    Right value -> lift $ putEscapedString value
  lift $ putChar '"'

```

### Rust

Rust区分了*failures*和*panics*。

panics是一种断言，是为我所说的logic failures而设计的;它是不可恢复的，会立即崩溃。	

failure意味着函数不能产生你可能预期的值。Rust鼓励你使用`Option <T>`（针对简单的情况，例如我所描述的simple domain errors）或`Result <T>`（除了会携带error外其余特性与`Option <T>`一致）。上述两种情况都是typed manual propagation，尽管Rust至少提供了一个标准宏，该宏包装了`Result<T>`的常见模式-匹配-返回模式。

Rust中的错误类型是一个非常简单的协议，非常类似于本文的所建议。

### Go

GO通常将可能失败的函数的最终结果以error result的形式返回。调用者应该手动检查结果是否为空。Go使用typed manual propagation。

Go中的错误类型是一个`error`接口，其中一个方法返回错误的字符串描述。

Go提供`defer`语句：


```
defer foo(x, y)

```

参数必须是一个函数调用（可以是方法调用，也可以是在结束时立即调用的闭包调用）。所有操作数都将立即求值并在延迟操作中捕获。在函数(通过任何方式)退出之后，所有的延迟操作都立即按照后进先出的顺序执行。由此可以看出，这与函数退出有关，与作用域退出无关，所以你可以自定义任意数量的延迟操作，起到隐式清除栈的作用。总的来说，这是一种不错的方式，可以执行临时清除操作。

它也是另一种流行的错误传播的关键部分，这种错误传播本质上是untyped automatic propagation。如果你调用`panic`以及某些与panic有类似效果的内联操作(如数组访问)，它会立即unwind stack，并在运行过程中运行延迟操作。如果函数的延迟操作调用`recover`，则`panic`停止，函数内的延迟操作正常执行结束后，函数返回。延迟的操作可以写入命名返回值，从而使函数可以将panic错误转换为正常的最终结果错误。

除非你真正理解它，一般不要将panic暴露在公共API中;recoverable errors应该与结果无关。

### Scripting languages

脚本语言通常都使用（显然是无类型的）automatic exception propagation。可能是因为使用无类型语言进行manual propagatio非常容易出错。它们几乎都适合于标准C ++ / Java / C＃风格的`throw` / `try` / `catch`。 但是，Ruby使用了不同的关键字。

不过，我觉得Python使用异常要比大多数其他脚本语言多得多。

# Proposal

## Automatic propagation

Swift应该使用automatic propagation错误，而不是依赖于程序员手动检查并返回错误。就常见的错误处理任务而言，automatic propagation所需的代码样板会少很多。这会引入显式control flow的问题，但是我们可以使用marked propagation解决该问题；如下所述。

我们没有理由抛弃使用传统的`throw` / `catch`。当然也有其他选择，比如`raise`/`handle`。理论上来说，这种转变会让Swift与以往的异常处理产生差异。从其他语言转过来的人对异常处理有很多预期，这些预期可能并不适用于Swift。然而只要我们的错误模型与标准的异常模型足够相似，人们就会不自觉的将它们联系起来；因此不需要去解释我们将要做的事情。所以，使用不同的关键字似乎也没什么大不了的。

因此，Swift应该提供一个`throw`运算式。它需要一个类型为`Error`的操作数，并按照一定形式抛出任意类型。它的动态行为是将控制权转移到满足该操作数处理条件的的最内层闭包`catch`子句。一个简单的例子。

```
if timeElapsed() > timeThreshold { throw HomeworkError.Overworked }

```

`catch`子句包含匹配错误的模式。我们希望将`try`关键字用于marked propagation，这么做更合适，因此`catch`子句将被附加到一个通用的`do`语句中：

```
do {
  ...

} catch HomeworkError.Overworked {
  // a conditionally-executed catch clause

} catch _ {
  // a catch-all clause
}

```

Swift还应该提供一些用于manual propagation的工具。我们的库应该提供一个标准的类似Rust的`Result <T>`的枚举，以及一​​组丰富的工具，例如：

* 一个函数，用于判断产生错误的闭包并将结果捕获为`Result<T>`。
* 一个函数，通过返回`Result <T>`的值或在当前上下文中传播错误来解析`Result <T>`
* 一个待实现的库，用来在合适的时机传递`Result<T>`。
* dispatch_sync的重载，它接受产生错误的闭包并在当前上下文中传播错误。
* 待补充

## Typed propagation

Swift应该采用静态强制类型传播(enforced typed propagation)。默认情况下，函数不应抛出异常）。编译器应该拒绝在禁止抛出异常的上下文中调用可以抛出异常的函数。

函数类型应表明函数是否抛出异常。即使是头等（first-class）函数值也需要跟踪。不能抛出异常的函数是抛出异常函数的子类。

如果函数是可以抛出异常的，可以在函数定义或者函数类型上加入`throws`子句:

```
// 不允许这个函数抛出异常
 func foo() -> Int {
   // 然而这是个语义错误
   return try stream.readInt()
 }

 // 允许这个函数抛出异常
 func bar() throws -> Int {
   return try stream.readInt()
 }

 // 把'throws'放在箭头前很明智
 // 函数类型和隐式（）结果类型的语法一致。
 func baz() throws {
   if let byte = try stream.getOOB() where byte == PROTO_RESET {
     reset()
   }
 }

 // 'throw'出现在函数类型的位置上。
 func fred(_ callback: (UInt8) throws -> ()) throws {
    while true {
      let code = try stream.readByte()
      if code == OPER_CLOSE { return }
      try callback(code)
    }
 }

 // 它仅适用于curried函数的最里层的函数
 // 此函数的类型：
 //   (Int) -> (Int) throws -> Int
 func jerry(_ i: Int)(j: Int) throws -> Int {
   // 在不能抛出异常的函数上使用'throw'并不是错误。
   return i + j
 }
 
```

在这里使用关键字的原因是，因为它更适合函数声明。函数声明的使用通常比函数类型的使用至少多一个数量级。在函数声明的所有其他标点符号中，标点符号很容易丢失或错误，尤其是`!`这类在参数末尾起作用的符号。在返回值旁边使用关键字是有意义的，因为它本质上是结果的一部分，程序员可以一目了然的看到这两部分。关键字出现在箭头之前的原因很简单，因为在函数和初始化声明中，箭头是可选的(以及返回类型的其余部分)。返回类型的存在会影响关键字的位置，这是很愚蠢的做法，并且会使添加非空返回类型感到别扭。`throws`关键字本身应该是描述性的，它也能很好的代表抛出异常这个动作，并且与函数浑然一体。因此，`throw`变成`throws`;如果我们用`raise`来代替，那就会是`raises`，我个人觉得它不合适（unappealing），因为我不能把它与抛出异常联系起来。

函数是否能抛出异常（throw）不能作为函数重载的条件。也就是说，以下不合法的：

```
func foo() { ... } // 在无法抛出异常的上下文中调用
func foo() throws { ... } // 在可以抛出异常的上下文中调用
```

能够根据带参函数是否抛出异常来重载高阶函数是很有意义的；不难想象，如果算法不需要担心异常，可以更高效地实现它们。（然而，我们并不想特别鼓励重复模式。如果主类型检查能够准确地决定函数值是否能抛出异常，那么事情就简单了。）

Typed propagation检查通常可以在经过类型检查的函数主体的第二遍操作中执行：如果不允许某个函数抛出，遍历其函数体并验证是否没有`throw`表达式或调用。如果必须要标记全部可以抛出异常(throw)的调用,可以在类型检查之前完成此操作，以便从语法上确定函数是否可以抛出异常(throw)；当然，后面的步骤仍然是必需的，但是这样做的能力极大地简化了类型检查器的实现，如下所述。由于要把这部分工作加入排期，可能会砍掉某些类型系统的功能。(重要的是要理解使用marked propagation不是目的，目的是为了更方便的实现上述操作。)

对于声明函数的高阶使用来说，准确地决定函数值是否可以抛出异常很容易。但是通常对匿名函数来说就会出现问题。我们不想要求闭包被明确地标记为throwing或non-throwing。但是fully-accurate inference算法需要类型可检查的函数体，我们不能总是抛开闭包的上下文对匿名函数进行类型检查。因此，可以在类型检查之前进行一次遍历（pass），以从语法上推断闭包是否抛出，然后在类型检查之后进行第二次遍历（pass），以验证该推断的正确性，以上两步可以为我们提供判断依据。这可能会破坏某些合理的代码结构，但是多次遍历可以让我们尝试不破坏某些特定case。

Typed propagation对所有类型的多态性都有影响：

### Higher-order polymorphism

我们应该使编写具有多态性的高阶函数变得容易。 无论他们的参数是否抛出异常(throw)。这可以通过一种非常简单的方式来完成：如果一组命名参数中的任何一个可以抛出异常，则函数可以声明它是可以抛出抛出异常的。 例如（使用strawman语法）：

```

func map<T, U>(_ array: [T], fn: T throws -> U) throwsIf(fn) -> [U] {
  ...
}

```

不需要比逻辑或（disjunction）更复杂的逻辑运算符了。你可以构造出真正奇怪的代码，其中仅当函数在其中一个参数没有抛出异常时才抛出异常，但它是人为设计的，很难想象如果没有非常复杂的方法如何对它们进行类型检查。类似地，你可以构造这样一种情况，即函数是否可以抛出异常取决于其他参数的值，比如“我是否应该抛出异常”标志，但很难想象在语言中正确处理这种情况有什么重要意义。这个模式完全足以表达正常的高阶内容。

实际上，尽管上面的strawman语法允许函数具体说明是由于哪个函数参数的原因导致调用者抛出异常，大多数情况下，几乎不可能出现函数中所有的函数参数都可以抛出异常，一般可能只有一个函数参数会抛出异常。因此，最好仅使用一个`rethrows`注解，并且有一个模糊的规划以便在未来将其参数化。

这种传播检查是对通用传播检查器的简单扩展。普通检查器发现（see）不允许某个函数传播出去，并寻找传播点。 条件检查器会发现（see）一个函数具有条件传播子句，并在假定列出的函数没有抛出异常的情况下寻找传播点（包括查看任何条件传播子句时）。 该参数必须是`let`。

我们可能确实需要在第一个正式版本实现高阶多态性，因为短路运算需要我们用到它。

### Generic polymorphism

能够对操作是否产生错误进行 parameterize protocols和protocol conformances将很有用。缺少此功能意味着协议作者必须决定要么保守地允许抛出conformances，这会导致强制让遵守该协议的所有范性代码处理可能是假的错误，或者全面地禁止，这又会导致禁止本身可以抛出的类型的conformances。

有几种不同的方法可以解决这个问题，经过一些调查，我确信它们是可行的。不幸的是，第一个正式版本不包含这些方法。目前，标准库应该提供不能抛出异常的协议，尽管这会限制一些潜在的conformances。(值得注意的是，这种conformances在今天通常是不合法的，因为它们需要以某种方式返回一个错误结果。)

通用多态性和高阶多态的未来发展方向是，将错误传播作为一个通用的、用户可扩展的效果（effect）跟踪系统中许多可能的功能之一。类型系统可以检查在特定上下文中被允许的特定操作：阻塞操作只在阻塞上下文中允许。

### Error type

Swift标准库将提供一个接口非常简单的协议`Error`(在本文中没有描述)。 标准模版应该是被为定位一个`enum`类型的conformance：

```
enum HomeworkError : Error {
  case Overworked
  case Impossible
  case EatenByCat(Cat)
  case StopStressingMeWithYourRules
}
```	

`enum`提供了错误的命名空间，命名空间中包含可能出现的错误列表以及每个选项的可选值。

目前，一个domain中的错误列表是固定的，但允许将来以普通枚举弹性的方式扩展，并且该标准技术在将来会很好用。

请注意，这与包含error domain、error code和optional user data的`NSError模型`完全对应。我们希望将系统错误域导入为遵循这种方法并实现`Error`协议的枚举。NSError和CFError本身也会遵从`Error`协议。

客观表现层(仍然被固定)可以有效地将`NSError`作为`Error`导入，反之亦然。通过使用合法的类型名称作为domain key，使用枚举器作为error code，并将payload转换为user data，应该有可能将遵守`Error`协议的任意Swift`enum`转换为`NSError`。

在需要错误时分配内存是可以接受的，但是我们的表现层(representation)不应阻止优化器将`throw`直接转发给`catch`并删除中间错误对象。

## Marked propagation

Swift应该使用标记传播（marked propagation）：应该有一些轻量级的语法来修饰任何已知可以抛出异常的对象（当然，除了throw表达式本身）。

我们建议的语法是使用重定位`try`来修饰任意表达式：

```
// 这个try对readBool()起作用
if try stream.readBool() {

  // 这个try同时这些方法调用起作用
  let x = try stream.readInt() + stream.readInt()

  // 这是个语义错误，它需要try
  var y = stream.readFloat()

  // 这是合法的，try对整个语句起作用
  try y += stream.readFloat()
}

```

开发者可以将`try`写在括号内、特定的参数上、列表元素上以此来严格`限定作用域`。

```
// 语义错误：尝试仅修饰括号表达式。
let x = (try stream.readInt()) + stream.readInt()

// try对第一个数组元素起作用 。
// 当然开发者可以把try写在外部，以对整个数组起作用
let array = [ try foo(), bar(), baz() ]
```

一些程序员可能希望这么做来让特定的抛出异常调用非常清晰。另一个开发者可能更倾向于知道语句中的哪些内容会抛出异常。

我们曾经还考虑了将标记放入调用参数的可能性，例如：

```
parser.readKeys(&strings, try)
```

只要抛出异常的调用在语法上遵守方法调用的规则就可以运行，；这包括对自由函数，方法和初始化程序的调用。但是，它实际上要求Swift禁止操作符、属性、下标访问器抛出异常，这可能不是一个合理的限制，尤其是对于操作符。这也有点不自然，它迫使用户标记每一个call site，而不能一次标记语句中的全部内容。

标记自动闭包会遇到问题。在大多数情况下，我们想假装自动闭包的表达式是在闭包的上下文中计算的;我们不想同时标记自动闭包内的调用和使用自动闭包的函数的调用！我们应该通过类型检查来检查这种模式:调用一个有`throwsIf`修饰自动闭包参数的函数。

```

autoreleasepool {
  let string = parseString(try)
  ...
}

```

没有标记对autoreleasepool的调用，因为这会破坏编写类似于语句的函数的功能。但是，在这些跟踪闭包使用和真正的内联语句之间还有很大的区别，比如return、break和continue的行为。将函数标记为类似声明（statement-like）的属性是解决这两个问题的必要步骤。然而，在闭包中完全按照预先设计实现它是不容易的。

### Asserting markers

类型化传播是一种假设检验（hypothesis-checking）机制，因此存在误报的常见问题。 （当然，基本的健全(soundness)可以消除误报：编译器应强制程序员处理所有错误源。）在这种情况下，误报意味着API声明会抛出异常，但实际上不可能动态地出现错误。

例如，如果图像不存在、连接失败、文件数据格式不正确或出现其他众多问题中的任何一个，那么从URL加载图片的函数通常会被设计成可以产生错误的。我们通常期望程序员处理该错误。但是程序员可能会合理地使用相同的API来加载一个完全由他们控制的图片，比如图片来自于他们程序的私有资源。针对“关闭”此类调用的错误检查，我们不应该在语法上做限制。

重要的一点是，我们不想太轻易忽略错误。即使有有意义的堆栈信息跟踪了错误，但是这次报错的完整上下文已经丢失了，并且很难恢复，因此忽视错误通常会带来很糟糕的调试经历。被忽视的错误可能会导致事情恶化，在抽象层“无害地(harmlessly)”忽略的错误可能会在其他地方导致另一个错误。 当然第二个错误也可以忽略，等等。但是这只会让程序变得越来越难理解和调试，也会导致日志文件被越来越多的被忽略的错误的碎片信息占满。最后，忽略错误会导致大量类型安全和安全性问题，因为这会让程序携带大量的垃圾数据和被破坏的常量运行。

相反，我们只想（相对）简单地将静态问题变成动态问题，就像assertions和！操作符所做的那样。当前，这需要一个显式的操作，否则我们将完全弃用typed propagation。并且他应该专门用在调用上，这样程序员就必须对每个操作都单独做决策了。但是我们已经有了一个明确的，专门用于call sited的注解：`try`操作符。因此，基于以上得出的解决方案是允许使用`try`的变体，它可以断言错误不会从它的操作数中被抛出。基于我们目前的设计语言可以得出结论，使用通过使关键字`try！`使用通用的“be careful, this is unsafe”标记。

`try!`的是不是太容易编写了？毕竟这是一个不安全的操作。我立马就能反驳这个疑问，在编写这方面上，它不比`!`操作符要差。与`！`一样，谨慎的程序员可能希望对其进行更深入的研究，并且你可以很容易地想象到使用它的代码库肯定会在注解中解释。但更重要的是，就像`!`只是静态不安全，而且当程序员出错时，它肯定会失败。因此，尽管你可以很容易地想象（并推演）粗心的程序员随意使用它来让类型检查器放水，但是这对整个程序来说不是个可靠的手段：最终程序员还是必须得学习怎么使用这个功能，否则他们的程序将无法运行。

虽然粗心的程序员使用`try!` 确实破坏​​错误安全性，但是，比起隐式传播的替代方案，这种不安全的方案更好。一个粗心的程序员不会因为我们没给他提供相应的功能而写出好的错误处理代码。相反，他们会使用`do/catch`代码块，那么处理的压力就转移到了静默捕获错误上，即便如此，开销也比断言和记录要小。

未来的版本中，当我们增加对universal error的支持时，我们也会重新考虑`try!`的行为。一种可能是`try!`应该将它的操作数作为universal error抛出，这将允许紧急修复。或者，我们可能尝试`try!`断言，即使是unversal erros也不会抛出。这就要求在两种错误之间提供一个更具通用性的语言类型。目前我们不需要为此花太多心思。

## Other syntax

### Clean-up actions

Swift应提供一份特定（ad hoc）清理行动的声明。

与Go不同，我认为它应该与作用域退出，而不是和函数退出联系在一起。这很容易看出一组将在作用域退出时执行的`defer`操作：这就是该作用域的全部`defer`语句。相反，在Go中，你必须了解函数执行的动态历史。这还消除了与变量捕获相关的一些语义和性能问题，因为`defer`操作发生时作用域的内还没有清理。这么做也会有缺点，它不太符合“事务性”语法对所有操作都可以撤销的习惯，但是这种风格无论如何都会存在跨越函数边界的问题。

我认为`defer`是一个合理的名称，虽然我们也可以考虑`finally`。我将在本文的其余部分使用`defer`。

任意语句都可以放在`defer`后面，编译器应该拒绝可能提前中止的操作，无论是通过抛出异常还是通过`return`，`break`，`continue`。

例如：

```
if exists(filename) {
  let file = open(filename, O_READ)
  defer close(file)

  while let line = try file.readline() {
    ...
  }

  // close occurs here, at the end of the formal scope.
}

```

我们应该考虑提供一种简便的方法来标记仅在引发错误时才应执行`defer`操作。使用仅在操作结束时设为`true`的标志就是一种简便的方式。标记方法通常更有用，因为它允许针对任何提前退出（例如`return`）采取措施，不仅用于错误传播。

### using

Swift应该考虑提供一个`using`语句，该语句获取资源，将其保留固定的时间段，有选择地将其绑定到一个名称，然后在受控语句（controlled statement）退出时释放它。

`using`与`defer`有许多相似之处。它没有包含`defer`，`defer`主要作用于临时的和无标记的清理。但`using`对于类型定向清除的常见模式很方便。

我们不认为这是第一个版本中必不可少的功能。


## C and Objective-C Interoperation

最重要的是，Swift的错误模型要尽可能简洁地与Objective-C api交互。

通常，我们想尝试将可以产生错误的API以throw的形式导入。如果此方案行不通，我们将把API以普通non-throwing函数导入。如果将函数以throw的方式导入会导致对调用的过多改动，在这种情况下才会采取后一种安全的方式。 也就是说，如果开发人员在编写代码时假定API将作为throw导入，但实际上Swift无法以这种方式导入API，这就会导致代码不能编译的严重问题。

幸运的是，这对于错误输出参数的常见模式来说是正确的:如果Swift不能以throw的形式导入函数，那么它会保留输出参数，如果开发人员没有传递错误参数，编译器就会发出警告（complain）。但是，可以想象API以不同的方式返回错误的“实质”；比如一个仅设置`errno`的POSIX API。当此类API仅部分以throw的形式导​​入时，需要格外小心。

让我们深入了解下细节。

### Error types

`NSError`和`CFError`需要实现`Error`协议。通过使用限定类型名作为domain key、枚举器作为 error code并将payload转换为user data，应该可以将遵守`Error`协议的任意Swift`enum`转换为`NSError`。

将系统枚举识别为错误域（error domains）是一个注解问题。 在第一个正式版中，Swift只会对一些常见域名（doamins）进行特殊处理。

### Objective-C method error patterns

目前来说，ObjC中最常见的错误模式是使用自动释放的`NSError **`输出参数。我们不建议在缺少此类参数时还将函数自动以`throws`的形式导入。

如果API中包含有`NSError**`而不打算将其用作错误输出参数，API一定要对其进行标记。

#### Detecting an error

以下很多方法可以获取用来判断是否产生错误的某种关键结果：

* 最常见的模式是`BOOL`结果，其中false表示错误产生。 这看起来是很明确的。Swift应该将这些方法导入，就像它们返回了`Void`一样。

* 指针结果也很常见，其中`nil`结果通常表示错误产生。我了解到此规则有一些特殊情况，也就是`nil`是有效的，很显然，调用者的意图是检查非`nil`错误。不过，我还没有在Cocoa中找到这样的API。我曾经引用的已经声明的API确实存在非空结果，但是会通过带有BOOL值的输出参数返回结果。因此，对于Objective-C来说，默认`nil`结果代表错误产生似乎是一个明智的决定。 但是，CF可能是另外一种情况。

当结果为`nil`表示错误产生时，Swift应该将方法以返回非可选结果的形式导入。

* 少数CF API返回`void`，据我所知，针对这些API，调用者应该检查非`nil`错误。


对于其他哨兵（sentinel）情况，我们可以考虑添加新的clang属性，以告诉编译器哪个是哨兵（sentinel）：

* 有几个API会返回`NSInteger `和`NSUInteger `。至少其中一些API在错误产生时会返回0，但是这似乎不是一个合理的通用假设（assumption）。
* AVFoundation提供了两个返回`AVKeyValueStatus `的方法。如果API返回`AVKeyValueStatusFailed `就代表错误产生。这很有趣，至少它不是零。

clang属性将标明如何判断错误的返回值。例如：

```
+ (NSInteger)writePropertyList:(id)plist
                      toStream:(NSOutputStream *)stream
                        format:(NSPropertyListFormat)format
                       options:(NSPropertyListWriteOptions)opt
                         error:(out NSError **)error
  NS_ERROR_RESULT(0)

- (AVKeyValueStatus)statusOfValueForKey:(NSString *)key
                                  error:(NSError **)
  NS_ERROR_RESULT(AVKeyValueStatusFailed);

```

我们还应该提供一个Clang属性，该属性标明判断错误的正确方法是检查输出参数。这两个属性都可能被静态分析器使用，而不仅仅是用在Swift。 （例如，他们可以尝试检测无效的错误检查。）

在我看来使用常量就能满足需求。但是如果必须将参数泛化为简单的表达式，也是有可能实现的。

#### The error parameter

对于带有`NSError **`输出参数的Objective-C方法，容易理解的导入规则是简单的用throws标记并删除掉对应的输出参数短语。`NSAttributedString `的一个方法里就有一个例子：

```

- (NSData *)dataFromRange:(NSRange)range
       documentAttributes:(NSDictionary *)dict
                    error:(NSError **)error;

```

导入结果如下：

```
func dataFromRange(_ range: NSRange,
                   documentAttributes dict: NSDictionary) throws -> NSData
```

然而，滥用这个规则会使Objective-C交互出现问题，因为多个方法可以通过同一种方式导入。如果原始的Objective-C声明可以从Swift声明中明确地重新构建，那么编译器和程序员都能更好地理解这个模型。

以下两种情况容易产生歧义：

* 错误参数可能出现在selector的任意位置。 也就是说，`foo：bar：error：`和`foo：error：bar：`都会以`foo：bar：`的形式导入。               
* 错误参数可以有一个任意的selector chunk;也就是说，在导入之后，`foo:error:`和`foo:withError:`都会以`foo:`的形式导入。

为了达到重建的目的，我们仅在`error`参数为最后一个参数且相应的selector为`error：`或第一个chunk时应用规则。根据经验，除了以下两组API，其他公共API都适用上述规则：

* `NSObject`的`ISyncSessionDriverDelegate`分类声明了六个这样的方法

```
- (BOOL)sessionDriver:(ISyncSessionDriver *)sender
        didRegisterClientAndReturnError:(NSError **)outError;
```

幸运的是，这些委托方法在Lion中都已弃用，而Swift目前也不导入已弃用的方法。

* NSFileCoordinator有六种方法，其中`error：`是倒数第二个参数，其后是block参数。 据我所知这些方法还没有被弃用。

当然，用户的代码也可能不遵守这一规则。

我认为对于Swift来说，不将这些方法以`throws`的形式导入是可以接受的，将原始的错误参数保留在适当的位置，就好像它们没有遵循可理解的头部模式（an intelligible pattern in the header）一样。

此转换规则将从`NSDocument`导入像这样的方法：

```
- (NSDocument *)duplicateAndReturnError:(NSError **)outError;

```

导入成:

```
func duplicateAndReturnError() throws -> NSDocument

```

保留`AndReturnError `固然让我很别扭，但是我更不愿看到因此失去了自动重建Objective-C签名方法的能力。这种模式很常见，但是很难通用。考虑一下NSManagedObject的这个方法：
```
- (BOOL)validateForDelete:(NSError **)error;
```
导入结果如下：
```
func validateForDelete() throws
```

这看起来是一个很好的导入。


### CoreFoundation functions

CF api通常使用`CFErrorRef`，但存在两个问题。

首先，我们对错误对象的内存管理规则没有信心。它总是返回+1吗？

其次，我不太确定如何检测错误的产生：

* 有很多函数返回`Boolean`或`bool`。这些函数可能与Objective-C的约定保持一致：false表示错误。
* 类似地，有许多函数返回对象引用。同样的，我们需要一个规则来确定是否`nil`结果意味着错误。
* 有少量API返回`CFIndex`，所有API显然具有相同的规则，即零值表示错误。 （这些是序列化API，因此返回里什么也不写似乎是一个合理的错误。）但是就像Objective-C一样，这似乎不是一个合理的默认假设（assumption）。
* ColorSyncProfile有几个返回`float`的相关函数!显然，它们都应该通过判断错误结果是否已经写入来确定是否检查错误。

也有一些API没有使用`CFErrorRef`。例如，CoreVideo中的大多数`CVDisplayLink `会返回他们自己的`CVReturn `枚举类型，并且很多都不止一个错误值。显然，除非CoreVideo写一个overlay，否则这些API不会以throw的形式导入。

### Other C APIs

原则上，我们可以将POSIX函数以throwing函数的形式导入Swift中，以填补`errno`中的错误。但是，几乎很难想象通过自动导入规则来做这件事。我们很可能将它们包装在一个overlay中。

## Implementation design

对于我一直关注的显式类型错误的错误传播，应该使用implicit manual propagation。实现最好要考虑到非错误路径，可以采用将错误路径代码移动函数最后等等手段，甚至可以通过析构方法而不是直接内联代码来执行清理，但是我们也不能对性能造成严重影响。换句话来说，我们不应该使用table-based做unwind。

universal errors的传播应该采用table-based unwind。`catch`处理程序可以捕获这两者，并根据需要将unwind异常映射到Error值。精心设计的析构函数旨在解决Swift的特殊需求，我们可以将析构函数专属到unwind tables中，以此来减少对代码量的影响，因为通常情况下不需要加载这些表。


参考文章：

[异常概念和处理机制，try-catch-finally，throw和throws，自定义异常](https://www.cnblogs.com/wzy330782/p/5335394.html)

[setjmp()和longjmp()函数](https://www.cnblogs.com/zzdbullet/p/9932122.html)

[C语言中一种更优雅的异常处理机制](https://blog.csdn.net/hello_wyq/article/details/826312)

[为c语言实现异常处理机制（全）](https://blog.csdn.net/xombat/article/details/83162026?utm_medium=distribute.pc_relevant.none-task-blog-title-1&spm=1001.2101.3001.4242)

[实现C语言的异常处理机制 Implementing Exceptions in C](https://blog.csdn.net/lin_strong/article/details/82925190)

[C语言的不完整类型（incomplete type）和前置声明](http://www.voidcn.com/article/p-dsixnffi-k.html)

[c++ 异常处理（1）](https://www.cnblogs.com/catch/p/3604516.html)

[c++ 异常处理（2）](https://www.cnblogs.com/catch/p/3619379.html)

[C++异常机制的实现方式和开销分析](http://baiy.cn/doc/cpp/inside_exception.htm#%E6%A0%88%E5%9B%9E%E9%80%80%EF%BC%88Stack_Unwind%EF%BC%89%E6%9C%BA%E5%88%B6)

[浅谈C++ 异常处理的语义和性能](https://www.cnblogs.com/ly8838/p/3961119.html)

[C++ 中的异常和堆栈展开](https://docs.microsoft.com/zh-cn/cpp/cpp/exceptions-and-stack-unwinding-in-cpp?view=vs-2019)

[C++ exception类：C++标准异常的基类](http://c.biancheng.net/view/2333.html)

[C++/C++11中std::exception的使用](https://blog.csdn.net/fengbingchun/article/details/78303734)

[RAII](https://blog.csdn.net/quinta_2018_01_09/article/details/93638251)

[初探 Objective-C/C++ 异常处理实现机制](https://mp.weixin.qq.com/s/4Rcaee6kwWmrS3v_M9y0KQ)

[Objective-C try/catch异常处理机制原理](https://www.cnblogs.com/markhy/p/3169035.html)

[Swift 2 throws 全解析 - 从原理到实践](https://onevcat.com/2016/03/swift-throws/)

[关于 Swift Error 的分类](https://onevcat.com/2017/10/swift-error-category/)

[不算“真正的语言”？详说Swift 2.0中的错误处理](https://www.csdn.net/article/2015-07-01/2825095)

[窥探Swift编程之错误处理与异常抛出](https://www.cnblogs.com/ludashi/p/5206522.html)

[通过swift的错误处理窥视其设计理念](https://www.jianshu.com/p/b885a7672012)

[JavaScript错误处理完全指南](https://mp.weixin.qq.com/s/I9ZrCsoNo7jrOHj8a9UW1A)

[Swift 4.2 新特性详解 Conditional Conformance 的更新](https://www.jianshu.com/p/fee9f1146aed)

[【译】Swift 泛型宣言  kemchenj  关注](https://www.jianshu.com/p/81bcc2d409f5)

[Magical Error Handling in Swift](https://www.raywenderlich.com/1050-magical-error-handling-in-swift)

[Error Handling in Swift 2.0](https://apple-swift.readthedocs.io/en/latest/ErrorHandling.html)

[Swift中的Error](http://hongchaozhang.github.io/blog/2017/10/20/errors-in-swift/)

[swift 指针学习记录](https://www.jianshu.com/p/103591adc3d1)

[FATALERROR](https://swifter.tips/fatalerror/)

[谈一谈Go的异常处理机制——panic和recover的使用和原理](https://blog.csdn.net/qq_27682041/article/details/78786689)

[关于Go defer的详细使用](https://www.cnblogs.com/phpper/archive/2019/12/04/11984161.html)

[关于 Go 中的 panic 的使用以及误用](https://studygolang.com/articles/16267)

[c# - 捕获和重新抛出.NET异常的最佳实践](https://www.coder.work/article/225979)

[理解函数里的side effects](https://blog.csdn.net/ustcyy91/article/details/80374401)

[Size, Stride, Alignment](https://swiftunboxed.com/internals/size-stride-alignment/)

[Intel的instruction boundaries是什么](https://blog.csdn.net/leoufung/article/details/48828197)

[数据库访问抽象层系列](https://blog.csdn.net/bobkentblog/article/details/45055905)

[高阶函数](https://www.liaoxuefeng.com/wiki/897692888725344/923030148673312)

[Vtable内存布局分析](https://www.cnblogs.com/xhb19960928/archive/2004/01/13/11720314.html)

[JAVA多态理解（包含他人经典例子）](https://blog.csdn.net/qq_27008179/article/details/53004782)

[编译器Clang介绍](https://www.cnblogs.com/markhy/p/3169035.html)

[关于在高级语言中不建议使用goto语句的看法](https://blog.csdn.net/weixin_45797022/article/details/105449201?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param)

[Improving performance by better code locality.](https://easyperf.net/blog/2018/07/09/Improving-performance-by-better-code-locality)