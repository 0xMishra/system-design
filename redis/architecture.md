
Redis is an in memory key value store designed for speed and and low latency. It is designed to keep everything on RAM rather than on disk. Let's understand the architecture of Redis.



### **Why do we need** Redis?
Redis is primarily used as an in memory cache, session storage, rate limiter, message queue, pub sub etc.. Redis is used In many popular applications like Twitter, Github, stack overflow etc. 



### Flow of request 
[![flow of request](https://app.eraser.io/workspace/JHNdOtxPZe1ZBBIaKjsy/preview?figure=9_8-qYGgvoxwgfVRxT1lr&type=embed)](https://app.eraser.io/workspace/JHNdOtxPZe1ZBBIaKjsy?figure=9_8-qYGgvoxwgfVRxT1lr)

1. The client makes a request to the server.
2. Server checks Redis cache for the data requested. If data exists in the cache the server retrieves it from the cache and returns it to the client.
3. Otherwise if the data doesn't exist in the cache the server queries it from the DB and stores it in the cache.
4. Finally the requested data is returned to the client. 


### Capacity estimation
Let's dive into the capacity estimation of our cache



**_Throughput:_**

Redis can handle 100k to 500k operations/second per instance. But it depends on:

1. Complexity of commands executed.
2. Hardware specifications.
3. Network speed and strength.


**_Latency:_** 

Less than 1ms (<1ms)



**_Memory:_**

Since redis stores data in the RAM so it depends on the RAM itself. Eg. if 64 gb RAM the ~60gb database.



If the redis node is 64gb and the dataset grows up to say 70gb then redis needs an **eviction policy.**  Eviction policies are used to free the instance in case of data overload. There are different types of eviction policies.

1. **LRU** ( least recently used): It evicts the least recently used key in redis meaning whatever hasn't been used in the longest time. Used in session storage.
2. **LFU** ( least frequently used): It evicts the least frequently used key in redis meaning whatever has been accessed the least amount of time. For example DNS resolver caches. 




### Core components


**RESP**

Redis Serialization Protocol is a wire format Redis uses to facilitate communication between the client and the server.



It is intentionally made to be an extremely simple format. It is a plain ASCII text, easy to parse and human readable. Every command that client sends and every response from redis travels as a RESP byte over TCP socket.



The protocol encodes data using a type prefix character on each line:

**(+)** means a simple string (+OK) 

**(-)** means an error

**( :)** means an integer

**($)** means a bulk string with a declared byte length

**(*)** means an array. Every Redis command is just an array of bulk strings

**(\r\n)** is an end of line marker between the parts of message. 



Let's take an example for this command: `SET USER:1 ana` 

this command will be encoded as: *3 -> 3 elements

$3\r\nSET\r\n  -> SET

$6\r\nUSER:1\r\n -> USER:1

$3\r\nana\r\n -> Ana



Final encoded string would be: `*3\r\n$3\r\nSET\r\n$6\r\nUSER:1\r\n$3\r\nana\r\n` 



**Event Loop**

Redis is single threaded for command executions. But then how can a single threaded redis handle thousands of client connections?



Let us take an example say we have 1000 clients wants to connect to redis.

client 1, client 2.............client 1000

Now opening a new thread for each of these clients would be terrible for performance because threads are costly as they require a lot of processes to run smoothly such as CPU scheduling, thread stack, context switching, synchronization etc.



If one  thread stack costs 1 MB of ram then 1000 stacks would be  ~1 GB of ram just for thread stacks. 

So you it's not optimal at all to use a new thread for each client and that is why it is single threaded.



Redis uses sockets to connect to each client. But instead of continuously checking each socket for a connection itself what redis does it uses Kernel api called `epoll_wait()`  and goes to sleep when data arrives on any client connection then kernel wakes redis up and returns the list of sockets that are ready.



`epoll_wait()`  api tells the kernel you are watching the sockets let me know when something interesting happens.



Example:

client A -> no data

client B -> no data

client C -> get user:1



`epoll_wait()`  returns :

socket C is ready



Event loop is a high level reactor pattern implementation that uses epoll to detect ready sockets and then processes commands.

[![Event loop](https://app.eraser.io/workspace/JHNdOtxPZe1ZBBIaKjsy/preview?figure=6yPGPdyq3KZFAksdCAceP&type=embed)](https://app.eraser.io/workspace/JHNdOtxPZe1ZBBIaKjsy?figure=6yPGPdyq3KZFAksdCAceP)

 



**In-memory storage engine**

It may appear as if redis store data in a Hashmap:`unordered_map<string,string>` 



It doesn't use hash map, redis engine is optimized for fast lookups and low memory usage.

Redis doesn't have an internal representation per data type. It uses different encodings and switches between them based on the size of the data. This is an abstraction that is not visible to the client but redis saves a lot of memory doing this.



```
// simplified redis object structure 
struct redisObject {
  unsigned type,
  unsigned encoding,
  unsigned lru,
  int refCount,
  void *ptr
} robj;
```
type -> original data type

encoding -> formate redis uses to store that data

Lru -> eviction policy

refCount -> use for memory management

*ptr -> pointer to actual value to be stored

Every single data type whether it is a string, a list or set is wrapped in this struct.



Let's take an example of integer value 30. The struct would look something like this:

```
30 -> robj obj{
  type = OBJ_string,
  encoding = OBJ_ENC_INT,
  ptr = (void*)30 // directly stored here
};
```


Let's take example of string "hello". If the length of this string <44 bytes it would look something like:

```
"hello" -> robj obj{
  type = OBJ_string,
  encoding = EMBSTR,
  ptr = &("hello") // directly stored here
};
```


For a longer string let's say "this is a really long string......." we have to do double allocation, one for the object and one for the string:

```
"this is a really long string......." -> robj obj{
  type = OBJ_string,
  encoding = RAW,
  ptr = rmalloc(sizeof(SDS) + strlen(long string)) // SDS- simple dynamic string
};
```


Let's take a look at a command: `SET USER:1 name ana age 20` 

This will be stored as a **Listpack** something like this: `USER:1 -> [4|"name"|3|"ana"|3|"age"|2|20]` 



**Listpack -> hashTable conversion conditions:**

- The listpack will be converted into hashset in 2 conditions
- If entries exceed 128 fields, for example user adds 129th key say relative:"brother".
- if one value exceeds 64Bytes. for example user's address becomes more than 64 bytes.
- Both conditions can independently trigger the update.
These thresholds are configurable but once the thresholds is updated you cannot go back and reduce it. Once your hash grows beyond the threshold redis converts it into a proper hash table with O(1) lookup



For sorted sets it works the same way:

- small sorted sets: Listpack encoding O(N) rank queries but tiny memory
- large sorted sets: skiplist plus a hash table with O(1) lookup by member


### Persistence
Redis lives in RAM and RAM is volatile. If power goes out, OS kills the process or the process crashes for some reason all your data is gone. That is why we have persistence in redis. 



Persistence means "how much of the data we can recover". It depends on what kind of persistence you use. For example RDB or AOF are two popular persistence strategies.



**RDB - point in time snapshots**

When RDB fires , Redis calls `fork()`  which creates a child process which is an exact copy of the parents memory space this child walks every key in the keypspace and serializes it and dumps it into a file called `dump.rdb`. 

The parent keeps serving requests as if nothing happened. The reason this works without blocking the parent is copy-on-write (COW).



When the parent make changes to the memory page after the fork the OS creates a copy of that page for the parent and child still uses the original page. So the child is kind of frozen in time at the moment of fork and gets consistent snapshot without any locking.



However the issue is that if the parent is writing heavily during the snapshot the COW copies pile up and memory usage can spike up.

Other issue is you can only recover up to the last snapshot but if you snapshot every 5 minutes and the server crasher 4 minutes in then you last 4 minutes of writes.



Hence, this is not effective when the workload is write intensive.





**AOF - Append Only File**

Instead of saving point in time snapshots like RDB redis logs every write operation it executes

These commands are appended to a file( appendonly.aof) in the same format as redis protocol commands.

On restart redis replays this file from top to bottom and reconstructs the full dataset.



The key config here is `fsync` (Write) policies where the actual durability trade-offs lives.

- fsync always: on every single write redis calls fsync and forces the OS the to flush the file to the disk. This has zero data loss but fsync is a syscall which blocks until the write is confirmed so throughput drops sharply.
- fsync everysec: fsync once per second. It balances durability and performance. Max 1 sec data loss.
- fsync no: let the OS decide when to flush ( fastest but can cause seconds of data loss).


Redis 7 supports a mix of RDB + AOF format. The AOF file starts with an RDB snapshot (fast to load) then appends only the delta commands since the snapshot.



### High level architecture
[![HLD](https://app.eraser.io/workspace/JHNdOtxPZe1ZBBIaKjsy/preview?figure=QCXKYaM7Vr6LOZKvatmfQ&type=embed)](https://app.eraser.io/workspace/JHNdOtxPZe1ZBBIaKjsy?figure=QCXKYaM7Vr6LOZKvatmfQ)



