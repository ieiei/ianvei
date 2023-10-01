Rust is intended to be a language for systems programming, but it isn’t always clear what that really means. At the very least, a systems programming language is usually expected to allow the programmer to write programs that do not rely on the operating system and can run directly on the hardware, whether that is a thousandcore supercomputer or an embedded device with a single-core ARM processor with a clock speed of 72MHz and 256KiB of memory. 

In this chapter, we’ll take a look at how you can use Rust in unorthodox environments, such as those without an operating system, or those that don’t even have the ability to dynamically allocate memory! Much of our discussion will focus on the #![no_std] attribute, but we’ll also investigate 

Rust’s alloc module, the Rust runtime (yes, Rust does technically have a runtime), and some of the tricks you have to play to write up a Rust binary for use in such an environment. 

## Opting Out of the Standard Library

As a language, Rust consists of multiple independent pieces. First there’s the compiler, which dictates the grammar of the Rust language and imple- ments type checking, borrow checking, and the final conversion into machine-runnable code. Then there’s the standard library, std, which implements all the useful common functionality that most programs need—things like file and network access, a notion of time, facilities for printing and reading user input, and so on. But std itself is also a compos- ite, building on top of two other, more fundamental libraries called core and alloc. In fact, many of the types and functions in std are just re-exports from those two libraries. 

The core library sits at the bottom of the standard library pyramid and contains any functionality that depends on nothing but the Rust language itself and the hardware the resulting program is running on—things like sorting algorithms, marker types, fundamental types such as Option and Result, low-level operations such as atomic memory access methods, and compiler hints. The core library works as if the operating system does not exist, so there is no standard input, no filesystem, and no network. Similarly, there is no memory allocator, so types like Box, Vec, and HashMap are nowhere to be seen. 

Above core sits alloc, which holds all the functionality that depends on dynamic memory allocation, such as collections, smart pointers, and dynamically allocated strings (String). We’ll get back to alloc in the next section. 

Most of the time, because std re-exports everything in core and alloc, developers do not need to know about the differences among the three libraries. This means that even though Option technically lives in core::option::Option, you can access it through std::option::Option. 

However, in an unorthodox environment, such as on an embedded device where there is no operating system, the distinction is crucial. While it’s fine to use an Iterator or to sort a list of numbers, an embedded device may simply have no meaningful way to access a file (as that requires a file- system) or print to the terminal (as that requires a terminal)—so there’s no File or println!. Furthermore, the device may have so little memory that dynamic memory allocation is a luxury you can’t afford, and thus anything that allocates memory on the fly is a no-go—say goodbye to Box and Vec. 

Rather than force developers to carefully avoid those basic constructs in such environments, Rust provides a way to opt out of anything but the core functionality of the language: the #![no_std] attribute. This is a crate- level attribute (#!) that switches the prelude (see the box on page 213) for the crate from std::prelude to core::prelude so that you don’t accidentally depend on anything outside of core that might not work in your target environment. 

However, that is *all* the #![no_std] attribute does—it does not prevent you from bringing in the standard library explicitly with extern std. This may be surprising, as it means a crate marked #![no_std] may in fact not be compatible with a target environment that does not support std, but this design decision was intentional: it allows you to mark your crate as being no_std-compatible but to still use features from the standard library when certain features are enabled. For example, many crates have a feature named std that, when enabled, gives access to more sophisticated APIs and integra- tions with types that live in std. This allows crate authors to both supply the core implementation for constrained use cases and add bells and whistles for consumers on more standard platforms. 

**N O T E** *Since features should be additive, prefer an* *std**-enabling feature to an* *std**-disabling one. Otherwise, if* any *crate in a consumer’s dependency graph enables the no-**std* *feature,* all *consumers will be given access only to the bare-bones API without* *std* *sup- port, which may then mean that APIs they depend on aren’t available, causing them to no longer compile.* 

> **THE PRELUDE** 
>
> Have you ever wondered why there are some types and traits—like Box, Iterator, Option, and Clone—that are available in every Rust file without you needing to use them? Or why you don’t need to use any of the macros in the standard library (like vec![])? The reason is that every Rust module automatically imports the Rust standard prelude with an implicit use std::prelude::rust_2021::* (or similar for other editions), which brings all the exports from the crate’s chosen edition’s prelude into scope . The prelude modules themselves aren’t special beyond this auto-inclusion—they are merely collections of pub use statements for key types, traits, and macros that the Rust developers expect to be commonly used . 

### Dynamic Memory Allocation

As we discussed in Chapter 1, a machine has many different regions of memory, and each one serves a distinct purpose. There’s static memory for your program code and static variables, there’s the stack for function-local variables and function arguments, and there’s the heap for, well, every- thing else. The heap supports allocating variably sized regions of memory at runtime, and those allocations stick around for however long you want them to. This makes heap memory extremely versatile, and as a result, you find it used everywhere. Vec, String, Arc and Rc, and the collection types are all implemented in heap memory, which allows them to grow and shrink over time and to be returned from functions without the borrow checker complaining. 

Behind the scenes, the heap is really just a huge chunk of contiguous memory that is managed by an *allocator*. It’s the allocator that provides the illusion of distinct allocations in the heap, ensuring that those allocations do not overlap and that regions of memory that are no longer in use are reused. By default Rust uses the system allocator, which is generally the one dictated by the standard C library. This works well for most use cases, but if necessary, you can override which allocator Rust will use through the GlobalAlloc trait combined with the #[global_allocator] attribute, which requires an imple- mentation of an alloc method for allocating a new segment of memory and dealloc for returning a past allocation to the allocator to reuse. 

In environments without an operating system, the standard C library is also generally not available, and so neither is the standard system alloca- tor. For that reason, #![no_std] also excludes all types that rely on dynamic memory allocation. But since it’s entirely possible to implement a memory allocator without access to a full-blown operating system, Rust allows you to opt back into just the part of the Rust standard library that requires an allocator without opting into all of std through the alloc crate. The alloc crate comes with the standard Rust toolchain (just like core and std) and contains most of your favorite heap-allocation types, like Box, Arc, String, Vec, and BTreeMap. HashMap is not among them, since it relies on random num- ber generation for its key hashing, which is an operating system facility. To use types from alloc in a no_std context, all you have to do is replace any imports of those types that previously had use std:: with use alloc:: instead. Do keep in mind, though, that depending on alloc means your #![no_std] crate will no longer be usable by any program that disallows dynamic mem- ory allocation, either because it doesn’t have an allocator or because it has too little memory to permit dynamic memory allocation in the first place. 

**NOTE** *Some programming domains, like the Linux kernel, may allow dynamic memory allocation only if out-of-memory errors are handled gracefully (that is, without pan- icking). For such use cases, you’ll want to provide* *try_* *versions of any methods you expose that might allocate. The* *try_* *methods should use fallible methods of any inner types (like the currently unstable* *Box::try_new* *or* *Vec::try_reserve**) rather than ones that just panic (like* *Box::new* *or* *Vec::reserve)* *and propagate those errors out to the caller, who can then handle them appropriately.* 

It might strike you as odd that it’s possible to write nontrivial crates that use *only* core. After all, they can’t use collections, the String type, the net- work, or the filesystem, and they don’t even have a notion of time! The trick to core-only crates is to utilize the stack and static allocations. For example, for a heapless vector, you allocate enough memory up front—either in static memory or in a function’s stack frame—for the largest number of elements you expect the vector to be able to hold, and then augment it with a usize that tracks how many elements it currently holds. To push to the vector, you write to the next element in the (statically sized) array and increment a variable that tracks the number of elements. If the vector’s length ever reaches the static size, the next push fails. Listing 12-1 gives an example of such a heapless vector type implemented using const generics. 

``` rust
 struct ArrayVec<T, const N: usize> {
	 values: [Option<T>; N],
	 len: usize,
 }
 impl<T, const N: usize> ArrayVec<T, N> {
	 fn try_push(&mut self, t: T) -> Result<(), T> {
		 if self.len == N {
			 return Err(t);
		 }
		 self.values[self.len] = Some(t);
		 self.len += 1;
		 return Ok(());
	 }
 } 
```


Listing 12-1: A heapless vector type 

We make ArrayVec generic over both the type of its elements, T, and the maximum number of elements, N, and then represent the vector as an array of N *optional* Ts. This structure always stores N Option<T>, so it has a size known at compile time and can be stored on the stack, but it can still act like a vector by using runtime information to inform how we access the array. 

**NOTE** *We could have implemented* *ArrayVec* *using* *[MaybeUninit; N]* *to avoid the over- head of the* *Option**, but that would require using unsafe code, which isn’t warranted for this example.* 

### The Rust Runtime

You may have heard the claim that Rust doesn’t have a runtime. While that’s true at a high level—it doesn’t have a garbage collector, an inter- preter, or a built-in user-level scheduler—it’s not really true in the strictest sense. Specifically, Rust does have some special code that runs before your main function and in response to certain special conditions in your code, which really is a form of bare-bones runtime. 

**The Panic Handler** 

The first bit of such special code is Rust’s *panic handler*. When Rust code panics by invoking panic! or panic_any, the panic handler dictates what happens next. When the Rust runtime is available—as is the case on most targets that supply std—the panic handler first invokes the *panic hook* set via std::panic::set_hook, which prints a message and optionally a backtrace to standard error by default. It then either unwinds the current thread’s stack or aborts the process, depending on the panic setting chosen for cur- rent compilation (either through Cargo configuration or arguments passed directly to rustc). 

However, not all targets provide a panic handler. For example, most embedded targets do not, as there isn’t necessarily a single implementation that makes sense across all the uses for such a target. For targets that don’t supply a panic handler, Rust still needs to know what to do when a panic occurs. To that end, we can use the #[panic_handler] attribute to decorate a single function in the program with the signature fn(&PanicInfo) -> !. That function is called whenever the program invokes a panic, and it is passed information about the panic in the form of a core::panic::PanicInfo. What the function does with that information is entirely unspecified, but it can never return (as indicated by the ! return type). This is important, since the Rust compiler assumes that no code that follows a panic is run. 

There are many valid ways for a panic handler to avoid returning. The standard panic handler unwinds the thread’s stack and then terminates the thread, but a panic handler can also halt the thread using loop {}, abort the program, or do anything else that makes sense for the target platform, even as far as resetting the device. 

***Program Initialization***

Contrary to popular belief, the main function is not the first thing that runs in a Rust program. Instead, the main symbol in a Rust binary actually points to a function in the standard library called lang_start. That function per- forms the (fairly minimal) setup for the Rust runtime, including stashing the program’s command-line arguments in a place where std::env::args can get to them, setting the name of the main thread, handling panics in the main function, flushing standard output on program exit, and setting up signal handlers. The lang_start function in turn calls the main function defined in your crate, which then doesn’t need to think about how, for example, Windows and Linux differ in how command-line arguments are passed in. 

This arrangement works well on platforms where all of that setup is sensible and supported, but it presents a problem on embedded platforms where main memory may not even be accessible when the program starts. On such platforms, you’ll generally want to opt out of the Rust initialization code entirely using the #![no_main] crate-level attribute. This attribute completely omits lang_start, meaning you as the developer must figure out how the pro- gram should be started, such as by declaring a function with #[export_name = "main"] that matches the expected launch sequence for the target platform. 

**NOTE** *On platforms that truly run no code before they jump to the defined start symbol, like most embedded devices, the initial values of static variables may not even match what’s specified in the source code. In such cases, your initialization function will need to explicitly initialize the various static memory segments with the initial data values specified in your program binary.* 

**The Out-of-Memory Handler** 

If you write a program that wishes to use alloc but is built for a platform that does not supply an allocator, you must dictate which allocator to use using the #[global_allocator] attribute mentioned earlier in the chapter. But you also have to specify what happens if that global allocator fails to allocate memory. Specifically, you need to define an *out-of-memory handler* to say what should happen if an infallible operation like Vec::push needs to allocate more memory, but the allocator cannot supply it. 

The default behavior of the out-of-memory handler on std-enabled platforms is to print an error message to standard error and then abort the process. However, on a platform that, for example, doesn’t have standard error, that obviously won’t work. At the time of writing, on such platforms your program must explicitly define an out-of-memory handler using the unstable attribute #[lang = "oom"]. Keep in mind that the handler should almost certainly prevent future execution, as otherwise the code that tried to allocate will continue executing without knowing that it did not receive the memory it asked for! 

**N O T E** *By the time you read this, the out-of-memory handler may already have been stabilized under a permanent name (**#[alloc_error_handler]**, most likely). Work is also under- way to give the default* *std* *out-of-memory handler the same kind of “hook” function- ality as Rust’s panic handler, so that code can change the out-of-memory behavior on the fly through a method like* *set_alloc_error_hook**.* 

### Low-Level Memory Accesses

In Chapter 10, we discussed the fact that the compiler is given a fair amount of leeway in how it turns your program statements into machine instructions, and that the CPU is allowed some wiggle room to execute instructions out of order. Normally, the shortcuts and optimizations that the compiler and CPU can take advantage of are invisible to the semantics of the program—you can’t generally tell whether, say, two reads have been reordered relative to each other or whether two reads from the same memory location actually result in two CPU load instructions. This is by design. The language and hardware designers carefully specified what semantics programmers commonly expect from their code when it runs so that your code generally does what you expect it to. 

However, no_std programming sometimes takes you beyond the usual border of “invisible optimizations.” In particular, you’ll often communicate with hardware devices through *memory mapping*, where the internal state of the device is made available in carefully chosen regions in memory. For example, while your computer starts up, the memory address range 0xA0000– 0xBFFFF maps to a crude graphics rendering pipeline; writes to individual bytes in that range will change particular pixels (or blocks, depending on the mode) on the screen. 

When you’re interacting with device-mapped memory, the device may implement custom behavior for each memory access to that region of memory, so the assumptions your CPU and compiler make about regular memory loads and stores may no longer hold. For instance, it is common for hardware devices to have memory-mapped registers that are modified when they’re read, meaning the reads have side effects. In such cases, the compiler can’t safely elide a memory store operation if you read the same memory address twice in a row! 

A similar issue arises when program execution is suddenly diverted in ways that aren’t represented in the code and thus that the compiler can- not expect. Execution might be diverted if there is no underlying operat- ing system to handle processor exceptions or interrupts, or if a process receives a signal that interrupts execution. In those cases, the execution of the active segment of code is stopped, and the CPU starts executing instructions in the event handler for whatever event triggered the diversion instead. Normally, since the compiler can anticipate all possible executions, it arranges its optimizations so that executions cannot observe when opera- tions have been performed out of order or optimized away. However, since the compiler can’t predict these exceptional jumps, it also cannot plan for them to be oblivious to its optimizations, so these event handlers might actually observe instructions that have run in a different order than those in the original program code. 

To deal with these exceptional situations, Rust provides *volatile* memory operations that cannot be elided or reordered with respect to other volatile operations. These operations take the form of std::ptr::read_volatile and std::ptr::write_volatile. Volatile operations are exactly the right fit for accessing memory-mapped hardware resources: they map directly to memory access operations with no compiler trickery, and the guarantee that volatile operations aren’t reordered relative to one another ensures that hardware operations with possible side effects don’t happen out of order even when they would normally look interchangeable (such as a load of one address and a store to a different address). The no-reordering guarantee also helps the exceptional execution situation, as long as any code that touches memory accessed in an exceptional context uses only volatile memory operations. 

**N O T E** *There is also a* *std::sync::atomic::compiler_fence* *function that prevents the compiler from reordering non-volatile memory accesses. You’ll very rarely need a compiler fence, but its documentation is an interesting read.* 

> **INCLUDING ASSEMBLY CODE** 

> These days, you rarely need to drop down to writing assembly code to accomplish any given task . But for low-level hardware programming where you need to initialize CPUs at boot or issue strange instructions to manipulate memory mappings, assembly code is still sometimes required . At the time of writing, there is an RFC and a mostly complete implementation of inline assembly syntax on nightly Rust, but nothing has been stabilized yet, so I won’t discuss the syntax in this book . 

> It’s still possible to write assembly on stable Rust—you just need to get a lit- tle creative . Specifically, remember build scripts from Chapter 11? Well, Cargo build scripts can emit certain special directives to standard output to augment Cargo’s standard build process, including cargo:rustc-link-lib=static=*xyz\* to link the static library file libxyz.a into the final binary, and cargo:rustc-link- search:*/some/path* to add /some/path to the search path for link objects . Using those, we can add a build.rs to the project that compiles a standalone assembly file (.s) to an object file (.o) using the target platform’s compiler and then repackages it into a static archive (.a) using the appropriate archiving tool (usually ar) . The project then emits those two Cargo directives, pointing at where it placed the static archive—probably in OUT_DIR—and we’re off to the races! If the target platform doesn’t change, you can even include the precompiled .a when publishing your crate so that consumers don’t need to rebuild it . 

### Misuse-Resistant Hardware Abstraction

Rust’s type system excels at encapsulating unsafe, hairy, and otherwise unpleasant code behind safe, ergonomic interfaces. Nowhere is that more important than in the infamously complex world of low-level systems programming, littered with magic hardware-defined values pulled from obscure manuals and mysterious undocumented assembly instruction incantations to get devices into just the right state. And all that in a space where a runtime error might crash more than just a user program! 

In no_std programs, it is immensely important to use the type system to make illegal states impossible to represent, as we discussed in Chapter 3. If certain combinations of register values cannot occur at the same time, then create a single type whose type parameters indicate the current state of the relevant registers, and implement only legal transitions on it, like we did for the rocket example in Listing 3-2. 

**N O T E** *Make sure to also review the advice from Chapter 3 on API design—all of that applies in the context of* *no_std* *programs as well!* 

For example, consider a pair of registers where at most one register should be “on” at any given point in time. Listing 12-2 shows how you can represent that in a (single-threaded) program in a way makes it impossible to write code that violates that invariant. 

```rust
// raw register address -- private submodule
mod registers;
pub struct On;
pub struct Off;
pub struct Pair<R1, R2>(PhantomData<(R1, R2)>);
impl Pair<Off, Off> {
	pub fn get() -> Option<Self> {
		static mut PAIR_TAKEN: bool = false;
		 if unsafe { PAIR_TAKEN } {
		 None 
	 } else { 
		  // Ensure initial state is correct.
			 registers::off("r1");
			 registers::off("r2");
			 unsafe { PAIR_TAKEN = true };
			 Some(Pair(PhantomData))
		}}
		 pub fn first_on(self) -> Pair<On, Off> {
			 registers::set_on("r1");
			 Pair(PhantomData)
			// .. and inverse for -> Pair<Off, On>
 }
 impl Pair<On, Off> {
	 pub fn off(self) -> Pair<Off, Off> {
		 registers::set_off("r1");
		 Pair(PhantomData)
	}
} 
// .. and inverse for Pair<Off, On>
```

*Listing 12-2: Statically ensuring correct operation*

There are a few noteworthy patterns in this code. The first is that we ensure only a single instance of Pair ever exists by checking a private static Boolean in its only constructor and making all methods consume self. We then ensure that the initial state is valid and that only valid state transitions are possible to express, and therefore the invariant must hold globally. 

The second noteworthy pattern in Listing 12-2 is that we use PhantomData to take advantage of zero-sized types and represent runtime information statically. That is, at any given point in the code the types tell us what the runtime state *must* be, and therefore we don’t need to track or check any state related to the registers at runtime. There’s no need to check that r2 isn’t already on when we’re asked to enable r1, since the types prevent writ- ing a program in which that is the case. 

### Cross-Compilation

Usually, you’ll write no_std programs on a computer with a full-fledged operating system running and all the niceties of modern hardware, but ultimately run it on a dinky hardware device with 93/4 bits of RAM and a sock for a CPU. That calls for *cross-compilation*—you need to compile the code in your development environment, but compile it *for* the sock. That’s not the only context in which cross-compilation is important, though. For example, it’s increasingly common to have one build pipeline produce binary artifacts for all consumer platforms rather than trying to have a build pipe- line for every platform your consumers may be using, and that means using cross-compilation. 

**NOTE** *If you’re actually compiling for something sock-like with limited memory, or even something as fancy as a potato, you may want to set the* *opt-level* *Cargo configuration to* *"s"* *to optimize for smaller binary sizes.* 

Cross-compiling involves two platforms: the *host* platform and the *target* platform. The host platform is the one doing the compiling, and the target platform is the one that will eventually run the output of the compilation. We specify platforms as *target triples*, which take the form *machine-vendor-os*. The *machine* part dictates the machine architecture the code will run on, such as x86_64, armv7, or wasm32, and tells the compiler what instruction set to use for the emitted machine code. The *vendor* part generally takes the value of pc on Windows, apple on macOS and iOS, and unknown everywhere else, and doesn’t affect compilation in any meaningful way; it’s mostly irrelevant and can even be left out. The *os* part tells the compiler what format to use for the final binary artifacts, so a value of linux dictates Linux *.so* files, win- dows dictates Windows *.dll* files, and so on. 

**NOTE** *By default, Cargo assumes that the target platform is the same as the host platform, which is why you generally never have to tell Cargo to, say, compile for Linux when you’re already on Linux. Sometimes you may want to use* *--target* *even if the CPU and OS of the target are the same, though, such as to target the* *musl* *implementation of* *libc**.* 

To tell Cargo to cross-compile, you simply pass it the --target <*target triple*> argument with your triple of choice. Cargo will then take care of forwarding that information to the Rust compiler so that it generates binary artifacts that will work on the given target platform. Cargo will also take care to use the appropriate version of the standard library for that platform—after all, the standard library contains a lot of conditional compilation directives (using #[cfg(...)]) so that the right system calls get invoked and the right architecture-specific implementations are used, so we can’t use the standard library for the host platform on the target. 

The target platform also dictates what components of the standard library are available. For example, while x86_64-unknown-linux-gnu includes the full std library, something like thumbv7m-none-eabi does no, and doesn’t even define an allocator, so if you use alloc without defining one explicitly, you’ll get a build error. This comes in handy for testing that code you write *actually* doesn’t require std (recall that even with #![no_std] you can still have use std::, since no_std opts out of only the std prelude). If you have your continuous integration pipeline build your crate with --target thumbv7m-none-eabi, any attempt to access components from anything but core will trigger a build failure. Crucially, this will also check that your crate doesn’t accidentally bring in dependencies that themselves use items from std (or alloc). 

> **PLATFORM SUPPORT** 

> The standard Rust installer, Rustup, doesn’t install the standard library for all the target triples that Rust supports by default . That would be a waste of space and bandwidth . Instead, you have to use the command rustup target add to install the appropriate standard library versions for additional targets . If no version of the standard library exists for your target platform, you’ll have to compile it from source yourself by adding the rust-src Rustup component and using Cargo’s (currently unstable) build-std feature to also build std (and/or core and alloc) when building any crate . 

> If your target is not supported by the Rust compiler—that is, if rustc doesn’t even know about your target triple—you’ll have to go one step further and teach rustc about the properties of the triple using a custom target specification . How you do that is both currently unstable and beyond the scope of this book, but a search for “custom target specification json” is a good place to start . 

## 总结

In this chapter, we’ve covered what lies beneath the standard library—or, more precisely, beneath std. We’ve gone over what you get with core, how you can extend your non-std reach with alloc, and what the (tiny) Rust runtime adds to your programs to make fn main work. We’ve also taken a look at how you can interact with device-mapped memory and otherwise handle the unorthodox execution patterns that can happen at the very lowest level of hardware programming, and how to safely encapsulate at least some of the oddities of hardware in the Rust type system. Next, we’ll move from the very small to the very large by discussing how to navigate, understand, and maybe even contribute to the larger Rust ecosystem. 


