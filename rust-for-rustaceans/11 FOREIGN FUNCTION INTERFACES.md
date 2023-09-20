
Not all code is written in Rust. It’s shocking, I know. Every so often, you’ll need to interact with code written in other languages, either by calling into such code from Rust or by allowing that code to call your Rust code. You can achieve this through *foreign function interfaces (FFI)*. 

In this chapter we’ll first look at the primary mechanism Rust provides for FFI: the extern keyword. We’ll see how to use extern both to expose Rust functions and statics to other languages and to give Rust access to functions and static variables provided from outside the Rust bubble. Then, we’ll walk through how to align Rust types with types defined in other languages and explore some of the intricacies of allowing data to flow across the FFI boundary. Finally, we’ll talk about some of the tools you’ll likely want to use if you’re doing any nontrivial amount of FFI. 

**N O T E** *While I often refer to FFI as being about crossing the boundary between one language and another, FFI can also occur entirely inside Rust-land. If one Rust program shares memory with another Rust program but the two aren’t compiled together—say, if you’re using a dynamically linked library in your Rust program that happens to be written in Rust, but you just have the C-compatible* .so *file—the same complications arise.* 

**Crossing Boundaries with extern** 

FFI is, ultimately, all about accessing bytes that originate somewhere out- side your application’s Rust code. For that, Rust provides two primary build- ing blocks: *symbols*, which are names assigned to particular addresses in a given segment of your binary that allow you to share memory (be it for data or code) between the external origin and your Rust code, and *calling conventions* that provide a common understanding of how to call functions stored in such shared memory. We’ll look at each of these in turn. 

**Symbols** 

Any binary artifact that the compiler produces from your code is filled with symbols—every function or static variable you define has a symbol that points to its location in the compiled binary. Generic functions may even have multiple symbols, one for each monomorphization of the function the compiler generates! 

Normally, you don’t have to think about symbols—they’re used internally by the compiler to pass around the final address of a function or static variable in your binary. This is how the compiler knows what location in memory each function call should target when it generates the final machine code, or where to read from if your code accesses a static variable. Since you don’t usually refer to symbols directly in your code, the compiler defaults to choosing semirandom names for them—you may have two functions called foo in different parts of your code, but the compiler will generate distinct symbols from them so that there’s no confusion. 

However, using random names for symbols won’t work when you want to call a function or access a static variable that isn’t compiled at the same time, such as code that’s written in a different language and thus compiled by a different compiler. You can’t tell Rust about a static variable defined in C if the symbol for that variable has a semirandom name that keeps changing. Conversely, you can’t tell Python’s FFI interface about a Rust function if you can’t produce a stable name for it. 

To use a symbol with an external origin, we also need some way to tell Rust about a variable or function in such a manner that the compiler will look for that same symbol defined elsewhere rather than defining its own (we’ll talk about how that search happens later). Otherwise, we would just end up with two identical symbols for that function or static variable, and no sharing would take place. In fact, in all likelihood, compilation would fail since any code that referred to that symbol wouldn’t know which definition (that is, which address) to use for it! 

**194** 

Chapter 11 

**NOTE** *A quick note about terminology: a symbol can be* declared *multiple times but* defined *only once. Every declaration of a symbol will link to the same single definition for that symbol at linking time. If no definition for a declaration is found, or if there are multiple definitions, the linker will complain.* 

**An Aside on Compilation and Linking** 

Compiler crash course time! Having a rough idea of the complicated process of turning code into a runnable binary will help you understand FFI better. You see, the compiler isn’t one monolithic program but is (typically) broken down into a handful of smaller programs that each perform distinct tasks and run one after the other. At a high level, there are three distinct phases to compilation—*compilation*, *code generation*, and *linking*—handled by three different components. 

The first phase is performed by what most people tend to think of as “the compiler”; it deals with type checking, borrow checking, monomorphization, and other features we associate with a given programming language. This phase generates no machine code but rather a low-level representation of the code that uses heavily annotated abstract machine operations. That low-level representation is then passed to the code generation tool, which is what produces machine code that can actually run on a given CPU. 

These two operations, taken together, do not have to be run in a single big pass over the whole codebase all at once. Instead, the codebase can be sliced into smaller chunks that are then run through compilation concur- rently. For example, Rust generally compiles different crates independently and in parallel as long as there isn’t a dependency between them. It can also invoke the code generation tool for independent crates separately to pro- cess them in parallel. Rust can often even compile multiple smaller slices of a single crate separately! 

Once the machine code for every piece of the application has been generated, those pieces can then be wired together. This is done in the linking phase by, unsurprisingly, the linker. The linker’s primary job is to take all the binary artifacts, called *object files*, produced by code generation, stitch them together into a single file, and then replace every reference to a symbol with the final memory address of that symbol. This is how you can define a function in one crate and call it from another but still compile the two crates separately. 

The linker is what enables FFI to work. It doesn’t care how each of the input object files were constructed; it just dutifully links together all the object files and then resolves any shared symbols. One object file may originally have been Rust code, one originally C code, and one may be a binary blob downloaded from the internet; as long as they all use the same symbol names, the linker will make sure that the resulting machine code uses the correct cross-referenced addresses for any shared symbols. 

A symbol can be linked either *statically* or *dynamically*. Static linking is the simplest, as each reference to a symbol is simply replaced with the address of that symbol’s definition. Dynamic linking, on the other hand, ties each reference to a symbol to a bit of generated code that tries to find the symbol’s definition when the program *runs*. We’ll talk more about these linking modes a little later. Rust generally defaults to static linking for Rust code, and dynamic linking for FFI. 

**Using extern** 

The extern keyword is the mechanism that allows us to declare a symbol as residing within a foreign interface. Specifically, it declares the existence of a symbol that’s defined elsewhere. In Listing 11-1 we define a static variable called RS_DEBUG in Rust that we make available to other code via FFI. We also declare a static variable called FOREIGN_DEBUG whose definition is unspecified but will be resolved at linking time. 

```
  #[no_mangle]
  pub static RS_DEBUG: bool = true;
  extern {
	  static FOREIGN_DEBUG: bool;
  }
```

Listing 11-1: Exposing a Rust static variable, and accessing one declared elsewhere, through FFI 

The #[no_mangle] attribute ensures that RS_DEBUG retains that name dur- ing compilation rather than having the compiler assign it another symbol name to, for example, distinguish it from another (non-FFI) RS_DEBUG static variable elsewhere in the program. The variable is also declared as pub since it’s a part of the crate’s public API, though that annotation isn’t strictly necessary on items marked #[no_mangle]. Note that we don’t use extern for RS_DEBUG, since it’s defined here. It will still be accessible to link against from other languages. 

The extern block surrounding the FOREIGN_DEBUG static variable denotes that this declaration refers to a location that Rust will learn at linking time based on where the definition of the same symbol is located. Since it’s defined elsewhere, we don’t give it an initialization value, just a type, which should match the type used at the definition site. Because Rust doesn’t know anything about the code that defines the static variable, and thus can’t check that you’ve declared the correct type for the symbol, FOREIGN_DEBUG can be accessed only inside an unsafe block. 

**NOTE** *Static variables in Rust aren’t mutable by default, regardless of whether they’re in an* *extern* *block. These variables are always available from any thread, so mutable access would pose a data race risk. You can declare a* *static* *as* *mut**, but if you do, it becomes unsafe to access.* 

The procedure to declare FFI functions is very similar. In Listing 11-2, we make hello_rust accessible to non-Rust code and pull in the external hello_foreign function. 

**196** Chapter 11 

```
#[no_mangle]
pub extern fn hello_rust(i: i32) { ... }
extern {
    fn hello_foreign(i: i32);
}
```

Listing 11-2: Exposing a Rust function, and accessing one defined elsewhere, through FFI 

The building blocks are all the same as in Listing 11-1 with the exception that the Rust function is declared using extern fn, which we’ll explore in the next section. 

If there are multiple definitions of a given extern symbol like FOREIGN_ DEBUG or hello_foreign, you can explicitly specify which library the symbol should link against using the #[link] attribute. If you don’t, the linker will give you an error saying that it’s found multiple definitions for the symbol in question. For example, if you prefix an extern block with #[link(name = "crypto")], you’re telling the linker to resolve any symbols (whether statics or functions) against a linked library named “crypto.” You can also rename an external static or function in your Rust code by annotating its declara- tion with #[link_name = "**"], and then the item links to whatever name you wish. Similarly, you can rename a Rust item for export using #[export_name = "**"]. 

**Link Kinds** 

\#[link] also accepts the argument kind, which dictates how the items in the block should be linked. The argument defaults to "dylib", which signifies C-compatible dynamic linking. The alternative kind value is "static", which indicates that the items in the block should be linked fully at compile time (that is, statically). This essentially means that the external code is wired directly into the binary produced by the compiler, and thus doesn’t need to exist at runtime. There are a few other kinds as well, but they are much less common and outside the scope of this book. 

There are several trade-offs between static and dynamic linking, but the main considerations are security, binary size, and distribution. First, dynamic linking tends to be more secure because it makes it easier to upgrade libraries independently. Dynamic linking allows whoever deploys a binary that contains your code to upgrade libraries your code links against without having to recompile your code. If, say, libcrypto gets a security update, the user can update the crypto library on the host and restart the binary, and the updated library code will be used automatically. With static compilation, the library’s code is hardwired into the binary, so the user would have to recompile your code against an upgraded version of the library to get the update. 

Dynamic linking also tends to produce smaller binaries. Since static compilation includes any linked code into the final binary output, and any code that code in turn pulls in, it produces larger binaries. With dynamic linking, each external item includes just a small bit of wrapper code that loads the indicated library at runtime and then forwards the access. 

Foreign Function Interfaces **197** 

So far, static linking may not seem very attractive, but it has one big advantage over dynamic linking: ease of distribution. With dynamic linking, anyone who wants to run a binary that includes your code must *also* have any libraries your code links against. Not only that, but they must make sure the version of each such library they have is compatible with what your code expects. This may be fine for libraries like glibc or OpenSSL that are available on most systems, but it poses a problem for more obscure libraries. The user then needs to be aware that they should install that library and must hunt for it in order to run your code! With static linking, the library’s code is embedded directly into the binary output, so the user doesn’t need to install it themselves. 

Ultimately, there isn’t a *right* choice between static and dynamic linking. Dynamic linking is usually a good default, but static compilation may be a better option for particularly constrained deployment environments or for very small or niche library dependencies. Use your best judgment! 

**Calling Conventions** 

Symbols dictate *where* a given function or variable is defined, but that’s not enough to allow function calls across FFI boundaries. To call a foreign function in any language, the compiler also needs to know its *calling convention*, which dictates the assembly code to use to invoke the function. We won’t get into the actual technical details of each calling convention here, but as a general overview, the convention dictates: 

- How the stack frame for the call is set up 

- How arguments are passed (whether on the stack or in registers, in order or in reverse) 

- How the function is told where to jump back to when it returns 

- How various CPU states, like registers, are restored in the caller after the function completes 

  Rust has its own unique calling convention that isn’t standardized and is allowed to be changed by the compiler over time. This works fine as long as all function definitions and calls are compiled by the same Rust compiler, but it is problematic if you want interoperability with external code because that external code doesn’t know about the Rust calling convention. 

  Every Rust function is implicitly declared with extern "Rust" if you don’t declare anything else. Using extern on its own, as in Listing 11-2, is shorthand for extern "C", which means “use the standard C calling convention.” The shorthand is there because the C calling convention is what you want in nearly every case of FFI. 

**NOTE** *Unwinding generally works only with regular Rust functions. If you unwind across the end of a Rust function that isn’t* *extern "Rust"**, your program will abort. Unwinding across the FFI boundary into external code is undefined behavior. With RFC 2945, Rust gained a new* *extern* *declaration,* *extern "C-unwind"**; this permits unwinding across FFI boundaries in particular situations, but if you wish to use it you should read the RFC carefully.* 


Rust also supports a number of other calling conventions that you supply as a string following the extern keyword (in both fn and block context). For example, extern "system" says to use the calling convention of the operating system’s standard library interface, which at the time of writing is the same as "C" everywhere except on Win32, which uses the "stdcall" calling convention. In general, you’ll rarely need to supply a calling convention explicitly unless you’re working with particularly platform-specific or highly optimized external interfaces, so just extern (which is extern "C") will be fine. 


**N O T E** *A function’s calling convention is part of its type. That is, the type* *extern "C" fn()* *is not the same as* *fn()* *(or* *extern "Rust" fn()**), which is different again from* *extern "system" fn()**.* 

**OTHER BINARY ARTIFACTS** 

Normally, you compile Rust code only to run its tests or build a binary that you’re then going to distribute or run . Unlike in many other languages, you don’t generally compile a Rust library to distribute it to others—if you run a command like cargo publish, it just wraps up your crate’s source code and uploads it to crates.io . This is mostly because it is difficult to distribute generic code as anything but source code . Since the compiler monomorphizes each generic function to the provided type arguments, and those types may be defined in the caller’s crate, the compiler must have access to the function’s generic form, which means no optimized machine code! 

Technically speaking, Rust does compile binary library artifacts, called rlibs, of each dependency that it combines in the end . These rlibs include the informa- tion necessary to resolve generic types, but they are specific to the exact com- piler used and can’t generally be distributed in any meaningful way . 

So what do you do if you want to write a library in Rust that you then want to interface with from another programming language? The solution is to produce C-compatible library files in the form of dynamically linked libraries (.so files on Unix, .dylib files on macOS, and .dll files on Windows) and stati- cally linked libraries (.a files on Unix/macOS and .lib files on Windows) . Those files look like files produced by C code, so they can also be used by other lan- guages that know how to interact with C . 

To produce these C-compatible binary artifacts, you set the crate-type field of the [lib] section of your Cargo.toml file . The field takes an array of val- ues, which would normally just be "lib" to indicate a standard Rust library (an rlib) . Cargo applies some heuristics that will set this value automatically if your crate is clearly not a library (for example, if it's a procedural macro), but best practice is to set this value explicitly if you’re producing anything but a good ol’ Rust library . 

There are a number of different crate types, but the relevant ones here are "cdylib" and "staticlib", which produce C-compatible library files that are dynamically and statically linked, respectively . Keep in mind that when you produce one of these artifact types, only publicly available symbols are available—that is, public and #[no_mangle] static variables and functions . Things like types and constants won’t be available, even if they’re marked pub, since they have no meaningful representation in a binary library file . 

**Types Across Language Boundaries** 

With FFI, type layout is crucial; if one language lays out the memory for some shared data one way but the language on the other side of the FFI boundary expects it to be laid out differently, then the two sides will inter- pret the data inconsistently. In this section, we’ll look at how to make types match up over FFI, and other aspects of types to be aware of when you cross the boundaries between languages. 

**Type Matching** 

Types aren’t shared across the FFI boundary. When you declare a type in Rust, that type information is lost entirely upon compilation. All that’s communicated to the other side is the bits that make up values of that type. You therefore need to declare the type for those bits on both sides of the boundary. When you declare the Rust version of the type, you first must make sure the primitives contained within the type match up. For example, if C is used on the other side of the boundary, and the C type uses an int, the Rust code had better use the exact Rust equivalent: an i32. To take some of the guesswork out of that process, for interfaces that use C-like types the Rust standard library provides you with the correct C types in the std::os::raw module, which defines type c_int = i32, type c_char = i8/u8 depending on whether char is signed, type c_long = i32/i64 depending on the target pointer width, and so on. 

*Take particular note of quirky integer types in C like* *__be32**. These often do not trans- late directly to Rust types and may be best left as something like* *[u8; 4]**. For example,* *__be32* *is always encoded as big-endian, whereas Rust’s* *i32* *uses the endianness of the current platform.* 

With more complex types like vectors and strings, you usually need to do the mapping manually. For example, since C tends to represent a string as a sequence of bytes terminated with a 0 byte, rather than a UTF-8–encoded string with the length stored separately, you cannot generally use Rust’s string types over FFI. Instead, assuming the other side uses a C-style string representation, you should use the std::ffi::CStr and std::ffi::CString types for borrowed and owned strings, respectively. For vectors, you’ll likely want to use a raw pointer to the first element and then pass the length separately—the Vec::into_raw_parts method may come in handy for that. 

**N O T E** 

For types that contain other types, such as structs and unions, you also need to deal with layout and alignment. As we discussed in Chapter 2, Rust lays out types in an undefined way by default, so at the very least you will want to use #[repr(C)] to ensure that the type has a deterministic layout and alignment that mirrors what’s (likely and hopefully) used across the FFI boundary. If the interface also specifies other configurations for the type, such as manually setting its alignment or removing padding, you’ll need to adjust your #[repr] accordingly. 

A Rust enum has multiple possible C-style representations depending on whether the enum contains data or not. Consider an enum without data, like this: 

```
enum Foo { Bar, Baz }
```

With #[repr(C)], the type Foo is encoded using just a single integer of the same size that a C compiler would choose for an enum with the same number of variants. The first variant has the value 0, the second the value 1, and so on. You can also manually assign values to each variant, as shown in Listing 11-3. 

```
#[repr(C)]
enum Foo {
	Bar = 1, 
	Baz = 2, } 
```

Listing 11-3: Defining explicit variant values for a dataless enum 

*Technically, the specification says that the first variant’s value is* *0* *and every subse- quent variant’s value is one greater than that of the previous one. This makes a dif- ference if you manually set the value for some variants but not others—those you do not set will continue from the last one you did set.* 

You should be careful about mapping enum-like types in C to Rust this way, however, as only the values for defined variants are valid for an instance of the enum type. This tends to get you into trouble with C-style enumerations that often function more like bitsets, where variants can be bitwise ORed together to produce a value that encapsulates multiple variants at once. In the example from Listing 11-3, for instance, a value of 3 produced by taking Bar | Baz would not be valid for Foo in Rust! If you need to model a C API that uses an enumeration for a set of bitflags that can be set and unset individually, consider using a newtype wrapper around an inte- ger type, with associated constants for each variant and implementations of the various Bit* traits for improved ergonomics. Or use the bitflags crate. 

*For fieldless enums, you can also pass a numeric type to* *#[repr]* *to use a different type than* *isize* *for the discriminator. For example,* *#[repr(u8)]* *will encode the dis- criminator using a single unsigned byte. For a data-carrying enum, you can pass* *#[repr(C, u8)]* *to get the same effect.* 

**N O T E** 

Foreign Function Interfaces **201** 

On an enum that contains data, the #[repr(C)] attribute causes the enum to be represented using a *tagged union*. That is, it is represented in memory by a #[repr(C)] struct with two fields, where the first is the discriminator as it would be encoded if none of the variants had fields, and the second is a union of the data structures for each variant. For a concrete example, con- sider the enum and associated representation in Listing 11-4. 

```
#[repr(C)]
enum Foo {
	Bar(i32), 
    Baz { a: bool, b: f64 }
}
// is represented as
#[repr(C)]
enum FooTag { Bar, Baz }
#[repr(C)]
struct FooBar(i32);
#[repr(C)]
struct FooBaz{ a: bool, b: f64 }
#[repr(C)]
union FooData {
	bar: FooBar, 
	baz: FooBaz,
}
#[repr(C)]
struct Foo {
	tag: FooTag, 
	data: FooData
}
```

Listing 11-4: Rust enums with *#[repr(C)]* are represented as tagged unions. 

**THE NICHE OPTIMIZATION IN FFI** 

In Chapter 9 we talked about the niche optimization, where the Rust compiler uses invalid bit patterns to represent enum variants that hold no data . The fact that this optimization is guaranteed leads to an interesting interaction with FFI . Specifically, it means that nullable pointers can always be represented in FFI types using an Option-wrapped pointer type . For example, a nullable function pointer can be represented as Option<extern fn(...)>, and a nullable data pointer can be represented as Option<*mut T> . These will transparently do the right thing if an all-zero bit pattern value is provided, and will represent it as None in Rust . 

**Allocations** 

When you allocate memory, that allocation belongs to its allocator and can be freed only by that same allocator. This is the case if you use multiple allocators within Rust and also if you are allocating memory both in Rust and with some allocator on the other side of the FFI boundary. You’re free to send pointers across the boundary and access that memory to your heart’s content, but when it comes to releasing the memory again, it needs to be returned to the appropriate allocator. 

Most FFI interfaces will have one of two configurations for handling allocation: either the caller provides data pointers to chunks of memory or the interface exposes dedicated freeing methods to which any allocated resources should be returned when they are no longer needed. Listing 11-5 shows an example of Rust declarations of some signatures from the OpenSSL library that use implementation-managed memory. 

```
// One function allocates memory for a new object.
extern fn ECDSA_SIG_new() -> *mut ECDSA_SIG;
// And another accepts a pointer created by new
// and deallocates it when the caller is done with it.
extern fn ECDSA_SIG_free(sig: *mut ECDSA_SIG);
```

Listing 11-5: An implementation-managed memory interface 

The functions ECDSA_SIG_new and ECDSA_SIG_free form a pair, where the caller is expected to call the new function, use the returned pointer for as long as it needs (likely by passing it to other functions in turn), and then finally pass the pointer to the free function once it’s done with the referenced resource. Presumably, the implementation allocates memory in the new function and deallocates it in the free function. If these functions were defined in Rust, the new function would likely use Box::new, and the free function would invoke Box::from_raw and then drop the value to run its destructor. 

Listing 11-6 shows an example of caller-managed memory. 

```
// An example of caller-managed memory.
// The caller provides a pointer to a chunk of memory,
// which the implementation then uses to instantiate its own types.
// No free function is provided, as that happens in the caller.
extern fn BIO_new_mem_buf(buf: *const c_void, len: c_int) -> *mut BIO
```

Listing 11-6: A caller-managed memory interface 

Here, the BIO_new_mem_buf function instead has the caller supply the backing memory. The caller can choose to allocate memory on the heap, or use whatever other mechanism it deems fit for obtaining the required memory, and then passes it to the library. The onus is then on the caller to ensure that the memory is later deallocated, but only once it is no longer needed by the FFI implementation! 

You can use either of these approaches in your FFI APIs or even mix and match them if you wish. As a general rule of thumb, allow the caller to pass in memory when doing so is feasible, since it gives the caller more freedom to manage memory as it deems appropriate. For example, the caller may be using a highly specialized allocator on some custom operating system, and may not want to be forced to use the standard allocator your implementation would use. If the caller can pass in the memory, it might even avoid allocations entirely if it can instead use stack memory or reuse already allocated memory. However, keep in mind that the ergonomics of a caller-managed interface are often more convoluted, since the caller must now do all the work to figure out how much memory to allocate and then set that up before it can call into your library. 

In some instances, it may even be impossible for the caller to know ahead of time how much memory to allocate—for example, if your library’s types are opaque (and thus not known to the caller) or can change over time, the caller won’t be able to predict the size of the allocation. Similarly, if your code has to allocate more memory while it is running, such as if you’re building a graph on the fly, the amount of memory needed may vary dynamically at runtime. In such cases, you will have to use implementation- managed memory. 

When you’re forced to make a trade-off, go with caller-allocated memory for anything that is either *large* or *frequent*. In those cases the caller is likely to care the most about controlling the allocations itself. For anything else, it’s probably okay for your code to allocate and then expose destructor functions for each relevant type. 

**Callbacks** 

You can pass function pointers across the FFI boundary and call the referenced function through those pointers as long as the function pointer’s type has an extern annotation that matches the function’s calling convention. That is, you can define an extern "C" fn(c_int) -> c_int in Rust and then pass a reference to that function to C code as a callback that the C code will eventually invoke. 

You do need to be careful using callbacks around panics, as having a panic unwind past the end of a function that is anything but extern "Rust" is undefined behavior. The Rust compiler will currently automatically abort if it detects such a panic, but that may not always be the behavior you want. Instead, you may want to use std::panic::catch_unwind to detect the panic in any function marked extern, and then translate the panic into an error that is FFI-compatible. 

**Safety** 

When you write Rust FFI bindings, most of the code that actually interfaces with the FFI will be unsafe and will mainly revolve around raw pointers. However, your goal should be to ultimately present a *safe* Rust interface on top of the FFI. Doing so mainly comes down to reading carefully through the invariants of the unsafe interface you are wrapping and then ensuring you uphold them all through the Rust type system in the safe interface. The three most important elements of safely encapsulating a foreign interface are capturing & versus &mut accurately, implementing Send and Sync appropriately, and ensuring that pointers cannot be accidentally confused. I’ll go over how to enforce each of these next. 

**References and Lifetimes** 

If there’s a chance external code will modify data behind a given pointer, make sure that the safe Rust interface has an exclusive reference to the relevant data by taking &mut. Otherwise a user of your safe wrapper might accidentally read from memory that the external code is simultaneously modifying, and all hell will break loose! 

You’ll also want to make good use of Rust lifetimes to ensure that all pointers live for as long as the FFI requires. For example, imagine an external interface that lets you create a Context and then lets you create a Device from that Context with the requirement that the Context remain valid for as long as the Device lives. In that case, any safe wrapper for the interface should enforce that requirement in the type system by having Device hold a lifetime associ- ated with the borrow of Context that the Device was created from. 

**Send and Sync** 

Do not implement Send and Sync for types from an external library unless that library explicitly documents that those types are thread-safe! It is the safe Rust wrapper’s job to ensure that safe Rust code *cannot* violate the invariants of the external code and thus trigger undefined behavior. 

Sometimes, you may even want to introduce dummy types to enforce external invariants. For example, say you have an event loop library with the interface given in Listing 11-7. 

```
extern fn start_main_loop();
extern fn next_event() -> *mut Event;
```

Listing 11-7: A library that expects single-threaded use 

Now suppose that the documentation for the external library states that next_event may be called only by the same thread that called start_main_loop. However, here we have no type that we can avoid implementing Send for! Instead, we can take a page out of Chapter 3 and introduce additional marker state to enforce the invariant, as shown in Listing 11-8. 

```
pub struct EventLoop(std::marker::PhantomData<*const ()>);
pub fn start() -> EventLoop {
    unsafe { ffi::start_main_loop() };
    EventLoop(std::marker::PhantomData)
}
impl EventLoop {
    pub fn next_event(&self) -> Option<Event> {
        let e = unsafe { ffi::next_event() };
	// ... 
}} 
```

Listing 11-8: Enforcing an FFI invariant by introducing auxiliary types 

The empty type EventLoop doesn’t actually connect with anything in the underlying external interface but rather enforces the contract that you call next_event only after calling start_main_loop, and only on the same thread. You enforce the “same thread” part by making EventLoop neither Send nor Sync, by having it hold a phantom raw pointer (which itself is neither Send nor Sync). 

Using PhantomData<*const ()> to “undo” the Send and Sync auto-traits as we do here is a bit ugly and indirect. Rust does have an unstable compiler feature that enables negative trait implementations like impl !Send for EventLoop {}, but it’s surprisingly difficult to get its implementation right, and it likely won’t stabilize for some time. 

You may have noticed that nothing prevents the caller from invoking start_main_loop multiple times, either from the same thread or from another thread. How you’d handle that would depend on the semantics of the library in question, so I’ll leave it to you as an exercise. 

**Pointer Confusion** 

In many FFI APIs, you don’t necessarily want the caller to know the internal representation for each and every chunk of memory you give it pointers to. The type might have internal state that the caller shouldn’t fiddle with, or the state might be difficult to express in a cross-language-compatible way. For these kinds of situations, C-style APIs usually expose *void pointers*, writ- ten out as the C type void*, which is equivalent to *mut std::ffi::c_void in Rust. A type-erased pointer like this is, effectively, *just* a pointer, and does not convey anything about the thing it points to. For that reason, these kinds of pointers are often referred to as *opaque*. 

Opaque pointers effectively serve the role of visibility modifiers for types across FFI boundaries—since the method signature does not say what’s being pointed to, the caller has no option but to pass around the pointer as is and use any available FFI methods to provide visibility into the referenced data. Unfortunately, since one *mut c_void is indistinguishable from another, there’s nothing stopping a user from taking an opaque pointer as is returned from one FFI method and supplying it to a method that expects a pointer to a *different* opaque type. 

We can do better than this in Rust. To mitigate this kind of pointer type confusion, we can avoid using *mut c_void directly for opaque pointers in FFI, even if the actual interface calls for a void*, and instead con- struct different empty types for each distinct opaque type. For example, in Listing 11-9 I use two distinct opaque pointer types that cannot be confused. 

```
#[non_exhaustive] #[repr(transparent)] pub struct Foo(c_void);
#[non_exhaustive] #[repr(transparent)] pub struct Bar(c_void);
extern {
    pub fn foo() -> *mut Foo;
    pub fn take_foo(arg: *mut Foo);
    pub fn take_bar(arg: *mut Bar);
} 
```

Listing 11-9: Opaque pointer types that cannot be confused 

Since Foo and Bar are both zero-sized types, they can be used in place of () in the extern method signatures. Even better, since they are now distinct types, Rust won’t let you use one where the other is required, so it’s now impossible to call take_bar with a pointer you got back from foo. Adding the #[non_exhaustive] annotation ensures that the Foo and Bar types cannot be constructed outside of this crate. 

**bindgen and Build Scripts** 

Mapping out the Rust types and externs for a larger external library can be quite a chore. Big libraries tend to have a large enough number of type and method signatures to match up that writing out all the Rust equivalents is time-consuming. They also have enough corner cases and C oddities that some patterns are bound to require more careful thought to translate. 

Luckily, the Rust community has developed a tool called bindgen that significantly simplifies this process as long as you have C header files avail- able for the library you want to interface with. bindgen essentially encodes all the rules and best practices we’ve discussed in this chapter, plus a number of others, and wraps them up in a configurable code generator that takes in C header files and spits out appropriate Rust equivalents. 

bindgen provides a stand-alone binary that generates the Rust code for C headers once, which is convenient when you want to check in the bindings. This process allows you to hand-tune the generated bindings, should that be necessary. If, on the other hand, you want to generate the bindings automatically on every build and just include the C header files in your source code, bindgen also ships as a library that you can invoke in a custom *build script* for your package. 

**NOTE** *If you check in the bindings directly, keep in mind that they will be correct only on the platform they were generated for. Generating the bindings in a build script will generate them specifically for the current target platform, which is less likely to cause platform-related layout inconsistencies.* 

You declare a build script by adding build = "**" to the [package] section of your *Cargo.toml*. This tells Cargo that, before compiling your crate, it should compile ** as a stand-alone Rust program and run it; only then should it compile the source code of your crate. The build script also gets its own dependencies, which you declare in the [build- dependencies] section of your *Cargo.toml*. 

**NOTE** *If you name your build script* build.rs*, you don’t need to declare it in your* Cargo.toml*.* 

Build scripts come in very handy with FFI—they can compile a bundled C library from source, dynamically discover and declare additional build flags to be passed to the compiler, declare additional files that Cargo should check for changes for the purposes of recompilation, and, you guessed it, generate additional source files on the fly! 

Though build scripts are very versatile, beware of making them too aware of the environment they run in. While you can use a build script to detect if the Rust compiler version is a prime or if it’s going to rain in Istanbul tomorrow, making your compilation dependent on such conditions may make builds fail unexpectedly for other developers, which leads to a poor development experience. 

The build script can write files to a special directory supplied through the OUT_DIR environment variable. The same directory and environment variable are also accessible in the Rust source code at compile time so that it can pick up files generated by the build script. To generate and use Rust types from a C header, you first have your build script use the library ver- sion of bindgen to read in a *.h* file and turn it into a file called, say, *bindings.rs* inside OUT_DIR. You then add the following line to any Rust file in your crate to include *bindings.rs* at compilation time: 

```
include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

Since the code in *bindings.rs* is autogenerated, it’s generally best prac- tice to place the bindings in their own crate and give the crate the same name as the library the bindings are for, with the suffix -sys (for example, openssl-sys). If you don’t follow this practice, releasing new versions of your library will be much more painful, as it is illegal for two crates that link against the same external library through the links key in *Cargo.toml* to coexist in a given build. You would essentially have to upgrade the entire ecosystem to the new major version of your library all at once. Separating just the bindings into their own crate allows you to issue new major versions of the wrapper crate that can be adopted incrementally. The separation also allows you to cut a breaking release of the crate with those bindings if the Rust bindings change—say, if the header files themselves are upgraded or a bindgen upgrade causes the generated Rust code to change slightly— without *also* having to cut a breaking release of the crate that safely wraps the FFI bindings. 

**N O T E** *Remember that if you include any of the types from the* *-sys* *crate in the public inter- face of your main library crate, changing the dependency on the* *-sys* *crate to a new major version still constitutes a breaking change for your main library!* 

If your crate instead produces a library file that you intend others to use through FFI, you should also publish a C header file for its interface to make it easier to generate native bindings to your library from other languages. However, that C header file then needs to be kept up to date as your crate changes, which can become cumbersome as your library grows in size. Fortunately, the Rust community has also developed a tool to automate this task: cbindgen. Like bindgen, cbindgen is a build tool, and it also comes as both a binary and a library for use in build scripts. Instead of taking in a C header file and producing Rust, it takes Rust in and produces a C header file. Since the C header file represents the main computer-readable 

description of your crate’s FFI, I recommend manually looking it over to make sure the autogenerated C code isn’t too unwieldy, though in general cbindgen tends to produce fairly reasonable code. If it doesn’t, file a bug! 

>**C++** 

>I’ve mainly focused on C in this chapter as it’s the language most commonly used to describe cross-language interfaces for libraries you can link against . Nearly every programming language provides some way to interact with C libraries, since they are so ubiquitous . While C++ feels closely related to C, and many high-profile libraries are written in C++, it’s a very different beast when it comes to FFI . Generating types and signatures to match a C header is relatively straightforward, but that is not at all the case for C++ . At the time of writing, bindgen has decent support for generating bindings to C++, but they are often lacking in ergonomics . For example, you generally have to manually call constructors, destructors, overloaded operators, and the like . Some C++ features like template specialization also aren’t supported at all . If you do have to interface with C++, I recommend you give the cxx crate a try . 

## Summary

In this chapter, we’ve covered how to use the extern keyword to call out of Rust into external code, as well as how to use it to make Rust code accessible to external code. We’ve also discussed how to align Rust types with types on the other side of the FFI boundary, and some of the common pitfalls in trying to get code written in two different languages to mesh well. Finally, we talked about the bindgen and cbindgen tools, which make the experience of keeping FFI bindings up to date much more pleasant. In the next chapter, we’ll look at how to use Rust in more restricted environments, like embedded devices, where the standard library may not be available and where even a simple operation like allocating memory may not be possible. 


**RUST WITHOUT THE STANDARD LIBRARY** 

Rust is intended to be a language for systems programming, but it isn’t always clear what that really means. At the very least, a systems programming language is usually expected to allow the programmer to write programs that do not rely on the operating system and can run directly on the hardware, whether that is a thousand core supercomputer or an embedded device with a single-core ARM processor with a clock speed of 72MHz and 256KiB of memory. 

In this chapter, we’ll take a look at how you can use Rust in unorthodox environments, such as those without an operating system, or those that don’t even have the ability to dynamically allocate memory! Much of our discussion will focus on the #![no_std] attribute, but we’ll also investigate 