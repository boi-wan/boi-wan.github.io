---
date: "2026-02-21"
draft: true
title: "Moving RAIDZ1 to RAIDZ2 with fake disk(s)"
tags: ["TrueNAS", "ZFS", "homelab"]
showReadingTime: true
---

I have been running TrueNAS in my homelab for a while now. Recently, I decided to reshape the pools, both for better redundancy and to make better use of the disks I have available.

The biggest pool I have is a RAIDZ1 with 3\*8TB disks. It was fitting my needs, until they changed. I started to have more data to store, and so I decided to move to 4\*8TB disks in RAIDZ2. This way, I can have more storage space and better protection against disk failures.

> [!NOTE]
> RAIDZ1 can only tolerate the failure of one disk, while RAIDZ2 can tolerate the failure of two disks. This means that if I were to stick with RAIDZ1 and one of my disks were to fail, I would be at risk of losing all my data.

> [!NOTE]
> There is no in‑place conversion from RAIDZ1 to RAIDZ2; the supported approach is to backup the data, create a new pool with the new configuration and restore the data. However, this means to get twice the storage space needed for the data during the transition, which is not always feasible (and it really depends on the budget and available hardware).

code example:

```scala
val oldPool = "raidz1_pool"
val newPool = "raidz2_pool"
// Step 1: Backup the data
// This can be done using zfs send/receive or by copying the data to an external drive
backupData(oldPool, "backup_location")
```

## The workaround

Luckily, there is a workaround that unblocks reshaping the pool without needing to buy a completely new set of disks. The basics of the workaround are:

- ensuring to have a backup of the data (!!!)
- destroy the existing RAIDZ1 pool
- create a new RAIDZ2 pool with the existing disks and one or more "fake" disks (i.e. files that will be used as virtual disks)
- copy the data from the backup to the new pool
- once the data is copied, remove the fake disk(s) from the pool and replace them with the real disks (if you have more than 4 disks, you can add them one by one, replacing the fake disk(s) each time)

This plan allows the transition to RAIDZ2 without needing to have twice the storage space during the transition, but it does require some careful planning and execution. In particular, you need to ensure that you have a reliable backup of the data in case anything goes wrong.

## Backing up the data

Before starting, I made sure to have a backup of all the data. I used a single disk pool to store the backup, which was sufficient for my needs. If you are brave enough, you can use the new disk itself to store the backup, but I would not recommend it as it adds an extra layer of risk to the process.

###
