
Programming rarely happens in a vacuum these days—nearly every Rust crate you build is likely to take dependencies on *some* code that wasn’t written by you. Whether this trend is good, bad, or a little of both is a subject of heavy debate, but either way, it’s a reality of today’s developer experience. 

In this brave new interdependent world, it’s more important than ever to have a solid grasp of what libraries and tools are available and to stay up to date on the latest and greatest of what the Rust community has to offer. This chapter is dedicated to how you can leverage, track, understand, and contribute back to the Rust ecosystem. Since this is the final chapter, in the closing section I’ll also provide some suggestions of additional resources you can explore to continue developing your Rust skills. 

### What’s Out There?

Despite its relative youth, Rust already has an ecosystem large enough that it’s hard to keep track of everything that’s available. If you know what you want, you may be able to search your way to a set of appropriate crates and then use download statistics and superficial vibe-checks on each crate’s repository to determine which may make for reasonable dependencies. However, there’s also a plethora of tools, crates, and general language features that you might not necessarily know to look for that could potentially save you countless hours and difficult design decisions. 

In this section, I’ll go through some of the tools, libraries, and Rust features I have found helpful over the years in the hopes that they may come in useful for you at some point too! 

***Tools***

First off, here are some Rust tools I find myself using regularly that you should add to your toolbelt: 

```rust
cargo-deny
```

Provides a way to lint your dependency graph. At the time of writing, you can use cargo-deny to allow only certain licenses, deny-list crates or specific crate versions, detect dependencies with known vulnerabilities or that use Git sources, and detect crates that appear multiple times with different versions in the dependency graph. By the time you’re reading this, there may be even more handy lints in place. 

```
cargo-expand
```

Expands macros in a given crate and lets you inspect the output, which makes it much easier to spot mistakes deep down in macro transcribers or procedural macros. cargo-expand is an invaluable tool when you’re writing your own macros. 

```rust
cargo-hack
```

Helps you check that your crate works with any combination of features enabled. The tool presents an interface similar to that of Cargo itself (like cargo check, build, and test) but gives you the ability to run a given command with all possible combinations (the *powerset*) of the crate’s features. 

```
cargo-llvm-lines
```

Analyzes the mapping from Rust code to the intermediate representation (IR) that’s passed to the part of the Rust compiler that actually generates machine code (LLVM), and tells you which bits of Rust code produce the largest IR. This is useful because a larger IR means longer compile times, so identifying what Rust code generates a bigger IR (due to, for example, monomorphization) can highlight opportunities for reducing compile times. 

```
cargo-outdated
```

Checks whether any of your dependencies, either direct or transitive, have newer versions available. Crucially, unlike cargo update, it even tells you about new major versions, so it’s an essential tool for check- ing if you’re missing out on newer versions due to an outdated major version specifier. Just keep in mind that bumping the major version of a dependency may be a breaking change for your crate if you expose that dependency’s types in your interface! 

```rust
cargo-udeps
```

Identifies any dependencies listed in your *Cargo.toml* that are never actually used. Maybe you used them in the past but they’ve since become redundant, or maybe they should be moved to dev-dependencies; what- ever the case, this tool helps you trim down bloat in your dependency closure. 

While they’re not specifically tools for developing Rust, I highly recommend fd and ripgrep too—they’re excellent improvements over their predecessors find and grep and also happen to be written in Rust themselves. I use both every day. 

**Libraries** 

Next up are some useful but lesser-known crates that I reach for regularly, and that I suspect I will continue to depend on for a long time: 

```rust
bytes
```

Provides an efficient mechanism for passing around subslices of a single piece of contiguous memory without having to copy or deal with lifetimes. This is great in low-level networking code where you may need multiple views into a single chunk of bytes, and copying is a no-no. 

```rust
criterion
```

A statistics-driven benchmarking library that uses math to eliminate noise from benchmark measurements and reliably detect changes in performance over time. You should almost certainly be using it if you’re including micro-benchmarks in your crate. 

```
cxx
```

Provides a safe and ergonomic mechanism for calling C++ code from Rust and Rust code from C++. If you’re willing to invest some time into declaring your interfaces more thoroughly in advance in exchange for much nicer cross-language compatibility, this library is well worth your attention. 

```
flume
```

Implements a multi-producer, multi-consumer channel that is faster, more flexible, and simpler than the one included with the Rust stan- dard library. It also supports both asynchronous and synchronous oper- ation and so is a great bridge between those two worlds. 

```
hdrhistogram
```

A Rust port of the High Dynamic Range (HDR) histogram data structure, which provides a compact representation of histograms across a wide range of values. Anywhere you currently track averages or min/ max values, you should most likely be using an HDR histogram instead; it can give you much better insight into the distribution of your metrics. 

```
heapless
```

Supplies data structures that do not use the heap. Instead, heapless’s data structures are all backed by static memory, which makes them perfect for embedded contexts or other situations in which allocation is undesirable. 

```rust
itertools
```

Extends the Iterator trait from the standard library with lots of new convenient methods for deduplication, grouping, and computing pow- ersets. These extension methods can significantly reduce boilerplate in code, such as where you manually implement some common algorithm over a sequence of values, like finding the min and max at the same time (Itertools::minmax), or where you use a common pattern like check- ing that an iterator has exactly one item (Itertools::exactly_one). 

```
nix
```

Provides idiomatic bindings to system calls on Unix-like systems, which allows for a much better experience than trying to cobble together the C-compatible FFI types yourself when working with something like libc directly. 

```
pin-project
```

Provides macros that enforce the pinning safety invariants for anno- tated types, which in turn provide a safe pinning interface to those types. This allows you to avoid most of the hassle of getting Pin and Unpin right for your own types. There’s also pin-project-lite, which avoids the (currently) somewhat heavy dependency on the procedural macro machinery at the cost of slightly worse ergonomics. 

```
ring
```

Takes the good parts from the cryptography library BoringSSL, written in C, and brings them to Rust through a fast, simple, and hard-to-misuse interface. It’s a great starting point if you need to use cryptography in your crate. You’ve already most likely come across this in the rustls library, which uses ring to provide a modern, secure-by-default TLS stack. 

```
slab
```

Implements an efficient data structure to use in place of HashMap<Token, T>, where Token is an opaque type used only to differentiate between entries in the map. This kind of pattern comes up a lot when managing resources, where the set of current resources must be managed centrally but indi- vidual resources must also be accessible somehow. 

```
static_assertions
```

Provides static assertions—that is, assertions that are evaluated at, and thus may fail at, compile time. You can use it to assert things like that a type implements a given trait (like Send) or is of a given size. I highly recommend adding these kinds of assertions for code where those guarantees are likely to be important. 

```
structopt
```

Wraps the well-known argument parsing library clap and provides a way to describe your application’s command line interface entirely using the Rust type system (plus macro annotations). When you parse your appli- cation’s arguments, you get a value of the type you defined, and you thus get all the type checking benefits, like exhaustive matching and IDE auto-complete. 

```
thiserror
```

Makes writing custom enumerated error types, like the ones we discussed in Chapter 4, a joy. It takes care of implementing the recommended traits and following the established conventions and leaves you to define just the critical bits that are unique to your application. 

```
tower
```

Effectively takes the function signature async fn(Request) -> Response and implements an entire ecosystem on top of it. At its core is the Service trait, which represents a type that can turn a request into a response (something I suspect may make its way into the standard library one day). This is a great abstraction to build anything that looks like a ser- vice on top of. 

```
tracing
```

Provides all the plumbing needed to efficiently trace the execution of your applications. Crucially, it is agnostic to the types of events you’re tracing and what you want to do with those events. This library can be used for logging, metrics collection, debugging, profiling, and obviously tracing, all with the same machinery and interfaces. 

***Rust Tooling***

The Rust toolchain has a few features up its sleeve that you may not know to look for. These are usually for very specific use cases, but if they match yours, they can be lifesavers! 

**Rustup** 

Rustup, the Rust toolchain installer, does its job so efficiently that it tends to fade into the background and get forgotten about. You’ll occasionally use it to update your toolchain, set a directory override, or install a component, but that’s about it. However, Rustup supports one very handy trick that it’s worthwhile to know about: the toolchain override shorthand. You can pass +toolchain as the first argument to any Rustup-managed binary, and the binary will work as if you’d set an override for the given toolchain, run the command, and then reset the override back to what it was previously. So, cargo +nightly miri will run Miri using the nightly toolchain, and cargo +1.53.0 check will check if the code compiles with Rust 1.53.0. The latter comes in particularly handy for checking that you haven’t broken your minimum supported Rust version contract. 

Rustup also has a neat subcommand, doc, that opens a local copy of the Rust standard library documentation for the current version of the Rust compiler in your browser. This is invaluable if you’re developing on the go without an internet connection! 

**Cargo** 

Cargo also has some handy features that aren’t always easy to discover. The first of these is cargo tree, a Cargo subcommand built right into Cargo itself for inspecting a crate’s dependency graph. This command’s primary purpose is to print the dependency graph as a tree. This can be useful on its own, but where cargo tree really shines is through the --invert option: it takes a crate identifier and produces an inverted tree showing all the dependency paths from the current crate that bring in that dependency. So, for example, cargo tree -i rand will print all of the ways in which the current crate depends on any version of rand, including through transitive dependencies. This is invaluable if you want to eliminate a dependency, or a particular version of a dependency, and wonder why it still keeps being pulled in. You can also pass the -e features option to include information about why each Cargo feature of the crate in question is enabled. 

Speaking of Cargo subcommands, it’s really easy to write your own, whether for sharing with other people or just for your own local development. When Cargo is invoked with a subcommand it doesn’t recognize, it checks whether a program by the name cargo-$subcommand exists. If it does, Cargo invokes that program and passes it any arguments that were passed on the command line—so, cargo foo bar will invoke cargo-foo with the argument bar. Cargo will even integrate this command with cargo help by translating cargo help foo into a call to cargo-foo --help. 

As you work on more Rust projects, you may notice that Cargo (and Rust more generally) isn’t exactly forgiving when it comes to disk space. Each project gets its own target directory for its compilation artifacts, and over time you end up accumulating several identical copies of compiled artifacts for common dependencies. Keeping artifacts for each project separate is a sensible choice, as they aren’t necessarily compatible across projects (say, if one project uses different compiler flags than another). But in most developer environments, sharing build artifacts is entirely reasonable and can save a fair amount of compilation time when switching between projects. Luckily, configuring Cargo to share build artifacts is simple: just set [build] target in your *~/.cargo/config.toml* file to the directory you want those shared artifacts to go in, and Cargo will take care of the rest. No more tar- get directories in sight! Just make sure you clean out that directory every now and again too, and be aware that cargo clean will now clean *all* of your projects’ build artifacts. 

**N O T E** *Using a shared build directory can cause problems for projects that assume that com- piler artifacts will always be under the* target/ *subdirectory, so watch out for that. Also note that if a project* does *use different compiler flags, you’ll end up recompiling affected dependencies every time you move into or out of that project. In such cases, you’re best off overriding the target directory in that project’s Cargo configuration to a distinct location.* 

Finally, if you ever feel like Cargo is taking a suspiciously long time to build your crate, you can reach for the currently unstable Cargo -Ztimings flag. Running Cargo with that flag outputs information about how long it took to process each crate, how long build scripts took to run, what crates had to wait for what other crates to finish compiling, and tons of other useful metrics. This might highlight a particularly slow dependency chain that you can then work to eliminate, or reveal a build script that compiles a native dependency from scratch that you can make use system libraries instead. If you want to dive even deeper, there’s also rustc -Ztime-passes, which emits information about where time is spent inside of the compiler for each crate—though that information is likely only useful if you’re look- ing to contribute to the compiler itself. 

**rustc** 

The Rust compiler also has some lesser-known features that can prove useful to enterprising developers. The first is the currently unstable -Zprint- type-sizes argument, which prints the sizes of all the types in the current crate. This produces a lot of information for all but the tiniest crates but 

is immensely valuable when trying to determine the source of unexpected time spent in calls to memcpy or to find ways to reduce memory use when allocating lots of objects of a particular type. The -Zprint-type-sizes argument also displays the computed alignment and layout for each type, which may point you to places where turning, say, a usize into a u32 could have a significant impact on a type’s in-memory representation. After you debug a particular type’s size, alignment, and layout, I recommend adding static assertions to make sure that they don’t regress over time. You may also be interested in the variant_size_differences lint, which issues a warning if a crate contains enum types whose variants significantly differ in size. 

**N O T E**  *To call* *rustc* *with particular flags, you have a few options: you can either set them in the* *RUSTFLAGS* *environment variable or* *[build] rustflags* *in your* .cargo/config.toml *to have them apply to every invocation of* *rustc* *from Cargo, or you can use* *cargo rustc**, which will pass any arguments you provide only to the* *rustc* *invocation for the current crate.* 

If your profiling samples look weird, with stack frames reordered or entirely missing, you could also try -Cforce-frame-pointers = yes. Frame point- ers provide a more reliable way to unwind the stack—which is done a lot during profiling—at the cost of an extra register being used for function calls. Even though stack unwinding *should* work fine with just regular debug symbols enabled (remember to set debug = true when using the release pro- file), that’s not always the case, and frame pointers may take care of any issues you do encounter. 

***The Standard Library***

The Rust standard library is generally considered to be small compared to those of other programming languages, but what it lacks in breadth, it makes up for in depth; you won’t find a web server implementation or an X.509 certificate parser in Rust’s standard library, but you will find more than 40 different methods on the Option type alongside over 20 trait implementations. For the types it does include, Rust does its best to make avail- able any relevant functionality that meaningfully improves ergonomics, so you avoid all that verbose boilerplate that can so easily arise otherwise. In this section, I’ll present some types, macros, functions, and methods from the standard library that you may not have come across before, but that can often simplify or improve (or both) your code. 

**Macros and Functions** 

Let’s start off with a few free-standing utilities. First up is the **write!** macro, which lets you use format strings to write into a file, a network socket, or any- thing else that implements Write. You may already be familiar with it—but one little-known feature of write! is that it works with both std::io::Write and std::fmt::Write, which means you can use it to write formatted text directly into a String. That is, you can write use std::fmt::Write; write!(&mut s, "{}+1={}", x, x + 1); to append the formatted text to the String s! 

The **iter::once** function takes any value and produces an iterator that yields that value once. This comes in handy when calling functions that take iterators if you don’t want to allocate, or when combined with Iterator::chain to append a single item to an existing iterator. 

We briefly talked about **mem::replace** in Chapter 1, but it’s worth bring- ing up again in case you missed it. This function takes an exclusive refer- ence to a T and an owned T, swaps the two so that the referent is now the owned T, and returns ownership of the previous referent. This is useful when you need to take ownership of a value in a situation where you have only an exclusive reference, such as in implementations of Drop. See also mem::take for when T: Default. 

**Types** 

Next, let’s look at some handy standard library types. The **BufReader** and **BufWriter** types are a must for I/O operations that issue many small read or write calls to the underlying I/O resource. These types wrap the respec- tive underlying Read or Write and implement Read and Write themselves, but they additionally buffer the operations to the I/O resource such that many small reads do only one large read, and many small writes do only one large write. This can significantly improve performance as you don’t have to cross the system call barrier into the operating system as often. 

The **Cow** type, mentioned in Chapter 3, is useful when you want flex- ibility in what types you hold or need flexibility in what you return. You’ll rarely use Cow as a function argument (recall that you should let the caller allocate if necessary), but it’s invaluable as a return type as it allows you to accurately represent the return types of functions that may or may not allocate. It’s also a perfect fit for types that can be used as inputs *or* outputs, such as core types in RPC-like APIs. Say we have a type EntityIdentifier like in Listing 13-1 that is used in an RPC service interface. 

```
struct EntityIdentifier {
    namespace: String,
    name: String,
} 
```

Listing 13-1: A representation of a combined input/output type that requires allocation 

Now imagine two methods: get_entity takes an EntityIdentifier as an argument, and find_by returns an EntityIdentifier based on some search parameters. The get_entity method requires only a reference since the identifier will (presumably) be serialized before being sent to the server. But for find_by, the entity will be deserialized from the server response and must therefore be represented as an owned value. If we make get_entity take &EntityIdentifier, it will mean callers must still allocate owned Strings to call get_entity even though that’s not required by the interface, since it’s required to construct an EntityIdentifier in the first place! We could instead introduce a separate type for get_entity, EntityIdenifierRef, that holds only &str types, but then we’d have two types to represent one thing. Cow to the rescue! Listing 13-2 shows an EntityIdentifier that instead holds Cows internally. 

```
struct EntityIdentifier<'a> {
    namespace: Cow<'a, str>,
    name: Cow<'a str>,
} 
```

Listing 13-2: A representation of a combined input/output type that does not require allocation 

With this construction, get_entity can take any EntityIdentifier<'_>, which allows the caller to use just references to call the method. And find_ by can return EntityIdentifier<'static>, where all the fields are Cow::Owned. One type shared across both interfaces, with no unnecessary allocation requirements! 

**N O T E**  *If you implement a type this way, I recommend you also provide an* *into_owned* *method that turns an* *<'a>* *instance into a* *<'static>* *instance by calling* *Cow::into_ owned* *on all the fields. Otherwise, users will have no way to make longer-lasting clones of your type when all they have is an* *<'a>**.* 

The **std::sync::Once** type is a synchronization primitive that lets you run a given piece of code exactly once, at initialization time. This is great for initialization that’s part of an FFI where the library on the other side of the FFI boundary requires that the initialization is performed only once. 

The **VecDeque** type is an oft-neglected member of std::collections that I find myself reaching for surprisingly often—basically, whenever I need a stack or a queue. Its interface is similar to that of a Vec, and like Vec its in-memory representation is a single chunk of memory. The difference is that VecDeque keeps track of both the start and end of the actual data in that single allocation. This allows constant-time push and pop from *either* side of the VecDeque, meaning it can be used as a stack, as a queue, or even both at the same time. The cost you pay is that the values are no longer necessarily contiguous in memory (they may have wrapped around), which means that VecDeque<T> does not implement AsRef<[T]>. 

**Methods** 

Let’s round off with a rapid-fire look at some neat methods. First up is **Arc::make_mut**, which takes a &mut Arc<T> and gives you a &mut T. If the Arc is the last one in existence, it gives you the T that was behind the Arc; other- wise, it allocates a new Arc<T> that holds a clone of the T, swaps that in for the currently referenced Arc, and then gives &mut to the T in the new single- ton Arc. 

The **Clone::clone_from** method is an alternative form of .clone() that lets you reuse an instance of the type you clone rather than allocate a new one. In other words, if you already have an x: T, you can do x.clone_from(y) rather than x = y.clone(), and you might save yourself some allocations. 

**std::fmt::Formatter::debug_\*** is by far the easiest way to implement Debug yourself if #[derive(Debug)] won’t work for your use case, such as if you want to include only some fields or expose information that isn’t exposed by the Debug implementations of your type’s fields. When implementing the fmt method of Debug, simply call the appropriate debug_ method on the Formatter that’s passed in (debug_struct or debug_map, for example), call the included methods on the resulting type to fill in details about the type (like field to add a field or entries to add a key/value entry), and then call finish. 

**Instant::elapsed** returns the Duration since an Instant was created. This is much more concise than the common approach of creating a new Instant and subtracting the earlier instance. 

**Option::as_deref** takes an Option<P> where P: Deref and returns Option<&P::Target> (there’s also an as_deref_mut method). This simple operation can make functional transformation chains that operate on Option much cleaner by avoiding the inscrutable .as_ref().map(|r| &**r). 

**Ord::clamp** lets you take any type that implements Ord and clamp it between two other values of a given range. That is, given a lower limit min and an upper limit max, x.clamp(min, max) returns min if x is less than min, max if x is greater than max, and x otherwise. 

**Result::transpose** and its counterpart **Option::transpose** invert types that nest Result and Option. That is, transposing a Result<Option<T>, E> gives an Option<Result<T, E>>, and vice versa. When combined with ?, this operation can make for cleaner code when working with Iterator::next and similar methods in fallible contexts. 

**Vec::swap_remove** is Vec::remove’s faster twin. Vec::remove preserves the order of the vector, which means that to remove an element in the middle, it must shift all the later elements in the vector down by one. This can be very slow for large vectors. Vec::swap_remove, on the other hand, swaps the to-be-removed element with the last element and then truncates the vector’s length by one, which is a constant-time operation. Be aware, though, that it will shuffle your vector around and thus invalidate old indexes! 

### Patterns in the Wild

As you start exploring codebases that aren’t your own, you’ll likely come across a couple of common Rust patterns that we haven’t discussed in the book so far. Knowing about them will make it easier to recognize them, and thus understand their purpose, when you do encounter them. You may even find use for them in your own codebase one day! 

**Index Pointers** 

Index pointers allow you to store multiple references to data within a data structure without running afoul of the borrow checker. For example, if you want to store a collection of data so that it can be efficiently accessed in more than one way, such as by keeping one HashMap keyed by one field and one keyed by a different field, you don’t want to store the underlying data multiple times too. You could use Arc or Rc, but they use dynamic reference counting that introduces unnecessary overhead, and the extra bookkeeping requires you to store additional bytes per entry. You could use references, but the lifetimes become difficult if not impossible to manage because the data and the references live in the same data structure (it’s a self-referential data structure, as we discussed in Chapter 8). You could use raw pointers combined with Pin to ensure the pointers remain valid, but that introduces a lot of complexity as well as unsafety you then need to carefully consider. 

Most crates use index pointers—or, as I like to call them, *indeferences*— instead. The idea is simple: store each data entry in some indexable data structure like a Vec, and then store just the index in a derived data struc- ture. To then perform an operation, first use the derived data structure to efficiently find the data index, and then use the index to retrieve the referenced data. No lifetimes needed—and you can even have cycles in the derived data representation if you wish! 

The indexmap crate, which provides a HashMap implementation where the iteration order matches the map insertion order, provides a good example of this pattern. The implementation has to store the keys in two places, both in the map of keys to values and in the list of all the keys, but it obvi- ously doesn’t want to keep two copies in case the key type itself is large. So, it uses index pointers. Specifically, it keeps all the key/value pairs in a single Vec and then keeps a mapping from key hashes to Vec indexes. To iterate over all the elements of the map, it just walks the Vec. To look up a given key, it hashes that key, looks that hash up in the mapping, which yields the key’s index in the Vec (the index pointer), and then uses that to get the key’s value from the Vec. 

The petgraph crate, which implements graph data structures and algo- rithms, also uses this pattern. The crate stores one Vec of all node values and another of all edge values and then only ever uses the indexes into those Vecs to refer to a node or edge. So, for example, the two nodes associ- ated with an edge are stored in that edge simply as two u32s, rather than as references or reference-counted values. 

The trick lies in how you support deletions. To delete a data entry, you first need to search for its index in all of the derived data structures and remove the corresponding entries, and then you need to remove the data from the root data store. If the root data store is a Vec, removing the entry will also change the index of one other data entry (when using swap_remove), so you then need to go update all the derived data structures to reflect the new index for the entry that moved. 

**Drop Guards** 

Drop guards provide a simple but reliable way to ensure that a bit of code runs even in the presence of panics, which is often essential in unsafe code. An example is a function that takes a closure f: FnOnce and executes it under mutual exclusion using atomics. Say the function uses compare_exchange (dis- cussed in Chapter 10) to set a Boolean from false to true, calls f, and then sets the Boolean back to false to end the mutual exclusion. But consider what happens if f panics—the function will never get to run its cleanup, and no other call will be able to enter the mutual exclusion section ever again. 

It’s possible to work around this using catch_unwind, but drop guards provide an alternative that is often more ergonomic. Listing 13-3 shows how, in our current example, we can use a drop guard to ensure the Boolean always gets reset. 

```
fn mutex(lock: &AtomicBool, f: impl FnOnce()) {
    // .. while lock.compare_exchange(false, true).is_err() ..
    struct DropGuard<'a>(&'a AtomicBool);
    impl Drop for DropGuard<'_> {
        fn drop(&mut self) {
            lock.store(true, Ordering::Release);
        }}
        let _guard = DropGuard(lock);
f(); } 
```

Listing 13-3: Using a drop guard to ensure code gets run after an unwinding panic 

We introduce the local type DropGuard that implements Drop and place the cleanup code in its implementation of Drop::drop. Any necessary state can be passed in through the fields of DropGuard. Then, we construct an instance of the guard type just before we call the function that might panic, which is f here. When f returns, whether due to a panic or because it returns normally, the guard is dropped, its destructor runs, the lock is released, and all is well. 

It’s important that the guard is assigned to a variable that is dropped at the end of the scope, after the user-provided code has been executed. This means that even though we never refer to the guard’s variable again, it needs to be given a name, as let _ = DropGuard(lock) would drop the guard immediately—before the user-provided code even runs! 

**N O T E** *Like* *catch_unwind**, drop guards work only when panics unwind. If the code is com- piled with* *panic=abort**, no code gets to run after the panic.* 

This pattern is frequently used in conjunction with thread locals, when library code may wish to set the thread local state so that it’s valid only for the duration of the execution of the closure, and thus needs to be cleared afterwards. For example, at the time of writing, Tokio uses this pattern to provide information about the executor calling Future::poll to leaf resources like TcpStream without having to propagate that information through function signatures that are visible to users. It’d be no good if the thread local state continued to indicate that a particular executor thread was active even after Future::poll returned due to a panic, so Tokio uses a drop guard to ensure that the thread local state is reset. 

**N O T E**  *You’ll often see* *Cell* *or* *Rc* *used in thread locals. This is because thread locals are accessible only through shared references, since a thread might access a thread local again that it is already referencing somewhere higher up in the call stack. Both types provide interior mutability without incurring much overhead because they’re intended only for single-threaded use, and so are ideal for this use case.* 

**Extension Traits** 

Extension traits allow crates to provide additional functionality to types that implement a trait from a different crate. For example, the itertools crate provides an extension trait for Iterator, which adds a number of con- venient shortcuts for common (and not so common) iterator operations. As another example, tower provides ServiceExt, which adds several more ergo- nomic operations to wrap the low-level interface in the Service trait from tower-service. 

Extension traits tend to be useful either when you do not control the base trait, as with Iterator, or when the base trait lives in a crate of its own so that it rarely sees breaking releases and thus doesn’t cause unnecessary ecosystem splits, as with Service. 

An extension trait extends the base trait it is an extension of (trait ServiceExt: Service) and consists solely of provided methods. It also comes with a blanket implementation for any T that implements the base trait (impl<T> ServiceExt for T where T: Service {}). Together, these conditions ensure that the extension trait’s methods are available on anything that implements the base trait. 

***Crate Predules***

In Chapter 12, we talked about the standard library prelude that makes a number of types and traits automatically available without you having to write any use statements. Along similar lines, crates that export multiple types, traits, or functions that you’ll often use together sometimes define their own prelude in the form of a module called prelude, which re-exports some particularly common subset of those types, traits, and functions. There’s nothing magical about that module name, and it doesn’t get used automatically, but it serves as a signal to users that they likely want to add use *somecrate*::prelude::* to files that want to use the crate in question. The * is a *glob import* and tells Rust to use all publicly available items from the indicated module. This can save quite a bit of typing when the crate has a lot of items you’ll usually need to name. 

**N O T E**  *Items used through* *** *have lower precedence than items that are used explicitly by name. This is what allows you to define items in your own crate that overlap with what’s in the standard library prelude without having to specify which one to use.* 

Preludes are also great for crates that expose a lot of extension traits, since trait methods can be called only if the trait that defines them is in scope. For example, the diesel crate, which provides ergonomic access to relational databases, makes extensive use of extension traits so you can write code like: 

```
posts.filter(published.eq(true)).limit(5).load::<Post>(&connection)
```

This line will work only if all the right traits are in scope, which the prelude takes care of. 

In general, you should be careful when adding glob imports to your code, as they can potentially turn additions to the indicated module into backward-incompatible changes. For example, if someone adds a new trait to a module you glob-import from, and that new trait makes a method foo available on a type that already had some other foo method, code that calls foo on that type will no longer compile as the call to foo is now ambiguous. Interestingly enough, while the existence of glob imports makes any module addition a technically breaking change, the Rust RFC on API evolution (RFC 1105; see *https://rust-lang.github.io/rfcs/1105-api-evolution.html*) does *not* require a library to issue a new major version for such a change. The RFC goes into great detail about why, and I recommend you read it, but the gist is that minor releases are allowed to require minimally invasive changes to dependents, like having to add type annotations in edge cases, because oth- erwise a large fraction of changes would require new major versions despite being very unlikely to actually break any consumers. 

Specifically in the case of preludes, using glob imports is usually fine when recommended by the vending crate, since its maintainers know that their users will use glob imports for the prelude module and thus will take that into account when deciding whether a change requires a major version bump. 

### Staying Up to Date

Rust, being such a young language, is evolving rapidly. The language itself, the standard library, the tooling, and the broader ecosystem are all still in their infancy, and new developments happen every day. While staying on top of all the changes would be infeasible, it’s worth your time to keep up with significant developments so that you can take advantage of the latest and greatest features in your projects. 

For monitoring improvements to Rust itself, including new language features, standard library additions, and core tooling upgrades, the official Rust blog at *https://blog.rust-lang.org/* is a good, low-volume place to start. It mainly features announcements for each new Rust release. I recommend you make a habit of reading these, as they tend to include interesting tid- bits that will slowly but surely deepen your knowledge of the language. To dig a little deeper, I highly recommend reading the detailed changelogs for Rust and Cargo as well (links can usually be found near the bottom of each release announcement). The changelogs surface changes that weren’t large enough to warrant a paragraph in the release notes but that may be just what you need two weeks from now. For a less frequently updated news source, check in on *The Edition Guide* at *https://doc.rust-lang.org/edition-guide/*, which outlines what’s new in each Rust edition. Rust editions tend to be released every three years. 

**N O T E** *Clippy is often able to tell you when you can take advantage of a new language or standard library feature—always enable Clippy!* 

If you’re curious about how Rust itself is developed, you may also want to subscribe to the *Inside Rust* blog at *https://blog.rust-lang.org/inside-rust/*. It includes updates from the various Rust teams, as well as incident reports, larger change proposals, edition planning information, and the like. To get involved in Rust development yourself—which I highly encourage, as it’s a lot of fun and a great learning experience—you can check out the vari- ous Rust working groups at *https://www.rust-lang.org/governance/*, which each focus on improving a specific aspect of Rust. Find one that appeals to you, check in with the group wherever it meets and ask how you may be able to help. You can also join the community discussion about Rust internals over at *https://internals.rust-lang.org/*; this is another great way to get insight into the thought that goes into every part of Rust’s design and development. 

As is the case for most programming languages, much of Rust’s value is derived from its community. Not only do the members of the Rust community constantly develop new work-saving crates and discover new Rust- specific techniques and design patterns, but they also collectively and continuously help one another understand, document, and explain how to take best advantage of the Rust language. Everything I have covered in this book, and much more, has already been discussed by the community in thousands of comment threads, blog posts, and Twitter and Discord conversations. Dipping into these discussions even just once in a while is almost guaranteed to show you new things about a language feature, a technique, or a crate that you didn’t already know. 

The Rust community lives in a lot of places, but some good places to start are the Users forum (*https://users.rust-lang.org/*), the Rust subreddit (*https://www.reddit.com/r/rust/*), the Rust Community Discord (*https://discord .gg/rust-lang-community*), and the Rust Twitter account (*https://twitter.com/ rustlang*). You don’t have to engage with all of these, or all of the time— pick one you like the vibe of, and check in occasionally! 

A great single location for staying up to date with ongoing developments is the *This Week in Rust* blog (*https://this-week-in-rust.org/*), a “weekly summary of [Rust’s] progress and community.” It links to official announcements and changelogs as well as popular community discussions and resources, interest- ing new crates, opportunities for contributions, upcoming Rust events, and Rust job opportunities. It even lists interesting language RFCs and compiler PRs, so this site truly has it all! Discerning what information is valuable to you and what isn’t may be a little daunting, but even just scrolling through and clicking occasional links that appear interesting is a good way to keep a steady stream of new Rust knowledge trickling into your brain. 

**N O T E** *Want to look up when a particular feature landed on stable? Can I Use... (*https://caniuse.rs/*) has you covered.* 

### What Next?

So, you’ve read this book front to back, absorbed all the knowledge it imparts, and are still hungry for more? Great! There are a number of other excellent resources out there for broadening and deepening your knowledge and understanding of Rust, and in this very final section I’ll give you a survey of some of my favorites so that you can keep learning. I’ve divided them into subsections based on how different people prefer to learn so that you can find resources that’ll work for you. 

**N O T E** *A challenge with learning on your own, especially in the beginning, is that progress is hard to perceive. Implementing even the simplest of things can take an outsized amount of time when you have to constantly refer to documentation and other resources, ask for help, or debug to learn how some aspect of Rust works. All of that non-coding work can make it seem like you’re treading water and not really improv- ing. But you’re* learning*, which is progress in and of itself—it’s just harder to notice and appreciate.* 

**Learn by Watching** 

Watching experienced developers code is essentially a life hack to remedy the slow starting phase of solo learning. It allows you to observe the pro- cess of designing and building while utilizing someone else’s experience. Listening to experienced developers articulate their thinking and explain tricky concepts or techniques as they come up can be an excellent alter- native to struggling through problems on your own. You’ll also pick up a variety of auxiliary knowledge like debugging techniques, design patterns, and best practices. Eventually you will have to sit down and do things your- self—it’s the only way to check that you actually understand what you’ve observed—but piggybacking on the experience of others will almost certainly make the early stages more pleasant. And if the experience is interactive, that’s even better! 

So, with that said, here are some Rust video channels that I recommend: 

Perhaps unsurprisingly, my own channel: *https://www.youtube.com/c/ JonGjengset/*. I have a mix of long-form coding videos and short(er) code- based theory/concept explanation videos, as well as occasional videos that dive into interesting Rust coding stories. 

The *Awesome Rust Streaming* listing: *https://github.com/jamesmunns/ awesome-rust-streaming/*. This resource lists a wide variety of developers who stream Rust coding or other Rust content. 

The channel of Tim McNamara, the author of *Rust in Action*: *https:// www.youtube.com/c/timClicks/*. Tim’s channel, like mine, splits its time between implementation and theory, though Tim has a particular knack for creative visual projects, which makes for fun viewing. 

Jonathan Turner’s *Systems with JT* channel: *https://www.youtube.com/c/ SystemswithJT/*. Jonathan’s videos document their work on Nushell, their take on a “new type of shell,” providing a great sense of what it’s like to work on a nontrivial existing codebase. 

Ryan Levick’s channel: *https://www.youtube.com/c/RyanLevicksVideos/*. Ryan mainly posts videos that tackle particular Rust concepts and walks through them using concrete code examples, but he also occasionally does implementation videos (like FFI for Microsoft Flight Simulator!) and deep dives into how well-known crates work under the hood. 

Given that I make Rust videos, it should come as no surprise that I am a fan of this approach to teaching. But this kind of receptive or interactive learning doesn’t have to come in the form of videos. Another great avenue for learning from experienced developers is pair programming. If you have a colleague or friend with expertise in a particular aspect of Rust you’d like to learn, ask if you can do a pair-programming session with them to solve a problem together! 

***Learn by Doing***

Since your ultimate goal is to get better at writing Rust, there’s no substitute for programming experience. No matter what or how many resources you learn from, you need to put that learning into practice. However, finding a good place to start can be tricky, so here I’ll give some suggestions. 

Before I dive into the list, I want to provide some general guidance on how to pick projects. First, choose a project that *you* care about, without worrying too much whether others care about it. While there are plenty of popular and established Rust projects out there that would love to have you as a contributor, and it’s fun to be able to say “I contributed to the well- known library X,” your first priority must be your own interest. Without concrete motivation, you’ll quickly lose steam and find contributing to be a chore. The very best targets are projects that you use yourself and have experienced problems with—go fix them! Nothing is more satisfying than getting rid of a long-standing personal nuisance while also contributing back to the community. 

Okay, so back to project suggestions. First and foremost, consider con- tributing to the Rust compiler and its associated tools. It’s a high-quality codebase with good documentation and an endless supply of issues (you probably know of some yourself), and there are several great mentors who can provide outlines for how to approach solving issues. If you look through the issue tracker for issues marked E-easy or E-mentor, you’ll likely find a good candidate quickly. As you gain more experience, you can keep level- ing up to contribute to trickier parts. 

If that’s not your cup of tea, I recommend finding something you use frequently that’s written in another language and porting it to Rust—not necessarily with the intention of replacing the original library or tool, but just because the experience will allow you to focus on writing Rust without having to spend too much time coming up with all the functionality your- self. If it turns out well, the fact that it already exists suggests that someone else also needed it, so there may be a wider audience for your port too! Data structures and command-line tools often make for great porting subjects, but find a niche that appeals to you. 

Should you be more of a “build it from scratch” kind of person, I recommend looking back at your own development experience so far and thinking about similar code you’ve ended up writing in multiple projects (whether in Rust or in other languages). Such repetition tends to be a good signal that something is reusable and could be turned into a library. If nothing comes to mind, David Tolnay maintains a list of smaller utility crates that other Rust developers have requested at *https://github.com/dtolnay/request-for- implementation/* that may provide a source of inspiration. If you’re looking for something more substantial and ambitious, there’s also the Not Yet Awesome list at *https://github.com/not-yet-awesome-rust/not-yet-awesome-rust/* that lists things that should exist in Rust but don’t (yet). 

***Learn by Reading***

Although the state of affairs is constantly improving, finding good Rust reading material beyond the beginner level can still be tricky. Here’s a collection of pointers to some of my favorite resources that continue to teach me new things or serve as good references when I have particularly niche or nuanced questions. 

First, I recommend looking through the official virtual Rust books linked from *https://www.rust-lang.org/learn/*. Some, like the Cargo book, are more reference-like while others, like the Embedded book, are more guide- like, but they’re all deep sources of solid technical information about their respective topics. *The Rustonomicon* (*https://doc.rust-lang.org/nomicon/*), in particular, is a lifesaver when you’re writing unsafe code. 

Two more books that are worth checking out are the *Guide to rustc Development* (*https://rustc-dev-guide.rust-lang.org/*) and the *Standard Library Developers Guide* (*https://std-dev-guide.rust-lang.org/*). These are fantastic resources if you’re curious about how the Rust compiler does what it does or how the standard library is designed, or if you want some pointers before you try your hand at contributing to Rust itself. The official Rust guidelines are also a treasure trove of information; I’ve already mentioned the *Rust API Guidelines* (*https://rust-lang.github.io/api-guidelines/*) in the book, but a *Rust Unsafe Code Guidelines Reference* is also available (*https://rust-lang.github.io/unsafe-code-guidelines/*), and by the time you read this book there may be more. 

**N O T E** *One of the resources listed at* https://www.rust-lang.org/learn/ *is the Rust Reference, which is essentially a full specification for the Rust language. While parts of it are quite dry, like the exact grammar used for parsing or basics about the in- memory representations of the primitive types, some of it is fascinating reading, like the section on type layout and the enumeration of behavior considered undefined.* 

There are also a number of unofficial virtual Rust books that are enormously valuable collections of experience and knowledge. *The Little Book of Rust Macros* (*https://veykril.github.io/tlborm/*), for example, is indispensable if you want to write nontrivial declarative macros, and *The Rust Performance Book* (*https://nnethercote.github.io/perf-book/*) is filled with tips and tricks for improving the performance of Rust code both at the micro and the macro level. Other great resources include the *Rust Fuzz Book* (*https:// rust-fuzz.github.io/book/*), which explores fuzz testing in more detail, and the *Rust Cookbook* (*https://rust-lang-nursery.github.io/rust-cookbook/*), which suggests idiomatic solutions to common programming tasks. There’s even a resource for finding more books, *The Little Book of Rust Books* (*https://lborb. github.io/book/unofficial.html* )! 

If you prefer more hands-on reading, the Tokio project has published *mini-redis* (*https://github.com/tokio-rs/mini-redis/*), an incomplete but idiomatic implementation of a Redis client and server that’s extremely well documented and specifically written to serve as a guide to writing asynchronous code. If you’re more of a data structures person, *Learn Rust with Entirely Too Many Linked Lists* (*https://rust-unofficial.github.io/too-many-lists/*) is an enlightening and fun read that gets into lots of gnarly details about ownership and references. If you’re looking for something closer to the hardware, Philipp Oppermann’s *Writing an OS in Rust* (*https://os.phil-opp.com/*) goes through the whole operating system stack in great detail while teaching you good Rust patterns in the process. I also highly recommend Amos’s collection of articles (*https://fasterthanli.me/tags/rust/*) if you want a wide sampling of interesting deep dives written in a conversational style. 

When you feel more confident in your Rust abilities and need more of a quick reference than a long tutorial, I’ve found the *Rust Language Cheat Sheet* (*https://cheats.rs/*) great for looking things up quickly. It also provides very nice visual explanations for most topics, so even if you’re looking up something you’re not intimately familiar with already, the explanations are pretty approachable. 

And finally, if you want to put all of your Rust understanding to the test, go give David Tolnay’s *Rust Quiz* (*https://dtolnay.github.io/rust-quiz/*) a try. There are some real mind-benders in there, but each question comes with a thorough explanation of what’s going on, so even if you get one wrong, you’ll have learned from the experience! 

***Learn by Teaching***

My experience has been that the best way to learn something well and thoroughly, by far, is to try to teach it to others. I have learned an enormous amount from writing this book, and I learn new things every time I make a new Rust video or podcast episode. So, I wholeheartedly recommend that you try your hand at teaching others about some of the things you’ve learned from reading this book or that you learn from here on out. It can take whatever form you prefer: in person, writing a blog post, tweet- ing, making a video or podcast, or giving a talk. The important thing is that you try to convey your newfound knowledge in your own words to someone who doesn’t already understand the topic—in doing so, you also give back to the community so that the next you that comes along has a slightly easier time getting up to speed. Teaching is a humbling and deeply educational experience, and I cannot recommend it highly enough. 

*Whether you’re looking to teach or be taught, make sure to visit Awesome Rust Mentors (*https://rustbeginners.github.io/awesome-rust-mentors/*).* 

## Summary

In this chapter, we’ve covered Rust beyond what exists in your local work- space. We surveyed useful tools, libraries, and Rust features; looked at how to stay up to date as the ecosystem continues to evolve; and then discussed how you can get your hands dirty and contribute back to the ecosystem yourself. Finally, we discussed where you can go next to continue your Rust journey now that this book has reached its end. And with that, there’s little more to do than to declare: 
``` rust
} 
```