# The arrayMap Operator

map(array => array.map(...)) is an extremely common pattern in RxJS, this operator abtracts that away, letting the user directly map the values in emitted arrays.
This operator will function as a simple map for emitted values that arn't arrays.

```JavaScript
function arrayMap<T, R>(fn: (val: T) => R): OperatorFunction<T|T[], R|R[]> {
  return s => s.pipe(
    map(x => Array.isArray(x)? x.map(fn) : fn(x))
  );
}
```

# The mapArrayMap Operator

This is the same as above, only it will only accept arrays as emitted values

```JavaScript
function mapArrayMap<T, R>(fn: (val: T) => R): OperatorFunction<T[], R[]> {
  return s => s.pipe(
    map(x => x.map(fn))
  );
}
```
