# The TransformSubject **WIP**

I've been trying to create a Subject that implicitly contains some transformation.

Consider this pattern:

```JavaScript
const inputSubject = new Subject<number>();
const resultObservable = inputSubject.pipe(
  delay(1000),
  map(x => `${x * 5}`)
);
/** ... **/
resultObservable.subscribe(console.log);
inputSubject.next(5); // delay 1s, then prints "25"
```

Here I subscribe to one object and emit on another. Even though the two lines of code are next to one another, (without looking at how `resultObservable` is created) it's not immediately clear that `inputSubject.next(5);` creates this output. It is especially bothersome if I want to return both objects to the caller.

```JavaScript
function getThing(){
  return {
    input: new Subject<number>(),
    output: inputSubject.pipe(
      delay(1000),
      map(x => `${x * 5}`)
    )
  };
}
```

It would be much nicer to have one of the following

```JavaScript
function getThing(){
  return new TransformSubject<number,string>().pipe(
    delay(1000),
    map(x => `${x * 5}`)
  );
}
```

or

```JavaScript
function getThing(){
  return new TransformSubject<number,string>(
    pipe(
      delay(1000),
      map(x => `${x * 5}`)
    )
  );
}
```

So that if you subscribe to the TransformSubject, you get results that have been transformed. If you emit into its `.next(v)`, you generate the next starting value.

----

## Part Way to a Solution

This is an attempt at the second approach. When somebody subscribes to the subject, they're actually returned the transformed results.

```JavaScript
class TransformSubject<T,R> extends Subject<T>{
  private _tranf: OperatorFunction<T,R>;

  constructor(fn: OperatorFunction<T,R>){
    super();
    this._tranf = fn;
  }

  public subscribeT(observer){
    return super.pipe(
      this._tranf,
      share()
    ).subscribe(observer);
  }
}
```

Here it is in use:

```JavaScript
const times5Subject = new TransformSubject<number, string>(
  pipe(
    delay(1000),
    map(x => `${x * 5}`)
  )
);

times5Subject.subscribeT(console.log);
times5Subject.next(10);
```

The issue is that I've implemented a new `subscribe` function called `subscribeT`. I can't override the original subscribe function because the transform part of the subject still relies on the old subscribe function. I'm not sure how to overcome that problem.

I'm not entirely sure what I'm doing is possible by extending `Subject`.

Here's a version that works, but it doesn't extend subject and I've not tested it extensively. I'm not sure how nicely this would play with the rest of RxJS. For the basic cases where you might use this sort of thing, this will work. If you're going to try composing this as though it were a fully fledged subject, I'm hesitant to beleive it would work. At least it must implement more of what a subject does. 


```JavaScript
class TransformSubject<T,R>{
  private _input: Subject<T>;
  private _output: Observable<R>;

  public next: (v:T) => void;
  public error: (err: any) => void;
  public complete: () => void;
  public pipe: (op) => Observable<any>;

  public subscribe: (subscriber) => Subscription
  public unsubscribe: () => void;

  public get closed(){
    return this._input.closed; 
  }

  constructor(fn: OperatorFunction<T,R>){

    this._input = new Subject<T>();
    this._output = this._input.pipe(
      fn, 
      share()
    );

    this.next = this._input.next.bind(this._input);
    this.error = this._input.error.bind(this._input);
    this.complete = this._input.complete.bind(this._input);

    this.subscribe = this._output.subscribe.bind(this._output);
    this.unsubscribe = () => {
      this._input.complete();
      this._input.unsubscribe();
    }

    this.pipe = this._output.pipe.bind(this._output);
  }
}
```
