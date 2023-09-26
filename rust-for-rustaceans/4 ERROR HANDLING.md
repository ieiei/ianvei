对于除了最简单的程序之外的所有程序，您都会有可能失败的方法。在本章中，我们将研究表示、处理和传播这些故障的不同方法以及每种方法的优缺点。我们将首先探索表示错误的不同方法，包括枚举和擦除，然后检查一些需要不同表示技术的特殊错误情况。接下来，我们将了解处理错误的各种方法以及错误处理的未来。

值得注意的是，Rust 中错误处理的最佳实践仍然是一个活跃的话题，在撰写本文时，生态系统尚未确定一个单一的、统一的方法。因此，本章将重点关注基本原理和技术，而不是推荐特定的包或模式。

**Representing Errors** 

当您编写可能失败的代码时，要问自己的最重要的问题是您的用户将如何与返回的任何错误进行交互。用户是否需要确切地知道发生了哪个错误以及发生了什么问题的细节，或者他们只是简单地记录发生了错误并尽可能地继续前进？为了理解这一点，我们必须看看错误的性质是否可能影响调用者收到错误后执行的操作。这反过来将决定我们如何表示不同的错误。

您有两个主要选项来表示错误：枚举和擦除。也就是说，您可以让错误类型*枚举*可能的错误条件，以便调用者可以区分它们，或者您可以只向调用者提供单个*不透明*错误。让我们依次讨论这两个选项。

**Enumeration** 

对于我们的示例，我们将使用一个库函数将字节从某个输入流复制到某个输出流中，就像 std::io::copy 一样。用户为您提供两个流，一个用于读取，一个用于写入，然后您将字节从一个流复制到另一个流。在此过程中，任何一个流都完全有可能失败，此时复制必须停止并向用户返回错误。在这里，用户可能想知道失败的是输入流还是输出流。例如，在 Web 服务器中，如果在将文件流式传输到客户端时输入流发生错误，则可能是因为磁盘被弹出，而如果输出流发生错误，则可能是客户端刚刚断开连接。后者可能是服务器应该忽略的错误，因为到新连接的复制仍然可以完成，而前者可能需要关闭整个服务器！

在这种情况下，我们想要枚举错误。用户需要能够区分不同的错误情况，以便能够做出适当的响应，因此我们使用一个名为 CopyError 的枚举，每个变体代表错误的一个单独的根本原因，如清单 4-1 所示。

```
 pub enum CopyError {
   In(std::io::Error),
   Out(std::io::Error),
 }
```

*Listing 4-1: An enumerated error type*

每个变体还包括遇到的错误，以便为调用者提供尽可能多的有关出错的信息。

在创建自己的错误类型时，您需要采取许多步骤来使错误类型与 Rust 生态系统的其他部分很好地配合。首先，您的错误类型应该实现 std::error::Error 特征，它为调用者提供用于内省错误类型的通用方法。我们感兴趣的主要方法是 Error::source，它提供了一种查找错误根本原因的机制。这最常用于打印回溯，该回溯显示一直追溯到错误根本原因的跟踪。对于我们的 CopyError 类型，source 的实现很简单：我们匹配 self 并提取并返回内部 std::io::Error。

其次，您的类型应该同时实现“显示”和“调试”，以便调用者可以有意义地打印您的错误。如果您实现 Error 特征，则这是必需的。一般来说，您的 Display 实现应该对出现的问题给出一行描述，并且可以轻松地将其折叠到其他错误消息中。显示格式应该是小写并且没有尾随标点符号，以便它可以很好地适应其他较大的错误报告。调试应该提供更具描述性的错误，包括可能有助于追踪错误原因的辅助信息，例如端口号、请求标识符、文件路径等，#[derive(Debug)] 通常就足够了。

**NOTE** * In older Rust code, you may see references to the* *Error::description* *method, but this has been deprecated in favor of* *Display**.* 

第三，如果可能的话，您的类型应该同时实现发送和同步，以便用户能够跨线程边界共享错误。如果您的错误类型不是线程安全的，您会发现几乎不可能在多线程上下文中使用您的包。实现 Send 和 Sync 的错误类型也更容易与非常常见的 std::io::Error 类型一起使用，该类型能够包装实现 Error、Send 和 Sync 的错误。当然，并非所有错误类型都可以合理地进行发送和同步，例如如果它们与特定的线程本地资源相关联，那没关系。您可能也不会跨线程边界发送这些错误。但是，在将 Rc<String> 和 RefCell<bool> 类型放入错误中之前需要注意这一点。

最后，在可能的情况下，您的错误类型应该是“静态”。这样做最直接的好处是，它允许调用者更轻松地在调用堆栈中传播错误，而不会遇到生命周期问题。它还使您的错误类型能够更轻松地与类型擦除的错误类型一起使用，我们很快就会看到。

**Opaque Errors** 

现在让我们考虑一个不同的例子：图像解码库。您为库提供了一堆字节来解码，它使您可以访问各种图像操作方法。如果解码失败，用户需要能够找出解决问题的方法，因此必须了解原因。但是，到底是图像头中的大小字段无效，还是压缩算法无法解压缩块，这重要吗？可能不会——即使应用程序知道确切的原因，也无法从这两种情况中有意义地恢复。在这种情况下，作为库作者的您可能希望提供单一的、不透明的错误类型。这也使您的库更易于使用，因为到处都只使用一种错误类型。此错误类型应实现 Send、Debug、Display 和 Error（包括适当的源方法），但除此之外，调用者不需要了解更多信息。您可以在内部表示更细粒度的错误状态，但无需将这些状态公开给库的用户。这样做只会不必要地增加 API 的大小和复杂性。

您的不透明错误类型到底应该是什么主要取决于您。它可能只是一个具有所有私有字段的类型，仅公开用于显示和自省错误的有限方法，或者它可能是一个严重类型擦除的错误类型，如 Box<dyn Error + Send + Sync + 'static>，它不显示任何内容更重要的是，它是一个错误，并且通常根本不会让您的用户进行内省。决定错误类型的不透明程度主要取决于错误是否有超出其描述的有趣内容。使用 Box<dyn Error>，您的用户除了冒泡错误之外别无选择。如果它确实没有任何有价值的信息可以呈现给用户，那么这可能没问题——例如，如果它只是一条动态错误消息，或者是来自程序内部的大量不相关错误之一。但是，如果错误有一些有趣的方面，例如行号或状态代码，您可能希望通过具体但不透明的类型来公开它。

**N O T E***总的来说，社区的共识是错误应该很少，因此不应该给“幸福之路”增加太多成本。因此，错误通常位于指针类型后面，例如 * *Box* * 或 * *Arc**。这样，它们就不太可能增加太多的总体* *结果* *它们所包含的类型的大小。*

使用类型擦除错误的好处之一是，它允许您轻松组合来自不同来源的错误，而无需引入其他错误类型。也就是说，类型擦除的错误通常可以很好地“组合”，并允许您表达一组开放式错误。如果您编写一个返回类型为 Box<dyn Error + ...> 的函数，那么您可以使用 ?跨该函数内的不同错误类型，针对各种不同的错误，它们都将转换为一种常见的错误类型。

The 'static bound on Box<dyn Error + Send + Sync + 'static> 在删除方面值得花更多时间。我在上一节中提到，它对于让调用者传播错误而不用担心失败方法的生命周期界限很有用，但它有一个更大的目的：访问向下转型。 *向下转型*是将一种类型的项目转换为更具体类型的过程。这是 Rust 允许您在运行时访问类型信息的少数情况之一；这是动态语言通常提供的更通用类型反射的有限情况。在错误的上下文中，当 dyn 错误最初属于该类型时，向下转换允许用户将 dyn 错误转换为具体的底层错误类型。例如，如果用户收到的错误是 std::io::Error 类型的 std::io::ErrorKind ::WouldBlock，则用户可能希望采取特定操作，但他们不会在任何其他情况下采取相同的操作案件。如果用户收到 dyn 错误，他们可以使用 Error::downcast_ref 尝试将错误向下转换为 std::io::Error。 downcast_ref 方法返回一个 Option，它告诉用户向下转换是否成功。这是关键的观察： downcast_ref 仅当参数为“静态”时才有效。如果我们返回一个非“静态”的不透明错误，我们就会剥夺用户进行此类错误自省的能力（如果他们愿意的话）。

对于库的类型擦除错误（或者更一般地说，其类型擦除类型）是否是其公共且稳定的 API 的一部分，生态系统中存在一些分歧。也就是说，如果库中的方法 foo 将 lib::MyError 作为 Box<dyn Error> 返回，那么更改 foo 以返回不同的错误类型是否会是重大更改？类型签名没有改变，但用户可能编写了代码，假设他们可以使用向下转型将该错误转回 lib::MyError。我对此事的看法是，您选择返回 Box<dyn Error> （而不是 lib::MyError）是有原因的，除非明确记录，否则不能保证有关向下转换的任何特定内容。

*虽然* *Box* *是一种有吸引力的类型擦除错误类型，但与直觉相反，它本身并不实现* *Error**。因此，请考虑添加您自己的* *BoxError* *类型，以便在*实现* *Error**的库中进行类型擦除。*

您可能想知道 Error::downcast_ref 如何是安全的。也就是说，它如何知道提供的 dyn Error 参数是否确实属于给定类型 T？标准库甚至有一个名为 Any 的特征，它是为 *any* 类型实现的，并且为 dyn Any 实现了 downcast_ref ——这怎么可能呢？答案在于编译器支持的类型 std::any::TypeId，它允许您获取任何类型的唯一标识符。 Error 特征有一个隐藏的提供方法，称为 type_id，其默认实现是返回 TypeId::of::<Self>()。类似地，Any 有一个针对 T 的 impl Any 的全面实现，并且在该实现中，其 type_id 返回相同的值。在这些 impl 块的上下文中，Self 的具体类型是已知的，因此这个 type_id 是真实类型的类型标识符。这提供了 downcast_ref 所需的所有信息。 downcast_ref 调用 self.type_id，它通过动态大小类型的 vtable（参见第 2 章）转发到底层类型的实现，并将其与提供的向下转型类型的类型标识符进行比较。如果它们匹配，则 dyn Error 或 dyn Any 背后的类型实际上是 T，并且从对一个的引用转换为对另一个的引用是安全的。

**Special Error Cases** 

Some functions are fallible but cannot return any meaningful error if they fail. Conceptually, these functions have a return type of Result<T, ()>. In some codebases, you may see this represented as Option<T> instead. While both are legitimate choices for the return type for such a function, they convey different semantic meanings, and you should usually avoid “simplifying” a Result<T, ()> to Option<T>. An Err(()) indicates that an operation failed and should be retried, reported, or otherwise handled exceptionally. None, on the other hand, conveys only that the function has nothing to return; it is usually not considered an exceptional case or something that should be handled. You can see this in the #[must_use] annotation on the Result type—when you get a Result, the language expects that it is important to handle both cases, whereas with an Option, neither case actually needs to be handled. 

**NOTE** *You should also keep in mind that* *()* *does not implement the* *Error* *trait. This means that it cannot be type-erased into* *Box* *and can be a bit of a pain to use with* *?**. For this reason, it is often better to define your own unit struct type, implement* *Error* *for it, and use that as the error instead of* *()* *in these cases.* 

Some functions, like those that start a continuously running server loop, only ever return errors; unless an error occurs, they run forever. Other functions never error but need to return a Result nonetheless, for example, to match a trait signature. For functions like these, Rust provides the *never type*, written with the ! syntax. The never type represents a value that can never be generated. You cannot construct an instance of this type yourself— the only way to make one is by entering an infinite loop or panicking, or through a handful of other special operations that the compiler knows never return. With Result, when you have an Ok or Err that you know will never be used, you can set it to the ! type. If you write a function that returns Result<T, !>, you will be unable to ever return Err, since the only way to do so is to enter code that will never return. Because the compiler knows that any variant with a ! will never be produced, it can also optimize your code with that in mind, such as by not generating the panic code for an unwrap on Result<T, !>. And when you pattern match, the compiler knows that any variant that contains a ! does not even need to be listed. Pretty neat! 

One last curious error case is the error type std::thread::Result. Here’s its definition: 

``` rust
type Result<T> = Result<T, Box<dyn Any + Send + 'static>>;
```

The error type is type-erased, but it’s not erased into a dyn Error as we’ve seen so far. Instead, it is a dyn Any, which guarantees only that the error is *some* type, and nothing more . . . which is not much of a guarantee at all. The reason for this curious-looking error type is that the error variant of std::thread::Result is produced only in response to a panic; specifically, if you try to join a thread that has panicked. In that case, it’s not clear that there’s much the joining thread can do other than either ignore the error or panic itself using unwrap. In essence, the error type is “a panic” and the value is “whatever argument was passed to panic!,” which can truly be any type (even though it’s usually a formatted string). 

Rust’s ? operator acts as a shorthand for *unwrap or return early*, for working easily with errors. But it also has a few other tricks up its sleeve that are worth knowing about. First, ? performs type conversion through the From trait. In a function that returns Result<T, E>, you can use ? on any Result<T, X> where E: From<X>. This is the feature that makes error erasure through Box<dyn Error> so appealing; you can just use ? everywhere and not worry about the particular error type, and it will usually “just work.” 

**FROM AND INTO** 

The standard library has many conversion traits, but two of the core ones are From and Into . It might strike you as odd to have two: if we have From, why do we need Into, and vice versa? There are a couple of reasons, but let’s start with the historical one: it wouldn’t have been possible to have just one in the early days of Rust due to the coherence rules discussed in Chapter 2 . Or, more specifically, what the coherence rules used to be . 

Suppose you want to implement two-way conversion between some local type you have defined in your crate and some type in the standard library . You can write impl<T> From<Vec<T>> for MyType<T> and impl<T> Into<Vec<T>> for MyType<T> easily enough, but if you only had From or Into, you would have to write impl<T> From<MyType<T>> for Vec<T> or impl<T> Into<MyType<T>> for Vec<T> . However, the compiler used to reject those implementations! Only since Rust 1.41.0, when the exception for covered types was added to the coherence rules, are they legal . Before that change, it was necessary to have both traits . And since much Rust code was written before Rust 1.41.0, neither trait can be removed now . 

Beyond that historical fact, however, there are also good ergonomic reasons to have both of these traits, even if we could start from scratch today . It is often significantly easier to use one or the other in different situations . For example, if you’re writing a method that takes a type that can be turned into a Foo, would you rather write fn(impl Into<Foo>) or fn<T>(T) where Foo: From<T>? And conversely, to turn a string into a syntax identifier, would you rather write Ident::from("foo") or <_ as Into<Ident>>::into("foo")? Both of these traits have their uses, and we’re better off having them both. 

Given that we do have both, you may wonder which you should use in your code today . The answer, it turns out, is pretty simple: implement From, and use Into in bounds . The reason is that Into has a blanket implementation for any T that implements From, so regardless of whether a type explicitly implements From or Into, it implements Into! 

Of course, as simple things frequently go, the story doesn’t quite end there . Since the compiler often has to “go through” the blanket implementation when Into is used as a bound, the reasoning for whether a type implements Into is more complicated than whether it implements From . And in some cases, the compiler is not quite smart enough to figure that puzzle out . For this reason, the ? operator at the time of writing uses From, not Into . Most of the time that doesn’t make a difference, because most types implement From, but it does mean that error types from old libraries that implement Into instead may not work with ? . As the compiler gets smarter, ? will likely be “upgraded” to use Into, at which point that problem will go away, but it's what we have for now . 

Error Handling **63** 

The second aspect of ? to be aware of is that this operator is really just syntax sugar for a trait tentatively called Try. At the time of writing, the Try trait has not yet been stabilized, but by the time you read this, it’s likely that it, or something very similar, will have been settled on. Since the details haven’t all been figured out yet, I’ll give you only an outline of how Try works, rather than the full method signatures. At its heart, Try defines a wrapper type whose state is either one where further computation is useful (the happy path), or one where it is not. Some of you will correctly think of monads, though we won’t explore that connection here. For example, in the case of Result<T, E>, if you have an Ok(t), you can continue on the happy path by unwrapping the t. If you have an Err(e), on the other hand, you want to stop executing and produce the error value immediately, since further computation is not possible as you don’t have the t. 

What’s interesting about Try is that it applies to more types than just Result. An Option<T>, for example, follows the same pattern—if you have a Some(t), you can continue on the happy path, whereas if you have a None, you want to yield None instead of continuing. This pattern extends to more complex types, like Poll<Result<T, E>>, whose happy path type is Poll<T>, which makes ? apply in far more cases than you might expect. When Try stabilizes, we may see ? start to work with all sorts of types to make our happy path code nicer. 

The ? operator is already usable in fallible functions, in doctests, and in fn main. To reach its full potential, though, we also need a way to scope this error handling. For example, consider the function in Listing 4-2. 

```
fn do_the_thing() -> Result<(), Error> {
  let thing = Thing::setup()?;
  // .. code that uses thing and ? ..
  thing.cleanup();
  Ok(()) 
} 
```

*Listing 4-2: A multi-step fallible function using the *?* operator*

这不会完全按预期工作。任何 ？设置和清理之间将导致整个函数提前返回，这将跳过清理代码！这就是 *try 块* 旨在解决的问题。 try 块的行为非常类似于单次迭代循环，其中 ?使用break而不是return，并且块的最终表达式有一个隐式的break。现在，我们可以修复清单 4-2 中的代码，使其始终进行清理，如清单 4-3 所示。

```rust
fn do_the_thing() -> Result<(), Error> {
  let thing = Thing::setup()?;
  let r = try {
    // .. code that uses thing and ? ..
  };
  thing.cleanup();
  r
 }
```

*Listing 4-3: A multi-step fallible function that always cleans up after itself* 


Try blocks are also not stable at the time of writing, but there is enough of a consensus on their usefulness that they’re likely to land in a form similar to that described here. 



## Summary

This chapter covered the two primary ways to construct error types in Rust: enumeration and erasure. We looked at when you may want to use each one and the advantages and drawbacks of each. We also took a look at some of the behind-the-scenes aspects of the ? operator and considered how ? may become even more useful going forward. In the next chapter, we’ll take a step back from the code and look at how you *structure* a Rust project. We’ll look at feature flags, dependency management, and versioning as well as how to manage more complex crates using workspaces and subcrates. See you on the next page! 
