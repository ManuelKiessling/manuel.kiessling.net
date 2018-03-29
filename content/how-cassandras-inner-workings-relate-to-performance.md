---
date: 2016-07-11T16:13:00+01:00
lastmod: 2016-07-11T16:13:00+01:00
title: "How Cassandra's inner workings relate to performance"
description: "At Galeria.de, we learned the hard way that it's critical to understand the inner workings of the distributed masterless database Cassandra if one wants to experience good performance during reads and writes. This post describes some of the details of how Cassandra works under the hood, and shows how understanding these details helps to anticipate which use patterns work well and which don't."
authors: ["manuelkiessling"]
slug: 2016/07/11/how-cassandras-inner-workings-relate-to-performance
canonical: http://blog.kaufhof.io/tutorials/2016/02/29/how-cassandras-inner-workings-relate-to-performance
---

<em>This is a cross post from the <a href="http://galeria-kaufhof.github.io/tutorials/2016/02/29/how-cassandras-inner-workings-relate-to-performance">GALERIA Kaufhof Technology Blog</a>.</em>

<h2 id="about">About</h2>

<p>At Galeria.de, we learned the hard way that it's critical to understand the inner workings of the distributed
masterless database Cassandra if one wants to experience good performance during reads and writes. This post describes
some of the details of how Cassandra works under the hood, and shows how understanding these details helps to anticipate
which use patterns work well and which don't.</p>

<h2 id="network-and-node-storage-architecture">Network and node storage architecture</h2>

<p>Roughly speaking, there are two main areas within the Cassandra architecture that play a deciding role with regards to
query performance &#8211; the network of nodes that form the database cluster, and the local storage on each of those nodes.</p>

<p>Efficient queries must be efficient network-wise as well as storage I/O wise. Let's dig into both areas and see how
things work under the hood. If we understand the inner workings of both, we should be prepared to anticipate why certain
table-structure/query combinations are efficient and some are not.</p>

<h3 id="the-network">The network</h3>

<p>A production Cassandra setup always consists of multiple nodes, where a node is one Cassandra server process on one
system. All nodes are connected via the network. There isn't any kind of “master” node &#8211; all nodes are created equal.</p>

<p>Logically, the data in a cluster is organized into keyspaces, which contain tables. Tables contain rows, and rows have
columns.</p>

<p>Physically, the content of a table row is always stored on the hard drive of at least one node in the cluster, and,
depending on how the keyspace has been defined upon creation, this row content is replicated to 0 or more other nodes
in the cluster. If all of this doesn't make sense now, it will once you've read this post.</p>

<p>In this post, we always assume that our setup is a cluster of 5 nodes, numbered 1 to 5. A 5 node Cassandra cluster is
visualized as follows:</p>

<p><img width="50%" src="http://galeria-kaufhof.github.io/assets/images/cassandra/Cassandra cluster.svg" /></p>

<p>Note that this is a logical visualization, not a network diagram &#8211; in reality each node talks to all other nodes in the
cluster, not only to its neighbours.</p>

<p>We further assume that we have created a keyspace “galeria” (which is going to hold the data tables we are going to
create) as follows:</p>

<div class="highlighter-rouge"><pre class="highlight"><code>CREATE KEYSPACE galeria WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor': 3 };
</code></pre>
</div>

<p>The replication factor defines that whatever row we insert into a table within this keyspace is stored on three
different nodes in the cluster.</p>

<p>We can now create a table “users” within this keyspace like this:</p>

<div class="highlighter-rouge"><pre class="highlight"><code>USE galeria;

CREATE TABLE users (
    username TEXT,
    firstname TEXT,
    lastname TEXT,
    PRIMARY KEY (username)
);
</code></pre>
</div>

<p>When we insert a row into this table as follows:</p>

<div class="highlighter-rouge"><pre class="highlight"><code>INSERT INTO users (username, firstname, lastname) VALUES ('jdoe', 'John', 'Doe');
</code></pre>
</div>

<p>then the following happens network-wise:</p>

<h4 id="step-1-client-coordinator-connection">Step 1: Client-Coordinator connection</h4>
<p>Our client (i.e., the process which issues the CQL statement) connects to a so-called <em>coordinator node</em>. This is not a
special node within our cluster &#8211; any node that happens to receive a query from a client also happens to act as the
coordinator for this query.</p>

<h4 id="step-2-mapping-the-partition-key-value-to-a-cluster-node">Step 2: Mapping the partition key value to a cluster node</h4>
<p>The first thing the coordinator node needs to do upon receiving the insert query is to find out where in the cluster the
row data for this INSERT needs to be persisted. Because <code class="inline">username</code> is the first (and in this case only) part of the
primary key, it acts as the partition key. The value of the partition key column is what's used by the coordinator to
determine the first node onto which to store the row. To do so, a hash function &#8211; the so-called <em>partitioner</em> &#8211; is
applied on the value, and the result is a token. This token then tells the cluster about the target node, because each
node is responsible for a certain range of tokens. Assumed that tokens would run from 0 to 49 (in reality, the token
range is much larger), we can visualize the tokens-to-nodes mapping as follows:</p>

<p><img width="50%" src="http://galeria-kaufhof.github.io/assets/images/cassandra/Cassandra cluster with tokens.svg" /></p>

<p>That is, node 3 holds those rows of table “users” in keyspace “galeria” for which the value of column “username” results
in a hash function token from 20 to 29.</p>

<p>For example, let's just make up that the username column value “jdoe” would result in token value 17. This means that
the cluster must store the according row at least on node 2.</p>

<h4 id="step-3-determine-replication-nodes">Step 3: Determine replication nodes</h4>
<p>“At least” because what also comes into play is the replication factor of the keyspace holding the user table (which
contains the row in question). In our example, this factor is 3, which means that the row we create via the INSERT query
needs to be stored on two more nodes, besides its “main” node, 2. The algorithm for this is simple &#8211; additional
replicas of the row are stored on the next nodes clockwise in the ring &#8211; in our example, nodes 3 and 4.</p>

<p>Note that the replication of the write in question happens on all three nodes (2, 3, and 4) simultaneously, and not
one-after-the-other. This detail is important because it explains why Cassandra is relatively optimistic regarding the
to-disk-sync of a node's commit log (see chapter “The node storage” for more on this).</p>

<p>As said, the replica order is based on the logical structure of the cluster &#8211; the cluster sees itself as an ordered ring
structure, where each node has a “neighbour” node that comes “after” it in the ring.</p>

<h4 id="step-4-wait-for-node-write-operations-to-finish-and-report-back-to-the-client">Step 4: Wait for node write operations to finish and report back to the client</h4>
<p>Once enough nodes have reported that their local write operations succeeded (see chapter “The node storage” for the
details), the coordinator node in turn reports the success of the INSERT operation back to the client. Here, “enough”
nodes depends on the <em>replication factor &lt;-&gt; query consistency level</em> relation for this operation. If we insert a row
into a table that belongs to a keyspace with a replication factor of <code class="inline">3</code>, and the query was issued with a consisteny
level of <code class="inline">QUORUM</code>, then <code class="inline">2</code> nodes (the quorum of 3 nodes) acknowledging the write is considered a success by the
coordinator.</p>

<h3 id="the-node-storage">The node storage</h3>

<p>Let's “zoom in” on our node 2 and have a look at what happens in terms of its local storage when it receives the write
request from the coordinator node as a result of the INSERT query issued by the client. As noted, the same operations
happen on nodes 3 and 4 simultaneously.</p>

<p>While the final goal is to have the data stored in a so-called <em>SSTable</em> on disk, several moving parts are involved in
persisting the row data on a node:</p>

<p><img width="50%" src="http://galeria-kaufhof.github.io/assets/images/cassandra/Cassandra node storage.svg" /></p>

<p>Why are there three different storage mechanisms combined in order to persist data and make data retrievable? The reason
is that only the interplay of these three mechanisms gives us a database that will, if used correctly, allow for
efficient data writes, efficient data reads, as well as durable storage of large amounts of data &#8211; which is great
because we certainly want a database that handles our INSERTs quickly, answers our SELECTs fast, doesn't loose any of
our data while doing so, and stores more data than what fits into expensive and therefore limited memory.</p>

<p>So, looks like we have four important qualities that we want to have covered on the storage level: fast writes, fast
reads, plenty of capacity, durable storage.</p>

<p>Each one of the three mechanisms (commit log, MemTable, SSTables) covers at most three of these four qualities:</p>

<p>If we would only care about storing a lot of data in a durable way, the commit log would do &#8211; writing into it is fast,
and it is stored on the harddisk (but finding and reading all data for a desired row is very slow).</p>

<p>If we would only care about read and write performance, the MemTable would do &#8211; writing to and reading from it is fast
because all data of the MemTable lives in memory (but memory size is limited, and the data is not stored in a durable
way).</p>

<p>If we would only care about fast reads and durable storage of lots of data, then the SSTables would do, because these
are stored on disk in a structure that allows to quickly locate a desired data element (but writing data into this
structure is slow).</p>

<p>If we want to cover all four qualities, all three mechanisms need to be combined. Let's see how this works in practice.</p>

<h4 id="the-commit-log">The commit log</h4>
<p>The first step taken storage-wise is to write the row data of our INSERT into the commit log.</p>

<p>The commit log is part of the process because it ensures data durability even in crash scenarios. Even if the server
crashes during a write operation &#8211; if the data made it into the commit log, our data is safe and it can be recovered
when the node is coming up again.</p>

<p>Note that, as mentioned above, Cassandra is quite optimistic in regards to actually syncing the commit log writes to
disk &#8211; per default, this happens every 10 seconds, but the node immediately acknowledges the write to the coordinator
(after also writing to the MemTable, see below), without waiting for the fsync. This means that there is a window of up
to 10 seconds during which, in case of a server crash, the data is not persisted on the harddrive of the crashing server
node, although the coordinator will think it is.</p>

<p>What in theory sounds highly problematic in terms of data durability isn't a big deal in practice. Cassandra assumes
that data is always replicated, and two participating server nodes crashing within the same 10 seconds window is very
unlikely.</p>

<h4 id="the-memtable">The MemTable</h4>
<p>Next in line is the so-called MemTable. Why does the MemTable exist? When receiving a read request, Cassandra cannot
retrieve the requested data efficiently from the commit log &#8211; to allow for fast writes, it is append-only, which means
it contains data in the order that write requests arrived, which in turn means that in order to retrieve all data for a
row would mean to sequentially scan through the whole commit log from top to bottom, which would be prohibitively
expensive in terms of disk I/O.</p>

<p>The layout of SSTables, on the other hand, <em>is</em> optimized for efficient lookup of disk-stored row data. However,
Cassandra cannot update the SSTable holding the data for a given row synchronously upon each write, because this would
result in a huge amount of random disk I/O operations, making the write scenario prohibitively expensive in terms of
disk I/O. To circumvent this, SSTables are <em>never</em> updated &#8211; instead, they are created only from time to time, and are
only written to once, and are then immutable (read-only) &#8211; new, additional SSTables are created to cover new row data or
updates to existing row data.</p>

<p>Let's close the circle: If row data cannot be retrieved from the commit log efficiently, and data isn't put into
SSTables immediately, then another data structure is required in order to answer read requests <em>immediately</em> (as soon as
the data is written to the node) <strong>and</strong> <em>efficiently</em>.</p>

<p>And thus, MemTables come into play. Each server node has one MemTable for each table it carries. A MemTable lives,
as the name implies, in memory, and is mutable, i.e., row data is read from it and written to it as needed, which thanks
to the I/O performance of computer memory, is not prohibitively expensive &#8211; a fact that is nicely illustrated by one of
my all time favorite tables, where typical computer-world timings are compared to typical timings that humans can relate
to:</p>

<div class="highlighter-rouge"><pre class="highlight"><code>1 CPU cycle                      0.3 ns     1 s
Level 1 cache access             0.9 ns     3 s
Level 2 cache access             2.8 ns     9 s
Level 3 cache access            12.9 ns    43 s
Main memory access               120 ns     6 m
Solid-state disk I/O          50-150 μs   2-6 days
Rotational disk I/O             1-10 ms  1-12 months
Internet: SF to NYC               40 ms     4 years
Internet: SF to UK                81 ms     8 years
Internet: SF to Australia        183 ms    19 years
OS virtualization reboot           4 s    423 years
SCSI command time-out             30 s   3000 years
Hardware virtualization reboot    40 s   4000 years
Physical system reboot             5 m     32 millenia
</code></pre>
</div>

<p>Thus, reading and writing data from and to main memory versus from and to disk is like waiting for your salad order to
be finished in 6 minutes versus getting the salad somewhere between the day after tomorrow and next year. Sounds like a
MemTable does the job.</p>

<p>And thus, right after appending the INSERT data to the commit log, the node puts the same row data into the MemTable
structure. At this point, the data is both durable (commit log) and efficiently retrieveable (MemTable), and thus, the
data node can acknowledge the write to the coordinator node: Thanks, I have the data, and I'm able to provide it quickly
if anyone asks.</p>

<h4 id="the-sstables">The SSTables</h4>
<p>As already mentioned, this situation is fine for the moment, but without the third mechanism &#8211; SSTables &#8211; we'd quickly
run into problems once the node has to hold more data than the size of its memory allows.</p>

<p>SSTables certainly are the most interesting data structure of the three. A new SSTable is created whenever the MemTable
reaches a certain size (at which point it is considered “full”).</p>

<p>As said, SSTables are immutable, and this results in a certain fragmentation of row data.</p>

<p>Let's assume we would issue the following three write statements, spread over a longer period of time:</p>

<div class="highlighter-rouge"><pre class="highlight"><code>INSERT INTO users (username, firstname, lastname) VALUES ('jdoe', '', '');

UPDATE users SET firstname = 'John' WHERE username = 'jdoe';

UPDATE users SET lastname = 'Doe' WHERE username = 'jdoe';

UPDATE users SET firstname = 'John B.' WHERE username = 'jdoe';
</code></pre>
</div>

<p>Let's further assume that between each of these operations, a lot of other CQL operations took place, and thus, between
these three operations, the MemTable of the target node became full several times and has been flushed into new
SSTables. Our row data is now distributed as follows:</p>

<div class="highlighter-rouge"><pre class="highlight"><code>Memtable: Knows nothing about the row anymore because it has been flushed

SSTable 1: Row data has been stored under partition key 'jdoe', with firstname = '' and lastname = ''

SSTable 2: Row data has been stored under partition key 'jdoe', with firstname = 'John'

SSTable 3: Row data has been stored under partition key 'jdoe', with lastname = 'Doe'

SSTable 4: Row data has been stored under partition key 'jdoe', with firstname = 'John B.'
</code></pre>
</div>

<p>Now imagine we would like to retrieve the full row data via
<code class="inline">SELECT firstname, lastname FROM users WHERE username = 'jdoe'</code>. It's not enough to look into the newest SSTable,
because it only knows about the latest data change for the row. Cassandra has to go through all SSTables, and must put
together the full set of latest row data, while also resolving multiple updates to the same column using the timestamp
of the write event: In our case, the correct <em>firstname</em> value is <code class="inline">John B.</code> in SSTable 4, making the value stored in
SSTable 2 irrelevant.</p>

<p>As said, the structure of an SSTable is optimized for efficient reads from disk &#8211; entries in one SSTable are sorted by
partition key, and an index of all partition keys is loaded into memory when an SSTable is openend. Looking up
a row therefore is only one disk seek, with further sequential reads for retrieving the actual row data &#8211; thus, no
expensive random I/O needs to be performed.</p>

<p>However, if we run a Cassandra cluster over a long period of time, we get more and more SSTables. And because collecting
the actual data for a requested row means searching through more and more SSTables, row reads become less efficient over
time.</p>

<p>In order to avoid this kind of read performance deterioration, Cassandra runs a regular optimization process called
<em>compaction</em>. This process takes multiple SSTables files and combines them into one new SSTable file &#8211; the result could,
for example, look like this:</p>

<div class="highlighter-rouge"><pre class="highlight"><code>SSTable 1 (previously 1 &amp; 2): Row data stored under partition key 'jdoe', with firstname = 'John' and lastname = ''

SSTable 2 (previously 3 &amp; 4): Row stored under partition key 'jdoe', with firstname = 'John B.' and lastname = 'Doe'
</code></pre>
</div>

<p>After compaction, less files need to be searched in order to gather row data. (And there are other measures employed by
Cassandra in order to further reduce disk operations &#8211; for example, a bloom filter is used to determine SSTables that
can be skipped when looking up data).</p>

<h3 id="performance-of-cassandra-operations">Performance of Cassandra operations</h3>

<p>Musings about the performance of a Cassandra operation boil down to two questions: How much work will the coordinator
node have to do in terms of network I/O, and how much work will a participating data node have to do in terms of local
disk I/O?</p>

<p>The fastest operations are those for which the coordinator has to talk to only one data node, and where the approached
data node can find the requested information by searching as little as possible through as little SSTables as possible.
An expensive operation is one where the coordinator node has to talk to all nodes on the cluster, and where each of the
nodes has to scan a lot through many SSTables to retrieve the requested information.</p>

<p>However, if we <em>do</em> need to retrieve a lot of data, we actually <em>want</em> to shoulder the burden of bringing this data up
onto multiple nodes, in order to light the burden each node shoulder &#8211; that is, after all, one of the main reasons
for choosing a database that is distributed and therefore horizontally scalable.</p>

<p>That's why so-called <em>hotspots</em> can be a problem. Imagine if our primary key is a column holding the day of week, as
follows:</p>

<div class="highlighter-rouge"><pre class="highlight"><code>CREATE TABLE logins (
 day_of_week TEXT,
 username TEXT,
 PRIMARY KEY (day_of_week, username)
);
</code></pre>
</div>

<p>Every time a user logs into the system, we write their username into this table, with <code class="inline">day_of_week</code> set to <code class="inline">"monday"</code>,
<code class="inline">"tuesday"</code>, etc.</p>

<p>Because the partition key decides about the node that has to store data for a particular row, each and every write to
this table that happens on a Monday will have to be handled by the same node. Thus, the whole I/O burden for user login
logging will be on one particular node for the whole day (and on another node on another day). Even if your Cassandra
cluster is 1,000 nodes big &#8211; if the I/O throughput limit of the “Monday node” is exhausted on a given Monday, you will
run into problems when trying to log yet another login event for this day.</p>

<p>If we want to retrieve the list of all usernames that logged in on Monday, this is inefficient, too. The query</p>

<div class="highlighter-rouge"><pre class="highlight"><code>SELECT username FROM logins WHERE day_of_week = 'monday';
</code></pre>
</div>

<p>will result in approaching the one node that maps to the <em>“monday”</em> partition key value, and the I/O burden of reading
through all SSTable entries for this day of week is on this one node, while all other nodes stay idle.</p>

<p>The need for balancing table structures and query strategies between the two, often conflicting, goals of spreading
data evenly around the cluster while also minimizing the number of partitions that have to be read is perfectly
explained in <a href="http://www.datastax.com/dev/blog/basic-rules-of-cassandra-data-modeling">Basic Rules of Cassandra Data Modeling</a>, the first and most important document to read if one wants to
use Cassandra.</p>

<p>While our post provides a glimpse under the hood, the Data Modeling post approaches its recommendations from an
“outside” perspective, and teaches the what and how of data modeling and querying. If you mix in the “inside”
view from our post, you should be well equipped to anticipate performance behaviours of your cluster. Look at a table
structure and the queries operating on it. Then think about what happens on the network, and what happens on the disk
of a node during query execution. This should set you on the right track for most cases.</p>

<p>As a final example for this, consider the following keyspace and table:</p>

<div class="highlighter-rouge"><pre class="highlight"><code>CREATE KEYSPACE galeria WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor': 1 };

CREATE TABLE products (
    name TEXT,
    color TEXT,
    PRIMARY KEY (name, color)
);
</code></pre>
</div>

<p>We fill this as follows:</p>

<div class="highlighter-rouge"><pre class="highlight"><code>INSERT INTO products (name, color) VALUES ('Towel', 'red');
INSERT INTO products (name, color) VALUES ('Towel', 'blue');

INSERT INTO products (name, color) VALUES ('Shirt', 'yellow');
INSERT INTO products (name, color) VALUES ('Shirt', 'red');

INSERT INTO products (name, color) VALUES ('Jacket', 'red');
INSERT INTO products (name, color) VALUES ('Jacket', 'blue');
</code></pre>
</div>

<p>Now let's compare these two queries:</p>

<div class="highlighter-rouge"><pre class="highlight"><code>SELECT * FROM products WHERE name IN ('Towel', 'Jacket') AND color = 'red';

SELECT * FROM products LIMIT 1;
</code></pre>
</div>

<p>Which one is more efficient?</p>

<p>The first looks more complex &#8211; we are asking for two different primary keys, plus a clustering key value. The resultset
is two rows, the red towel and the red jacket.</p>

<p>The second one looks simple: All we need is one row.</p>

<p>But in fact, the second query is about 10x as complex as the first one when it comes to query execution. We can even
visualize this. Here is a screenshot showing the output of the statement trace for both queries (you get the tracing
output for all following queries by issueing <code class="inline">TRACING ON</code> on the CQL shell). The statement trace lists all network and
disk operations that need to be run in order to satisfy the query. I've taken screenshots of both text outputs, pasted
them next to each other, and rotated the image by 90 degree. The output for the first query is on top, the output for
the second is beneath it.</p>

<p><img width="50%" src="http://galeria-kaufhof.github.io/assets/images/cassandra/tracing.gif" /></p>

<p>Although no detail information is visible at this scale, we can clearly see how the second query required way more
network and disk operations compared to the first. Why is this?</p>

<p>We have a keyspace with a replication factor of <code class="inline">1</code>, that is, the row data for one partition key is stored on one node
in the cluster. I've run both queries on a 3-node cluster. The first query specifically asks for two partition key
values &#8211; thanks to the hash algorithm, the coordinator node can calculate which nodes it needs to connect to, and on
those nodes, the index of the SSTables leads to the row data without unneccessary disk seek overhead.</p>

<p>The second query is much harder to satisfy: Because no partition key value is provided, the coordinator can not know
which nodes have rows for the table. It has to ask every single node. On each node, again because there is no partition
key value, each SSTable must be sequentially scanned for possible row data.</p>

<p>And this is only on a 3-node cluster. Imagine running the queries on a 1000-node cluster &#8211; not much changes for the
first query, because asking for two partition key values still means visiting at most two nodes (or only one, if both
values happen to resolve to the same node). But for the second query, the situation is even worse: now 1000 nodes need
to be visited for potential row data &#8211; even if only one single row is eventually returned.</p>

<p>With our understanding of the network and storage mechanisms employed by Cassandra, this kind of behaviour can be
anticipated, which helps to avoid unhealthy table structures and query strategies.</p>
