---
layout: posts
title: The overlay filesystem
---
# The overlay filesystem

Docker makes heavy use of union filesystems in order to layer images. There are several union filesystem implementations. When installed on Ubuntu, Docker uses its overlay2 driver based on overlayfs in the Linux kernel.

Rather than filesystems, overlayfs mounts  directories that reside on filesystems that are already mounted.

## References

Kernel doc: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/filesystems/overlayfs.txt
How to: https://askubuntu.com/questions/109413/how-do-i-use-overlayfs and http://jasonwryan.com/blog/2015/01/19/overlayfs

## Quick and dirty howto

<pre><code>
$ mkdir up
$ mkdir down
$ mkdir combined
$ mkdir /tmp/work
</code></pre>
Populate <code>up</code> with files.
Populate <code>down</code> with files.
<pre><code>
# mount -t overlay -o lowerdir=down,upperdir=up,workdir=/tmp/work overlay combined
</code></pre>
Result: files in <code>up</code> and <code>down</code> appear together in <code>combined</code>.
Or: Instead of a third <code>combined</code> directory, we simply merge all <code>down</code> files into
<code>up</code>:
<pre><code>
# mount -t overlay -o lowerdir=down,upperdir=up,workdir=/tmp/work overlay up
</code></pre>

**What happens with files that have the same name in <code>up</code> and <code>down</code>?**
The files in down will be hidden. Only the files in up will be visible. After
unmounting, all files in down will be visible again.

**What happens when anything is changed in the <code>combined</code> directory?**
Changes to the <code>combined</code> directory are done in <code>up</code>. <code>down</code> will be unchanged.

When a file in the combined directory is modified that actually resides in
<code>down</code>, it will first be copied to <code>up</code> before modification.
After unmount, a file with the same name exists in both <code>up</code> and <code>down</code>, but only the version
in <code>up</code> is modified.

**What happens when a new object is created in the combined directory?**
After unmount, it will be in <code>up</code>.

Since lower directories remain unchanged, they can be shared by several
overlay mounts.

Let's play with an overlay filesystem. This is on Ubuntu 16.04. We start by creating the directories, adding some files and merging them.

<pre><code>
$ mkdir up down combined
$ cp /etc/passwd up
$ cp /etc/group down
$ mkdir wdir
$ sudo mount -t overlay -o lowerdir=down,upperdir=up,workdir=wdir overlay combined
$ ll -i up down combined
combined:
total 8
67157324 -rw-r--r--. 1 bbausch bbausch  704 Feb 20 01:18 group
 2411565 -rw-r--r--. 1 bbausch bbausch 1490 Feb 20 01:18 passwd

down:
total 4
67157324 -rw-r--r--. 1 bbausch bbausch 704 Feb 20 01:18 group

up:
total 4
2411565 -rw-r--r--. 1 bbausch bbausch 1490 Feb 20 01:18 passwd
</code></pre>
In the code above, note that the inodes in the <code>combined</code> directory are identical to those in <code>up</code> and <code>down</code>, depending on the actual location of a file.

<pre><code>
$ echo anotherfile > down/anotherfile
$ ll -i up down combined
combined:
total 12
67157325 -rw-rw-r--. 1 bbausch bbausch   12 Feb 20 01:20 anotherfile
67157324 -rw-r--r--. 1 bbausch bbausch  704 Feb 20 01:18 group
 2411565 -rw-r--r--. 1 bbausch bbausch 1490 Feb 20 01:18 passwd

down:
total 8
67157325 -rw-rw-r--. 1 bbausch bbausch  12 Feb 20 01:20 anotherfile
67157324 -rw-r--r--. 1 bbausch bbausch 704 Feb 20 01:18 group

up:
total 4
2411565 -rw-r--r--. 1 bbausch bbausch 1490 Feb 20 01:18 passwd
</code></pre>
A file added to <code>down</code> is also visible in <code>combined</code>.

<pre><code>
$ echo modified >> combined/anotherfile
$ ll -i up down combined
combined:
total 12
 2411575 -rw-rw-r--. 1 bbausch bbausch   21 Feb 20 01:20 anotherfile
67157324 -rw-r--r--. 1 bbausch bbausch  704 Feb 20 01:18 group
 2411565 -rw-r--r--. 1 bbausch bbausch 1490 Feb 20 01:18 passwd

down:
total 8
67157325 -rw-rw-r--. 1 bbausch bbausch  12 Feb 20 01:20 anotherfile
67157324 -rw-r--r--. 1 bbausch bbausch 704 Feb 20 01:18 group

up:
total 8
2411575 -rw-rw-r--. 1 bbausch bbausch   21 Feb 20 01:20 anotherfile
2411565 -rw-r--r--. 1 bbausch bbausch 1490 Feb 20 01:18 passwd
</code></pre>
When the new file is modified, a copy is made to <code>up</code>. From the inode number, it's obvious that the file visible in <code>combined</code> is identical to the one in <code>up</code>.

<pre><code>
$ echo newfile > combined/newfile
$ ll -i up down combined
combined:
total 16
 2411575 -rw-rw-r--. 1 bbausch bbausch   21 Feb 20 01:20 anotherfile
67157324 -rw-r--r--. 1 bbausch bbausch  704 Feb 20 01:18 group
 2411576 -rw-rw-r--. 1 bbausch bbausch    8 Feb 20 01:21 newfile
 2411565 -rw-r--r--. 1 bbausch bbausch 1490 Feb 20 01:18 passwd

down:
total 8
67157325 -rw-rw-r--. 1 bbausch bbausch  12 Feb 20 01:20 anotherfile
67157324 -rw-r--r--. 1 bbausch bbausch 704 Feb 20 01:18 group

up:
total 12
2411575 -rw-rw-r--. 1 bbausch bbausch   21 Feb 20 01:20 anotherfile
2411576 -rw-rw-r--. 1 bbausch bbausch    8 Feb 20 01:21 newfile
2411565 -rw-r--r--. 1 bbausch bbausch 1490 Feb 20 01:18 passwd
</code></pre>
A new file created in <code>combined</code> goes to <code>up</code>. Generally, <code>down</code> remains unchanged.
<pre><code>
$ rm combined/anotherfile
$ ll -i up down combined
combined:
ls: cannot access combined/anotherfile: No such file or directory
total 12
       ? ??????????? ? ?       ?          ?            ? anotherfile
67157324 -rw-r--r--. 1 bbausch bbausch  704 Feb 20 01:18 group
 2411576 -rw-rw-r--. 1 bbausch bbausch    8 Feb 20 01:21 newfile
 2411565 -rw-r--r--. 1 bbausch bbausch 1490 Feb 20 01:18 passwd

down:
total 8
67157325 -rw-rw-r--. 1 bbausch bbausch  12 Feb 20 01:20 anotherfile
67157324 -rw-r--r--. 1 bbausch bbausch 704 Feb 20 01:18 group

up:
total 8
2411577 c---------. 1 root    root    0, 0 Feb 20 01:21 anotherfile
2411576 -rw-rw-r--. 1 bbausch bbausch    8 Feb 20 01:21 newfile
2411565 -rw-r--r--. 1 bbausch bbausch 1490 Feb 20 01:18 passwd
$
</code></pre>
This error is a bit unexpected. Is this intentional?

The mount command doesn't provide much information about overlay mounts, but
/proc/mounts includes all mount options. In the case of Docker, the filesystem
of a running container looks like this:

<code>
$ grep overlay /proc/mounts
overlay /var/lib/docker/overlay2/05b043f6b90428fea70be4e8ec418e8680293290facd93360e558a06cfbad3f2/merged overlay rw,relatime,lowerdir=/var/lib/docker/overlay2/l/HWNEZLITLB2BDFHOAP7SPS2FXR:/var/lib/docker/overlay2/l/E5MMCSFLS4HNCNYHH27ELJBDKG:/var/lib/docker/overlay2/l/QYSC4CL5BBU5XK5UIT7UDWLW4F:/var/lib/docker/overlay2/l/C3TZTATSWBGG3LERPZRI2MCM7A:/var/lib/docker/overlay2/l/QE4LY3B5JW6RWFTEMUFKPU7IV2:/var/lib/docker/overlay2/l/G2XL5HZSBXUORXHDC3FBOBG7YN,upperdir=/var/lib/docker/overlay2/05b043f6b90428fea70be4e8ec418e8680293290facd93360e558a06cfbad3f2/diff,workdir=/var/lib/docker/overlay2/05b043f6b90428fea70be4e8ec418e8680293290facd93360e558a06cfbad3f2/work 0 0
...
</code>


Concepts of "whiteout" and "opaque" objects not clear.
