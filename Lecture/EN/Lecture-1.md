# Lecture 1: Introduction

## 1. Distributed system

A group of computers cooperating to provide a service

Examples:

> - we all use distributed systems popular apps backends. e.g for messaging big web sites, DNS, phone system
> - often built from services that are internally distributed databases, transaction systems, "big data" processing frameworks, authentication services
> - sometimes distribution itself is the service cloud systems like Amazon AWS

In repo is mostly about infrastructure services. e.g storage for big web sites, MapReduce, peer-to-peer sharing

#### It's not easy to build systems this way:
> - Concurrency
> - Complex interactions
> - Hard to get high performance
> - Partial failure

#### Why do people build distribured systems?
> - To increse capacity via parallel processing
> - To tolerate faults via replication
> - To match distribution of physical devices e.g. sensors
> - To increase security via isolation

#### Labs:
> - Goal: use and implement some important techniques, experience with distributed programming
> - 5 lab:
>> - Lab 1: Distributed big-data framework (like MapReduce)
>> - Lab 2: Client/server vs unreliable network
>> - Lab 3: Fault tolerance using replication (Raft)
>> - Lab 4: A fault-tolerant database
>> - Lab 5: Scalable database performance via sharding

## 2. Main Topics

This is a repo about infrastructure for applications.
> - Storage.
> - Communication.
> - Computation.

A big goal: hide the complexity of distribution from applications.

#### Topic: Fault tolerance
> - 1000s of servers, big network -> always something broken. We'd like to hide these failures from the application.
> - "High availbility": service continues despite failures
> - Big idea: replicated servers. If one server crashes, can proceed using the other(s).

#### Topic: Consistency
> - General-purpose infrastructure needs well-defined behavior. E.g. "read(x) yields the value from the most recent write(x)."
> - Achieving good behavior is hard! E.g. "replica" servers are hard to keep identical.

#### Topic: Performance
> - The goal: scalable throughput. Nx server -> Nx total throughput via parallel CPU, RAM, disk, net.
> - Scaling gets harder as N grows:
>> - Load imbalance
>> - Slowest-of-N latency.
>> - Some things don't speed up with N: initialization, interaction.

#### Topic: Tradeoffs
> - Fault-tolerance, consistency, and performance are enemies
> - Fault-tolerance and consistency require communication
>> - E.g. Send data to backup server
>> - E.g. Check if cached data is up-to-date communication is often slow and non-scalable
> - Many designs sacrifice consistency to gain speed.
>> - E.g. read(x) might *not* yield the latest write(x)! Painful for application programmers (or users).
> - We'll see many design points in the consistency/performance spectrum.

#### Topic: Implementation
> - RPC, threads, concurrency control, configuration. 

## 3. Case Study: MapReduce

#### MapReduce overview
> - Context: Multi-hour computations on multi-terabyte data-sets. E.g. build search index, or sort, or analyze structure of web only practical with 1000s of computers applications not written by distribured systems experts
> - Big goal: easy for non-specialist programmers, programmer just defines Map and Reduce functions often fairly simple sequential code
> - Mr manages, and hides, all aspects of distributions!

#### Abstract view of a MapReduce jon -- word count
![alt text](image.png)

1. Input is (already) split into M pieces
2. MR calls Map() for each input split, produces list of k,v pairs "intermediate" data each Map() call is a "task"
3. When Maps are done, MR gathers all intermediate v's for each k, and passes each key + values to Reduce call
4. Final output is set of <k,v> pairs from Reduce()s

#### Word-count code
> - Map(d)
>>>>> chop d into words

>>>>> for each word w

>>>>>>>> emit(w, "1")

> - Reduce(k, v[])
>>>>> emit(len(v[]))

#### MapReduce scales well:
> - N "worker" computers (might) get you Nx throughput. Map()s can run in parallel, since they don't interact. Same for Reduce()s.
> - Thus more computers -> more throughput

#### MapReduce hides many details:
> - Sending map+reduce function code to servers
> - Tracking which tasks have finished "shuffling" intermediate data from Maps to Reduces balacing load over servers, recovering from crashed servers

#### To get these benefits, MapReduce imposes rules:
> - No interaction or state (other than via intermediate output).
> - Just the one Map/Reduce pattern for data flow.
> - No real-time or streaming processing.


## REFERENCES
### https://pdos.csail.mit.edu/6.824/notes/l01.txt