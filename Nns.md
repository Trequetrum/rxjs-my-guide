# Never Nest Subscriptions

Also, never say never

One of the first things you’re likely to read as you’re learning about RxJS is that nesting subscriptions is a code smell. You shouldn’t do it. Of course, like any code smell, there’s a time and a place.

The only thing a nested subscription does that `mergeMap` can’t accomplish (and more) is that it doesn’t manage subscriptions at all. You can unsubscribe from `mergeMap` and it will, in turn, unsubscribe from all the merged streams whose values it is emitting. No chance of a memory leak. 

This is explained with more detail here: [Memory Management](MemoryManagement.md).

Here’s an example problem where the best solution I’ve come across (so far) manages its own subscriptions (and nests `subscribe()` calls 1 level deep): [Managing Observable Subscriptions with Synchronous Streams](Mosss.md)

Short of some corner cases, however, `mergeMap` is a superset of nested subscriptions.

## Unnesting subscriptions with mergeMap

TODO
