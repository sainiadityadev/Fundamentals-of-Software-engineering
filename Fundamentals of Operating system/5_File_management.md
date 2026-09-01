# File system

A file system is an abstraction layer that sits on top of raw block storage (SSDs or HDDs) and presents that storage to users and applications as files and directories.

Internally, the file system translates all file operations into reads and writes against an array of fixed-size logical blocks exposed by the storage device.

Databases and storage engines sometimes skip the file system entirely and work directly on raw block storage. This avoids the file system overhead.

Popular file systems are FAT32, ext4(linux), NTFS(windows), APFS(apple).

## Page Cache

The page cache is an in-kernel memory cache maintained by the OS. Every file system block that is read from or written to disk is cached in memory as a virtual memory page.<br>
The page cache is global across all processes. If two applications read the same file, both benefit from the same cached blocks.

Application writes to a block, OS writes to page cache(not directly to disk). OS buffers the write intentionally.<br>
The page cache entry marked as dirty. After some time, OS decides to flush dirty pages to disk.

## fsync()

Because the OS buffers writes asynchronously, applications that require durability (databases, write-ahead logs) cannot rely on `write()` alone. `fsync(fd)` tells the OS:<br>
Flush all dirty pages for this file descriptor to disk.<br>
fsync is expensive. It blocks until all writes are physically committed.
