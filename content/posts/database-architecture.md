---
title: "Architecture of a DBMS (Part 1)"
date: 2022-10-18
---

# Architecture of a Database Management System

The objective of this series of posts is to present a deep dive on database internals, by the end
of this post you'll have a holistic view of what a DBMS looks like under the hood.

In this first post we will discuss **storage management** and how we map data present on disk (block
device) in memory. Because of how lengthy the post might end up I will leave references at the end
for readers who would like a deeper dive into the subject.

This part on storage management won't deal with indexing mechanisms and external memory data structures
that will be left to a following post.

## Managing Files

All major OLTP databases store data on-disk, when we refer to disk storage we will often use the
term *block device*. Modern operating systems often abstract away the hardware disk (HDD or SSD)
using a file system. File systems are central to how the OS deals with the hardware.

For example a file can be seen as a sequence of bytes, all file metadata (name, size, status) is
handled separately by the file system. In Unix systems you can interact with files using syscalls
such as `open`, `read` or `write`. How the file is opened and when the data is written belongs
to the realm of the kernel. Most programming languages abstract the syscalls around their standard
libraries. In C++ for example you will often be dealing with `std::fstream` in Go there's `os.File`.

Most database systems implement some sort of storage manager that abstracts away the underlying
resource handles `os.File` or `std::fstream` to provide a thread-safe API or to implement their
own buffered IO.

You can see PostgreSQL implementation [here](https://github.com/postgres/postgres/tree/master/src/backend/storage/smgr).

Implementing a storage manager is not that difficult, the idea is to keep a hashmap that maps files
by their names to the underlying resource handle.

```go

type StorageManager interface {
  Open(filename string) *os.File
  Write(filename string, data []byte) error
  Read(filename string) ([]byte, error)
  Close(filename string)
}

type DiskManager struct {
  sync.Mutex
  openFiles map[string]*os.File
}

```

The reason you'd like to support multiple files is to have a split physical location for each
database you'd might manage. Some storage engines such as InnoDB might have multiple files one
for each table, others will often do a sort of de-normalization where multiple tabls that have
logical references between each will exist in the same file[^1].

The storage manager is at the lowest level of the architecture, it's the last frontier between
the database process and the actual data on disk.

Because of how costly I/O operations are modifications to the data don't actually happen at the file
level. Each file read or write will take between a few microseconds up to milliseconds, therefore
if every database operation accessed the file the latency cost will be so huge requests might end up
needing seconds to finish. So to avoid this we tend to do something similar to what operating
systems do.

## Paging and I/O

Computers are very fast, at least at the CPU level, the major bottleneck for processing data is
actually moving data between external storage and the CPU.

One way to solve this is to assume spatial locality and fetch data in chuncks (64 bytes for a single
cache line, 4KB for RAM, 4KB~16KB for disk)

The numbers will often depend on the device and operating system but the main point is everytime
your program manipulates bytes that don't live in registers, you are manipulating multiple bytes at
the time.

For example if you ask the OS to write a 100 bytes buffer, the buffer will often write a 4KB blob
with the remaining bytes being null. Same goes for reading an array from RAM the first fetch will
load up to 64 bytes elements (4 uint32 for example) at the time.

So to optimize I/O access for a database the same concept of "buffering" or "paging" is implemented
this abstraction is called a **Block** and you can think of a block as a pointer to a fixed size
chunck of the underlying file.

```cpp
struct block_id_t {
  char[] filename;
  size_t offset;
};

```

Essentially every disk operation will be done on blocks and not on files, when we read from a file
we will often read a fixed size chunck at the given offset or write a fixed size chunck. Goes both
ways.

Now to map the data in the files in memory we will use a `Page` , pages essentially map the content
of a block in memory.

```cpp
typedef char[4096] Page;

```

Every modification to the data will now occur at the page level, if you mutate a row or insert a new
row the modification will happen to the paged row before being flushed to disk[^2].

PostgreSQL implements paging as well [see
bufpage.c](https://github.com/postgres/postgres/tree/master/src/backend/storage/page)

Pages won't usually store records end to end, more often a page will implement a `Header` that
contains metadata about the page contents and a checksum that ensures some sort of data integrity.

While pages exist as in-memory buffers, the content of a page are said to be *pinned* to a block
the pinning mechanism is handled by a higher level data structure, but as long as a page is *pinned*
to a given block all modifications on the page will eventually be reflected in the underlying file
where the block points to.

We mentionned earlier that paging is also done by the operating system, while this mapping is
different than what we just discussed some databases use the operating system pages as an
alternative to implementing their own. This mechanism is exposed at the OS level by the `mmap`
syscall, `mmap` maps the data on disk to the caller process address space this means the
responsibility for mapping the changes to the disk aren't handled by the database by the operating
system.

Using `mmap` is not recommended usually it can often conflict with transaction safety and cause
stalling i.e the database doesn't know the ground truth about the data on disk.
There are ways to avoid such problems such as the `mlock` syscall which effectively locks the page
in-memory not allowing the OS to do the flushing for more information on the subject you can refer
to the benchmark study by Crotty et al[^3].

# Buffer Management

At this point we've been looking at the DBMS architecture from the bottom-up, we went from file
organization to mapping file contents in memory. But how do we organize access to pages in a thread
safe manner ? And how do we keep memory from blowing up, since memory is finite ?

To answer these questions we will need a new abstract our new data structure will require two
things :

1. Allow concurrent access to pages.
2. Limit memory usage to avoid OOM and other shenanigans.

The way to provide concurrent access to pages in a shared data model will often come down to
choosing some sort of synchronization primitive (Spinlock, Mutex, Latch...) limiting memory
usage can be done by using [pooling](https://en.wikipedia.org/wiki/Pool_(computer_science)).

Pooling is a way to allow re-use of existing heap allocated objects, you can think of a pool
as a box, each time you want to use page you ask the pool for one once you're finished you
put the page back into the pool.

```go
type BufferManager struct {
  sync.Mutex
  bufferPool []Page
  size uint32
}

```

Now whenever a client requires access to a page, the buffer manager will pin the page to the choosen
block and return the page. Let's say our buffer manager has a pool of 16 pages and all pages are
pinned by clients. How do we respond to this request ? Well first we need to unpin one of the
existing pages in the pool, but how do we decide which page to unpin ?

There are several ways to decide, some popular ways include FIFO or LRU, the FIFO strategy serves
pages as first in first out meaning the first page that was pinned is the first one to be unpinned,
the LRU strategy on the other hand unpins the least recently used page.

# Conclusion and Coming Up Next

To recap :

* DBMS manages data on disk using files.
* Files are accessed in fixed size chuncks called blocks.
* DBMS map each block to a block identifier which serves as a pointer to the block on file.
* Blocks are mapped in memory to fixed size slices called Pages.
* Page access is managed by the Buffer Manager which provides thread safe access to pages and limits
memory usage.

![architecture-of-dbms](/architecture-of-dbms-1.png)

Let's say two clients request access to the same page, the page is pinned to a block and both
clients mutate the page. If both clients mutate the same record which mutation do we choose ?
What if we had a power surge and the dbms was in the middle of mutating a given page how do
we recover from this ?

All these questions will be answered in coming up posts when we discuss concurrency management with
transactions and recovery management with logging.

# References

[^1]: [Database Storage - Andy Pavlo CMU
  15-445](https://www.youtube.com/watch?v=df-l2PxUidI&list=PLSE8ODhjZXjaKScG3l0nuOiDTTqpfnWFf&index=4)

[^2]: Most operating systems implement asynchronous I/O so when you call `write` on a file the data
  is not written immediately but kept in memory for a while until the kernel decides to do it, this
  behavior can forced by calling the `flush` syscall.

[^3]: [Crotty et al. Are you sure you want to use `mmap` in your
  DBMS](https://db.cs.cmu.edu/mmap-cidr2022/)
