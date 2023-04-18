# Chapter2.The What, Where, When, and How of Data Processing

摘录来自
Streaming Systems
Tyler Akidau
https://itunes.apple.com/WebObjects/MZStore.woa/wa/viewBook?id=0
此材料可能受版权保护。
As discussed in Chapter 1, windowing is the process of slicing up a data source along temporal boundaries. Common windowing strategies include fixed windows, sliding windows, and sessions windows, as demonstrated in Figure 2-4.

![image-20220410233440292](/Users/Laobe/Documents/images/image-20220410233440292.png)

<center><i>Figure 2-4. Example windowing strategies. Each example is shown for three different keys, highlighting the difference between aligned windows (which apply across all the data) and unaligned windows (which apply across a subset of the data).</i></center>
To get a better sense of what windowing looks like in practice, let’s take our integer summation pipeline and window it into fixed, two-minute windows. With Beam, the change is a simple addition of a Window.into transform, which you can see highlighted in Example 2-2.

Example 2-2. Windowed summation code

```java
 PCollection<KV<Team, Integer>> totals = input
 .apply(Window.into(FixedWindows.of(TWO_MINUTES)))
  .apply(Sum.integersPerKey());

```

Recall that Beam provides a unified model that works in both batch and streaming because semantically batch is really just a subset of streaming. As such, let’s first execute this pipeline on a batch engine; the mechanics are more straightforward, and it will give us something to directly compare against when we switch to a streaming engine. Figure 2-5 presents the result.

![image-20220410233616375](/Users/Laobe/Documents/images/image-20220410233616375.png)

<center><i>Figure 2-5. Windowed summation on a batch engine</i></center>
As before, inputs are accumulated in state until they are entirely consumed, after which output is produced. In this case, however, instead of one output, we get four: a single output, for each of the four relevant two-minute event-time windows.

At this point we’ve revisited the two main concepts that I introduced in Chapter 1: the relationship between the event-time and processing-time domains, and windowing. If we want to go any further, we’ll need to start adding the new concepts mentioned at the beginning of this section: triggers, watermarks, and accumulation.

### Going Streaming: When and How

We just observed the execution of a windowed pipeline on a batch engine. But, ideally, we’d like to have lower latency for our results, and we’d also like to natively handle unbounded data sources. Switching to a streaming engine is a step in the right direction, but our previous strategy of waiting until our input has been consumed in its entirety to generate output is no longer feasible. Enter triggers and watermarks.

#### When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!

Triggers provide the answer to the question: “When in processing time are results materialized?” Triggers declare when output for a window should happen in processing time (though the triggers themselves might make those decisions based on things that happen in other time domains, such as watermarks progressing in the event-time domain, as we’ll see in a few moments). Each specific output for a window is referred to as a pane of the window.

Though it’s possible to imagine quite a breadth of possible triggering semantics,3 conceptually there are only two generally useful types of triggers, “and practical applications almost always boil down using either one or a combination of both:

Repeated update triggers

​		These periodically generate updated panes for a window as its contents evolve. These updates can be materialized with every new record, or they can happen after some processing-time delay, such as once a minute. The choice of period for a repeated update trigger is primarily an exercise in balancing latency and cost.

Completeness triggers

​		These materialize a pane for a window only after the input for that window is believed to be complete to some threshold. This type of trigger is most analogous to what we’re familiar with in batch processing: only after the input is complete do we provide a result. The difference in the trigger-based approach is that the notion of completeness is scoped to the context of a single window, rather than always being bound to the completeness of the entire input.

Repeated update triggers are the most common type of trigger encountered in streaming systems. They are simple to implement and simple to understand, and they provide useful semantics for a specific type of use case: repeated (and eventually consistent) updates to a materialized dataset, analogous to the semantics you get with materialized views in the database world.

Completeness triggers are less frequently encountered, but provide streaming semantics that more closely align with those from the classic batch processing world. They also provide tools for reasoning about things like missing data and late data, both of which we discuss shortly (and in the next chapter) as we explore the underlying primitive that drives completeness triggers: watermarks.

But first, let’s start simple and look at some basic repeated update triggers in action. To make the notion of triggers a bit more concrete, let’s go ahead and add the most straightforward type of trigger to our example pipeline: a trigger that fires with every new record, as shown in Example 2-3.

*Example 2-3. Triggering repeatedly with every record*

```java
  PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
                .triggering(Repeatedly(AfterCount(1))));
  .apply(Sum.integersPerKey())
```

If we were to run this new pipeline on a streaming engine, the results would look something like that shown in Figure 2-6.

![image-20220410234103265](/Users/Laobe/Documents/images/image-20220410234103265.png)

<center><i>Figure 2-6. Per-record triggering on a streaming engine</i></center>
You can see how we now get multiple outputs (panes) for each window: once per corresponding input. This sort of triggering pattern works well when the output stream is being written to some sort of table that you can simply poll for results. Any time you look in the table, you’ll see the most up-to-date value for a given window, and those values will converge toward correctness over time.

One downside of per-record triggering is that it’s quite chatty. When processing large-scale data, aggregations like summation provide a nice opportunity to reduce the cardinality of the stream without losing information. This is particularly noticeable for cases in which you have high-volume keys; for our example, massive teams with lots of active players. Imagine a massively multiplayer game in which players are split into one of two factions, and you want to tally stats on a per-faction basis. It’s probably unnecessary to update your tallies with every new input record for every player in a given faction. Instead, you might be happy updating them after some processing-time delay, say every second, or every minute. The nice side effect of using processing-time delays is that it has an equalizing effect across high-volume keys or windows: the resulting stream ends up being more uniform cardinality-wise.

There are two different approaches to processing-time delays in triggers: aligned delays (where the delay slices up processing time into fixed regions that align across keys and windows) and unaligned delays (where the delay is relative to the data observed within a given window). A pipeline with unaligned delays might look like Example 2-4, the results of which are shown in Figure 2-7.

*Example 2-4. Triggering on aligned two-minute processing-time boundaries*

```java
  PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(Repeatedly(AlignedDelay(TWO_MINUTES)))
  .apply(Sum.integersPerKey());```
```

![image-20220410235350595](/Users/Laobe/Documents/images/image-20220410235350595.png)

<center><i>Figure 2-7. Two-minute aligned delay triggers (i.e., microbatching)

This sort of aligned delay trigger is effectively what you get from a microbatch streaming system like Spark Streaming. The nice thing about it is predictability; you get regular updates across all modified windows at the same time. That’s also the downside: all updates happen at once, which results in bursty workloads that often require greater peak provisioning to properly handle the load. The alternative is to use an unaligned delay. That would look something Example 2-5 in Beam. Figure 2-8 presents the results.

*Example 2-5. Triggering on unaligned two-minute processing-time boundaries*

``` java
  PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(Repeatedly(UnalignedDelay(TWO_MINUTES))
  .apply(Sum.integersPerKey()); 
```

![image-20220410235955884](/Users/Laobe/Documents/images/image-20220410235955884.png)

<center><i>Figure 2-8. Two-minute unaligned delay triggers
Contrasting the unaligned delays in Figure 2-8 to the aligned delays in Figure 2-6, it’s easy to see how the unaligned delays spread the load out more evenly across time. The actual latencies involved for any given window differ between the two, sometimes more and sometimes less, but in the end the average latency will remain essentially the same. From that perspective, unaligned delays are typically the better choice for large-scale processing because they result in a more even load distribution over time.

Repeated update triggers are great for use cases in which we simply want periodic updates to our results over time and are fine with those updates converging toward correctness with no clear indication of when correctness is achieved. However, as we discussed in Chapter 1, the vagaries of distributed systems often lead to a varying level of skew between the time an event happens and the time it’s actually observed by your pipeline, which means it can be difficult to reason about when your output presents an accurate and complete view of your input data. For cases in which input completeness matters, it’s important to have some way of reasoning about completeness rather than blindly trusting the results calculated by whichever subset of data happen to have found their way to your pipeline. Enter watermarks.

#### *When*: Watermarks

Watermarks are a supporting aspect of the answer to the question: “When in processing time are results materialized?” Watermarks are temporal notions of input completeness in the event-time domain. Worded differently, they are the way the system measures progress and completeness relative to the event times of the records being processed in a stream of events (either bounded or unbounded, though their usefulness is more apparent in the unbounded case).

Recall this diagram from Chapter 1, slightly modified in Figure 2-9, in which I described the skew between event time and processing time as an ever-changing function of time for most real-world distributed data processing systems.
![image-20220411000128928](/Users/Laobe/Documents/images/image-20220411000128928.png)

<center><i>Figure 2-9. Event-time progress, skew, and watermarks
That meandering red line that I claimed represented reality is essentially the watermark; it captures the progress of event-time completeness as processing time progresses. Conceptu“ally, you can think of the watermark as a function, F(P) → E, which takes a point in processing time and returns a point in event time.4 That point in event time, E, is the point up to which the system believes all inputs with event times less than E have been observed. In other words, it’s an assertion that no more data with event times less than E will ever be seen again. Depending upon the type of watermark, perfect or heuristic, that assertion can be a strict guarantee or an educated guess, respectively:

Perfect watermarks

​		For the case in which we have perfect knowledge of all of the input data, it’s possible to construct a perfect watermark. In such a case, there is no such thing as late data; all data are early or on time.

Heuristic watermarks

​		For many distributed input sources, perfect knowledge of the input data is impractical, in which case the next best option is to provide a heuristic watermark. Heuristic watermarks use whatever information is available about the inputs (partitions, ordering within partitions if any, growth rates of files, etc.) to provide an estimate of progress that is as accurate as “possible. In many cases, such watermarks can be remarkably accurate in their predictions. Even so, the use of a heuristic watermark means that it might sometimes be wrong, which will lead to late data. We show you about ways to deal with late data soon.

Because they provide a notion of completeness relative to our inputs, watermarks form the foundation for the second type of trigger mentioned previously: completeness triggers. Watermarks themselves are a fascinating and complex topic, as you’ll see when you get to Slava’s watermarks deep dive in Chapter 3. But for now, let’s look at them in action by updating our example pipeline “to utilize a completeness trigger built upon watermarks, as demonstrated in Example 2-6.

*Example 2-6. Watermark completeness trigger*

``` java
  PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(AfterWatermark()))
  .apply(Sum.integersPerKey()); 
```

Now, an interesting quality of watermarks is that they are a class of functions, meaning there are multiple different functions F(P) → E that satisfy the properties of a watermark, to varying degrees of success. As I noted earlier, for situations in which you have perfect knowledge of your input data, it might be possible to build a perfect watermark, which is the ideal situation. But for cases in which you lack perfect knowledge of the inputs or for which it’s simply too computationally expensive to calculate the perfect watermark, you might instead choose to utilize a heuristic for defining your watermark. The point I want to make here is that the given watermark algorithm in use is independent from the pipeline itself. We’re not going to discuss in detail what it means to implement a watermark here (Slava does that in Chapter 3). For now, to help drive home this idea that a given input set can have different watermarks applied to it, let’s take a look at our pipeline in Example 2-6 when executed on the same dataset but using two distinct watermark implementations (Figure 2-10): on the left, a perfect watermark; on the right, a heuristic watermark.

In both cases, windows are materialized as the watermark passes the end of the window. The perfect watermark, as you might expect, perfectly captures the event-time completeness of the pipeline as time progresses. In contrast, the specific algorithm used for the heuristic watermark on the right fails to take the value of 9 into account,5 which drastically changes the shape of the materialized outputs, both in terms of output latency and correctness (as seen by the incorrect answer “of 5 that’s provided for the [12:00, 12:02) window).

The big difference between the watermark triggers from Figure 2-9 and the repeated update triggers we saw in Figures 2-5 through 2-7 is that the watermarks give us a way to reason about the completeness of our input. Until the system materializes an output for a given window, we know that the system does not yet believe the inputs to be complete. This is especially important for use cases in which you want to reason about a lack of data in the input, or missing data.

![image-20220411000439474](/Users/Laobe/Documents/images/image-20220411000439474.png)

<center><i>Figure 2-10. Windowed summation on a streaming engine with perfect (left) and heuristic (right) watermarks
A great example of a missing-data use case is outer joins. Without a notion of completeness like watermarks, how do you know when to give up and emit a partial join rather than continue to wait for that join to complete? You don’t. And basing that decision on a processing-time delay, which is the common approach in streaming systems that lack true watermark support, is not a safe way to go, because of the variable nature of event-time skew we “spoke about in Chapter 1: as long as skew remains smaller than the chosen processing-time delay, your missing-data results will be correct, but any time skew grows beyond that delay, they will suddenly become incorrect. From this perspective, event-time watermarks are a critical piece of the puzzle for many real-world streaming use cases which must reason about a lack of data in the input, such as outer joins, anomaly detection, and so on.

Now, with that said, these watermark examples also highlight two shortcomings of watermarks (and any other notion of completeness), specifically that they can be one of the following:

Too slow

​		When a watermark of any type is correctly delayed due to known unprocessed data (e.g., slowly growing input logs due to network bandwidth constraints), that translates directly into delays in output if advancement of the watermark is the only thing you depend on for stimulating results.

​		This is most obvious in the left diagram of Figure 2-10, for which the late arriving 9 holds back the watermark for all the subsequent windows, even though the input data for those windows become complete earlier. This is particularly apparent for the second window, [12:02, 12:04), for which it takes nearly seven minutes from the time the first value in the window occurs until we see any results for the window whatsoever. The heuristic watermark in this example doesn’t suffer the same issue quite so badly (five minutes until output), but don’t take that to mean heuristic watermarks never suffer from watermark lag; that’s really just a consequence of the record I chose to omit from the heuristic watermark in this specific example.

​		The important point here is the following: Although watermarks provide a very useful notion of completeness, depending upon completeness for producing output is often not ideal from a latency perspective. Imagine a dashboard that contains valuable metrics, windowed by hour or day. It’s unlikely you’d want to wait a full hour or day to begin seeing results for the current window; that’s one of the pain points of using classic batch systems to power such systems. Instead, it would be much nicer to see the results for those windows refine over time as the inputs evolve and eventually become complete.

Too fast

​		When a heuristic watermark is incorrectly advanced earlier than it should be, it’s possible for data with event times before the watermark to arrive some time later, creating late data. This is what happened in the example on the right: the watermark advanced past the end of the first window before all the input data for that window had been observed, resulting in an incorrect output value of 5 instead of 14. This shortcoming is strictly a problem with heuristic watermarks; their heuristic nature implies they will sometimes be wrong. As a result, relying on them alone for determining when to materialize output is insufficient if you care about correctness.

In Chapter 1, I made some rather emphatic statements about notions of completeness being insufficient for most use cases requiring robust out-of-order processing of unbounded data streams. These two shortcomings—watermarks being too slow or too fast—are the foundations for those arguments. You simply cannot get both low latency and correctness out of a system that relies solely on notions of completeness.6 So, for cases for which you do want the best of both worlds, what’s a person to do? Well, if repeated update triggers provide low-latency updates but no way to reason about completeness, and watermarks provide a notion of completeness but variable and possible high latency, why not combine their powers together?

#### *When*: Early/On-Time/Late Triggers FTW!

We’ve now looked at the two main types of triggers: repeated update triggers and completeness/watermark triggers. In many case, neither of them alone is sufficient, but the combination of them together is. Beam recognizes this fact by providing an extension of the standard watermark trigger that also supports repeated update triggering on either side of the watermark. This is known as the early/on-time/late trigger because it partitions the panes that are materialized by the compound trigger into three categories:

- Zero or more early panes, which are the result of a repeated update trigger that periodically fires up until the watermark passes the end of the window. The panes generated by these firings contain speculative results, but allow us to observe the evolution of the window over time as new input data arrive. This compensates for the shortcoming of watermarks sometimes being too slow.
- A single on-time pane, which is the result of the completeness/watermark trigger firing after the watermark passes the end of the window. This firing is special because it provides an assertion that the system now believes the input for this window to be complete.7 This means that it is now safe to reason about missing data; for example, to emit a partial join when performing an outer join.
- Zero or more late panes, which are the result of another (possibly different) repeated update trigger that periodically fires any time late data arrive after the watermark has passed the end of the window. In the case of a perfect watermark, there will always be zero late panes. But in the case of a heuristic watermark, any data the watermark failed to properly account for will result in a late firing. This compensates for the shortcoming of watermarks being too fast.

Let’s see how this looks in action. We’ll update our pipeline to use a periodic processing-time trigger with an aligned delay of one minute for the early firings, and a per-record trigger for the late firings. That way, the early firings will give us some amount of batching for high-volume windows (thanks to the fact that the trigger will fire only once per minute, regardless of the throughput into the window), but we won’t introduce unnecessary latency for the late firings, which are hopefully somewhat rare if we’re using a reasonably accurate heuristic watermark. In Beam, that looks Example 2-7 (Figure 2-11 shows the results).

*Example 2-7. Early, on-time, and late firings via the early/on-time/late API*

``` java
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(AfterWatermark()
			     .withEarlyFirings(AlignedDelay(ONE_MINUTE))
			     .withLateFirings(AfterCount(1))))
  .apply(Sum.integersPerKey());
```

![image-20220411011728041](/Users/Laobe/Documents/images/image-20220411011728041.png)

<center><i>Figure 2-11. Windowed summation on a streaming engine with early, on-time, and late firings

This version has two clear improvements over Figure 2-9:

- For the “watermarks too slow” case in the second window, [12:02, 12:04): we now provide periodic early updates once per minute. The difference is most stark in the perfect watermark case, for which time-to-first-output is reduced from almost seven minutes down to three and a half; but it’s also clearly improved in the heuristic case, as well. Both versions now provide steady refinements over time (panes with values 7, 10, then 18), with relatively minimal latency between the input becoming complete and materialization of the final output pane for the window.

- For the “heuristic watermarks too fast” case in the first window, [12:00, 12:02): when the value of 9 shows up late, we immediately incorporate it into a new, corrected pane with value of 14.

One interesting side effect of these new triggers is that they effectively normalize the output pattern between the perfect and heuristic watermark versions. Whereas the two versions in Figure 2-10 were starkly different, the two versions here look quite similar. They also look much more similar to the various repeated update version from Figures 2-6 through 2-8, with one important difference: thanks to the use of the watermark trigger, we can also reason about input completeness in the results we generate with the early/on-time/late trigger. This allows us to better handle use cases that care about missing data, like outer joins, anomaly detection, and so on.

The biggest remaining difference between the perfect and heuristic early/on-time/late versions at this point is window lifetime bounds. In the perfect watermark case, we know we’ll never see any more data for a window after the watermark has passed the end of it, hence we can drop all of our state for the window at that time. In the heuristic watermark case, we still need to hold on to the state for a window for some amount of time to account for late data. But as of yet, our system doesn’t have any good way of knowing just how long state needs to be kept around for each window. That’s where allowed lateness comes in.

#### *When*: Allowed Lateness (i.e., Garbage Collection)

Before moving on to our last question (“How do refinements of results relate?”), I’d like to touch on a practical necessity within long-lived, out-of-order stream processing systems: garbage collection. In the heuristic watermarks example in Figure 2-11, the persistent state for each window lingers around for the entire lifetime of the example; this is necessary to allow us to appropriately deal with late data when/if they arrive. But while it would be great to be able to keep around all of our persistent state until the end of time, in reality, when dealing with an unbounded data source, it’s often not practical to keep state (including metadata) for a given window indefinitely; we’ll eventually run out of disk space (or at the very least tire of paying for it, as the value for older data diminishes over time).

As a result, any real-world out-of-order processing system needs to provide some way to bound the lifetimes of the windows it’s processing. A clean and concise way of doing this is by defining a horizon on the allowed lateness within the system; that is, placing a bound on how late any given record may be (relative to the watermark) for the system to bother processing it; any data that arrives after this horizon are simply dropped. After you’ve bounded how late individual data may be, you’ve also established precisely how long the state for windows must be kept around: until the watermark exceeds the lateness horizon for the end of the window. But in addition, you’ve also given the system the liberty to immediately drop any data later than the horizon as soon as they’re observed, which means the system doesn’t waste resources processing data that no one cares about.

> <center>MEASURING LATENESS</center>
>It might seem a little odd to be specifying a horizon for handling late data using the very metric that resulted in the late data in the first place (i.e., the heuristic watermark). And in some sense it is. But of the options available, it’s arguably the best. The only other practical option would be to specify the horizon in processing time (e.g., keep windows around for 10 minutes of processing time after the watermark passes the end of the window), but using processing time would leave the garbage collection policy vulnerable to issues within the pipeline itself (e.g., workers crashing, causing the pipeline to stall for a few minutes), which could lead to windows that didn’t actually have a chance to handle late data that they otherwise should have. By specifying the horizon in the event-time domain, garbage collection is directly tied to the actual progress of the pipeline, which decreases the likelihood that a window will miss its opportunity to handle late data appropriately.
> 
>Note however, that not all watermarks are created equal. When we speak of watermarks in this book, we generally refer to low watermarks, which pessimistically attempt to capture the event time of the oldest unprocessed record the system is aware of. The nice thing about dealing with lateness via low watermarks is that they are resilient to changes in event-time skew; no matter how large the skew in a pipeline may grow, the low watermark will always track the oldest outstanding event known to the system, providing the best guarantee of correctness possible.
> 
>In contrast, some systems may use the term “watermark” to mean other things. For example, watermarks in Spark Structured Streaming are high watermarks, which optimistically track the event time of the newest record the system is aware of. When dealing with lateness, the system is free to garbage collect any window older than the high watermark adjusted by some user-specified lateness threshold. In other words, the system allows you to specify the maximum amount of event-time skew you expect to see in your pipeline, and then throws away any data outside of that skew window. This can work well if skew within your pipeline remains within some constant delta, but is more prone to incor-rectly discarding data than low watermarking schemes.

Because the interaction between allowed lateness and the watermark is a little subtle, it’s worth looking at an example. Let’s take the heuristic watermark pipeline from Example 2-7/Figure 2-11 and add in Example 2-8 a lateness horizon of one minute (note that this particular horizon has been chosen strictly because it fits nicely into the diagram; for real-world use cases, a larger horizon would likely be much more practical):

*Example 2-8. Early/on-time/late firings with allowed lateness*

```java
 PCollection<KV<Team, Integer>> totals = input
 .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(
                 AfterWatermark()
                   .withEarlyFirings(AlignedDelay(ONE_MINUTE))
                   .withLateFirings(AfterCount(1)))
               .withAllowedLateness(ONE_MINUTE))
 .apply(Sum.integersPerKey());
```

The execution of this pipeline would look something like Figure 2-12, in which I’ve added the following features to highlight the effects of allowed lateness:

- The thick black line denoting the current position in processing time is now annotated with ticks indicating the lateness horizon (in event time) for all active windows.

- When the watermark passes the lateness horizon for a window, that window is closed, which means that all state for the window is discarded. I leave around a dotted rectangle showing the extent of time (in both domains) that the window covered when it was closed, with a little tail extending to the right to denote the “lateness horizon for the window (for contrasting against the watermark).
- For this diagram only, I’ve added an additional late datum for the first window with value 6. The 6 is late, but still within the allowed lateness horizon and thus is incorporated into an updated result with value 11. The 9, however, arrives beyond the lateness horizon, so it is simply dropped.

![image-20220411013108095](/Users/Laobe/Documents/images/image-20220411013108095.png)

<center><i>“Figure 2-12. Allowed lateness with early/on-time/late firings”
Two final side notes about lateness horizons:

- To be absolutely clear, if you happen to be consuming data from sources for which perfect watermarks are available, there’s no need to deal with late data, and an allowed lateness horizon of zero seconds will be optimal. This is what we saw in the perfect watermark portion of Figure 2-10.
- One noteworthy exception to the rule of needing to specify lateness horizons, even when heuristic watermarks are in use, would be something like computing global aggregates over all time for a tractably finite number of keys (e.g., computing the total number of visits to your site over all time, grouped by web browser family). In this case, the number of active windows in the system is bounded by the limited keyspace in use. As long as the number of keys remains manageably low, there’s no need to worry about limiting the lifetime of windows via allowed lateness.

Practicality sated, let’s move on to our fourth and final question.

#### *How*: Accumulation

When triggers are used to produce multiple panes for a single window over time, we find ourselves confronted with the last question: “How do refinements of results relate?” In the examples we’ve seen so far, each successive pane is built upon the one immediately preceding it. However, there are actually three8 different modes of accumulation:

Discarding

​		Every time a pane is materialized, any stored state is discarded. This means that each successive pane is independent from any that came before. Discarding mode is useful when the downstream consumer is performing some sort of accumulation itself; for example, when sending integers into a system that expects to receive deltas that it will sum together to produce a final count.

Accumulating

​		As in Figures 2-6 through 2-11, every time a pane is materialized, any stored state is retained, and future inputs are accumulated “into the existing state. This means that each successive pane builds upon the previous panes. Accumulating mode is useful when later results can simply overwrite previous results, such as when storing output in a key/value store like HBase or Bigtable.

Accumulating and retracting

​		This is like accumulating mode, but when producing a new pane, it also produces independent retractions for the previous pane(s). Retractions (combined with the new accumulated result) are essentially an explicit way of saying “I previously told you the result was X, but I was wrong. Get rid of the X I told you last time, and replace it with Y.” There are two cases for which retractions are particularly helpful:

- When consumers downstream are regrouping data by a different dimension, it’s entirely possible the new value may end up keyed differently from the previous value and thus end up in a different group. In that case, the new value can’t just overwrite the old value; you instead need the retraction to remove the old value
- When dynamic windows (e.g., sessions, which we look at more closely in a few moments) are in use, the new value might be replacing more than one previous window, due to window merging. In this case, it can be difficult to determine from the new window alone which old windows are being replaced. Having explicit retractions for the old windows makes the task straightforward. We see an example of this in detail in Chapter 8.

The different semantics for each group are somewhat clearer when seen side-by-side. Consider the two panes for the second window (the one with event-time range [12:06, 12:08)) in Figure 2-11 (the one with early/on-time/late triggers). Table 2-1 shows what the values for each pane would look like across the three accumulation modes (with accumulating mode being the specific mode used in Figure 2-11 itself).

*Table 2-1. Comparing accumulation modes using the second window from Figure 2-11*

​											Discarding  Accumulating Accumulating & Retracting

<hr/>
Pane 1: inputs=[3]  			3 3 3

<hr>

Pane 2: inputs=[8, 1] 		9 12 12, –3

<hr>

Value of final normal pane 	9 12 12 

<hr>

Sum of all panes 					12 15 12

<hr>

Let’s take a closer look at what’s happening:

Discarding

​		Each pane incorporates only the values that arrived during that specific pane. As such, the final value observed does not fully capture the total sum. However, if you were to sum all of the independent panes themselves, you would arrive at a correct answer of 12. This is why discarding mode is useful when the downstream consumer itself is performing some sort of aggregation on the materialized panes.

Accumulating

​		As in Figure 2-11, each pane incorporates the values that arrived during that specific pane, plus all of the values from previous panes. As such, the final value observed correctly captures the total sum of 12. If you were to sum up the individual panes themselves, however, you’d effectively be double-counting the inputs from pane 1, giving you an incorrect total sum of 15. This is why accumulating mode is most useful when you can simply overwrite previous values with new values: the new value already incorporates all of the data seen thus far.

Accumulating and retracting

​		Each pane includes both a new accumulating mode value as well as a retraction of the previous pane’s value. As such, both the last value observed (excluding retractions) as well as the total sum of all materialized panes (including retractions) provide you with the correct answer of 12. This is why retractions are so powerful.

Example 2-9 demonstrates discarding mode in action, illustrating the changes we would make to Example 2-7:

*Example 2-9. Discarding mode version of early/on-time/late firings*

```java
PCollection<KV<Team, Integer>> totals = input 
 .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(
                 AfterWatermark()
                   .withEarlyFirings(AlignedDelay(ONE_MINUTE))
                   .withLateFirings(AtCount(1)))
               .discardingFiredPanes())
  .apply(Sum.integersPerKey()); 
```

Running again on a streaming engine with a heuristic watermark would produce output like that shown in Figure 2-13.

![image-20220411100332642](/Users/Laobe/Documents/images/image-20220411100332642.png)

<center><i>Figure 2-13. Discarding mode version of early/on-time/late firings on a streaming engine
Even though the overall shape of the output is similar to the accumulating mode version from Figure 2-11, note how none of the panes in this discarding version overlap. As a result, each output is independent from the others.
If we want to look at retractions in action, the change would be similar, as shown in Example 2-10. ??? depicts the results.

*Example 2-10. Accumulating and retracting mode version of early/on-time/late firings*

````java
PCollection<KV<Team, Integer>> totals = input
	.apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(
                 AfterWatermark()
                   .withEarlyFirings(AlignedDelay(ONE_MINUTE))
                   .withLateFirings(AtCount(1)))
               .accumulatingAndRetractingFiredPanes())
  .apply(Sum.integersPerKey());
````

![image-20220411100511977](/Users/Laobe/Documents/images/image-20220411100511977.png)

<center><i>Accumulating and retracting mode version of early/late firings on a streaming engine
Because the panes for each window all overlap, it’s a little tricky to see the retractions clearly. The retractions are indicated in red, which combines with the overlapping blue panes to yield a slightly purplish color. I’ve also horizontally shifted the values of the two outputs within a given pane slightly (and separated them with a comma) to make them easier to differentiate.

![image-20220411100715173](/Users/Laobe/Documents/images/image-20220411100715173.png)

<center><i>Figure 2-14 combines the final frames of Figures 2-9, 2-11 (heuristic only), and side-by-side, providing a nice visual contrast of the three modes.

“As you can imagine, the modes in the order presented (discarding, accumulating, accumulating and retracting) are each successively more expensive in terms of storage and computation costs. To that end, choice of accumulation mode provides yet another dimension for making trade-offs along the axes of correctness, latency, and cost.

### Summary

With this chapter complete, you now understand the basics of robust stream processing and are ready to go forth into the world and do amazing things. Of course, there are eight more chapters anxiously waiting for your attention, so hopefully you won’t go forth like right now, this very minute. But regardless, let’s recap what we’ve just covered, lest you forget any of it in your haste to amble forward. First, the major concepts we touched upon:

Event time versus processing time

​		The all-important distinction between when events occurred and when they are observed by your data processing system.

Windowing

​		The commonly utilized approach to managing unbounded data by slicing it along temporal boundaries (in either processing time or event time, though we narrow the definition of windowing in the Beam Model to mean only within event time).

Triggers

​		The declarative mechanism for specifying precisely when materialization of output makes sense for your particular use case.

Watermarks

​		The powerful notion of progress in event time that provides a means of reasoning about completeness (and thus missing data) in an out-of-order processing system operating on unbounded data.

Accumulation

​		The relationship between refinements of results for a single window for cases in which it’s materialized multiple times as it evolves.

Second, the four questions we used to frame our exploration:

- What results are calculated? = transformations.
- Where in event time are results calculated? = windowing.
- When in processing time are results materialized? = triggers plus watermarks.
- How do refinements of results relate? = accumulation.

Third, to drive home the flexibility afforded by this model of stream processing (because in the end, that’s really what this is all about: balancing competing tensions like correctness, latency, and cost), a recap of the major variations in output we were able to achieve over the same dataset with only a minimal amount of code change:

![image-20220411100943349](/Users/Laobe/Documents/images/image-20220411100943349.png)![image-20220411101010199](/Users/Laobe/Documents/images/image-20220411101010199.png)![image-20220411101048378](/Users/Laobe/Documents/images/image-20220411101048378.png)



Integer summation
Example 2-1 / Figure 2-3

Integer summation
Fixed windows batch
Example 2-2 / Figure 2-5

Integer summation
Fixed windows streaming
Repeated per-record trigger
Example 2-3 / Figure 2-6



All that said, at this point, we’ve really looked at only one type of windowing: fixed windowing in event time. As we know, there are a number of dimensions to windowing, and I’d like to touch upon at least two more of those before we call it day with the Beam Model. First, however, we’re going to take a slight detour to dive deeper into the world of watermarks, as this knowledge will help frame future discussions (and be fascinating in and of itself). Enter Slava, stage right...
