# 🐘 Hadoop Interview Questions — Complete Master Guide

> **Covers:** Fundamentals · Intermediate Concepts · Important Commands · Scenario-Based · FAANG/MAANG Level  
> **Philosophy:** Every explanation in the simplest way possible. Every solution is optimal.

---

## 📑 Table of Contents

| # | Section |
|---|---------|
| 1 | [Fundamentals](#-section-1-fundamentals) |
| 2 | [Intermediate Concepts](#-section-2-intermediate-concepts) |
| 3 | [Important Commands Asked in Interviews](#-section-3-important-commands-asked-in-interviews) |
| 4 | [Scenario-Based Questions](#-section-4-scenario-based-questions) |
| 5 | [FAANG / MAANG Level Questions](#-section-5-faangmaang-level-questions) |
| 6 | [Quick Reference Cheat Sheet](#-quick-reference-cheat-sheet) |

---

## 📘 Section 1: Fundamentals

---

### Q1. What is Hadoop? Why was it created?

**Answer:**  
Hadoop is an **open-source framework** that lets you store and process massive amounts of data across many cheap machines (commodity hardware) in a distributed, parallel manner.

**Simple analogy:**  
Imagine you have a 10,000-page book to read in one hour. Instead of one person reading it all, you split pages across 100 people — they each read 100 pages and report back. Hadoop works exactly like this for data.

**Why it was created:**  
In the early 2000s, Google faced a problem — data was growing faster than any single machine could handle. They published two papers:
- **GFS (Google File System, 2003)** → Inspired **HDFS**
- **MapReduce (2004)** → Inspired **Hadoop MapReduce**

Yahoo's Doug Cutting implemented these ideas as open-source Hadoop (named after his son's toy elephant).

---

### Q2. What are the four core components of Hadoop?

**Answer:**

| Component | What it does | Simple analogy |
|-----------|-------------|----------------|
| **HDFS** | Stores data across machines in blocks | A giant distributed hard drive |
| **MapReduce** | Processes data in parallel across machines | Assembly line — split work, combine results |
| **YARN** | Manages CPU & memory resources | Airport control tower — directs all traffic |
| **Hadoop Common** | Shared utilities and libraries | Shared toolbox everyone uses |

---

### Q3. What is HDFS and how does it work?

**Answer:**  
HDFS (Hadoop Distributed File System) is Hadoop's **storage layer**. It splits large files into fixed-size **blocks** and distributes those blocks across multiple machines.

**Key components:**

| Component | Role |
|-----------|------|
| **NameNode** | Master — stores metadata (which block is where). Does NOT store actual data. |
| **DataNode** | Worker — stores actual data blocks. |
| **Secondary NameNode** | NOT a backup! Periodically merges edit logs to keep NameNode healthy. |

**Simple analogy:**  
- NameNode = Library catalog (tells you which shelf a book is on)  
- DataNode = Actual bookshelves holding the books

**How a 500 MB file gets stored:**
```
File: sales_2024.csv (500 MB)
  ├── Block 1 (128 MB) → copies on DataNode 1, DataNode 3, DataNode 5
  ├── Block 2 (128 MB) → copies on DataNode 2, DataNode 4, DataNode 6
  ├── Block 3 (128 MB) → copies on DataNode 1, DataNode 2, DataNode 4
  └── Block 4 (116 MB) → copies on DataNode 3, DataNode 5, DataNode 6
                                          ↑
                              3 copies = replication factor 3
```

---

### Q4. What is the default block size in HDFS and why is it 128 MB?

**Answer:**  
Default block size = **128 MB** (64 MB in older versions, Hadoop 1.x).

**Why so large?**
- Fewer metadata entries for NameNode to track → less RAM usage
- Sequential reads are faster — one 128 MB read vs 1000 small reads
- Reduces disk seek time dramatically
- Designed for **batch processing** (throughput matters, not latency)

**Trade-off:**  
If you store a 1 KB file, it still occupies one full block's worth of metadata in the NameNode — very wasteful. This is the root cause of the **small files problem**.

> You can change block size per file: `hadoop fs -D dfs.blocksize=67108864 -put file.txt /data/`

---

### Q5. What is Replication in HDFS?

**Answer:**  
Every block is stored **3 times by default** (replication factor = 3) on different machines.

**Why 3?**
- **Fault tolerance** — if 2 machines crash simultaneously, data still exists on the 3rd
- **Read performance** — HDFS can serve reads from the nearest replica
- **Data locality** — run computation on the same machine that holds the data

**Rack awareness (smart placement):**
```
Replica 1 → Node on Rack A  (same node that wrote the data)
Replica 2 → Node on Rack B  (different rack — survives rack failure)
Replica 3 → Another node on Rack B  (bandwidth optimization)
```
This means even if an entire rack's switch fails, your data is safe on Rack A.

---

### Q6. What is MapReduce? Explain with an example.

**Answer:**  
MapReduce is a **programming model** for processing huge datasets by breaking work into two simple phases:

1. **Map** — Each mapper works independently on its chunk of data → outputs key-value pairs
2. **Reduce** — Aggregates/combines all mapper outputs with the same key

**Word count example (the "Hello World" of Hadoop):**
```
Input text: "cat dog cat bird dog cat"

── MAP PHASE ─────────────────────────────────────
  Mapper 1 (processes "cat dog cat"):
      emits: (cat,1)  (dog,1)  (cat,1)

  Mapper 2 (processes "bird dog cat"):
      emits: (bird,1)  (dog,1)  (cat,1)

── SHUFFLE & SORT (Hadoop groups same keys) ──────
      bird  →  [1]
      cat   →  [1, 1, 1]
      dog   →  [1, 1]

── REDUCE PHASE ──────────────────────────────────
  Reducer: sums the list for each key
      bird  →  1
      cat   →  3
      dog   →  2

Output: bird:1, cat:3, dog:2
```

---

### Q7. What is YARN and why was it introduced?

**Answer:**  
YARN = **Yet Another Resource Negotiator** (Hadoop 2.x+)

**The problem with Hadoop 1.x:**  
Everything was managed by a single **JobTracker** — it scheduled jobs AND managed resources. This was a bottleneck and a single point of failure. Also, only MapReduce jobs could run on the cluster.

**YARN's solution — split responsibilities:**

| Component | Role |
|-----------|------|
| **ResourceManager** | Cluster master — knows total available CPU & memory |
| **NodeManager** | Per-machine agent — manages containers on that node |
| **ApplicationMaster** | Per-job brain — negotiates resources & monitors tasks |

**Big benefit:** YARN is a general-purpose resource manager. **Spark, Flink, Tez, HBase** all run on YARN — not just MapReduce.

```
Think of YARN like an OS:
  ResourceManager = OS kernel (allocates resources)
  NodeManager     = Process manager per machine
  ApplicationMaster = Each app manages itself
```

---

### Q8. What is the difference between NameNode and Secondary NameNode?

| Feature | NameNode | Secondary NameNode |
|---------|----------|--------------------|
| Role | Cluster master — metadata server | Checkpoint helper only |
| Stores actual data? | ❌ No | ❌ No |
| Is it a backup/failover? | — | ❌ **NO — common misconception!** |
| What it does | Keeps fsimage + edits log in RAM | Merges fsimage + edits log periodically |
| SPOF (Single Point of Failure)? | Yes (Hadoop 1.x) | No |

> ⚠️ **Interview trap:** "Secondary NameNode" sounds like a backup — it is NOT. If NameNode crashes, Secondary NameNode cannot take over. For real HA, use **Standby NameNode** with ZooKeeper (Hadoop 2.x+).

---

### Q9. What is the "small files problem" in Hadoop?

**Answer:**  
HDFS is built for **large files**. But every file — no matter how small — consumes **~150 bytes of NameNode RAM** for its metadata.

**Why this is a problem:**
```
Scenario: 10 million files of 1 KB each
  Total actual data     = 10 GB  (tiny!)
  NameNode memory used  = 10,000,000 × 150 bytes = ~1.5 GB just for metadata
  
  If NameNode has 32 GB RAM → max ~200 million files → NameNode becomes bottleneck
```

**Fixes:**

| Solution | How it helps |
|----------|-------------|
| **HAR files** (`hadoop archive`) | Bundles many small files into one archive file |
| **SequenceFile** | Stores key-value pairs — small files become values in one big file |
| **CombineFileInputFormat** | Combines multiple small files into a single InputSplit for MapReduce |
| **Spark coalesce()** | Merges many small partitions into fewer, larger ones |
| **Hive bucketing** | Groups data into fixed buckets — prevents file explosion |

---

### Q10. Difference between HDFS and a traditional file system?

| Feature | HDFS | Traditional FS (ext4, NTFS) |
|---------|------|------------------------------|
| Scale | Petabytes across 1000s of nodes | Terabytes on one machine |
| Hardware | Cheap commodity machines | Expensive high-end servers |
| Access pattern | Write-once, read-many (batch) | Random read/write |
| Latency | High (minutes) | Low (milliseconds) |
| Fault tolerance | Built-in via block replication | RAID or external backup |
| Data locality | Yes — compute moves to data | Not applicable |
| Modification | Append-only (Hadoop 2.x) | Full in-place edit |
| Best for | Analytics, ETL, batch processing | Transactional, interactive apps |

---

## 📗 Section 2: Intermediate Concepts

---

### Q11. What are the three Hadoop run modes?

| Mode | Description | Use Case |
|------|-------------|----------|
| **Standalone** | Single JVM, no HDFS, uses local filesystem | Unit testing, debugging |
| **Pseudo-distributed** | All daemons on one machine, HDFS enabled | Development/learning |
| **Fully distributed** | Multiple machines, full cluster | Production |

---

### Q12. What is a Combiner and when should you use it?

**Answer:**  
A Combiner is a **mini-reducer that runs locally on each mapper's machine** before data is sent across the network to the actual reducer.

**Goal:** Reduce the volume of data shuffled over the network.

**Word Count example:**
```
WITHOUT Combiner:
  Mapper output → network → Reducer
  (cat,1)(cat,1)(cat,1)(dog,1)(dog,1)  ← 5 pairs sent over wire

WITH Combiner (local aggregation first):
  Mapper output → Combiner → network → Reducer
  (cat,3)(dog,2)  ← only 2 pairs sent over wire
  
  = 60% less network traffic!
```

**When can you use a Combiner?**  
Only when the operation is **commutative** (a+b = b+a) and **associative** (a+(b+c) = (a+b)+c):
- ✅ Sum, Count, Max, Min
- ❌ Average (you need total count + total sum separately — can't combine averages)

---

### Q13. Explain the Shuffle and Sort phase in detail.

**Answer:**  
Shuffle & Sort is the **hidden phase between Map and Reduce** — it's what Hadoop does automatically. It's also usually the **biggest performance bottleneck**.

**Step-by-step:**
```
Step 1: PARTITION
  Each mapper decides which reducer gets which key
  (Default: hash(key) % numReducers)
  key="cat" → goes to Reducer 2
  key="dog" → goes to Reducer 0

Step 2: SPILL (Map-side)
  Mapper writes sorted output to a circular in-memory buffer (100 MB default)
  When buffer hits 80% → spills to local disk as a sorted file
  Multiple spill files are merged into one sorted file per mapper

Step 3: SHUFFLE (network transfer)
  Each reducer pulls (fetches) its partition of data from ALL mappers
  This is the only step that uses the network

Step 4: MERGE + SORT (Reduce-side)
  Reducer merges all fetched files into one sorted stream
  Groups records with the same key together
  Passes to the reduce() function
```

---

### Q14. What is an InputSplit?

**Answer:**  
An InputSplit is a **logical chunk of data** assigned to one Mapper.

```
Physical reality:       HDFS Block (128 MB, actual bytes on disk)
Logical view for job:   InputSplit (what one Mapper will process)

Usually: 1 HDFS Block = 1 InputSplit = 1 Mapper
But with CombineFileInputFormat: many small files → 1 InputSplit → 1 Mapper
```

**Number of mappers = Number of InputSplits** — this is how Hadoop determines parallelism.

---

### Q15. What is Speculative Execution?

**Answer:**  
In a large cluster, some machines are slow (hardware wear, network issues, GC pauses). One slow task can delay an entire job.

**Speculative execution = launch a duplicate of slow tasks on other machines. Use whichever finishes first. Kill the other.**

```
Normal:           Task A (fast node) ████████ done in 2 min
                  Task B (slow node) ████████████████████ 10 min ← blocks job

With Speculative: Task B (slow node) ████████████████████ 10 min
                  Task B' (new node) ████████ done in 2.5 min ← used! B killed.
                  Job finishes in 2.5 min instead of 10 min
```

**Enable in config:**
```xml
<property><name>mapreduce.map.speculative</name><value>true</value></property>
<property><name>mapreduce.reduce.speculative</name><value>true</value></property>
```

**Disable when:** tasks have side effects (writing to databases) — duplicate tasks = duplicate writes.

---

### Q16. What are the different file formats in Hadoop?

| Format | Type | Splittable | Compression | Schema | Best For |
|--------|------|-----------|-------------|--------|----------|
| **Text/CSV** | Row | Yes | Slow | No | Simple, interoperability |
| **SequenceFile** | Row | Yes | Yes | No | MapReduce intermediate data |
| **Avro** | Row | Yes | Yes | Yes (JSON) | Streaming, schema evolution |
| **ORC** | Columnar | Yes | Excellent | Yes | Hive, analytics queries |
| **Parquet** | Columnar | Yes | Excellent | Yes | Spark, Impala, general analytics |

**Why columnar (ORC/Parquet) is faster for analytics:**
```
Table has 100 columns. Your query: SELECT revenue, date FROM sales WHERE date > '2024-01-01'

Row format:   Reads ALL 100 columns per row just to get 2 → wastes 98x I/O
Columnar:     Reads ONLY revenue + date columns → 98% less I/O
              + predicate pushdown skips entire row groups where date is old
```

**Rule of thumb:** Use **ORC** with Hive, **Parquet** with Spark.

---

### Q17. What is Hadoop High Availability (HA)?

**Answer:**  
In Hadoop 1.x, NameNode was a **Single Point of Failure**. If it crashed → entire cluster offline.

**HA solution (Hadoop 2.x+):** Two NameNodes running simultaneously.

```
Active NameNode   ←── handles all reads/writes
Standby NameNode  ←── in sync, ready to take over in seconds

How they stay in sync:
  ┌──────────────────────────────────────────┐
  │            JournalNodes (3+)             │
  │  (quorum-based shared edit log storage)  │
  └──────────────────────────────────────────┘
        ↑ Active NN writes edits here
        ↑ Standby NN reads edits from here continuously

DataNodes → send block reports to BOTH NameNodes

ZooKeeper → monitors Active NN health
         → triggers automatic failover if Active NN dies
```

**Failover time:** Automatic (seconds) with ZooKeeper, or manual with `hdfs haadmin -failover`.

---

### Q18. What is the difference between `fsimage` and `edits log`?

**Answer:**

| Term | What it is | Analogy |
|------|-----------|---------|
| **fsimage** | A complete snapshot of the filesystem metadata at a point in time | A photograph of the catalog |
| **edits log** | A transaction log of every change since the last fsimage | A diary of changes after the photo |

**How NameNode uses them:**
```
On startup:
  NameNode = fsimage + replay all edits → current state in RAM

Problem:
  Over time, edits log grows huge → slow restart, high memory

Secondary NameNode's job:
  Periodically: new_fsimage = old_fsimage + edits  (called "checkpoint")
  Then replaces fsimage + clears edits log
  → Keeps edits log small
```

---

## 💻 Section 3: Important Commands Asked in Interviews

> **Quick mental map:** `hadoop fs` = HDFS file operations | `hadoop dfsadmin` = cluster admin | `mapred` = job management | `yarn` = resource management

---

### 🗂️ HDFS File System Commands (`hadoop fs` or `hdfs dfs`)

> `hadoop fs` and `hdfs dfs` are interchangeable for HDFS. `hadoop fs` also supports other filesystems (S3, local).

---

#### Listing & Navigation

```bash
# List files in HDFS directory
hadoop fs -ls /user/hadoop/data/
hadoop fs -ls -R /data/          # recursive listing (all subdirectories)
hadoop fs -ls -h /data/          # human-readable sizes (KB, MB, GB)

# Check disk usage
hadoop fs -du /data/             # size of each file/dir inside /data/
hadoop fs -du -s /data/          # total size of /data/ only (summary)
hadoop fs -du -h -s /data/       # human-readable summary

# Count files, directories, bytes
hadoop fs -count /data/          # outputs: DIR_COUNT FILE_COUNT CONTENT_SIZE PATH
```

---

#### Upload & Download

```bash
# Upload from local to HDFS
hadoop fs -put localfile.txt /user/hadoop/         # copy local → HDFS
hadoop fs -put *.csv /data/raw/                    # upload multiple files
hadoop fs -copyFromLocal localfile.txt /hdfs/path/ # same as -put

# Move local file to HDFS (deletes local after upload)
hadoop fs -moveFromLocal localfile.txt /hdfs/path/

# Download from HDFS to local
hadoop fs -get /hdfs/path/file.txt ./localdir/     # copy HDFS → local
hadoop fs -copyToLocal /hdfs/path/file.txt ./      # same as -get

# Move from HDFS to local (deletes HDFS copy after download)
hadoop fs -moveToLocal /hdfs/path/file.txt ./      
```

---

#### Read File Contents

```bash
# Print file contents to terminal
hadoop fs -cat /data/file.txt                  # entire file

# Print first N lines (like Unix head)
hadoop fs -cat /data/file.txt | head -20

# Print last N lines (like Unix tail)
hadoop fs -cat /data/file.txt | tail -20

# Read compressed file directly
hadoop fs -text /data/file.gz                  # auto-decompresses and prints

# tail-like for large files (efficient — doesn't read whole file)
hadoop fs -tail /data/logs/app.log             # last 1 KB of file
```

---

#### Create & Delete

```bash
# Create a directory
hadoop fs -mkdir /user/hadoop/newdir
hadoop fs -mkdir -p /user/hadoop/deep/nested/dir   # create parent dirs too

# Delete a file
hadoop fs -rm /data/file.txt
hadoop fs -rm -skipTrash /data/file.txt            # skip trash (permanent delete)

# Delete a directory (recursive)
hadoop fs -rm -r /data/old_directory/
hadoop fs -rm -r -skipTrash /data/old_directory/   # skip trash

# Empty the trash
hadoop fs -expunge
```

---

#### Copy & Move (within HDFS)

```bash
# Copy within HDFS
hadoop fs -cp /hdfs/source/file.txt /hdfs/dest/
hadoop fs -cp -p /hdfs/source/ /hdfs/dest/         # preserve timestamps, permissions

# Move within HDFS
hadoop fs -mv /hdfs/old/path/file.txt /hdfs/new/path/

# Copy from one cluster to another (DistCp — distributed copy)
hadoop distcp hdfs://cluster1:8020/data/ hdfs://cluster2:8020/data/
hadoop distcp -update hdfs://src/data/ hdfs://dest/data/   # only copy changed files
hadoop distcp -overwrite hdfs://src/ hdfs://dest/          # overwrite existing
```

---

#### Permissions & Ownership

```bash
# Change permissions (Unix-style)
hadoop fs -chmod 755 /data/file.txt
hadoop fs -chmod -R 755 /data/                 # recursive

# Change owner
hadoop fs -chown hadoop:hadoop /data/file.txt
hadoop fs -chown -R hadoop /data/              # recursive

# Change group
hadoop fs -chgrp analytics /data/file.txt
```

---

#### File Information & Verification

```bash
# Check if path exists (returns exit code 0 if exists, 1 if not)
hadoop fs -test -e /data/file.txt && echo "exists" || echo "not found"
hadoop fs -test -d /data/dir/                  # test if directory
hadoop fs -test -z /data/file.txt              # test if file is zero bytes

# Show file status (block size, replication, permissions)
hadoop fs -stat /data/file.txt
hadoop fs -stat "%b %r %n" /data/file.txt      # blocksize, replication, name

# Get file checksum (verify data integrity)
hadoop fs -checksum /data/file.txt

# Show last N bytes of a file
hadoop fs -tail /data/file.txt
```

---

#### Replication Control

```bash
# Change replication factor for a specific file
hadoop fs -setrep 2 /data/file.txt             # set replication to 2
hadoop fs -setrep -R 2 /data/                  # recursive
hadoop fs -setrep -w 3 /data/file.txt          # wait until replication completes

# View replication factor
hadoop fs -ls /data/file.txt                   # 3rd column = replication factor
```

---

#### Append & Merge

```bash
# Append content to an existing HDFS file
hadoop fs -appendToFile localfile.txt /hdfs/existing_file.txt

# Merge multiple HDFS files into one local file
hadoop fs -getmerge /data/parts/ merged_output.txt
hadoop fs -getmerge -nl /data/parts/ merged_output.txt  # add newline between files

# Merge HDFS files into another HDFS file (use distcp or cat)
hadoop fs -cat /data/parts/part-* | hadoop fs -put - /data/merged.txt
```

---

### 🔧 HDFS Admin Commands (`hdfs dfsadmin`)

```bash
# View overall cluster health and stats
hdfs dfsadmin -report
# Shows: live/dead DataNodes, total capacity, used space, block info

# Trigger a filesystem check (safe mode details, block reports)
hdfs dfsadmin -safemode get         # check current safemode status
hdfs dfsadmin -safemode enter       # manually enter safemode (cluster read-only)
hdfs dfsadmin -safemode leave       # manually exit safemode
hdfs dfsadmin -safemode wait        # wait until NN exits safemode (use in scripts)

# Refresh DataNode list (after adding/removing nodes)
hdfs dfsadmin -refreshNodes

# Check and repair filesystem metadata
hdfs fsck /                         # check entire HDFS filesystem
hdfs fsck /data/file.txt            # check specific file
hdfs fsck / -files -blocks -locations   # detailed block location info
hdfs fsck / -list-corruptfileblocks     # list all corrupt blocks

# Decommission a DataNode gracefully
# Step 1: Add node to dfs.hosts.exclude in hdfs-site.xml
# Step 2: Refresh nodes:
hdfs dfsadmin -refreshNodes
# Step 3: Monitor until decommission complete (hdfs dfsadmin -report)

# Balance data across DataNodes (useful after adding new nodes)
hdfs balancer -threshold 10         # balance until no node differs by >10% from average
hdfs balancer -threshold 5 -policy datanode   # node-level balancing

# Upgrade/finalize/rollback
hdfs dfsadmin -finalizeUpgrade      # commit a Hadoop upgrade
hdfs dfsadmin -rollbackUpgrade query  # check rollback status

# Refresh service ACLs without restart
hdfs dfsadmin -refreshServiceAcl
```

---

### 🏃 MapReduce Job Commands (`mapred` / `yarn`)

```bash
# Submit a MapReduce job
hadoop jar /path/to/job.jar com.example.WordCount /input /output

# List running/completed jobs
mapred job -list                    # list all running jobs
mapred job -list all                # list all jobs (running + completed + failed)

# Get status of a specific job
mapred job -status job_1234567890_0001

# Kill a running job
mapred job -kill job_1234567890_0001

# Kill a specific task attempt
mapred job -kill-task attempt_1234567890_0001_m_000000_0

# Fail a specific task (for testing speculative execution)
mapred job -fail-task attempt_1234567890_0001_m_000000_0

# Get job counters (very useful for debugging performance)
mapred job -counter job_1234567890_0001 "Map-Reduce Framework" "Map input records"

# Get job history log location
mapred job -history all /output/
```

---

### 🎛️ YARN Resource Manager Commands

```bash
# List all running applications
yarn application -list
yarn application -list -appStates ALL          # all states (running, finished, failed)
yarn application -list -appTypes SPARK         # filter by type

# Get application status
yarn application -status application_1234567890_0001

# Kill a YARN application
yarn application -kill application_1234567890_0001

# View application logs (very useful for debugging failures)
yarn logs -applicationId application_1234567890_0001
yarn logs -applicationId application_1234567890_0001 -containerId <container_id>
yarn logs -applicationId application_1234567890_0001 -nodeAddress <host:port>

# Check cluster resource status
yarn node -list                     # list all NodeManagers
yarn node -list -all                # include decommissioned nodes
yarn node -status <node-id>         # detailed node info (memory, CPU, containers)

# Queue information
yarn queue -status default          # check queue capacity, usage
yarn schedulerconf                  # view scheduler configuration
```

---

### 📦 Hadoop Archive (HAR) Commands — Fix Small Files

```bash
# Create a HAR archive (bundles many small files into one)
hadoop archive -archiveName data.har -p /hdfs/source/dir/ /hdfs/dest/

# List contents of HAR file
hadoop fs -ls har:///hdfs/dest/data.har/

# Read a file from HAR archive
hadoop fs -cat har:///hdfs/dest/data.har/small_file.txt

# HAR files are read-only — to modify, extract, change, re-archive
hadoop fs -cp har:///hdfs/dest/data.har/file.txt /hdfs/extracted/file.txt
```

---

### 🔄 Sqoop Commands (RDBMS ↔ HDFS)

```bash
# Import a MySQL table to HDFS
sqoop import \
  --connect jdbc:mysql://db-server:3306/sales_db \
  --username admin \
  --password secret \
  --table orders \
  --target-dir /data/orders \
  --num-mappers 4

# Import with Parquet format
sqoop import \
  --connect jdbc:mysql://db-server:3306/sales_db \
  --username admin --password secret \
  --table orders \
  --target-dir /data/orders_parquet \
  --as-parquetfile \
  --num-mappers 8

# Incremental import (only import new rows — great for daily loads)
sqoop import \
  --connect jdbc:mysql://db-server:3306/sales_db \
  --username admin --password secret \
  --table orders \
  --incremental append \
  --check-column order_id \
  --last-value 100000 \
  --target-dir /data/orders

# Import with WHERE clause (partial table)
sqoop import \
  --connect jdbc:mysql://db-server:3306/sales_db \
  --username admin --password secret \
  --table orders \
  --where "status='COMPLETED' AND created_date >= '2024-01-01'" \
  --target-dir /data/orders_completed

# Import all tables from a database
sqoop import-all-tables \
  --connect jdbc:mysql://db-server:3306/sales_db \
  --username admin --password secret \
  --warehouse-dir /data/mysql_import/

# Export from HDFS to MySQL
sqoop export \
  --connect jdbc:mysql://db-server:3306/sales_db \
  --username admin --password secret \
  --table orders_summary \
  --export-dir /data/processed/orders_summary \
  --num-mappers 4

# List databases and tables
sqoop list-databases --connect jdbc:mysql://db-server:3306/ --username admin --password secret
sqoop list-tables --connect jdbc:mysql://db-server:3306/sales_db --username admin --password secret
```

---

### ⚙️ Configuration & Daemon Management

```bash
# Start / Stop all Hadoop daemons
start-dfs.sh               # starts NameNode, DataNode, Secondary NameNode
stop-dfs.sh
start-yarn.sh              # starts ResourceManager, NodeManager
stop-yarn.sh
start-all.sh               # starts everything (not recommended for production)
stop-all.sh

# Start individual daemons
hadoop-daemon.sh start namenode
hadoop-daemon.sh start datanode
hadoop-daemon.sh start secondarynamenode
yarn-daemon.sh start resourcemanager
yarn-daemon.sh start nodemanager

# Format NameNode (⚠️ DANGER — only on first setup, erases all data!)
hdfs namenode -format

# Check Java processes running
jps
# Typical output on master:  NameNode, ResourceManager, SecondaryNameNode
# Typical output on worker:  DataNode, NodeManager
```

---

### 🔐 Security Commands (Kerberos)

```bash
# Authenticate and get Kerberos ticket
kinit username@REALM.COM          # prompts for password
kinit -kt /path/to/user.keytab username@REALM.COM  # use keytab (for automation)

# View current Kerberos tickets
klist                             # shows TGT + service tickets + expiry times

# Destroy all Kerberos tickets (logout)
kdestroy

# Check HDFS security settings
hdfs dfsadmin -report             # includes auth info

# Get delegation token (for long-running jobs)
hdfs fetchdt --webservice http://namenode:50070 /tmp/delegation.token
```

---

### 🎯 Interview-Favorite One-Liners (High Probability)

```bash
# How many blocks does a file have?
hdfs fsck /data/file.txt -files -blocks | grep "Block locations"

# Find all corrupt files
hdfs fsck / -list-corruptfileblocks 2>/dev/null | grep -v "^$"

# Check HDFS space used vs available
hadoop fs -df -h /

# Count total number of files in a directory
hadoop fs -count /data/ | awk '{print $2}'

# Find files larger than 1 GB
hadoop fs -ls -R /data/ | awk '{if ($5 > 1073741824) print $0}'

# Check which DataNodes hold blocks of a file
hdfs fsck /data/file.txt -files -blocks -locations

# Monitor a running MapReduce job (poll every 5 seconds)
watch -n 5 'mapred job -status job_1234567890_0001'

# Compress a file during HDFS upload
hadoop fs -put localfile.txt /data/ -D dfs.replication=2

# Change block size for a specific upload
hadoop fs -D dfs.blocksize=268435456 -put bigfile.txt /data/   # 256 MB block

# Check NameNode memory usage
hdfs dfsadmin -report | grep "Heap Memory"
```

---

## 🎯 Section 4: Scenario-Based Questions

---

### S1. Your MapReduce job is running very slowly. How do you diagnose and fix it?

**Answer — Systematic debugging approach:**

**Step 1: Check for data skew**
```
Symptom: One reducer at 95% progress, all others at 100% — job hangs
Diagnosis: hadoop fs -cat /output/part-r-00005 | wc -l  (one file much larger)

Fix A: Custom Partitioner to spread hot keys
Fix B: Salt the hot keys → aggregate in two passes
Fix C: Use a Combiner to pre-aggregate on mapper side
```

**Step 2: Check for too many small files**
```
Symptom: 100,000 mappers launched for a 1 GB dataset (1 mapper per 10 KB file)
Diagnosis: mapred job -status <job_id> | grep "map tasks"

Fix: Use CombineFileInputFormat
  job.setInputFormatClass(CombineFileInputFormat.class);
  CombineFileInputFormat.setMaxInputSplitSize(job, 134217728); // 128 MB
```

**Step 3: Reduce shuffle data with compression**
```xml
<!-- mapred-site.xml -->
<property><name>mapreduce.map.output.compress</name><value>true</value></property>
<property><name>mapreduce.map.output.compress.codec</name>
          <value>org.apache.hadoop.io.compress.SnappyCodec</value></property>
```

**Step 4: Tune number of reducers**
```
Rule of thumb: numReducers = 0.95 × (numNodes × containersPerNode)
Too few reducers → bottleneck on a few nodes
Too many reducers → too many tiny output files + overhead
```

**Step 5: Check GC / memory pressure**
```xml
<property><name>mapreduce.map.java.opts</name><value>-Xmx2048m</value></property>
<property><name>mapreduce.reduce.java.opts</name><value>-Xmx4096m</value></property>
```

---

### S2. The HDFS NameNode is down. What do you do?

**Scenario A: HA is configured (Hadoop 2.x)**
```
1. ZooKeeper automatically detects Active NN failure
2. Triggers failover to Standby NN (becomes new Active)
3. Cluster continues — downtime: seconds
4. Action items:
   a. Fix the failed NameNode hardware/software
   b. Restart it as the new Standby: hdfs namenode -standby
   c. Verify with: hdfs haadmin -getServiceState nn1
```

**Scenario B: No HA (Hadoop 1.x — disaster scenario)**
```
1. Immediately locate latest fsimage + edits from Secondary NameNode
   (usually at: /hadoop/dfs/namesecondary/)

2. Copy to NameNode machine:
   scp secondary-nn:/hadoop/dfs/namesecondary/* namenode:/hadoop/dfs/name/

3. Start NameNode:
   hdfs namenode -importCheckpoint

4. Verify cluster:
   hdfs dfsadmin -report

5. Lesson learned: ALWAYS configure HA in production!
```

---

### S3. You need to join two huge datasets (1 TB each) in MapReduce. What's your approach?

**Answer:**

**Option 1: Reduce-Side Join (safe, general)**
```
How: Both datasets go through map → shuffle → reduce
     Reducer joins records with matching keys
     
     Mapper A: (order_id, "ORDER:" + order_data)
     Mapper B: (order_id, "DETAIL:" + detail_data)
     Reducer:  for each order_id → combine ORDER + DETAIL records

Pros: Works for any size
Cons: Shuffles 2 TB total over network → expensive
```

**Option 2: Map-Side Join / Broadcast Join (BEST — use when one dataset is small)**
```
How: Load the smaller dataset into distributed cache
     Each Mapper holds the small dataset in a HashMap in memory
     No reducer needed!

Code pattern:
  setup() {
    // Load small table (e.g., 500 MB) from DistributedCache into HashMap
    Path[] paths = context.getLocalCacheFiles();
    HashMap<String, String> lookupTable = loadToHashMap(paths[0]);
  }
  map(key, value, context) {
    String joined = lookupTable.get(value.getOrderId());
    context.write(key, new Text(value + "|" + joined));
  }

Benefit: Zero shuffle! 100x faster than reduce-side join.
Limit: Small table must fit in mapper's memory (tune -Xmx accordingly)
```

**Option 3: Sort-Merge Join (both datasets pre-sorted)**
```
How: Sort both datasets by join key in advance
     Mappers read corresponding partitions side-by-side
     No shuffle needed

Use when: You have pre-partitioned, pre-sorted datasets (e.g., from previous jobs)
```

**Interview answer strategy:** Start with reduce-side (safe), pivot to map-side (optimal), mention Spark for modern workloads.

---

### S4. How do you handle data skew in a MapReduce job?

**Answer:**  
Data skew = one key has far more records than others.

**Real-world example:** E-commerce orders by country — "USA" has 500M records, "Maldives" has 200.

**Solution 1: Key salting (two-phase aggregation)**
```
Phase 1: Add random suffix to hot key
  "USA_0", "USA_1", "USA_2", "USA_3", "USA_4"
  → spreads USA across 5 reducers

Phase 2: Combine the partial results
  USA_0(count=100M) + USA_1(count=120M) + ... → USA(count=500M)
```

**Solution 2: Custom Partitioner**
```java
public int getPartition(Text key, ..., int numPartitions) {
    if (key.toString().equals("USA")) {
        // Spread hot key across multiple reducers using random bucket
        return (key.hashCode() & Integer.MAX_VALUE + new Random().nextInt(5)) % numPartitions;
    }
    return (key.hashCode() & Integer.MAX_VALUE) % numPartitions;
}
```

**Solution 3: Combiner (simplest fix)**
```
Pre-aggregate on mapper side → reducer receives much less data per key
```

**Solution 4: Switch to Spark**
```python
# Spark handles skew better with adaptive query execution
df.repartition(200, "country").groupBy("country").count()
# Or use AQE: spark.sql.adaptive.skewJoin.enabled=true
```

---

### S5. Your team stores millions of small log files daily in HDFS. Performance is degrading. How do you fix it?

**Answer:**

**Diagnosis:**
```bash
hadoop fs -count /data/logs/          # check file count
hdfs dfsadmin -report | grep "Files"  # check NameNode metadata load
```

**Fix 1: Compact files using SequenceFile at ingestion**
```
Instead of writing millions of 1 KB files:
  Write to a SequenceFile where:
    Key   = filename or timestamp
    Value = file content

Result: 1 file containing 10,000 records vs 10,000 files
```

**Fix 2: Daily compaction job (Spark)**
```python
# Run nightly — reads small files, writes one large file per partition
spark.read.text("/data/logs/2024-01-01/") \
    .coalesce(10) \
    .write.mode("overwrite") \
    .parquet("/data/logs_compacted/2024-01-01/")
```

**Fix 3: Use CombineFileInputFormat for processing**
```java
job.setInputFormatClass(CombineFileInputFormat.class);
CombineFileInputFormat.setMaxInputSplitSize(job, 134217728); // 128 MB
// Combines 1000 x 128 KB files into one 128 MB InputSplit
```

**Fix 4: Use HAR for archival**
```bash
hadoop archive -archiveName logs_jan.har -p /data/logs/2024-01/ /data/archive/
hadoop fs -rm -r /data/logs/2024-01/   # remove originals
```

---

### S6. How do you move 10 TB of data from HDFS Cluster A to Cluster B with minimal downtime?

**Answer:**

**Use DistCp (Distributed Copy) — it uses MapReduce internally**

```bash
# Basic cross-cluster copy
hadoop distcp \
  hdfs://clusterA-namenode:8020/data/ \
  hdfs://clusterB-namenode:8020/data/

# Optimized: only copy new or changed files (-update)
hadoop distcp \
  -update \
  -skipcrccheck \
  -m 50 \
  hdfs://clusterA:8020/data/ \
  hdfs://clusterB:8020/data/

# Options explained:
# -update      → only copy files that don't exist or are different (idempotent)
# -skipcrccheck → faster — skip checksum comparison (use only if you trust network)
# -m 50        → use 50 mappers in parallel (tune based on bandwidth)
# -bandwidth 100 → limit to 100 MB/s per mapper (protect production workloads)
```

**Migration strategy:**
```
Day 1-N:  distcp -update (runs in background, copies bulk of data)
          Run again daily to keep up with new data
Day N+1:  Maintenance window → distcp -update (final delta)
          Switch applications to point to Cluster B
          Verify → decommission Cluster A
```

---

### S7. How do you optimize a Hive query on Hadoop that takes 4 hours?

**Answer — Optimization hierarchy (always try in this order):**

```sql
-- 1. Partition your table (most impactful — skip entire partitions)
CREATE TABLE orders (
  order_id BIGINT,
  customer_id INT,
  amount DOUBLE
) PARTITIONED BY (dt STRING, country STRING)
STORED AS ORC;

-- Bad query:  SELECT * FROM orders WHERE dt = '2024-01-01'
-- Good query: Hive prunes partition, reads ONLY 2024-01-01 folder

-- 2. Use ORC + compression
STORED AS ORC TBLPROPERTIES ("orc.compress"="SNAPPY");

-- 3. Use Tez instead of MapReduce execution engine
SET hive.execution.engine = tez;   -- 3x-10x faster for most queries

-- 4. Enable Cost-Based Optimizer (CBO) — Hive picks best join order
SET hive.cbo.enable = true;
SET hive.compute.query.using.stats = true;
ANALYZE TABLE orders COMPUTE STATISTICS;                 -- build stats first
ANALYZE TABLE orders COMPUTE STATISTICS FOR COLUMNS;

-- 5. Enable vectorized execution (process 1024 rows at a time, not 1)
SET hive.vectorized.execution.enabled = true;
SET hive.vectorized.execution.reduce.enabled = true;

-- 6. Auto-convert small table joins to map-side joins
SET hive.auto.convert.join = true;
SET hive.mapjoin.smalltable.filesize = 25000000;  -- 25 MB threshold

-- 7. Enable bucketing for repeated large joins
CREATE TABLE orders CLUSTERED BY (customer_id) INTO 256 BUCKETS;
CREATE TABLE customers CLUSTERED BY (customer_id) INTO 256 BUCKETS;
SET hive.optimize.bucketmapjoin = true;  -- now uses bucket map join

-- 8. Reduce number of output files (small files fix)
SET hive.merge.mapfiles = true;          -- merge small map output files
SET hive.merge.mapredfiles = true;       -- merge small reduce output files
SET hive.merge.size.per.task = 256000000;  -- target 256 MB per file
```

---

## 🏆 Section 5: FAANG/MAANG Level Questions

---

### F1. Design a system to process 100 TB of clickstream data daily using the Hadoop ecosystem.

**Answer:**

**First, clarify requirements (always do this in system design):**
- Latency needed? (Batch = hours, Near-real-time = minutes, Real-time = seconds)
- Query pattern? (Ad-hoc SQL, pre-built dashboards, ML features)
- Retention? (Raw logs: 90 days, aggregated: forever)
- Team size? (Affects complexity of architecture)

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                   INGESTION LAYER                           │
│  Web/Mobile Apps → Apache Kafka (replicated, partitioned    │
│                    by user_id for ordering guarantees)      │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
┌────────▼──────────┐        ┌───────────▼──────────┐
│   BATCH LAYER     │        │    SPEED LAYER        │
│ Apache Spark on   │        │ Spark Structured      │
│ YARN (daily jobs) │        │ Streaming / Flink     │
│ Reads from Kafka  │        │ (5-min micro-batches) │
│ or HDFS           │        │                       │
└────────┬──────────┘        └───────────┬──────────┘
         │                               │
┌────────▼──────────┐        ┌───────────▼──────────┐
│   STORAGE LAYER   │        │   SERVING LAYER       │
│ HDFS (ORC/Parquet)│        │ HBase (low-latency    │
│ Partitioned by:   │        │ lookups, recent data) │
│   date/event_type │        │                       │
└────────┬──────────┘        └───────────┬──────────┘
         │                               │
┌────────▼───────────────────────────────▼──────────┐
│               QUERY LAYER                         │
│ Apache Hive / Presto / Impala (ad-hoc SQL)        │
│ Spark SQL (complex analytics)                     │
└────────────────────┬──────────────────────────────┘
                     │
┌────────────────────▼──────────────────────────────┐
│         VISUALIZATION / API LAYER                 │
│  Superset / Tableau / Custom API                  │
└───────────────────────────────────────────────────┘
```

**Key design decisions:**

| Concern | Decision | Why |
|---------|----------|-----|
| Storage format | ORC (Hive) / Parquet (Spark) | Columnar, 5-10x compression |
| Partitioning | `dt=YYYY-MM-DD/event_type=click` | Partition pruning for queries |
| Retention | Raw: 90 days HDFS → S3 Glacier | Cost optimization |
| Small files | Daily compaction Spark job | Prevent NameNode overload |
| Fault tolerance | Kafka RF=3, HDFS RF=3, YARN HA | No data loss |
| Security | Kerberos + Ranger policies | Access control per team |

---

### F2. Explain exactly-once semantics in a Hadoop pipeline. How do you implement it?

**Answer:**

**The three guarantees:**

| Guarantee | Data loss possible? | Duplicates possible? | Difficulty |
|-----------|--------------------|-----------------------|-----------|
| At-most-once | ✅ Yes | ❌ No | Easy |
| At-least-once | ❌ No | ✅ Yes | Medium |
| Exactly-once | ❌ No | ❌ No | Hard |

**Why exactly-once is hard in distributed systems:**  
If a task writes data then crashes before reporting success, the framework retries — potentially writing twice.

**Strategy 1: Idempotent output (most practical)**
```
Write to a staging directory, then atomically rename:

Step 1: Job writes to /tmp/job_abc123/output/
Step 2: On success → rename /tmp/job_abc123/output/ to /data/final/2024-01-01/
        (HDFS rename is atomic)
Step 3: If job fails and reruns → overwrites /tmp/job_abc123/ (safe, idempotent)

Key: destination is determined by job ID, not content → same run = same output
```

**Strategy 2: Hadoop's FileOutputCommitter (built-in two-phase commit)**
```
Phase 1 (task commit): Each task writes to _temporary/<task_attempt_id>/
Phase 2 (job commit):  On success, commitJob() renames _temporary/ to final path
                       If any task fails → retry with new attempt_id → no partial writes
```

**Strategy 3: Deduplication via unique keys**
```
Every event has a globally unique event_id (UUID)
Before writing, check HBase/Redis: does event_id exist?
  If yes → already processed, skip
  If no  → process + write + mark as processed in HBase

Trade-off: Requires a distributed lock/lookup per record → latency cost
```

**Strategy 4: Spark Structured Streaming checkpointing**
```python
spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "broker:9092") \
    .option("subscribe", "events") \
    .load() \
    .writeStream \
    .format("parquet") \
    .option("checkpointLocation", "/hdfs/checkpoints/job1/") \  # ← stores Kafka offsets
    .option("path", "/hdfs/output/events/") \
    .trigger(processingTime="5 minutes") \
    .start()

# Checkpoint stores Kafka offsets → restart from exact offset → no re-processing
# Combine with idempotent writes → effectively exactly-once
```

---

### F3. How does Hadoop achieve fault tolerance at every layer?

**Answer:**

**Layer 1: HDFS (Storage)**
```
Block replication:
  3 copies on different nodes (different racks)
  If DataNode dies → NameNode detects via missed heartbeat (10 min default)
  → Triggers re-replication: remaining blocks copied to new nodes
  → Cluster maintains RF=3 automatically

NameNode HA:
  Active + Standby NN (Hadoop 2.x)
  JournalNodes keep edit logs in sync
  ZooKeeper triggers automatic failover in seconds

Rack awareness:
  Replicas on ≥2 different racks
  → Survives entire rack failure (rack switch dies)
```

**Layer 2: MapReduce (Compute)**
```
Task failure:
  Each task attempt tracked by ApplicationMaster
  If task fails → retry up to 4 times on different nodes (mapreduce.map.maxattempts)
  If all 4 fail → job fails with error

Node failure mid-job:
  ApplicationMaster detects → reschedules all incomplete tasks from that node
  Completed tasks: not re-run (output on HDFS is safe)

ApplicationMaster failure:
  ResourceManager restarts AM up to 2 times (yarn.resourcemanager.am.max-attempts)
  AM restores state from HDFS checkpoint

Speculative execution:
  Launches duplicate of slow/straggler tasks → first to finish wins
```

**Layer 3: YARN (Resource Management)**
```
NodeManager failure:
  ResourceManager detects missed heartbeats
  Marks node as lost → reschedules containers elsewhere

ResourceManager failure:
  RM HA: Active + Standby RM
  ZooKeeper leader election
  Standby becomes Active → running applications resume
  State stored in ZooKeeper + shared state store (HDFS or ZK)
```

---

### F4. Compare MapReduce vs Apache Spark. When would you still use MapReduce in 2024?

**Answer:**

| Aspect | MapReduce | Apache Spark |
|--------|-----------|-------------|
| Processing model | Disk-based (read HDFS → map → write disk → read → reduce → write HDFS) | In-memory DAG (pipelines stages, spills only if needed) |
| Speed | Baseline | 10x–100x faster for iterative/in-memory workloads |
| API | Verbose Java (mapper/reducer classes) | High-level (DataFrame, SQL, Python/Scala/R/Java) |
| Fault tolerance | Re-run failed task from HDFS | RDD lineage — recompute only lost partitions |
| Streaming | ❌ Not supported | ✅ Structured Streaming |
| ML | Limited (Mahout, mostly abandoned) | ✅ MLlib built-in |
| Memory requirement | Low (can run with 1 GB heap) | Needs substantial RAM (GBs per executor) |
| Startup time | High (JVM per task) | Low (executor JVMs stay alive between tasks) |

**When would you still choose MapReduce in 2024?**

1. **Memory-constrained environments**: Processing a 100 TB dataset on nodes with only 8 GB RAM — MapReduce streams from disk without needing to hold data in memory
2. **Mature, certified pipelines**: Banks/government systems with compliance requirements only certify specific, proven tools
3. **Extreme stability requirements**: MapReduce has been production-proven for 15+ years with zero major bugs; Spark still gets non-trivial breaking changes
4. **Piggyback on existing infrastructure**: If the team has deep MapReduce expertise and the jobs work fine, rewriting = risk with little reward

**Honest answer for FAANG:** In 2024, Spark is the default for new development. MapReduce is primarily legacy maintenance.

---

### F5. What is the CAP theorem and how does HDFS fit into it?

**Answer:**

**CAP theorem:** A distributed system can guarantee only **2 of these 3** properties simultaneously:

```
        C — Consistency
       / \   All nodes see the same data at the same time
      /   \
     /     \
    A───────P
    |       |
Availability  Partition Tolerance
Every request  System works even
gets a response  if network splits
```

**HDFS is CP (Consistency + Partition Tolerance):**

```
C: When a write is acknowledged → all subsequent reads return that data
   HDFS won't acknowledge a write until all replica writes succeed

P: HDFS continues operating despite network partitions
   (some operations degrade, but cluster doesn't fully stop)

Sacrifices A: During failover window (seconds to minutes), 
              some operations may be rejected rather than returning stale data
              
→ Correct for batch analytics: you'd rather wait or get an error 
  than silently process wrong data
```

**Contrast with other systems:**

| System | CAP Type | Trade-off |
|--------|----------|-----------|
| HDFS | CP | Availability sacrificed during failures |
| HBase | CP | Same — strong consistency for transactional reads |
| Cassandra | AP | Eventual consistency — prioritizes being always available |
| MongoDB | CP (configurable) | Can tune toward AP with eventual consistency reads |

---

### F6. Design a migration plan for 5 PB of HDFS data to AWS S3.

**Answer:**

**Phase 1: Discovery & Assessment (Weeks 1-4)**
```
1. Inventory all datasets:
   hadoop fs -ls -R / > hdfs_inventory.txt
   hadoop fs -du -h -s /data/ /logs/ /user/ /tmp/

2. Classify data:
   Hot   (accessed daily)   → migrate last
   Warm  (weekly access)    → migrate second
   Cold  (rare/archival)    → migrate first

3. Audit all consumers:
   - MapReduce jobs reading from HDFS paths
   - Hive tables pointing to HDFS locations
   - Spark jobs, Sqoop imports, Flume sinks

4. Cost analysis:
   - S3 storage cost vs on-prem hardware depreciation
   - Egress costs (S3 charges per GB transferred out)
   - EMR costs vs YARN cluster maintenance
```

**Phase 2: Parallel Ingestion (Months 1-3)**
```bash
# Use S3DistCp for efficient parallel copy
# (Optimized version of DistCp for S3)

# Start with cold/archival data
hadoop jar /usr/share/aws/emr/s3-dist-cp/lib/s3-dist-cp.jar \
  --src hdfs:///data/archive/2022/ \
  --dest s3://company-datalake/archive/2022/ \
  --outputCodec snappy \
  --groupBy '.*/(year=\d+/month=\d+)/.*' \    # maintain partitioning
  --numWorkers 50

# Validate checksums
aws s3api list-objects --bucket company-datalake --prefix archive/2022/ \
  | jq '.Contents[].ETag' > s3_checksums.txt
# Compare with HDFS checksums
```

**Phase 3: Migrate Compute (Month 3-4)**
```python
# Update Spark jobs from HDFS to S3A
# Before:
df = spark.read.parquet("hdfs:///data/sales/")

# After:
df = spark.read.parquet("s3a://company-datalake/sales/")

# Configure S3A connector in core-site.xml:
# fs.s3a.endpoint = s3.amazonaws.com
# fs.s3a.access.key = <IAM role — never hardcode>
# fs.s3a.connection.maximum = 200

# Run both old + new pipelines in parallel for 2 weeks
# Compare row counts, checksums of outputs
```

**Phase 4: Cutover & Decommission (Week N)**
```
1. Freeze new writes to HDFS
2. Run final S3DistCp -update (sync delta)
3. Update all pipelines to S3 paths
4. Validate outputs for 2 weeks
5. Decommission HDFS DataNodes one by one
   hdfs dfsadmin -decommission datanode01
   Monitor: hdfs dfsadmin -report | grep Decommission
6. Shut down NameNode last
```

**Key technical concerns:**
- **S3 consistency**: AWS S3 now offers strong read-after-write consistency (since Nov 2020)
- **Performance**: S3 has ~10x higher latency vs HDFS → use larger Parquet files, fewer requests
- **Cost optimization**: Use S3 Intelligent-Tiering for automatic hot/cold classification
- **Security**: IAM roles instead of static keys; S3 bucket policies; encryption at rest (SSE-S3)

---

### F7. Design a real-time fraud detection system using the Hadoop ecosystem.

**Answer:**

**Requirements:**
- Detect fraud within 100ms of transaction
- Process 50,000 transactions/second peak
- Historical pattern analysis (last 30 days)
- Model retraining weekly

```
Architecture:

[POS Terminals / Mobile / Web]
            │
            ▼
    Apache Kafka
    (topic: transactions, partitioned by account_id)
    (replication factor = 3, retention = 7 days)
            │
            ├─────────────────────────────────────┐
            │                                     │
            ▼                                     ▼
  ┌─────────────────────┐             ┌───────────────────────┐
  │  REAL-TIME PATH     │             │   BATCH PATH          │
  │  Apache Flink       │             │   Apache Spark        │
  │  (per-event, <5ms)  │             │   (weekly retraining) │
  │                     │             │   Reads from HDFS     │
  │  Rules engine:      │             │   MLlib trains model  │
  │  - >5 txn/1 min     │             │   Saves to HDFS       │
  │  - location jump    │             │                       │
  │  ML scoring:        │             └───────────┬───────────┘
  │  - model from HDFS  │                         │
  └──────────┬──────────┘                    ┌────▼─────────┐
             │                               │  HDFS        │
             │                               │  (ORC format)│
             │ BLOCK / FLAG                  │  Raw logs    │
             ▼                               │  Model store │
  ┌─────────────────────┐                    └──────────────┘
  │  Decision Service   │
  │  (<100ms response)  │
  │  Redis cache for    │
  │  account features   │
  └─────────┬───────────┘
            │
            ├── Block → Return decline to POS
            └── Flag  → Write to HBase (fraud_alerts table)
                     → Alert risk team via Kafka topic: alerts
```

**Key design points:**

| Requirement | Solution | Why |
|-------------|----------|-----|
| < 100ms latency | Flink + Redis feature cache | Flink is sub-millisecond; Redis for O(1) lookups |
| Account history | HBase (row key = account_id) | Random reads in milliseconds |
| Audit trail | All decisions to HDFS ORC | Immutable, queryable via Hive |
| Model freshness | Weekly Spark retraining | Fraud patterns change; stale model = more misses |
| Scale to 50K TPS | Kafka partitioning by account_id | Parallelism + ordering per account |
| Feature store | Redis (last 100 transactions per account) | Expiring TTL, fast writes and reads |

---

### F8. Explain HDFS Federation and when you'd use it.

**Answer:**

**Problem with single NameNode namespace:**
```
Single NameNode holds all metadata in RAM.
Rule of thumb: ~150 bytes per file/block entry

64 GB NameNode RAM → max ~400 million files
→ Multi-petabyte clusters with billions of small files hit this limit
→ NameNode becomes the bottleneck even with HA
```

**HDFS Federation = Multiple independent NameNodes, sharing the same DataNodes**

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  NameNode 1  │  │  NameNode 2  │  │  NameNode 3  │
│  Manages:    │  │  Manages:    │  │  Manages:    │
│  /user/*     │  │  /data/*     │  │  /projects/* │
│  /tmp/*      │  │  /logs/*     │  │  /warehouse/ │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
              ┌──────────▼──────────┐
              │   Shared DataNodes  │
              │  (Block Pool per NN)│
              │  DN1, DN2, ... DN500│
              └─────────────────────┘

Each NameNode manages its own block pool.
DataNodes serve blocks for ALL NameNodes.
Failure of NN1 only affects /user/* and /tmp/* — not /data/* or /projects/*
```

**Benefits:**
- Scales metadata horizontally (more NNs = more files supported)
- Namespace isolation (teams get dedicated NNs, no noisy neighbor on metadata)
- Failure isolation (one NN down doesn't kill entire cluster)
- Independent upgrades per namespace

**When to use:**
- Cluster has > 100 million files
- Multi-tenant cluster with many teams
- Need metadata isolation for compliance/security
- Metadata has become the bottleneck despite NN HA

---

### F9. Walk through what happens internally when you run `hadoop fs -put file.txt /data/`

**Answer — Full internal walkthrough:**

```
Step 1: CLIENT INITIALIZATION
  Client reads hdfs-site.xml → finds NameNode address (fs.defaultFS)
  Creates DFSClient → opens RPC connection to NameNode

Step 2: CREATE REQUEST TO NAMENODE
  Client: "I want to create /data/file.txt"
  NameNode checks:
    ✓ Does /data/ exist?
    ✓ Does /data/file.txt already exist? (error if yes, unless -f flag)
    ✓ Does this user have write permission on /data/?
  NameNode responds: "OK. For Block 1, write to: DN1, DN3, DN5"
    (NM picks DataNodes using rack awareness algorithm)
  NameNode adds file to namespace with state = "under construction"

Step 3: DATA PIPELINE SETUP (pipelined replication)
  Client → opens TCP connection to DN1 (closest/first replica)
  DN1   → opens TCP connection to DN3
  DN3   → opens TCP connection to DN5
  (Client only talks to DN1 — DN1 forwards to DN3 → DN3 to DN5)

Step 4: DATA TRANSFER (streaming)
  Client sends data packets (64 KB each) to DN1
  DN1 writes to local disk + forwards packet to DN3
  DN3 writes to local disk + forwards packet to DN5
  DN5 writes to local disk
  
  ACK chain (reverse):
  DN5 → ACK → DN3 → ACK → DN1 → ACK → Client
  (Client advances only after all 3 replicas acknowledge each packet)

Step 5: BLOCK COMPLETION
  After all packets sent → Client sends "block complete" to NameNode
  NameNode records: Block1 → {DN1, DN3, DN5}
  
  If file > 128 MB → repeat Steps 2-5 for Block 2, Block 3, etc.

Step 6: FILE CLOSE
  Client calls FSDataOutputStream.close()
  Client notifies NameNode: "file complete"
  NameNode changes file state from "under construction" to "complete"
  File is NOW visible to other readers (HDFS uses lease-based consistency)

Error handling during write:
  If DN3 fails mid-write:
    DN1 detects broken pipeline → notifies NameNode
    NameNode allocates a replacement DataNode (e.g., DN7)
    DN1 rebuilds pipeline: DN1 → DN5 → DN7
    Write continues from last acknowledged packet
    NameNode will asynchronously fix replication after write completes
```

---

### F10. How does Kerberos authentication work in Hadoop?

**Answer:**

**Simple analogy:**  
Kerberos is like a **concert wristband system**:
1. Show your ID at the box office → get a wristband (TGT)
2. At each event gate, show wristband → get an entry stamp (Service Ticket)
3. Show stamp to enter the specific event (access specific service)

**Technical flow in Hadoop:**

```
Components:
  KDC (Key Distribution Center) = The "box office"
  TGT (Ticket Granting Ticket)  = The "wristband"
  Service Ticket                = The "entry stamp"

Step 1: User Authentication
  $ kinit alice@COMPANY.COM  (prompts for password)
  Client → KDC Authentication Service: "I'm alice, here's my password hash"
  KDC → Client: TGT (encrypted, expires in 24h by default)
  TGT stored in: /tmp/krb5cc_<uid>

Step 2: Get Service Ticket for HDFS NameNode
  Client → KDC Ticket Granting Service: "I need to talk to HDFS/namenode@COMPANY.COM"
  Client presents TGT (proof of identity)
  KDC → Client: Service Ticket for NameNode (encrypted with NameNode's key)

Step 3: Access NameNode
  Client → NameNode: "Here's my Service Ticket"
  NameNode decrypts with its own key → verifies → grants access
  NameNode → Client: "Access granted for alice"

Step 4: Delegation Tokens (for long-running jobs)
  Problem: MapReduce tasks on worker nodes need HDFS access but can't use TGT
  Solution: NameNode issues a Delegation Token (a limited proxy credential)
  JobClient stores token in job configuration
  Worker tasks use the Delegation Token to access HDFS directly
  Token has shorter TTL and limited renewal rights
```

**Key commands:**
```bash
kinit alice@COMPANY.COM              # authenticate (interactive password)
kinit -kt /etc/security/alice.keytab alice@COMPANY.COM  # keytab (automation)
klist                                # view tickets + expiry
klist -e                             # show encryption types
kdestroy                             # destroy all tickets
kinit -R                             # renew TGT before expiry
```

---

### F11. How would you implement a Lambda Architecture for a large-scale Hadoop system?

**Answer:**

**What is Lambda Architecture?**  
A design pattern that handles both batch (accurate but slow) and stream (fast but approximate) processing by running both in parallel and merging results.

```
┌────────────────────────────────────────────────────────────┐
│                     LAMBDA ARCHITECTURE                    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Raw Data → ─────────────┬────────────────────────────    │
│                           │                               │
│              ┌────────────▼───────────┐                   │
│              │     BATCH LAYER        │                   │
│              │  Spark on HDFS         │                   │
│              │  Reprocesses ALL data  │                   │
│              │  (correct, complete)   │                   │
│              │  Latency: hours/days   │                   │
│              └────────────┬───────────┘                   │
│                           │                               │
│  Raw Data → ─────────────┼────────────────────────────    │
│                           │                               │
│              ┌────────────▼───────────┐                   │
│              │     SPEED LAYER        │                   │
│              │  Spark Streaming/Flink │                   │
│              │  Processes recent data │                   │
│              │  (approximate, fast)   │                   │
│              │  Latency: seconds      │                   │
│              └────────────┬───────────┘                   │
│                           │                               │
│              ┌────────────▼───────────┐                   │
│              │    SERVING LAYER       │                   │
│              │  Merge batch views +   │                   │
│              │  real-time views       │                   │
│              │  (Hive + HBase)        │                   │
│              └────────────────────────┘                   │
└────────────────────────────────────────────────────────────┘

Query = Batch view (Hive) + Speed view (HBase/Redis)
        (full historical)  (last few hours)
```

**Real example — daily active users:**
```
Batch layer (Spark, runs nightly):
  → Computes exact DAU for all dates → writes to Hive ORC table

Speed layer (Flink, continuous):
  → Computes approximate DAU for today → writes to Redis with 5-min TTL

Query (Presto):
  SELECT date, dau FROM hive_batch WHERE date < CURRENT_DATE
  UNION ALL
  SELECT CURRENT_DATE, GET_REDIS('dau_today')
```

**When to use Lambda vs Kappa:**
- **Lambda**: Different teams own batch vs streaming, or correctness of historical data is critical
- **Kappa** (streaming only): Simpler operationally — one system, one codebase; use when stream reprocessing is fast enough

---

## 🔑 Quick Reference Cheat Sheet

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CORE COMPONENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
HDFS + MapReduce + YARN + Hadoop Common

HDFS INTERNALS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Block size:           128 MB (default)
Replication factor:   3 (default)
NameNode:             Metadata ONLY — no actual data
DataNode:             Actual block storage
Secondary NameNode:   Checkpoint helper — NOT a backup!
Standby NameNode:     True HA failover
NameNode memory:      ~150 bytes per file/block entry
Max files (64 GB NN): ~400 million files
Heartbeat interval:   3 seconds (DataNode → NameNode)
Dead node threshold:  10 minutes of missed heartbeats

MAPREDUCE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Phases:               Map → Shuffle & Sort → Reduce
Combiner:             Mini-reducer, runs locally on mapper output
InputSplit:           Logical chunk assigned to one Mapper
# Mappers:            = # InputSplits
Shuffle bottleneck:   Usually the slowest phase
Speculative exec:     Duplicate slow tasks on other nodes

YARN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ResourceManager:      Cluster master (allocates resources)
NodeManager:          Per-node agent (manages containers)
ApplicationMaster:    Per-job manager (tracks tasks)
Container:            A unit of resource (CPU + Memory)

FILE FORMATS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Best for Hive:        ORC
Best for Spark:       Parquet
Streaming/schema:     Avro
MR intermediate:      SequenceFile

KEY COMMANDS (Must Know)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
hadoop fs -ls /           List HDFS directory
hadoop fs -put f /data/   Upload to HDFS
hadoop fs -get /data/f .  Download from HDFS
hadoop fs -rm -r /dir/    Delete directory
hadoop fs -du -h -s /     Check disk usage
hadoop fs -chmod 755 /f   Change permissions
hadoop fs -setrep 2 /f    Change replication
hdfs dfsadmin -report     Cluster health report
hdfs fsck /               Filesystem check
yarn application -list    List running apps
yarn logs -applicationId  View app logs
mapred job -list          List MR jobs
mapred job -kill <id>     Kill a MR job
hadoop distcp src dest    Copy between clusters
jps                       Check running daemons

PERFORMANCE TIPS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Small files:          CombineFileInputFormat / SequenceFile / HAR
Data skew:            Key salting + two-phase aggregation
Slow shuffle:         Enable Snappy compression on map output
Slow queries (Hive):  ORC + partition pruning + Tez engine
Join optimization:    Map-side join for small tables
Fault tolerance:      Replication RF=3 + YARN speculative execution

CAP THEOREM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
HDFS:       CP (Consistency + Partition tolerance)
HBase:      CP
Cassandra:  AP (Availability + Partition tolerance)

ECOSYSTEM TOOLS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Sqoop:       RDBMS ↔ HDFS transfer
Hive:        SQL on Hadoop (batch)
HBase:       NoSQL on HDFS (real-time)
Kafka:       Distributed message queue
Spark:       In-memory distributed compute (MapReduce replacement)
Flink:       True streaming compute
Oozie:       Workflow scheduler for Hadoop jobs
ZooKeeper:   Distributed coordination (HA, leader election)
Ranger:      Security & access control
Kerberos:    Authentication
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 📚 Topics to Study Next

After mastering Hadoop, these are the natural next steps:

| Topic | Why It Matters |
|-------|---------------|
| **Apache Spark** | Modern replacement for MapReduce — must-know |
| **Apache Hive** | SQL interface to Hadoop — used heavily in data engineering |
| **Apache HBase** | When you need real-time reads/writes on Hadoop data |
| **Apache Kafka** | How data gets into Hadoop in modern pipelines |
| **Apache Flink** | True streaming — better than Spark Streaming for latency |
| **Apache Airflow** | How to orchestrate and schedule Hadoop/Spark jobs |
| **Delta Lake / Iceberg** | ACID transactions on data lakes — the future of HDFS |
| **AWS EMR / GCP Dataproc** | Managed Hadoop in the cloud — replaces on-prem clusters |
| **Apache Ranger / Atlas** | Security and governance — important for enterprise roles |
| **Presto / Trino** | Fast interactive SQL on HDFS/S3 at petabyte scale |

---

*Good luck with your interviews! Remember: understand the WHY behind every answer — that's what separates good candidates from great ones. 🚀*
