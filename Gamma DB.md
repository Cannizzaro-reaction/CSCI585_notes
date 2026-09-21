# The Gamma Database Machine Project

> Week 2: [ieee90.pdf](https://pages.cs.wisc.edu/~dewitt/includes/paralleldb/ieee90.pdf)

## Architecture

### Shared nothing architecture

**Node structure:**

<img src="week2_assets/node_structure.png" style="zoom: 33%;" />

* **Each node** has its own network interface (NIC, Network Interface Card), DRAM (Dynamic Random Access Memory), mass storage device (disk),  and processor (CPU).
* When you format a disk to be a certain block size, a certain piece of the memory inside DRAM is also formatted to contain frames of that block size. (size of the frame in buffer pool = block size in disk)
* When you need page 1, a copy of this page will be written into a memory buffer (in DRAM)
* How to find that page in memory? -- By the hash table. The hash table is in key-value format, key: disk page number, value: its address in memory
* When looking for a page -> Can be found in hash table? -> Yes, get the location. -> No, get from disk and put to the main memory.
* When something gets changed in the memory, it is marked as dirty. After flushing it back, it get clean. (No need to flush every time something has changed)
* **DRAM** - quick access (nanoseconds), volatile (power-off, then nothing is remained); **Disk** - slow access (milliseconds), non-volatile

**Shared-nothing architecture:**

<img src="week2_assets/shared_nothing.png" style="zoom: 50%;" />

* Since each node has their own CPU, NIC, disk, and DRAM, the need for cooperation is reduced -> Don't need to waste time on network switch

**Gamma process architecture:**

<img src="week2_assets/gamma_process_architecture.png" style="zoom: 50%;" />

* **Catalog Manager**: A central repository of all conceptual and internal schema information for each database. The query manager utilizes the catalog manager to make sure that the syntax of the query is correct.

* **Query Manager**: One query manager process is associated with each active Gamma **user**. Responsible for query parsing (create query trees), optimization, compilation, etc.

  * Single-site query (e.g. select employees with salary=60k) : the QM identifies the appropriate fragment and issue that query to the appropriate process here. (Single-site query will bypass the scheduler process)
  * Multi-site query: The **scheduler** becomes involved. QM sends the query to the scheduler. The scheduler process activates  **operator processes** at each query processor selected to execute the operator, gather the results, and returns it to QM to display for the user.

* **Scheduler Processes**: Each multisite query is controlled by a scheduler process.  Scheduler processes can be run on any processor. **A centralized dispatching process** is used to assign scheduler processes to multi-site queries.

* **Operator Process**:  At least one Operator Process is employed **at each processor** participating in the execution of the operator. These processes are started in advance (not when doing queries).

  A message is sent from the scheduler to each thread, telling it to become a select, a project, hash-join build, has-join probe, etc. The message specifies the role of the thread.



## Key techniques

### Data declustering

* Partitioning attribute

**Strategy**:

* No partitioning attribute: random, round-robin
* Single attribute: hash (e.g. mod), range
  * When selection is done on the partitioning attribute: selection only needs to be done on a single node
  * When selection is done on the non-partitioning attribute: all nodes need to do the query
* What we get from partitioning -- a fragment/shard



### Query processing algorithm

Example: $TMP2 \leftarrow \sigma_{dname=Toy}(Emp\bowtie Dept)$

<img src="week2_assets/query_example.png" style="zoom: 50%;" />

1. Initiate 3 processes: one for **storage** into TMP2, one for the **select** operator, one for the **join** operator

2. Read the Dept table and produces all the records with dname = Toy, forwarding them to the **Join operator** of Step 2

   > The select operator reads the records of a **fragment** and produce those satisfying a selection predicate as its output **stream**

3. The **Join Build operator** *builds* a hash table on dname of its received records

4. Read the Emp record and forward to the **Join operator**. This operator *probes* the hash table for the qualifying record, forwarding them to the **Store operator** of Step 1



**[The join operator: 2 steps --> step 3 (build) + step 4 (probe)]**

* **Build**: Constructs a hash table on the join attribute

  * Tuples from the inner relation are inserted into a memory-resident hash table by hashing on the join attribute value

  * Usually use the **smaller** relation as the inner relation (smaller hash table)

  * The result hash value is then used as an index into the **split table** to obtain **the address of the destination process** that should receive the tuple 

    With split tables, data can be sent directly from an operator to the destination process. Data doesn't flow through scheduler!

  * All tuples with the same join attribute value are in the same partitioned bucket

    <img src="week2_assets/split_table.png" style="zoom: 67%;" />

* **Probe**: Look up the hash table for a joining record

  * Tuples from the outer relation are used to probe the hash table for matching tuples. Every time it finds a match, it produces an output.
  * Probe begins after Build completes (no overlap)



**Scheduler: controls and coordinates where to send its stream of records**

<img src="week2_assets/scheduler.png" style="zoom: 67%;" />

* After a process is initiated, this process gives its socket address back to the scheduler.
* When initiating the SELECT process (#3), the scheduler tells the processor that it should send the data to the port number that JOIN BUILD process just gave back to it (#2).
* When SELECT is done, it directly sends all its data to the JOIN BUILD process, and tells the scheduler it gets done. Then the JOIN BUILD process starts to build hash table.
* When hash table has been built, the scheduler initiates the STORE process and the JOIN PROBE process. Then initiates the SCAN process to read Emp tuples, and send these data to the port number of JOIN PROBE.
* After the JOIN process, the result heads to the temporary table. The scheduler has given it the port number of the STORE process when initiating.



**How to do when there's not enough memory? -> Hybrid hash-join**

1. Make sure the inner (build) table is the smaller of the two tables, say Dept.

2. **Break the inner table into M buckets** using a hash function. The ideal function ensures each bucket is small enough to fit in memory. **Write these buckets to disk**.

3. Break the outer table (Emp) into M buckets using the same hash function used in Step 2. Write these buckets to disk.

4. For i = 1 to M:
  a. Stage Bucket i of Dept in an in-memory hash table
  b. Read Bucket i of Emp and probe the hash table for joining records

  > Records in bucket i of the inner table (Dept) may join with the records in bucket i of the outer table (Emp) only.
  >
  > **Hybrid Hash-Join (optimization): The first bucket of the inner table (Dept) does not have to be written to disk**, it can be staged in memory. Similarly, the first bucket of the outer table (Emp) is not required to be written to disk. It may probe the hash table of the inner table (Dept) for joining records.

**Parallel version of the hybrid hash-join: two kinds of split tables, faster I/O for each fragment** 

<img src="week2_assets/parallel_hybrid_hash_join.png" style="zoom: 67%;" />

* **Partitioning split table**: Separates the joining relations into N logical buckets
* **Joining split table**: Route tuples to their respective joining processor (i.e. split each bucket to K disks) -> parallelizing the joining phase



### Failure management - chained declustering

**Interleaved declustering**:

<img src="week2_assets/interleaved_declustering.png" style="zoom: 67%;" />

* The set of disks are divided into units of size N called **clusters**
* Each backup fragment is subdivided into N−1 subfragments, and each subfragment is placed on a different disk within the same cluster
* Failure of Node 1 distributes its load on Nodes 0, 2 and 3 --> when a node fails, the load can be distributed evenly across the other 3 nodes
* This kind of system can only tolerate one failure in one cluster



**Chained declustering**:

<img src="week2_assets/chained_declustering.png" style="zoom: 67%;" />

* The backup copy is stored on the adjacent node (only on one node)
* With the failure of node 1, primary fragment R1 can no longer be accessed and thus its backup fragment r1 on node 2 must be used for processing queries that would normally have been directed to R1. However, instead of requiring node 2 to process all accesses to both R2 and r1, chained declustering offloads **6/7-ths of the accesses to R2 by redirecting them to r2 at node 3**. In turn, 5/7 ths of access to R3 at node 3 are sent to R4 instead.
* This dynamic reassignment of the workload results in **an increase of 1/7-th** in the workload of each remaining node in the cluster
* Every time two adjacent nodes fail, data will be lost



## Performance

* **Speedup**: Perform a complex query/update faster. Ideally, it should become more than N times faster with N nodes

* **Scaleup**: Maintain the same speed for a complex query/update as the database grows by increasing the number of nodes