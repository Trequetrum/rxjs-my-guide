# RxJS The Basics

An array is to memory what an observable is to time. It represents a stream of data. The 2nd piece of data comes after the 1st and before the 3rd. That’s a sort of vague answer, but if you’re newer to functional/reactive/asynconous programming, vague is good for now. Thinking too much about the paradigm can bog you down. 

RxJS lets the paradigm follow very naturally. You’ll learn to see the beauty in avoiding shared state, mutable data, and side-effects. You’ll learn to appreciate how events naturally tell you about program state. You’ll love that the order things execute are encoded neatly in your program and don’t sit abstractly in your head.

That all comes on its own. It doesn’t need a thorough explanation. *“The proof of the pudding is in the eating”*

----

### So it’s like an array?
 
A lot of the abstractions that you can apply to an array, you can use with an observable. They both represent sets of data and are otherwise not very constrained. A good place to start is to understand some of the basic operations you may choose to perform with a stream and how they compare with the same operators done to an array.

```JavaScript
const result = [1,2,3,4,5,6].
  filter(x => x != 3).
  map(x => x * 5).
  slice(0,4);
console.log(result);
//output: [5,10,20,25]
```
```JavaScript
of(1,2,3,4,5,6).pipe(
  filter(x => x != 3),
  map(x => x * 5),
  take(4)
).subscribe(result =>
  console.log(result)
);
// output: 5 10 20 25
```

Here we have two snippets of code that are performing some operations on a set of numbers. Their result is roughly the same. The second snippet is using observables so `of`, `pipe`, and `subscribe` might seem foreign.

If you run these two snippets step-by-step through a debugger, you’ll notice a pretty stark difference. In the `Array` snippet, every value is filtered before the first value is mapped. Then every value is mapped before the resulting array is sliced.

In the `Observable` snippet, the 1st value passes the filter, then gets mapped (multiplied by 5), then is counted as the 1st of 4 values that get accepted. All that happens before the 2nd value passes the filter.

Once the observable stream has 4 values, it doesn’t need any more. The second snippet never checks if `6 != 3` and never mapped `6 => 30`. It just stops after 4 values make it to the `take(4)` operator.

The `Array` snippet, on the other hand, does this extra work. Each step along the way has no way to know that at the end of it all, the last value will get tossed. If the computation on 6 was expensive, that’d be wasted time.

This isn’t the problem Observables were built to solve, but this benefit is one of the side effects of this approach. Observables must function this way because they don’t know if the stream they’re operating over will ever end, nor how long they must wait between values. If you’ve worked with generators before, this may all come naturally but otherwise it may take some time/effort for this to properly make sense.

----

### Making an Observable

Generally, you’re unlikely to need to make an Observable from scratch. Many libraries return Observables. You can turn any iterable (like arrays), promises, or observable-like data into an *RxJS Observable* using `from`. You can turn any piece of data into a stream with `of` and you can pass as many parameters into `of` as you’d like.

Here’s a from-scratch Observable that emits a few things:

```JavaScript
const ob$ = new Observable(observer => {

  observer.next({message: “This is an object”});
  observer.next(564);
  observer.next(“This is a string”);

  observer.complete();

  return { 
    unsubscribe: () => {/* Do nothing */}
  }

});
```

Okay, so what’s happening here?

`ob$` is a stream. If given an observer, it will run the code above. In this case, the observer it’s given needs to have defined two functions. A ‘next’ function and a `complete` function (We'll ignore errors for now, but they get a callback too). 

How do we give `ob$` an observer? We do this by subscribing. 

```JavaScript
const observer = {
  next: value => console.log(“Our observer’s next callback was called with the value: “, value),
  complete: () => console.log(“Our observer’s complete callback was called”)
}
ob$.subscribe(observer);
```

It is at the moment we subscribe (and not before) that the code we gave `ob$`’s constructor above gets run.

Here’s the output:

```
Our observer’s next callback was called with the value: { message: This is an object }
Our observer’s next callback was called with the value: 564
Our observer’s next callback was called with the value: This is a string
Our observer’s complete callback was called
```

 
