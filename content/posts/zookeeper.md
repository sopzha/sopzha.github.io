+++
date = 2024-12-25T03:00:00-05:00
draft = false
title = 'system summary: zookeeper'
+++

### Overview

Zookeeper is a coordination service for distributed applications. It exposes
a simple client API that applications can use to implement higher level
primitives.

### Data model
ZooKeeper's data model is similar to a standard file system. Data used for
coordination is stored in znodes, organized in a hierarchical namespace. Each
znode can be referenced by clients via a file system pathname. For example,
`/A/B` denotes the path to znode `B`, whose parent is `A`.

There are two types of znodes that clients can create.
1. *Regular* znodes are explicitly deleted.
2. *Ephemeral* znodes can be either explicitly deleted or automatically removed
when the session in which they were created terminates.

### Client API
Below is a subset of the ZooKeeper API.

`create(path, data, flags)` creates a znode with the pathname `path`, stores
`data[]` in it, and returns the name of the new znode. `flags` enables a client
to select the type of znode.

`delete(path, version)` deletes the znode `path` if that znode is at the
expected version.

`exists(path, watch)` returns true if the znode `path` exists,
otherwise false. The `watch` flag enables the client to receive notifications
of changes to the znode.

`getData(path, watch)` returns the data and metadata associated with the znode
`path`.

`setData(path, data, version)` writes `data[]` to znode `path` if the version
number is the current version of the znode.

`getChildren(path, watch)` returns the set of names of the children of the
znode `path`.

### Example primitives
Distributed applications can use the ZooKeeper API to implement powerful
primitives. Two examples of useful primitives are configuration management and
leader election.

#### Configuration management
1. Configuration data is stored in a znode.
2. Each process obtain its configuration by reading the configuration znode,
with the watch flag set to true.
3. If the configuration is updated, processes are notified and read the new
configuration.

#### Leader election
1. A parent znode (e.g. `/election`) is created to serve as a namespace for the
election.
2. Each participant creates an ephemeral sequential znode under the parent (e.g.
`/election/node-0001`). If the participant's session ends, this znode is
automatically deleted.
3. Each participant gets a list of all children znodes and sorts them based on
sequential ID.
4. The participant whose znode has the smallest sequential ID is elected as the leader.
5. Non-leaders set a watch on the znode immediately preceding their own in the
sorted list. If the predecessor is deleted, the participant rechecks the list
to see if they are now the leader.


### Example application
The Fetching Service (FS) in the Yahoo! crawler relies on ZooKeeper. This
service has master processes that command processes responsible for fetching pages.

The FS uses the configuration management primitive. Configuration data is
stored in znodes. The FS registers watches on these znodes, such that when
configuration data is updated, ZooKeeper notifies the FS to dynamically apply
the new configuration.

The FS additionally uses the leader election primitive in order to elect master
processes and recover from failed masters.


### Implementation
ZooKeeper contains a cluster of servers called an ensemble that maintains an
in-memory database replicated across all servers. The ensemble consists of one
leader and multiple followers.

Each client connects to exactly one server to submit requests.
- Read requests are serviced from the local database replica.
- Write requests are processed by an agreement protocol. The leader broadcasts
a proposal for the request to all followers. If a majority of followers agrees
to the proposal, the leader applies the change to its local state and notifies
all followers to do the same.

<!-- TODO: insert diagram -->

While data is primarily stored in memory, ZooKeeper periodically writes changes
to disk to enable recovery from failure.

### Evaluation
ZooKeeper provides strong consistency guarantees, more specifically FIFO client
ordering of all operations (i.e. all requests from a single client are processed
in the order they were sent) and linearizable writes.

ZooKeeper is also highly performant, achieving throughput of hundreds of
thousands of operations per second for read-dominant workloads. Write-intensive
workloads, on the other hand, may not be as efficient because writes require
majority agreement among ZooKeeper servers.

### References
https://dl.acm.org/doi/10.5555/1855840.1855851
https://zookeeper.apache.org/doc/r3.9.3/zookeeperOver.html
