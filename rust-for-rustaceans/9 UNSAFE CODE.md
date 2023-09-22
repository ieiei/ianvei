The mere mention of unsafe code often elicits strong responses from many in the Rust community, and from many of those watching Rust from the sidelines. While some maintain it’s “no big deal,” others decry it as “the reason all of Rust’s promises are a lie.” In this chapter, I hope to pull back the curtain a bit to explain what unsafe is, what it isn’t, and how you should go about using it safely. At the time of writing, and likely also when you read this, Rust’s precise requirements for unsafe code are still being determined, and even if they were all nailed down, the complete description would be beyond the scope of this book. Instead, I’ll do my best to arm you with the building blocks, intuition, and tooling you’ll need to navigate your way through most unsafe code. 

Your main takeaway from this chapter should be this: unsafe code is the mechanism Rust gives developers for taking advantage of invariants that, for whatever reason, the compiler cannot check. We’ll look at the ways in which unsafe does that, what those invariants may be, and what we can do with it as a result. 

> **INVARIANTS** 

> Throughout this chapter, I’ll be talking a lot about invariants . Invariant is just a fancy way of saying “something that must be true for your program to be correct .” For example, in Rust, one invariant is that references, using & and &mut, do not dangle—they always point to valid values . You can also have application- or library-specific invariants, like “the head pointer is always ahead of the tail pointer” or “the capacity is always a power of two .” Ultimately, invariants represent all the assumptions required for your code to be correct . However, you may not always be aware of all the invariants that your code uses, and that’s where bugs creep in . 

Crucially, unsafe code is not a way to skirt the various rules of Rust, like borrow checking, but rather a way to enforce those rules using reasoning that is beyond the compiler. When you write unsafe code, the onus is on you to ensure that the resulting code is safe. In a way, unsafe is misleading as a keyword when it is used to allow unsafe operations through unsafe {}; it’s not that the contained code *is* unsafe, it’s that the code is allowed to perform otherwise unsafe operations because in this particular context, those operations *are* safe. 

The rest of this chapter is split into four sections. We’ll start with a brief examination of how the keyword itself is used, then explore what unsafe allows you to do. Next, we’ll look at the rules you must follow in order to write safe unsafe code. Finally, I’ll give you some advice about how to actually go about writing unsafe code safely. 

### The unsafe Keyword

Before we discuss the powers that unsafe grants you, we need to talk about its two different meanings. The unsafe keyword serves a dual purpose in Rust: it marks a particular function as unsafe to call *and* it enables you to invoke unsafe functionality in a particular code block. For example, the method in Listing 9-1 is marked as unsafe, even though it contains no unsafe code. Here, the unsafe keyword serves as a warning to the caller that there are additional guarantees that someone who writes code that invokes decr must manually check. 

```
 impl<T> SomeType<T> {
	 pub unsafe fn decr(&self) {
		 self.some_usize -= 1;
	 }
 }
```

*Listing 9-1: An unsafe method that contains only safe code*


Listing 9-2 illustrates the second usage. Here, the method itself is not marked as unsafe, even though it contains unsafe code. 

```rust
impl<T> SomeType<T> {
    pub fn as_ref(&self) -> &T {
        unsafe { &*self.ptr }
    }
}
```

*Listing 9-2: A safe method that contains unsafe code*

These two listings differ in their use of unsafe because they embody different contracts. decr requires the caller to be careful when they call the method, whereas as_ref assumes that the caller *was* careful when invoking other unsafe methods (like decr). To see why, imagine that SomeType is really a reference-counted type like Rc. Even though decr only decrements a number, that decrement may in turn trigger undefined behavior through the safe method as_ref. If you call decr and then drop the second-to-last Rc of a given T, the reference count drops to zero and the T will be dropped—but the program might still call as_ref on the last Rc, and end up with a dangling reference. 

**N O T E** *Undefined behavior *describes the consequences of a program that violates invariants of the language at runtime. In general, if a program triggers undefined behavior, the outcome is entirely unpredictable. We’ll cover undefined behavior in greater detail later in this chapter.* 

Conversely, as long as there is no way to corrupt the Rc reference count using safe code, it is always safe to dereference the pointer inside the Rc the way the code for as_ref does—the fact that &self exists is proof that the pointer must still be valid. We can use this to give the caller a safe API to an otherwise unsafe operation, which is a core piece of how to use unsafe responsibly. 

For historical reasons, every unsafe fn contains an implicit unsafe block in Rust today. That is, if you declare an unsafe fn, you can always invoke any unsafe methods or primitive operations inside that fn. However, that decision is now considered a mistake, and it’s currently being reverted through the already accepted and implemented RFC 2585. This RFC warns about having an unsafe fn that performs unsafe operations without an explicit unsafe block inside it. The lint will also likely become a hard error in future editions of Rust. The idea is to reduce the “footgun radius”—if every unsafe fn is one giant unsafe block, then you might accidentally perform unsafe operations without realizing it! For example, in decr in Listing 9-1, under the current rules you could also have added *std::ptr::null() without any unsafe annotation. 

The distinction between unsafe as a marker and unsafe blocks as a mechanism to enable unsafe operations is important, because you must think about them differently. An unsafe fn indicates to the caller that they have to be careful when calling the fn in question and that they must ensure that the function’s documented safety invariants hold. 

Meanwhile, an unsafe block implies that whoever wrote that block carefully checked that the safety invariants for any unsafe operations performed inside it hold. If you want an approximate real-world analogy, unsafe fn is an unsigned contract that asks the author of calling code to “solemnly swear X, Y, and Z.” Meanwhile, unsafe {} is the calling code’s author signing off on all the unsafe contracts contained within the block. Keep that in mind as we go through the rest of this chapter. 

### Great Power

So, once you sign the unsafe contract with unsafe {}, what are you allowed to do? Honestly, not that much. Or rather, it doesn’t enable that many new features. Inside an unsafe block, you are allowed to dereference raw pointers and call unsafe fns. 

That’s it. Technically, there are a few other things you can do, like accessing mutable and external static variables and accessing fields of unions, but those don’t change the discussion much. And honestly, that’s enough. Together, these powers allow you to wreak all sorts of havoc, like turning types into one another with mem::transmute, dereferencing raw pointers that point to who knows where, casting &'a to &'static, or making types shareable across thread boundaries even though they’re not thread-safe. 

In this section, we won’t worry too much about what can go wrong with these powers. We’ll leave that for the boring, responsible, grown-up section that comes after. Instead, we’ll look at these neat shiny new toys and what we can do with them. 

**Juggling Raw Pointers** 

One of the most fundamental reasons to use unsafe is to deal with Rust’s raw pointer types: *const T and *mut T. You should think of these as more or less analogous to &T and &mut T, except that they don’t have lifetimes and are not subject to the same validity rules as their & counterparts, which we’ll discuss later in the chapter. These types are interchangeably referred to as *pointers* and *raw pointers*, mostly because many developers instinctively refer to references as pointers, and calling them raw pointers makes the distinction clearer. 

Since fewer rules apply to * than &, you can cast a reference to a pointer even outside an unsafe block. Only if you want to go the other way, from * to &, do you need unsafe. You’ll generally turn a pointer back into a reference to do useful things with the pointed-to data, such as reading or modifying its value. For that reason, a common operation to use on pointers is unsafe { &*ptr } (or &mut *). The * there may look strange as the code is just constructing a reference, not dereferencing the pointer, but it makes sense if you look at the types; if you have a *mut T and want a &mut T, then &mut ptr would just give you a &mut *mut T. You need the * to indicate that you want the mutable reference to what ptr is a pointer *to*. 

> **POINTER TYPES** 

> You may be wondering what the difference is between *mut T and *const T and std::ptr::NonNull<T> . Well, the exact specification is still being worked out, but the primary practical difference between *mut T and *const T/ NonNull<T> is that *mut T is invariant in T (see “Lifetime Variance” in Chapter 1), whereas the other two are covariant . As the names imply, *const T and NonNull<T> differ primarily in that NonNull<T> is not allowed to be a null pointer, whereas *const T is . 

> My best advice in choosing among these types is to use your intuition about whether you would have written &mut or & if you were able to name the relevant lifetime . If you would have written &, and you know that the pointer is never null, use NonNull<T> . It benefits from a cool optimization called the niche optimization: basically, since the compiler knows that the type can never be null, it can use that information to represent types like Option<NonNull<T>> without any extra overhead, since the None case can be represented by setting the NonNull to be a null pointer! The null pointer value is a niche in the NonNull<T> type . If the pointer might be null, use *const T . And if you would have written &mut T, use *mut T . 

**Unrepresentable Lifetimes** 

As raw pointers do not have lifetimes, they can be used in circumstances where the liveness of the value being pointed to cannot be expressed statically within Rust’s lifetime system, such as a self-pointer in a self-referential struct like the generators we discussed in Chapter 8. A pointer that points into self is valid for as long as self is around (and doesn’t move, which is what Pin is for), but that isn’t a lifetime you can generally name. And while the entire self-referential type may be 'static, the self-pointer isn’t—if it were static, then even if you gave away that pointer to someone else, they could continue to use it forever, even after self was gone! Take the type in Listing 9-3 as an example; here we attempt to store the raw bytes that make up a value alongside its stored representation. 

```
struct Person<'a> {
    name: &'a str,
	age: usize, 
} 

struct Parsed {
    bytes: [u8; 1024],
    parsed: Person<'???>,
} 
```
*Listing 9-3: Trying, and failing, to name the lifetime of a self-referential reference*

The reference inside Person wants to refer to data stored in bytes in Parsed, but there is no lifetime we can assign to that reference from Parsed. It’s not 'static or something like 'self (which doesn’t exist), because if Parsed is moved, the reference is no longer valid. 

Since pointers do not have lifetimes, they circumvent this problem because you don’t have to be able to name the lifetime. Instead, you just have to make sure that when you do use the pointer, it’s still valid, which is what you sign off on when you write unsafe { &*ptr }. In the example in Listing 9-3, Person would instead store a *const str and then unsafely turn that into a &str at the appropriate times when it can guarantee that the pointer is still valid. 

A similar issue arises with a type like Arc, which has a pointer to a value that’s shared for some duration, but that duration is known only at runtime when the last Arc is dropped. The pointer is kind-of, sort-of 'static, but not really—like in the self-referential case, the pointer is no longer valid when the last Arc reference goes away, so the lifetime is more like 'self. In Arc’s cousin, Weak, the lifetime is also “when the last Arc goes away,” but since a Weak isn’t an Arc, the lifetime isn’t even tied to self. So, Arc and Weak both
 use raw pointers internally. 

**Pointer Arithmetic** 

With raw pointers, you can do arbitrary pointer arithmetic, just like you can in C, by using .offset(), .add(), and .sub() to move the pointer to any byte that lives within the same allocation. This is most often used in highly space-optimized data structures, like hash tables, where storing an extra pointer for each element would add too much overhead and using slices isn’t possible. Those are fairly niche use cases, and we won’t be talking more about them in this book, but I encourage you to read the code for hashbrown::RawTable (*https://github.com/rust-lang/hashbrown/*) if you want to learn more! 

The pointer arithmetic methods are unsafe to call even if you don’t want to turn the pointer into a reference afterwards. There are a couple of reasons for this, but the main one is that it is illegal to make a pointer point beyond the end of the allocation that it originally pointed to. Doing so triggers undefined behavior, and the compiler is allowed to decide to eat your code and replace it with arbitrary nonsense that only a compiler could understand. If you do use these methods, read the documentation carefully! 

**To Pointer and Back Again** 

Often when you need to use pointers, it’s because you have some normal Rust type, like a reference, a slice, or a string, and you have to move to the world of pointers for a bit and then go back to the original normal type. Some of the key standard library types therefore provide you with a way to turn them into their raw constituent parts, such as a pointer and a length for a slice, and a way to turn them back into the whole using those same parts. For example, you can get a slice’s data pointer with as_ptr and its length with []::len. You can then reconstruct the slice by providing those same values to std::slice::from_raw_parts. Vec, Arc, and String have similar methods that return a raw pointer to the underlying allocation, and Box has Box::into_raw and Box::from_raw, which do the same thing. 

**Playing Fast and Loose with Types** 

Sometimes, you have a type T and want to treat it as some other type U. Whether that’s because you need to do lightning-fast zero-copy parsing or because you need to fiddle with some lifetimes, Rust provides you with some (very unsafe) tools to do so. 

The first and by far most widely used of these is pointer casting: you can cast a *const T to any other *const U (and the same for mut), and you don’t even need unsafe to do it. The unsafety comes into play only when you later try to use the cast pointer as a reference, as you have to assert that the raw pointer can in fact be used as a reference to the type it’s pointing to. 

This kind of pointer type casting comes in particularly handy when working with foreign function interfaces (FFI)—you can cast any Rust pointer to a *const std::ffi::c_void or *mut std::ffi::c_void, and then pass that to a C function that expects a void pointer. Similarly, if you get a void pointer from C that you previously passed in, you can trivially cast it back into its original type. 

Pointer casts are also useful when you want to interpret a sequence of bytes as plain old data—types like integers, Booleans, characters, and arrays, or #[repr(C)] structs of these—or write such types directly out as a byte stream without serialization. There are a lot of safety invariants to keep in mind if you want to try to do that, but we’ll leave that for later. 

***Calling Unsafe Functions***

Arguably unsafe’s most commonly used feature is that it enables you to call unsafe functions. Deeper down the stack, most of those functions are unsafe because they operate on raw pointers at some fundamental level, but higher up the stack you tend to interact with unsafety primarily through function calls. 

There’s really no limit to what calling an unsafe function might enable, as it is entirely up to the libraries you interact with. But *in general*, unsafe functions can be divided into three camps: those that interact with non Rust interfaces, those that skip safety checks, and those that have custom invariants. 

**Foreign Function Interfaces** 

Rust lets you declare functions and static variables that are defined in a language other than Rust using extern blocks (which we’ll discuss at length in Chapter 11). When you declare such a block, you’re telling Rust that the items appearing within it will be implemented by some external source when the final program binary is linked, such as a C library you are integrating with. Since externs exist outside of Rust’s control, they are inherently unsafe to access. If you call a C function from Rust, all bets are off—it might overwrite your entire memory contents and clobber all your neatly arranged references into random pointers into the kernel somewhere. Similarly, an extern static variable could be modified by external code at any time, and could be filled with all sorts of bad bytes that don’t reflect its declared type at all. In an unsafe block, though, you can access externs to your heart’s delight, as long as you’re willing to vouch for the other side of the extern behaving according to Rust’s rules. 

**I’ll Pass on Safety Checks** 

Some unsafe operations can be made entirely safe by introducing additional runtime checks. For example, accessing an item in a slice is unsafe since you might try to access an item beyond the length of the slice. But, given how common the operation is, it’d be unfortunate if indexing into a slice was unsafe. Instead, the safe implementation includes bounds checks that (depending on the method you use) either panic or return an Option if the index you provide is out of bounds. That way, there is no way to cause undefined behavior even if you pass in an index beyond the slice’s length. Another example is in hash tables, which hash the key you provide rather than letting you provide the hash yourself; this ensures that you’ll never try to access a key using the wrong hash. 

However, in the endless pursuit of ultimate performance, some developers may find these safety checks add just a little too much overhead in their tightest loops. To cater to situations where peak performance is paramount and the caller knows that the indexes are in bounds, many data structures provide alternate versions of particular methods without these safety checks. Such methods usually include the word unchecked in the name to indicate that they blindly trust the provided arguments to be safe and that they do not do any of those pesky, slow safety checks. Some examples are NonNull::new_unchecked, slice::get_unchecked, NonZero::new_unchecked, Arc::get _mut_unchecked, and str::from_utf8_unchecked. 

In practice, the safety and performance trade-off for unchecked methods is rarely worth it. As always with performance optimization, measure first, then optimize. 

**Custom Invariants** 

Most uses of unsafe rely on custom invariants to some degree. That is, they rely on invariants beyond those provided by Rust itself, which are specific to the particular application or library. Since so many functions fall into this category, it’s hard to give a good general summary of this class of unsafe functions. Instead, I’ll give some examples of unsafe functions with custom invariants that you may come across in practice and want to use: 

```
MaybeUninit::assume_init
```

The MaybeUninit type is one of the few ways in which you can store values that are not valid for their type in Rust. You can think of a MaybeUninit<T> as a T that may not be legal to use as a T at the moment. For example, a MaybeUninit<NonNull> is allowed to hold a null pointer, a MaybeUninit<Box> is allowed to hold a dangling heap pointer, and a MaybeUninit<bool> is allowed to hold the bit pattern for the number 3 (normally it must be 0 or 1). This comes in handy if you are constructing a value bit by bit or are dealing with zeroed or uninitialized memory that will eventually be made valid (such as by being filled through a call to std::io::Read::read). The assume_init function asserts that the MaybeUninit now holds a valid value for the type T and can therefore be used as a T. 

```
ManuallyDrop::drop
```

The ManuallyDrop type is a wrapper type around a type T that does not drop that T when the ManuallyDrop is dropped. Or, phrased differently, it decouples the dropping of the outer type (ManuallyDrop) from the dropping of the inner type (T). It implements safe access to the T through DerefMut<Target = T> but also provides a drop method (separately from the drop method of the Drop trait) to drop the wrapped T *without* dropping the ManuallyDrop. That is, the drop function takes &mut self despite dropping the T, and so leaves the ManuallyDrop behind. This comes in handy if you have to explicitly drop a value that you cannot move, such as in implementations of the Drop trait. Once that value is dropped, it is no longer safe to try to access the T, which is why the call to drop is unsafe—it asserts that the T will never be accessed again. 

```
std::ptr::drop_in_place
```

drop_in_place lets you call a value’s destructor directly through a pointer to that value. This is unsafe because the pointee will be left behind after the call, so if some code then tries to dereference the pointer, it’ll be in for a bad time! This method is particularly useful when you may want to reuse memory, such as in an arena allocator, and need to drop an old value in place without reclaiming the surrounding memory. 

```
Waker::from_raw
```

In Chapter 8 we talked about the Waker type and how it is made up of a data pointer and a RawWaker that holds a manually implemented vtable. Once a Waker has been constructed, the raw function pointers in the vtable, such as wake and drop, can be called from safe code (through Waker::wake and drop(waker), respectively). Waker::from_raw is where the asynchronous executor asserts that all the pointers in its vtable are in fact valid function pointers that follow the contract set forth in the documentation of RawWakerVTable. 

```
std::hint::unreachable_unchecked
```

The hint module holds functions that give hints to the compiler about the surrounding code but do not actually produce any machine code. The unreachable_unchecked function in particular tells the compiler that it is impossible for the program to reach a section of the code at runtime. This in turn allows the compiler to make optimizations based on that knowledge, such as eliminating conditional branches to that location. Unlike the unreachable! macro, which panics if the code does reach the line in question, the effects of an erroneous unreachable_unchecked are hard to predict. The compiler optimizations may cause peculiar and hard-to-debug behavior, not to mention that your program will continue running when something it believed to be true was not! 

```
std::ptr::{read,write}_{unaligned,volatile}
```

The ptr module holds a number of functions that let you work with *odd* pointers—those that do not meet the assumptions that Rust generally makes about pointers. The first of these functions are read_unaligned and write_unaligned, which let you access pointers that point to a T even if that T is not stored according to T’s alignment (see the section on alignment in Chapter 2). This might happen if the T is contained directly in a byte array or is otherwise packed in with other values without proper padding. The second notable pair of functions is read_volatile and write_volatile, which let you operate on pointers that don’t point to normal memory. Concretely, these functions will always access the given pointer (they won’t be cached in a register, for example, even if you read the same pointer twice in a row), and the compiler won’t reorder the volatile accesses relative to other volatile accesses. Volatile operations come in handy when working with pointers that aren’t backed by normal DRAM memory—we’ll discuss this further in Chapter 11. Ultimately, these methods are unsafe because they dereference the given pointer (and to an owned T, at that), so you as the caller need to sign off on all the contracts associated with doing so. 

```
std::thread::Builder::spawn_unchecked
```

The normal thread::spawn that we know and love requires that the provided closure is 'static. That bound stems from the fact that the spawned thread might run for an indeterminate amount of time; if we were allowed to use a reference to, say, the caller’s stack, the caller might return well before the spawned thread exits, rendering the reference invalid. Sometimes, however, you know that some non-'static value in the caller will outlive the spawned thread. This might happen if you join the thread before dropping the value in question, or if the value is dropped only strictly after you know the spawned thread will no longer use it. That’s where spawn_unchecked comes in—it does not have the 'static bound and thus lets you implement those use cases as long as you’re willing to sign the contract saying that no unsafe accesses will happen as a result. Be careful of panics, though; if the caller panics, it might drop values earlier than you planned and cause undefined behavior in the spawned thread! 

Note that all of these methods (and indeed all unsafe methods in the standard library) provide explicit documentation for their safety invariants, as should be the case for any unsafe method. 

***Implementing Unsafe Traits***

Unsafe traits aren’t unsafe to *use*, but unsafe to *implement*. This is because unsafe code is allowed to rely on the correctness (defined by the trait’s documentation) of the implementation of unsafe traits. For example, to implement the unsafe trait Send, you need to write unsafe impl Send for .... Like unsafe functions, unsafe traits generally have custom invariants that are (or at least should be) specified in the documentation for the trait. Thus, it’s difficult to cover unsafe traits as a group, so here too I’ll give some common examples from the standard library that are worth going over. 

**Send and Sync** 

The Send and Sync traits denote that a type is safe to send or share across thread boundaries, respectively. We’ll talk more about these traits in Chapter 10, but for now what you need to know is that they are auto-traits, and so they’ll usually be implemented for most types for you by the compiler. But, as tends to be the case with auto-traits, Send and Sync will not be implemented if any members of the type in question are not themselves Send or Sync. 

In the context of unsafe code, this problem occurs primarily due to raw pointers, which are neither Send nor Sync. At first glance, this might seem reasonable: the compiler has no way to know who else may have a raw pointer to the same value or how they may be using it at the moment, so how can the type be safe to send across threads? Now that we’re seasoned unsafe developers though, that argument seems weak—after all, dereferencing a raw pointer is already unsafe, so why should handling the invariants of Send and Sync be any different? 

Strictly speaking, raw pointers could be both Send and Sync. The problem is that if they were, the types that contain raw pointers would automatically be Send and Sync themselves, even though their author might not realize that was the case. The developer might then unsafely dereference the raw pointers without ever thinking about what would happen if those types were sent or shared across thread boundaries, and thus inadvertently introduce undefined behavior. Instead, the raw pointer types block these automatic implementations as an additional safeguard to unsafe code to make authors explicitly sign the contract that they have also followed the Send and Sync invariants. 

**NOTE** *A common mistake with unsafe implementations of* *Send* *and* *Sync* *is to forget to add bounds to generic parameters:* *unsafe impl Send for MyUnsafeType {}**.* 

**GlobalAlloc** 

The GlobalAlloc trait is how you implement a custom memory allocator in Rust. We won’t talk too much about that topic in this book, but the trait itself is interesting. Listing 9-4 gives the required methods for the GlobalAlloc trait. 

```
pub unsafe trait GlobalAlloc {
    pub unsafe fn alloc(&self, layout: Layout) -> *mut u8;
    pub unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);
} 
```
*Listing 9-4: The *GlobalAlloc* trait with its required methods*

At its core, the trait has one method for allocating a new chunk of memory, alloc, and one for deallocating a chunk of memory, dealloc. The Layout argument describes the type’s size and alignment, as we discussed in Chapter 2. Each of those methods is unsafe and carries a number of safety invariants that its callers must uphold. 

GlobalAlloc itself is also unsafe because it places restrictions on the implementer of the trait, not the caller of its methods. Only the unsafety of the trait ensures that implementers agree to uphold the invariants that Rust itself assumes of its memory allocator, such as in the standard library’s implementation of Box. If the trait was not unsafe, an implementer could safely implement GlobalAlloc in a way that produced unaligned pointers or incorrectly sized allocations, which would trigger unsafety in otherwise safe code that assumes that allocations are sane. This would break the rule that safe code should not be able to trigger memory unsafety in other safe code, and thus cause all sorts of mayhem. 

**Surprisingly Not Unpin** 

The Unpin trait is not unsafe, which comes as a surprise to many Rust developers. It may even come as a surprise to you after reading Chapter 8. After all, the trait is supposed to ensure that self-referential types aren’t invalidated if they’re moved after they have established internal pointers (that is, after they’ve been placed in a Pin). It seems strange, then, that Unpin can be used to safely remove a type from a Pin. 

There are two main reasons why Unpin isn’t an unsafe trait. First, it’s unnecessary. Implementing Unpin for a type that you control does not grant you the ability to safely pin or unpin a !Unpin type; that still requires unsafety in the form of a call to Pin::new_unchecked or Pin::get_unchecked_mut. Second, there is already a safe way for you to unpin any type you control: the Drop trait! When you implement Drop for a type, you’re passed &mut self, even if your type was previously stored in a Pin and is !Unpin, all without any unsafety. That potential for unsafety is covered by the invariants of Pin::new_unchecked, which must be upheld to create a Pin of such an !Unpin type in the first place. 

**When to Make a Trait Unsafe** 

Few traits in the wild are unsafe, but those that are all follow the same pattern. A trait should be unsafe if safe code that assumes that trait is implemented correctly can exhibit memory unsafety if the trait is *not* implemented correctly. 

The Send trait is a good example to keep in mind here—safe code can easily spawn a thread and pass a value to that spawned thread, but if Rc were Send, that sequence of operations could trivially lead to memory unsafety. Consider what would happen if you cloned an Rc<Box> and sent it to another thread: the two threads could easily both try to deallocate the Box since they do not correctly synchronize access to the Rc’s reference count. 

The Unpin trait is a good counterexample. While it is possible to write unsafe code that triggers memory unsafety if Unpin is implemented incorrectly, no entirely safe code can trigger memory unsafety due to an implementation of Unpin. It’s not always easy to determine that a trait can be safe (indeed, the Unpin trait was unsafe throughout most of the RFC process), but you can always err on the side of making the trait unsafe, and then make it safe later on if you realize that is the case! Just keep in mind that that is a backward incompatible change. 

Also keep in mind that just because it feels like an incorrect (or even malicious) implementation of a trait would cause a lot of havoc, that’s not necessarily a good reason to make it unsafe. The unsafe marker should first and foremost be used to highlight cases of *memory* unsafety, not just something that can trigger errors in business logic. For example, the Eq, Ord, Deref, and Hash traits are all safe, even though there is likely much code out in the world that would go haywire if faced with a malicious implementation of, say, Hash that returned a different random hash each time it was called. This extends to unsafe code too—there is almost certainly unsafe code out there that would be memory-unsafe in the presence of such an implementation of Hash—but that does not mean Hash should be unsafe. The same is true for an implementation of Deref that dereferenced to a different (but valid) target each time. Such unsafe code would be relying on a contract of Hash or Deref that does not actually hold; Hash never claimed that it was deterministic, and neither did Deref. Or rather, the authors of those implementations never used the unsafe keyword to make that claim! 

**N O T E**  *An important implication of traits like* *Eq**,* *Hash**, and* *Deref* *being safe is that unsafe code can rely only on the* safety *of safe code, not its correctness. This applies not only to traits, but to all unsafe/safe code interactions.* 

### Great Responsibility

So far, we’ve looked mainly at the various things that you are allowed to do with unsafe code. But unsafe code is allowed to do those things only if it does so safely. Even though unsafe code can, say, dereference a raw pointer, it must do so only if it knows that pointer is valid as a reference to its pointee at that moment in time, subject to all of Rust’s normal requirements of references. In other words, unsafe code is given access to tools that could be used to do unsafe things, but it must do only safe things using those tools. 

That, then, raises the question of what *safe* even means in the first place. When is it safe to dereference a pointer? When is it safe to transmute between two different types? In this section, we’ll explore some of the key invariants to keep in mind when wielding the power of unsafe, look at some common gotchas, and get familiar with some of the tools that help you write safer unsafe code. 

The exact rules around what it means for Rust code to be safe are still being worked out. At the time of writing, the Unsafe Code Guidelines Working Group is hard at work nailing down all the dos and don’ts, but many questions remain unanswered. Most of the advice in this section is more or less settled, but I’ll make sure to call out any that isn’t. If anything, I’m hoping that this section will teach you to be careful about making assumptions when you write unsafe code, and prompt you to double-check the Rust reference before you declare your code production-ready. 

**What Can Go Wrong?** 

We can’t really get into the rules unsafe code must abide by without talking about what happens if you violate those rules. Let’s say you do mutably access a value from multiple threads concurrently, construct an unaligned reference, or dereference a dangling pointer—now what? 

Unsafe code that is not ultimately safe is referred to as having *undefined behavior*. Undefined behavior generally manifests in one of three ways: not at all, through visible errors, or through invisible corruption. The first is the happy case—you wrote some code that is truly not safe, but the compiler generated sane code that the computer you’re running the code on executes in a sane way. Unfortunately, the happiness here is very brittle. Should a new and slightly smarter version of the compiler come along, or some surrounding code cause the compiler to apply another optimization, the code may no longer do something sane and tip over into one of the worse cases. Even if the same code is compiled by the same compiler, if it runs on a different platform or host, the program might act differently! This is why it is important to avoid undefined behavior even if everything currently seems to work fine. Not to do so is like playing a second round of Russian roulette just because you survived the first. 

Visible errors are the easiest undefined behavior to catch. If you dereference a null pointer, for example, your program will (in all likelihood) crash with an error, which you can then debug back to the root cause. That debugging may itself be difficult, but at least you have a notification that something is wrong. Visible errors can also manifest in less severe ways, such as deadlocks, garbled output, or panics that are printed but don’t trigger a program exit, all of which tell you that there is a bug in your code that you have to go fix. 

The worst manifestation of undefined behavior is when there is no immediate visible effect, but the program state is invisibly corrupted. Transaction amounts might be slightly off from what they should be, backups might be silently corrupted, or random bits of internal memory could be exposed to external clients. The undefined behavior could cause ongoing corruption, or extremely infrequent outages. Part of the challenge with undefined behavior is that, as the name implies, the behavior of the nonsafe unsafe code is not defined—the compiler might eliminate it entirely, dramatically change the semantics of the code, or even miscompile surrounding code. What that does to your program is entirely dependent on what the code in question does. The unpredictable impact of undefined behavior is the reason why *all* undefined behavior should be considered a serious bug, no matter how it *currently* manifests. 

> **WHY UNDEFINED BEHAVIOR?** 

>  An argument that often comes up in conversations about undefined behavior is that the compiler should emit an error if code exhibits undefined behavior instead of doing something weird and unpredictable . That way, it would be near impossible to write bad unsafe code! 

> Unfortunately, that would be impossible because undefined behavior is rarely explicit or obvious . Instead, what usually happens is that the compiler simply applies optimizations under the assumption that the code follows the specification . Should that turn out to not be the case—which is rarely clear until runtime—it’s difficult to predict what the effect might be . Maybe the optimization is still valid, and nothing bad happens; but maybe it’s not, and the semantics of the code end up slightly different from that of the unoptimized version . 

> If we were to tell compiler developers that they aren’t allowed to assume anything about the underlying code, what we’d really be telling them is that they cannot perform a wide range of the optimizations that they implement with great success today . Nearly all sophisticated optimizations make assumptions about what the code in question can and cannot do according to the language specification . 

> If you want a good illustration of how specifications and compiler optimizations interact in strange ways where it’s hard to assign blame, I recommend reading through Ralf Jung’s blog post “We Need Better Language Specs” (https://www.ralfj.de/blog/2020/12/14/provenance.html) . 

**Validity** 

Perhaps the most important concept to understand before writing unsafe code is *validity*, which dictates the rules for what values inhabit a given type—or, less formally, the rules for a type’s values. The concept is simpler than it sounds, so let’s dive into some concrete examples. 

**Reference Types** 

Rust is very strict about what values its reference types can hold. Specifically, references must never dangle, must always be aligned, and must always point to a valid value for their target type. In addition, a shared and an exclusive reference to a given memory location can never exist at the same time, and neither can multiple exclusive references to a location. These rules apply regardless of whether your code uses the references or not—you are not allowed to create a null reference even if you then immediately discard it! 

Shared references have the additional constraint that the pointee is not allowed to change during the reference’s lifetime. That is, any value the pointee contains must remain exactly the same over its lifetime. This applies transitively, so if you have an & to a type that contains a *mut T, you are not allowed to ever mutate the T through that *mut even though you could write code to do so using unsafe. The *only* exception to this rule is a value wrapped by the UnsafeCell type. All other types that provide interior mutability, like Cell, RefCell, and Mutex, internally use an UnsafeCell. 

An interesting result of Rust’s strict rules for references is that for many years, it was impossible to safely take a reference to a field of a packed or partially uninitialized struct that used repr(Rust). Since repr(Rust) leaves a type’s layout undefined, the only way to get the address of a field was by writing &some_struct.field as *const _. However, if some_struct is packed, then some_struct.field may not be aligned, and thus creating an & to it is illegal! Further, if some_struct isn’t fully initialized, then the some_struct reference itself cannot exist! In Rust 1.51.0, the ptr::addr_of! macro was stabilized, which added a mechanism for directly obtaining a reference to a field without first creating a reference, fixing this particular problem. Internally, it is implemented using something called *raw references* (not to be confused with raw pointers), which directly create pointers to their operands rather than going via a reference. Raw references were introduced in RFC 2582 but haven’t been stabilized themselves yet at the time of writing. 

**Primitive Types** 

Some of Rust’s primitive types have restrictions on what values they can hold. For example, a bool is defined as being 1 byte large but is only allowed to hold the value 0x00 or the value 0x01, and a char is not allowed to hold a surrogate or a value above char::MAX. Most of Rust’s primitive types, and indeed most of Rust’s types overall, also cannot be constructed from uninitialized memory. These restrictions may seem arbitrary, but again often stem from the need to enable optimizations that wouldn’t be possible otherwise. 

A good illustration of this is the niche optimization, which we discussed briefly when talking about pointer types earlier in this chapter. To recap, the niche optimization tucks away the enum discriminant value in the wrapped type in certain cases. For example, since a reference cannot ever be all zeros, an Option<&T> can use all zeros to represent None, and thus avoid spending an extra byte (plus padding) to store the discriminator byte. The compiler can optimize Booleans in the same way and potentially take it even further. Consider the type Option<Option<bool>>>. Since the compiler knows that the bool is either 0x00 or 0x01, it’s free to use 0x02 to represent Some(None) and 0x03 to represent None. Very nice and tidy! But if someone were to come along and treat the byte 0x03 as a bool, and then place that value in an Option<Option<bool>> optimized in this way, bad things would happen. 

It bears repeating that it’s not important whether the Rust compiler currently implements this optimization or not. The point is that it is allowed to, and therefore any unsafe code you write must conform to that contract or risk hitting a bug later on should the behavior change. 

**Owned Pointer Types** 

Types that point to memory they own, like Box and Vec, are generally subject to the same optimizations as if they held an exclusive reference to the pointed-to memory unless they’re explicitly accessed through a shared reference. Specifically, the compiler assumes that the pointed-to memory is not shared or aliased elsewhere, and makes optimizations based on that assumption. For example, if you extracted the pointer from a Box and then constructed two Boxes from that same pointer and wrapped them in ManuallyDrop to prevent a double-free, you’d likely be entering undefined behavior territory. That’s the case even if you only ever access the inner type through shared references. (I say “likely” because this isn’t fully settled in the language reference yet, but a rough consensus has arisen.) 

**Storing Invalid Values** 

Sometimes you need to store a value that isn’t currently valid for its type. The most common example of this is if you want to allocate a chunk of memory for some type T and then read in the bytes from, say, the network. Until all the bytes have been read in, the memory isn’t going to be a valid T. Even if you just tried to read the bytes into a slice of u8, you would have to zero those u8s first, because constructing a u8 from uninitialized memory is also undefined behavior. 

The MaybeUninit<T> type is Rust’s mechanism for working with values that aren’t valid. A MaybeUninit<T> stores exactly a T (it is #[repr(transparent)]), but the compiler knows to make no assumptions about the validity of that T. It won’t assume that references are non-null, that a Box<T> isn’t dangling, or that a bool is either 0 or 1. This means it’s safe to hold a T backed by uninitialized memory inside a MaybeUninit (as the name implies). MaybeUninit is also a very useful tool in other unsafe code where you have to temporarily store a value that may be invalid. Maybe you have to store an aliased Box<T> or stash a char surrogate for a second—MaybeUninit is your friend. 

You will generally do only three things with a MaybeUninit: create it using the MaybeUninit::uninit method, write to its contents using MaybeUninit::as _mut_ptr, or take the inner T once it is valid again with MaybeUninit::assume_init. As its name implies, uninit creates a new MaybeUninit<T> of the same size as a T that initially holds uninitialized memory. The as_mut_ptr method gives you a raw pointer to the inner T that you can then write to; nothing stops you from reading from it, but reading from any of the uninitialized bits is undefined behavior. And finally, the unsafe assume_init method consumes the MaybeUninit<T> and returns its contents as a T following the assertion that the backing memory now makes up a valid T. 

Listing 9-5 shows an example of how we might use MaybeUninit to safely initialize a byte array without explicitly zeroing it. 

```
fn fill(gen: impl FnMut() -> Option<u8>) {
    let mut buf = [MaybeUninit::<u8>::uninit(); 4096];
    let mut last = 0;
    for (i, g) in std::iter::from_fn(gen).take(4096).enumerate() {
        buf[i] = MaybeUninit::new(g);
		last = i + 1; } 
	    // Safety: all the u8s up to last are initialized.
	let init: &[u8] = unsafe {
		MaybeUninit::slice_assume_init_ref(&buf[..last])
	}; 
 } 
```
*Listing 9-5: Using *MaybeUninit* to safely initialize an array*

While we could have declared buf as [0; 4096] instead, that would require the function to first write out all those zeros to the stack before executing, even if it’s going to overwrite them all again shortly thereafter. Normally that wouldn’t have a noticeable impact on performance, but if this was in a sufficiently hot loop, it might! Here, we instead allow the array to keep whatever values happened to be on the stack when the function was called, and then overwrite only what we end up needing. 

**N O T E** *Be careful with dropping partially initialized memory. If a panic causes an unexpected early drop before the* *MaybeUninit* *has been fully initialized, you’ll have to take care to drop only the parts of* *T* *that are now valid, if any. You* can *just drop the* *MaybeUninit* *and have the backing memory forgotten, but if it holds, say, a* *Box**, you might end up with a memory leak!* 

***Panics***

An important and often overlooked aspect of ensuring that code using unsafe operations is safe is that the code must also be prepared to handle panics. In particular, as we discussed briefly in Chapter 5, Rust’s default panic handler on most platforms will not crash your program on a panic but will instead *unwind* the current thread. An unwinding panic effectively drops everything in the current scope, returns from the current function, drops everything in the scope that enclosed the function, and so on, all the way down the stack until it hits the first stack frame for the current thread. If you don’t take unwinding into account in your unsafe code, you may be in for trouble. For example, consider the code in Listing 9-6, which tries to efficiently push many values into a Vec at once. 

```
impl<T: Default> Vec<T> {
    pub fn fill_default(&mut self) {
        let fill = self.capacity() - self.len();
        if fill == 0 { return; }
        let start = self.len();
        unsafe {
// ... do something with init ...

	self.set_len(start + n);
            for i in 0..fill {
                *self.get_unchecked_mut(start + i) = T::default();
	}}
}}
```

Listing 9-6: A seemingly safe method for filling a vector with *Default* values 

Consider what happens to this code if a call to T::default panics. First, fill_default will drop all its local values (which are just integers) and then return. The caller will then do the same. At some point up the stack, we get to the owner of the Vec. When the owner drops the vector, we have a problem: the length of the vector now indicates that we own more Ts than we actually produced due to the call to set_len. For example, if the very first call to T::default panicked when we aimed to fill eight elements, that means Vec::drop will call drop on eight Ts that actually contain uninitialized memory! 

The fix in this case is simple: the code must update the length *after* writing all the elements. We wouldn’t have realized there was a problem if we didn’t carefully consider the effect of unwinding panics on the correctness of our unsafe code. 

When you’re combing through your code for these kinds of problems, you’ll want to look out for any statements that may panic, and consider whether your code is safe if they do. Alternatively, check whether you can convince yourself that the code in question will never panic. Pay particular attention to anything that calls user-provided code—in those cases, you have no control over the panics and should assume that the user code will panic. 

A similar situation arises when you use the ? operator to return early from a function. If you do this, make sure that your code is still safe if it does not execute the remainder of the code in the function. It’s rarer for ? to catch you off guard since you opted into it explicitly, but it’s worth keeping an eye out for. 

**Casting** 

As we discussed in Chapter 2, two different types that are both #[repr(Rust)] may be represented differently in memory even if they have fields of the same type and in the same order. This in turn means that it’s not always obvious whether it is safe to cast between two different types. In fact, Rust doesn’t even guarantee that two instances of a single type with generic arguments that are themselves laid out the same way are represented the same way. For example, in Listing 9-7, A and B are not guaranteed to have the same in-memory representation. 

```
struct Foo<T> {
    one: bool,
    two: PhantomData<T>,
}
struct Bar;
struct Baz;
type A = Foo<Bar>;
type B = Foo<Baz>;
```

Listing 9-7: Type layout is not predictable. 

The lack of guarantees for repr(Rust) is important to keep in mind when you do type casting in unsafe code—just because two types feel like they should be interchangeable, that is not necessarily the case. Casting between two types that have different representations is a quick path to undefined behavior. At the time of writing, the Rust community is actively working out the exact rules for how types are represented, but for now, very few guarantees are given, so that’s what we have to work with. 

Even if identical types were guaranteed to have the same in-memory representation, you’d still run into the same problem when types are nested. For example, while UnsafeCell<T>, MaybeUninit<T>, and T all really just hold a T, and you can cast between them to your heart’s delight, that goes out the window once you have, for example, an Option<MaybeUninit<T>>. Though Option<T> may be able to take advantage of the niche optimization (using some invalid value of T to represent None for the Option), MaybeUninit<T> can hold any bit pattern, so that optimization does not apply, and an extra byte must be kept for the Option discriminator. 

It’s not just optimizations that can cause layouts to diverge once wrapper types come into play. As an example, take the code in Listing 9-8; here, the layout of Wrapper<PhantomData<u8>> and Wrapper<PhantomData<i8>> is completely different even though the provided types are both empty! 

```
struct Wrapper<T: SneakyTrait> {
    item: T::Sneaky,
    iter: PhantomData<T>,
}
trait SneakyTrait {
    type Sneaky;
}
impl SneakyTrait for PhantomData<u8> {
    type Sneaky = ();
}
impl SneakyTrait for PhantomData<i8> {
    type Sneaky = [u8; 1024];
} 
```

*Listing 9-8: Wrapper types make casting hard to get right.*

All of this isn’t to say that you can never cast types in Rust. Things get a lot easier, for example, when you control all of the types involved and their trait implementations, or if types are #[repr(C)]. You just need to be aware that Rust gives very few guarantees about in-memory representations, and write your code accordingly! 

**The Drop Check** 

The Rust borrow checker is, in essence, a sophisticated tool for ensuring the soundness of code at compile time, which is in turn what gives Rust a way to express code being “safe.” How exactly the borrow checker does its job is beyond the scope of this book, but one check, the *drop check*, is worth going through in some detail since it has some direct implications for unsafe code. To understand drop checking, let’s put ourselves in the Rust compiler’s shoes for a second and look at two code snippets. First, take a look at the little three-liner in Listing 9-9 that takes a mutable reference to a variable and then mutates that same variable right after. 

```
let mut x = true;
let foo = Foo(&mut x);
x = false;
```

Listing 9-9: The implementation of *Foo* dictates whether this code should compile 

Without knowing the definition of Foo, can you say whether this code should compile or not? When we set x = false, there is still a foo hanging around that will be dropped at the end of the scope. We know that foo contains a mutable borrow of x, which would indicate that the mutable borrow that’s necessary to modify x is illegal. But what’s the harm in allowing it? It turns out that allowing the mutation of x is problematic only if Foo implements Drop—if Foo doesn’t implement Drop, then we know that Foo won’t touch the reference to x after its last use. Since that last use is before we need the exclusive reference for the assignment, we can allow the code! On the other hand, if Foo does implement Drop, we can’t allow this code, since the Drop implementation may use the reference to x. 

Now that you’re warmed up, take a look at Listing 9-10. In this not-sostraightforward code snippet, the mutable reference is buried even deeper. 

```
fn barify<’a>(_: &’a mut i32) -> Bar<Foo<’a>> { .. }
let mut x = true;
let foo = barify(&mut x);
x = false;
```

Listing 9-10: The implementations of both *Foo* and *Bar* dictate whether this code should compile 

Again, without knowing the definitions of Foo and Bar, can you say whether this code should compile or not? Let’s consider what happens if Foo implements Drop but Bar does not, since that’s the most interesting case. Usually, when a Bar goes out of scope, or otherwise gets dropped, it’ll still have to drop Foo, which in turn means that the code should be rejected for the same reason as before: Foo::drop might access the reference to x. However, Bar may not contain a Foo directly at all, but instead just a PhantomData<Foo<'a>> or a &'static Foo<'a>, in which case the code is actually okay—even though the Bar is dropped, Foo::drop is never invoked, and the reference to x is never accessed. This is the kind of code we want the compiler to accept because a human will be able to identify that it’s okay, even if the compiler finds it difficult to detect that this is the case. 

The logic we’ve just walked through is the drop check. Normally it doesn’t affect unsafe code too much as its default behavior matches user expectations, with one major exception: dangling generic parameters. Imagine that you’re implementing your own Box<T> type, and someone places a &mut x into it as we did in Listing 9-9. Your Box type needs to implement Drop to free memory, but it doesn’t access T beyond dropping it. Since dropping a &mut does nothing, it should be entirely fine for code to access &mut x again after the last time the Box is accessed but before it’s dropped! To support types like this, Rust has an unstable feature called dropck_eyepatch (because it makes the drop check partially blind). The feature is likely to remain unstable forever and is intended to serve only as a temporary escape hatch until a proper mechanism is devised. The dropck_eyepatch feature adds a #[may_dangle] attribute, which you can add as a prefix for generic lifetimes and types in a type’s Drop implementation to tell the drop check machinery that you won’t use the annotated lifetime or type beyond drop- ping it. You use it by writing: 

```
unsafe impl<#[may_dangle] T> Drop for ..
```

This escape hatch allows a type to declare that a given generic parameter isn’t used in Drop, which enables use cases like Box<&mut T>. However, it also introduces a new problem if your Box<T> holds a raw heap pointer, *mut T, and allows T to dangle using #[may_dangle]. Specifically, the *mut T makes Rust’s drop check think that your Box<T> doesn’t own a T, and thus that it doesn’t call T::drop either. Combined with the may_dangle assertion that we don’t access T when the Box<T> is dropped, the drop check now con- cludes that it’s fine to have a Box<T> where the T doesn’t live until the Box is dropped (like our shortened &mut x in Listing 9-10). But that’s not true, since we *do* call T::drop, which may itself access, say, a reference to said x. 

Luckily, the fix is simple: we add a PhantomData<T> to tell the drop check that even though the Box<T> doesn’t hold any T, and won’t access T on drop, it does still own a T and will drop one when the Box is dropped. Listing 9-11 shows what our hypothetical Box type would look like. 

```
struct Box<T> {
  t: NonNull<T>, // NonNull not *mut for covariance (Chapter 1)
  _owned: PhantomData<T>, // For drop check to realize we drop a T
}
unsafe impl<#[may_dangle] T> for Box<T> { /* ... */ }
```

Listing 9-11: A definition for *Box* that is maximally flexible in terms of the drop check 

This interaction is subtle and easy to miss, but it arises only when you use the unstable #[may_dangle] attribute. Hopefully this subsection will serve as a warning so that when you see unsafe impl Drop in the wild in the future, you’ll know to look for a PhantomData<T> as well! 

**N O T E** *Another consideration for unsafe code concerning* *Drop* *is to make sure that you have a* *Type* *that lets* *T* *continue to live after* *self* *is dropped. For example, if you’re implementing delayed garbage collection, you need to also add* *T: 'static**. Otherwise, if* *T = WriteOnDrop<&mut U>**, the later access or drop of* *T* *could trigger undefined behavior!* 

### Coping with Fear

With this chapter mostly behind you, you may now be more afraid of unsafe code than you were before you started. While that is understandable, it’s important to stress that it’s not only *possible* to write safe unsafe code, but most of the time it’s not even that difficult. The key is to make sure that you handle unsafe code with care; that’s half the struggle. And be really sure that there isn’t a safe implementation you can use instead before resorting to unsafe. 

In the remainder of this chapter, we’ll look at some techniques and tools that can help you be more confident in the correctness of your unsafe code when there’s no way around it. 

***Manage Unsafe Boundaries***

It’s tempting to reason about unsafety *locally*; that is, to consider whether the code in the unsafe block you just wrote is safe without thinking too much about its interaction with the rest of the codebase. Unfortunately, that kind of local reasoning often comes back to bite you. A good example of this is the Unpin trait—you may write some code for your type that uses Pin::new_unchecked to produce a pinned reference to a field of the type, and that code may be entirely safe when you write it. But then at some later point in time, you (or someone else) might add a safe implementation of Unpin for said type, and suddenly the unsafe code is no longer safe, even though it’s nowhere near the new impl! 

Safety is a property that can be checked only at the privacy boundary of all code that relates to the unsafe block. *Privacy boundary* here isn’t so much a formal term as an attempt at describing “any part of your code that can fiddle with the unsafe bits.” For example, if you declare a public type Foo in a module bar that is marked pub or pub(crate), then any other code in the same crate can implement methods on and traits for Foo. So, if the safety of your unsafe code depends on Foo not implementing particular traits or methods with particular signatures, you need to remember to recheck the safety of that unsafe block any time you add an impl for Foo. If, on the other hand, Foo is not visible to the entire crate, then a much smaller set of scopes is able to add problematic implementations, and thus, the risk of accidentally adding an implementation that breaks the safety invariants goes down accordingly. If Foo is private, then only the current module and any submodules can add such implementations. 

The same rule applies to access to fields: if the safety of an unsafe block depends on certain invariants over a type’s fields, then any code that can touch those fields (including safe code) falls within the privacy boundary of the unsafe block. Here, too, minimizing the privacy boundary is the best approach—code that cannot get to the fields cannot mess up your invariants!  

Because unsafe code often requires this wide-reaching reasoning, it’s best practice to encapsulate the unsafety in your code as best you can. Provide the unsafety in the form of a single module, and strive to give that module an interface that is entirely safe. That way you only need to audit the internals of that module for your invariants. Or better yet, stick the unsafe bits in their own crate so that you can’t leave any holes open by accident! 

It’s not always possible to fully encapsulate complex unsafe interactions to a single, safe interface, however. When that’s the case, try to narrow down the parts of the public interface that have to be unsafe so that you have only a very small number of them, give them names that clearly communicate that care is needed, and then document them rigorously. 

It is sometimes tempting to remove the unsafe marker on internal APIs so that you don’t have to stick unsafe {} throughout your code. After all, inside your code you know never to invoke frobnify if you’ve previously called bazzify, right? Removing the unsafe annotation can lead to cleaner code but is usually a bad decision in the long run. A year from now, when your codebase has grown, you’ve paged out some of the safety invariants, and you “just want to hack together this one feature real quick,” chances are that you’ll inadvertently violate one of those invariants. And since you don’t have to type unsafe, you won’t even think to check. Plus, even if you never make mistakes, what about other contributors to your code? Ultimately, cleaner code is not a good enough argument to remove the intentionally noisy unsafe marker. 

***Read and Write Documentation***

It goes without saying that if you write an unsafe function, you must document the conditions under which that function is safe to call. Here, both clarity and completeness are important. Don’t leave any invariants out, even if you’ve already written them somewhere else. If you have a type or module that requires certain global invariants—invariants that must always hold for all uses of the type—then remind the reader that they must also uphold the global invariants in every unsafe function’s documentation too. Developers often read documentation in an ad hoc, on-demand manner, so you can assume they have probably not read your carefully written module-level documentation and need to be given a nudge to do so. 

What may be less obvious is that you should also document all unsafe implementations and blocks—think of this as providing proof that you do indeed uphold the contract the operation in question requires. For example, slice::get_unchecked requires that the provided index is within the bounds of the slice; when you call that method, put a comment just above it explaining how you know that the index is in fact guaranteed to be in bounds. In some cases, the invariants that the unsafe block requires are extensive, and your comments may get long. That’s a good thing. I have caught mistakes many times by trying to write the safety comment for an unsafe block and realizing halfway through that I actually don’t uphold a key invariant. You’ll also thank yourself a year down the road when you have to modify this code and ensure it’s still safe. And so will the contributor to your project who just stumbled across this unsafe call and wants to understand what’s going on. 

Before you get too deep into writing unsafe code, I also highly recommend that you go read the Rustonomicon (*https://doc.rust-lang.org/nomicon/*) cover to cover. There are so many details that are easy to miss, and will come back to bite you if you’re not aware of them. We’ve covered many of them in this chapter, but it never hurts to be more aware. You should also make liberal use of the Rust reference whenever you’re in doubt. It’s added to regularly, and chances are that if you’re even slightly unsure about whether some assumption you have is right, the reference will call it out. If it doesn’t, consider opening an issue so that it’ll be added! 

***Check Your Work***

Okay, so you’ve written some unsafe code, you’ve double- and triple-checked all the invariants, and you think it’s ready to go. Before you put it into production, there are some automated tools that you should run your test suite through (you have a test suite, right?). 

The first of these is Miri, the mid-level intermediate representation interpreter. Miri doesn’t compile your code into machine code but instead interprets the Rust code directly. This provides Miri with far more visibility into what your program is doing, which in turn allows it to check that your program doesn’t do anything obviously bad, like read from uninitialized memory. Miri can catch a lot of very subtle and Rust-specific bugs and is a lifesaver for anyone writing unsafe code. 

Unfortunately, because Miri has to interpret the code to execute it, code run under Miri often runs orders of magnitude slower than its compiled counterpart. For that reason, Miri should really be used only to execute your test suite. It can also check only the code that actually runs, and thus won’t catch issues in code paths that your test suite doesn’t reach. You should think of Miri as an extension of your test suite, not a replacement for it. 

There are also tools known as *sanitizers*, which instrument machine code to detect erroneous behavior at runtime. The overhead and fidelity of these tools vary greatly, but one widely loved tool is Google’s AddressSanitizer. It detects a large number of memory errors, such as use-after-free, buffer overflows, and memory leaks, all of which are common symptoms of incorrect unsafe code. Unlike Miri, these tools operate on machine code and thus tend to be fairly fast—usually within the same order of magnitude. But like Miri, they are constrained to analyzing the code that actually runs, so here too a solid test suite is vital. 

The key to using these tools effectively is to automate them through your continuous integration pipeline so they’re run for every change, and to ensure that you add regression tests over time as you discover errors. The tools get better at catching problems as the quality of your test suite improves, so by incorporating new tests as you fix known bugs, you’re earning double points back, so to speak! 

Finally, don’t forget to sprinkle assertions generously through unsafe code. A panic is always better than triggering undefined behavior! Check all of your assumptions with assertions if you can—even things like the size of a usize if you rely on that for safety. If you’re concerned about runtime cost, make use of the debug_assert* macros and the if cfg!(debug_assertions) || cfg!(test) construct to execute them only in debug and test contexts. 

> **A HOUSE OF CARDS?** 

> Unsafe code can violate all of Rust’s safety guarantees, and this is often touted as a reason why Rust’s whole safety argument is a charade . The concern is that it takes only one bit of incorrect unsafe code for the whole house to come crashing down and all safety to be lost . Proponents of this argument then sometimes argue that at the very least only unsafe code should be able to call unsafe code, so that the unsafety is visible all the way to the highest level of the application . 

> The argument is understandable—it is true that the safety of Rust code relies on the safety of all the transitive unsafe code it ends up invoking . And indeed, if some of that unsafe code is incorrect, it may have implications for the safety of the program overall . However, what this argument misses is that all successful safe languages provide a facility for language extensions that are not expressible in the (safe) surface language, usually in the form of code written in C or assembly . Just as Rust relies on the correctness of its unsafe code, the safety of those languages relies on the correctness of those extensions. 

> Rust is different in that it doesn’t have a separate extension language, but instead allows extensions to be written in what amounts to a dialect of Rust (unsafe Rust) . This allows much closer integration between the safe and unsafe code, which in turn reduces the likelihood of errors due to impedance mis- matches at the interface between the two, or due to developers being familiar with one but not the other . The closer integration also makes it easier to write tools that analyze the correctness of the unsafe code’s interaction with the safe code, as exemplified by tools like Miri . And since unsafe Rust continues to be subject to the borrow checker for any operation that isn’t explicitly unsafe, there remain many safety checks in place that aren’t present when developers must drop down to a language like C . 

## Summary

In this chapter, we’ve walked through the powers that come with the unsafe keyword and the responsibilities we accept by leveraging those powers. We also talked about the consequences of writing unsafe unsafe code, and how you really should be thinking about unsafe as a way to swear to the compiler that you’ve manually checked that the indicated code is still safe. In the next chapter, we’ll jump into concurrency in Rust and see how you can get all those cores on your shiny new computer to pull in the same direction! 