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

As there is no in‑place conversion from RAIDZ1 to RAIDZ2, the supported approach is to backup the data, create a new pool with the new configuration and restore the data. However, this means you will have to buy/get ALL the disks needed for the storage space during the transition, which is not always feasible (and it really depends on the budget and available hardware).

And this is the case for me: as I only have the budget for one more disk, I cannot have all the disks needed for both RAIDZ1 and RAIDZ2 during the transition. This is a common issue for homelabbers, who often have limited resources and need to make the most of what they have.

## The workaround

Luckily, there is a workaround that unblocks reshaping the pool without needing to buy a completely new set of disks. The basics of the workaround are:

- backup the data from the existing RAIDZ1 pool to the new disk
- destroy the existing RAIDZ1 pool
- create a new RAIDZ2 pool with the existing disks and one or more "fake" disks (i.e. files that will be used as virtual disks)
- copy the data from the the new disk to the new pool
- once the data is copied, remove the fake disk(s) from the pool and replace them with the real disks (if you have more than 4 disks, you can add them one by one, replacing the fake disk(s) each time)

This plan allows the transition without needing to have ALL the disks for both RAIDZ1 and RAIDZ2 during the transition, but it does require some careful planning and execution. In particular, you need to ensure that you have a reliable backup of the data in case anything goes wrong.

> [!WARNING]
> I took this approach as an experiment, and it worked for me. Keep in mind that I've worked with non critical data, so I was not risking anything important. If you have critical data, I would recommend against this approach, as it adds an extra layer of risk to the process.

## Execution

### 1. Add the new disk and create a temporary pool

- Physically install the new disk and reboot if needed so TrueNAS sees it.
- In TrueNAS UI, check Storage → Disks, identify the new drive (e.g. `da4` or `sdd`) by serial/model. ​
- Create a temporary single‑disk pool (call it `temptank` for example):
  - Storage → Pools → Add → Create new pool.
  - Add only the new disk as a single‑disk vdev.
  - Disable any special/dedup vdevs; just a plain pool.
  - Create datasets mirroring your current layout (same names) to simplify send/receive (e.g. `temptank/data` if your current pool has `tank/data`). This is optional but helps keep things organized.

### 2. Copy data from tank RAIDZ1 → `temptank`

Use ZFS send/receive so you preserve snapshots, permissions, and dataset properties.

- On the TrueNAS shell, snapshot all datasets on `tank`:

  ```bash
  zfs snapshot -r tank@pre-migration
  ```

- Replicate everything to `temptank`:

  ```bash
  zfs send -R tank@pre-migration | zfs recv -F temptank
  ```

  - `-R` sends all descendant datasets and properties.
  - `-F` forces rollback on the destination if needed.​

- Verify:

  ```bash
  zfs list tank
  zfs list temptank
  ```

  Spot‑check a few directories via SMB/NFS or shell to be sure the data and permissions look correct.

- Run a scrub on `temptank` to ensure data integrity before proceeding:
  ```bash
  zpool scrub temptank
  zpool status temptank
  ```
  Wait until it completes cleanly.

At this point, you have a consistent copy of all data on the single 8TB pool
