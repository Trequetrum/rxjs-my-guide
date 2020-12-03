# Making an Observable

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

---

### It Could Be Easier

Here's a few ways to create the same observable using RxJS' creation operators:

```JavaScript
const ob$ = of(
  { message: "This is an object" },
  564,
  "This is a string"
)

const ob$ = from([
  { message: "This is an object" },
  564,
  "This is a string"
])

const ob$ = defer(() => ([
  { message: "This is an object" },
  564,
  "This is a string"
]))
```
