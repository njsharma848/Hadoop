# HDFS & Hadoop — Complete Interview Preparation Guide
## Basic | Medium | Hard | Scenario-Based Day-to-Day Commands

---

# PART 1 — BASIC LEVEL QUESTIONS

---

## Q1. What is Hadoop? What problem does it solve?

**Answer:**

Hadoop is an open-source framework for **storing and processing massive datasets** across clusters of commodity (inexpensive) hardware. It was created to solve a fundamental problem: by 2004, datasets were growing faster than any single machine could store or process them.

**The Two Problems Hadoop Solves:**

| Problem | Hadoop's Solution |
|---------|------------------|
| Storage — data too big for one machine | HDFS — distributes data across hundreds of machines |
| Processing — computation too slow on one machine | MapReduce / YARN — distributes computation across the same machines |

**Core Philosophy:** Instead of moving large data to a powerful computer, move the computation to where the data already lives. This is called **data locality**.

**Hadoop Ecosystem Components:**

```
┌─────────────────────────────────────────────────────────┐
│                    APPLICATIONS                         │
│         Hive │ Pig │ HBase │ Spark │ Impala             │
├─────────────────────────────────────────────────────────┤
│              RESOURCE MANAGEMENT                        │
│                      YARN                               │
├────────────────────────┬────────────────────────────────┤
│   DISTRIBUTED STORAGE  │   DISTRIBUTED PROCESSING       │
│         HDFS           │        MapReduce                │
└────────────────────────┴────────────────────────────────┘
              Commodity Hardware Cluster
```

---

## Q2. What is HDFS? How is it different from a regular file system?

**Answer:**

HDFS (Hadoop Distributed File System) is a **distributed, fault-tolerant file system** designed to store very large files across multiple machines and provide high-throughput access to application data.

**Key Differences from a Regular File System:**

| Aspect | Regular File System (ext4, NTFS) | HDFS |
|--------|----------------------------------|------|
| Storage location | Single machine | Hundreds of machines |
| File size | Limited by disk size | Petabytes |
| Data access pattern | Random read/write | Write-once, read-many |
| Fault tolerance | RAID or manual backup | Automatic 3x replication |
| Optimised for | Low latency | High throughput |
| Block size | 4 KB | 128 MB (default) |
| Use case | OS, applications | Batch analytics, Big Data |

**HDFS Design Assumptions:**
- Hardware failure is the norm, not the exception
- Files are large (GB to TB range)
- Data is written once and read many times
- Moving computation to data is better than moving data to computation

---

## Q3. What are the main components of HDFS?

**Answer:**

HDFS has a **master-slave architecture** with three main components:

### 1. NameNode (Master)
- The **brain** of HDFS — manages the file system namespace
- Stores all **metadata**: file names, directory structure, permissions, block locations
- Knows WHICH blocks belong to WHICH file and WHERE each block lives
- Does **not** store actual data
- Single point of management (High Availability mode has 2 NameNodes — Active + Standby)

### 2. DataNode (Workers/Slaves)
- The **muscle** of HDFS — stores actual data blocks
- Reports to NameNode via **heartbeats** every 3 seconds
- Reports **block reports** every 6 hours (list of all blocks it holds)
- Serves read/write requests directly to clients
- A cluster can have hundreds to thousands of DataNodes

### 3. Secondary NameNode
- **Commonly misunderstood** — it is NOT a backup NameNode
- Its job is to perform **checkpointing**: periodically merge the edit log with the fsimage to prevent the edit log from growing too large
- Reduces NameNode restart time
- In HA setups, the Secondary NameNode is replaced by the **Standby NameNode**

```
                    ┌─────────────────┐
                    │   NameNode      │
                    │  (Metadata)     │
                    │  fsimage +      │
                    │  edit log       │
                    └────────┬────────┘
                             │ Heartbeat + Block Reports
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
       ┌──────────┐   ┌──────────┐   ┌──────────┐
       │DataNode 1│   │DataNode 2│   │DataNode 3│
       │ Block A1 │   │ Block A2 │   │ Block A1 │
       │ Block B2 │   │ Block A1 │   │ Block B1 │
       │ Block C1 │   │ Block B1 │   │ Block C1 │
       └──────────┘   └──────────┘   └──────────┘
```

---

## Q4. What is a Block in HDFS? What is the default block size?

**Answer:**

A **block** is the smallest unit of data storage in HDFS. When a file is stored in HDFS, it is divided into fixed-size blocks, and each block is stored independently across different DataNodes.

**Default Block Size:**
- Hadoop 1.x: **64 MB**
- Hadoop 2.x and later: **128 MB** (configurable via `dfs.blocksize`)

**Example:**

```
File: sales_data.csv = 450 MB

HDFS splits it into:
  Block 1: 128 MB  → stored on DataNode 1, 3, 5 (3 replicas)
  Block 2: 128 MB  → stored on DataNode 2, 4, 6 (3 replicas)
  Block 3: 128 MB  → stored on DataNode 1, 2, 4 (3 replicas)
  Block 4:  66 MB  → stored on DataNode 3, 5, 6 (3 replicas)
```

**Why 128 MB and not smaller (like 4 KB)?**

- Minimises NameNode memory usage — fewer blocks = less metadata
- Reduces client-NameNode communication overhead
- Optimised for sequential reads of large files
- Allows MapReduce tasks to process one block per task (data locality)

**Why not larger (like 1 GB)?**

- Reduces parallelism — fewer blocks = fewer parallel tasks
- A single failed task means re-processing a huge chunk of data

---

## Q5. What is Replication in HDFS? What is the default replication factor?

**Answer:**

Replication is HDFS's primary mechanism for **fault tolerance**. Each block is stored on multiple DataNodes simultaneously. If one DataNode fails, the data is still available from another.

**Default Replication Factor: 3**

```
Block A (128 MB):
  Copy 1 → DataNode 1 (same rack as client — for speed)
  Copy 2 → DataNode 4 (different rack — for fault tolerance)
  Copy 3 → DataNode 7 (different rack — extra safety)
```

**Rack Awareness — How HDFS Places Replicas:**

```
Rack 1:          Rack 2:          Rack 3:
┌──────────┐     ┌──────────┐     ┌──────────┐
│DataNode 1│ ←── │DataNode 4│     │DataNode 7│ ←──
│ Replica 1│     │ Replica 2│     │ Replica 3│
└──────────┘     └──────────┘     └──────────┘
  ^                                    ^
  First replica on local node   Third replica on different rack
```

**Rack-Aware Placement Rule (default):**
- Replica 1 → Local DataNode (or same rack as client)
- Replica 2 → DataNode on a **different rack**
- Replica 3 → DataNode on the **same rack** as Replica 2 but different node

This balances: write performance (fewer cross-rack hops) + fault tolerance (survives rack failure).

**Configuring Replication:**
```bash
# Set replication factor at cluster level
hdfs dfs -setrep -R 3 /user/data/

# Set replication for a specific file
hdfs dfs -setrep 2 /user/data/small_file.csv

# Check replication factor of a file
hdfs dfs -stat %r /user/data/sales.parquet
```

---

## Q6. What is the NameNode's fsimage and edit log?

**Answer:**

The NameNode maintains the file system state using two critical files:

### fsimage (File System Image)
- A **snapshot** of the entire file system metadata at a point in time
- Stored on the NameNode's local disk
- Contains: directory tree, file metadata, block-to-file mappings, permissions
- Read into memory when NameNode starts up

### edit log (Transaction Log)
- A **sequential log** of every change made to the file system after the last fsimage was written
- Every create, delete, rename, permission change is appended here
- Grows continuously during NameNode operation

### The Problem (Without Secondary NameNode / Checkpointing)
```
Day 1:   fsimage (10 GB) + edit log (0 MB)
Day 30:  fsimage (10 GB) + edit log (8 GB)  ← NameNode restart takes 18 GB to load
Day 180: fsimage (10 GB) + edit log (40 GB) ← Startup takes hours
```

### Checkpointing (Secondary NameNode's Job)
```
1. Secondary NameNode downloads fsimage + current edit log from NameNode
2. Loads fsimage into memory, replays edit log on top of it
3. Produces a new merged fsimage (checkpoint)
4. Uploads new fsimage back to NameNode
5. NameNode replaces old fsimage with new one, starts a fresh edit log

Result: edit log stays small, NameNode startup is fast
```

---

## Q7. What is YARN? What are its components?

**Answer:**

YARN (Yet Another Resource Negotiator) is the **resource management and job scheduling layer** of Hadoop. Introduced in Hadoop 2.x, it decoupled resource management from MapReduce, enabling multiple processing frameworks (Spark, Tez, Flink) to run on the same cluster.

**YARN Components:**

### 1. ResourceManager (Master)
- Global resource allocator for the entire cluster
- Tracks which nodes have how much CPU and memory available
- Two components internally:
  - **Scheduler** — allocates resources (containers) to applications
  - **ApplicationsManager** — manages submitted applications lifecycle

### 2. NodeManager (Worker — one per node)
- Manages resources on a single machine
- Launches and monitors **containers** (isolated resource allocations)
- Reports resource usage back to ResourceManager

### 3. ApplicationMaster (Per Application)
- Created for each submitted job — runs inside a container
- Negotiates resources from ResourceManager
- Coordinates execution of tasks within its application
- For MapReduce: MRAppMaster; For Spark: SparkApplicationMaster

### 4. Container
- A resource allocation unit — bundle of CPU + memory on a specific node
- Tasks run inside containers

```
Client submits job
        │
        ▼
┌───────────────────┐
│  ResourceManager  │ ← Cluster-wide resource tracking
│  ┌─────────────┐  │
│  │  Scheduler  │  │
│  └─────────────┘  │
└───────┬───────────┘
        │ Allocates containers
┌───────┴────────────────────────────┐
│         NodeManager (Node 1)       │
│  ┌───────────────┐ ┌────────────┐  │
│  │ApplicationMstr│ │ Container  │  │
│  │  (Task coord) │ │  (Task 1)  │  │
│  └───────────────┘ └────────────┘  │
└────────────────────────────────────┘
```

---

## Q8. What is MapReduce? Explain with a simple example.

**Answer:**

MapReduce is a **programming model** for processing large datasets in parallel across a cluster. It breaks a big problem into two phases:

- **Map Phase** — split the problem, process each piece independently
- **Reduce Phase** — combine the results

### Simple Example: Word Count

**Input file:**
```
hello world hello
world hadoop hello
```

**Map Phase** (each mapper processes one line):
```
Mapper 1 (Line 1): hello→1, world→1, hello→1
Mapper 2 (Line 2): world→1, hadoop→1, hello→1
```

**Shuffle & Sort** (framework groups by key):
```
hadoop → [1]
hello  → [1, 1, 1]
world  → [1, 1]
```

**Reduce Phase** (each reducer sums values for one key):
```
hadoop → 1
hello  → 3
world  → 2
```

**Output:**
```
hadoop  1
hello   3
world   2
```

### MapReduce Flow
```
Input Data
    │
    ▼
[Split into blocks]
    │
    ├──▶ Mapper 1 ──▶ (key, value) pairs
    ├──▶ Mapper 2 ──▶ (key, value) pairs
    └──▶ Mapper 3 ──▶ (key, value) pairs
              │
              ▼
    [Shuffle & Sort — group by key]
              │
    ├──▶ Reducer 1 ──▶ Final output
    └──▶ Reducer 2 ──▶ Final output
              │
              ▼
        HDFS Output
```

---

## Q9. What is the difference between Hadoop 1.x and Hadoop 2.x?

**Answer:**

| Feature | Hadoop 1.x | Hadoop 2.x (YARN) |
|---------|-----------|-------------------|
| Resource management | Embedded in MapReduce (JobTracker) | Separate YARN layer |
| Processing frameworks | MapReduce only | MapReduce, Spark, Tez, Flink, etc. |
| Scalability | ~4,000 nodes | ~10,000+ nodes |
| NameNode HA | No HA — single point of failure | Active/Standby NameNode |
| HDFS Federation | Not supported | Supported (multiple NameNodes) |
| JobTracker memory | Single JT manages all tasks | Distributed (ApplicationMasters per job) |
| Max cluster nodes | ~4,000 | ~10,000+ |

---

## Q10. What is a Combiner in MapReduce?

**Answer:**

A Combiner is a **mini-reducer that runs locally on each mapper node** before the shuffle phase. It reduces the amount of data transferred across the network during shuffle.

**Without Combiner:**
```
Mapper on Node 1 output:
  hello→1, hello→1, hello→1, world→1

All of this is sent over network to Reducer
Network transfer: 4 records
```

**With Combiner:**
```
Mapper on Node 1 output:
  hello→1, hello→1, hello→1, world→1

Combiner runs locally:
  hello→3, world→1

Only 2 records sent over network instead of 4
```

**Key Point:** A Combiner can only be used when the reduce operation is **commutative and associative** — e.g., sum, count, max, min. It cannot be used for operations like average (because partial averages cannot be simply combined).

---

---

# PART 2 — MEDIUM LEVEL QUESTIONS

---

## Q11. How does HDFS handle DataNode failure?

**Answer:**

HDFS is designed assuming DataNode failures are routine. Here is the complete failure detection and recovery sequence:

### Step 1 — Heartbeat Timeout Detection
```
Each DataNode sends heartbeat to NameNode every 3 seconds
If NameNode receives no heartbeat for 10 minutes (dfs.namenode.heartbeat.recheck-interval)
→ NameNode marks that DataNode as DEAD
```

### Step 2 — Block Re-replication
```
NameNode checks which blocks were stored on the dead DataNode
For each under-replicated block:
  NameNode instructs a surviving DataNode that has the block
  to copy it to another healthy DataNode
  
Result: Replication factor is restored automatically
```

### Step 3 — Client Read Failover
```
Client holds a list of DataNodes for each block (from NameNode)
If DataNode 1 fails during read:
  Client automatically switches to DataNode 2 (next replica)
  Read continues transparently — application is unaware of failure
```

### Step 4 — Decommissioning (Planned Removal)
```
# Planned DataNode removal — safe procedure
# 1. Add node to dfs.hosts.exclude file
# 2. Refresh NameNode
hdfs dfsadmin -refreshNodes

# NameNode replicates all blocks away from that node first
# DataNode enters DECOMMISSIONING state
# Once all blocks have enough replicas elsewhere → DECOMMISSIONED
# Now safe to shut down
```

---

## Q12. What is HDFS Federation? Why is it needed?

**Answer:**

**The Problem:** In a standard HDFS cluster, there is ONE NameNode managing ALL files. The NameNode stores all metadata in RAM. This creates a scalability ceiling:

```
Single NameNode RAM (e.g., 128 GB) → can hold metadata for ~1 billion files maximum
Beyond this → NameNode runs out of memory → cluster cannot grow
```

**HDFS Federation** introduces **multiple independent NameNodes**, each managing a separate **namespace volume** (namespace + block pool).

```
Traditional HDFS:
  NameNode (single) → manages /user, /data, /tmp, /hbase
  All DataNodes report to this one NameNode

HDFS Federation:
  NameNode 1 → manages /user namespace
  NameNode 2 → manages /data namespace
  NameNode 3 → manages /hbase namespace

  All DataNodes → report to ALL NameNodes (store blocks for all)
```

**Benefits:**
- Eliminates NameNode memory ceiling
- Allows different teams to manage different namespaces independently
- NameNode failure in one namespace does not affect others
- Enables horizontal scaling of metadata capacity

**Key Concepts:**
- **Namespace Volume** — a NameNode's namespace + its associated block pool
- **Block Pool** — a set of blocks belonging to a single namespace, stored across all DataNodes
- DataNodes store blocks from **all** block pools simultaneously

---

## Q13. What is NameNode High Availability (HA)? How does it work?

**Answer:**

In standard Hadoop, the NameNode is a **single point of failure**. If it crashes, the entire cluster is unavailable until it restarts (which can take 30+ minutes on large clusters). HA solves this.

### HA Architecture

```
Active NameNode ←──── QJM (Quorum Journal Manager) ────▶ Standby NameNode
        │               (Shared Edit Log Storage)               │
        │                                                        │
        │         ZooKeeper (Leader Election)                    │
        └────────────────────────────────────────────────────────┘
        
All DataNodes send heartbeats and block reports to BOTH NameNodes
```

### Key Components

**1. Quorum Journal Manager (QJM)**
- A cluster of **JournalNodes** (minimum 3) that store the shared edit log
- Active NameNode writes every edit to a majority of JournalNodes
- Standby NameNode reads from JournalNodes continuously to stay up-to-date
- Requires at least 3 JournalNodes (tolerates 1 failure)

**2. ZooKeeper + ZKFC (ZooKeeper Failover Controller)**
- ZooKeeper tracks which NameNode is Active
- ZKFC runs on each NameNode machine, monitors NameNode health
- If Active NameNode fails → ZKFC detects failure → triggers ZooKeeper election → Standby becomes Active

**3. Fencing**
- Prevents **split-brain** (both NameNodes thinking they are Active)
- Fencing mechanism: SSH command to kill old Active NameNode's process before new one takes over

### Failover Sequence
```
1. Active NameNode crashes
2. ZKFC on Active NameNode machine detects failure (health check fails)
3. ZKFC releases ZooKeeper session lock
4. ZKFC on Standby NameNode acquires lock → becomes ZKFC leader
5. Fencing applied to old Active NameNode (SSH kill, STONITH)
6. Standby NameNode transitions to Active state
7. Cluster continues — total downtime: 30–60 seconds (automatic failover)
```

---

## Q14. Explain the HDFS Write Process in detail.

**Answer:**

Writing a file to HDFS is a multi-step process involving the client, NameNode, and multiple DataNodes.

```
Client wants to write: report.csv (300 MB)
Block size: 128 MB → 3 blocks
Replication factor: 3
```

### Step-by-Step Write Process

**Step 1 — Client requests file creation**
```
Client → NameNode: "I want to create /data/report.csv"
NameNode checks:
  - Does file already exist? (throws error if yes, unless overwrite)
  - Does client have write permission?
  - Is there enough space?
NameNode → Client: "OK, proceed"
```

**Step 2 — Client requests block allocation for Block 1**
```
Client → NameNode: "Allocate block 1 for /data/report.csv"
NameNode:
  - Assigns a block ID
  - Selects 3 DataNodes for replicas (rack-aware selection)
NameNode → Client: [DataNode1, DataNode4, DataNode7]
```

**Step 3 — Client establishes a pipeline**
```
Client connects to DataNode1
DataNode1 connects to DataNode4
DataNode4 connects to DataNode7

Pipeline: Client → DN1 → DN4 → DN7
```

**Step 4 — Data flows through the pipeline**
```
Client sends 64 KB packets to DN1
DN1 stores packet locally + forwards to DN4
DN4 stores packet locally + forwards to DN7
DN7 stores packet locally + sends ACK back to DN4
DN4 sends ACK back to DN1
DN1 sends ACK back to Client

Client sends next packet only after receiving ACK (or uses async window)
```

**Step 5 — Block 1 complete, request Block 2**
```
After 128 MB written → pipeline closed
Client → NameNode: "Block 1 done, allocate block 2"
New pipeline established for block 2
Process repeats for blocks 2 and 3
```

**Step 6 — File close**
```
Client → NameNode: "Close /data/report.csv"
NameNode records file as complete in fsimage + edit log
File is now visible and readable by other clients
```

**Write Pipeline Diagram:**
```
Client
  │ 64 KB packets
  ▼
DataNode 1 ──packet──▶ DataNode 4 ──packet──▶ DataNode 7
           ◀──ACK────             ◀──ACK────
◀──ACK────
```

---

## Q15. Explain the HDFS Read Process in detail.

**Answer:**

### Step-by-Step Read Process

**Step 1 — Client opens file**
```
Client → NameNode: "Open /data/report.csv"
NameNode returns:
  - List of blocks for the file
  - For each block: list of DataNodes holding replicas (sorted by proximity to client)
```

**Step 2 — Client reads blocks directly from DataNodes**
```
For Block 1:
  Client connects to closest DataNode (e.g., same rack)
  Client reads 128 MB directly from DataNode
  No NameNode involvement during actual data transfer

For Block 2:
  Client connects to closest DataNode for block 2
  Reads 128 MB

For Block 3:
  Client reads remaining 44 MB
```

**Step 3 — DataNode failure during read**
```
If DataNode fails mid-read:
  Client marks that DataNode as bad (for this read session)
  Client tries next DataNode in the replica list
  Read continues — application is unaware
```

**Step 4 — Checksum verification**
```
Every block has a stored checksum
Client verifies checksum after reading each block
If checksum mismatch (data corruption):
  Client reports bad block to NameNode
  Client reads from another replica
  NameNode schedules re-replication of the good block
```

**Read Flow Diagram:**
```
                  ┌──────────┐
Client ──(1)──▶   │NameNode  │  returns block locations
       ◀──(2)───  └──────────┘

Client ──(3)──▶ DataNode 1 ──(4)──▶ Client  (Block 1, direct)
Client ──(5)──▶ DataNode 4 ──(6)──▶ Client  (Block 2, direct)
Client ──(7)──▶ DataNode 2 ──(8)──▶ Client  (Block 3, direct)
```

**Key point:** NameNode is involved only in the metadata lookup. **Data never flows through the NameNode** — always directly between client and DataNode.

---

## Q16. What is Speculative Execution in Hadoop?

**Answer:**

In a large cluster, occasionally one machine runs much slower than others (due to hardware issues, OS processes, network problems). This one **straggler task** can slow down the entire job.

**Speculative Execution** is Hadoop's solution: if a task is running significantly slower than the average, launch a **duplicate (speculative) copy** of that task on another machine. Whichever finishes first — the original or the speculative copy — wins. The other is killed.

```
Task 1: ████████████ (done, 30 sec)
Task 2: ████████████ (done, 32 sec)
Task 3: ██ (straggler — 10 sec in, expected to take 5 min)
         └──▶ Speculative Task 3b launched on another node
              Task 3b: ████████████ (done, 45 sec)
              Task 3 (original) is killed
Total job time: ~45 sec instead of ~5 min
```

**Configuration:**
```xml
<!-- mapred-site.xml -->
<property>
  <name>mapreduce.map.speculative</name>
  <value>true</value>
</property>
<property>
  <name>mapreduce.reduce.speculative</name>
  <value>true</value>
</property>
```

**When to disable:** If tasks are not idempotent (e.g., writing to external databases), speculative execution can cause duplicate writes. Disable it for such jobs.

---

## Q17. What is the difference between InputSplit and Block in HDFS?

**Answer:**

This is a frequently confused concept:

| Aspect | HDFS Block | InputSplit |
|--------|-----------|------------|
| Layer | Storage layer (HDFS) | Processing layer (MapReduce) |
| Definition | Physical chunk of data on disk | Logical chunk of data for one Mapper |
| Size | Fixed (default 128 MB) | Variable — determined by InputFormat |
| Location | DataNode disk | Logical — metadata only |
| Contains | Raw bytes | Byte offset + length + locations |

**Key Insight:**
- A Block is a **physical** storage unit
- An InputSplit is a **logical** unit that tells a Mapper which data to process
- By default, InputSplit size = Block size (one Mapper per block)
- But InputSplits can span block boundaries or be smaller than blocks depending on InputFormat

```
File: access.log (300 MB)
Blocks: [Block1: 0-128MB] [Block2: 128-256MB] [Block3: 256-300MB]

Default InputFormat (FileInputFormat):
InputSplit1 → Block1 (0-128MB, DataNode1)  → Mapper 1
InputSplit2 → Block2 (128-256MB, DataNode2) → Mapper 2
InputSplit3 → Block3 (256-300MB, DataNode3) → Mapper 3

CombineFileInputFormat (combines small files):
InputSplit1 → Block1 + part of Block2 → Mapper 1 (reduces mapper overhead)
```

---

## Q18. What is Rack Awareness in HDFS and why does it matter?

**Answer:**

Rack awareness means HDFS knows the **physical network topology** of the cluster — specifically, which DataNodes are on which network switches (racks). This knowledge is used to make intelligent replica placement decisions.

**Why it matters:**

```
Network Bandwidth:
  Same machine:  Fastest (in-process)
  Same rack:     ~1 Gbps (single switch)
  Cross-rack:    ~1 Gbps but through additional switch (higher latency, shared bandwidth)

Network Failure Domains:
  Rack switch failure → all nodes on that rack become unreachable
```

**Rack-Aware Replica Placement (3 replicas):**
```
Rack 1 (Switch A):     Rack 2 (Switch B):
  DataNode 1 ← Replica 1    DataNode 4 ← Replica 2
  DataNode 2                DataNode 5 ← Replica 3
  DataNode 3                DataNode 6

Rule:
  Replica 1 → Local rack (write performance — fewer hops)
  Replica 2 → Different rack (survives rack failure)
  Replica 3 → Same rack as Replica 2, different node
              (reduces cross-rack bandwidth vs placing on 3rd rack)
```

**Configure Rack Awareness:**
```xml
<!-- core-site.xml -->
<property>
  <name>net.topology.script.file.name</name>
  <value>/etc/hadoop/rack-topology.sh</value>
</property>
```

```bash
# rack-topology.sh — maps IP to rack name
#!/bin/bash
# Input: IP address, Output: rack name
case $1 in
  192.168.1.*) echo "/rack1" ;;
  192.168.2.*) echo "/rack2" ;;
  *) echo "/default-rack" ;;
esac
```

---

## Q19. What is Small Files Problem in HDFS? How do you solve it?

**Answer:**

**The Problem:**

The NameNode stores metadata for **every file** in RAM. A small file (e.g., 1 KB) uses the same amount of NameNode memory as a large file (128 MB) — approximately **150 bytes per file/block**.

```
10 million files × 150 bytes = 1.5 GB NameNode RAM just for metadata
If those files are each 1 KB:
  Total data: 10 million × 1 KB = ~10 GB
  But each has its own block: 10 million blocks on DataNodes
  Processing overhead: 10 million Map tasks (each task startup ~1 sec = 2778 hours)
```

**Solutions:**

### Solution 1 — HAR (Hadoop Archive)
```bash
# Combine small files into a .har archive
hadoop archive -archiveName sales.har -p /data/small_files/ /data/archives/

# Access files inside the archive
hdfs dfs -ls har:///data/archives/sales.har/
```

### Solution 2 — SequenceFile
- Binary format that bundles small files as key-value pairs
- key = filename, value = file content
- Supports compression and splitting

### Solution 3 — CombineFileInputFormat (at processing time)
```python
# In Spark — combine small files when reading
spark.conf.set("spark.sql.files.maxPartitionBytes", "134217728")  # 128 MB
spark.conf.set("spark.sql.files.openCostInBytes", "4194304")       # 4 MB

df = spark.read.option("mergeSchema", "false") \
               .parquet("s3://data/small_files/")
# Spark combines many small Parquet files into fewer partitions
```

### Solution 4 — Compaction Job
```bash
# Periodically merge small files into large ones
# Run a MapReduce or Spark job to read all small files and write one large file
hdfs dfs -getmerge /data/small_files/ /local/merged_file.txt
hdfs dfs -put /local/merged_file.txt /data/merged/
```

---

## Q20. What are the differences between HDFS and Amazon S3?

**Answer:**

| Aspect | HDFS | Amazon S3 |
|--------|------|-----------|
| Type | Distributed File System | Object Storage |
| Consistency | Strong consistency | Strong consistency (since 2020) |
| Namespace | Hierarchical (true directories) | Flat (prefixes simulate directories) |
| Latency | Low (co-located with compute) | Higher (network call to AWS) |
| Data locality | Yes — compute moves to data | No — data must move to compute |
| Rename operation | O(1) — metadata only | O(n) — copies all objects |
| Cost | High (must own hardware) | Low (pay per GB stored) |
| Scalability | Limited by cluster size | Virtually unlimited |
| Durability | 3 replicas (99.9%) | 11 nines (99.999999999%) |
| Best for | Low-latency batch processing | Data lake, decoupled storage |

**Critical difference for Spark on S3:**
```
HDFS rename (used by Spark committer) = atomic metadata operation = fast
S3 rename = copy all files + delete originals = slow + not atomic
→ Use S3A committer (S3AOutputCommitter) or Delta Lake for correct S3 writes
```

---

---

# PART 3 — HARD LEVEL QUESTIONS

---

## Q21. Explain the NameNode memory calculation. How many files can a NameNode handle?

**Answer:**

The NameNode stores all file system metadata in **RAM** (Java heap). The amount of memory needed is directly proportional to the number of files + directories + blocks.

**Memory per object (approximate):**

| Object | Memory |
|--------|--------|
| File metadata (INode) | ~150–200 bytes |
| Directory metadata (INode) | ~150 bytes |
| Block metadata | ~150 bytes |
| One file with one block | ~300–350 bytes |

**Calculation:**

```
Scenario: 100 million files, average 2 blocks per file

Inodes:       100,000,000 × 200 bytes = 20 GB
Blocks:       200,000,000 × 150 bytes = 30 GB
Total:        ~50 GB NameNode heap needed

NameNode JVM overhead (GC, class data, etc.): ~20–30%
Total JVM heap needed: ~65 GB

Recommendation: Provision NameNode with 128 GB RAM for this cluster
```

**Rules of thumb:**
- 1 GB NameNode heap → supports approximately 1 million files/blocks
- 100 GB NameNode heap → supports approximately 100 million files/blocks
- Maximum practical limit with a single NameNode: ~500 million to 1 billion objects

**Tuning NameNode heap:**
```bash
# hadoop-env.sh
export HADOOP_NAMENODE_OPTS="-Xmx128g -Xms128g"
# Set Xmx = Xms to avoid heap resizing pauses
```

---

## Q22. What happens during NameNode recovery? Walk through the startup sequence.

**Answer:**

When a NameNode starts (either fresh start or after crash), it goes through a careful recovery sequence:

### Phase 1 — Load fsimage into memory
```
NameNode reads /dfs/name/current/fsimage_XXXX from local disk
Loads the entire file system snapshot into Java heap
For a large cluster (100M files): this can take 5–30 minutes
Memory: fsimage_XXXX → in-memory INode tree
```

### Phase 2 — Replay edit log
```
NameNode reads all edit log segments since the fsimage checkpoint
Replays each transaction on the in-memory tree:
  edits_0000001-0000500 → replay 500 transactions
  edits_0000501-0001000 → replay 500 transactions
  edits_inprogress_0001001 → replay in-progress transactions
Result: in-memory tree is fully up-to-date
```

### Phase 3 — Enter Safe Mode
```
NameNode enters SAFE MODE automatically:
  - File system is read-only
  - No block allocations, no deletions
  - Waits for DataNodes to check in and report their blocks
```

### Phase 4 — DataNode block reports
```
Each DataNode sends a FULL block report to NameNode:
  "I have blocks: [blk_1001, blk_1002, blk_1007, ...]"
NameNode reconciles:
  - Which blocks are under-replicated?
  - Which blocks have too many replicas?
  - Which blocks are corrupt?
```

### Phase 5 — Exit Safe Mode
```
Safe Mode exit condition:
  - dfs.namenode.safemode.threshold-pct = 0.999 (default)
  - 99.9% of expected blocks reported as available
  - Minimum replication factor met for most blocks

After exit:
  - Cluster goes LIVE
  - Writes are allowed
  - Under-replicated blocks are scheduled for re-replication
```

### Manual Safe Mode Commands
```bash
# Check safe mode status
hdfs dfsadmin -safemode get

# Force enter safe mode (for maintenance)
hdfs dfsadmin -safemode enter

# Force exit safe mode
hdfs dfsadmin -safemode leave

# Wait until safe mode exits (useful in scripts)
hdfs dfsadmin -safemode wait
```

---

## Q23. What is HDFS Erasure Coding? How does it differ from Replication?

**Answer:**

**The Problem with 3x Replication:**
```
1 PB of data × 3 replicas = 3 PB storage needed
Storage overhead: 200%
```

**Erasure Coding (EC)** is an alternative to replication that provides similar fault tolerance with **much less storage overhead** (typically 50% overhead instead of 200%).

**How EC Works (Reed-Solomon algorithm):**

EC divides data into **data blocks** and **parity blocks**. The parity blocks are computed from the data blocks mathematically. Lost data blocks can be **reconstructed** from the remaining blocks.

**Example: RS-6-3-1024k policy**
```
Data blocks: 6
Parity blocks: 3
Stripe width: 9 blocks total

Original data: split into 6 data cells
Parity = mathematical function of the 6 data cells

Storage: 9 blocks for 6 blocks of data
Overhead: 3/6 = 50% (vs 200% for 3x replication)

Fault tolerance: can lose any 3 of the 9 blocks and still reconstruct data
```

**Visual Comparison:**

```
Replication (3x):
  Data: [A][A][A] [B][B][B] [C][C][C]
  Stores 9 block-equivalents for 3 blocks of data
  Overhead: 200%

Erasure Coding (RS-6-3):
  Data:   [D1][D2][D3][D4][D5][D6]
  Parity: [P1][P2][P3]
  Stores 9 block-equivalents for 6 blocks of data
  Overhead: 50%
```

**EC Commands in HDFS:**
```bash
# Check available EC policies
hdfs ec -listPolicies

# Enable EC policy on a directory
hdfs ec -setPolicy -policy RS-6-3-1024k -path /data/cold_storage/

# Verify EC policy on a path
hdfs ec -getPolicy -path /data/cold_storage/

# Convert replication to EC (requires cluster reconfiguration)
hdfs ec -enablePolicy -policy RS-6-3-1024k
```

**When to Use Each:**

| Use Case | Recommendation |
|----------|---------------|
| Hot data (frequently accessed) | Replication — lower read latency |
| Cold/archive data (rarely accessed) | Erasure Coding — huge storage savings |
| Small files | Replication — EC has high CPU overhead for small data |
| Large files (> 1 GB) | Erasure Coding — cost savings justify CPU overhead |
| Latency-sensitive reads | Replication — EC reconstruction adds latency |

**Trade-off:**
```
EC cons:
  - Higher CPU overhead (parity computation on write, reconstruction on partial failure)
  - Higher read latency when a block is missing (must reconstruct)
  - Requires minimum cluster size (at least 9 DataNodes for RS-6-3)
  - Not suitable for small files
```

---

## Q24. Explain Hadoop's Capacity Scheduler and Fair Scheduler. When would you use each?

**Answer:**

Both are YARN schedulers that decide **which applications get cluster resources** when multiple jobs are running simultaneously.

### Capacity Scheduler (Default in most distributions)

**Concept:** The cluster is divided into **queues**, each guaranteed a minimum percentage of total resources. Each queue can use more resources when available (borrowed from idle queues).

```
Total Cluster: 1000 containers

Queue Configuration:
  engineering  → 50% guaranteed (500 containers)
    ├── spark      → 30% of total (300 containers)
    └── mapreduce  → 20% of total (200 containers)
  analytics    → 30% guaranteed (300 containers)
  adhoc        → 20% guaranteed (200 containers)

If analytics is idle:
  engineering can borrow up to 80% of total cluster resources
```

**Configuration (capacity-scheduler.xml):**
```xml
<property>
  <name>yarn.scheduler.capacity.root.queues</name>
  <value>engineering,analytics,adhoc</value>
</property>
<property>
  <name>yarn.scheduler.capacity.root.engineering.capacity</name>
  <value>50</value>
</property>
<property>
  <name>yarn.scheduler.capacity.root.engineering.maximum-capacity</name>
  <value>80</value>  <!-- Can borrow up to 80% -->
</property>
```

### Fair Scheduler

**Concept:** All applications share cluster resources **equally** over time. No fixed queues are required. Each application gets a fair share — if one job is running alone, it gets 100%; when a second job starts, each gets 50%.

```
Time 0: Job A alone       → A gets 100%
Time 5: Job B submitted   → A gets 50%, B gets 50%
Time 10: Job A finishes   → B gets 100%

No queue configuration needed for simple cases
```

**When to Use Each:**

| Criteria | Capacity Scheduler | Fair Scheduler |
|----------|--------------------|----------------|
| Multi-tenant cluster with SLA guarantees | ✅ Preferred | ❌ No guarantee |
| Org with multiple teams | ✅ Queue per team | ✅ Pool per team |
| Interactive/ad-hoc queries | ❌ May wait | ✅ Gets resources quickly |
| Preemption needed | ✅ Supported | ✅ Supported |
| Simple setup | ❌ Complex XML config | ✅ Simpler |
| Enterprise (CDH/HDP) | ✅ Default | Available |

---

## Q25. What is the impact of GC (Garbage Collection) on NameNode performance? How do you tune it?

**Answer:**

The NameNode JVM heap can grow to 128+ GB. Standard JVM Garbage Collection on heaps this large causes **Stop-the-World (STW) pauses** that freeze the NameNode for seconds to minutes — during which DataNode heartbeats are not acknowledged, triggering false DataNode dead detection and unnecessary re-replication.

### The Problem
```
Large NameNode heap: 128 GB
Full GC event:
  STW pause = 30–120 seconds
  During this time:
    - DataNodes see no heartbeat response
    - DataNode timeout: 10 minutes (default)
    - But frequent GC can cause cascading timeouts
    - MapReduce/Spark jobs fail waiting for NameNode RPC
```

### Solution — Use G1GC (Garbage First GC)

G1GC is designed for large heaps — it collects garbage **incrementally** and keeps STW pauses under a target threshold (e.g., 200ms).

```bash
# hadoop-env.sh — NameNode JVM options
export HADOOP_NAMENODE_OPTS="
  -Xmx128g
  -Xms128g
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:G1HeapRegionSize=32m
  -XX:InitiatingHeapOccupancyPercent=35
  -XX:+ParallelRefProcEnabled
  -XX:+PrintGCDetails
  -XX:+PrintGCDateStamps
  -Xloggc:/var/log/hadoop/namenode-gc.log
"
```

### Key JVM Parameters Explained

| Parameter | Purpose |
|-----------|---------|
| `-XX:+UseG1GC` | Enable G1 garbage collector |
| `-XX:MaxGCPauseMillis=200` | Target max STW pause of 200ms |
| `-XX:G1HeapRegionSize=32m` | Larger regions for large heap |
| `-XX:InitiatingHeapOccupancyPercent=35` | Start GC earlier to avoid full GC |
| `-XX:+ParallelRefProcEnabled` | Process references in parallel |

### Monitoring GC
```bash
# Check GC activity in real time
jstat -gcutil <namenode_pid> 1000 100

# Parse GC logs for pause times
grep "GC pause" /var/log/hadoop/namenode-gc.log | awk '{print $NF}'

# HDFS admin — check NameNode RPC call queue length (GC proxy indicator)
hdfs dfsadmin -report | grep "RPC"
```

---

---

# PART 4 — SCENARIO-BASED DAY-TO-DAY HDFS COMMANDS

---

## Scenario 1 — Navigating and Exploring HDFS

```bash
# List files and directories in HDFS root
hdfs dfs -ls /

# List files in a specific directory (with human-readable sizes)
hdfs dfs -ls /user/hadoop/data/

# List files recursively in all subdirectories
hdfs dfs -ls -R /user/hadoop/

# List with human-readable file sizes
hdfs dfs -ls -h /data/

# Check disk usage of a directory (human readable)
hdfs dfs -du -h /user/hadoop/data/

# Check total disk usage summary of a directory
hdfs dfs -du -s -h /user/hadoop/data/

# Count files, directories, and bytes in a path
hdfs dfs -count /user/hadoop/data/
# Output: DIR_COUNT  FILE_COUNT  CONTENT_SIZE  PATHNAME

# Show disk usage across the cluster
hdfs dfsadmin -report
```

---

## Scenario 2 — Uploading Data to HDFS

```bash
# Upload a local file to HDFS
hdfs dfs -put /local/path/sales.csv /user/hadoop/data/

# Upload with specific replication factor
hdfs dfs -D dfs.replication=2 -put /local/path/sales.csv /user/hadoop/data/

# Upload entire directory from local to HDFS
hdfs dfs -put /local/raw_data/ /user/hadoop/raw_data/

# copyFromLocal — same as put (explicit name)
hdfs dfs -copyFromLocal /local/path/file.csv /hdfs/path/

# moveFromLocal — moves file (deletes local after upload)
hdfs dfs -moveFromLocal /local/path/file.csv /hdfs/path/

# Upload and overwrite if file already exists
hdfs dfs -put -f /local/path/sales.csv /user/hadoop/data/

# Append to an existing HDFS file
hdfs dfs -appendToFile /local/new_records.csv /user/hadoop/data/sales.csv
```

---

## Scenario 3 — Downloading Data from HDFS

```bash
# Download a file from HDFS to local filesystem
hdfs dfs -get /user/hadoop/data/sales.csv /local/downloads/

# copyToLocal — same as get
hdfs dfs -copyToLocal /user/hadoop/data/sales.csv /local/downloads/

# Merge multiple HDFS files into one local file
# Very useful for MapReduce output (many part-00000 files)
hdfs dfs -getmerge /user/hadoop/output/ /local/final_output.csv

# getmerge with newline separator between files
hdfs dfs -getmerge -nl /user/hadoop/output/ /local/final_output.csv

# Download and immediately process (pipe to command)
hdfs dfs -cat /user/hadoop/data/config.json | python process.py
```

---

## Scenario 4 — Reading and Inspecting Files

```bash
# Print contents of a file to stdout
hdfs dfs -cat /user/hadoop/data/sales.csv

# View first N lines of a file (like head)
hdfs dfs -cat /user/hadoop/data/sales.csv | head -20

# View last N lines (like tail)
hdfs dfs -cat /user/hadoop/data/sales.csv | tail -20

# Read a compressed file directly
hdfs dfs -cat /user/hadoop/data/sales.csv.gz | zcat | head -20

# View a Parquet file (requires parquet-tools)
hdfs dfs -get /user/hadoop/data/sales.parquet /tmp/
parquet-tools cat /tmp/sales.parquet | head -20

# Text command — handles compressed files automatically
hdfs dfs -text /user/hadoop/data/sales.csv.gz | head -20

# Check file properties (size, replication, block size)
hdfs dfs -stat "%n %r %b %o" /user/hadoop/data/sales.csv
# %n = name, %r = replication, %b = size in bytes, %o = block size
```

---

## Scenario 5 — File and Directory Management

```bash
# Create a directory
hdfs dfs -mkdir /user/hadoop/new_project/

# Create directory and all parent directories (like mkdir -p)
hdfs dfs -mkdir -p /user/hadoop/projects/2024/q1/raw/

# Copy a file within HDFS (does not leave HDFS)
hdfs dfs -cp /user/hadoop/data/sales.csv /user/hadoop/backup/sales_backup.csv

# Move/rename a file within HDFS
hdfs dfs -mv /user/hadoop/data/old_name.csv /user/hadoop/data/new_name.csv

# Move entire directory
hdfs dfs -mv /user/hadoop/data/staging/ /user/hadoop/data/processed/

# Delete a file
hdfs dfs -rm /user/hadoop/data/temp_file.csv

# Delete a directory and all contents (recursive)
hdfs dfs -rm -r /user/hadoop/data/old_project/

# Skip trash when deleting (permanent delete — use carefully)
hdfs dfs -rm -r -skipTrash /user/hadoop/data/large_temp/

# Empty the HDFS trash (free up space used by deleted files)
hdfs dfs -expunge
```

---

## Scenario 6 — Permissions and Ownership

```bash
# Check permissions on a directory
hdfs dfs -ls /user/hadoop/data/
# Output format: permissions  replication  owner  group  size  date  path
# -rw-r--r--   3  hadoop  supergroup  1234567  2024-01-15 09:30  /user/hadoop/data/sales.csv

# Change permissions (same syntax as Linux chmod)
hdfs dfs -chmod 755 /user/hadoop/data/

# Change permissions recursively
hdfs dfs -chmod -R 755 /user/hadoop/data/

# Change owner
hdfs dfs -chown analyst /user/hadoop/data/sales.csv

# Change owner and group
hdfs dfs -chown analyst:data_team /user/hadoop/data/sales.csv

# Change owner recursively
hdfs dfs -chown -R analyst:data_team /user/hadoop/data/

# Change group
hdfs dfs -chgrp data_team /user/hadoop/data/

# Set Access Control List (ACL) — fine-grained permissions
hdfs dfs -setfacl -m user:bob:rwx /user/hadoop/data/
hdfs dfs -setfacl -m group:analysts:r-x /user/hadoop/data/

# Get ACL of a path
hdfs dfs -getfacl /user/hadoop/data/
```

---

## Scenario 7 — Replication Management

```bash
# Check replication factor and block locations for a file
hdfs fsck /user/hadoop/data/sales.csv -files -blocks -locations

# Change replication factor for a specific file
hdfs dfs -setrep 2 /user/hadoop/data/sales.csv

# Change replication factor for entire directory (recursive)
hdfs dfs -setrep -R 2 /user/hadoop/data/

# Wait for replication to complete before returning
hdfs dfs -setrep -w 3 /user/hadoop/data/important_file.csv

# Check current replication of a file
hdfs dfs -stat %r /user/hadoop/data/sales.csv
# Output: 3

# List under-replicated blocks across the cluster
hdfs dfsadmin -report | grep "Under replicated"

# Full HDFS health check — shows missing blocks, corrupt blocks
hdfs fsck / -summary
```

---

## Scenario 8 — Health Checks and Troubleshooting

```bash
# Full HDFS filesystem check
hdfs fsck /

# Check specific directory for corruption
hdfs fsck /user/hadoop/data/ -files -blocks -locations

# Show only corrupt files
hdfs fsck / -list-corruptfileblocks

# Delete corrupt files (use with caution)
hdfs fsck / -delete

# Move corrupt files to /lost+found
hdfs fsck / -move

# Show cluster summary report
hdfs dfsadmin -report

# Show live DataNodes only
hdfs dfsadmin -report -live

# Show dead DataNodes only
hdfs dfsadmin -report -dead

# Refresh NameNode's host list (after adding/removing nodes)
hdfs dfsadmin -refreshNodes

# Trigger a checkpoint (compaction of fsimage + editlog)
hdfs dfsadmin -saveNamespace

# Enter safe mode
hdfs dfsadmin -safemode enter

# Leave safe mode
hdfs dfsadmin -safemode leave

# Check safe mode status
hdfs dfsadmin -safemode get

# Show NameNode status (web UI equivalent in CLI)
hdfs dfsadmin -metasave /tmp/meta_dump.txt
```

---

## Scenario 9 — Working with Snapshots

Snapshots capture the state of a directory at a point in time. Useful for backups and rollbacks.

```bash
# Enable snapshot capability on a directory
hdfs dfsadmin -allowSnapshot /user/hadoop/data/

# Create a snapshot
hdfs dfs -createSnapshot /user/hadoop/data/ snapshot_2024_01_15

# List snapshots of a directory
hdfs dfs -listSnapshots /user/hadoop/data/

# Compare differences between two snapshots
hdfs snapshotDiff /user/hadoop/data/ snapshot_2024_01_15 snapshot_2024_01_16

# Access files in a snapshot (read-only)
hdfs dfs -ls /user/hadoop/data/.snapshot/snapshot_2024_01_15/

# Restore a file from snapshot
hdfs dfs -cp /user/hadoop/data/.snapshot/snapshot_2024_01_15/sales.csv \
            /user/hadoop/data/sales_restored.csv

# Delete a snapshot
hdfs dfs -deleteSnapshot /user/hadoop/data/ snapshot_2024_01_15

# Disable snapshots on a directory
hdfs dfsadmin -disallowSnapshot /user/hadoop/data/
```

---

## Scenario 10 — Balancing the Cluster

Over time, DataNodes can become unbalanced (some full, some nearly empty) due to:
- New DataNodes being added
- Rack-unaware data writes
- Skewed data distributions

```bash
# Start the HDFS balancer
hdfs balancer

# Run balancer with custom threshold (default 10% imbalance allowed)
hdfs balancer -threshold 5

# Run balancer with bandwidth limit (bytes/sec per DataNode)
hdfs balancer -bandwidth 52428800  # 50 MB/s

# Run balancer and specify specific DataNodes to balance
hdfs balancer -include /etc/hadoop/nodes_to_balance.txt

# Run in background
nohup hdfs balancer -threshold 5 > /var/log/hadoop/balancer.log 2>&1 &

# Check balancer progress
tail -f /var/log/hadoop/balancer.log

# Set balancer bandwidth dynamically (without restart)
hdfs dfsadmin -setBalancerBandwidth 52428800  # 50 MB/s
```

---

## Scenario 11 — Monitoring Space and Quotas

```bash
# Check total HDFS capacity, used, and remaining space
hdfs dfs -df -h /

# Check disk usage of specific directories
hdfs dfs -du -h /user/

# Set a space quota on a directory (bytes)
hdfs dfsadmin -setSpaceQuota 107374182400 /user/team_analytics/
# 107374182400 bytes = 100 GB

# Set a namespace quota (max number of files + directories)
hdfs dfsadmin -setQuota 100000 /user/team_analytics/

# Check quotas on a directory
hdfs dfs -count -q -h /user/team_analytics/
# Output: QUOTA  REM_QUOTA  SPACE_QUOTA  REM_SPACE_QUOTA  DIR_COUNT  FILE_COUNT  SIZE  PATH

# Clear space quota
hdfs dfsadmin -clrSpaceQuota /user/team_analytics/

# Clear namespace quota
hdfs dfsadmin -clrQuota /user/team_analytics/
```

---

## Scenario 12 — Real-World Pipeline Operations

### Check if a file exists before processing (in shell scripts)
```bash
# Check file existence — exit code 0 if exists, 1 if not
if hdfs dfs -test -e /user/hadoop/data/daily_load_2024_01_15.csv; then
  echo "File exists — proceeding with processing"
else
  echo "File missing — skipping today's job"
  exit 1
fi

# Check if path is a directory
hdfs dfs -test -d /user/hadoop/data/

# Check if file is empty
hdfs dfs -test -z /user/hadoop/data/daily_load.csv
```

### Monitor pipeline output directory
```bash
# Wait for all MapReduce/Spark output files to appear
OUTPUT_DIR="/user/hadoop/output/job_20240115/"
EXPECTED_FILES=10

while true; do
  FILE_COUNT=$(hdfs dfs -ls $OUTPUT_DIR | grep "part-" | wc -l)
  echo "Found $FILE_COUNT / $EXPECTED_FILES output files"
  if [ "$FILE_COUNT" -ge "$EXPECTED_FILES" ]; then
    echo "Job complete"
    break
  fi
  sleep 30
done
```

### Clean up old pipeline data (retention policy)
```bash
#!/bin/bash
# Delete HDFS data older than 30 days
# List directories with their dates
RETENTION_DAYS=30
BASE_PATH="/user/hadoop/data/daily/"

hdfs dfs -ls $BASE_PATH | grep "^d" | while read -r line; do
  DIR_DATE=$(echo $line | awk '{print $6}')
  DIR_PATH=$(echo $line | awk '{print $8}')
  DIR_EPOCH=$(date -d "$DIR_DATE" +%s)
  CUTOFF_EPOCH=$(date -d "$RETENTION_DAYS days ago" +%s)
  
  if [ "$DIR_EPOCH" -lt "$CUTOFF_EPOCH" ]; then
    echo "Deleting old directory: $DIR_PATH"
    hdfs dfs -rm -r -skipTrash "$DIR_PATH"
  fi
done
```

### Move data between clusters using DistCp
```bash
# DistCp — Distributed Copy — uses MapReduce to copy large amounts of data in parallel

# Copy from one cluster to another
hadoop distcp hdfs://cluster1:8020/data/sales/ hdfs://cluster2:8020/data/sales/

# Copy from HDFS to S3
hadoop distcp hdfs:///user/hadoop/data/ s3a://my-bucket/hadoop-data/

# Copy with specific number of maps (parallel copy threads)
hadoop distcp -m 20 hdfs:///source/data/ hdfs:///dest/data/

# Update mode — only copy files that differ or are missing
hadoop distcp -update hdfs:///source/ hdfs:///dest/

# Overwrite existing files at destination
hadoop distcp -overwrite hdfs:///source/ hdfs:///dest/

# Skip CRC validation (faster for large transfers)
hadoop distcp -skipcrccheck hdfs:///source/ hdfs:///dest/

# Bandwidth limit per map (bytes/sec)
hadoop distcp -bandwidth 100 hdfs:///source/ hdfs:///dest/
# 100 MB/s per mapper
```

---

## Scenario 13 — Archiving and Compression

```bash
# Create a Hadoop Archive (HAR) — bundles many small files
hadoop archive \
  -archiveName project_data.har \
  -p /user/hadoop/small_files/ \
  /user/hadoop/archives/

# List contents of a HAR archive
hdfs dfs -ls har:///user/hadoop/archives/project_data.har/

# Read a file from inside a HAR archive
hdfs dfs -cat har:///user/hadoop/archives/project_data.har/file1.csv

# Compress a file using codec
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-*.jar \
  -input /user/hadoop/data/large.csv \
  -output /user/hadoop/data/large.csv.gz \
  -mapper "gzip -c" \
  -reducer NONE

# Check compression type of a file
file /tmp/downloaded_file
# Or check magic bytes
hdfs dfs -cat /user/hadoop/data/file.gz | file -
```

---

## Quick Reference — Most Used HDFS Commands

```
EXPLORE:
  hdfs dfs -ls -h /path/          List files (human readable)
  hdfs dfs -du -s -h /path/       Total size of directory
  hdfs dfs -count /path/          Count files + dirs + bytes

TRANSFER:
  hdfs dfs -put /local /hdfs/     Upload file
  hdfs dfs -get /hdfs/ /local/    Download file
  hdfs dfs -getmerge /hdfs/ /out  Merge many files to one
  hadoop distcp /src /dest        Parallel cluster-to-cluster copy

MANAGE:
  hdfs dfs -mkdir -p /path/       Create directory
  hdfs dfs -cp /src /dest         Copy within HDFS
  hdfs dfs -mv /src /dest         Move within HDFS
  hdfs dfs -rm -r /path/          Delete recursively
  hdfs dfs -chmod -R 755 /path/   Change permissions

INSPECT:
  hdfs fsck /path/ -files -blocks Check file health + block locations
  hdfs dfsadmin -report           Cluster health summary
  hdfs dfsadmin -safemode get     Check safe mode status
  hdfs dfs -setrep -R 3 /path/    Change replication factor

QUOTAS:
  hdfs dfsadmin -setSpaceQuota N /path/   Set max bytes
  hdfs dfs -count -q -h /path/           Check quota usage
```

---

*Reference Guide: HDFS & Hadoop Interview Preparation*
*Levels: Basic | Medium | Hard | Scenario-Based Commands*
*Total Questions: 25 | Command Scenarios: 13*
