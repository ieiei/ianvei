
This chapter provides some ideas for structuring your Rust projects. For simple projects, the structure set up by cargo new is likely to be something you think little about. You may add some modules to split up the code, and some dependencies for additional functionality, but that’s about it. However, as a project grows in size and complexity, you’ll find that you need to go beyond that. Maybe the compilation time for your crate is getting out of hand, or you need conditional dependencies, or you need a better strategy for continuous integration. In this chapter, we will look at some of the tools that the Rust language, and Cargo in particular, provide that make it easier to manage such things. 

**Features** 

*Features* are Rust’s primary tool for customizing projects. At its core, a feature is just a build flag that crates can pass to their dependencies in order to add optional functionality. Features carry no semantic meaning in and of themselves—instead, *you* choose what a feature means for *your* crate. 

Generally, we use features in three ways: to enable optional dependencies, to conditionally include additional components of a crate, and to augment the behavior of the code. Note that all of these uses are *additive*; features can add to the functionality of the crate, but they shouldn’t generally do things like remove modules or replace types or function signatures. This stems from the principle that if a developer makes a simple change to their *Cargo.toml*, such as adding a new dependency or enabling a feature, that shouldn’t make their crate stop compiling. If a crate has mutually exclusive features, that principle quickly falls by the wayside—if crate A depends on one feature of crate C, and crate B on another mutually exclusive feature of C, adding a dependency on crate B would then break crate A! For that reason, we generally follow the principle that if crate A compiles against crate C with some set of features, it should also compile if all features are enabled on crate C. 

Cargo leans into this principle quite hard. For example, if two crates (A and B) both depend on crate C, but they each enable different features on C, Cargo will compile crate C only once, with *all* the features that either A or B requires. That is, it’ll take the union of the requested features for C across A and B. Because of this, it’s generally hard to add mutually exclusive features to Rust crates; chances are that some two dependents will depend on the crate with different features, and if those features are mutually exclusive, the downstream crate will fail to build. 

*I highly recommend that you configure your continuous integration infrastructure to check that your crate compiles for any combination of its features. One tool that helps you do this is* *cargo-hack**, which you can find at* https://github.com/taiki-e/cargo-hack/. 

**Defining and Including Features** 

Features are defined in *Cargo.toml*. Listing 5-1 shows an example of a crate named foo with a simple feature that enables the optional dependency syn. 

```toml
[package]
name = "foo"
...
[features]
derive = ["syn"]
[dependencies]
syn = { version = "1", optional = true }
```

*Listing 5-1: A feature that enables an optional dependency*

When Cargo compiles this crate, it will not compile the syn crate by default, which reduces compile time (often significantly). The syn crate will be compiled only if a downstream crate needs to use the APIs enabled by the derive feature and explicitly opts in to it. Listing 5-2 shows how such a downstream crate bar would enable the derive feature, and thus include the syn dependency. 

```toml
[package]
name = "bar"
...
[dependencies]
foo = { version = "1", features = ["derive"] }
```

*Listing 5-2: Enabling a feature of a dependency*

Some features are used so frequently that it makes more sense to have a crate opt out of them rather than in to them. To support this, Cargo allows you to define a set of default features for a crate. And similarly, it allows you to opt out of the default features of a dependency. Listing 5-3 shows how foo can make its derive feature enabled by default, while also opting out of some of syn’s default features and instead enabling only the ones it needs for the derive feature. 

```
[package]
name = "foo"
...
[features]
derive = ["syn"]
default = ["derive"]
[dependencies.syn]
version = "1"
default-features = false
features = ["derive", "parsing", "printing"]
optional = true
```

*Listing 5-3: Adding and opting out of default features, and thus optional dependencies*

Here, if a crate depends on foo and does not explicitly opt out of the default features, it will also compile foo’s syn dependency. In turn, syn will be built with only the three listed features, and no others. Opting out of default features this way, and opting in to only what you need, is a great way to cut down on your compile times! 

**OPTIONAL DEPENDENCIES AS FEATURES** 

When you define a feature, the list that follows the equal sign is itself a list of features . This might, at first, sound a little odd—in Listing 5-3, syn is a dependency, not a feature . It turns out that Cargo makes every optional dependency a feature with the same name as the dependency . You’ll see this if you try to add a feature with the same name as an optional dependency; Cargo won’t allow it . Support for a different namespace for features and dependencies is in the works in Cargo, but has not been stabilized at the time of writing . In the meantime, if you want to have a feature named after a dependency, you can rename the dependency using package = "" to avoid the name collision . The list of features that a feature enables can also include features of dependencies . For example, you can write derive = ["syn/derive"] to have your derive feature enable the derive feature of the syn dependency . 

Project Structure **69** 

**Using Features in Your Crate** 

When using features, you need to make sure your code uses a dependency only if it is available. And if your feature enables a particular component, you need to make sure that if the feature isn’t enabled, the component is not included. 

You achieve this using *conditional compilation*, which lets you use annotations to give conditions under which a particular piece of code should or should not be compiled. Conditional compilation is primarily expressed using the #[cfg] attribute. There is also the closely related cfg! macro, which lets you change runtime behavior based on similar conditions. You can do all sorts of neat things with conditional compilation, as we’ll see later in this chapter, but the most basic form is #[cfg(feature = "some-feature")], which makes it so that the next “thing” in the source code is compiled only if the some-feature feature is enabled. Similarly, if cfg!(feature = "some-feature") is equivalent to if true only if the derive feature is enabled (and if false other wise). 

The #[cfg] attribute is used more often than the cfg! macro, because the macro modifies runtime behavior based on the feature, which can make it difficult to ensure that features are additive. You can place #[cfg] in front of certain Rust *items*—such as functions and type definitions, impl blocks, modules, and use statements—as well as on certain other constructs like struct fields, function arguments, and statements. The #[cfg] attribute can’t go just anywhere, though; where it can appear is carefully restricted by the Rust language team so that conditional compilation can’t cause situations that are too strange and hard to debug. 

Remember that modifying certain public parts of your API may inadvertently make a feature nonadditive, which in turn may make it impossible for some users to compile your crate. You can often use the rules for backward compatible changes as a rule of thumb here—for example, if you make an enum variant or a public struct field conditional upon a feature, then that type must also be annotated with #[non_exhaustive]. Otherwise, a dependent crate that does not have the feature enabled may no longer compile if the feature is added due to some second crate in the dependency tree. 

**NOTE** *If you’re writing a large crate where you expect that your users will need only a subset of the functionality, you should consider making it so that larger components (usually modules) are guarded by features. That way, users can opt in to, and pay the compilation cost of, only the parts they really need.* 

**Workspaces** 

Crates play many roles in Rust—they are the vertices in the dependency graph, the boundaries for trait coherence, and the scopes for compilation features. Because of this, each crate is managed as a single compilation unit; the Rust compiler treats a crate more or less as one big source file compiled as one chunk that is ultimately turned into a single binary output (either a binary or a library). 

While this simplifies many aspects of the compiler, it also means that large crates can be painful to work with. If you change a unit test, a comment, or a type in one part of your application, the compiler must re-evaluate the entire crate to determine what, if anything, changed. Internally, the compiler implements a number of mechanisms to speed up this process, like incremental recompilation and parallel code generation, but ultimately the size of your crate is a big factor in how long your project takes to compile. 

For this reason, as your project grows, you may want to split it into multiple crates that internally depend on one another. Cargo has just the feature you need to make this convenient: workspaces. A *workspace* is a collection of crates (often called *subcrates*) that are tied together by a top-level *Cargo.toml* file like the one shown in Listing 5-4. 

```rust
[workspace]
members = [
  "foo",
  "bar/one",
  "bar/two",
 ]
```

*Listing 5-4: A workspace Cargo .toml*

The members array is a list of directories that each contain a crate in the workspace. Those crates all have their own *Cargo.toml* files in their own subdirectories, but they share a single *Cargo.lock* file and a single output directory. The crate names don’t need to match the entry in members. It is common, but not required, that crates in a workspace share a name prefix, usually chosen as the name of the “main” crate. For example, in the tokio crate, the members are called tokio, tokio-test, tokio-macros, and so on. 

Perhaps the most important feature of workspaces is that you can interact with all of the workspace’s members by invoking cargo in the root of the workspace. Want to check that they all compile? cargo check will check them all. Want to run all your tests? cargo test will test them all. It’s not quite as convenient as having everything in one crate, so don’t go splitting everything into minuscule crates, but it’s a pretty good approximation. 

*Cargo commands will generally do the “right thing” in a workspace. If you ever need to disambiguate, such as if two workspace crates both have a binary by the same name, use the* *-p* *flag (for package). If you are in the subdirectory for a particular workspace crate, you can pass* *--workspace* *to perform the command for the entire workspace instead.* 

Once you have a workspace-level *Cargo.toml* with the array of workspace members, you can set your crates to depend on one another using path dependencies, as shown in Listing 5-5. 

Project Structure **71** 

```toml
# bar/two/Cargo.toml
[dependencies]
one = { path = "../one" }
# bar/one/Cargo.toml
[dependencies]
foo = { path = "../../foo" }
```

*Listing 5-5: Intercrate dependencies among workspace crates*

Now if you make a change to the crate in *bar/two*, then only that crate is re-compiled, since foo and *bar/one* did not change. It may even be faster to compile your project from scratch, since the compiler does not need to evaluate your entire project source for optimization opportunities. 

**SPECIFYING INTRA-WORKSPACE DEPENDENCIES** 

The most obvious way to specify that one crate in a workspace depends on another is to use the path specifier, as shown in Listing 5-5 . However, if your individual subcrates are intended for public consumption, you may want to use version specifiers instead . 

Say you have a crate that depends on a Git version of the one crate from the bar workspace in Listing 5-5 with one = { git = ". . ." }, and a released version of foo (also from bar) with foo = "1.0.0" . Cargo will dutifully fetch the one Git repository, which holds the entire bar workspace, and see that one in turn depends on foo, located at ../../foo inside the workspace . But Cargo doesn’t know that the released version foo = "1.0.0" and the foo in the Git repository are the same crate! It considers them two separate dependencies that just happen to have the same name . 

You may already see where this is going . If you try to use any type from foo (1.0.0) with an API from one that accepts a type from foo, the compiler will reject the code . Even though the types have the same name, the compiler can’t know that they are the same underlying type . And the user will be thoroughly confused, since the compiler will say something like “expected foo::Type, got foo::Type .” 

The best way to mitigate this problem is to use path dependencies between subcrates only if they depend on unpublished changes . As long as one works with foo 1.0.0, it should list foo = "1.0.0" in its dependencies . Only if you make a change to foo that one needs should you change one to use a path dependency . And once you release a new version of foo that one can depend on, you should remove the path dependency again . 

This approach also has its shortcomings . Now if you change foo and then run the tests for one, you’ll see that one will be tested using the old foo, which may not be what you expected . You’ll probably want to configure your continuous integration infrastructure to test each subcrate both with the latest released versions of the other subcrates and with all of them configured to use path dependencies . 

**Project Configuration** 

Running cargo new sets you up with a minimal *Cargo.toml* that has the crate’s name, its version number, some author information, and an empty list of dependencies. That will take you pretty far, but as your project matures, there are a number of useful things you may want to add to your *Cargo.toml*. 

**Crate Metadata** 

The first and most obvious thing to add to your *Cargo.toml* is all the metadata directives that Cargo supports. In addition to obvious fields like description and homepage, it can be useful to include information such as the path to a *README* for the crate (readme), the default binary to run with cargo run (default-run), and additional keywords and categories to help *crates.io* categorize your crate. 

For crates with a more convoluted project layout, it’s also useful to set the include and exclude metadata fields. These dictate which files should be included and published in your package. By default, Cargo includes all files in a crate’s directory except any listed in your *.gitignore* file, but this may not be what you want if you also have large test fixtures, unrelated scripts, or other auxiliary data in the same directory that you *do* want under version control. As their names suggest, include and exclude allow you to include only a specific set of files or exclude files matching a given set of patterns, respectively. 

*If you have a crate that should never be published, or should be published only to certain alternative registries (that is, not to* crates.io*), you can set the* *publish* *directive to* *false* *or to a list of allowed registries.* 

The list of metadata directives you can use continues to grow, so make sure to periodically check in on the Manifest Format page of the Cargo reference (*https://doc.rust-lang.org/cargo/reference/manifest.html*). 

**Build Configuration** 

*Cargo.toml* can also give you control over how Cargo builds your crate. The most obvious tool for this is the build parameter, which allows you to write a completely custom build program for your crate (we’ll revisit this in Chapter 11). However, Cargo also provides two smaller, but very useful, mechanisms that we’ll explore here: patches and profiles. 

**[patch]** 

The [patch] section of *Cargo.toml* allows you to specify a different source for a dependency that you can use temporarily, no matter where in your dependencies the patched dependency appears. This is invaluable when you need to compile your crate against a modified version of some transitive dependency to test a bug fix, a performance improvement, or a new minor release you’re about to publish. Listing 5-6 shows an example of how you might temporarily use a variant of a set of dependencies. 

Project Structure **73** 

```rust
[patch.crates-io]
# use a local (presumably modified) source
regex = { path = "/home/jon/regex" }
# use a modification on a git branch
serde = { git = "https://github.com/serde-rs/serde.git", branch = "faster" }
# patch a git dependency
[patch.'https://github.com/jonhoo/project.git']
project = { path = "/home/jon/project" }
```

Listing 5-6: Overriding dependency sources in Cargo .toml using *[patch]*

Even if you patch a dependency, Cargo takes care to check the crate versions so that you don’t accidentally end up patching the wrong major version of a crate. If you for some reason transitively depend on multiple major versions of the same crate, you can patch each one by giving them distinct identifiers, as shown in Listing 5-7. 

```
[patch.crates-io]
nom4 = { path = "/home/jon/nom4", package = "nom" }
nom5 = { path = "/home/jon/nom5", package = "nom" }
```

Listing 5-7: Overriding multiple versions of the same crate in Cargo .toml using *[patch]* 

Cargo will look at the *Cargo.toml* inside each path, realize that /nom4 contains major version 4 and that /nom5 contains major version 5, and patch the two versions appropriately. The package keyword tells Cargo to look for a crate by the name nom in both cases instead of using the dependency identifiers (the part on the left) as it does by default. You can use package this way in your regular dependencies as well to rename a dependency! 

Keep in mind that patches are not taken into account in the package that’s uploaded when you publish a crate. A crate that depends on your crate will use only its own [patch] section (which may be empty), not that of your crate! 

**CRATES VS. PACKAGES** 

You may wonder what the difference between a package and a crate is. These two terms are often used interchangeably in informal contexts, but they also have specific definitions that vary depending on whether you’re talking about the Rust compiler, Cargo, crates.io, or something else . I personally think of a crate as a Rust module hierarchy starting at a root.rs file (one where you can use crate-level attributes like #![feature])—usually something like lib.rs or main.rs . In contrast, a package is a collection of crates and metadata, so essentially all that’s described by a Cargo.toml file . That may include a library crate, multiple binary crates, some integration test crates, and maybe even multiple workspace members that themselves have Cargo.toml files . 

**[profile]** 

The [profile] section lets you pass additional options to the Rust compiler in order to change the way it compiles your crate. These options fall primarily into three categories: performance options, debugging options, and options that change code behavior in user-defined ways. They all have different defaults depending on whether you are compiling in debug mode or in release mode (other modes also exist). 

The three primary performance options are opt-level, codegen-units, and lto. The opt-level option tweaks runtime performance by telling the compiler how aggressively to optimize your program (0 is “not at all,” 3 is “as much as you can”). The higher the setting, the more optimized your code will be, which *may* make it run faster. Extra optimization comes at the cost of higher compile times, though, which is why optimizations are generally enabled only for release builds. 

**NOTE** *You can also set* *opt-level* *to* *"s"* *to optimize for binary size, which may be important on embedded platforms.* 

The codegen-units option is about compile-time performance. It tells the compiler how many independent compilation tasks (*code generation units*) it is allowed to split the compilation of a single crate into. The more pieces a large crate’s compilation is split into, the faster it will compile, since more threads can help compile the crate in parallel. Unfortunately, to achieve this speedup, the threads need to work more or less independently, which means code optimization suffers. Imagine, for example, that the segment of a crate compiling in one thread could benefit from inlining some code in a different segment—since the two segments are independent, that inlining can’t happen! This setting, then, is a trade-off between compile-time performance and runtime performance. By default, Rust uses an effectively unbounded number of codegen units in debug mode (basically, “compile as fast as you can”) and a smaller number (16 at the time of writing) in release mode. 

The lto setting toggles *link-time optimization (LTO)*, which enables the compiler (or the linker, if you want to get technical about it) to jointly optimize bits of your program, known as *compilation units*, that were originally compiled separately. The exact details of LTO are beyond the scope of this book, but the basic idea is that the output from each compilation unit includes information about the code that went into that unit. After all the units have been compiled, the linker makes another pass over all of the units and uses that additional information to optimize the combined compiled code. This extra pass adds to the compile time but recovers most of the runtime performance that may have been lost due to splitting the compilation into smaller parts. In particular, LTO can offer significant performance boosts to performance-sensitive programs that might benefit from cross-crate optimization. Beware, though, that cross-crate LTO can add a lot to your compile time. 

Rust performs LTO across all the codegen units within each crate by default in an attempt to make up for the lost optimizations caused by using many codegen units. Since the LTO is performed only within each crate, rather than across crates, this extra pass isn’t too onerous, and the added compile time should be lower than the amount of time saved by using a lot of codegen units. Rust also offers a technique known as *thin LTO*, which allows the LTO pass to be mostly parallelized, at the cost of missing some optimizations a “full” LTO pass would have found. 

**NOTE** *LTO can be used to optimize across foreign function interface boundaries in many cases, too. See the* *linker-plugin-lto rustc* *flag for more details.* 

The [profile] section also supports flags that aid in debugging, such as debug, debug-assertions, and overflow-checks. The debug flag tells the compiler to include debug symbols in the compiled binary. This increases the binary size, but it means that you get function names and such, rather than just instruction addresses, in backtraces and profiles. The debug-assertions flag enables the debug_assert! macro and other related debug code that isn’t compiled otherwise (through cfg(debug_assertions)). Such code may make your program run slower, but it makes it easier to catch questionable behavior at runtime. The overflow-checks flag, as the name implies, enables overflow checks on integer operations. This slows them down (notice a trend here?) but can help you catch tricky bugs early on. By default, these are all enabled in debug mode and disabled in release mode. 

**[profile.\*.panic]** 

The [profile] section has another flag that deserves its own subsection: panic. This option dictates what happens when code in your program calls panic!, either directly or indirectly through something like unwrap. You can set panic to either unwind (the default on most platforms) or abort. We’ll talk more about panics and unwinding in Chapter 9, but I’ll give a quick summary here. 

Normally in Rust, when your program panics, the thread that panicked starts *unwinding* its stack. You can think of unwinding as forcibly returning recursively from the current function all the way to the bottom of that thread’s stack. That is, if main called foo, foo called bar, and bar called baz, a panic in baz would forcibly return from baz, then bar, then foo, and finally from main, resulting in the program exiting. A thread that unwinds will drop all values on the stack normally, which gives the values a chance to clean up resources, report errors, and so on. This gives the running system a chance to exit gracefully even in the case of a panic. 

When a thread panics and unwinds, other threads continue running unaffected. Only when (and if) the thread that ran main exits does the program terminate. That is, the panic is generally isolated to the thread in which the panic occurred. 

This means unwinding is a double-edged sword; the program is limping along with some failed components, which may cause all sorts of strange behaviors. For example, imagine a thread that panics halfway through updating the state in a Mutex. Any thread that subsequently acquires that Mutex must now be prepared to handle the fact that the state may be in a partially updated, inconsistent state. For this reason, some synchronization primitives (like Mutex) will remember if a panic occurred when they were last accessed 

**WARNING** 

and communicate that to any thread that tries to access the primitive subsequently. If a thread encounters such a state, it will normally also panic, which leads to a cascade that eventually terminates the entire program. But that is arguably better than continuing to run with corrupted state! 

The bookkeeping needed to support unwinding is not free, and it often requires special support by the compiler and the target platform. For example, many embedded platforms cannot unwind the stack efficiently at all. Rust therefore supports a different panic mode: abort ensures the whole program simply exits immediately when a panic occurs. In this mode, no threads get to do any cleanup. This may seem severe, and it is, but it ensures that the program is never running in a half-working state and that errors are made visible immediately. 

*The panic setting is global—if you set it to* *abort**, all your dependencies are also compiled with* *abort**.* 

You may have noticed that when a thread panics, it tends to print a *backtrace*: the trail of function calls that led to where the panic occurred. This is also a form of unwinding, though it is separate from the unwinding panic behavior discussed here. You can have backtraces even with panic=abort by passing -Cforce-unwind-tables to rustc, which makes rustc include the information necessary to walk back up the stack while still terminating the program on a panic. 

**PROFILE OVERRIDES** 

You can set profile options for just a particular dependency, or a particular profile, using profile overrides . For example, Listing 5-8 shows how to enable aggressive optimizations for the serde crate and moderate optimizations for all other crates in debug mode, using the [profile.**.package.**] syntax . 

```toml
[profile.dev.package.serde]
opt-level = 3
[profile.dev.package."*"]
opt-level = 2
```

*Listing 5-8: Overriding profile options for a specific dependency or for a specific mode*

This kind of optimization override can be handy if some dependency would be prohibitively slow in debug mode (such as decompression or video encoding), and you need it optimized so that your test suite won’t take several days to complete . You can also specify global profile defaults using a [profile.dev] (or similar) section in the Cargo configuration file in ~/.cargo/config . 

When you set optimization parameters for a specific dependency, keep in mind that the parameters apply only to the code compiled as part of that crate; if serde in this example has a generic method or type that you use in your crate,  the code of that method or type will be monomorphized and optimized in your crate, and your crate’s profile settings will apply, not those in the profile override for serde . 

**Conditional Compilation** 

Most Rust code you write is universal—it’ll work the same regardless of what CPU or operating system it runs on. But sometimes you’ll have to do something special to get the code to work on Windows, on ARM chips, or when compiled against a particular platform application binary interface (ABI). Or maybe you want to write an optimized version of a particular function when a given CPU instruction is available, or disable some slow but uninteresting setup code when running in a continuous integration (CI) environment. To cater to cases like these, Rust provides mechanisms for *conditional compilation*, in which a particular segment of code is compiled only if certain conditions are true of the compilation environment. 

We denote conditional compilation with the cfg keyword that you saw earlier in the chapter in “Using Features in Your Crate.” It usually appears in the form of the #[cfg(condition)] attribute, which says to compile the next item only if condition is true. Rust also has #[cfg_attr(condition, attribute)], which is compiled as #[attribute] if condition holds and is a no-op otherwise. You can also evaluate a cfg condition as a Boolean expression using the cfg!(condition) macro. 

Every cfg construct takes a single condition made up of options, like feature = "some-feature", and the combinators all, any, and not, which do what you would probably expect. Options are either simple names, like unix, or key/value pairs like those used by feature conditions. 

There are a number of interesting options you can make compilation dependent on. Let’s go through them, from most common to least common: 

**Feature options** 

You’ve already seen examples of these. Feature options take the form feature = "name-of-feature" and are considered true if the named feature is enabled. You can check for multiple features in a single condition using the combinators. For example, any(feature = "f1", feature = "f2") is true if either feature f1 or feature f2 is enabled. 

**Operating system options** 

These use key/value syntax with the key target_os and values like windows, macos, and linux. You can also specify a family of operating systems using target_family, which takes the value windows or unix. These are common enough that they have received their own named short forms, so you can use cfg(windows) and cfg(unix) directly. For example, if you wanted a particular code segment to be compiled only on macOS and Windows, you would write: #[cfg(any(windows, target_os = "macos"))]. 

**Context options** 

These let you tailor code to a particular compilation context. The most common of these is the test option, which is true only when the crate is being compiled under the test profile. Keep in mind that test is set only for the crate that is being tested, not for any of its dependencies. This also means that test is not set in your crate when running integration tests; it’s the integration tests that are compiled under the test profile, whereas your actual crate is compiled normally (that is, without test set). The same applies to the doc and doctest options, which are set only when building documentation or compiling doctests, respectively. There’s also the debug_assertions option, which is set in debug mode by default. 

**Tool options** 

Some tools, like clippy and Miri, set custom options (more on that later) that let you customize compilation when run under these tools. Usually, these options are named after the tool in question. For example, if you want a particular compute-intensive test not to run under Miri, you can give it the attribute #[cfg_attr(miri, ignore)]. 

**Architecture options** 

These let you compile based on the CPU instruction set the compiler is targeting. You can specify a particular architecture with target_arch, which takes values like x86, mips, and aarch64, or you can specify a particular platform feature with target_feature, which takes values like avx or sse2. For very low-level code, you may also find the target_endian and target_pointer_width options useful. 

**Compiler options** 

These let you adapt your code to the platform ABI it is compiled against and are available through target_env with values like gnu, msvc, and musl. For historical reasons, this value is often empty, especially on GNU platforms. You normally need this option only if you need to interface directly with the environment ABI, such as when linking against an ABI-specific symbol name using #[link]. 

While cfg conditions are usually used to customize code, some can also be used to customize dependencies. For example, the dependency winrt usually makes sense only on Windows, and the nix crate is probably useful only on Unix-based platforms. Listing 5-9 gives an example of how you can use cfg conditions for this: 

```toml
[target.'cfg(windows)'.dependencies]
winrt = "0.7"
[target.'cfg(unix)'.dependencies]
nix = "0.17"
```

*Listing 5-9: Conditional dependencies*

Project Structure **79** 

Here, we specify that winrt version 0.7 should be considered a dependency only under cfg(windows) (so, on Windows), and nix version 0.17 only under cfg(unix) (so, on Linux, macOS, and other Unix-based platforms). One thing to keep in mind is that the [dependencies] section is evaluated very early in the build process, when only certain cfg options are available. In particular, feature and context options are not yet available at this point, so you cannot use this syntax to pull in dependencies based on features and contexts. You can, however, use any cfg that depends only on the target specification or architecture, as well as any options explicitly set by tools that call into rustc (like cfg(miri)). 

**NOTE** *While we’re on the topic of dependency specifications, I highly recommend that you set up your CI infrastructure to perform basic auditing of your dependencies using tools like* *cargo-deny* *and* *cargo-audit**. These tools will detect cases where you transitively depend on multiple major versions of a given dependency, where you depend on crates that are unmaintained or have known security vulnerabilities, or where you use licenses that you may want to avoid. Using such a tool is a great way to raise the quality of your codebase in an automated way!* 

It’s also quite simple to add your own custom conditional compilation options. You just have to make sure that --cfg=myoption is passed to rustc when rustc compiles your crate. The easiest way to do this is to add your --cfg to the RUSTFLAGS environment variable. This can come in handy in CI, where you may want to customize your test suite depending on whether it’s being run on CI or on a dev machine: add --cfg=ci to RUSTFLAGS in your CI setup, and then use cfg(ci) and cfg(not(ci)) in your code. Options set this way are also available in *Cargo.toml* dependencies. 

**Versioning** 

All Rust crates are versioned and are expected to follow Cargo’s implementation of semantic versioning. *Semantic versioning* dictates the rules for what kinds of changes require what kinds of version increases and for which versions are considered compatible, and in what ways. The RFC 1105 standard itself is well worth reading (it’s not horribly technical), but to summarize, it differentiates between three kinds of changes: breaking changes, which require a major version change; additions, which require a minor version change; and bug fixes, which require only a patch version change. RFC 1105 does a decent job of outlining what constitutes a breaking change in Rust, and we’ve touched on some aspects of it elsewhere in this book. 

I won’t go into detail here about the exact semantics of the different types of changes. Instead, I want to highlight some less straightforward ways version numbers come up in the Rust ecosystem, which you need to keep in mind when deciding how to version your own crates. 

**Minimum Supported Rust Version** 

The first Rust-ism is the *minimum supported Rust version (MSRV)*. There is much debate in the Rust community about what policy projects should adhere to when it comes to their MSRV and versioning, and there’s no truly good answer. The core of the problem is that some Rust users are limited to using older versions of Rust, often in an enterprise setting where they have little choice. If we constantly take advantage of newly stabilized APIs, those users will not be able to compile the latest versions of our crates and will be left behind. 

There are two techniques crate authors can use to make life a little easier for users in this position. The first is to establish an MSRV policy promising that new versions of a crate will always compile with any stable release from the last *X* months. The exact number varies, but 6 or 12 months is common. With Rust’s six-week release cycle, that corresponds to the latest four or eight stable releases, respectively. Any new code introduced to the project must compile with the MSRV compiler (usually checked by CI) or be held until the MSRV policy allows it to be merged as is. This can sometimes be a pain, as it means these crates cannot take advantage of the latest and greatest the language has to offer, but it will make life easier for your users. 

The second technique is to make sure to increase the minor version number of your crate any time that the MSRV changes. So, if you release version 2.7.0 of your crate and that increases your MSRV from Rust 1.44 to Rust 1.45, then a project that is stuck on 1.44 and that depends on your crate can use the dependency version specifier version = "2, <2.7" to keep the project working until it can move on to Rust 1.45. It’s important that you increment the minor version, not just the patch version, so that you can still issue critical security fixes for the previous MSRV release by doing another patch release if necessary. 

Some projects take their MSRV support so seriously that they consider an MSRV change a breaking change and increment the major version number. This means that downstream projects will explicitly have to opt in to an MSRV change, rather than opting out—but it also means that users who do not have such strict MSRV requirements will not see future bug fixes without updating their dependencies, which may require *them* to issue a breaking change as well. As I said, none of these solutions are without drawbacks. 

Enforcing an MSRV in the Rust ecosystem today is challenging. Only a small subset of crates provide any MSRV guarantees, and even if your dependencies do, you will need to constantly monitor them to know when they increase their MSRV. When they do, you’ll need to do a new release of your crate with the restricted version bounds mentioned previously to make sure your MSRV doesn’t also change. This may in turn force you to forego security and performance updates made to your dependencies, as you’ll have to continue using older versions until your MSRV policy permits updating. And that decision also carries over to your dependents. There have been proposals to build MSRV checking into Cargo itself, but nothing workable has been stabilized as of this writing. 

**Minimal Dependency Versions** 

When you first add a dependency, it’s not always clear what version specifier you should give that dependency. Programmers commonly choose the latest version, or just the current major version, but chances are that both of those choices are wrong. By “wrong,” I don’t mean that your crate won’t compile, but rather that making that choice may cause strife for users of your crate down the line. Let’s look at why each of these cases is problematic. 

First, consider the case where you add a dependency on hugs = "1.7.3", the latest published version. Now imagine that a developer somewhere depends on your crate, but they also depend on some other crate, foo, that itself depends on hugs. Further imagine that the author of foo is really careful about their MSRV policy, so they depend on hugs = "1, <1.6". Here, you’ll run into trouble. When Cargo sees hugs = "1.7.3", it considers only versions >=1.7. But then it sees that foo’s dependency on hugs requires <1.6, so it gives up and reports that there is no version of hugs compatible with all the requirements. 

**NOTE** *In practice, there are a number of reasons why a crate may explicitly not want a newer version of a dependency. The most common ones are to enforce MSRV, to meet enterprise auditing requirements (the newer version will contain code that hasn’t been audited), and to ensure reproducible builds where only the exact listed version is used.* 

This is unfortunate, as it could well be that your crate compiles fine with, say, hugs 1.5.6. Maybe it even compiles fine with *any* 1.X version! But by using the latest version number, you are telling Cargo to consider only versions at or beyond that minor version. Is the solution to use hugs = "1" instead, then? No, that’s not quite right either. It could be that your code truly does depend on something that was added only in hugs 1.6, so while 1.6.2 would be fine, 1.5.6 would not be. You wouldn’t notice this if you were only ever compiling your crate in situations where a newer version ends up getting used, but if some crate in the dependency graph specifies hugs = "1, <1.5", your crate would not compile! 

The right strategy is to list the earliest version that has all the things your crate depends on and to make sure that this remains the case even as you add new code to your crate. But how do you establish that beyond trawling the changelogs, or through trial and error? Your best bet is to use Cargo’s unstable -Zminimal-versions flag, which makes your crate use the minimum acceptable version for all dependencies, rather than the maximum. Then, set all your dependencies to just the latest major version number, try to compile, and add a minor version to any dependencies that don’t. Rinse and repeat until everything compiles fine, and you now have your minimum version requirements! 

It’s worth noting that, like with MSRV, minimal version checking faces an ecosystem adoption problem. While you may have set all your version specifiers correctly, the projects you depend on may not have. This makes the Cargo minimal versions flag hard to use in practice (and is why it’s still unstable). If you depend on foo, and foo depends on bar with a specifier of bar = "1" when it actually requires bar = "1.4", Cargo will report that it failed to compile foo no matter how you list foo because the -Z flag tells it 

to always prefer minimal versions. You can work around this by listing bar directly in *your* dependencies with the appropriate version requirement, but these workarounds can be painful to set up and maintain. You may end up listing a large number of dependencies that are only really pulled in through your transitive dependencies, and you’ll have to keep that list up to date as time goes on. 

**NOTE** *One current proposal is to present a flag that favors minimal versions for the current crate but maximal ones for dependencies, which seems quite promising.* 

**Changelogs** 

For all but the most trivial crates, I highly recommend keeping a changelog. There is little more frustrating than seeing that a dependency has received a major version bump and then having to dig through the Git logs to figure out what changed and how to update your code. I recommend that you do not just dump your Git logs into a file named *changelog*, but instead keep a manual changelog. It is much more likely to be useful. 

A simple but good format for changelogs is the Keep a Changelog format documented at *https://keepachangelog.com/*. 

**Unreleased Versions** 

Rust considers version numbers even when the source of a dependency is a directory or a Git repository. This means that semantic versioning is important even when you have not yet published a release to *crates.io*; it matters what version is listed in your *Cargo.toml* between releases. The semantic versioning standard does not dictate how to handle this case, but I’ll provide a workflow that works decently well without being too onerous. 

After you’ve published a release, immediately update the version number in your *Cargo.toml* to the next patch version with a suffix like *-alpha.1*. If you just released 2.0.3, make the new version 2.0.4-alpha.1. If you just released an alpha, increment the alpha number instead. 

As you make changes to the code between releases, keep an eye out for additive or breaking changes. If one happens, and the corresponding version number has not changed since the last release, increment it. For example, if the last released version is 2.0.3, the current version is 2.0.4-alpha.2, and you make an additive change, make the version with the change 2.1.0-alpha.1. If you made a breaking change, it becomes 3.0.0-alpha.1 instead. If the corresponding version increase has already been made, just increment the alpha number. 

When you make a release, remove the suffix (unless you want to do a prerelease), then publish, and start from the top. 

This process is effective because it makes two common workflows work much better. First, imagine that a developer depends on major version 2 of your crate, but they need a feature that’s currently available only in Git. Then you commit a breaking change. If you don’t increase the major version at the same time, their code will suddenly fail in unexpected ways, either by failing to compile or as a result of weird runtime issues. If you follow the procedure laid out here, they’ll instead be notified by Cargo that a breaking change has occurred, and they’ll have to either resolve that or pin a specific commit. 

Next, imagine that a developer needs a feature they just contributed to your crate, but which isn’t part of any released version of your crate yet. They’ve used your crate behind a Git dependency for a while, so other developers on their project already have older checkouts of your crate’s repository. If you do not increment the major version number in Git, this developer has no way to communicate that their project now relies on the feature that was just merged. If they push their change, their fellow developers will find that the project no longer compiles, since Cargo will reuse the old checkout. If, on the other hand, the developer can increment the minor version number for the Git dependency, then Cargo will realize that the old checkout is outdated. 

This workflow is by no means perfect. It doesn’t provide a good way to communicate multiple minor or major changes between releases, and you still need to do a bit of work to keep track of the versions. However, it does address two of the most common issues Rust developers run into when they work against Git dependencies, and even if you make multiple such changes between releases, this workflow will still catch many of the issues. 

If you’re not too worried about small or consecutive version numbers in releases, you can improve this suggested workflow by simply always incrementing the appropriate part of the version number. Be aware, though, that depending on how frequently you make such changes, this may make your version numbers quite large! 

## Summary

In this chapter, we’ve looked at a number of mechanisms for configuring, organizing, and publishing crates, for both your own benefit and that of others. We’ve also gone over some common gotchas when working with dependencies and features in Cargo that now hopefully won’t catch you out in the future. In the next chapter we’ll turn to testing and dig into how you go beyond Rust’s simple #[test] functions that we know and love. 