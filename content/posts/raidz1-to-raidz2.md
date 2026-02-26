---
date: "2026-02-21"
draft: true
title: "Moving RAIDZ1 to RAIDZ2 with fake disk(s)"
tags: ["TrueNAS", "ZFS", "homelab"]
showReadingTime: true
---

> [!TIP]
> This post is written around TrueNAS SCALE 25.04.x, running `zfs-2.3.0-1`and
> `zfs-kmod-2.3.0-1` but the general approach should work on other versions of TrueNAS and even on FreeBSD/Linux with ZFS, as long as the underlying ZFS features are supported.

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
- In TrueNAS UI, check _Storage › Disks_, identify the new drive (e.g. `da4` or `sdd`) by serial/model.
- Create a temporary single‑disk pool (call it `temp8t` for example):
  - _Storage › Pools › Add › Create new pool_.
  - Add only the new disk as a single‑disk vdev.
  - Disable any special/dedup vdevs; just a plain pool.
  - Create datasets mirroring your current layout (same names) to simplify send/receive (e.g. `temp8t/data` if your current pool has `tank/data`). This is optional but helps keep things organized.

### 2. Copy data from `tank` RAIDZ1 to `temp8t`

Use ZFS send/receive so you preserve snapshots, permissions, and dataset properties.

> [!WARNING]
> The following commands are meant to be run with `sudo`.

- On the TrueNAS shell, snapshot all datasets on `tank`:

  ```bash
  zfs snapshot -r tank@pre-migration
  ```

- Replicate everything to `temp8t`:

  ```bash
  zfs send -R tank@pre-migration | zfs recv -F temp8t
  ```

  - `-R` sends all descendant datasets and properties.
  - `-F` forces rollback on the destination if needed.​

- Verify:

  ```bash
  zfs list tank
  zfs list temp8t
  ```

  Spot‑check a few directories via SMB/NFS or shell to be sure the data and permissions look correct.

- Run a scrub on `temp8t` to ensure data integrity before proceeding:
  ```bash
  zpool scrub temp8t
  zpool status temp8t
  ```
  Wait until it completes cleanly.

At this point, you have a consistent copy of all data on the single 8TB pool.

> [!TIP]
> TrueNAS SCALE 25.x has a UI for creating replication tasks that handle ZFS send/receive, including local replication between pools on the same system.
> This replicates your recursive snapshot exactly as in the shell commands (`zfs snapshot -r + zfs send -R | zfs recv -F`).
> Go to _Data Protection › Replication Tasks › Add_ (or _Advanced Replication Creation_ for full options).

### 3. Destroy the old RAIDZ1 tank pool

Stop services and shares that use tank (SMB, NFS, jails, apps, VMs) in the TrueNAS UI.

- Export/destroy tank via the UI: _Storage › Pools › tank › three‑dot menu › Export/Disconnect › Destroy_.

- Confirm the three original disks are now “available” in _Storage › Disks_. You should see the three disks that were part of the RAIDZ1 pool as free disks.

### 4. Create a degraded RAIDZ2 pool with a fake disk

Goal: build the final RAIDZ2 vdev as 4×8TB, but initially use 3 real disks + 1 fake (sparse file).

#### 4.1 Create the fake disk file

> [!NOTE]
> The `temp8t` pool may have been create as _ready-only_ during the replication step, so you may need to set it to _read/write_ before creating the fake disk file.

Use a dataset on `temp8t` for the sparse file so nothing else is at risk when it fills.

- Create a dedicated dataset:

  ```bash
  zfs create /mnt/temp8t/fake
  ```

- Create an 8TB sparse file:

  ```bash
  truncate -s 8T /mnt/temp8t/fake/fake8t.img
  ```

- Create a _loop_ device for that file:

  ```bash
  losetup -f /mnt/temp8t/fake/fake8t.img
  losetup -a   # find /dev/loopX used
  ```

Now you have a pseudo‑disk (loopX) of size 8TB acting as the fourth member.

#### 4.2 Create the new RAIDZ2 tank pool

- Identify the three real 8TB disks (e.g. /dev/da0 /dev/da1 /dev/da2).

- Create the RAIDZ2 pool from shell (easier to be explicit):

  ```bash
  zpool create -f -o ashift=12 tank raidz2 \
    /dev/da0 /dev/da1 /dev/da2 /dev/loop0
  ```

  - `ashift=12` is recommended for 4K‑sector disks.
  - replace `loop0` with the actual loop device.

- Confirm:
  ```bash
  zpool status tank
  zpool list
  ```
  You now have your final RAIDZ2 layout, with one member being a fake disk. The pool should show as ONLINE.

### 5. Copy data from temp8t › RAIDZ2 tank

Again, using ZFS send/receive (or via GUI)

- Snapshot the temporary pool:
  ```bash
  zfs snapshot -r temp8t@to-tank
  ```
- Send everything to tank:

  ```bash
  zfs send -R temp8t@to-tank | zfs recv -F tank
  ```

- Verify:

  ```bash
  zfs list tank
  ```

  Validate a few directories and permissions via UI/shell.
  ​

- Scrub `tank`:

  ```bash
  zpool scrub tank
  zpool status tank
  ```

  Confirm no errors.
  ​
  At this stage, your data lives on the RAIDZ2 pool, but one vdev member is the fake disk.

### 6. Destroy `temp8t` and replace the fake disk with the real 8TB

Once you’re fully confident in the new tank contents:

- Stop any remaining services using `temp8t` (if any).

- Destroy `temp8t` via the UI (Storage › Pools › `temp8t` › Export/Disconnect › Destroy).

- Identify the fake member in `zpool status tank` (it will show as /dev/loopX or /temp8t/fake/fake8t.img).

- Replace fake with the real disk:

  ```bash
  zpool replace tank /dev/loopX /dev/sdX
  ```

  - Replace `/dev/sdX` with the new device.
  - This triggers a resilver onto the real disk.

- Monitor resilver:

  ```bash
  zpool status tank
  ```

  Wait until resilver completes and pool is ONLINE with all four real disks.
  ​

- After resilver:
  - Detach and remove the fake file/loop device:

    ```bash
    losetup -d /dev/loopX
    rm /temp8t/fake/fake8t.img
    ```

The pool is now a proper 4×8TB RAIDZ2 using only your four physical drives.
