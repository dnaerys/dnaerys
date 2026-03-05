### Dnaerys

_Dnaerys_ is a state-of-the-art, _**distributed, horizontally scalable, in-memory low latency genome variant store**_ — a specialised analytical DBMS
designed to store genetic variations and execute genomic algorithms at scale.

---

#### _TL;DR_

Documentation: [dnaerys.org](https://dnaerys.org/) <br>
License: [EULA](https://github.com/dnaerys/dnaerys/blob/master/EULA) <br>
Installation: <br>
- CLI ctl tool: [releases](https://github.com/dnaerys/dnaerys/releases)
- Cluster: `docker pull dnaerys/dnaerys-k8s:latest`

---

#### _Why a Specialised Variant Store ?_

On a small scale, genomic data can be managed well in almost any modern DBMS. However, as datasets grow to hundreds of thousands of WGS samples,
DBMS response times become the primary bottleneck for downstream applications.

Historically, horizontally scalable variant stores relied on general-purpose, disk-based architectures (Data Warehouses and Data Lakes).
This disk-oriented design — often coupled with remote storage — puts them at a _**three-order-of-magnitude disadvantage**_ compared to
purpose-built, in-memory systems.

> ##### **Thesis**
> _**General-purpose DBMS architectures are sub-optimal for low-latency, scalable variant stores.**_

---

**<H5> I. Query Plan Optimization and Code Generation </H5>**
Specialized systems avoid the heavy execution stages mandatory in general-purpose engines:

* **The Optimizer Burden:** Generating an optimal plan for arbitrary queries is an _**NP-Complete problem**_. General-purpose optimizers rely on heuristics that often result in sub-optimal paths for genomic data.
* **Direct Execution:** Dnaerys implements optimal execution plans directly. Since the genomic data model and queries are known *a priori*, we eliminate the need for runtime query optimizers, code generation, or plan interpretation.

**<H5> II. Storage Model Efficiency </H5>**
Traditional Relational DBMS models face significant scaling hurdles in genomics:

* **Row-oriented (NSM):** These are unfit for in-memory genomic data; the structural overhead makes RAM requirements prohibitively expensive.
* **Column-oriented (DSM/PAX):** While better at compression, the main memory required for large genomic datasets remains beyond the limit of practical feasibility in most general-purpose implementations.

**<H5> III. Primary Data Allocation and Access Latency </H5>**

* **Architectural Overheads:** Managing internal buffer pools and other subsystems in disk-based systems incur significant performance cost.
It is simple to see by placing all database files on RAM disks and eliminating disk I/O. Main-memory designed systems usually outperform
disk-based DBMS counterparts running on RAM disks.
* **Physical Access Times:** Persistent storage is slower than RAM by approximately _**three orders of magnitude**_.
* **The Cloud Penalty:** Solutions based on object stores (Snowflake, Databricks, ClickHouse, etc.) are affected by:
    * **Network Overheads:** latency from serialization and TLS.
    * **Data Locality:** These architectures **pull data to the query**, whereas Dnaerys **pushes query to the data**.

---

#### _The Solution_

Dnaerys provides a specialised environment for high-throughput genomics:

* **Real-time Genome Exploration at Scale:** Provides millisecond-range response times for queries.
* **Horizontal Scalability:** Scales to _**hundreds of thousands of WGS samples**_ while maintaining strict latency constraints.
* **In-Memory Analytics:** Executes compute-heavy genetic algorithms directly inside the database engine.
* **LLM-native:** Provides seamless access for LLMs to whole genomic datasets via [MCP servers](https://github.com/dnaerys/onekgpd-mcp).

---

#### _Design Principles_

Dnaerys is engineered for predictable latency and high availability in distributed environments.

**<H5> I. Distributed Architecture </H5>**

* **Homogeneous Distributed System:** Dnaerys runs identical software on all nodes. Every node is a peer with equal roles; any node can accept and coordinate queries.
* **Shared-Nothing & In-Memory:** To eliminate I/O bottlenecks, each node stores its entire partition of data in RAM, utilizing a shared-nothing architecture to maximize independent scaling.
* **Horizontal Scalability:** Designed so that query latency remains within the same constraints even as the data volume and the number of nodes grow.
* **Column-Oriented OLAP:** Optimized for analytical workloads, using columnar storage to minimize memory footprint and accelerate scan-heavy genomic queries.

**<H5> II. High-Performance Execution & Specialization </H5>**

* **Code Specialization:**
    * **No Runtime Overhead:** Since the data model is known *a priori*, query logic is pre-written. This avoids the latency penalties of runtime code generation or compilation (JIT).
    * **Hardware-Aware:** Leverages the JVM for low-level specialization, including _SIMD_ and _NUMA_.
* **Modern Processing Model:**
    * Uses a **materialization query execution model** with a bottom-to-top approach.
    * Utilizes **push-based task assignment** to minimize scheduling overhead.
* **Parallelization:** Employs a hybrid **Actor-based and Multi-Threaded (MT)** models to maximize CPU utilization across distributed nodes.

**<H5> III. Resilience & The PACELC Model </H5>**
Dnaerys operates primarily as a **PA/EC** system (Availability over Consistency during partitions; Consistency over Latency during normal operation).

* **Network Partition Handling (PA):** In the event of a network split, sub-clusters continue to serve clients. Responses are transparently marked as _"potentially incomplete"_ if the query targets data on unreachable nodes.
* **Configurable Consistency:** Can be tuned to a **PC/EC** system via an optional split-brain resolver for environments requiring strict consistency.
* **Advanced Fault Detection:**
    * **[φ Accrual Failure Detector](https://dspace.jaist.ac.jp/dspace/bitstream/10119/4784/1/IS-RR-2004-010.pdf):** A flexible, adaptive detector that adjusts to varying network latencies and deviations.
    * **Push-Pull Gossip Protocol:** Ensures rapid cluster-state awareness across all nodes without a central coordinator.
* **Data Integrity:** Multi-layer validation and integrity checks, including non-cryptographic hashes, ensure detection of data corruption.

**<H5> IV. Cloud-Scale Operations </H5>**

* **Deployment Flexibility:** Runs seamlessly on _Kubernetes clusters_ and is equally at home on infrastructure ranging from laptops to enterprise clouds.
* **Observability-First:** Designed with deep focus on _Observability and Ops_, ensuring that performance bottlenecks are visible in real-time.
* **Graceful Degradation:** If a node fails, the cluster remains operational. The system prioritizes uptime, serving all available data while flagging missing segments to the requester.

---

