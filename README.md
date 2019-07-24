# zfsrep - Replicate a ZFS snapshot to a destination zpool

This script takes ZFS file system snapshots created on FreeBSD and replicates
them to a destination ZFS pool via use of `zfs send` and `zfs recv`. It
was written as a companion script to `zfssnap` and supersedes the
`clonesnap.sh` script that I was using, and was created so that I could also
replicate snapshots from the source pool to the destination, something not
possible when using `rsync`.  It was modified to support `zfs-auto-snapshot`
once I retired `zfssnap` in favour of `sysutils/zfstools`.


## Usage

```
zfsrep [-v] [-m] [-S] -s snapname srcpool destpool
```
* `-v`: Be verbose
* `-m`: Import the destination pool if it hasn't been already
* `-S`: Initial `zpool scrub` at end of replication on destination pool
* `-s snapname`: The snapshot type to replicate (eg "`monthly`")
* `srcpool`: The source pool
* `destpool`: The destination pool

If the `destpool` is imported via the `-m` argument it will be exported at the
end of the replication, unless `-S` is specified, in which case the scrub will
start and a human will need to keep an eye on the result and manually export
the destination pool when complete.

A full replication is performed if the destination pool is empty, otherwise an
incremental replication based on the last received snapshot is done.  This may
cause issues if you have waited too long to run this script, eg five weeks
after the last weekly which is kept on a four-week rotation, as the script
won't be able to find the last snapshot on the source pool.  The script will
not recover from this situation; it will exit and you will probably need to
destroy the destination pool and start from scratch.


## Requirements

* FreeBSD 9 or later
* A patched version of `sysutils/zfstools`

The patch to `sysutils/zfstools` can be found in the `files` directory. Copy it
into `/usr/ports/sysutils/zfstools/files` and build the port using whatever
method you prefer.


### Why is a patch for sysutils/zfstools required?

When `systutils/zfstools` version 0.3.2 was released (or it may have been 0.3.1
and I failed to notice; I do recall being surprised when I finally did),
a change was made where unmounted datasets were excluded from the list of
datasets to snapshot.  This turned out to be a problem for ZFS Boot
Environments that had been set up per the standard when using `bsdinstall` in
FreeBSD 10, which I had replicated for my FreeBSD 9 installation to ease future
upgrades.  Consider the following file system layout of one of my systems:

```
NAME                         MOUNTPOINT          CANMOUNT
zroot                        none                      on
zroot/ROOT                   none                      on
zroot/ROOT/11.2-RELEASE-p11  /                     noauto
zroot/tmp                    /tmp                      on
zroot/usr                    /usr                     off
zroot/usr/home               /usr/home                 on
zroot/usr/home/awaters       /usr/home/awaters         on
zroot/usr/ports              /usr/ports                on
zroot/var                    /var                     off
zroot/var/crash              /var/crash                on
zroot/var/log                /var/log                  on
zroot/var/mail               /var/mail                 on
zroot/var/tmp                /var/tmp                  on
```

`zroot/usr` and `zroot/var` have the mount points that we would expect, but
they aren't actually mounted.  Their content lives in the root file system of
the boot environment, `zroot/ROOT/11.2-RELEASE-p11` in this case.  However the
mount points for these file systems (and any other ZFS properties that have been
set locally) are inherited by the child file systems, so the child file systems
will be mounted in the correct place under the boot environment's root.

What happens though if you use `zfsrep` to replicate a ZFS root zpool via
a snapshot that does not include `zroot/usr` and `zroot/var` (or whatever you
named your pool) because they were excluded from the snapshot due to being
unmounted?  Well, `zfsrep` mounts the destination pool using an alternative
root directory ("altroot") in `/tmp` of the format `/tmp/zfsrep-XXXXXX`.  As
a result, with no mount point option to inherit from `zroot/usr` or `zroot/var`
their child file systems will inherit their mount point from `zroot`, and you
will see things like `zroot/var/log` having a mount point of
`/tmp/zfsrep-XXXXXX/log`, which is probably not what you were expecting.  Or it
might have been something like `/log` due to the altroot being temporary;
I can't remember and I don't have a desire at the moment to start breaking
things in order to confirm it.

So my at-the-time solution was to patch the functionality to take snapshots of
unmounted datasets back into `zfs-auto-snapshot`, specifically:

* `-a --all-datasets`: Snapshot all datasets, including unmounted ones.

With that in place, the entries in `/etc/crontab` are as below, noting that the
"keep zero sized snapshots" option (`-k`) is also used:

```
# ZFS pool snapshots
*/15    *   *   *   *   root    /usr/local/sbin/zfs-auto-snapshot -ak frequent  4
@hourly                 root    /usr/local/sbin/zfs-auto-snapshot -ak hourly   24
@daily                  root    /usr/local/sbin/zfs-auto-snapshot -ak daily     7
@weekly                 root    /usr/local/sbin/zfs-auto-snapshot -ak weekly    4
@monthly                root    /usr/local/sbin/zfs-auto-snapshot -ak monthly  12
```

Is all of this still necessary?  Should I just import the pool with the `-N`
option? I don't know.  I seem to recall not mounting with an altroot caused
issues with ZFS dataset property inheritance, but I'm fuzzy on the details.
I might be motivated to test it out one day, except for the bugs I've
encountered with the process mentioned below.


## Apparent bug in "zfs recv"

Around FreeBSD 10.3 I started encountering a bug when using `zfsrep`, usually
with monthly snap shots.  `zfs recv` is invoked with the `-F` argument to force
a roll-back of the destination pool so that any snapshots and file systems that
do not exist on the source pool are removed from the destination pool.  This
mostly worked fine, but would often in the case of the monthly snapshot produce
one of two outcomes:

1. Kernel panic, stack trace, halted system
2. Spontaneous system reboot

Neither of these were particularly pleasing to see.

This behaviour does not occur when performing the clean up initiated via the
`-F` option to `zfs recv` manually via the use of `zfs destroy` prior to
running `zfsrep`.  This leads me to believe the bug is in the `zfs recv` code.

Further odd behaviour has been observed since FreeBSD 11.1, where the
destination pool will sometimes not export successfully because it is "busy",
despite all the file systems in that pool being successfully unmounted.
A reboot of the entire system is required to fix this.

Both of these apparent bugs in FreeBSD's ZFS code are not terribly welcome, and
have lead me to question whether or not ZFS really is viable on FreeBSD for the
moment.  Of course, it could be the way I'm doing it, but the fact that even
the FreeBSD Project itself is [rebasing the Illumos-based ZFS code to zfsonlinux
(ZoL)](https://lists.freebsd.org/pipermail/freebsd-fs/2018-December/027085.html)
due to issues like lack of upstream development on the Illumos-based code and
several fixed bugs and race conditions in the ZoL code suggests I may have
encountered some of those unfixed bugs.  Time will tell, I guess.

