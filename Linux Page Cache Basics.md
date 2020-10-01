# Linux页缓存基础

在linux环境，**Page Cache** 可以加速对非易失性存储中文件的访问，在首次对硬盘读写的时候，同时会将数据存储在内存中的未使用区域，用做cache。当数据再次被被读取时，可以从内存中的这个cache快速地读取到。本文将介绍有关Page Cache的相关背景信息。

## Page Cache or Buffer Cache

“Buffer Cache” 通常是指页面缓存。低于2.2版的Linux内核同时具有Page Cache和Buffer Cache。从2.4内核开始，这两个缓存合并，即 Page Cache。

## Functional Approach

### Memory Usage

在linux环境， free -m 的  Cached 列，显示当前用于页面缓存的内存大小，-m 指定单位为MB

```shell
pang@DESKTOP-RUB64NC:~$ free -m
              total        used        free      shared  buff/cache   available
Mem:          12176        9162        2790          17         223        2883
Swap:         36864         726       36137
```

### Writing

当数据写入时，会首先写入 Page Cache, 并作为脏页进行管理。Dirty 表示数据存储在Page Cache, 需要优先写入到存储设备中。脏页中的内容会被周期性地传输( 系统调用 sync 或者 fsync 可以主动触发传输 )到底层存储设备。系统。底层存储可以是一个RAID控制器或者硬盘存储器（The system may, in this last instance, be a RAID controller or the hard disk directly.  ）。  

下面的例子中，创建一个10M的文件，会首先写入到 Page Cache中。文件创建之后，脏页数会增大，直到数据被写入到底层存储中，在这里，我们通过手动执行`sync`命令来将数据写入底层存储。

注：这里可能要找比较旧的系统，自动刷新的频率比较慢... 不然可能一瞬间就刷到硬盘了.

```
wfischer@pc:~$ dd if=/dev/zero of=testfile.txt bs=1M count=10
10+0 records in
10+0 records out
10485760 bytes (10 MB) copied, 0,0121043 s, 866 MB/s
wfischer@pc:~$ cat /proc/meminfo | grep Dirty
Dirty:             10260 kB
wfischer@pc:~$ sync
wfischer@pc:~$ cat /proc/meminfo | grep Dirty
Dirty:                 0 kB
```

#### pdflush  2.6.31 Kernel 以上版本

pdflus 线程 确保 脏页会被定期写入到底层存储设备中

#### 单设备回写  2.6.32 Kernel 以上版本

由于 pdflush 存在很多性能问题，Jens Axboe 为 linux kernel 2.6.32  开发了一个新的，更高效的回写机制

### Reading

文件块不仅仅在写数据的时候会加载到Page Cache, 读的时候也会。举个例子，当你先后两次读取读取一个100MB的文件时，第二次会比第一次快得多，因为相应的文件块可以直接从内存中的PageCache获取，不需要再次从硬盘中读取。下面的例子展示了当一个200M的视频文件播放以后，Page Cache 变大的情况。

```shell
user@adminpc:~$ free -m
             total       used       free     shared    buffers     cached
Mem:          3884       1812       2071          0         60       1328
-/+ buffers/cache:        424       3459
Swap:         1956          0       1956
user@adminpc:~$ vlc video.avi
[...]
user@adminpc:~$ free -m
             total       used       free     shared    buffers     cached
Mem:          3884       2056       1827          0         60       1566
-/+ buffers/cache:        429       3454
Swap:         1956          0       1956
user@adminpc:~$
```

如果 Linux 需要的内存空间超过了当前可用的空间，当前不在使用中的Page Cache区域将会被自动删除(当然有很多缓存淘汰算法，比如LRU, LFU 等等)。

### Page Cache 的优化

在Page Cache中自动存储文件块有很多好处。而对于一些数据，比如日志文件或者Mysql dump文件，它们在写入之后基本就不再需要了，这些文件块经常会不必要地使用到Page Cache。另外一些更需要Page Cache的文件块，可能会因为更新的日志文件和Mysql dump，而被过早地从Page Cache中被删除。

使用gzip压缩定期执行logrotate可以对这种情况有所帮助。当一个500MB的日志文件被压缩成 10MB时，原始日志文件的缓存空间会失效，Page Cache中又会多出 490MB的可用空间，这样就减少了由于日志文件不断增长而导致有用的文件块从页面缓存中移出的危险。

因此，如果某些应用程序正常不需要将文件存储在缓存中，这是合理的，也是被允许的。