# Linux Inodes — Reference Notes

---

## What an Inode Is

An inode (index node) is a data structure that stores all the metadata about a file **except its name and actual content**. Every file and directory on a Linux filesystem has exactly one inode.

## What an Inode Stores

| Attribute | Example |
|---|---|
| File type | regular file, directory, symlink, socket |
| Permissions | `rwxr-xr-x` |
| Owner (UID) and group (GID) | `wasadmin:wasadmin` |
| File size | in bytes |
| Timestamps | atime (access), mtime (modify), ctime (metadata change) |
| Link count | how many filenames point to this inode |
| Pointers to data blocks | where the actual file content physically lives on disk |

**What it does NOT store: the filename.** The filename lives in the *directory entry*, which is just a mapping of `name → inode number`. The inode itself has no idea what it's called.

## Visualizing It

```
Directory entry (in /opt/IBM/WebSphere/logs/):
  "SystemOut.log"  →  inode 884521

Inode 884521 contains:
  - permissions: rw-r--r--
  - owner: wasadmin
  - size: 45MB
  - timestamps
  - pointers to actual data blocks on disk
```

## Check a File's Inode Number

```bash
ls -i SystemOut.log
```
```
884521 SystemOut.log
```

Or with more detail:

```bash
stat SystemOut.log
```
```
  File: SystemOut.log
  Size: 47185920        Blocks: 92160     IO Block: 4096   regular file
Device: 803h/2051d      Inode: 884521      Links: 1
Access: (0644/-rw-r--r--)  Uid: (1001/wasadmin)   Gid: (1001/wasadmin)
```

---

## Why This Matters Operationally

### 1. "Disk full" but `df` shows space available — you've run out of inodes, not blocks

Every filesystem has a **fixed number of inodes** set at creation time — even if you have terabytes of free space, if you've created millions of tiny files (common with runaway log rotation or session files), you can exhaust inodes and get "No space left on device" with plenty of disk space showing free.

```bash
# Check inode usage vs block usage
df -i          # inode usage
df -h          # block/space usage
```

If `df -i` shows `100% IUse` but `df -h` shows plenty of free space — that's your answer. Common culprit on WAS servers: excessive session persistence files, orphaned temp files, or a logging misconfiguration creating thousands of small files.

### 2. Deleting a file doesn't free space if a process still has it open

```bash
rm SystemOut.log
```

This removes the *directory entry* (the name → inode mapping), but the inode and its data blocks **aren't actually freed until the link count reaches zero AND no process has the file open**. If WAS still has `SystemOut.log` open for writing when you `rm` it, disk space isn't reclaimed — this is the classic "I deleted the huge log file but `df -h` still shows the disk full" scenario.

**Fix without restarting the JVM:**

```bash
# Find the process still holding the deleted file open
lsof +L1 | grep deleted

# Truncate it in place instead of deleting
> /opt/IBM/WebSphere/logs/SystemOut.log
# or
: > SystemOut.log
```

### 3. Hard Links vs Symbolic Links — Tied Directly to Inodes

```bash
# Hard link — points to the SAME inode
ln original.txt hardlink.txt
ls -i original.txt hardlink.txt
# Both show the SAME inode number

# Symbolic link — its own inode, pointing to a PATH (not the same inode)
ln -s original.txt symlink.txt
ls -i original.txt symlink.txt
# Different inode numbers
```

A hard link count is exactly the "Links" field seen in `stat` output above — it's why deleting the *original* file doesn't destroy the data if a hard link still references the same inode (the data only vanishes when the link count hits zero).

### 4. Link Count Also Matters for Directories

```bash
ls -id /opt/IBM/WebSphere/AppServer
```

A directory's link count is always at least 2 (itself, plus `.` inside it), plus one more for every subdirectory it contains (`..` entries).

---

## Quick Reference Table

| Command | Purpose |
|---|---|
| `ls -i` | Show inode number for a file |
| `stat file` | Show full inode metadata |
| `df -i` | Show inode usage per filesystem |
| `find / -inum <number>` | Find a file by its inode number |
| `lsof +L1` | Show open files with a link count of 0 (deleted but still held open) |