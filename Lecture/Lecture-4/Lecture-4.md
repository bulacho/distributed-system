# Lecture 4: Consistency, Linearizability
Today's topic: consistency models, specifically linearizability

We need to be able to reason about correct behavior for network services. -> consistency model

## Consistency model
> - A specification for the relationship of different client's views of a service
> - Focus on key/value storage with network clients
>   - put(k, v) -> done
>   - get(k) -> v
> - In ordinary programming, there's nothing to talk about a read yields the last value that was written

## Correct behavior
> - Concurrent reads/writes
> - Replicas
> - Caches
> - Failure, recovery
> - Lost message
> - Retransmission

## Why does a storage system need a formal consistency model?
> - For applications, hard to be correct w/o guarantees from storage
>   - e.g. producer computes, then executes
```
          put("result", 27)
          put("done", true)
```
>   - consumer executes
```
          while get("done") != true
            pause
          v = get(result)
```
> - for services, hard to design/implement/optimize w/o a specification
>   - e.g. is it OK for clients to read from GFS replicas (rather than primary)?

## Lots of consistency model
> - Sometimes driven by desire to simplify application programmer's lives and sometimes codifying behavior that was convenient for implementors also, lots of overlapping definitions from different fields
>   - e.g. FS, databases, CPU memory

## Linearizability
> - It's a specification -- a requirement for how a service must behave
> - It's usually what people mean by "strong consistency".
>   - Matches programmer intuitions reasonably well.
>   - But riles out many optimization

## Starting point
> - We assume that there's a serial spec for what individual operations do serial = a single server executing operations one at a time
```
  db[]
  put(k, v):
    db[k] = v
    return true
  get(k):
    return db[k]
```

## What about concurrent client operations?
> - A client sends a request;
>   - Takes some time crossing network;
>   - Server computes, talks to replicas,...
>   - Reply moves through network;
>   - Client receives reply
> - Other clients may send/receive/ be waiting during that time!
> - We need a way to describe concurrent scenarios, so we can talk about which are/aren't valid

## Definition: A history
> - Describes a time-line of possibly-concurrent operations
> - Each operation has invocation and response times (RPC) as well as argument and return values
> - Example:
```
  C1: |-Wx1-| |-Wx2-|
  C2:   |---Rx2---|
``` 
> - the x-axis is real time 
>   - indicates the time at which client sent request
>   - indicates the time at which the client received the reply
> - "Wx1" means "write value 1 to record x" -- put(x, 1)
> - "Rx1" means "a read of record x yielded value 1" -- get(x) -> 1
> - C1 sent put(x, 1), got reply, sent put(x, 2), got reply
> - C2 get(x), got reply=2
> - note that writes have responses; no return value other than "done"

## A history
> - is usually a trace of what clients saw in an actual execution
> - used to check correct behavior
> - also used by designers in "would this be OK" though experiments

## definition: a history is linearizable if
> - you can find a point in time for each operation between its invocation and response, and
> - the history's result values are the same as serial execution in that point order.

## Linearizable systems are not limited to just read and write operations:
> - delete
> - increment
> - append
> - test-and-test (to implement locks) any operation on server state

## Application programmers like linearizability -- it's relatively easy to use:
> - reads see fresh data -- not stale
> - all clients see the same data (when there aren't writes)
> - all clients see data changes in the same order

## What about other consistency models?
> - can they allow better performance?
> - do they have intuitive semantics?
> - example: eventual consistency -- a weak model
>   - multiple copies of the data (e.g. in different datacenters, for speed)
>   - a read consults any one replica (e.g. closest)
>   - a write updates any one replica (e.g. closest) client gets response when that one update is done
>   - replicas synchronize updates in the background eventually, other replicas will see my update

## Eventual consistency is pretty popular
> - faster than linearizability especially if replicas are in different cities for fault-tolerance
> - and more available -- any one replica will do no waiting for primary/backup communication