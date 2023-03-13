# Map of the BRT changes in openZFS

## Changes are here: [Pull Requeset - Block Cloning #13392](https://github.com/openzfs/zfs/pull/13392/files)

## Overall

"Block Cloning allows to clone a file (or a subset of its blocks) into another (or the same) file by just creating additional references to the data blocks without copying the data itself. Block Cloning can be described as a fast, manual deduplication."

## Description of Effect of changes

"In many ways Block Cloning is similar to the existing deduplication, but there are some important differences:

- Deduplication is automatic and Block Cloning is not - one has to use a dedicated system call(s) to clone the given file/blocks.
Deduplication keeps all data blocks in its table, even those referenced just ones. Block Cloning creates an entry in its tables only when there are at least two references to the given data block. If the block was never explicitly cloned or the second to last reference was dropped, there will be neither space nor performance overhead.
- Deduplication needs data to work - one needs to pass real data to the write(2) syscall, so hash can be calculated. Block Cloning doesn't require data, just block pointers to the data, so it is extremely fast, as we pay neither the cost of reading the data, nor the cost of writing the data - we operate exclusively on metadata.
- If the D (dedup) bit is not set in the block pointer, it means that the block is not in the dedup table (DDT) and we won't consult the DDT when we need to free the block. Block Cloning must be consulted on every free, because we cannot modify the source BP (eg. by setting something similar to the D bit), thus we have no hint if the block is in the Block Reference Table (BRT), so we need to look into the BRT.
- The BRT entry is much smaller than the DDT entry - for BRT we only store 64bit offset and 64bit reference counter.
- Dedup keys are cryptographic hashes, so two blocks that are close to each other on disk are most likely in totally different parts of the DDT. The BRT entry keys are offsets into a single top-level VDEV, so data blocks from one file should have BRT entries close to each other.
- Scrub will only do a single pass over a block that is referenced multiple times in the DDT. Unfortunately it is not currently (if at all) possible with Block Cloning and block referenced multiple times will be scrubbed multiple times.
- Deduplication requires cryptographically strong hash as a checksum or additional data verification. Block Cloning works with any checksum algorithm or even with checksumming disabled.
- As mentioned above, the BRT entries are much smaller than the DDT entries. To uniquely identify a block we just need its vdevid and offset. We also need to maintain a reference counter. The vdevid will often repeat, as there is a small number of top-level VDEVs and a large number of blocks stored in each VDEV. We take advantage of that to reduce the BRT entry size further by maintaining one BRT for each top-level VDEV, so we can then have only offset and counter as the BRT entry."

## `cmd/zdb/zdb_il.c`

- Changes are largely unimportant
- Defines `zil_prt_rec_clone_range()`
  - Not any commenting to help figure out what it does
  - Return to Later
- Modifies the `zil_rec_info` to include a new field struct.

## `cmd/ztest.c`

- Increases maximum log length
- Related to `cmd/zdb/zdb_il.c` changes

## `include/Makefile.am`

- Just adds the make of the headerfiles.

## Header Files

- Updating definitions in the following files: `include/os/freebsd/zfs/sys/zfs_znode_impl.h`, `include/os/linux/kernel/linux/mod_compat.h`, `include/os/linux/zfs/sys/zfs_znode_impl.h`, `include/sys/bitmap.h`, `include/sys/brt.h`, `include/sys/dbuf.h`,`include/sys/ddt.h`, `include/sys/dmu.h`, `include/sys/fs/zfs.h`, `include/sys/spa_impl.h`, `include/sys/zfs_debug.h`, `include/sys/zfs_vnops.h`, `include/sys/zfs_znode.h`, `include/sys/zil.h`, `include/sys/zio.h`, `include/sys/zio_impl.h` and `include/zfeature_common.h`
- Interesting chages in: `include/sys/brt.h`, `include/sys/zfs_vnops.h`, and `include/sys/zfs_znode.h`.

## `module/os/freebsd/zfs/zfs_vnops_os.c`

- Free BSD specific copy file code
- Implements the `vop_copy_file_range_args` struct and `zfs_freebsd_copy_file_range()`

## `module/os/freebsd/zfs/zfs_znode.c`

- Defines `zfs_rlimit_fsize()`.

## `module/zcommon/zfeature_common.c`

- Registers the block cloning feature? Not sure what or why this exists

## `module/zcommon/zpool_prop.c`

- When a zpool is registered with using block referencing then it has different setup done
- Essentially this and `module/zcommon/zfeature_common.c` are allowing the pools to know that they are configured with block referencing enabled.

## `module/zfs/brt.c`

<mark> - TODO</mark>

## `module/zfs/dbuf.c`

- updates `dbuf_read_hole()` function to take a new block pointer argument
  - This function handle reads on dbufs that are holes, if necessary.  This function  requires that the dbuf's mutex is held. Returns success (0) if action taken, ENOENT if no action was taken.
- Further changes to `dbuf_read_impl()`
  - This function "Drops db_mtx and the parent lock specified by dblt and tag before returning."


## `module/zfs/ddt.c`

- Defines `ddt_addref()`. "This function is used by Block Cloning (brt.c) to increase reference counter for the DDT entry if the block is already in DDT. Return false if the block, despite having the D bit set, is not present in the DDT. Currently this is not possible but might be in the future. See the comment below."
  - Uses the function `ddt_lookup()` to find the `ddt_entry_t *` struct in the table
  - `BP_GET_NDVAS` (defined in `include/sys/spa.h`) is used to find the physical address (dde is the deduplication table entry found with the `ddt_lookup()` function mentioned above)

    ```c
    ddt_phys_t *ddp;
    ddp = &dde->dde_phys[BP_GET_NDVAS(bp)];
    ```

  - Deduplication table has to be exited with `ddt_exit()`

## `module/zfs/dmu.c`

- GENERALLY CONFUSED AS TO WHAT IS HAPPENING IN THE DMU HERE (I KNOW THAT IT IS IMPORTANT THOUGH)
- DMU is the data management unit. It is responsible for managing dnodes. Dnodes  also  describe  objects  in  the  MOS  layer  such  as  the  objects  that  represent filesystems, snapshots, clones, ZVOLs, space maps, property lists, and dead-block lists (referred to as deadlists).
- The DMU keeps track of the location of each block on the pool and provides features such as checksumming and redundancy to ensure data integrity and availability.

1. Block allocation and management: The DMU is responsible for allocating and managing blocks on the pool, including determining which blocks are free, which are in use, and which are part of redundant data sets.
2. Data compression: The DMU supports data compression, which can help reduce the amount of space needed to store data on the pool.
3. Checksumming and error correction: The DMU uses checksums to ensure data integrity, and can detect and correct errors in stored data.
4. Snapshot management: The DMU manages the creation and deletion of snapshots, which are read-only copies of the pool at a particular point in time.
5. Resilvering and rebuilding: The DMU is responsible for rebuilding redundant data sets in the event of disk failures, and for resilvering data when disks are added or replaced in the pool.

- Added `dmu_read_l0_bps()`
  - `dmu_read_l0_bps(objset_t *os, uint64_t object, uint64_t offset, uint64_t length, dmu_tx_t *tx, blkptr_t *bps, size_t *nbpsp)`
  - Confused about what this function actually does.

- Also adds `dmu_brt_clone()`

## `module/zfs/zfs_ioctl.c`

- Surprisingly nothing done, just updated copyright.

## `module/zfs/spa.c`

- SPA is the storage pool allocator
  - The SPA is responsible for allocating the storage space in a pool of disks.
- With the `spa_t` struct you are able to get the root vdev by doing `vdev_t *rvd = spa->spa_root_vdev;` 
- Then loads the brt for that spa with `brt_load()`
- Updates also allow for the creation of a BRT in a specific spa object with `brt_create()`.
- There is also some delay becuase the spa is triggered with a transaction group in `brt_pending_apply()`

## `module/zfs/zfs_vnops.c`

- vnops() stands for "Virtual Node Operations" and is a term commonly used in the context of the FreeBSD operating system. In FreeBSD, "virtual nodes" are used to represent various system resources, such as files, directories, sockets, and so on. The vnops interface provides a set of functions that are used by the operating system to manipulate these virtual nodes.
- the vnops functions include operations such as opening and closing virtual nodes, reading from and writing to them, seeking within them, setting their attributes, and so on. These functions are typically implemented by file system drivers that are responsible for managing the virtual nodes that correspond to their respective file systems.
- overall, the vnops interface is an important part of the FreeBSD kernel that enables the operating system to provide a uniform and consistent way of interacting with different types of system resources.
- The changes done to the BRT add a handful of these operations
- First new operation in `zfs_enter_two()`
  - It takes in two `zfsvfs_t` structures
  - Swaps them around so that the lower one is opened first
  - Then uses`zfs_enter()` to open both vfs.
- Second new function is `zfs_exit_two()`. It jsut exits two vfs's at once (making sure they are not the same)

- CRUCIAL NEW FUNCTION `zfs_clone_range()`
  - We split each clone request in chunks that can fit into a single ZIL log entry. Each ZIL log entry can fit 130816 bytes for a block cloning operation (see zil_max_log_data() and zfs_log_clone_range()). This gives us room for storing 1022 block pointers. On success, the function return the number of bytes copied in *lenp. Note, it doesn't return how much bytes are left to be copied."
  - Important operations `inos = inzfsvfs->z_os;`
  and `outos = outzfsvfs->z_os;`
  - Determines that both storage pools are the same (can not clone across pools) (inzp and outzp are pointers to `znode_t`).

    ```c
    inzfsvfs = ZTOZSB(inzp);
    outzfsvfs = ZTOZSB(outzp);
    inos = inzfsvfs->z_os;
    outos = outzfsvfs->z_os;

    if (dmu_objset_spa(inos) != dmu_objset_spa(outos)) {
      zfs_exit_two(inzfsvfs, outzfsvfs, FTAG);
      return (SET_ERROR(EXDEV));
    }
    ```

  - Because we can copy across datasets we need to be able to potentially open two at the same time. This is the reason for having the `zfs_enter_two()` and `zfs_exit_two()` functions.
  - Checks that the storage pool has block cloning feature enabled with:

    ```c
    (!spa_feature_is_enabled(dmu_objset_spa(outos) SPA_FEATURE_BLOCK_CLONING))
    ```

  - Source file can not be in quarantine, this is because the source file's flags are not coppied over. This is done with:

    ```c
    inzp->z_pflags & ZFS_AV_QUARANTINED
    ```

  - Callers might not be able to detect properly that we are read-only, so check it explicitly here. with `zfs_is_readonly(outzfsvfs)`
  - If immutable or not appending then fail. Intentionally allow ZFS_READONLY through here.

    ```c
    (outzp->z_pflags & ZFS_IMMUTABLE) != 0)
    ```

  - Cant be cloning the same file `(inzp == outzp)`

  - Block sizes of soruce and desitination must be the same. Checked with the following:
  
    ```c
    (inblksz != outzp->z_blksz && outzp->z_size > inblksz)
    ```
  
  - Offsets and len must be at block boundaries.

    ```c
    ((inoff % inblksz) != 0 || (outoff % inblksz) != 0)
    ```

  - Length must be multipe of blksz, except for the end of the file.

    ```c
    ((len % inblksz) != 0 && (len < inzp->z_size - inoff || len < outzp->z_size - outoff))
    ```

  - Also confirms that the cloned fiel offesets are not more than the maxofset
  
    ```c
    (inoff >= MAXOFFSET_T || outoff >= MAXOFFSET_T)
    ```

  - uses the `SA_ADD_BULK_ATTR()` funciton
    - Not sure what it does... Confused...
  
  - Allocates sapce for the block pointers with: `kmem_alloc()`
    - Passing it the size of a block pointer mulitplied by the most blocks possible in one batch `maxblocks`
    - `maxblocks = zil_max_log_data(zilog, sizeof (lr_clone_range_t)) sizeof (bps[0]);`
  - Starts a transaction with `dmu_tx_create(outos)`

  - Functions with the `dmu` that I dont understand
    - `dmu_tx_hold_sa()`
    - `dmu_tx_hold_write_by_dnode()`
    - `zfs_sa_upgrade_txholds()`
    - `dmu_tx_assign()`
    - `dmu_brt_clone()`
    - `dmu_tx_commit()`

  - Clears additional information with `zfs_clear_setid_bits_if_necessary()` if needed.
- Usual pattern would be to call zfs_clone_range() from zfs_replay_clone(), but we cannot do that, because when replaying we don't have source znode available. This is why we need a dedicated replay function. `zfs_clone_range_replay()`
  - Does not acutally call `zfs_clone_range()`.
  - Only has one of the znode pointers
  - Does a lot of the same checks but just an the one storage pool
    - Checks if things are enabled and nothing is illegal
  - Starts transation the same way with `dmu_tx_create(zfsvfs->z_os)`
  - Calls many of the same DMU functions that I have listed above

## `module/zfs/zil.c`

- Logging stuff for the brt (ZIL is the ZFS intent log)
- `zil_claim_clone_range()`

## `module/zfs/zio.c`

- Writes the `zio_write_override()` function
- Also has code to free the block reference table `zio_brt_free()`
- 

## `module/zfs/zvol.c`

- More code around "replay," not sure what that is
- Defines `zvol_replay_clone_range()`

## `tests/zfs-tests/tests/functional/cli_root/zpool_get/zpool_get.cfg`

- Test file

