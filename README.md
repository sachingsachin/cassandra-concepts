# cassandra-concepts


## Type of keys

1. `Partitioning Key` - Used to distribute the data across nodes.  
2. `Clustering Key` - Within a node, clustering key is used to get the data by ID. Additionally, the data is kept sorted based on clustering key so that range queries can be performed.


## Multiple fields as keys

`Partitioning Key` can be made up of more than one field. This helps to distribute the data more evenly across nodes.  
Example: for an `orders` table, if only the `category` is chosen as the partitioning key, then it is very likely that more orders will land up in the `phone` partition rather than the `laptop` partition (Number of phone users are way more than laptop users). If that happens, then the node holding the phone orders will begin to underperform. However, if you add year and month to the partition key, then each category gets further split by years and months. Hence the data gets very evenly distributed on all nodes regardless of the category.

`Clustering Key` can also be made up of more than one field. This helps to do range queries and sorting operations by more than one key.  
Example: we can have `days` and `hours` as the clustering keys for an `orders` table. This helps to run following queries:
1. Get orders for a day
2. Get orders between day X and day Y
3. Get orders for X day between Hour Y and Hour Z


## Distribution of data across nodes via vnodes

Output of a hash function is a number (integer or long).
In traditional distributed systems, each node is responsible for a contiguous range of this hash function's output.  
A simplified example: Assume that hash function range will output a number from 1 to 100.
If there are 5 nodes in the cluster, then a simple way is to divide it is as contiguos ranges:
1. Node 1: 1 to 20
2. Node 2: 21 to 40
3. Node 3: 41 to 60
4. Node 4: 61 to 80
5. Node 5: 81 to 100

However, this is inflexible from adding/removing nodes perspective because any change of node count changes the hash-distribution among nodes in a very big way. Below is an example of taking away one node:

1. Node 1: 1 to 25
2. Node 2: 26 to 50
3. Node 3: 51 to 75
4. Node 4: 76 to 100


So an easier way is to divide the above 0 to 100 hash-output range into say 20 vnodes such that each v-node is resposible for a range of 5.  
These vnodes can then be easily assigned to any node. We need not assign contiguous hash function ranges to a physical node.
Rather we can assgin vnodes to each node and they can be discontiguous.

Earlier, we had each node handled keys whose hash fell between X<sub>n</sub> and Y<sub>n</sub>.  
Where as in the vnodes model, each node handles K vnodes where each vnode handles keys whose hash falls between X<sub>n</sub> and Y<sub>n</sub>.  

Example:
For a hash function that outputs a number from 1 to 100, we could have 20 vnodes such that:
1. vnode1: 1 to 5
2. vnode2: 6 to 10
3. ...
4. vnode20: 96 to 100

And the distribution of vnodes among 5 physical nodes can look like:
1. Node 1: vn1, vn7, vn8, vn20
2. Node 2: vn2, vn3, vn10, vn19
3. And so on

Note that we have 20 vnodes/5 nodes = 4 vnodes / node

So each node now is responsible for small contiguous ranges each of which is called a vnode.
But the overall responsibility of each node is now no longer contiguous.

When a new node is added, each of the existing node can now give a few of their vnodes to the new node so that data distribution is spread equally. Since each node gives a little bit of its data, the bootstrapping load is equal on each node. At the same time, the distribution of data can be changed one node at a time.
If the vnode concept was not available, then the only option would be to split the token-range into half or one-Nth on each node, making it impossible to incrementally add a node to the cluster.

https://www.datastax.com/blog/virtual-nodes-cassandra-12 provides some more informtation on the same.



## Consistent Hashing

The above is an implementation of [consistent hashing](https://www.toptal.com/big-data/consistent-hashing).  
Idea behind consistent hashing is to keep the hashing of keys constant even though servers are added or reduced from the distributed hashing. Example:  

If we were to just distribute keys naively as:  
$~~~~~~~~~~~$ server = hash(key) / N  
where `N` is the number of servers.  

In this scheme, whenever number of servers `N` changes, the server for each key would change completely and that would result in a lot of data migration from one server to another. To prevent this, the hash(key) is not mapped to server but to a virtual server number (basically same as vnode). Then every server randomly picks equal number of these vnodes (more or less) and whenever N changes, all other servers reduce or increase their vnodes accordingly. Beauty of this solution is that most vnodes do not leave their current server and load of moving some vnodes is equally spread on all servers.  


## Masterless architecture

[Reference](https://cassandra.apache.org/_/cassandra-basics.html)  
Cassandra has a masterless architecture – any node in the database can provide the exact same functionality as any other node – contributing to Cassandra’s robustness and resilience.  
The node that gets the request at any particular moment is called a coordinator. Any node can act as the coordinator. The coordinator node does the hashing on the key to find the right destinaton for the read/write operation. In addition it also waits for X number of replicas to respond positively where X is the consistency chosen for that query. The consitency can be in write operation or read operation. To avoid stale reads at any point, its recommended to have write-consistency + read-consistency > replication factor of the data.


## Compaction Strategies


### Size-Tiered Compaction Strategy (STCS)
Just merges 4 similar sized SS-tables into 1.
The value 4 is default but can be configured.
This strategy is great for write-intensive cluster but can result in very large SS-Tables over time.
And larger those SS-tables become over time, the more difficult they become to merge.
This strategy also wastes more disk space because the larger SS-tables continue to hold several versions of older data for a long time.
And due to so many versions being there, this strategy also hurts read performance.

### Level Compaction Strategy (LCS)
Not very clearly explained but [this explanation](https://www.scylladb.com/2018/01/31/compaction-series-leveled-compaction/) seems decent.
The idea in this strategy is to break-up bigger SS-Tables formed in STCS into smaller ones such that no SS-Table is more than a given size (default 160 MB).
And to make it manage-able, levels are introduced. Each level contains entire token range, but different number of SS-tables.

Example:
1. Level 1 contains 10 SS-Tables covering entire token range.
2. Level 2 contains 100 SS-Tables covering entire token range.
3. Level 3 contains 1000 SS-Tables covering entire token range.
And so on

Level 0 is where mem-tables are flushed into disk. At this level, STCS is used.
As soon as level 0 contains 1 SCTS-compacted SS-Table of size greater than 160 MB, that big SS-Table is taken and merged with all the 10 SS-Tables of level 1.
The resulting data at level 1 is then split again into smaller SS-Tables at level 1, while keeping each SS-Table less than 160 MB.

The splitting into smaller SS-tables is such that the resulting SS-Tables are non-overlapping with clearly defined token ranges.
Hence each table roughly stores 10% of the token ranges.

SSTables at level 2 are 100 - so each of them holds roughly 1% of token ranges.
But that is the beauty of LCS.
This kind of sorted-splitting ensures that roughly each SSTable at level X corresponds to about 10 SSTables at level X+1
And whenever you merge an SSTable from level X to X+1, you are only merging 11-12 SSTables.
Also, merges at higher levels are on increasingly smaller token ranges and hence complete very fast.



### Time Window Compaction Stratgey (TWCS)
This strategy is useful for time-series data (Time-series data is the one where timestamp is used as part of the clustering key).
When doing compaction in this strategy, SSTables are compacted according to time-window.
Example: all SS-Tables containing data from 10pm to 11pm are compacted into a single SS-table after 11 pm
It is important to have a TTL with time-series data so that after a while, the older SS-Tables just get dropped off as a whole instead of being compacted.
This can happen if a large chunk of entries in older SS-Tables expire at the same time.
When this happens, compaction is just removing the older SS-Tables - much much faster than analyzing different SS-Tables and merging them.

### 

Read more [here](https://docs.datastax.com/en/dse/5.1/dse-arch/datastax_enterprise/dbInternals/dbIntHowDataMaintain.html#dbIntHowDataMaintain__dml_types_of_compaction)

## Snitch

[Snitch](https://cassandra.apache.org/doc/stable/cassandra/architecture/snitch.html) is used in Cassandra to know about the nodes regions and availability zones. Region in Cassandra is called datacenter and availability zone is called a rack. This knowledge is essential for Cassandra to provide different kinds of tunable consistencies.


## Gossip

[Gossip](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/architecture/archGossipAbout.html) is a protocl for Cassandra nodes to share state with one another. Every second, each Cassandra node communicates its state to randomly picked 3 other nodes. The states have versions associated with them as well so that older states are easier to overwrite while any stale state is ignored if communicated accidentally.


