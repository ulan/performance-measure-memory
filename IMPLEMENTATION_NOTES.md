# Implementation Notes

## Approach 1: process isolation of web pages
A fast implementation is possible if the browser puts a cross-origin isolated web page into a separate process.
Such implementation simply returns the sizes of the relevant heaps and lists origins present on each heap.
Note that the browser has to keep track of origins and workers of the web page, which slightly complicates the implementation.

## Approach 2: heap segregation by origin
An object that is allocated by JavaScript code can be attributed to the origin of [the running execution context](https://www.ecma-international.org/ecma-262/10.0/index.html#running-execution-context).
Segregating objects by origin during allocation allows fast implementation of `performance.measureMemory` with fine-grained per-origin breakdown.
This comes at the cost of a more complex allocator and heap organisation with a separate space/partition for each origin.
Note that origin attribution is not precise for objects shared between multiple origins.
Another subtlety is that the origin that keeps an object alive (i.e. retains the object) is not necessarily the same origin that allocated the object.
Moreover, both origins may differ from [the origin of the object's constructor](https://tc39.es/ecma262/#sec-getfunctionrealm) This is because origins of the same JavaScript agent can synchronously script with each other and can pass objects to each other.
The implementation can sidestep this problem by making the breakdown more coarse-grained and merging the memory usage of all origins of the same JavaScript agent.

## Approach 3: accounting during garbage collection
The following algorithm shows how to carry out per-origin memory measurement in the marking phase of a Mark-Sweep garbage collector.

Setup:

1. Assume that there is a partial function `InferRealm(object)` that uses implementation dependent heuristics to quickly compute the realm of the object or fails if that is not possible.
For example, the function could return [the realm of the object's constructor](https://tc39.es/ecma262/#sec-getfunctionrealm).
2. Let `realms` be the set of realms present on the heap at the start of garbage collection.
3. For each `realm` in `realms` create a marking worklist `worklist[realm]`.
4. Create a special marking worklist `worklist[unknown]` for shared/unattributed objects.
5. Iterate roots and push the discovered objects onto `worklist[unknown]`.

Marking worklist draining:
1. Pop an `object` from one of the non-empty worklists `worklist[realm]`, where `realm` can be also `unknown`.
2. If `InferRealm(object)` succeeds, then change `realm` to its result.
3. If `realm` is not `realms`, then it was created after the start of garbage collection and it does not have a worklist. In that case change `realm` to `unknown`.
4. Account the size of `object` to the origin of `realm`.
5. Iterate the reference in the object and push newly discovered to `worklist[realm]`.

The algorithm precisely attributes objects with known realms.
Additionally, objects that are not shared between multiple realms are accounted for precisely.
However, attribution of shared objects that do not have known realms is non-deterministic.
Shared strings and code objects are likely to be affected, so it might be worthwhile to add step 3a for such objects:

3.a. if `object` can be shared between multiple realms, then change `realm` to `unknown`.

The algorithm [was implemented](https://bugs.chromium.org/p/chromium/issues/detail?id=973627) in Chrome/V8 and it adds 10%-20% overhead to garbage collection.