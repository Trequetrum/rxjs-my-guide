# RxJS-MyGuide
What I've learned about RxJS so far

## Preamble

There are many RxJS articles and guides out there. This one is being written as a bit of a fresh perspective. It isn’t comprehensive and it doesn’t guide a user from knowing nothing. It just hits topics that interested me and which took a lot of self-discovery to fully understand.

### Table of Contents:
- [Introduction](Intro.md) (WIP)
- [Kinda Like Arrays](Silaa.md)
- [Creating Observables](Creating.md) (WIP)
- [Understanding concatMap, mergeMap, & switchMap](Ucms.md)
- [Never Nest Subscriptions](Nns.md) (WIP)
- [Memory Management](MemoryManagement.md)
- Patterns:
  - [Consecutive HTTP Calls](Chttpcp.md)
- Custom Observables
  - [Stopwatch](Stopwatch.md)
  - [TransformSubject](transformSubject.md) (WIP)
- Custom Static Operators
  - [cascadeMerge](cascadeMerge.md)
  - [partitionOn](partitionOn.md)
  - [prefer](prefer.md)
- Custom Pipeable Operators
  - [log](tapLog.md)
  - [arrayMap](arrayMap.md)
  - [sourceResult Merge](resultMap.md)s
  - [intervalArray](intervalArray.md)
  - [filterFirst](filterFirst.md)
  - [initialize](initialize.md)
  - [catchEmpty](catchEmpty.md)
  - [startWithStream](startWithStream.md)
  - [~startWithDefer~](startWithDefer.md)
  - [delayWhile](delayWhile.md)
  - [bufferedExhaustMap](bufferedExhaustMap.md) (v2)
  - [priorityDebounceTime](priorityDebounceTime.md)
  - [predicate/priorityThrottleTime](priorityThrottleTime.md)
  - [ngZone run/runOutside](ngZoneOperators.md)
- Mini-Projects
  - [Concurrency Limiter for Disparate Streams](Clds.md)
  - [Managing Observable Subscriptions with Synchronous Streams](Mosss.md)

### Glossary:
  - **WIP**: Work in Progress
  - **V{n}**: There are n relatively distinct/interesting versions
