# The Consecutive HTTP Call Pattern

Here I've created a hugely simplified HTTP call pattern. Each call requires the result from the previous call and each call's result is important to the final result. `fakeHTTPCall` returns a number after a second. 

This pattern can be adapted to arbitrarily complex calls, but the take-away is the `call.pipe(map(new => ({...old, new})))` pipeline. This stops you from nesting switchMap in order to keep earlier values via closure. 

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

The output is this object:

```JSON
{
  firstCall: 981,
  secondCall: 11,
  thirdCall: 8080,
  fourthCall: 0,
  fifthCall: 981
}
```

----

### More Than One Way to Skin a Cat

This has identicle output and is cleaner for situations (like this one) that lend itself to recursive calls. If it's five unique calls with different results, this appraoch gets bloated. 

The final map is to align with the output above, but it's pretty unessesary. 

```JavaScript
fakeHttpCall().pipe(
  expand(x => fakeHttpCall(x)),
  take(5),
  toArray(),
  map(arr => ({
    firstCall: arr[0],
    SecondCall: arr[1],
    thirdCall: arr[2],
    fourthCall: arr[3],
    fifthCall: arr[4]
  }))
).subscribe(console.log);
```
