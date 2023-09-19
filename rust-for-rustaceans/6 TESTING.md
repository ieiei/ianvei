

In this chapter, we’ll look at the various ways in which you can extend Rust’s testing capabilities and what other kinds of testing you may want to add into your testing mix. Rust comes with a number of built-in testing facilities that are well covered in *The Rust Programming Language*, represented primarily by the #[test] attribute and the *tests/* directory. These will serve you well across a wide range of applications and scales and are often all you need when you are getting started with a project. However, as the codebase develops and your testing needs grow more elaborate, you may need to go beyond just tagging #[test] onto individual functions. 

This chapter is divided into two main sections. The first part covers Rust testing mechanisms, like the standard testing harness and conditional testing code. The second looks at other ways to evaluate the correctness of your Rust code, such as benchmarking, linting, and fuzzing. 

**Rust Testing Mechanisms** 

To understand the various testing mechanisms Rust provides, you must first understand how Rust builds and runs tests. When you run cargo test --lib, the only special thing Cargo does is pass the --test flag to rustc. This flag tells rustc to produce a test binary that runs all the unit tests, rather than just compiling the crate’s library or binary. Behind the scenes, --test has two primary effects. First, it enables cfg(test) so that you can conditionally include testing code (more on that in a bit). Second, it makes the compiler generate a *test harness*: a carefully generated main function that invokes each #[test] function in your program when it’s run. 

**The Test Harness** 

The compiler generates the test harness main function through a mix of procedural macros, which we’ll discuss in greater depth in Chapter 7, and a light sprinkling of magic. Essentially, the harness transforms every func- tion annotated by #[test] into a test *descriptor*—this is the procedural macro part. It then exposes the path of each of the descriptors to the generated main function—this is the magic part. The descriptor includes information like the test’s name, any additional options it has set (like #[should_panic]), and so on. At its core, the test harness iterates over the tests in the crate, runs them, captures their results, and prints the results. So, it also includes logic to parse command line arguments (for things like --test-threads=1), capture test output, run the listed tests in parallel, and collect test results. 

As of this writing, Rust developers are working on making the magic part of test harness generation a publicly available API so that developers can build their own test harnesses. This work is still at the experimental stage, but the proposal aligns fairly closely with the model as it exists today. Part of the magic that needs to be figured out is how to ensure that #[test] functions are available to the generated main function even if they are inside private submodules. 

Integration tests (the tests in *tests/*) follow the same process as unit tests, with the one exception that they are each compiled as their own separate crate, meaning they can access only the main crate’s public inter- face and are run against the main crate compiled without #[cfg(test)]. A test harness is generated for each file in *tests/*. Test harnesses are not gen- erated for files in subdirectories under *tests/* to allow you to have shared submodules for your tests. 

**NOTE** *If you explicitly want a test harness for a file in a subdirectory, you can opt in to that by calling the file* main.rs*.* 

Rust does not require that you use the default test harness. You can instead opt out of it and implement your own main method that represents the test runner by setting harness = false for a given integration test in *Cargo.toml*, as shown in Listing 6-1. The main method that you define will then be invoked to run the test. 

```
[[test]]
name = "custom"
path = "tests/custom.rs"
harness = false
```

*Listing 6-1: Opting out of the standard test harness*

Without the test harness, none of the magic around #[test] happens. Instead, you’re expected to write your own main function to run the testing code you want to execute. Essentially, you’re writing a normal Rust binary that just happens to be run by cargo test. That binary is responsible for handling all the things that the default harness normally does (if you want to support them), such as command line flags. The harness property is set separately for each integration test, so you can have one test file that uses the standard harness and one that does not. 

**ARGUMENTS TO THE DEFAULT TEST HARNESS** 

The default test harness supports a number of command line arguments to configure how the tests are run . These aren’t passed to cargo test directly but rather to the test binary that Cargo compiles and runs for you when you run cargo test . To access that set of flags, pass -- to cargo test, followed by the arguments to the test binary . For example, to see the help text for the test binary, you’d run cargo test -- --help . 

A number of handy configuration options are available through these com- mand line arguments . The --nocapture flag disables the output capturing that normally happens when you run Rust tests . This is useful if you want to observe a test’s output in real time rather than all at once after the test has failed . You can use the --test-threads option to limit how many tests run concurrently, which is helpful if you have a test that hangs or segfaults and you want to figure out which one it is by running the tests sequentially . There’s also a --skip option for skipping tests that match a certain pattern, --ignored to run tests that would normally be ignored (such as those that require an external program to be run- ning), and --list to just list all the available tests . 

Keep in mind that these arguments are all implemented by the default test harness, so if you disable it (with harness = false), you’ll have to implement the ones you need yourself in your main function! 

Integration tests without a harness are primarily useful for bench- marks, as we’ll see later, but they also come in handy when you want to run tests that don’t fit the standard “one function, one test” model. For exam- ple, you’ll frequently see harnessless tests used with fuzzers, model check- ers, and tests that require a custom global setup (like under WebAssembly or when working with custom targets). 

**#[cfg(test)]** 

When Rust builds code for testing, it sets the compiler configuration flag test, which you can then use with conditional compilation to have code that is compiled out unless it is specifically being tested. On the surface, this may seem odd: don’t you want to test exactly the same code that’s going into production? You do, but having code exclusively available when testing allows you to write better, more thorough tests, in a few ways. 

**MOCKING** 

When writing tests, you often want tight control over the code you’re testing as well as any other types that your code may interact with . For example, if you are testing a network client, you probably do not want to run your unit tests over a real network but instead want to directly control what bytes are emitted by the “network” and when . Or, if you’re testing a data structure, you want your test to use types that allow you to control what each method returns on each invocation . You may also want to gather metrics such as how often a given method was called or whether a given byte sequence was emitted . 

These “fake” types and implementations are known as mocks, and they are a key feature of any extensive unit test suite . While you can often do the work needed to get this kind of control manually, it’s nicer to have a library take care of most of the nitty-gritty details for you . This is where automated mocking comes into play . A mocking library will have facilities for generating types (including functions) with particular properties or signatures, as well as mechanisms to control and introspect those generated items during a test execution . 

Mocking in Rust generally happens through generics—as long as your program, data structure, framework, or tool is generic over anything you might want to mock (or takes a trait object), you can use a mocking library to gener- ate conforming types that will instantiate those generic parameters . You then write your unit tests by instantiating your generic constructs with the generated mock types, and you’re off to the races! 

In situations where generics are inconvenient or inappropriate, such as if you want to avoid making a particular aspect of your type generic to users, you can instead encapsulate the state and behavior you want to mock in a dedicated struct . You would then generate a mocked version of that struct and its methods and use conditional compilation to use either the real or mocked implementation depending on cfg(test) or a test-only feature like cfg(feature = "test_mock_foo") . 

At the moment, there isn’t a single mocking library, or even a single mock- ing approach, that has emerged as the One True Answer in the Rust community . The most extensive and thorough mocking library I know of is the mockall crate, but that is still under active development, and there are many other contenders . 

**Test-Only APIs** 

First, having test-only code allows you to expose additional methods, fields, and types to your (unit) tests so the tests can check not only that the public API behaves correctly but also that the internal state is correct. For example, consider the HashMap type from hashbrown, the crate that implements the standard library HashMap. The HashMap type is really just a wrapper around a RawTable type, which is what implements most of the hash table logic. Suppose that after doing a HashMap::insert on an empty map, you want to check that a single bucket in the map is nonempty, as shown in Listing 6-2. 

```rust
#[test]
fn insert_just_one() {
  let mut m = HashMap::new();
  m.insert(42, ());
  let full = m.table.buckets.iter().filter(Bucket::is_full).count();
  assert_eq!(full, 1);
}
```

*Listing 6-2: A test that accesses inaccessible internal state and thus does not compile*

This code will not compile as written, because while the test code can access the private table field of HashMap, it cannot access the also private buckets field of RawTable, as RawTable lives in a different module. We could fix this by making the buckets field visibility pub(crate), but we really don’t want HashMap to be able to touch buckets in general, as it could accidentally corrupt the internal state of the RawTable. Even making buckets available as read-only could be problematic, as new code in HashMap may then start depending on the internal state of RawTable, making future modifications more difficult. 

The solution is to use #[cfg(test)]. We can add a method to RawTable that allows access to buckets only while testing, as shown in Listing 6-3, and thereby avoid adding footguns for the rest of the code. The code from Listing 6-2 can then be updated to call buckets() instead of accessing the private buckets field. 

```rust
impl RawTable {
  #[cfg(test)]
  pub(crate) fn buckets(&self) -> &[Bucket] {
    &self.buckets
  }
}
```

*Listing 6-3: Using *#[cfg(test)]* to make internal state accessible in the testing context **Bookkeeping for Test Assertions** *

The second benefit of having code that exists only during testing is that you can augment the program to perform additional runtime bookkeep- ing that can then be inspected by tests. For example, imagine you’re writ- ing your own version of the BufWriter type from the standard library. When testing it, you want to make sure that BufWriter does not issue system calls unnecessarily. The most obvious way to do so is to have the BufWriter keep track of how many times it has invoked write on the underlying Write. However, in production this information isn’t important, and keeping track of it introduces (marginal) performance and memory overhead. With #[cfg(test)], you can have the bookkeeping happen only when testing, as shown in Listing 6-4. 

```rust
struct BufWriter<T> {
  #[cfg(test)]
  write_through: usize,
  // other fields...
}
impl<T: Write> Write for BufWriter<T> {
  fn write(&mut self, buf: &[u8]) -> Result<usize> {
  		// ... 
      if self.full() {
      #[cfg(test)]
      self.write_through += 1;
      let n = self.inner.write(&self.buffer[..])?;
 // ... 
}}  
```

*Listing 6-4: Using *#[cfg(test)]* to limit bookkeeping to the testing context*

Keep in mind that test is set only for the crate that is being compiled as a test. For unit tests, this is the crate being tested, as you would expect. For integration tests, however, it is the integration test binary being compiled as a test—the crate you are testing is just compiled as a library and so will not have test set. 

**Doctests** 

Rust code snippets in documentation comments are automatically run as test cases. These are commonly referred to as *doctests*. Because doctests appear in the public documentation of your crate, and users are likely to mimic what they contain, they are run as integration tests. This means that the doctests don’t have access to private fields and methods, and test is not set on the main crate’s code. Each doctest is compiled as its own dedicated crate and is run in isolation, just as if the user had copy-pasted the doctest into their own program. 

Behind the scenes, the compiler performs some preprocessing on doctests to make them more concise. Most importantly, it automatically adds an fn main around your code. This allows doctests to focus only on the important bits that the user is likely to care about, like the parts that actually use types and methods from your library, without including unnecessary boilerplate. 

You can opt out of this auto-wrapping by defining your own fn main in the doctest. You may want to do this, for example, if you want to write an asynchronous main function using something like #[tokio::main] async fn main, or if you want to add additional modules to the doctest. 

To use the ? operator in your doctest, you don’t normally have to use a custom main function as rustdoc includes some heuristics to set the return type to Result<(), impl Debug> if your code looks like it makes use of ? (for example, if it ends with Ok(())). If type inference gives you a hard time about the error type for the function, you can disambiguate it by changing the last line of the doctest to be explicitly typed, like this: Ok::<(), T>(()). 

Doctests have a number of additional features that come in handy as you write documentation for more complex interfaces. The first is the ability to hide individual lines. If you prefix a line of a doctest with a #, that line is included when the doctest is compiled and run, but it is not included in the code snippet generated in the documentation. This lets you easily hide details that are not important to the current example, such as implement- ing traits for dummy types or generating values. It is also useful if you wish to present a sequence of examples without showing the same leading code each time. Listing 6-5 gives an example of what a doctest with hidden lines might look like. 

```
/// Completely frobnifies a number through I/O.
///
/// In this first example we hide the value generation.
/// ```
/// # let unfrobnified_number = 0;
/// # let already_frobnified = 1;
/// assert!(frobnify(unfrobnified_number).is_ok());
/// assert!(frobnify(already_frobnified).is_err());
/// ```
///
/// Here's an example that uses ? on multiple types
/// and thus needs to declare the concrete error type,
/// but we don't want to distract the user with that.
/// We also hide the use that brings the function into scope.
/// ```
/// # use mylib::frobnify;
/// frobnify("0".parse()?)?;
/// # Ok::<(), anyhow::Error>(())
/// ```
///
/// You could even replace an entire block of code completely,
/// though use this _very_ sparingly:
/// ```
/// # /*
/// let i = ...;
/// # */
/// # let i = 42;
/// frobnify(i)?;
/// ```
fn frobnify(i: usize) -> std::io::Result<()> {
```

*Listing 6-5: Hiding lines in a doctest with *#* *

**NOTE** *Use this feature with care; it can be frustrating to users if they copy-paste an example and then it doesn’t work because of required steps that you’ve hidden.* 

Much like #[test] functions, doctests also support attributes that modify how the doctest is run. These attributes go immediately after the triple-backtick used to denote a code block, and multiple attributes can be separated by commas. 

Like with test functions, you can specify the should_panic attribute to indicate that the code in a particular doctest should panic when run, or ignore to check the code segment only if cargo test is run with the --ignored flag. You can also use the no_run attribute to indicate that a given doctest should compile but should not be run. 

The attribute compile_fail tells rustdoc that the code in the documenta- tion example should not compile. This indicates to the user that a particu- lar use is not possible and serves as a useful test to remind you to update the documentation should the relevant aspect of your library change. You can also use this attribute to check that certain static properties hold for your types. Listing 6-6 shows an example of how you can use compile_fail to check that a given type does not implement Send, which may be necessary to uphold safety guarantees in unsafe code. 

```
```compile_fail
# struct MyNonSendType(std::rc::Rc<()>);
fn is_send<T: Send>() {}
is_send::<MyNonSendType>();
```
Listing 6-6: Testing that code fails to compile with *compile_fail* 

compile_fail is a fairly crude tool in that it gives no indication of *why* the code does not compile. For example, if code doesn’t compile because of a missing semicolon, a compile_fail test will appear to have been successful. For that reason, you’ll usually want to add the attribute only after you have made sure that the test indeed fails to compile with the expected error. If you need more fine-grained tests for compilation errors, such as when developing macros, take a look at the trybuild crate. 

**Additional Testing Tools** 

There’s a lot more to testing than just running test functions and seeing that they produce the expected result. A thorough survey of testing techniques, methodologies, and tools is outside the scope of this book, but there are some key Rust-specific pieces that you should know about as you expand your testing repertoire. 

**Linting** 

You may not consider a linter’s checks to be tests, but in Rust they often can be. The Rust linter *clippy* categorizes a number of its lints as *correctness* 

**N O T E** *lints*. These lints catch code patterns that compile but are almost cer- tainly bugs. Some examples are a = b; b = a, which fails to swap a and b; std::mem::forget(t), where t is a reference; and for x in y.next(), which will iterate only over the first element in y. If you are not running clippy as part of your CI pipeline already, you probably should be. 

Clippy comes with a number of other lints that, while usually helpful, may be more opinionated than you’d prefer. For example, the type_complexity lint, which is on by default, issues a warning if you use a particularly involved type in your program, like Rc<Vec<Vec<Box<(u32, u32, u32, u32)>>>>. While that warning encourages you to write code that is easier to read, you may find it too pedantic to be broadly useful. If some part of your code erroneously triggers a particular lint, or you just want to allow a specific instance of it, you can opt out of the lint just for that piece of code with #[allow(clippy::name_of_lint)]. 

The Rust compiler also comes with its own set of lints in the form of warnings, though these are usually more directed toward writing idiomatic code than checking for correctness. Instead, correctness lints in the com- piler are simply treated as errors (take a look at rustc -W help for a list). 

*Not all compiler warnings are enabled by default. Those disabled by default are usu- ally still being refined, or are more about style than content. A good example of this is the “idiomatic Rust 2018 edition” lint, which you can enable with* *#![warn(rust_2018 _idioms)]**. When this lint is enabled, the compiler will tell you if you’re failing to take advantage of changes brought by the Rust 2018 edition. Some other lints that you may want to get into the habit of enabling when you start a new project are* *missing_docs* *and* *missing_debug_implementations**, which warn you if you’ve forgotten to document any public items in your crate or add* *Debug* *implementations for any public types, respectively.* 

**Test Generation** 

Writing a good test suite is a lot of work. And even when you do that work, the tests you write test only the particular set of behaviors you were con- sidering at the time you wrote them. Luckily, you can take advantage of a number of test generation techniques to develop better and more thor- ough tests. These generate input for you to use to check your application’s correctness. Many such tools exist, each with their own strengths and weaknesses, so here I’ll cover only the main strategies used by these tools: fuzzing and property testing. 

**Fuzzing** 

Entire books have been written about fuzzing, but at a high level the idea is simple: generate random inputs to your program and see if it crashes. If the program crashes, that’s a bug. For example, if you’re writing a URL parsing library, you can fuzz-test your program by systematically generating random strings and throwing them at the parsing function until it panics. Done naively, this would take a while to yield results: if the fuzzer starts with a, then b, then c, and so on, it will take it a long time to generate a tricky URL like http://[:]. In practice, modern fuzzers use code coverage metrics to explore different paths in your code, which lets them reach higher degrees of coverage faster than if the inputs were truly chosen at random. 

Fuzzers are great at finding strange corner cases that your code doesn’t handle correctly. They require little setup on your part: you just point the fuzzer at a function that takes a “fuzzable” input, and off it goes. For exam- ple, Listing 6-7 shows an example of how you might fuzz-test a URL parser. 

```rust
libfuzzer_sys::fuzz_target!(|data: &[u8]| {
  if let Ok(s) = std::str::from_utf8(data) {
      let _ = url::Url::parse(s);
  }
}); 
```

Listing 6-7: Fuzzing a URL parser with *libfuzzer*

The fuzzer will generate semi-random inputs to the closure, and any that form valid UTF-8 strings will be passed to the parser. Notice that the code here doesn’t check whether the parsing succeeds or fails—instead, it’s looking for cases where the parser panics or otherwise crashes due to inter- nal invariants that are violated. 

The fuzzer keeps running until you terminate it, so most fuzzing tools come with a built-in mechanism to stop after a certain number of test cases have been explored. If your input isn’t a trivially fuzzable type—something like a hash table—you can usually use a crate like arbitrary to turn the byte string that the fuzzer generates into a more complex Rust type. It feels like magic, but under the hood it’s actually implemented in a very straightfor- ward fashion. The crate defines an Arbitrary trait with a single method, arbitrary, that constructs the implementing type from a source of random bytes. Primitive types like u32 or bool read the necessary number of bytes from that input to construct a valid instance of themselves, whereas more complex types like HashMap or BTreeSet produce one number from the input to dictate their length and then call Arbitrary that number of times on their inner types. There’s even an attribute, #[derive(Arbitrary)], that implements Arbitrary by just calling arbitrary on each contained type! To explore fuzz- ing further, I recommend starting with cargo-fuzz. 

**Property-Based Testing** 

Sometimes you want to check not only that your program doesn’t crash but also that it does what it’s expected to do. It’s great that your add function didn’t panic, but if it tells you that the result of add(1, 4) is 68, it’s probably still wrong. This is where *property-based testing* comes into play; you describe a number of properties your code should uphold, and then the property testing framework generates inputs and checks that those properties indeed hold. 

A common way to use property-based testing is to first write a simple but naive version of the code you want to test that you are confident is cor- rect. Then, for a given input, you give that input to both the code you want to test and the simplified but naive version. If the result or output of the two implementations is the same, your code is good—that is the correct- ness property you’re looking for—but if it’s not, you’ve likely found a bug. You can also use property-based testing to check for properties not directly related to correctness, such as whether operations take strictly less time for one implementation than another. The common principle is that you want any difference in outcome between the real and test versions to be informa- tive and actionable so that every failure allows you to make improvements. The naive implementation might be one from the standard library that you’re trying to replace or augment (like std::collections::VecDeque), or it might be a simpler version of an algorithm that you’re trying optimize (like naive versus optimized matrix multiplication). 

If this approach of generating inputs until some condition is met sounds a lot like fuzzing, that’s because it is—smarter people than I have argued that fuzzing is “just” property-based testing where the property you’re testing for is “it doesn’t crash.” 

One downside of property-based testing is that it relies more heavily on the provided descriptions of the inputs. Whereas fuzzing will keep trying all possible inputs, property testing tends to be guided by developer anno- tations like “a number between 0 and 64” or “a string that contains three commas.” This allows property testing to more quickly reach cases that fuzz- ers may take a long time to encounter randomly, but it does require manual work and may miss important but niche buggy inputs. As fuzzers and prop- erty testers grow closer, however, fuzzers are starting to gain this kind of constraint-based searching capability as well. 

If you’re curious about property-based test generation, I recommend starting with the proptest crate. 

**TESTING SEQUENCES OF OPERATIONS** 

Since fuzzers and property testers allow you to generate arbitrary Rust types, you aren’t limited to testing a single function call in your crate . For example, say you want to test that some type Foo behaves correctly if you perform a particu- lar sequence of operations on it . You could define an enum Operation that lists operations, and make your test function take a Vec<Operation> . Then you could instantiate a Foo and perform each operation on that Foo, one after the other . Most testers have support for minimizing inputs, so they will even search for the smallest sequence of operations that still violates a property if a property- violating input is found! 

**Test Augmentation** 

Let’s say you have a magnificent test suite all set up, and your code passes all the tests. It’s glorious. But then, one day, one of the normally reliable tests inexplicably fails or crashes with a segmentation fault. There are two common reasons for these kinds of nondeterministic test failures: race conditions, where your test might fail only if two operations occur on different threads in a particular order, and undefined behavior in unsafe code, such as if some unsafe code reads a particular value out of uninitialized memory. 

Catching these kinds of bugs with normal tests can be difficult—often you don’t have sufficient low-level control over thread scheduling, memory layout and content, or other random-ish system factors to write a reliable test. You could run each test many times in a loop, but even that may not catch the error if the bad case is sufficiently rare or unlikely. Luckily, there are tools that can help augment your tests to make catching these kinds of bugs much easier. 

The first of these is the amazing tool *Miri*, an interpreter for Rust’s *mid-level intermediate representation (MIR)*. MIR is an internal, simplified representation of Rust that helps the compiler find optimizations and check properties without having to consider all of the syntax sugar of Rust itself. Running your tests through Miri is as simple as running cargo miri test. Miri *interprets* your code rather than compiling and running it like a normal binary, which makes the tests run a decent amount slower. But in return, Miri can keep track of the entire program state as each line of your code executes. This allows Miri to detect and report if your program ever exhibits certain types of undefined behavior, such as uninitialized memory reads, uses of values after they’ve been dropped, or out-of-bounds pointer accesses. Rather than having these operations yield strange program behaviors that may only sometimes result in observable test failures (like crashes), Miri detects them when they happen and tells you immediately. 

For example, consider the very unsound code in Listing 6-8, which creates two exclusive references to a value. 

```rust
let mut x = 42;
let x: *mut i32 = &mut x;
let (x1, x2) = unsafe { (&mut *x, &mut *x) };
println!("{} {}", x1, x2);
```

Listing 6-8: Wildly unsafe code that Miri detects is incorrect 

At the time of writing, if you run this code through Miri, you get an error that points out exactly what’s wrong: error: Undefined Behavior: trying to reborrow for Unique at alloc1383, but parent tag <2772> does not have an appropriate item in the borrow stack
```
 --> src/main.rs:4:6
  |
4 | let (x1, x2) = unsafe { (&mut *x, &mut *x) };
  |      ^^ trying to reborrow for Unique at alloc1383, but parent tag <2772>
does not have an appropriate item in the borrow stack
```

*Miri is still under development, and its error messages aren’t always the easiest to understand. This is a problem that’s being actively worked on, so by the time you read this, the error output may have already gotten much better!* 

Another tool worth looking at is *Loom*, a clever library that tries to ensure your tests are run with every relevant interleaving of concurrent operations. At a high level, Loom keeps track of all cross-thread synchronization points and runs your tests over and over, adjusting the order in which threads proceed from those synchronization points each time. So, if thread A and thread B both take the same Mutex, Loom will ensure that the test runs once with A taking it first and once with B taking it first. Loom also keeps track of atomic accesses, memory orderings, and accesses to UnsafeCell (which we’ll discuss in Chapter 9) and checks that threads do not access them inappropriately. If a test fails, Loom can give you an exact run- down of which threads executed in what order so you can determine how the crash happened. 

**Performance Testing** 

Writing performance tests is difficult because it is often hard to accurately model a workload that reflects real-world usage of your crate. But having such tests is important; if your code suddenly runs 100 times slower, that really should be considered a bug, yet without a performance test you may not spot the regression. If your code runs 100 times *faster*, that might also indicate that something is off. Both of these are good reasons to have auto- mated performance tests as part of your CI—if performance changes drasti- cally in either direction, you should know about it. 

Unlike with functional testing, performance tests do not have a common, well-defined output. A functional test will either succeed or fail, whereas a performance test may give you a throughput number, a latency profile, a number of processed samples, or any other metric that might be relevant to the application in question. Also, a performance test may require running a function in a loop a few hundred thousand times, or it might take hours running across a distributed network of multicore boxes. For that reason, it is difficult to speak about how to write performance tests in a general sense. Instead, in this section, we’ll look at some of the issues you may encounter when writing performance tests in Rust and how to mitigate them. Three particularly common pitfalls that are often overlooked are performance variance, compiler optimizations, and I/O overhead. Let’s explore each of these in turn. 

**Performance Variance** 

Performance can vary for a huge variety of reasons, and many factors affect how fast a particular sequence of machine instructions run. Some are obvious, like the CPU and memory clock speed, or how loaded the machine otherwise is, but many are much more subtle. For example, your kernel version may change paging performance, the length of your username might change the 

Testing **97** 

layout of memory, and the temperature in the room might cause the CPU to clock down. Ultimately, it is highly unlikely that if you run a benchmark twice, you’ll get the same result. In fact, you may observe significant variance, even if you are using the same hardware. Or, viewed from another perspective, your code may have gotten slower or faster, but the effect may be invisible due to differences in the benchmarking environment. 

There are no perfect ways to eliminate all variance in your performance results, unless you happen to be able to run benchmarks repeatedly on a highly diverse fleet of machines. Even so, it’s important to try to handle this measurement variance as best we can to extract a signal from the noisy measurements benchmarks give us. In practice, our best friend in combating variance is to run each benchmark many times and then look at the *distribution* of measurements rather than just a single one. Rust has tools that can help with this. For example, rather than ask “How long did this function take to run on average?” crates like hdrhistogram enable us to look at statistics like “What range of runtime covers 95% of the samples we observed?” To be even more rigorous, we can use techniques like null hypothesis testing from statistics to build some confidence that a measured difference indeed corresponds to a true change and is not just noise. 

A lecture on statistical hypothesis testing is beyond the scope of this book, but luckily much of this work has already been done by others. The criterion crate, for instance, does all of this and more for you. All you have to do is give it a function that it can call to run one iteration of your benchmark, and it will run it the appropriate number of times to be fairly sure that the result is reliable. It then produces a benchmark report, which includes a summary of the results, analysis of outliers, and even graphical representations of trends over time. Of course, it can’t eliminate the effects of just testing on a particular configuration of hardware, but it at least categorizes the noise that is measurable across executions. 

**Compiler Optimizations** 

Compilers these days are really clever. They eliminate dead code, compute complex expressions at compile time, unroll loops, and perform other dark magic to squeeze every drop of performance out of our code. Normally this is great, but when we’re trying to measure how fast a particular piece of code is, the compiler’s smartness can give us invalid results. For example, take the code to benchmark Vec::push in Listing 6-9. 

```
let mut vs = Vec::with_capacity(4);
let start = std::time::Instant::now();
for i in 0..4 {
  vs.push(i);
}
println!("took {:?}", start.elapsed());
```

Listing 6-9: A suspiciously fast performance benchmark 

**98** Chapter 6 

If you were to look at the assembly output of this code compiled in release mode using something like the excellent *godbolt.org* or cargo-asm, you’d immediately notice that something was wrong: the calls to Vec::with _capacity and Vec::push, and indeed the whole for loop, are nowhere to be seen. They have been optimized out completely. The compiler realized that nothing in the code actually required the vector operations to be per- formed and eliminated them as dead code. Of course, the compiler is com- pletely within its rights to do so, but for benchmarking purposes, this is not particularly helpful. 

To avoid these kinds of optimizations for benchmarking, the standard library provides std::hint::black_box. This function has been the topic of much debate and confusion and is still pending stabilization at the time of writing, but is so useful it’s worth discussing here nonetheless. At its core, it’s simply an identity function (one that takes x and returns x) that tells the compiler to assume that the argument to the function is used in arbitrary (legal) ways. It does not prevent the compiler from applying optimizations to the input argument, nor does it prevent the compiler from optimizing how the return value is used. Instead, it encourages the compiler to actually compute the argument to the function (under the assumption that it will be used) and to store that result somewhere accessible to the CPU such that black_box could be called with the computed value. The compiler is free to, say, compute the input argument at compile time, but it should still inject the result into the program. 

This function is all we need for many, though admittedly not all, of our benchmarking needs. For example, we can annotate Listing 6-9 so that the vector accesses are no longer optimized out, as shown in Listing 6-10. 

```
let mut vs = Vec::with_capacity(4);
let start = std::time::Instant::now();
for i in 0..4 {
  black_box(vs.as_ptr());
  vs.push(i);
  black_box(vs.as_ptr());
}
println!("took {:?}", start.elapsed());
```

Listing 6-10: A corrected version of Listing 6-9 

We’ve told the compiler to assume that vs is used in arbitrary ways
 on each iteration of the loop, both before and after the calls to push. This forces the compiler to perform each push in order, without merging or oth- erwise optimizing consecutive calls, since it has to assume that “arbitrary stuff that cannot be optimized out” (that’s the black_box part) may happen to vs between each call. 

Note that we used vs.as_ptr() and not, say, &vs. That’s because of the caveat that the compiler should assume black_box can perform any *legal* opera- tion on its argument. It is not legal to mutate the Vec through a shared refer- ence, so if we used black_box(&vs), the compiler might notice that vs will not change between iterations of the loop and implement optimizations based on that observation! 

Testing **99** 

**I/O Overhead Measurement** 

When writing benchmarks, it’s easy to accidentally measure the wrong thing. For example, we often want to get information in real time about how far along the benchmark is. To do that, we might write code like that in Listing 6-11, intended to measure how fast my_function runs: 

```
             let start = std::time::Instant::now();
             for i in 0..1_000_000 {
               println!("iteration {}", i);
               my_function();
             }
             println!("took {:?}", start.elapsed());


Listing 6-11: What are we really benchmarking here? 

This may look like it achieves the goal, but in reality, it does not actually measure how fast my_function is. Instead, this loop is most likely to tell us how long it takes to print a million numbers. The println! in the body of the loop does a lot of work behind the scenes: it turns a binary integer into decimal digits for printing, locks standard output, writes out a sequence of UTF-8 code points using at least one system call, and then releases the standard output lock. Not only that, but the system call might block if your terminal is slow to print out the input it receives. That’s a lot of cycles! And the time it takes to call my_function might pale in comparison. 

A similar thing happens when your benchmark uses random numbers. If you run my_function(rand::random()) in a loop, you may well be mostly measuring the time it takes to generate a million random numbers. The story is the same for getting the current time, reading a configuration file, or start- ing a new thread—these things all take a long time, relatively speaking, and may end up overshadowing the time you actually wanted to measure. 

Luckily, this particular issue is often easy to work around once you are aware of it. Make sure that the body of your benchmarking loop contains almost nothing but the particular code you want to measure. All other code should run either before the benchmark begins or outside of the measured part of the benchmark. If you’re using criterion, take a look at the different timing loops it provides—they’re all there to cater to benchmarking cases that require different measurement strategies! 

## Summary

In this chapter, we explored the built-in testing capabilities that Rust offers in great detail. We also looked at a number of testing facilities and techniques that are useful when testing Rust code. This is the last chapter that focuses on higher-level aspects of intermediate Rust use in this book. Starting with the next chapter on declarative and procedural macros, we will be focusing much more on Rust code. See you on the next page! 