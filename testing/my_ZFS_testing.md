# My ZFS Testing

## ZFS unit testing

`ztest(1)` documentation [here](https://manpages.debian.org/testing/zfs-test/ztest.1.en.html). `ztest` was written by the ZFS Developers as a ZFS unit test. The tool was developed in tandem with the ZFS functionality and was executed nightly as one of the many regression test against the daily build. As features were added to ZFS, unit tests were also added to `ztest`. In addition, a separate test development team wrote and executed more functional and stress tests.

To run the unit tests (by default it will run for 10 minutes).

```bash
sudo ztest -f / -VVV
```

## System Prerequisites

  1) Three scratch disks
  
        a) Specify the disks you wish to use in the $DISKS variable, as a space delimited list like this: `DISKS='vdb vdc vdd'`.  By default the zfs-tests.sh script will construct three loopback devices to be used for testing: `DISKS='loop0 loop1 loop2'`.

  2) A non-root user with a full set of basic privileges and the ability to sudo(8) to root without a password to run the test.
  3) Specify any pools you wish to preserve as a space delimited list in the $KEEP variable. All pools detected at the start of testing are added automatically.
  4) The ZFS Test Suite will add users and groups to test machine to verify functionality. Therefore it is strongly advised that a dedicated test machine, which can be a VM, be used for testing.
  
     a) On FreeBSD, `mountd(8)` must use `/etc/zfs/exports` as one of its export files – by default this can be done by setting `zfs_enable=yes` in `/etc/rc.conf`.

## Building ZFS Testing Suite

The ZFS Test Suite runs under the test-runner framework.  This framework
is built along side the standard ZFS utilities and is included as part of
zfs-test package.  The zfs-test package can be built from source as follows:

```bash
./configure
make pkg-utils
```

## Running the ZFS Test Suite
Simply run the zfs-tests.sh script:

```bash
/usr/share/zfs/zfs-tests.sh
```