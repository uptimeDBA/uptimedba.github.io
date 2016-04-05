---
title: "Oracle Physical or Logical Replication? – Consider the options"
categories:
- oracle
keywords: oracle, replication
summary: "Oracle physical and logical replication are two very different approaches but they may offer alternative solutions to the same problem."
thumb: questionmark_108x108.png
---

So you’ve identified a business need to replicate your data for some reason.

If you’re trying to provide a read-only copy of your data closer to your end users for performance or to reduce the load on your primary system, or as part of your HA/DR strategy, then you have a couple of choices to make. Oracle physical and logical replication are two very different approaches but they may offer alternatives to the same solution.

Oracle offers a read-only, physical replication solution in the form of Oracle Active Data Guard and a logical replication solution in the form of Oracle GoldenGate.
For Active Data Guard, the databases needs to be >= 11gR1. (10g had DataGuard with read only access, but not at the same time. You had to stop the redo log apply, effectively freezing the secondary database at a point in time, before opening the database in read-only mode. Afterwards the database was restarted in redo log apply mode and the outstanding logs applied to the secondary to bring it up to date again.)

The operating system, database type and version all need to be the same on both the source and target.
Access to the target needs to be 100% read only. Reports run from the read-only copy can’t create or change data in the target.
The replication lag (redo apply lag) needs to be within acceptable parameters for reporting from the data in a timely fashion.

So when would you choose Active Data Guard over GoldenGate or some other logical replication solution like Dbvisit Replicate or Dell SharePlex?
Some applications have data and data structures that are not well suited to logical replication. They may contain datatypes not supported or there may be tables without a unique identifier that slows down the logical replication to unacceptable levels. Physical replication doesn’t suffer from any of these problems because the replication works at a lower (binary) level.
If all you require is a full 1:1 copy of the data, then you probably won’t require all the selection, filtering, transformation, and heterogeneous capabilities that the logical replication products offer.

As an aside, the per processor license for Active Data Guard is also (slightly) cheaper than GoldenGate.
Physical replication may not always be an option for your data replication needs but under certain conditions it might fit, so it pays to check the products capabilities against your requirements.
