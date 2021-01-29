# The Consecutive HTTP Call Pattern

Here I've created a hugely simplified HTTP call pattern. Each call requires the result from the previous call and each call's result is important to the final result. `fakeHTTPCall` returns a number after a second. 

This pattern can be adapted to arbitrarily complex calls, but the take-away is the `call.pipe(map(new => ({...old, new})))` pipeline. This stops you from nesting mergeMap/switchMap in order to keep earlier values via closure. 

Here's a fake httpCall:

```JavaScript
function fakeHttpCall(num = 0): Observable<number> {
  const mapN = (num) => {
    switch(num){
      case 0:
        return 981;
      case 981: 
        return 11;
      case 11:
        return 8080;
      default:
        return 0;
    }
  }
  return of(mapN(num)).pipe(
    delay(1000)
  );
}
```

Here's how you might call this 5 times while using functional closure to keep each value until the end:

```JavaScript
fakeHttpCall().pipe(
  switchMap(firstCall => fakeHttpCall(firstCall).pipe(
    switchMap(secondCall => fakeHttpCall(secondCall).pipe(
      switchMap(thirdCall => fakeHttpCall(thirdCall).pipe(
        switchMap(fourthCall => fakeHttpCall(fourthCall).pipe(
          map(fifthCall => ({
            firstCall,
            secondCall,
            thirdCall,
            fourthCall,
            fifthCall
          }))
        ))
      ))
    ))
  ))
).subscribe(console.log);
```

The output is this object:

```JSON
{
  "firstCall": 981,
  "secondCall": 11,
  "thirdCall": 8080,
  "fourthCall": 0,
  "fifthCall": 981
}
```

While the output is nice, you can see how nested `switchMap`/`mergeMap`/`ect` becomes harder to follow. Here, each call is simple and how each response is used is simple, but imagine this filled out with complex buisness/transformation logic and you can imagine the headache this sort of call structure creates.

Deeply nested functions are notoriously difficult to debug in JavaScript so the extra effort of mapping into intermediate objects to hold the values you need in the next step (rather than nesting and getting intermediate values via functional closure) is well worth the effort.

Here's the pattern at work:

```JavaScript
fakeHttpCall().pipe(
  map(res => ({firstCall: res})),
  switchMap(oby => fakeHttpCall(oby.firstCall).pipe(
    map(res => ({...oby, secondCall: res}))
  )),
  switchMap(oby => fakeHttpCall(oby.secondCall).pipe(
    map(res => ({...oby, thirdCall: res}))
  )),
  switchMap(oby => fakeHttpCall(oby.thirdCall).pipe(
    map(res => ({...oby, fourthCall: res}))
  )),
  switchMap(oby => fakeHttpCall(oby.fourthCall).pipe(
    map(res => ({...oby, fifthCall: res}))
  ))
).subscribe(console.log);
```

The output is the exact same as before:

```JSON
{
  "firstCall": 981,
  "secondCall": 11,
  "thirdCall": 8080,
  "fourthCall": 0,
  "fifthCall": 981
}
```

As you can see, with intermediate objects, we don't require the call stack or functional closures to hold old values. We carry forward only what we need. It's also marginally faster as the runtime isn't required to travel up the call stack looking for variables. Really though, you should do it because it's cleaner, maintainable, and extendable and **not** in order to optimize early.

----

### Using resultSwitchMap

Here is a [link to resultSwitchMap](resultSwitchMap)'s implementation. 
This is the exact same pattern neatened up a little bit:

```JavaScript
fakeHttpCall().pipe(
  map(res => ({firstCall: res})),
  resultSwitchMap(oby => fakeHttpCall(oby.firstCall), "secondCall"),
  resultSwitchMap(oby => fakeHttpCall(oby.secondCall), "thirdCall"),
  resultSwitchMap(oby => fakeHttpCall(oby.thirdCall), "fourthCall"),
  resultSwitchMap(oby => fakeHttpCall(oby.fourthCall), "fifthCall")
).subscribe(console.log);
```

The output is the exact same as before:

```JSON
{
  "firstCall": 981,
  "secondCall": 11,
  "thirdCall": 8080,
  "fourthCall": 0,
  "fifthCall": 981
}
```
### Using sourcePayloadMap

This is a more general/flexible operator than resultSwitchMap as it doesn't require the input to have a specific shape. The two are largly interchangable as shown by the mapping operator after each call re-aligning the output with the calls above.

```JavaScript
fakeHttpCall().pipe(
  map(v => ({firstCall: v})),
  sourcePayloadMap(src => fakeHttpCall(src.firstCall)),
  map(({source, payload}) => ({...source, secondCall: payload})),
  sourcePayloadMap(src => fakeHttpCall(src.secondCall)),
  map(({source, payload}) => ({...source, thirdCall: payload})),
  sourcePayloadMap(src => fakeHttpCall(src.thirdCall)),
  map(({source, payload}) => ({...source, fourthCall: payload})),
  sourcePayloadMap(src => fakeHttpCall(src.fourthCall)),
  map(({source, payload}) => ({...source, fifthCall: payload})),
).subscribe(console.log);
```

The output is the exact same as before:

```JSON
{
  "firstCall": 981,
  "secondCall": 11,
  "thirdCall": 8080,
  "fourthCall": 0,
  "fifthCall": 981
}
```

### More Than One Way to Skin a Cat

This has identicle output and is cleaner for situations (like this one) that lend itself to recursive calls. If it's five unique calls with different results, this appraoch gets bloated. 

The final map is to align with the output above, but it's pretty unessesary. 

```JavaScript
fakeHttpCall().pipe(
  expand(x => fakeHttpCall(x)),
  map((x,i) => ({
    call: i+1,
    resp: x
  })),
  take(5)
).subscribe(console.log);
```
The output is 5 objects one after another:

```JavaScript
{call: 1, resp: 981}
{call: 2, resp: 11}
{call: 3, resp: 8080}
{call: 4, resp: 0}
{call: 5, resp: 981}
```
