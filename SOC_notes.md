# STREAM OF CONSCIOUSNESS NOTES:

## Jan 23:
- Goals for the week:
    -  Take look at btrFS stuff
    -  Look back at thread
    - Looking through the fork
    - Plan to meet on Thursday
    -  Postpone till after thursday
        - Attempting to get linux on the machine running ZFS and btrFS

## Jan 24:
- Reading BTRFS_The_Linux_B-tree_Filesystem.pdf -> good amount of similarities to ZFS but fundamentally different hierarchical structure
    -  No concrete mention of -`reflink`s but does a great job outlining btrFS basics and how writing and freeing blocks works
- Re exploring this feature request on ZFS https://github.com/openzfs/zfs/issues/405 and https://github.com/pjd/openzfs/commit/83921c797a65fbc46a72e2a90c40822a988dee28

## Jan 25:
- Working on the "how" presentation
- Updated bibliography with all new sources

## Jan 26:
- Read through feature request thread again https://github.com/openzfs/zfs/issues/405 "COW cp (--reflink) support #405"
    - Explored https://github.com/openzfs/zfs/discussions/4237
    - Also https://github.com/openzfs/zfs/issues/1063 "Please implement "cp --reflink=always" with ZoL #1063"
    - Also https://github.com/openzfs/zfs/issues/2554 "btrfs bedup equivalent #2554"
     - "There's 2 ways a block can be referenced multiple times: dedup and snapshots. You can't "instantly" clone a file because doing it that way doesn't meet either criteria. In contrast BTRFS seems to use some crazy layers of indirection and trees to let it do that."
- Presentation https://www.usenix.org/legacy/event/lsf07/tech/rodeh.pdf
- WAFL implementation https://community.netapp.com/t5/Microsoft-Virtualization-Discussions/What-is-SIS-Clone/td-p/6462
    - "SIS clone is a capability withing Data ONTAP to run the dedupe engine in reverse to clone a file.  Instead of copying all the block, and then removing the duplicate blockd at a later date.  Instead we create a sparse file  (non-allocated) and then SIS clone manipulates the WAFL pointers in the destination file to point to the blocks for the source file. The end result is an exact copy of the source file that is pre-deduped.  The benefit of SIS-Clone is we can manipulate WAFL pointers even faster than we can write block, so the copy operation is an order of magnitude quicker. "
- Already implemented in XFS https://strugglers.net/~andy/blog/2017/01/10/xfs-reflinks-and-deduplication/
- "FAST-Tracking REFLINK and Offline Deduplication, first for LINUX only #13349" https://github.com/openzfs/zfs/issues/13349
- Oracle ZFS has reflinks already? https://blogs.oracle.com/solaris/post/reflink3c-what-is-it-why-do-i-care-and-how-can-i-use-it
- Block Cloning Table branch https://github.com/openzfs/zfs/pull/13392
    - Explanation video https://www.youtube.com/watch?v=hYBgoaQC-vo

- Notes with pete:
    -  Why inst the naive approach sufficient
    -  Maybe dive deep into why the BRT has been implemented when it doesn't need to be

    TODO:
    - (done) How referencing is done 
    - (done) How snapshots do the referencing
    - (done) Grep the source code
    - (done) Find the field in the struct that is the reference count and find every usage of it
    - (done) One last deep dive into why the naive solution doesnt work
    - Get the BRT branch downloaded and compiled -> probably on Linux


## Jan 27:
- How referencing is done
    -  Reread block referencing section of The_Design_and_Implementation_of_the_FreeBSD_Operating_System
        - Space allocation handled by the Storage Pool Allocator (SPA) module (LOOK INTO SOURCE CODE OF)
            - SPA looks up available space in the MOS space map.
                1. Chooses the disk with the most free space
                2. From the fixed-sie space maps describing the space on the disk select the one that is the least fragmented
                3. Allocate a chunk of space with the needed size that is closest to the previous allocation
        - Freeing blocks -> ZFS tracks its used and available blocks using space maps, birth time, and deadliest
        - Deduplication also increments a blocks associated reference count.

- How snapshots do the referencing
    - Reread block referencing section of The_Design_and_Implementation_of_the_FreeBSD_Operating_System
        - Checkpoint to make sure everything is up to date
        - Allocate a new dnode that will represent the snapshot
        -  Copy over the block pointer from the current filesystem being snapshotted to the newly allocated snapshot dnode
        -  Link the new dnode to the head of the filesystem's snapshot list (youngest snapshot)
        - Add snapshot name etc...

- Existing design considerations:
    -  "File-cloning can be thought of as hard-links with copy-on-write properties Use-cases include cloning VM images and other large files  or being able to recover files from snapshots, fast and without additional space Implementation: Dedup comes to mind as the end-functionality is somewhat similar. That said we would like this to be a generic feature of ZFS and not a feature flag. We’d also like to avoid the performance issues that come with dedup. Current idea revolves around a new block-reference table, a mapping of vdev_id+offset to a refcount. The attempt would be to avoid overhead in reads/writes and just put the overhead in when freeing blocks with more than one reference (to make such operations performant we’d probably need to maintain most if not all of our structures/metadata in memory). There are a few implementation questions that we’d need to iron-out. For example, do we expect this to work across datasets? If that’s the case how would it affect send/receive assuming we want to maintain the cloning-property? What about memory? Can’t that table grow too big? We’d probably need to think hard of how the table is managed and how to avoid unpredictable performance."

- Watched https://www.youtube.com/watch?v=hij7PGGjevc talking about 98% done
    - For BRT we dont have a specific bit that we need to check table we have to check for every free.
        - What if we used one of the free bits in the dnode struct
    - Keys are not random so lookup is better and faster
        - vdev is in a range, and keep how many references is in the range
        -  Small 1mb per terabits
        - Helps us know if the block might be in the brt
            - We can immediately tell if it is not

## Jan 29th
- Watched design implementation talk https://www.youtube.com/watch?v=Y9HQ4RbqIEw
    - Could reuse dedup, just read the checksum from block pointers and make a new file pointing to the same checksum
        - "We dont want this to be another dedup"
    - "The problem is of course, if we have a data block on the disk and we have a block pointer pointing at this block, we cannot have a reference counter in this block pointer because, of course, we can not modify block pointer."
    - Powels idea was to create something similar to dedup
    - Things with a reference count of one dominate the dedup table
    - The block reference table will only have entries when their reference count is greater than one
   - In the dedup table they are data hashes so they can not be sorted really.
        - Not good for caching or easy lookup
    - "We have a block of data, and we have a block pointer pointing at the data, and we can not modify that. But once you call this clone file system call. We will read the existing block pointers, create new block pointers, just copying the data from the original block pointers. Storing them in a new file and creating entries in the block reference tables. The entry is only created when the block is actually cloned. So if you don't use the functionally there is no 'price to pay'." 
    - No performance impact on read or write, only when a block pointer is freed do we have to consult to block reference table.
    - "When we free the block we can not really tell if the block has more than one reference" ---- WHYYYYYY
    "Deduplication has the dedup bit set"
    - Deduplication Table entry is 392 bytes
    - Block Reference Table entry is 80 bytes or smaller
        - Plus it is sortable
        - "The block reference table is basically a mapping from the vdev + offset to the refcount. You do a lot better by putting it in a dataset that is sorted
    - One idea for better free performance would be to actually have a bit in the block pointer structure that then triggers us to look into the block reference table
    - To have send/receive work you would need massive post-processing after the fact
  In-memory cost is 10x smaller than DDT
How does snapshots increment block references... It should be THE SAME PROCESS

## Jan 30th

https://docs.google.com/document/d/1w2jv2XVYFmBVvG1EGf-9A5HBVsjAYoLIFZAnWHhV-BM/edit#heading=h.7pnof6czteeh (DEVELOPER NOTES)
Started looking at source code. Trying to find when block references are changed.

- https://github.com/openzfs/zfs/pull/13392
  - for BRT we only store 64bit offset and 64bit reference counter
- What if we added in a reference count 64bits into the block reference structure rather than the block reference table
There are 2, 64 bit spare spaces in each block reference structure
./include/sys/spa.h has the definition of the block pointer structure. Has two 64-bit padding chunks
  - If we use one of these, then the block reference table is irrelevant. We not longer have the negative side effects of freeing a block and therefore the cloning can be turned on by default and therefore be much better than a new deduplication table DDT but for the block pointer table. Now, just when freeing blocks it is as easy as decrementing the block clone reference count.

## Feb 1st

Might need dnodes to point at THE SAME BLOCK POINTER not just a duplicate -> he same block pointer

## Feb 21st

Getting ZFS compiled and running with archlinux: instructions followed:
https://blog.timo.page/installing-arch-linux-on-zfs:

The default ISO image you can download from the Arch Linux website does not support ZFS. That's why we have to create our own using the archiso tool. We will install the tool from the Arch Linux repositories.

1. Install docker image on x86 machine. Insert a host directory name to share the .iso with

    ```bash
    docker run --rm --privileged -it -v "$(pwd)/out":/root/iso/out -v  archlinux
    ```

2. Install the archiso tool, create a work directory and copy the `releng` profile to it.

    ```bash
    pacman -Sy archiso
    mkdir ~/iso
    cp -r /usr/share/archiso/configs/releng/* ~/iso
    ```

3. Add the ArchZFS repository to the Pacman configuration for our build and tell archiso to install the ZFS DKMS module and ZFS utils to our resulting ISO.

    ```bash
    echo -e '
    [archzfs]
    Server = https://archzfs.com/$repo/$arch
    SigLevel = Optional TrustAll' >> ~/iso/pacman.conf

    echo -e '
    linux-headers
    archzfs-dkms
    zfs-utils' >> ~/iso/packages.x86_64
    ```

4. Next, build the ISO. This can take some time. (FIRST TRY TOOK FROM 7:10-)

    ```bash
    mkarchiso -vo ~/iso/out ~/iso
    ```

5. Next is to write the ISO to USB, to do that first exit docker and check what the device is named.

    ```bash
    diskutil list
    ```

6. Unmount the USB disk

    ```bash
    diskutil unmountDisk /dev/disk3
    ```

7. Write the ISO to the USB.

    ```bash
    sudo dd bs=4m if=out/archlinux-2023.02.22-x86_64.iso of=/dev/disk3
    ```

8. Eject the USB when done (make sure the drive is formatted as FAT32)

    ```bash
    diskutil eject /dev/disk3
    ```

9. Partition

    ```parted
    parted -a opt /dev/nvme0n1
    print # Display current partition table
    mklabel gpt # Create new partition table, will destroy data!
    mkpart primary 5MB% 512MB # Boot/EFI
    mkpart primary 512MB 100% # remaining space
    set 1 boot on # Boot flag
    set 1 esp on # EFI flag
    quit
    ```

10. Let's create a new ZFS pool named "zroot". The options are a solid default for a pool for day to day desktop use. We'll build our final root filesystem in /mnt, so /mnt in the installer OS will be / in the installed system. Make sure to match the capitalization of the o/O letters!

    ``` bash
    zpool create \
    -o ashift=12 \
    -O acltype=posixacl -O canmount=off \
    -O dnodesize=auto -O normalization=formD \
    -O atime=off -O xattr=sa -O mountpoint=none \
    -R /mnt zroot /dev/sdb
    ```

11. You need at least one dataset for your root filesystem /. I'm creating an additional dataset for my /home directory, so I can take snapshots of my base system and it separately. You can create additional datasets if you want.


    ``` bash
    # Root dataset
    zfs create -o canmount=noauto -o mountpoint=/ zroot/rootfs

    # Set the root dataset as bootfs
    zpool set bootfs=zroot/rootfs zroot

    # Additional datasets…
    zfs create zroot/rootfs/home
    ```

12. Next, mount the root dataset. This will also mount any child datasets you created.

    ```bash
    zfs mount zroot/rootfs
    ```

13. Finally, we need to tell ZFS to create a zpool.cache file. This file contains information about out ZFS pool and can be loaded at boot time instead of re-importing the pool each time. Because /etc/zfs is part of the installer, we have to copy the cache file to our soon-to-be /etc/zfs in /mnt.

    ```bash
    mkdir -p  /mnt/etc/zfs
    zpool set cachefile=/etc/zfs/zpool.cache zroot
    cp /etc/zfs/zpool.cache /mnt/etc/zfs/zpool.cache
    ```

14. Setting up the boot partition. Format the boot partition with FAT32 and mount it into /mnt/boot.

    ```bash
    mkfs.vfat /dev/sda
    mkdir /mnt/boot
    mount /dev/nvme0n1p1 /mnt/boot
    ```

15. Generate filesystem table. Now that all filesystems for the new Arch installation are mounted in /mnt, we can generate the final fstab using the genfstab tool.

    ```bash
    genfstab -U -p /mnt >> /mnt/etc/fstab
    ```

16. System setup. Install base packages. Use pacstrap to install the base packages and a desktop environment. You can pick another desktop environment or shell and leave out SSH or ZSH if you want to.

    ```bash
    pacstrap /mnt base base-devel linux linux-headers linux-firmware grub efibootmgr \
    nano zsh gdm gnome openssh
    ```

17. Setup ZFS After chrooting into your new system, the first thing to do is to add the ArchZFS repository and install the ZFS DKMS module. After that, you can install additional packages you want in your new system.

    ```bash
    arch-chroot /mnt

    echo -e '
    [archzfs]
    Server = https://archzfs.com/$repo/x86_64' >> /etc/pacman.conf

    # ArchZFS GPG keys (see https://wiki.archlinux.org/index.php/Unofficial_user_repositories#archzfs)
    pacman-key -r DDF7DB817396A49B2A2723F7403BD972F75D9D76
    pacman-key --lsign-key DDF7DB817396A49B2A2723F7403BD972F75D9D76

    pacman -Sy zfs-dkms

    # Optional packages
    pacman -S nvidia-dkms intel-ucode
    ```

18. Boot setup. (Edit /etc/mkinitcpio.conf, find the line that defines build hooks (HOOKS=(...)) and add the ZFS hooks like this):

    ```bash
    HOOKS=(base udev autodetect modconf block keyboard keymap zfs filesystems)
    ```

19. After that, generate the image with

    ```bash
    mkinitcpio -p linux
    ```

20. Finally, let is set up GRUB. First, make sure to create the /boot/grub directory. Edit the file /etc/default/grub and add zfs=zroot/rootfs to the kernel parameters at GRUB_CMDLINE_LINUX_DEFAULT. Generate the GRUB configuration files and install GRUB with the according tools:

    ```bash
    mkdir /boot/grub
    nano /etc/default/grub # GRUB_CMDLINE_LINUX_DEFAULT="zfs=zroot/rootfs"
    grub-mkconfig -o /boot/grub/grub.cfg
    grub-install --target=x86_64-efi --efi-directory=/boot
    ```

21. Finalizing the installation. We need to enable some services for systemd to be able to handle and mount our ZFS datasets. Remember to enable your display manager service, too (gdm in my case).

    ```bash
    systemctl enable zfs.target zfs-import-cache \
    zfs-mount zfs-import.target gdm
    ```

22. The following are some common tasks like setting the correct timezone, locale, hostname, creating a new user and enable the use of sudo.

    ```bash
    # Set time and timezone
    ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime # Change according to location…
    hwclock --systohc # Sync with HW clock

    # Configure locales
    echo -e '
    de_DE.UTF-8 UTF-8
    en_US.UTF-8 UTF-8' >> /etc/locale.gen
    echo 'KEYMAP=de-latin1' > /etc/vconsole.conf
    echo 'LANG=de_DE.UTF-8' > /etc/locale.conf
    locale-gen

    # Set hostname
    echo myhostname > /etc/hostname
    echo -e '127.0.0.1 localhost\n::1 localhost\n127.0.1.1 myhostname' >> /etc/hosts

    groupadd sudo
    useradd -m -G sudo <username>
    EDITOR=nano visudo # uncomment sudo group
    passwd <username>
    ```

23. You're done! Exit the chroot with exit, unmount all filesystems, export the ZFS pool and reboot into your new OS!

    ```bash
    exit
    # Back in the installer shell…
    umount -R /mnt
    zfs umount -a
    zpool export -a
    ```