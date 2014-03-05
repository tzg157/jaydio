Java Direct I/O (Jaydio)
========================

Jaydio is a Java library for giving the programmer finer control over
file I/O, in part by bypassing the OS buffer cache. **Currently only Linux is
supported**, but Jaydio has big dreams of expanding to other platforms
and adding new functionality other than the vanilla direct I/O currently
offered. Jaydio serves as as a starting point for asynchronous / non-blocking direct I/O,
custom file page caching layers, and even advanced cache replacement policies
like [ARC](http://dbs.uni-leipzig.de/file/ARC.pdf)
or [2Q](http://www.inf.fu-berlin.de/lehre/WS06/DBS-Tech/Reader/2QBufferManagement.pdf).
Currently, it is useful for preventing the operating system from evicting
memory pages which need to stay "warm" in favor of non-critical file pages.


How it works
------------

Jaydio works by passing the `O_DIRECT` flag to the Linux `open()` system call.
Unfortunately, talking directly to disk imposes some nasty rules about how
many bytes can be written at a time, and where they can be written. In particular,
all writes must be in multiples of the file system block size, and they must
originate from chunks of memory aligned to the memory page size, and they must
also occur at file system block boundaries.
see `man 2 open` (search for `O_DIRECT`) and `man 3 posix_memalign` for
more details.

Fortunately, with Jaydio you don't have to worry about any of that. All of the
nasty alignment rule parameters are determined on-the-fly, so that native
libc file reads and writes may be used to bypass the OS cache. Jaydio uses
[JNA](https://github.com/twall/jna/) to accomplish all this parameter
querying and other native magic.

Example
-------

```java
int bufferSize = 1<<23; // Use 8 MiB buffers
byte[] buf = new byte[bufferSize];

DirectRandomAccessFile fin = 
        new DirectRandomAccessFile(new File("hello.txt"), "r", bufferSize);

DirectRandomAccessFile fout =
        new DirectRandomAccessFile(new File("world.txt"), "rw", bufferSize);

while (fin.getFilePointer() < fin.length()) {
    int remaining = (int)Math.min(bufferSize, fin.length()-fin.getFilePointer());
    fin.read(buf,0,remaining);
    fout.write(buf,0,remaining);
}

fin.close();
fout.close();
```


FAQ
===

#### Does it work?

Here is a plot of memory use on my system during a copy of a 1.4 GB file, under
different scenarios. The first scenario (red) corresponds to having plenty
of memory free (about 2.6 GB) **at the start of the copy**, so that, if need be, we could basically fit two
copies of file in memory. The second scenario (blue), my system has limited memory
available (about 900 MB) when copying begins. The third scenario (green), we use Jaydio for the file
copy (yay!), with buffers totaling about 36 MiB.
For each scenario, completion of the copy is indicated by a vertical dotted line.

![File copy comparison plot](http://smacke.net/img/jaydio-cp-plot.png)

As you can see, Jaydio successfully bypasses the OS buffer cache,
and uses no memory other than the small buffers allocated to it.
Interestingly enough, for scenario 2 (blue, RAM limited), we're only
taking marginally less time than direct I/O for the copy! This is likely
because we ran out of RAM early on, and so are effecively doing direct I/O
at the point where the blue curve flatlines, as we need to free up space
in the cache whenever we do a write.

Jaydio takes about 41 seconds to do the copy, RAM-limited `cp` takes
about 40.5 seconds, and RAM-abundant `cp` takes about 26.8 seconds
on my system, for a 1.4 GB file.

#### Why bypass file system cache? Isn't it there for a reason?

Yes, and in 99% of cases you will want to use it. As mentioned earlier,
Jaydio serves as a starting point for implementing more advanced cache
replacement policies than LRU, but there are also times when you want
to prevent the operating system from evicting certain memory pages in
favor of caching files.


#### Can't I just use something like `DONTNEED` or `NOREUSE` with `posix_fadvise` or `madvise` if I'm worried about that page cache stomping over useful memory pages?

As far as I know, `NOREUSE` is a no-op.

You could use `DONTNEED`, and you would likely see better performance (in terms of raw speed), too. Unfortunately,
this sort of strategy does not discriminate between cached pages that you want to save and those
that you want to write back to disk, if, say, the same file has been opened by multiple threads,
of differing priorities regarding memory accesses.
[Here](http://blog.mikemccandless.com/2010/06/lucene-and-fadvisemadvise.html) is one example of a
situation where this sort of strategy will not work.


#### I'm on Linux, but when I try to create a Jaydio `DirectRandomAccessFile` I get "Error opening file, got Invalid argument". What gives?

This is the very uninformative libc error that you get when something goes
wrong when opening a file. Chances are that your file system does not support
opening files with the `O_DIRECT` flag. For example, if you encrypt your home
directory using a file system like ecryptfs, you can't use direct I/O there.


Contributing
============
