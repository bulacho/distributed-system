# Lecture 2: Threads and RPC

## 1. Why go?
> - Good support for threads
> - convenient RPC
> - type- and memory- safe
> - garbage-collected (no use after freeing problems) threads + GC is particularly attractive!
> - not too complex
> - Go is often used in distributed systems

## 2. Thread
> - a useful structuring tool, but can be trickly
> - Go calls them goroutines; everyone else calls them threads

## 3. Thread = "thread of execution"
> - threads allow one program to do many things at once
> - each thread executes serially, just like a non-threaded program
> - the threads share memory
> - each thread includes some per-thread state:
>   - program counter, registers, stack

## 4. Why threads?
> - I/O concurrency
>   - Client sends request to many servers in parallel and waits for replies
>   - Server processes many simultaneous client requests.
>     - Eeach request may block.
>     - While waiting for the disk to read data for client X, processes a request from client Y.
> - Multicore performance
>   - Execute code in parallel on several cores
> - Convenience
>   - In backgrounde, once per second, check whether each worker is still alive

## 5. An alternative to threads
> - Write code that explicitly interleaves activities, in a single thread. Usually called "event-driven"
> - Keep a table of state about each activity, e.g each client request
> - One event loop that:
>   - Checks for neu input for each activity (e.g arrival of reply from server)
>   - Does the next step for each activity,
>   - updates state.
> - Event-driven can get you I/O concurrency,
>   - and eliminates thread costs (which can be substantial),
>   - but doesn't get multi-core speedup.
>   - and is painful to program

## 6. Thread challenges
> - Sharing data safely
>   - race condition = two threads use same memory at same time, one (or both) writes
>   - -> use locks(Go's sync.Mutex)
>   - -> or avoid sharing mutable data
> - Coordination between threads
>   - one thread is producing data, another thread is consuming it
>   - -> use Go channels or sync.Cond or sync.WaitGroup
> - Deadlock
>   - A cycle of threads waiting for each other
>   - via locks, or channels, or RPC

## 7. Web Crawler
> - Goal: fetch all web pages, e.g to feed to an indexer
> - Give it a starting web page
> - Web Crawler recursively follows all links
> - [diagram: pages, links, a DAG, a cycle]
> - Don't fetch a given page mor than once and don't get stuck in cycles.

## 8. Crawler challenges
> - Exploit I/O concurrency
>   - Network latency is more limiting than network capacity
>     - Internet latency: maybe 0.1 seconds, due to speed of light
>     - Internet throughput: maybe MB/sec or GB/sec
>   - Fetch many pages in parallel
>     - To increase URLs fetched per second
>     - => Use threads for concurrency
> - Fetch each URL only *once*
>   - Avoid wasting network bandwidth
>   - Avoid link cycles
>   - Be nice to remote servers
>   - => Need to remember which URLs visted
> - Know when finished

## 9. Solution
> - We'll look at three solutions
>   - 1. Serial
>   - 2. Concurrent, coordination via shared data
>   - 3. Concurrent, coordination via channels

## 10. Serial Crawler
> - Performs depth-first exploration via recursive Serial calls
> - The "fetched" map avoids repeats, breaks cycles a single map, passed by reference, caller sees callee's updates
> - Finished when all [recursive] links are explored
> - Fetches only one page at a time -- slow

## 11. ConcurrentMutex Crawler:
> - Creates a thread for each page fetch
>   - Many concurrent fetches, higher fetch rate
> - The "go func" creates a goroutine and starts it running func... is an "anonymus function" 
> - The threads share the fs.fetched map. So only one thread will fetch any given page
> - Why use Mutex?
>   - One reason:
>     - Two threads make simultaneous calls to ConcurrentMutex() with same URL. Due to two different pages containing link to same URL
>     - T1 read fetched[url], T2 reads fetched[url]. 
>     - Both see that url hasn't been fetched(fetch[url] = false)
>     - Both fetch, which is wrong
>     - The mutex causes one to wait while the other does both check and set. So only one thread sees fetched[url]==false
>     - We say "the lock protects fs.fetched[]". But note Go does not enforce any relationship between locks and data!
>     - The code between lock/unlock is often called a "critical section"
>   - Another reason:
>     - Internally, map is a complex data structure (tree? expandable hash?)
>     - Concurrent update/update may wreck internal invariants
>     - Concurrent update/read may crash the read

## 12. ConcurrentChannel Crawler
> - A Go chanel:
>   - A chanel is an object. ch := make(chan int)
>   - A channel lets one thread send an object to another thread. 
>   - ch <- x  (the sender waits until some goroutine receives)
>   - y := <- ch (a receiver waits until some goroutine sends)
>   - Also: for y := range ch
>   - Channels both communicate and sychronize
>   - Several threads can send and receive on a channel
>   - Send+Recv takes less than a microsecond -- fairly cheap
>   - Remember: sender blocks until the receiver receives! "synchronous" watch out for deadlock
> - ConcurrentChannel coordinator()
>   - coordinator() creates a worker goroutine to fetch each page
>   - worker() sends slice of page's URLs on a channel, mutilple workers send on the single channel
>   - coordinator() reads URL slices from the channel
> - Note: there is no recursion here; coordinator() creates all workers.
> - Note: no need to lock the fetched map, because it isn't shared!
> - How does the coordinator know it is done?
>   - Keeps count of workers in n.
>   - Each worker sends exactly one item on channel.
> - The channel does 2 things:
>   - 1. communication of values.
>   - 2. notification of events (e.g. thread termination)

## 13. Remote Procedure Call (RPC)
> - a key piece of distributed system machinery; all the labs use RPC
> - Goal: easy-to-program client/server communication
> - Hide details of network protocols
> - Convert data (strings, arrays, maps, ...) to "wire format"
> - portability/ interoperability

## 14. RPC message diagram
```
  Client               Server
   request--->
      <---response
```

## 15. Software structure
```
  Client app            handler fns
   stub fns              dispatcher
   RPC lib                RPC lib
     net  ----------------- net
```

## 16. Go example: kv.go
> - A toy key/value storage server -- Put(key,value), Get(key)->value
> - Uses Go's RPC library
> - Common:
>   - Declare Args and Reply struct for each server handler.
> - Client:
>   - connect()'s Dial() creates a TCP connection to the server
>   - get() and put() are client "stubs"
>   - Call() asks the RPC library to perform the call
>     - you specify connection, function name, arguments, place to put reply
>     - library marshalls args, sends request, waits, unmarshalls reply
>     - return value from Call() indicates whether it got a reply
>     - usually you'll also have a reply.Err indicating service-level failure
> - Server:
>   - Go requires server to declare an object with methods as RPC handlers
>   - Server then registers that object with the RPC library
>   - Server accepts TCP connections, gives them to RPC library
>   - The RPC library
>     - reads each request
>     - creates a new goroutine for this request
>     - unmarshalls request
>     - looks up the named object (in table create by Register())
>     - calls the object's named method (dispatch)
>     - marshalls reply
>     - writes reply on TCP connection
>   - The server's Get() and Put() handlers
>     - Must lock, since RPC library creates a new goroutine for each request
>     - read args; modify reply

> - A few details:
>   - Binding: how does client know what server computer to talk to?
>     - For Go's RPC, server name/port is an argument to Dial
>     - Big systems have some kind of name or configuration server
>   - Marshalling: format data into packets
>     - Go's RPC library can pass strings, arrays, objects, maps, &c
>     - Go passes pointers by copying the pointed-to data
>     - Cannot pass channels or functions
>     - Marshals only exported fields (i.e., fields w/ CAPITAL letter) 