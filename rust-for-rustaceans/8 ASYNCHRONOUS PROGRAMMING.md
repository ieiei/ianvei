
Asynchronous programming is, as the name implies, programming that is not synchronous. At a high level, an asynchronous operation is one that executes in the background—the program won’t wait for the asynchronous operation to complete but will instead continue to the next line of code immediately. 

If you’re not already familiar with asynchronous programming, that definition may feel insufficient as it doesn’t actually explain what asynchronous programming *is*. To really understand the asynchronous programming model and how it works in Rust, we have to first dig into what the alter- native is. That is, we need to understand the *synchronous* programming model before we can understand the *asynchronous* one. This is important in both clarifying the concepts and demonstrating the trade-offs of using asynchronous programming: an asynchronous solution is not always the right one! We’ll start this chapter by taking a quick journey through what motivates asynchronous programming as a concept in the first place; then we’ll dig into how asynchrony in Rust actually works under the hood. 

**What’s the Deal with Asynchrony?** 

Before we get to the details of the synchronous and asynchronous program- ming models, we first need to take a quick look at what your computer is actually doing when it runs your programs. 

Computers are fast. Really fast. So fast, in fact, that they spend most of their time waiting for things to happen. Unless you’re decompressing files, encoding audio, or crunching numbers, chances are that your CPU mostly sits idle, waiting for operations to complete. It’s waiting for a network packet to arrive, for the mouse to move, for the disk to finish writing some bytes, or maybe even just for a read from main memory to complete. From the CPU’s perspective, eons go by between most such events. When one does occur, the CPU runs a few more instructions, then goes back to waiting again. Take a look at your CPU utilization—it’s probably somewhere in the low single digits, and that’s likely where it hovers the majority of the time. 

**Synchronous Interfaces** 

Synchronous interfaces allow your program (or rather, a single thread in your program) to execute only a single operation at a time; each operation has to wait for the previous synchronous operation to finish before it gets to run. Most interfaces you see in the wild are synchronous: you call them, they go do some stuff, and eventually they return when the operation has completed and your program can continue from there. The reason for this, as we’ll see later in this chapter, is that making an operation asynchronous takes a fair bit of extra machinery. Unless you need the benefits of asynchrony, sticking to the synchronous model requires much less pomp and circumstance. 

Synchronous interfaces hide all this waiting; the application calls a function that says “write these bytes to this file,” and some time later, that function completes and the next line of code executes. Behind the scenes, what really happens is that the operating system queues up a write operation to the disk and then puts the application to sleep until the disk reports that it has finished the write. The application experiences this as the func- tion taking a long time to execute, but in reality it isn’t really executing at all, just waiting. 

An interface that performs operations sequentially in this way is also often referred to as *blocking*, since the operation in the interface that has to wait for some external event to happen in order for it to make progress *blocks* further execution until that event happens. Whether you refer to an interface as synchronous or blocking, the basic idea is the same: the appli- cation does not move on until the current operation finishes. While the operation is waiting, so is the application. 

Synchronous interfaces are usually considered to be easy to work with and simple to reason about, since your code executes just one line at a time. 

But they also allow the application to do only one thing at a time. That means if you want your program to wait for either user input or a network packet, you’re out of luck unless your operating system provides an opera- tion specifically for that. Similarly, even if your application could do some other useful work while the disk is writing a file, it doesn’t have that option as the file write operation blocks the execution! 

**Multithreading** 

By far the most common solution to allowing concurrent execution is to use *multithreading*. In a multithreaded program, each thread is responsible for executing a particular independent sequence of blocking operations, and the operating system multiplexes among the threads so that if any thread can make progress, progress is made. If one thread blocks, some other thread may still be runnable, and so the application can continue to do useful work. 

Usually, these threads communicate with each other using a synchronization primitive like a lock or a channel so that the application can still coordinate their efforts. For example, you might have one thread that waits for user input, one thread that waits for network packets, and another thread that waits for either of those threads to send a message on a channel shared between all three threads. 

Multithreading gives you *concurrency*—the ability to have multiple independent operations that can be executed at any one time. It’s up to the system running the application (in this case, the operating system) to choose among the threads that aren’t blocked and decide which to execute next. If one thread is blocked, it can choose to run another one that can make progress instead. 

Multithreading combined with blocking interfaces gets you quite far, and large swaths of production-ready software are built in this way. But this approach is not without its shortcomings. First, keeping track of all these threads quickly gets cumbersome; if you have to spin up a thread for every concurrent task, including simple ones like waiting for keyboard input, the threads add up fast, and so does the additional complexity needed to keep track of how all those threads interact, communicate, and coordinate. 

Second, switching between threads gets costly the more of them there are. Every time one thread stops running and another one starts back up in its place, you need to do a round-trip to the operating system scheduler, and that’s not free. On some platforms, spawning new threads is also a fairly heavyweight process. Applications with high performance needs often mitigate this cost by reusing threads and using operating system calls that allow you to block on many related operations, but ultimately you are left with the same problem: blocking interfaces require that you have as many threads as the number of blocking calls you want to make. 

Finally, threads introduce *parallelism* into your program. The distinction between concurrency and parallelism is subtle, but important: con- currency means that the execution of your tasks is interleaved, whereas parallelism means that multiple tasks are executing at the same time. If you have two tasks, their execution expressed in ASCII might look like _-_- (concurrency) versus ===== (parallelism). Multithreading does not necessarily imply parallelism—even though you have many threads, you might have only a single core, so only one thread is executing at a given time—but the two usually go hand in hand. You can make two threads mutually exclusive in their execution by using a Mutex or other synchronization primitive, but that introduces additional complexity—threads want to run in parallel. And while parallelism is often a good thing—who doesn’t want their pro- gram to run faster on more cores—it also means that your program must handle truly simultaneous access to shared data structures. This means moving from Rc, Cell, and RefCell to the more powerful but also slower Arc and Mutex. While you *may* want to use the latter types in your concurrent program to enable parallelism, threading *forces* you to use them. We’ll look at multithreading in much greater detail in Chapter 10. 

**Asynchronous Interfaces** 

Now that we’ve explored synchronous interfaces, we can look at the alter- native: asynchronous or *nonblocking* interfaces. An asynchronous interface is one that may not yield a result straightaway, and may instead indicate that the result will be available at some later time. This gives the caller the opportunity to do something else in the meantime rather than having to go to sleep until that particular operation completes. In Rust parlance, an asynchronous interface is a method that returns a Poll, as defined in Listing 8-1. 

```
enum Poll<T> {
    Ready(T),
Pending } 
```

*Listing 8-1: The core of asynchrony: the “here you are or come back later” type*

Poll usually shows up in the return type of functions whose names start with poll—these are methods that signal they can attempt an operation without blocking. We’ll get into how exactly they do that later in this chapter, but in general they attempt to perform as much as they can of the operation before they would normally block, and then return. And crucially, they remember where they left off so that they can resume execution later when additional progress can again be made. 

These nonblocking functions allow us to easily perform multiple tasks concurrently. For example, if you want to read from either the network or the user’s keyboard, whichever has an event available first, all you have to do is poll both in a loop until one of them returns Poll::Ready. No need for any additional threads or synchronization! 

The word *loop* here should make you a little nervous. You don’t want your program to burn through a loop three billion times a second when it may be minutes until the next input occurs. In the world of blocking interfaces, this wasn’t a problem since the operating system simply put the thread to sleep and then took care of waking it up when a relevant event occurred, but how do we avoid burning cycles while waiting in this brave new nonblocking world? That’s what much of the remainder of this chapter will be about. 

**Standardized Polling** 

To get to a world where every library can be used in a nonblocking fashion, we could have every library author cook up their own poll methods, all with slightly different names, signatures, and return types—but that would quickly get unwieldy. Instead, in Rust, polling is standardized through the Future trait. A simplified version of Future is shown in Listing 8-2 (we’ll get back to the real one later in this chapter). 

```
trait Future {
    type Output;
    fn poll(&mut self) -> Poll<Self::Output>;
}
```

Listing 8-2: A simplified view of the *Future* trait 

Types that implement the Future trait are known as *futures* and repre- sent values that may not be available yet. A future could represent the next time a network packet comes in, the next time the mouse cursor moves, or just the point at which some amount of time has elapsed. You can read Future<Output = Foo> as “a type that will produce a Foo in the future.” Types like this are often referred to in other languages as *promises*—they promise that they will eventually yield the indicated type. When a future eventually returns Poll::Ready(T), we say that the future *resolves* into a T. 

With this trait in place, we can generalize the pattern of providing poll methods. Instead of having methods like poll_recv and poll_keypress, we can have methods like recv and keypress that both return impl Future with an appropriate Output type. This doesn’t change the fact that you have to poll them—we’ll deal with that later—but it does mean that at least there is a standardized interface to these kinds of pending values, and we don’t need to use the poll_ prefix everywhere. 

**NOTE** *In general, you should not poll a future again after it has returned* *Poll::Ready**. If you do, the future is well within its rights to panic. A future that is safe to poll after it has returned* *Ready* *is sometimes referred to as a* fused *future.* 

**Ergonomic Futures** 

Writing a type that implements Future in the way I’ve described so far is quite a pain. To see why, first take a look at the fairly straightforward asyn- chronous code block in Listing 8-3 that simply tries to forward messages from the input channel rx to the output channel tx. 

```
async fn forward<T>(rx: Receiver<T>, tx: Sender<T>) {
	while let Some(t) = rx.next().await {
		tx.send(t).await;
	}
}
```

} 

*Listing 8-3: Implementing a channel-forwarding future using *async* and *await * 

This code, written using async and await syntax, looks very similar to its equivalent synchronous code and is easy to read. We simply send each mes- sage we receive in a loop until there are no more messages, and each await point corresponds to a place where a synchronous variant might block. Now think about if you instead had to express this code by manually implementing the Future trait. Since each call to poll starts at the top of the function, you’d need to package the necessary state to continue from the last place the code yielded. The result is fairly grotesque, as Listing 8-4 demonstrates. 

```
enum Forward<T> { 1
 WaitingForReceive(ReceiveFuture<T>, Option<Sender<T>>), WaitingForSend(SendFuture<T>, Option<Receiver<T>>), 

} 

impl<T> Future for Forward<T> {
 type Output = (); 2
 fn poll(&mut self) -> Poll<Self::Output> { 

match self { 3 Forward::WaitingForReceive(recv, tx) => { 
enum Forward<T> { 1
 WaitingForReceive(ReceiveFuture<T>, Option<Sender<T>>), WaitingForSend(SendFuture<T>, Option<Receiver<T>>), 

} 

impl<T> Future for Forward<T> {
 type Output = (); 2
 fn poll(&mut self) -> Poll<Self::Output> { 

match self { 3 Forward::WaitingForReceive(recv, tx) => { 

                if let Poll::Ready((rx, v)) = recv.poll() {
                    if let Some(v) = v {

let tx = tx.take().unwrap(); 4
 *self = Forward::WaitingForSend(tx.send(v), Some(rx)); 5 // Try to make progress on sending.
 return self.poll(); 6 

} else { 

		// No more items.
		Poll::Ready(())
	}
} else {
	Poll::Pending

} } 

            Forward::WaitingForSend(send, rx) => {
                if let Poll::Ready(tx) = send.poll() {
                    let rx = rx.take().unwrap();
                    *self = Forward::WaitingForReceive(rx.receive(), Some(tx));
                    // Try to make progress on receiving.
                    return self.poll();
                } else {
                    Poll::Pending

} } 

} } 

} 
```

*Listing 8-4: Manually implementing a channel-forwarding future*

You’ll rarely have to write code like this in Rust anymore, but it gives important insight into how things work under the hood, so let’s walk through it. First, we define our future type as an enum 1, which we’ll use to keep track of what we’re currently waiting on. This is a consequence of the fact that when we return Poll::Pending, the next call to poll will start at the top of the function again. We need some way to know what we were in the middle of so that we know which operation to continue on. Furthermore, we need to keep track of different information depending on what we’re doing: if we’re wait- ing for a receive to finish, we need to keep that ReceiveFuture (the definition of which is not shown in this example) so that we can poll it the next time we are polled ourselves, and the same goes for SendFuture. The Options here might strike you as weird too; we’ll get back to those shortly. 

When we implement Future for Forward, we declare its output type as () 2 because this future doesn’t actually return anything. Instead, the future resolves (with no result) when it has finished forwarding everything from the input channel to the output channel. In a more complete exam- ple, the Output of our forwarding type might be a Result so that it could com- municate errors from receive() and send() back up the stack to the function that’s polling for the completion of the forwarding. But this code is compli- cated enough already, so we’ll leave that for another day. 

When Forward is polled, it needs to resume wherever it last left off, which we find out by matching on the enum variant currently held in self 3. For whichever branch we go into, the first step is to poll the future that blocks progress for the current operation; if we’re trying to receive, we poll the ReceiveFuture, and if we’re trying to send, we poll the SendFuture. If that call to poll returns Poll::Pending, then we can make no progress, and we return Poll::Pending ourselves. But if the current future resolves, we have work to do! 

When one of the inner futures resolves, we need to update what the current operation is by switching which enum variant is stored in self. In order to do so, we have to move out of self to call Receiver::receive or Sender::send—but we can’t do that because all we have is &mut self. So, we store the state we have to move in an Option, which we move out of with Option::take 4. This is silly since we’re about to overwrite self anyway 5, and hence the Options will always be Some, but sometimes tricks are needed to make the borrow checker happy. 

Finally, if we do make progress, we then poll self again 6 so that if we can immediately make progress on the pending send or receive, we do so. This is actually necessary for correctness when implementing the real Future trait, which we’ll get back to later, but for now think of this as an optimization. 

We just hand-wrote a *state machine*: a type that has a number of possible states and moves between them in response to particular events. This was a fairly simple state machine, at that. Imagine having to write code like this for more complicated use cases where you have additional intermediate steps! 

Beyond writing the unwieldy state machine, we have to know the types of the futures that Sender::send and Receiver::receive return so that we can store them in our type. If those methods instead returned impl Future, we’d have no way to write out the types for our variants. The send and receive methods also have to take ownership of the sender and the receiver; if they did not, the lifetimes of the futures they returned would be tied to the borrow of self, which would end when we return from poll. But that would not work, since we’re trying to store those futures *in* self. 

**N O T E**  *You may have noticed that* *Receiver* *looks a lot like an asynchronous version of* *Iterator**. Others have noticed the same thing, and the standard library is on its way to adding a trait specifically for types that can meaningfully implement* *poll_next**. Down the line, these asynchronous iterators (often referred to as* streams*) may end up with first-class language support, such as the ability to loop over them directly!* 

Ultimately, this code is hard to write, hard to read, and hard to change. If we wanted to add error handling, for example, the code complexity would increase significantly. Luckily, there’s a better way! 

**async/await** 

Rust 1.39 gave us the async keyword and the closely related await postfix operator, which we used in the original example in Listing 8-3. Together, they provide a much more convenient mechanism for writing asynchronous state machines like the one in Listing 8-5. Specifically, they let you write the code in such a way that it doesn’t even look like a state machine! 

```
async fn forward<T>(rx: Receiver<T>, tx: Sender<T>) {
    while let Some(t) = rx.next().await {
        tx.send(t).await;
    }
}
```
*Listing 8-5: Implementing a channel-forwarding future using *async* and *await*, repeated from Listing 8-3 *

If you don’t have much experience with async and await, the difference between Listing 8-4 and Listing 8-5 might give you an idea of why the Rust community was so excited to see them land. But since this is an intermedi- ate book, let’s dive a little deeper to understand just how this short segment of code can replace the much longer manual implementation. To do that, we first need to talk about *generators*—the mechanism by which async and await are implemented. 

**Generators** 

Briefly described, a generator is a chunk of code with some extra compiler- generated bits that enables it to stop, or *yield*, its execution midway through and then resume from where it last yielded later on. Take the forward func- tion in Listing 8-3, for example. Imagine that it gets to the call to send, but the channel is currently full. The function can’t make any more progress, but it also cannot block (this is nonblocking code, after all), so it needs to return. Now suppose the channel eventually clears and we want to proceed with the send. If we call forward again from the top, it’ll call next again and the item we previously tried to send will be lost, so that’s no good. Instead, we turn forward into a generator. 

Whenever the forward generator cannot make progress anymore, it needs to store its current state somewhere so that when its execution even- tually resumes, it resumes in the right place with the right state. It saves the state through an associated data structure that’s generated by the compiler, which contains all the state of the generator at a given point in time. A method on that data structure (also generated) then allows the generator to resume from its current state, stored in &mut self, and updates the state again when the generator again cannot make progress. 

This “return but allow me to resume later” operation is called *yielding*, which effectively means it returns while keeping some extra state on the side. When we later want to resume a call to forward, we invoke the known entry point into the generator (the *resume method*, which is poll for async generators), and the generator inspects the previously stored state in self to decide what to do next. This is exactly the same thing we did manually in Listing 8-4! In other words, the code in Listing 8-5 loosely desugars to the hypothetical code shown in Listing 8-6. 

```
generator fn forward<T>(rx: Receiver<T>, tx: Sender<T>) {
    loop {
        let mut f = rx.next();
        let r = if let Poll::Ready(r) = f.poll() { r } else { yield };
        if let Some(t) = r {
            let mut f = tx.send(t);
            let _ = if let Poll::Ready(r) = f.poll() { r } else { yield };
        } else { break Poll::Ready(()); }
	}
} 
```
*Listing 8-6: Desugaring *async*/*await* into a generator *

At the time of writing, generators are not actually usable in Rust—they are only used internally by the compiler to implement async/await—but that may change in the future. Generators come in handy in a number of cases, such as to implement iterators without having to carry around a struct or to implement an impl Iterator that figures out how to yield items one at a time. 

If you look closely at Listings 8-5 and 8-6, they may seem a little magical once you know that every await or yield is really a return from the function. After all, there are several local variables in the function, and it’s not clear how they’re restored when we resume later on. This is where the compiler- generated part of generators comes into play. The compiler transparently injects code to persist those variables into and read them from the genera- tor’s associated data structure, rather than the stack, at the time of execu- tion. So if you declare, write to, or read from some local variable a, you are really operating on something akin to self.a. Problem solved! It’s all really quite marvelous. 

One subtle but important difference between the manual forward imple- mentation and the async/await version is that the latter can hold references across yield points. This enables functions like Receiver::next and Sender::send in Listing 8-5 to take &mut self rather than the self they took in Listing 8-4. If we tried to use a &mut self receiver for these methods in the manual state machine implementation, the borrow checker would have no way to enforce that the Receiver stored inside Forward cannot be referenced between when Receiver::next is called and when the future it returns resolves, and so it would reject the code. Only by moving the Receiver into the future can we convince the compiler that the Receiver is not otherwise accessible. Meanwhile, with async/await, the borrow checker can inspect the code before the compiler turns it into a state machine and verify that rx is indeed not accessed again until after the future is dropped, when the await on it returns. 

**THE SIZE OF GENERATORS** 

The data structure used to back a generator’s state must be able to hold the com- bined state at any one yield point . If your async fn contains, say, a [u8; 8192], those 8KiB must be stored in the generator itself . Even if your async fn contains only smaller local variables, it must also contain any future that it awaits, since it needs to be able to poll such a future later, when poll is invoked . 

This nesting means that generators, and thus futures based on async functions and blocks, can get quite large without any visible indicator of that increased size in your code . This can in turn impact your program’s runtime performance, since those giant generators may have to be copied across func- tion calls and in and out of data structures, which amounts to a fair amount of memory copying . In fact, you can usually identify when the size of your gener- ator-based futures is affecting performance by looking for excessive amounts of time spent in the memcpy function in your application’s performance profiles! 

Finding these large futures isn’t always easy, however, and often requires manually identifying long or complex chains of async functions . Clippy may be able to help with this in the future, but at the time of writing, you’re on your own . When you do find a particularly large future, you have two options: you can try to reduce the amount of local state the async functions need, or you can move the future to the heap (with Box::pin) so that moving the future just requires moving the pointer to it . The latter is by far the easiest way to go, but it also introduces an extra allocation and a pointer indirection . Your best bet is usually to put the problematic future on the heap, measure your performance, and then use your performance benchmarks to guide you from there . 

**Pin and Unpin** 

We’re not quite done. While generators are neat, a challenge arises from the technique as I’ve described it so far. In particular, it’s not clear what happens if the code in the generator (or, equivalently, the async block) takes a reference to a local variable. In the code from Listing 8-5, the future that rx.next() returns must necessarily hold a reference to rx if a next message 

is not immediately available so that it knows where to try again when the generator next resumes. When the generator yields, the future and the ref- erence the future contains get stashed away inside the generator. But what now happens if the generator is moved? Specifically, look at the code in Listing 8-7, which calls forward. 

```
async fn try_forward<T>(rx: Receiver<T>, tx: Sender<T>) -> Option<impl Future>
{
    let mut f = forward(rx, tx);
    if f.poll().is_pending() { Some(f) } else { None }
}
```

*Listing 8-7: Moving a future after polling it*

The try_forward function polls forward only once, to forward as many messages as possible without blocking. If the receiver may still produce more messages (that is, if it returned Poll::Pending instead of Poll::Ready(None)), those messages are deferred to be forwarded at some later time by returning the forwarding future to the caller, which may choose to poll again at a time when it sees fit. 

Let’s work through what happens here with what we know about async and await so far. When we poll the forward generator, it goes through the while loop some unknown number of times and eventually returns either Poll::Ready(()) if the receiver ended, or Poll::Pending otherwise. If it returns Poll::Pending, the generator contains a future returned from either rx.next() or tx.send(t). Those futures both contain a reference to one of the argu- ments initially provided to forward (rx and tx, respectively), which must also be stored in the generator. But when try_forward returns the entire genera- tor, the fields of the generator also move. Thus, rx and tx no longer reside at the same locations in memory, and the references stored in the stashed- away future are no longer pointing to the right data! 

What we’ve run into here is a case of a *self-referential* data structure: one that holds both data and references to that data. With generators, these self- referential structures are very easy to construct, and being unable to support them would be a significant blow to ergonomics because it would mean you wouldn’t be able to hold references across any yield point. The (ingenious) solution for supporting self-referential data structures in Rust comes in the form of the Pin type and the Unpin trait. Very briefly, Pin is a wrapper type that prevents the wrapped type from being (safely) moved, and Unpin is a marker trait that says the implementing type *can* be removed safely from a Pin. 

**Pin** 

There’s a lot of nuance to cover here, so let’s start with a concrete use of the Pin wrapper. Listing 8-2 gave you a simplified version of the Future trait, but we’re now ready to peel back one part of the simplification. Listing 8-8 shows the Future trait somewhat closer to its final form. 

```
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>) -> Poll<Self::Output>;
}
```

*Listing 8-8: A less simplified view of the *Future* trait with *Pin* *


In particular, this definition requires that you call poll on Pin<&mut Self>. Once you have a value behind a Pin, that constitutes a contract that that value will never move again. This means that you can construct self-references internally to your heart’s delight, exactly as you want for generators. 

**NOTE** *While* *Future* *makes use of* *Pin**,* *Pin* *is not tied to the* *Future* *trait—you can use* *Pin* *for* any *self-referential data structure.* 

But how do you get a Pin to call poll? And how can Pin ensure that the contained value won’t move? To see how this magic works, let’s look at the definition of std::pin::Pin and some of its key methods, shown in Listing 8-9. 

```
struct Pin<P> { pointer: P }
impl<P> Pin<P> where P: Deref {
  pub unsafe fn new_unchecked(pointer: P) -> Self;
}
impl<'a, T> Pin<&'a mut T> {
  pub unsafe fn get_unchecked_mut(self) -> &'a mut T;
}
impl<P> Deref for Pin<P> where P: Deref {
  type Target = P::Target;
  fn deref(&self) -> &Self::Target;
}
```

*Listing 8-9: *std::pin::Pin* and its key methods *

There’s a lot to unpack here, and we’re going to have to go over the definition in Listing 8-9 a few times before all the bits make sense, so please bear with me. 

First, you’ll notice that Pin holds a *pointer type*. That is, rather than hold some T directly, it holds a type P that dereferences through Deref into T. This means that rather than have a Pin<MyType>, you’ll have a Pin<Box<MyType>> or Pin<Rc<MyType>> or Pin<&mut MyType>. The reason for this design is simple— Pin’s primary goal is to make sure that once you place a T behind a Pin, that T won’t move, as doing so might invalidate self-references stored in the T. If the Pin just held a T directly, then simply moving the Pin would be enough to invalidate that invariant! In the remainder of this section, I’ll refer to P as the *pointer* type and T as the *target* type. 

Next, notice that Pin’s constructor, new_unchecked, is unsafe. This is because the compiler has no way to actually check that the pointer type indeed promises that the pointed-to (target) type won’t move again. Con- sider, for example, a variable foo on the stack. If Pin’s constructor were safe, we could do Pin::new(&mut foo), call a method that requires Pin<&mut Self> (and thus assumes that Self won’t move again), and then drop the Pin. At this point, we could modify foo as much as we liked, since it is no longer borrowed—including moving it! We could then pin it again and call the same method, which would be none the wiser that any self-referential point- ers it may have constructed the first time around would now be invalid. 


**PIN CONSTRUCTOR SAFETY** 

The other reason the constructor for Pin is unsafe is that its safety depends on the implementation of traits that are themselves safe . For example, the way that Pin<P> implements get_unchecked_mut is to use the implementation of DerefMut::deref_mut for P . While the call to get_unchecked_mut is unsafe, the impl DerefMut for P is not . Yet it receives a &mut self, and can thus freely (and without unsafe code) move the T . The same thing applies to Drop . The safety requirement for Pin::new_unchecked is therefore not only that the pointer type will not let the target type be moved again (like in the Pin<&mut T> example), but also that its Deref, DerefMut, and Drop implementations do not move the pointed-to value behind the &mut self they receive . 

We then get to the get_unchecked_mut method, which gives you a mutable reference to the T behind the Pin’s pointer type. This method is also unsafe, because once we give out a &mut T, the caller has to promise it won’t use that &mut T to move the T or otherwise invalidate its memory, lest any self- references be invalidated. If this method weren’t unsafe, a caller could 
call a method that takes Pin<&mut Self> and then call the safe variant of get_unchecked_mut on two Pin<&mut _>s, then use mem::swap to swap the values behind the Pin. If we were to then call a method that takes Pin<&mut Self> again on either Pin, its assumption that the Self hasn’t moved would be vio- lated, and any internal references it stored would be invalid! 

Perhaps surprisingly, Pin<P> always implements Deref<Target = T>, and that is entirely safe. The reason for this is that a &T does not let you move T without writing other unsafe code (UnsafeCell, for example, as we’ll discuss in Chapter 9). This is a good example of why the scope of an unsafe block extends beyond just the code it contains. If you wrote some code in one part of the application that (unsafely) replaced a T behind an & using UnsafeCell, then it *could* be that that &T initially came from a Pin<&mut T>, and that you have now violated the invariant that the T behind the Pin may never move, even though the place where you unsafely replaced the &T did not even men- tion Pin! 

*If you’ve browsed through the* *Pin* *documentation while reading this chapter, you may have noticed* *Pin::set**, which takes a* *&mut self* *and a* *::Target* *and safely changes the value behind the* *Pin**. This is possible because* *set* *does not return the value that was previously pinned—it simply drops it in place and stores the new value there instead. Therefore, it does not violate the pinning invariants: the old value was never accessed outside of a* *Pin* *after it was placed there.* 

**Unpin: The Key to Safe Pinning** 

At this point you might ask: given that getting a mutable reference is unsafe anyway, why not have Pin hold a T directly? That is, rather than require an indirection through a pointer type, you could instead make the contract for get_unchecked_mut that it is only safe to call if you haven’t moved the Pin. The answer to that question lies in a neat safe use of Pin that the pointer design enables. Recall that the whole reason we want Pin in the first place is so we can have target types that may contain references to themselves (like a generator) and give their methods a guarantee that the target type hasn’t moved and thus that internal self-references remain valid. Pin lets us use the type system to enforce that guarantee, which is great. But unfortunately, with the design so far, Pin is very unwieldy to work with. This is because it always requires unsafe code, even if you are working with a target type that doesn’t contain any self-references, and so doesn’t care whether it’s been moved or not. 

This is where the marker trait Unpin comes into play. An implementation of Unpin for a type simply asserts that the type is safe to move out of a Pin when used as a target type. That is, the type promises that it will never use any of Pin’s guarantees about the referent not moving again when used as a target type, and thus those guarantees may be broken. Unpin is an auto-trait, like Send and Sync, and so is auto-implemented by the compiler for any type that contains only Unpin members. Only types that explicitly opt out of Unpin (like generators) and types that contain those types are !Unpin. 

For target types that are Unpin, we can provide a much simpler safe interface to Pin, as shown in Listing 8-10. 

```
impl<P> Pin<P> where P: Deref, P::Target: Unpin {
    pub fn new(pointer: P) -> Self;
}
impl<P> DerefMut for Pin<P> where P: DerefMut, P::Target: Unpin {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

Listing 8-10: The safe API to *Pin* for *Unpin* target types 

To make sense of the safe API in Listing 8-10, think about the safety requirements of the unsafe methods from Listing 8-9: the function Pin::new_unchecked is unsafe because the caller must promise that the ref- erent cannot be moved outside of the Pin, and that the implementations of Deref, DerefMut, and Drop for the pointer type do not move the referent through the reference they receive. Those requirements are there to ensure that once we give out a Pin to a T, we never move that T again. But if the T is Unpin, it has declared that it does not care if it is moved even if it was previously pinned, so it’s fine if the caller does not satisfy any of those requirements! 

Similarly, get_unchecked_mut is unsafe because the caller must guarantee that it doesn’t move the T out of the &mut T—but with T: Unpin, T has declared that it’s fine being moved even after being pinned, so that safety require- ment is no longer important. This means that for Pin<P> where P::Target: Unpin, we can simply provide safe variants of both those methods (DerefMut being the safe version of get_unchecked_mut). In fact, we can even provide a Pin::into_inner that simply gives back the owned P if the target type is Unpin, since the Pin is essentially irrelevant! 

**Ways of Obtaining a Pin** 

With our new understanding of Pin and Unpin, we can now make progress toward using the new Future definition from Listing 8-8 that requires Pin<&mut Self>. The first step is to construct the required type. If the future type is Unpin, that step is easy—we just use Pin::new(&mut future). If it is not Unpin, we can pin the future in one of two main ways: by pinning to the heap or pinning to the stack. 

Let’s start with pinning to the heap. The primary contract of Pin is that once something has been pinned, it cannot move. The pinning API takes care of honoring that contract for all methods and traits on Pin, so the main role of any function that constructs a Pin is to ensure that if the Pin *itself* moves, the referent value does not move too. The easiest way to ensure that is to place the referent on the heap, and then place just a pointer to the refer- ent in the Pin. You can then move the Pin to your heart’s delight, but the tar- get will remain where it was. This is the rationale behind the (safe) method Box::pin, which takes a T and returns a Pin<Box<T>>. There’s no magic to it; it simply asserts that Box follows the Pin constructor, Deref, and Drop contracts. 

**UNPIN BOX** 

While we’re on the topic of Box, take a look at the implementation of Unpin for Box . The Box type unconditionally implements Unpin for any T, even if that T is not Unpin . This might strike you as odd, given the earlier assertion that Unpin is an auto-trait that is generally implemented for a type only if all of the type’s members are also Unpin . Box is an exception to this for the same reason that it can provide a safe Pin constructor: if you move a Box<T>, you do not move the T . In other words, the unconditional implementation asserts that you can move a Box<T> out of a Pin even if T cannot be moved out of a Pin . Note, however, that this does not enable you to move a T that is !Unpin out of a Pin<Box<T>> . 

The other option, pinning to the stack, is a little more involved, and at the time of writing requires a smidgen of unsafe code. We have to ensure that the pinned value cannot be accessed after the Pin with a &mut to it has been dropped. We accomplish that by shadowing the value as shown in the macro in Listing 8-11 or by using one of the crates that provide exactly this macro. One day it may even make it into the standard library! 

```
macro_rules! pin_mut {
    ($var:ident) => {
        let mut $var = $var;
        let mut $var = unsafe { Pin::new_unchecked(&mut $var) };
    }
} 
```

Listing 8-11: Macro for pinning to the stack 

By taking the name of the variable to pin to the stack, the macro ensures that the caller has the value it wants to pin somewhere on the stack already. The shadowing of $var ensures that the caller cannot drop the Pin and continue to use the unpinned value (which would breach the Pin contract for any target type that’s !Unpin). By moving the value stored in $var, the macro also ensures that the caller cannot drop the $var bind- ing the macro declarations without also dropping the original variable. Specifically, without that line, the caller could write (note the extra scope): 

```
let foo = /* */; { pin_mut!(foo); foo.poll() }; foo.mut_self_method();
```

Here, we give a pinned instance of foo to poll, but then we later use a &mut to foo without a Pin, which violates the Pin contract. With the extra reas- signment, on the other hand, that code would also move foo into the new scope, rendering it unusable after the scope ends. 

Pinning on the stack therefore requires unsafe code, unlike Box::pin, but avoids the extra allocation that Box introduces and also works in no_std environments. 

**Back to the Future** 

We now have our pinned future, and we know what that means. But you may have noticed that none of this important pinning stuff shows up in most asynchronous code you write with async and await. And that’s because the compiler hides it from you. 

Think back to when we discussed Listing 8-5, when I told you that <expr>.await desugars into something like: 

```
loop { if let Poll::Ready(r) = expr.poll() { break r } else { yield } }
```

That was an ever-so-slight simplification because, as we’ve seen, you can call Future::poll only if you have a Pin<&mut Self> for the future. The desug- aring is actually a bit more sophisticated, as shown in Listing 8-12. 

```
1 match expr {
 mut pinned => loop { 

2 match unsafe { Pin::new_unchecked(&mut pinned) }.poll() { Poll::Ready(r) => break r,
 Poll::Pending => yield, 

} } 
} 
```

Listing 8-12: A more accurate desugaring of *.await* 

The match 1 is a neat shorthand to not only ensure that the expansion remains a valid expression, but also move the expression result into a variable that we can then pin on the stack. Beyond that, the main new addition is the call to Pin::new_unchecked 2. That call is safe because for the containing async block to be polled, it must already be pinned due to the signature of Future::poll. And the async block was polled for us to reach the call to Pin::new_unchecked, so the generator state is pinned. Since pinned is stored in the generator that corresponds to the async block (it must be so that yield will resume correctly), we know that pinned will not move again. Furthermore, pinned is not accessible except through a Pin once we’re in the loop, so no code is able to move out of the value in pinned. Thus, we meet all the safety requirements of Pin::new_unchecked, and the code is safe. 

**Going to Sleep** 

We went pretty deep into the weeds with Pin, but now that we’re out the other side, there is another issue around futures that may have been mak- ing your brain itch. If a call to Future::poll returns Poll::Pending, you need something to call poll again at a later time to check whether you can make progress yet. That something is usually called the *executor*. Your executor could be a simple loop that polls all the futures you are waiting on until they’ve all returned Poll::Ready, but that would burn a lot of CPU cycles you could probably have used for other, more useful things, like running your web browser. Instead, we want the executor to do whatever useful work it can do, and then go to sleep. It should stay asleep until one of the futures can make progress, and only then wake up to do another pass, before going to sleep again. 

**Waking Up** 

The condition that determines when to check back with a given future var- ies widely. It might be “when a network packet arrives on this port,” “when the mouse cursor moves,” “when someone sends on this channel,” “when the CPU receives a particular interrupt,” or even “after this much time has passed.” On top of that, developers can write their own futures that wrap multiple other futures, and thus, they may have several wake-up conditions. Some futures may even introduce their own entirely custom wake events. 

To accommodate these many use cases, Rust introduces the notion of a Waker: a way to wake the executor to signal that progress can be made. The Waker is what makes the whole machinery around futures work. The executor constructs a Waker that integrates with the mechanism the executor uses to go to sleep, and passes the Waker in to any Future it polls. How? With the addi- tional parameter to Future::poll that I’ve hidden from you so far. Sorry about that. Listing 8-13 gives the final and true definition for Future—no more lies! 

```
             trait Future {
                 type Output;
                 fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
             }
```

Listing 8-13: The actual *Future* trait with *Context* 

The &mut Context contains the Waker. The argument is a Context, not a Waker directly, so that we can augment the asynchronous ecosystem with additional context for futures should that be deemed necessary. 

Asynchronous Programming **133** 

**134** Chapter 8 

**N O T E** 

The primary method on Waker is wake (and the by-reference variant wake _by_ref), which should be called when the future can again make progress. The wake method takes no arguments, and its effects are entirely defined
 by the executor that constructed the Waker. You see, Waker is secretly generic over the executor. Or, more precisely, whatever constructed the Waker gets to dictate what happens when Waker::wake is called, when a Waker is cloned, and when a Waker is dropped. This all happens through a manually implemented vtable, which functions similarly to the dynamic dispatch we discussed way back in Chapter 2. 

It’s a somewhat involved process to construct a Waker, and the mechan- ics of it aren’t all that important for using one, but you can see the building blocks in the RawWakerVTable type in the standard library. It has a constructor that takes the function pointers for wake and wake_by_ref as well as Clone and Drop. The RawWakerVTable, which is usually shared among all of an executor’s wakers, is bundled up with a raw pointer intended to hold data specific to each Waker instance (like which future it’s for) and is turned into a RawWaker. That is in turn passed to Waker::from_raw to produce a safe Waker that can be passed to Future::poll. 

**Fulfilling the Poll Contract** 

So far we’ve skirted around what a future actually does with a Waker. The idea is fairly simple: if Future::poll returns Poll::Pending, it is the future’s responsibility to ensure that *something* calls wake on the provided Waker
 when the future is next able to make progress. Most futures uphold this property by returning Poll::Pending only if some other future also returned Poll::Pending; in this way, it trivially fulfills the contract of poll since the inner future must follow that same contract. But there can’t be turtles all the way down. At some point, you reach a future that does not poll other futures but instead does something like write to a network socket or attempt to receive on a channel. These are commonly referred to as *leaf futures* since they have no children. A leaf future has no inner future but instead directly represents some resource that may not yet be ready to return a result. 

*The poll contract is the reason why the recursive* *poll* *call* 6 *back in Listing 8-4 is necessary for correctness.* 

Leaf futures typically come in one of two shapes: those that wait for events that originate within the same process (like a channel receiver), and those that wait for events external to the process (like a TCP packet read). Those that wait for internal events all tend to follow the same pattern: store the Waker where the code that will wake you up can find it, and have that code call wake on the Waker when it generates the relevant event. For example, consider a leaf future that has to wait for a message on an in-memory chan- nel. It stores its Waker inside the part of the channel that is shared between the sender and the receiver and then returns Poll::Pending. When a sender later comes along and injects a message into the channel, it notices the Waker left there by the waiting receiver and calls wake on the Waker before returning from send. Now the receiver is awoken, and the poll contract is upheld. 

**N O T E** 

Leaf futures that deal with external events are more involved, as the code that generates the event they’re waiting for knows nothing of futures or wakers. Most often the generating code is the operating system kernel, which knows when a disk is ready or a timer expires, but it could also be
 a C library that invokes a callback into Rust when an operation completes or some other such external entity. A leaf future wrapping an external resource like this could spin up a thread that executes a blocking system call (or waits for the C callback) and then use the internal waking mecha- nism, but that would be wasteful; you would spin up a thread every time an operation had to wait and be left with lots of single-use threads sitting around waiting for things. 

Instead, executors tend to provide implementations of leaf futures
 that communicate behind the scenes with the executor to arrange for
 the appropriate interaction with the operating system. How exactly this
 is orchestrated depends on the executor and the operating system, but roughly speaking the executor keeps track of all the event sources that it should listen for the next time it goes to sleep. When a leaf future realizes it must wait for an external event, it updates that executor’s state (which it knows about since it’s provided by the executor crate) to include that exter- nal event source alongside its Waker. When the executor can no longer make progress, it gathers all of the event sources the various pending leaf futures are waiting for and does a big blocking call to the operating system, telling it to return when *any* of the resources the leaf futures are waiting on have
 a new event. On Linux, this is usually achieved with the epoll system call; Windows, the BSDs, macOS, and pretty much every other operating system provide similar mechanisms. When that call returns, the executor calls wake on all the wakers associated with event sources that the operating system reported events for, and thus the poll contract is fulfilled. 

*A* reactor *is the part of an executor that leaf futures register event sources with and that the executor waits on when it has no more useful work to do. It is possible to separate the executor and the reactor, though bundling them together often improves performance as the two can be co-optimized more readily.* 

A knock-on effect of the tight integration between leaf futures and the executor is that leaf futures from one executor crate often cannot be used with a different executor. Or at least, they cannot be used unless the leaf future’s executor is *also* running. When the leaf future goes to store
 its Waker and register the event source it’s waiting for, the executor it was built against needs to have that state set up and needs to be running so that the event source will actually be monitored and wake eventually called. There are ways around this, such as having leaf futures spawn an executor if one is not already running, but this is not always advisable as it means that an application can transparently end up with multiple executors run- ning at the same time, which can reduce performance and mean you must inspect the state of multiple executors when debugging. 

Library crates that wish to support multiple executors have to be generic over their leaf resources. For example, instead of using a particular executor’s 

Asynchronous Programming **135** 

**136** Chapter 8 

TcpStream or File future type, a library can store a generic T: AsyncRead + AsyncWrite. However, the ecosystem has yet to settle on exactly what these traits should look like and which traits are needed, so for the moment it’s fairly diffi- cult to make code truly generic over the executor. For example, while AsyncRead and AsyncWrite are somewhat common across the ecosystem (or can be easily adapted if necessary), no traits currently exist for running a future in the background (*spawning*, which we’ll discuss later) or for representing a timer. 

**Waking Is a Misnomer** 

You may already have realized that Waker::wake doesn’t necessarily seem to *wake* anything. For example, for external events (as described in the previ- ous section), the executor is already awake, and it might seem silly for it to then call wake on a Waker that belongs to that executor anyway! The reality is that Waker::wake is a bit of a misnomer—in reality, it signals that a particular future is *runnable*. That is, it tells the executor that it should make sure to poll this particular future when it gets around to it rather than go to sleep again, since this future can make progress. This might wake the executor if it is currently sleeping so it will go poll that future, but that’s more of a side effect than its primary purpose. 

It is important for the executor to know which futures are runnable
 for two reasons. First, it needs to know when it can stop polling a future and go to sleep; it’s not sufficient to just poll each future until it returns Poll::Pending, since polling a later future might make it possible to progress an earlier future. Consider the case where two futures bounce messages back and forth on channels to one another. When you poll one, the other becomes ready, and vice versa. In this case, the executor should never go to sleep, as there is always more work to do. 

Second, knowing which futures are runnable lets the executor avoid polling futures unnecessarily. If an executor manages thousands of pending futures, it shouldn’t poll all of them just because an event made one of them runnable. If it did, executing asynchronous code would get very slow indeed. 

**Tasks and Subexecutors** 

The futures in an asynchronous program form a tree: a future may contain any number of other futures, which in turn may contain other futures, all the way down to the leaf futures that interact with wakers. The root of each tree is the future you give to whatever the executor’s main “run” function is. These root futures are called *tasks*, and they are the only point of contact between the executor and the futures tree. The executor calls poll on the task, and from that point forward the code of each contained future must figure out which inner future(s) to poll in response, all the way down to the relevant leaf. 

Executors generally construct a separate Waker for each task they poll so that when wake is later called, they know which task was just made runnable and can mark it as such. That is what the raw pointer in RawWaker is for—to dif- ferentiate between tasks while sharing the code for the various Waker methods. 

When the executor eventually polls a task, that task starts running from the top of its implementation of Future::poll and must decide from there how 

to get to the future deeper down that can now make progress. Since each future knows only about its own fields, and nothing about the whole tree, this all happens through calls to poll that each traverse one edge in the tree. 

The choice of which inner future to poll is often obvious, but not always. In the case of async/await, the future to poll is the one we’re blocked waiting for. But in a future that waits for the first of several futures to make prog- ress (often called a *select*), or for all of a set of futures (often called a *join*), there are many options. A future that has to make such a choice is basically a subexecutor. It could poll all of its inner futures, but doing so could be quite wasteful. Instead, these subexecutors often wrap the Waker they receive in poll’s Context with their own Waker type before they invoke poll on any inner future. In the wrapping code, they mark the future they just polled as runnable in their own state before they call wake on the original Waker. That way, when the executor eventually polls the subexecutor future again, the subexecutor can consult its own internal state to figure out which of its inner futures caused the current call to poll, and then only poll those. 

**BLOCKING IN ASYNC CODE** 

You must be careful about calling synchronous code from asynchronous code, since any time an executor thread spends executing the current task is time it’s not spending running other tasks . If a task occupies the current thread for a prolonged period of time without yielding back to the executor, which might happen when executing a blocking system call (like std::sync::sleep), running a subexecutor that doesn’t yield occasionally, or running in a tight loop with no awaits, then other tasks the current executor thread is responsible for won’t get to run during that time . Usually, this manifests as long delays between when certain tasks can make progress (such as when a client connects) and when they actually get to execute . 

Some multithreaded executors implement work-stealing techniques, where idle executor threads steal tasks from busy executor threads, but this is more of a mitigation than a solution . Ultimately, you could end up in a situation where all the executor threads are blocked, and thus no tasks get run until one of the blocking operations completes . 

In general, you should be very careful with executing compute-intensive operations or calling functions that could block in an asynchronous context . Such operations should either be converted to asynchronous operations where possible or executed on dedicated threads that then communicate using a primitive that does support asynchrony, like a channel . Some executors also provide mechanisms for indicating that a particular segment of asynchronous code might block or for yielding voluntarily in the context of loops that might otherwise not yield, which can compose part of the solution . A good rule of thumb is that no future should be able to run for more than 1 ms without return- ing Poll::Pending . 

Asynchronous Programming **137** 

**Tying It All Together with spawn** 

When working with asynchronous executors, you may come across an operation that spawns a future. We’re now in a position to explore what that means! Let’s do so by way of example. First, consider the simple server implementation in Listing 8-14. 

```
 async fn handle_client(socket: TcpStream) -> Result<()> {
	 // Interact with the client over the given socket.
 } 
 async fn server(socket: TcpListener) -> Result<()> {
	 while let Some(stream) = socket.accept().await? {
		 handle_client(stream).await?;
	 }
 } 
```

Listing 8-14: Handling connections sequentially 

The top-level server function is essentially one big future that listens for new connections and does something when a new connection arrives. You hand that future to the executor and say “run this,” and since you don’t want your program to then exit immediately, you’ll probably have the exec- utor block on that future. That is, the call to the executor to run the server future will not return until the server future resolves, which may be never (another client could always arrive later). 

Now, every time a new client connection comes in, the code in Listing 8-14 makes a new future (by calling handle_client) to handle that connection. Since the handling is itself a future, we await it and then move on to the next client connection. 

The downside of this approach is that we only ever handle one connec- tion at a time—there is no concurrency. Once the server accepts a connec- tion, the handle_client function is called, and since we await it, we don’t go around the loop again until handle_client’s return future resolves (presum- ably when that client has left). 

We could improve on this by keeping a set of all the client futures and having the loop in which the server accepts new connections also check all the client futures to see if any can make progress. Listing 8-15 shows what that might look like. 

```
 async fn server(socket: TcpListener) -> Result<()> {
	 let mut clients = Vec::new();
	 loop {
		 poll_client_futures(&mut clients)?;
		 if let Some(stream) = socket.try_accept()? {
			 clients.push(handle_client(stream));
		 }
	  } }   
```

Listing 8-15: Handling connections with a manual executor 



This at least handles many connections concurrently, but it’s quite convoluted. It’s also not very efficient because the code now busy-loops, switching between handling the connections we already have and accept- ing new ones. And it has to check each connection each time, since it won’t know which ones can make progress (if any). It also can’t await at any point, since that would prevent the other futures from making progress. You could implement your own wakers to ensure that the code polls only the futures that can make progress, but ultimately this is going down the path of devel- oping your own mini-executor. 

Another downside of sticking with just the one task for the server that internally contains the futures for all of the client connections is that the server ends up being single-threaded. There is just the one task and to poll it the code must hold an exclusive reference to the task’s future (poll takes Pin<&mut Self>), which only one thread can hold at a time. 

The solution is to make each client future its own task and leave it to the executor to multiplex among all the tasks. Which, you guessed it, you do by spawning the future. The executor will continue to block on the server future, but if it cannot make progress on that future, it will use its execution machinery to make progress on the other tasks in the meantime behind the scenes. And best of all, if the executor is multithreaded and your client futures are Send, it can run them in parallel since it can hold &muts to the separate tasks concurrently. Listing 8-16 gives an example of what this might look like. 

```
async fn server(socket: TcpListener) -> Result<()> {
    while let Some(stream) = socket.accept().await? {
        // Spawn a new task with the Future that represents this client.
        // The current task will continue to just poll for more connections
        // and will run concurrently (and possibly in parallel) with handle_client.
        spawn(handle_client(stream));
	} } 
```
Listing 8-16: Spawning futures to create more tasks that can be polled concurrently 

When you spawn a future and thus make it a task, it’s sort of like spawn- ing a thread. The future continues running in the background and is mul- tiplexed concurrently with any other tasks given to the executor. However, unlike a spawned thread, spawned tasks still depend on being polled by the executor. If the executor stops running, either because you drop it or because your code no longer runs the executor’s code, those spawned tasks will stop making progress. In the server example, imagine what will hap- pen if the main server future resolves for some reason. Since the executor has returned control back to your code, it cannot continue doing, well, anything. Multi-threaded executors often spawn background threads that continue to poll tasks even if the executor yields control back to the user’s code, but not all executors do this, so check your executor before you rely on that behavior! 


## Summary

In this chapter, we’ve taken a look behind the scenes of the asynchronous constructs available in Rust. We’ve seen how the compiler implements gen- erators and self-referential types, and why that work was necessary to sup- port what we now know as async/await. We’ve also explored how futures are executed, and how wakers allow executors to multiplex among tasks when only some of them can make progress at any given moment. In the next chapter, we’ll tackle what is perhaps the deepest and most discussed area of Rust: unsafe code. Take a deep breath, and then turn the page. 