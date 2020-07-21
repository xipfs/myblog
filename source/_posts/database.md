---
title: 数据库学习指南
date: 2020-07-21 12:11:54
author: Rootkit
top: false
mathjax: false
categories: Database
tags:
  - Database
---





# Databases

## General

- [x] 🎥 [**CMU 15-445 Intro to Database Systems**](https://www.youtube.com/playlist?list=PLSE8ODhjZXjbohkNBWQs_otTrBTrjyohi) (A Pavlo 2019)
- [x] 🎥 [**CMU 15-721 Advanced Database Systems**](https://www.youtube.com/playlist?list=PLSE8ODhjZXjasmrEd2_Yi1deeE360zv5O) (A Pavlo 2020)
- [x] 📖 [**Database Internals**](https://www.databass.dev) (A Petrov 2019)
- [x] 📖 [**Designing Data-Intensive Applications**](https://dataintensive.net/) (M Kleppmann 2017)
- [x] 📄 [**Readings in Database Systems**](http://www.redbook.io) (P Bailis, JM Hellerstein, M Stonebraker) _"The Red Book"_
- [x] 📄 [Architecture of a Database System](http://db.cs.berkeley.edu/papers/fntdb07-architecture.pdf) (JM Hellerstein, M Stonebraker, J Hamilton 2007)
- [x] 📖 [Data and Reality](https://www.amazon.com/Data-Reality-Perspective-Perceiving-Information/dp/1935504215) (W Kent, S Hoberman 2012)
- [x] 📖 [Database System Concepts](https://www.db-book.com/db7/index.html) (A Silberschatz, HF Korth, S Sudarshan 2019)
- [x] 📖 [Fundamentals of Database Systems](https://www.amazon.com/Fundamentals-Database-Systems-Ramez-Elmasri/dp/0133970779) (R Elmasri, SB Navathe 2015)
- [x] 📖 [Making Databases Work: the Pragmatic Wisdom of Michael Stonebraker](https://dl.acm.org/doi/book/10.1145/3226595) (ML Brodie 2018)
- [x] 🎥 [UC Berkeley CS186 Introduction to Database Systems](https://archive.org/details/UCBerkeley_Course_Computer_Science_186#) (J Hellerstein 2012)
- [x] 📄 [What's Really New with NewSQL?](https://db.cs.cmu.edu/papers/2016/pavlo-newsql-sigmodrec2016.pdf) (A Pavlo, M Aslett 2016)

## Transactions

- [x] 📄 [**A Critique of ANSI SQL Isolation Levels**](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf) (H Berenson et al 1995)
- [x] 🔗 [**Consistency Models**](https://jepsen.io/consistency) (Jepsen 2016)
- [x] 📄 [**Generalized Isolation Level Definitions**](http://pmg.csail.mit.edu/papers/icde00.pdf) (A Adya, B Liskov, P ONeil 2000)
- [x] 📖 [**Transaction Processing: Concepts and Techniques**](https://www.amazon.com/Transaction-Processing-Concepts-Techniques-Management/dp/1558601902#customerReviews) (J Gray, A Reuter 1992)
- [x] 📄 [ACIDRain: Concurrency-Related Attacks on Database-Backed Web Applications](http://www.bailis.org/papers/acidrain-sigmod2017.pdf) (P Bailis, T Warszawski 2017)
- [x] 📄 [An Empirical Evaluation of In-Memory Multi-Version Concurrency Control](https://yingjunwu.github.io/papers/vldb2017.pdf) (Y Wu et al 2017)
- [x] 📄 [Building Consistent Transactions with Inconsistent Replication](http://delivery.acm.org/10.1145/2820000/2815404/p263-zhang.pdf) (I Zhang et al 2015)
- [x] 📄 [Calvin: Fast Distributed Transactions for Partitioned Database Systems](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf) (DJ Abadi et al 2012) _"The Calvin paper"_
- [x] 📄 [Consistency in Non-Transactional Distributed Storage Systems](https://arxiv.org/pdf/1512.00168.pdf) (P Viotti, M Vukolić 2016)
- [x] 📄 [Highly Available Transactions: Virtues and Limitations](http://www.vldb.org/pvldb/vol7/p181-bailis.pdf) (P Bailis, JM Hellerstein et al 2013)
- [x] 📄 [Naming and Synchronization in a Decentralized Computer System](https://dspace.mit.edu/bitstream/handle/1721.1/16279/05331643-MIT.pdf) (DP Reed 1978) _"The MVCC paper"_
- [x] 📄 [Rethinking Serializable Multiversion Concurrency Control](http://www.jmfaleiro.com/pubs/multiversion-vldb2015.pdf) (JM Faleiro, DJ Abadi 2015)
- [x] 📄 [Scalable Atomic Visibility with RAMP Transactions](http://www.bailis.org/papers/ramp-sigmod2014.pdf) (P Bailis et al 2014)
- [x] 📄 [Serializable Isolation for Snapshot Databases](https://courses.cs.washington.edu/courses/cse444/08au/544M/READING-LIST/fekete-sigmod2008.pdf) (MJ Cahill, U Röhm, AD Fekete 2008)
- [x] 💬 [What Does Write Skew Look Like](http://justinjaffray.com/what-does-write-skew-look-like/) (J Jaffray 2018)

## Queries

- [x] 📄 [Access Path Selection in a Relational Database Management System](https://www2.cs.duke.edu/courses/compsci516/cps216/spring03/papers/selinger-etal-1979.pdf) (PG Selinger et al 1979)
- [x] 📄 [An Overview of Query Optimization in Relational Systems](https://web.stanford.edu/class/cs345d-01/rl/chaudhuri98.pdf) (S Chaudhuri 1998)
- [x] 📖 [Building Query Compilers](http://pi3.informatik.uni-mannheim.de/~moer/querycompiler.pdf) (G Moerkotte 2014)
- [x] 📄 [How to Architect a Query Compiler, Revisited](https://www.cs.purdue.edu/homes/rompf/papers/tahboub-sigmod18.pdf) (RY Tahboub, GM Essertel, T Rompf 2018)
- [x] 📄 [Optimization of Queries with User-Defined Predicates](http://www.vldb.org/conf/1996/P087.PDF) (S Chaudhur, K Shim 1999)
- [x] 📄 [The Cascades Framework for Query Optimization](https://www.cse.iitb.ac.in/infolab/Data/Courses/CS632/Papers/Cascades-graefe.pdf) (G Graefe 1995)

## Storage

- [x] 📄 [The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases](https://db.in.tum.de/~leis/papers/ART.pdf) (V Leis, A Kemper, T Neumann 2013)
- [x] 📄 [Aries: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging](https://cs.stanford.edu/people/chrismre/cs345/rl/aries.pdf) (C Mohan et al 1992)
- [x] 📄 [bLSM: A General Purpose Log Structured Merge Tree](http://www.eecs.harvard.edu/~margo/cs165/papers/gp-lsm.pdf) (R Sears, R Ramakrishnan 2012)
- [x] 📄 [Building a Bw-Tree Takes More Than Just Buzz Words](http://www.cs.cmu.edu/~huanche1/publications/open_bwtree.pdf) (Z Wang et al 2018)
- [x] 📄 [Jungle: Towards Dynamically Adjustable Key-Value Store by Combining LSM-Tree and Copy-On-Write B+-Tree](https://greensky00.github.io/pdf/jungle_hotstorage19.pdf) (JS Ahn et al 2019)
- [x] 📄 [The Bw-Tree: A B-tree for New Hardware Platforms](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/bw-tree-icde2013-final.pdf) (JJ Levandoski, DB Lomet, S Sengupta 2013)
- [x] 📄 [The Log-Structured Merge Tree](https://www.cs.umb.edu/~poneil/lsmtree.pdf) (P O'Neil, E Cheng, D Gawklik, E O'Neil 1996)
- [x] 📄 [The Ubiquitous B-Tree](http://cgi.di.uoa.gr/~ad/M149/ubiquitous_btree.pdf) (D Comer 1979)
- [x] 📄 [WiscKey: Separating Keys from Values in SSD-Conscious Storage](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf) (L Lu, TS Pillai, AC Arpaci-Dusseau, RH Arpaci-Dusseau 2016)

## Verification

- [x] 🎥 [SQLancer: Finding Logic Bugs in Database Management Systems](https://www.youtube.com/watch?v=Np46NQ6lqP8) (M Rigger 2020)
- [x] 📄 [Torturing Databases for Fun and Profit](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-zheng_mai.pdf) (M Zheng et al 2014)

## Vendors

### CockroachDB

- [x] 🔗 [**Architecture Overview**](https://www.cockroachlabs.com/docs/stable/architecture/overview.html)
- [x] 📄 [**CockroachDB: The Resilient Geo-Distributed SQL Database**](https://dl.acm.org/doi/pdf/10.1145/3318464.3386134) (R Taft et al 2020)
- [x] 💬 [**Serializable, Lockless, Distributed: Isolation in CockroachDB**](https://www.cockroachlabs.com/blog/serializable-lockless-distributed-isolation-cockroachdb/) (M Tracy 2016)
- [x] 💬 [How We Built a Cost-Based SQL Optimizer](https://www.cockroachlabs.com/blog/building-cost-based-sql-optimizer/) (A Kimball 2018)
- [x] 💬 [Living Without Atomic Clocks](https://www.cockroachlabs.com/blog/living-without-atomic-clocks/) (S Kimball 2016)

### FaunaDB

- [ ] 💬 [Spanner vs. Calvin: Distributed Consistency at Scale](https://fauna.com/blog/distributed-consistency-at-scale-spanner-vs-calvin) (DJ Abadi 2017)
- [x] 💬 [Time-Traveling Databases: Exploring Temporality in FaunaDB](https://fauna.com/blog/time-traveling-databases) (M Freels 2016)
- [x] 💬 [Unifying Relational, Document, Graph, and Temporal Data Models](https://fauna.com/blog/unifying-relational-document-graph-and-temporal-data-models) (C Anderson 2018)

### Google Bigtable

- [x] 📄 [**Bigtable: A Distributed Storage System for Structured Data**](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf) (F Chang et al 2006) _"The Bigtable paper"_

### Google F1

- [x] 📄 [**F1: A Distributed SQL Database That Scales**](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/41344.pdf) _"The F1 paper"_

### Google Spanner

- [x] 📄 [**Spanner: Google's Globally-Distributed Database**](http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/39966.pdf) (J Corbett et al 2012) _"The Spanner paper"_
- [x] 📄 [Spanner: Becoming a SQL System](https://dl.acm.org/doi/pdf/10.1145/3035918.3056103) (DF Bacon et al 2017)
- [x] 📄 [Spanner, TrueTime & The CAP Theorem](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45855.pdf) (E Brewer 2017)

### PostgreSQL

- [x] 📄 [**Looking Back at Postgres**](https://arxiv.org/pdf/1901.01973.pdf) (JM Hellerstein 2019)
- [x] 💬 [How Postgres Makes Transactions Atomic](https://brandur.org/postgres-atomicity) (B Leach 2017)
- [x] 📄 [Serializable Snapshot Isolation in PostgreSQL](https://drkp.net/papers/ssi-vldb12.pdf) (DRK Ports, K Grittner 2012)
- [x] 📖 [The Internals of PostgreSQL](http://www.interdb.jp/pg/) (H Suzuki 2015)

