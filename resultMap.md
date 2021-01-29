# Operators that further map the source and result of an observable together.

resultConcatMap, resultMergeMap, resultSwitchMap, & sourcePayloadMap

## resultConcatMap, resultMergeMap, & resultSwitchMap

Take an JS Object as input, and map the result of the projected Observable into the JS Object as a new property using the given `key`.

[The Consecutive HTTP Call Pattern](Chttpcp.md) provides a good example of this in use.

----

A Helper function

```JavaScript
function resultMap<Object>(
  type: "concat" | "merge" | "switch",
  project: (v: Object) => Observable<any>, 
  key = "result"
): MonoTypeOperatorFunction<Object>{
  const operator = 
    type === "concat" ?
    concatMap :
    type === "merge" ?
    mergeMap :
    switchMap;
    
  return pipe(
    operator(spread => project(spread).pipe(
      map((res: any) => ({...spread, [key]: res}))
    ))
  );
}
```

The three operators:

```JavaScript
function resultConcatMap<Object>(
  project: (v: Object) => Observable<any> , 
  key = "result"
): MonoTypeOperatorFunction<Object>{
  return resultMap("concat", project, key);
}

function resultMergeMap<Object>(
  project: (v: Object) => Observable<any> , 
  key = "result"
): MonoTypeOperatorFunction<Object>{
  return resultMap("merge", project, key);
}

function resultSwitchMap<Object>(
  project: (v: Object) => Observable<any> , 
  key = "result"
): MonoTypeOperatorFunction<Object>{
  return resultMap("switch", project, key);
}
```

## sourcePayloadMap

This is a more general/flexible operator than resultSwitchMap as it doesn't require the input to have a specific shape. If you map the input/output to meet what each operator expects, the two are interchangable.

[The Consecutive HTTP Call Pattern](Chttpcp.md) provides a good example of this in use.

```JavaScript
function sourcePayloadMap<T, R>(
  project: (v:T) => Observable<R>
): OperatorFunction<T, {source: T, payload: R}> {
  return pipe(
    mergeMap(source => project(source).pipe(
      map(payload => ({source, payload}))
    ))
  )
}
```
