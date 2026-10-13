# Chapter Overview

[TOC]

## ER Model

* Entity, Relationship, Attribute, Value

* Primary key, Superkey, Candidate key, Foreign key

* ER diagram

  <img src="week1_assets/diagram.png" alt="diagram" width="520">

* Normalization

  * 1NF: atomic attribute values
  * 2NF: no partial dependency
  * 3NF: no transitive dependency

* Other data model: relational model, network model, entity set model



## Gamma DB

<img src="week2_assets/gamma_process_architecture.png" alt="architecture" width="550">

* Data declustering

* Join process: build + probe (split table)

* Hybrid hash join

  * Parallel version: partitioning split table + joining split table

  <img src="week2_assets/parallel_hybrid_hash_join.png" alt="architecture" width="650">

* Chained declustering

* Concurrency control

  * local lock manager + deadlock detector (transaction wait-for-graph)
  * centralized deadlock detector



## Foundation DB

<img src="week4_assets/architecture.png" alt="architecture" width="500">

* OCC + MVCC

* Recovery: LSN, KCV, DV, PEV, RV

  <img src="week4_assets/recovery.png" alt="strict_serial" width="600">



## HRPS

* $$ RT(M) = \frac{CPU + Disk + Net}{M} + M \cdot CP + F\times CS$$
* $$FC=\frac{TuplesPerQ_{ave}}{M}$$
* Four factors for deciding the level of parallelism (M)



## BG Benchmark

<img src="week6_assets/SoAR.png" alt="architecture" width="550">

<img src="week6_assets/socialite.png" alt="architecture" width="500">

* SLA



## Grasper

<img src="week7_assets/network_architecture.png" alt="architecture" width="550">

* Ethernet / RDMA / IPoIB

  * RDMA: one-side, RecvBufs & SendBufs

* Physical design of data:

  <img src="week7_assets/data_physical_design.png" alt="architecture" width="600">

* Message dispatching: spawn, merge, transfer, split

* Query flow types: sequential (transfer), branch (spawn + merge), barrier (merge)

* Expert Model

  * Adaptive Parallelism Control
  * Tailored optimization
  * Thread binding & load balancing

* Workflow

  <img src="week7_assets/workflow.png" alt="architecture" width="400">

* Work stealing: same thread region -> same NUMA node -> other NUMA nodes