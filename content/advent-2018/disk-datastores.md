+++
author = ["Gleicon Moraes"]
title = "Golang and local datastores - fast and flexible data storage"
linktitle = "golang local datastores"
date = 2014-02-06T06:40:42Z
+++

Local datastores may not be a fit if you are building a web application 
that may have more than a single instance and a somewhat rich data schema.
But they are an important building block to know if you are looking for real fast 
temporary storage or can build your own replication.

In this post I will show how I've used key/value local databases in Go to 
build a database server named `Beano` that speaks memcached protocol and can 
hot-swap its whole dataset gracefully.

A short taxonomy of Go data storage libraries
============================================


I've had used SQLite, BerkeleyDB and knew about InnoDB but for some reason
I've never spent too much energy on them as I did on database servers.

But local data storages really hit me when I've read LevelDB's design document; 
it made me think on how well thought it was, using SST files andbloom filter 
to reduce disk usage.

Databases like LevelDB offers very little concurrency management - actually very 
little management tools at all. The database lives in a directory and can be accessed 
by one process at time. The [documentation](https://github.com/google/leveldb/blob/master/doc/index.md) is focused on C++ and 
its internals in a very practical way.

Its not anything new: the query pattern is Key/Value: you GET, PUT or DELETE 
data by a Key. There are an iterator interface plus some sort of transaction.

What is important to us is that among the databases that we will find in the 
Go ecosystem we will find databases based on LevelDB or some evolution of it 
as BadgerDB, some of them implemented after the venerable LMDB and completely 
novel designs. In most sense their APIs are not that distinct.

In that diverse world the taxonomy that I use is simple:

- What is the performance profile ? Databases based on variations of BTree 
will be great for reads, LSM Trees are great for writes.
- How's the data organised in the disk ? Single big file that all goroutines 
will write to, SST and WAL files that append data and lower the locking burden
- Is it Native Go code ? Easier to understand and contribute too (and frankly 
I had a bad time with signals while testing bindings to LevelDB).
- Does it implements iterators ? Can I order they keys under some criteria ?
- Does it implements transactions - most of them do and it is not like RDBMS transactions 
where you can commit or rollback concurrently. But they are useful for isolation.
- Compactions, snapshots and journaling are interesting features to explore.

Beano: born of a case of legacy migrations
==========================================

I was working with a set of legacy applications that had a rigorous process 
and availability requirements, in languages that would be hard to change the  
database implementation in less than one year. I needed to patch the dataset 
data quickly without changing the main database schema.

I'll show an application called [beano](https://github.com/gleicon/beano),
is a Key Value database server that understands most of Memcached protocol,
stores data locally and can swap the whole dataset live.

The reason I've built Beano is that I was dealing with a set of applications
that were hard to change, coupled to a service bus that already exposed a 
cache abstraction that ran memcached under it and the data at the memcached 
servers were basically a denormalised version of the main database schema.

We had had a script to warm up the cache by running pre-defined queries 
and setting the right keys on memcached, which done wrong would case a
lot of trouble as the database latency was high.

Based on that I've tried to implement a way of loading pre-defined datasets 
into memcached and swap them in runtime. My first shot was a new memcached 
feature at the time, which made GA later: pluggable backends. 
I've built [a LevelDB backend](https://github.com/gleicon/memcached_fs_engine) for memcached, 
and while at that [a meaningless redis backend](https://github.com/gleicon/memcached_redis_engine).

That worked but not as I wanted mostly because that level of C 
programming was beyond me. I was learning Go and the idea of implementing 
the parts of memcached that interested me and coupling with a local database 
was interesting. 

After some interactions with non-native LevelDB wrappers, signal issues and 
wire debugging to learn the memcached protocol I've had a server that would 
switch databases on the fly if you posted to a REST api while communicating 
through memcached clients or the abstractions I've had.

Beano's internals
=================

![architecture](/postimages/advent-2018/disk-datastores/beano_arch.png)

Finding out the proper database was and still what I work most so I've 
refactored the server code to allow for a pluggable backend through an 
interface. That allowed me, for example, to implement a bloom filter in 
front of BoltDB at the time to match with LevelDB architecture.

```go
package main

/*
 Backend interface
*/

type BackendDatabase interface {
	Set([]byte, []byte) error
	Add([]byte, []byte) error
	Replace([]byte, []byte) error
	Incr([]byte, uint) (int, error)
	Decr([]byte, uint) (int, error)
	Increment([]byte, int, bool) (int, error)
	Put([]byte, []byte, bool, bool) error
	Get([]byte) ([]byte, error)
	Range([]byte, int, []byte, bool) (map[string][]byte, error)
	Delete([]byte, bool) (bool, error)
	Close()
	Stats() string
	GetDbPath() string
	Flush() error
	BucketStats() error
}
```

All the operations to be implemented in a new datastore backend are defined 
on src/backend.go. These functions follow the Memcached protocol loosely. At
the file src/networking.go there's a channel that listen to database paths.

The process which swaps these backends communicates through this channel and 
coordinates with current active connections to be drained if it is a different db.

There is a provision for a new command but currently I'm using an API route `/api/v1/switchdb`

```go
func switchDBHandler(w http.ResponseWriter, req *http.Request) {
	if req.Method != "POST" {
        http.Error(w, "405 Method not allowed", 405)
		return
	}
	filename := req.FormValue("filename")
	if filename == "" {
		http.Error(w, "500 Internal error", 500)
		return
	}
	messages <- filename
	w.Write([]byte("OK"))
}
```

The only function that knows the implementations of backend interfaces is 
the following, in the same file:

```go
func loadDB(backend string, filename string) BackendDatabase {
	var vdb BackendDatabase
	var err error
	switch backend {
	case "boltdb":
		vdb, err = NewKVBoltDBBackend(filename, "memcached", 1000000)
		break

	case "badger":
		vdb, err = NewBadgerBackend(filename)
		break

	case "inmem":
		vdb, err = NewInmemBackend(1000000)
		break

	default:
	case "leveldb":
		vdb, err = NewLevelDBBackend(filename)
		break

	}
	if err != nil {
		log.Error("Error opening db %s", err)
		return nil
	}
	return vdb

}
```

There is a watchdog to receive the messages through the channel which will 
prepare the db for the new connections, and right after that lives the `accept()` 
loop that calls the protocol parsing function

```go
if err == nil {
	for {
		if conn, err := listener.Accept(); err == nil {
			totalConnections.Inc(1)
			go ms.Parse(conn, vdb)
		} else {
			networkErrors.Inc(1)
			log.Error(err.Error())
		}
	}
} else {
	networkErrors.Inc(1)
	log.Fatal(err.Error())
}
```

Note that `ms.Parse(conn, vdb)` will 

This is all to separate networking from the backend and implement hot swap. 
The parsing knows the backend interface, not the implementation details:

```go
func (ms MemcachedProtocolServer) Parse(conn net.Conn, vdb BackendDatabase) {
	totalThreads.Inc(1)
	currThreads.Inc(1)
	defer currThreads.Dec(1)
	conn.SetReadDeadline(time.Now().Add(time.Second * 10))
	defer conn.Close()
	startTime := time.Now()
	for {
        buf := bufio.NewReadWriter(bufio.NewReader(conn), bufio.NewWriter(conn))
        
        ...
    	responseTiming.Update(time.Since(startTime))
	}
}
```

With that structure we could create a new database with a standalone beano 
instance, using the warm-up scripts, transfer through rsync or other methods 
and have them hot swapped safely.

Datastorages
============

Now getting to the datastores, each one has a semantic around its transactions. Let's see the GET method on BadgerDB

```go
func (be badgerBackend) NormalizedGet(key []byte) ([]byte, error) {
	var item *badger.Item
	err := be.db.View(func(txn *badger.Txn) error {
		var err error
		item, err = txn.Get(key)
		return err
	})
	if err != nil {
		return nil, err
	}
	return item.Value() // []byte value, error
}
```

Now on LevelDB by means of [goleveldb, the library I've used](github.com/syndtr/goleveldb/leveldb):

```go
func (be LevelDBBackend) NormalizedGet(key []byte, ro *opt.ReadOptions) ([]byte, error) {
	v, err := be.db.Get(key, be.ro)
	// impedance mismatch w/ levigo: v should be nil, err should be nil for key not found
	if err == leveldb.ErrNotFound {
		err = nil
		v = nil
	}
	return v, err
}
```

There is a comment about `levigo`, because at some point in time I've used 
both libraries to provide native and non-native-with-wrapper LevelDB and 
compare performance and safety before switching libraries.

I've kept the `BoltDB` implementation around even after the project was 
archived to document what is possible with these abstraction. BoltDB 
had no layer between the request and hitting the disk, as LevelDB presents 
with a probabilistic data structure called `bloom filter` to provide a space 
aware key cache. 

```go
type KVBoltDBBackend struct {
	filename         string
	bucketName       string
	db               *bolt.DB
	expirationdb     *bolt.DB
	keyCache         map[string]*BloomFilterKeys
	maxKeysPerBucket int
}
```


Everytime a `GET` is performed, it has to check if the key was seen. 
Same for `PUT` and `ADD` - both functions have to load the bloom filter 
with the keys they are commiting.


```go
func (be KVBoltDBBackend) Get(key []byte) ([]byte, error) {
	var val []byte
	bf := be.keyCache[be.bucketName].Test(key)
	if bf == false {
		return nil, nil
	}
	err := be.db.View(func(tx *bolt.Tx) error {
		bucket := tx.Bucket([]byte(be.bucketName))
		if bucket == nil {
			return fmt.Errorf("Bucket %q not found!", be.bucketName)
		}

		val = bucket.Get(key)
		return nil
	})

	if err != nil {
		return nil, err
	}
	return val, nil
}
```

When a key is requested, `bf := be.keyCache[be.bucketName].Test(key)` 
tests if the key was added to the cache at some point in time. Bloom filters 
are biased to false positives but never to false negatives, meaning that if 
it never saw the key you could return `NOT FOUND` safely while if it saw 
the key there was a chance of a false positive result that would force a 
disk read to check and fetch. 

That helped run LevelDB and BoltDB with close performance for reads, 
while keeping the details local to the BoltDB implementation.

I've used BoltDB until it was archived and switched to BadgerDB when 
it got transactions. I recommend going with BadgerDB as the support is 
great, there is a blossoming community around it.

Conclusion
==========

Among more complex applications for fast local datastorages, there are solutions
like `segment.io` [message deduper for kafka](https://segment.com/blog/exactly-once-delivery/),
timeseries based software as [ts-cli](https://github.com/gleicon/ts-cli) 
and a range of software in the middle.

Beano's repository sits at [github](https://github.com/gleicon/beano) - 
ideas, issues, PRs are welcome. My plans are to look for new databases and 
fork the memcached protocol parsing out of it. 

If you like using known protocols to perform other functions, check my 
`redis` compatible server that only implements PFADD/PFCOUNT using 
HyperLogLog: [nazare](https://github.com/gleicon/nazare).
