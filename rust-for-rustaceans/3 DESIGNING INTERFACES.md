Every project, no matter how large or small, has an API. In fact, it usually has several. Some of these are user-facing, like an HTTP endpoint or a command line interface, and some are developer-facing, like a library’s public interface. On top of these, Rust crates also have a number of internal interfaces: every type, trait, and module boundary has its own miniature API that the rest of your code interfaces with. As your codebase grows in size and complexity, you’ll find it worth- while to invest some thought and care into how you design even the inter- nal APIs to make the experience of using and maintaining the code over time as pleasant as possible. 

In this chapter we’ll look at some of the most important considerations for writing idiomatic interfaces in Rust, whether the users of those inter- faces are your own code or other developers using your library. These essentially boil down to four principles: your interfaces should be *unsurprising*, *flexible*, *obvious*, and *constrained*. I’ll discuss each of these principles in turn, to provide some guidance for writing reliable and usable interfaces. 

I highly recommend taking a look at the Rust API Guidelines (*https:// rust-lang.github.io/api-guidelines/*) after you’ve read this chapter. There’s an excellent checklist you can follow, with a detailed run-through of each recommendation. Many of the recommendations in this chapter are also checked by the cargo clippy tool, which you should start running on your code if you aren’t already. I also encourage you to read through Rust RFC 1105 (*https://rust-lang.github.io/rfcs/1105-api-evolution.html*) and the chapter of *The Cargo Book* on SemVer compatibility (*https://doc.rust-lang.org/cargo/ reference/semver.html*), which cover what is and is not a breaking change in Rust. 

### Unsurprising

The Principle of Least Surprise, otherwise known as the Law of Least Astonishment, comes up a lot in software engineering, and it holds true for Rust interfaces as well. Where possible, your interfaces should be intuitive enough that if the user has to guess, they usually guess correctly. Of course, not everything about your application is going to be immediately intuitive in this way, but anything that *can* be unsurprising should be. The core idea here is to stick close to things the user is likely to already know so that they don’t have to relearn concepts in a different way than they’re used to. That way you can save their brain power for figuring out the things that are actually specific to your interface. 

There are a variety of ways you can make your interfaces predictable. Here, we’ll look at how you can use naming, common traits, and ergonomic trait tricks to help the user out. 

***Naming Practices***

A user of your interface will encounter it first through its names; they will immediately start to infer things from the names of types, methods, variables, fields, and libraries they come across. If your interface reuses names for things—say, methods and types—from other (perhaps common) inter- faces, the user will know they can make certain assumptions about your methods and types. A method called iter probably takes &self, and prob- ably gives you an iterator. A method called into_inner probably takes self and likely returns some kind of wrapped type. A type called SomethingError probably implements std::error::Error and appears in various Results. By reusing common names for the same purpose, you make it easier for the user to guess what things do and allow them to more easily understand the things that are different about your interface. 

A corollary to this is that things that share a name *should* in fact work the same way. Otherwise—for example, if your iter method takes self, or if your SomethingError type does not implement Error—the user will likely write incorrect code based on how they expect the interface to work. They will be surprised and frustrated and will have to spend time digging into how your interface differs from their expectations. When we can save the user this kind of friction, we should. 

***Common Traits for Types***

Users in Rust will also make the major assumption that everything in the interface “just works.” They expect to be able to print any type with {:?} and send anything and everything to another thread, and they expect that every type is Clone. Where possible, we should again avoid surprising the user and eagerly implement most of the standard traits even if we do not need them immediately. 

Because of the coherence rules discussed in Chapter 2, the compiler will not allow users to implement these traits when they need them. Users aren’t allowed to implement a foreign trait (like Clone) for a foreign type like one from your interface. They would instead need to wrap your interface type in their own type, and even then it may be quite difficult to write a reasonable implementation without access to the type’s internals. 

First among these standard traits is the Debug trait. Nearly every type can, and should, implement Debug, even if it only prints the type’s name. Using #[derive(Debug)] is often the best way to implement the Debug trait in your interface, but keep in mind that all derived traits automatically add the same bound for any generic parameters. You could also simply write your own implementation by leveraging the various debug_ helpers on fmt::Formatter. 

Tied in close second are the Rust auto-traits Send and Sync (and, to a lesser extent, Unpin). If a type does not implement one of these traits, it should be for a very good reason. A type that is not Send can’t be placed in a Mutex and can’t be used even transitively in an application that contains a thread pool. A type that is not Sync can’t be shared through an Arc or placed in a static variable. Users have come to expect that types *just work* in these contexts, especially in the asynchronous world where nearly everything runs on a thread pool, and will become frustrated if you don’t ensure that your types implement these traits. If your types cannot implement them, make sure that fact, and the reason why, is well documented! 

The next set of nearly universal traits you should implement is Clone and Default. These traits can be derived or implemented easily and make sense to implement for most types. If your type cannot implement these traits, make sure to call it out in your documentation, as users will usually expect to be able to easily create more (and new) instances of types as they see fit. If they cannot, they will be surprised. 

One step further down in the hierarchy of expected traits is the com- parison traits: PartialEq, PartialOrd, Hash, Eq, and Ord. The PartialEq trait is particularly desirable, because users will at some point inevitably have two instances of your type that they wish to compare with == or assert_eq!. Even if your type would compare equal for only the same instance of the type, it’s worth implementing PartialEq to enable your users to use assert_eq!. 

PartialOrd and Hash are more specialized, and may not apply quite as broadly, but where possible you will want to implement them too. This is especially true for types a user might use as the key in a map, or a type they may deduplicate using any of the std::collection set types, since they tend to require these bounds. Eq and Ord come with additional semantic require- ments on the implementing type’s comparison operations beyond those of PartialEq and PartialOrd. These are well documented in the documentation for those traits, and you should implement them *only* if you’re sure those semantics actually apply to your type. 

Finally, for most types, it makes sense to implement the serde crate’s Serialize and Deserialize traits. These can be easily derived, and the serde _derive crate even comes with mechanisms for overwriting the serialization for just one field or enum variant. Since serde is a third-party crate, you may not wish to add a required dependency on it. Most libraries therefore choose to provide a serde feature that adds support for serde only when the user opts into it. 

You might be wondering why I haven’t included the derivable trait Copy in this section. There are two things that set Copy apart from the other traits mentioned. The first is that users do not generally expect types to be Copy; quite to the contrary, they tend to expect that if they want two copies of something, they have to call clone. Copy changes the semantics of moving a value of the given type, which might surprise the user. This ties in to the second observation: it is very easy for a type to *stop* being Copy, because Copy types are highly restricted. A type that starts out simple can easily end up having to hold a String, or some other non-Copy type. Should that happen, and you have to remove the Copy implementation, that’s a backward incom- patible change. In contrast, you rarely have to remove a Clone implementa- tion, so that’s a less onerous commitment. 

***Ergonomic Trait Implementations***

Rust does not automatically implement traits for references to types that implement traits. To phrase this a different way, you cannot generally call fn foo<T: Trait>(t: T) with a &Bar, even if Bar: Trait. This is because Trait may contain methods that take &mut self or self, which obviously cannot be called on &Bar. Nonetheless, this behavior might be very surprising to a user who sees that Trait has only &self methods! 

For this reason, when you define a new trait, you’ll usually want to pro- vide blanket implementations as appropriate for that trait for &T where T: Trait, &mut T where T: Trait, and Box<T> where T: Trait. You may be able to implement only some of these depending on what receivers the methods of Trait have. Many of the traits in the standard library have similar implemen- tations, precisely because that leads to fewer surprises for the user. 

Iterators are another case where you’ll often want to specifically add trait implementations on references to a type. For any type that can be iterated over, consider implementing IntoIterator for both &MyType and &mut MyType where applicable. This makes for loops work with borrowed instances of your type as well out of the box, just like users would expect. 

***Wrapper Types***

Rust does not have object inheritance in the classical sense. However, the Deref trait and its cousin AsRef both provide something a little like inheritance. These traits allow you to have a value of type T and call methods on some type U by calling them directly on the T-typed value if T: Deref<Target = U>. This feels like magic to the user, and is generally great. 

If you provide a relatively transparent wrapper type (like Arc), there’s a good chance you’ll want to implement Deref so that users can call methods on the inner type by just using the . operator. If accessing the inner type does not require any complex or potentially slow logic, you should also consider implementing AsRef, which allows users to easily use a &WrapperType as an &InnerType. For most wrapper types, you will also want to implement From<InnerType> and Into<InnerType> where possible so that your users can easily add or remove your wrapping. 

You may also have come across the Borrow trait, which feels very similar to Deref and AsRef but is really a bit of a different beast. Specifically, Borrow is tailored for a much narrower use case: allowing the caller to supply any one of multiple essentially identical variants of the same type. It could, per- haps, have been called Equivalent instead. For example, for a HashSet<String>, Borrow allows the caller to supply either a &str *or* a &String. While the same could have been achieved with AsRef, that would not be safe without Borrow’s additional requirement that the target type implements Hash, Eq, and Ord exactly the same as the implementing type. Borrow also has a blanket imple- mentation of Borrow<T> for T, &T, and &mut T, which makes it convenient to use in trait bounds to accept either owned *or* referenced values of a given type. In general, Borrow is intended only for when your type is essentially equiva- lent to another type, whereas Deref and AsRef are intended to be imple- mented more widely for anything your type can “act as.” 

> **DEREF AND INHERENT METHODS** 

> The magic around the dot operator and Deref can get confusing and surprising when there are methods on T that take self . For example, given a value t: T, it is not clear whether t.frobnicate() frobnicates the T or the underlying U! 

> For this reason, types that allow you to transparently call methods on some inner type that isn’t known in advance should avoid inherent methods . It’s fine for Vec to have a push method even though it dereferences to a slice, since you know that slices won’t get a push method any time soon . But if your type dereferences to a user-controlled type, any inherent method you add may also exist on that user-controlled type, and thus cause issues . In these cases, favor static methods of the form fn frobnicate(t: T) . That way, t.frobnicate() always calls U::frobnicate, and T::frobnicate(t) can be used to frobnicate the T itself . 

### Flexible

Every piece of code you write includes, implicitly or explicitly, a contract. The contract consists of a set of requirements and a set of promises. The requirements are restrictions on how the code can be used, while the promises are guarantees about how the code can be used. When designing a new interface, you want to think carefully about this contract. A good rule of thumb is to avoid imposing unnecessary restrictions and to only make promises you can keep. Adding restrictions or removing promises usually requires a major semantic version change and is likely to break code else- where. Relaxing restrictions or giving additional promises, on the other hand, is usually backward compatible. 

In Rust, restrictions usually come in the form of trait bounds and argument types, and promises come in the form of trait implementations and return types. For example, compare the three function signatures in Listing 3-1. 

```
fn frobnicate1(s: String) -> String
fn frobnicate2(s: &str) -> Cow<'_, str>
fn frobnicate3(s: impl AsRef<str>) -> impl AsRef<str>
```

*Listing 3-1: Similar function signatures with different contracts*

These three function signatures all take a string and return a string, but they do so under very different contracts. 

The first function requires the caller to own the string in the form of the String type, and it promises that it will return an owned String. Since the contract requires the caller to allocate and requires us to return an owned String, we cannot later make this function allocation-free in a back- ward compatible way. 

The second function relaxes the contract: the caller can provide any reference to a string, so the user no longer needs to allocate or give up own- ership of a String. It also promises to give back a std::borrow::Cow, meaning it can return either a string reference or an owned String, depending on whether it needs to own the string. The promise here is that the function will always return a Cow, which means that we cannot, say, change it to use some other optimized string representation later. The caller must also spe- cifically provide a &str, so if they have, say, a pre-existing String of their own, they must dereference it to a &str to call our function. 

The third function lifts these restrictions. It requires only that the user pass in a type that can produce a reference to a string, and it promises only that the return value can produce a reference to a string. 

None of these function signatures is *better* than the others. If you need ownership of a string in the function, you can use the first argument type to avoid an extra string copy. If you want to allow the caller to take advan- tage of the case where an owned string was allocated and returned, the sec- ond function with a return type of Cow may be a good choice. Instead, what I want you to take away from this is that you should think carefully about what contract your interface binds you to, because changing it after the fact can be disruptive. 

In the remainder of this section I give examples of interface design decisions that often come up, and their implications for your interface contract. 

***Generic Arguments***

One obvious requirement your interface must place on users is what types they must provide to your code. If your function explicitly takes a Foo, the user must own and give you a Foo. There is no way around it. In most cases it pays off to use generics rather than concrete types, to allow the caller to pass any type that conforms to what your function actually needs, rather than only a particular type. Changing &str in Listing 3-1 to impl AsRef<str> is an example of this kind of relaxing. One way to go about relaxing requirements this way is to start with the argument fully generic with no bounds, and then just follow the compiler errors to discover what bounds you need to add. 

However, if taken to the extreme, this approach would make every argu- ment to every function its own generic type, which would be both hard to read and hard to understand. There are no hard-and-fast rules for exactly when you should or should not make a given parameter generic, so use your best judgment. A good rule of thumb is to make an argument generic if you can think of other types a user might reasonably and frequently want to use instead of the concrete type you started with. 

You may remember from Chapter 2 that generic code is duplicated for every combination of types ever used with the generic code through mono- morphization. With that in mind, the idea of making lots of arguments generic might make you worried about overly enlarging your binaries. In Chapter 2 we also discussed how you can use dynamic dispatch to mitigate this at a (usually) negligible performance cost, and that applies here too. For arguments that you take by reference anyway (recall that dyn Trait is not Sized, and that you need a wide pointer to use them), you can easily replace your generic argument with one that uses dynamic dispatch. For instance, instead of impl AsRef<str>, you could take &dyn AsRef<str>. 

Before you go running to do that, though, there are a few things you should consider. First, you are making this choice on behalf of your users, who cannot opt out of dynamic dispatch. If you know that the code you’re applying dynamic dispatch to will never be performance-sensitive, that
 may be fine. But if a user comes along who wants to use your library in their high-performance application, dynamic dispatch in a function that is called in a hot loop may be a deal breaker. Second, at the time of writing, using dynamic dispatch will work only when you have a simple trait bound like T: AsRef<str> or impl AsRef<str>. For more complex bounds, Rust does not know how to construct a dynamic dispatch vtable, so you cannot take, say, &dyn Hash + Eq. And finally, remember that with generics, the caller can always choose dynamic dispatch themselves by passing in a trait object. The reverse is not true: if you take a trait object, that is what the caller must provide. 

It may be tempting to start your interfaces off with concrete types and then turn them generic over time. This can work, but keep in mind that such changes are not necessarily backward compatible. To see why, imagine that you change a function from fn foo(v: &Vec<usize>) to fn foo(v: impl AsRef<[usize]>). While every &Vec<usize> implements AsRef<[usize]>, type inference can still cause issues for users. Consider what happens if the caller invokes foo with foo(&iter.collect()). In the original version, the com- piler could determine that it should collect into a Vec, but now it just knows that it needs to collect into some type that implements AsRef<[usize]>. And there could be multiple such types, so with this change, the caller’s code will no longer compile! 

***Object Safety*** 

When you define a new trait, whether or not that trait is object-safe (see the end of “Compilation and Dispatch” in Chapter 2) is an unwritten part of the trait’s contract. If the trait is object-safe, users can treat different types that implement your trait as a single common type using dyn Trait. If it isn’t, the compiler will disallow dyn Trait for that trait. You should prefer your traits to be object-safe even if that comes at a slight cost to the ergonomics of using them (such as taking impl AsRef<str> over &str), since object safety enables new ways to use your traits. If your trait must have a generic method, consider whether its generic parameters can be on the trait itself or if its generic arguments can also use dynamic dispatch to preserve the object safety of the trait. Alternatively, you can add a where Self: Sized trait bound to that method, which makes it possible to call the method only with a concrete instance of the trait (and not through dyn Trait). You can see examples of this pattern in the Iterator and Read traits, which are object-safe but provide some additional convenience methods on concrete instances. 

There is no single answer to the question of how many sacrifices you should be willing to make to preserve object safety. My recommendation is that you consider how your trait will be used, and whether it makes sense for users to want to use it as a trait object. If you think it’s likely that users will want to use many different instances of your trait together, you should work harder to provide object safety than if you don’t think that use case makes much sense. For example, dynamic dispatch would not be useful
 for the FromIterator trait because its one method does not take self, so
 you wouldn’t be able to construct a trait object in the first place. Similarly, std::io::Seek is fairly useless as a trait object on its own, because the only thing you would be able to do with such a trait object is seek, without being able to read or write. 

> **DROP TRAIT OBJECTS** 

>  You might think that the Drop trait is also useless as a trait object, since all
>  you can do with Drop as a trait object is to drop it . But it turns out there are some libraries that specifically just want to be able to drop arbitrary types . For example, a library that offers deferred dropping of values, such as for concur- rent garbage collection or just deferred cleanup, cares only that the values can be dropped, and nothing else . Interestingly enough, the story of Drop doesn’t end there; since Rust needs to be able to drop trait objects too, every vtable contains the drop method . Effectively, every dyn Trait is also a dyn Drop . 

Remember that object safety is a part of your public interface! If you modify a trait in an otherwise backward compatible way, such as by adding a method with a default implementation, but it makes the trait not object-safe, you need to bump your major semantic ver- sion number. 

***Borrowed vs. Owned***

For nearly every function, trait, and type you define in Rust, you must decide whether it should own, or just hold a reference to, its data. Whatever decision you make will have far-reaching impli- cations for the ergonomics and performance of your interface. Luckily, these decisions very often make themselves. 

If the code you write needs ownership of the data, such as to call methods that take self or to move the data to another thread, it must store the owned data. When your code must own data, it should generally also make the caller provide owned data, rather than taking values by reference and cloning them. This leaves the caller in control of allocation, and it is upfront about the cost of using the interface in question. 

On the other hand, if your code doesn’t need to own the data, it should operate on references instead. One common exception to this rule is with small types like i32, bool, or f64, which are just as cheap to store and copy directly as to store through references. Be wary of assuming this holds true for all Copy types, though;
 [u8; 8192] is Copy, but it would be expensive to store and copy it
 all over the place. 

Of course, in the real world, things are often less clear-cut. Sometimes, you don’t know in advance whether your code will need to own the data or not. For example, String::from_utf8_lossy needs to take ownership of the byte sequence that is passed to it only if it contains invalid UTF-8 sequences. In this case, the Cow type is your friend: it lets you operate on references if the data allows, and it lets you produce an owned value if necessary. 

Other times, reference lifetimes complicate the interface so much that it becomes a pain to use. If your users are struggling to get code to compile on top of your interface, that’s a sign that you may want to (even unnecessarily) take ownership of certain pieces of data. If you do this, start with data that is cheap to clone or is not part of anything performance-sensitive before you decide to heap-allocate what might be a huge chunk of bytes. 

***Fallible and Blocking Destructors***

Types centered on I/O often need to perform cleanup when they’re dropped. This may include flushing writes to disk, closing files, or gracefully terminat- ing connections to remote hosts. The natural place to perform this cleanup is in the type’s Drop implementation. Unfortunately, once a value is dropped, we no longer have a way to communicate errors to the user except by panick- ing. A similar problem arises in asynchronous code, where we wish to finish up when there is work pending. By the time drop is called, the executor may be shutting down, and we have no way to do more work. We could try to start another executor, but that comes with its own host of problems, such as blocking in asynchronous code, as we will see in Chapter 8. 

There is no perfect solution to these problems, and no matter what we do, some applications will inevitably fall back to our Drop implementation. For that reason, we need to provide best-effort cleanup through Drop. If cleanup errors, at least we tried—we swallow the error and move on. If an executor is still available, we might spawn a future to do cleanup, but if it never gets to run, we did what we could. 

However, we ought to provide a better alternative for users who wish to leave no loose threads. We can do this by providing an explicit destruc- tor. This usually takes the form of a method that takes ownership of self and exposes any errors (using -> Result<_, _>) or asynchrony (using async fn) that are inherent to the destruction. A careful user can then use that method to gracefully tear down any associated resources. 

*Make sure you highlight the explicit destructor in your documentation!* 

As always, there’s a trade-off. The moment you add an explicit destruc- tor, you will run into two issues. First, since your type implements Drop, you can no longer move out of any of that type’s fields in the destructor. This is because Drop::drop will still be called after your explicit destructor runs, and it takes &mut self, which requires that no part of self has been moved. Second, drop takes &mut self, not self, so your Drop implementation cannot simply call your explicit destructor and ignore its result (because it doesn’t own self). There are a couple of ways around these problems, none of which are perfect. 

The first is to make your top-level type a newtype wrapper around an Option, which in turn holds some inner type that holds all of the type’s fields. You can then use Option::take in both destructors, and call the inner type’s explicit destructor only if the inner type has not already been taken. Since the inner type does not implement Drop, you can take ownership of all the fields there. The downside of this approach is that all the methods you wish to provide on the top-level type must now include code to get through the Option (which you know is always Some since drop has not yet been called) to the fields on the inner type. 

The second workaround is to make each of your fields *takeable*. You can “take” an Option by replacing it with None (which is what Option::take does), but you can do this with many other types as well. For example, you can take a Vec or HashMap by simply replacing them with their cheap-to-construct default values—std::mem::take is your friend here. This approach works great if your types have sane “empty” values but gets tedious if you must wrap nearly every field in an Option and then modify every access of those fields with a matching unwrap. 

The third option is to hold the data inside the ManuallyDrop type, which dereferences to the inner type, so there’s no need for unwraps. You can also use ManuallyDrop::take in drop to take ownership at destruction time. The pri- mary downside of this approach is that ManuallyDrop::take is unsafe. There are no safety mechanisms in place to ensure that you don’t try to use the value inside the ManuallyDrop after you’ve called take or that you don’t call take multiple times. If you do, your program will silently exhibit undefined behavior, and bad things will happen. 

Ultimately, you should choose whichever of these approaches fits your application best. I would err on the side of going with the second option, and switching to the others only if you find yourself in a sea of Options. The ManuallyDrop solution is excellent if the code is simple enough that you can eas- ily check the safety of your code, and you are confident in your ability to do so. 

### Obvious

While some users may be familiar with aspects of the implementation that underpins your interface, they are unlikely to understand all of its rules and limitations. They won’t know that it’s never okay to call foo after calling bar, or that it’s only safe to call the unsafe method baz when the moon is at a 47-degree angle and no one has sneezed in the past 18 seconds. Only if the interface makes it clear that something strange is going on will they reach for the documentation or carefully read type signatures. It’s therefore criti- cal for you to make it as easy as possible for users to understand your inter- face and as hard as possible for them to use it incorrectly. The two primary techniques at your disposal for this are your documentation and the type system, so let’s look at each of those in turn. 

*You can also take advantage of naming to suggest to the user when there’s more to
 an interface than meets the eye. If a user sees a method named* *dangerous**, chances are they will read its documentation.* 

***Documentation***

The first step to making your interfaces transparent is to write good docu- mentation. I could write an entire book dedicated to how to write documen- tation, but let’s focus on Rust-specific advice here. 

First, clearly document any cases where your code may do something unexpected, or where it relies on the user doing something beyond what’s dictated by the type signature. Panics are a good example of both of these circumstances: if your code can panic, document that fact, along with the circumstances it might panic under. Similarly, if your code might return an error, document the cases in which it does. For unsafe functions, document what the caller must guarantee in order for the call to be safe. 

Second, include end-to-end usage examples for your code on a crate and module level. These are more important than examples for specific types or methods, since they give the user a feel for how everything fits together. With a decent high-level understanding of the interface’s struc- ture, the developer may soon realize what particular methods and types do and where they should be used. End-to-end examples also give the user a starting point for customizing their usage, and they can, and often will, copy-paste the example and then modify it to suit their needs. This kind of “learning by doing” tends to work better than having them try to piece something together from the components. 

*Very method-specific examples that show that, yes, the* *len* *method indeed returns the length are unlikely to tell the user anything new about your code.* 

Third, organize your documentation. Having all your types, traits, and functions in a single top-level module makes it difficult for the user to get a sense of where to start. Take advantage of modules to group together semantically related items. Then, use intra-documentation links to interlink items. If the documentation on type A talks about trait B, then it should link to that trait right there. If you make it easy for the user to explore your interface, they are less likely to miss important connections or dependen- cies. Also consider marking parts of your interface that are not intended to be public but are needed for legacy reasons with #[doc(hidden)], so that they do not clutter up your documentation. 

And finally, enrich your documentation wherever possible. Link to external resources that explain concepts, data structures, algorithms, or other aspects of your interface that may have good explanations elsewhere. RFCs, blog posts, and whitepapers are great for this, if any are relevant. Use #[doc(cfg(..))] to highlight items that are available only under certain con- figurations so the user quickly realizes why some method that’s listed in the documentation isn’t available. Use #[doc(alias = "...")] to make types and methods discoverable under other names that users may search for them by. In the top-level documentation, point the user to commonly used modules, features, types, traits, and methods. 

***Type System Guidance***

The type system is an excellent tool to ensure that your interfaces are obvi- ous, self-documenting, and misuse-resistant. You have several techniques at your disposal that can make your interfaces very hard to misuse, and thus, make it more likely that they will be used correctly. 

The first of these is *semantic typing*, in which you add types to repre- sent the *meaning* of a value, not just its primitive type. The classic example here is for Booleans: if your function takes three bool arguments, chances are some user will mess up the order of the values and realize it only after something has gone terribly wrong. If, on the other hand, it takes three arguments of distinct two-variant enum types, the user cannot get the order wrong without the compiler yelling at them: if they attempt to pass DryRun::Yes to the overwrite argument, that will simply not work, nor will passing Overwrite::No as the dry_run argument. You can apply semantic typ- ing beyond Booleans as well. For example, a newtype around a numeric type may provide a unit for the contained value, or it could constrain raw pointer arguments to only those that have been returned by another method. 

A closely related technique is to use zero-sized types to indicate that a particular fact is true about an instance of a type. Consider, for instance, a type called Rocket that represents the state of a real rocket. Some operations (methods) on Rocket should be available no matter what state the rocket is in, but some make sense only in particular situations. It is, for example, impossible to launch a rocket if it has already been launched. Similarly, it should probably not be possible to separate the fuel tank if the rocket has not yet launched. We could model these as enum variants, but then all the methods would be available at every stage, and we’d need to introduce pos- sible panics. 

Instead, as shown in Listing 3-2, we can introduce a generic parameter on Rocket, Stage, and use it to restrict what methods are available when. 

```  rust
1 struct Grounded;
 struct Launched;
 // and so on
 struct Rocket<Stage = Grounded> {
 2 stage: std::marker::PhantomData<Stage>, } 

3 impl Default for Rocket<Grounded> {} impl Rocket<Grounded> { 

​		pub fn launch(self) -> Rocket<Launched> { }

4 impl Rocket<Launched> {
 pub fn accelerate(&mut self) { } pub fn decelerate(&mut self) { } 

} 

5 impl<Stage> Rocket<Stage> {
 pub fn color(&self) -> Color { }
 pub fn weight(&self) -> Kilograms { } 
} 
```

*Listing 3-2: Using marker types to restrict implementations*

We introduce unit types to represent each stage of the rocket 1. We don’t actually need to store the stage—only the meta-information it provides— so we store it behind a PhantomData 2 to guarantee that it is eliminated at compile time. Then, we write implementation blocks for Rocket only when it holds a particular type parameter. You can construct a rocket only on the ground (for now), and you can launch it only from the ground 3. Only when the rocket has been launched can you control its velocity 4. There are some things you can always do with the rocket, no matter what state it is in, and those we place in a generic implementation block 5. You’ll notice that with the interface designed this way, it’s simply not possible for the user to call a method at the wrong time—we have encoded the usage rules in the types themselves, and made illegal states *unrepresentable*. 

This notion extends to many other domains as well; if your function ignores a pointer argument unless a given Boolean argument is true, it’s better to combine the two arguments instead. With an enum type with one variant for false (and no pointer) and one variant for true that holds a pointer, neither the caller nor the implementer can misunderstand the rela- tionship between the two. This is a powerful idea that I highly encourage you to make use of. 

Another small but useful tool in making interfaces obvious is the #[must _use] annotation. Add it to any type, trait, or function, and the compiler will issue a warning if the user’s code receives an element of that type or trait, or calls that function, and does not explicitly handle it. You may already have seen this in the context of Result: if a function returns a Result and you do not assign its return value somewhere, you get a compiler warning. Be care- ful not to overuse this annotation, though—add it only if the user is very likely to make a mistake if they are not using the return value. 

### Constrained

Over time, some user will depend on every property of your interface, whether bug or feature. This is especially true for publicly available librar- ies where you have no control over your users. As a result, you should think carefully before you make user-visible changes. Whether you’re adding a new type, field, method, or trait implementation or changing an existing one, you want to make sure that the change will not break existing users’ code, and that you are planning to keep that change around for a while. Frequent backward incompatible changes (major version increases in semantic versioning) are sure to draw the ire of your users. 

Many backward incompatible changes are obvious, like renaming a public type or removing a public method, but some are subtler and tie in deeply with the way Rust works. Here, we’ll cover some of the thornier subtle changes and how to plan for them. You’ll see that you need to bal- ance some of these against how flexible you want your interface to be— sometimes, something’s got to give. 

***Type Modifications*** 

Removing or renaming a public type will almost certainly break some user’s code. To counter this, you’ll want to take advantage of Rust’s visibility modi- fiers, like pub(crate) and pub(in path), whenever possible. The fewer public types you have, the more freedom you have to change things later without breaking existing code. 

User code can depend on your types in more ways than just by name, though. Consider the public type in Listing 3-3 and the given use of
 that code. 

```
// in your interface
pub struct Unit;
// in user code
let u = lib::Unit;
```

*Listing 3-3: An innocent-looking public type*

Now consider what happens if you add a private field to Unit. Even though the field you add is private, the change will still break the user’s code, because the constructor they relied on has disappeared. Similarly, consider the code and use in Listing 3-4. 

```
// in your interface
pub struct Unit { pub field: bool };
// in user code
fn is_true(u: lib::Unit) -> bool {
    matches!(u, Unit { field: true })
```

} 

*Listing 3-4: User code accessing a single public field*

Here, too, adding a private field to Unit will break user code, this time because Rust’s exhaustive pattern match checking logic is able to see parts of the interface that the user cannot see. It recognizes that there are more fields, even though the user code cannot access them, and rejects the user’s pattern as incomplete. A similar issue arises if we turn a tuple struct into a regular struct with named fields: even if the fields themselves are exactly the same, any old patterns will no longer be valid for the new type definition. 

Rust provides the #[non_exhaustive] attribute to help mitigate these issues. You can add it to any type definition, and the compiler will disallow the use of implicit constructors (like lib::Unit { field1: true }) and non- exhaustive pattern matches (that is, patterns without a trailing , ..) on that type. This is a great attribute to add if you suspect that you’re likely to modify a particular type in the future. It does constrain user code though, such as by taking away users’ ability to rely on exhaustive pattern matches, so avoid adding it if you think a given type is likely to remain stable. 

***Trait Implementations*** 

As you’ll recall from Chapter 2, Rust’s coherence rules disallow multiple implementations of a given trait for a given type. Since we do not know what implementations downstream code may have added, adding a blanket implementation of an existing trait is generally a breaking change. The same holds true for implementing a foreign trait for an existing type, or 

an existing trait for a foreign type—in both cases, the owner of the foreign trait or type may simultaneously add a conflicting implementation, so this must be a breaking change. 

Removing a trait implementation is a breaking change, but implement- ing traits for a *new* type is never a problem, since no crate can have imple- mentations that conflict with that type. 

Perhaps counterintuitively, you also want to be careful about imple- menting *any* trait for an existing type. To see why, consider the code in Listing 3-5. 

```rust
// crate1 1.0 
pub struct Unit;
put trait Foo1 { fn foo(&self) }
// note that Foo1 is not implemented for Unit
// crate2; depends on crate1 1.0
use crate1::{Unit, Foo1};
trait Foo2 { fn foo(&self) }
impl Foo2 for Unit { .. }
fn main() {
  Unit.foo();
}
```

*Listing 3-5: Implementing a trait for an existing type may cause problems.*

If you add impl Foo1 for Unit to crate1 without marking it a breaking change, the downstream code will suddenly stop compiling since the call to foo is now ambiguous. This can even apply to implementations of *new* public traits, if the downstream crate uses wildcard imports (use crate1::*). You will particularly want to keep this in mind if you provide a prelude module that you instruct users to use wildcard imports for. 

Most changes to existing traits are also breaking changes, such as changing a method signature or adding a new method. Changing a method signature breaks all implementations, and probably many uses, of the trait, whereas adding a new method “just” breaks all implementations. Adding a new method with a default implementation is fine though, since existing implementations will continue to apply. 

I say “generally” and “most” here, because as interface authors, we have a tool available to us that lets us skirt some of these rules: *sealed traits*. A sealed trait is one that can be used only, and not implemented, by other crates. This immediately makes a number of breaking changes non-breaking. For example, you can add a new method to a sealed trait, since you know there are no implementations outside of the current crate to consider. Similarly, you can implement a sealed trait for new foreign types, since you know the foreign crate that defined that type cannot have added a conflicting implementation. 

Sealed traits are most commonly used for *derived* traits—traits that provide blanket implementations for types that implement particular other traits. You should seal a trait only if it does not make sense for a foreign crate to implement your trait; it severely restricts the usefulness of the trait, since downstream crates will no longer be able to implement it for their own types. You can also use sealed traits to restrict which types can be used as type arguments, such as restricting the Stage type in the Rocket example from Listing 3-2 to only the Grounded and Launched types. 

Listing 3-6 shows how to seal a trait and how to then still add implemen- tations for it in the defining crate. 

``` rust
pub trait CanUseCannotImplement: sealed::Sealed 1 { .. } mod sealed { 

pub trait Sealed {}
 2 impl<T> Sealed for T where T: TraitBounds {}
 }
 impl<T> CanUseCannotImplement for T where T: TraitBounds {} 
```

*Listing 3-6: How to seal a trait and add implementations for it*

The trick is to add a private, empty trait as a supertrait of the trait you wish to seal 1. Since the supertrait is in a private module, other crates can- not reach it and thus cannot implement it. The sealed trait requires the underlying type to implement Sealed, so only the types that we explicitly allow 2 are able to ultimately implement the trait. 

*If you do seal a trait this way, make sure you document that fact so that users do not get frustrated trying to implement the trait themselves!* 

***Hidden Contracts*** 

Sometimes, changes you make to one part of your code affect the contract elsewhere in your interface in subtle ways. The two primary ways this hap- pens are through re-exports and auto-traits. 

**Re-Exports** 

If any part of your interface exposes foreign types, then any change to one of those foreign types is *also* a change to your interface. For example, consider what happens if you move to a new major version of a dependency and expose a type from that dependency as, say, an iterator type in your interface. A user that depends on your interface may also depend directly on that dependency and expect that the type your interface provides is the same as the one by the same name in that dependency. But if you change the major version of your dependency, that is no longer true even though the *name* of the type is the same. Listing 3-7 shows an example of this. 

```
// your crate: bestiter
pub fn iter<T>() -> itercrate::Empty<T> { .. }
// their crate
struct EmptyIterator { it: itercrate::Empty<()> }
EmptyIterator { it: bestiter::iter() }
```

*Listing 3-7: Re-exports make foreign crates part of your interface contract.*

Designing Interfaces **53** 

If your crate moves from itercrate 1.0 to itercrate 2.0 but otherwise does not change, the code in this listing will no longer compile. Even though no types have changed, the compiler believes (correctly) that itercrate1.0::Empty and itercrate2.0::Empty are *different* types. Therefore, you cannot assign the latter to the former, making this a breaking change in your interface. 

To mitigate issues like this, it’s often best to wrap foreign types using the newtype pattern, and then expose only the parts of the foreign type that you think are useful. In many cases, you can avoid the newtype wrap- per altogether by using impl Trait to provide only the very minimal contract to the caller. By promising less, you make fewer changes breaking. 

**THE SEMVER TRICK** 

The itercrate example may have rubbed you the wrong way . If the Empty type did not change, then why does the compiler not allow anything that uses it to keep working, regardless of whether the code is using version 1 .0 or 2 .0 of it? The answer is  .  .  . complicated . It boils down to the fact that the Rust compiler does not assume that just because two types have the same fields, they are the same . To take a simple example of this, imagine that itercrate 2 .0 added a #[derive(Copy)] for Empty . Now, the type suddenly has different move seman- tics depending on whether you are using 1 .0 or 2 .0! And code written with one in mind won’t work with the other . 

This problem tends to crop up in large, widely used libraries, where over time, breaking changes are likely to have to happen somewhere in the crate . Unfortunately, semantic versioning happens at the crate level, not the type level, so a breaking change anywhere is a breaking change everywhere . 

But all is not lost . A few years ago, David Tolnay (the author of serde, among a vast number of other Rust contributions) came up with a neat trick to handle exactly this kind of situation . He called it “the semver trick .” The idea is simple: if some type T stays the same across a breaking change (from 1 .0 to 2 .0, say), then after releasing 2 .0, you can release a new 1 .0 minor version that depends on 2 .0 and replaces T with a re-export of T from 2 .0 . 

By doing this, you’re ensuring that there is in fact only a single type T across both major versions . This, in turn, means that any crate that depends on 1 .0 will be able to use a T from 2 .0, and vice versa . And because this happens only for types you explicitly opt into with this trick, changes that were in fact breaking will continue to be . 

**Auto-Traits** 

Rust has a handful of traits that are automatically implemented for every type depending on what that type contains. The most relevant of these for this discussion are Send and Sync, though the Unpin, Sized, and UnwindSafe 

**54** Chapter 3 

traits have similar issues. By their very nature, these add a hidden promise made by nearly every type in your interface. These traits even propagate through otherwise type-erased types like impl Trait. 

Implementations for these traits are (generally) automatically added by the compiler, but that also means that they are *not* automatically added if they no longer apply. So, if you have a public type A that contains a private type B, and you change B so that it is no longer Send, then A is now *also* not Send. That is a breaking change! 

These changes can be hard to keep track of and are often not discov- ered until a user of your interface complains that their code no longer works. To catch these cases before they happen, it’s good practice to include some simple tests in your test suite that check that all your types implement these traits the way you expect. Listing 3-8 gives an example of what such a test might look like. 

```
fn is_normal<T: Sized + Send + Sync + Unpin>() {}
#[test]
fn normal_types() {
  is_normal::<MyType>();
}
```

*Listing 3-8: Testing that a type implements a set of traits*

Notice that this test does not run any code, but simply tests that the code compiles. If MyType no longer implements Sync, the test code will not compile, and you will know that the change you just made broke the auto- trait implementation. 

**HIDING ITEMS FROM DOCUMENTATION** 

The #[doc(hidden)] attribute lets you hide a public item from your documentation without making it inaccessible to code that happens to know it is there . This is often used to expose methods and types that are needed by macros, but not by user code . How such hidden items interact with your interface contract is a matter of some debate . In general, items marked as #[doc(hidden)] are only considered part of your contract insofar as their public effects; for example, if user code may end up containing a hidden type, then whether that type is Send or not is part of the contract, whereas its name is not . Hidden inherent methods and hidden trait methods on sealed traits are not generally part of your inter- face contract, though you should make sure to state this clearly in the documen- tation for those methods . And yes, hidden items should still be documented! 

## Summary

In this chapter we’ve explored the many facets of designing a Rust inter- face, whether it’s intended for external use or just as an abstraction bound- ary between the different modules within your crate. We covered a lot of specific pitfalls and tricks, but ultimately, the high-level principles are what should guide your thinking: your interfaces should be unsurprising, flex- ible, obvious, and constrained. In the next chapter, we will dig into how to represent and handle errors in Rust code. 