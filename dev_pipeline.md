# Dev Pipeline Walkthrough

## Building Module

```bash
# as user
gh repo clone kevin-ewing/zfs-internal-reference-count
cd zfs-internal-reference-count
git fetch
git branch -v -a
git checkout remotes/origin/zfs-2.1-release
git clean -f

# Ready to do stuff
./autogen.sh
env MAKE=gmake ./configure
gmake -j`sysctl -n hw.ncpu`
sudo gmake install
```

## Trying to test

Here I will try to copy over some of the code that pjd has done on the freeBSD module of ZFS. Instead of his huge function to do all the block cloning, though, I will just error(69);

```bash
/usr/home/kewing/zfs-internal-reference-count/zfs-test.sh
```

## Hand Testing Module

List disk devices with:

```bash
lsblk
```

Create a pool on the first free disk device:

```bash
sudo zpool create -f test /dev/vtbd1
```

To view the newly created pool type. `df`. This output shows creating and mounting of the `test` pool, and that is now accessible as a file system. Create files for users to browse.

```bash
cd /test
ls
sudo touch testFile
ls -al
```

Test if copy works, it does, but then how do I use the reference link flag?

```bash
sudo cp testFile anotherTestFile
```

Manually using `copy_file_range(2)`.

Move back to testing pool
```bash
sudo touch test1
sudo ../home/kewing/manual-testing/my-garbel test1

```

To get rid of the pool:

```bash
zpool destroy test
```

## Testing Suite on Module

Tests require fdescfs to be mount on /dev/fd. This can be done temporarily with:

<!-- ```bash
sudo mount -t fdescfs fdescfs /dev/fd
``` -->

### Should be as follows but does not work on freeBSD

```bash
./configure
make pkg-utils
```

## Running the ZFS Test Suite

Simply run the zfs-tests.sh script:

```bash
/usr/share/zfs/zfs-tests.sh
```
