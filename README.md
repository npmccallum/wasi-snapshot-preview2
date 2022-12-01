# WASI Snapshot Preview2

This document is a work in progress. It is an attempt to produce a new
snapshot of WASI using the features that we have in `.wit` today. This means
we are unable to use features like `resources`, `streams` and `futures` since
they are all not yet available. As of this writing, the only feature that is
currently required to make this file valid `.wit` is the resurrection of the
`use` syntax.

This snapshot borrows extensively from the work of others, usually adapting
only to the features of `.wit` today or ensuring that all functions take
a context argument. In some cases (such as `wasi-dns`), adapting to the
reduced feature set meant more invasive changes. But this represents an
attempt to build on the work of others. So a huge thanks to everyone who has
been working on various standards in this area.

# wasi-core

The `wasi-core` interface provides core types used by all other interfaces.

Since we lack WIT resources, we have here extended a traditional UNIX file
descriptor approach. This is not intended to be permanent, but solves this
short-term problem until we get something like the former WIT resources.

Each file descriptor type has a function `is-$type(descriptor) -> bool`. This
allows runtime detection of file descriptor types. In particular, it allows
tools like `wasi-libc` to accurately detect preopens types.

All resources must be destroyed by calling `close(descriptor)`. It is possible
to determine if a descriptor is valid by calling `is-valid(descriptor)`.

```wit
interface wasi-core {
    /// Error codes returned by functions.
    /// Not all of these error codes are returned by the functions provided by this
    /// API; some are used in higher-level library layers, and others are provided
    /// merely for alignment with POSIX.
    enum errno {
        /// Argument list too long. This is similar to `E2BIG` in POSIX.
        toobig,

        /// Permission denied.
        access,

        /// Address in use.
        addrinuse,

        /// Address not available.
        addrnotavail,

        /// Address family not supported.
        afnosupport,

        /// Resource unavailable, or operation would block.
        again,

        /// Connection already in progress.
        already,

        /// Bad message.
        badmsg,

        /// Device or resource busy.
        busy,

        /// Operation canceled.
        canceled,

        /// No child processes.
        child,

        /// Connection aborted.
        connaborted,

        /// Connection refused.
        connrefused,

        /// Connection reset.
        connreset,

        /// Resource deadlock would occur.
        deadlk,

        /// Destination address required.
        destaddrreq,

        /// Storage quota exceeded.
        dquot,

        /// File exists.
        exist,

        /// Bad address.
        fault,

        /// File too large.
        fbig,

        /// Host is unreachable.
        hostunreach,

        /// Identifier removed.
        idrm,

        /// Illegal byte sequence.
        ilseq,

        /// Operation in progress.
        inprogress,

        /// Interrupted function.
        intr,

        /// Invalid argument.
        inval,

        /// I/O error.
        io,

        /// Socket is connected.
        isconn,

        /// Is a directory.
        isdir,

        /// Too many levels of symbolic links.
        loop,

        /// File descriptor value too large.
        mfile,

        /// Too many links.
        mlink,

        /// Message too large.
        msgsize,

        /// Multihop attempted.
        multihop,

        /// Filename too long.
        nametoolong,

        /// Network is down.
        netdown,

        /// Connection aborted by network.
        netreset,

        /// Network unreachable.
        netunreach,

        /// Too many files open in system.
        nfile,

        /// No buffer space available.
        nobufs,

        /// No such device.
        nodev,

        /// No such file or directory.
        noent,

        /// Executable file format error.
        noexec,

        /// No locks available.
        nolck,

        /// Link has been severed.
        nolink,

        /// Not enough space.
        nomem,

        /// No message of the desired type.
        nomsg,

        /// Protocol not available.
        noprotoopt,

        /// No space left on device.
        nospc,

        /// Function not supported.
        nosys,

        /// Not a directory or a symbolic link to a directory.
        notdir,

        /// Directory not empty.
        notempty,

        /// State not recoverable.
        notrecoverable,

        /// Not supported, or operation not supported on socket.
        notsup,

        /// Inappropriate I/O control operation.
        notty,

        /// No such device or address.
        nxio,

        /// Value too large to be stored in data type.
        overflow,

        /// Previous owner died.
        ownerdead,

        /// Operation not permitted.
        perm,

        /// Broken pipe.
        pipe,

        /// Result too large.
        range,

        /// Read-only file system.
        rofs,

        /// Invalid seek.
        spipe,

        /// No such process.
        srch,

        /// Stale file handle.
        stale,

        /// Connection timed out.
        timedout,

        /// Text file busy.
        txtbsy,

        /// Cross-device link.
        xdev,
    }

    /// A handle to a resource whose lifetime needs to be manually managed.
    type descriptor = u32

    /// Some number of bytes of memory.
    ///
    /// This type should be 32 bits on wasm32 and 64 bits on wasm64.
    type usize = u32

    /// Returns whether or not the provided descriptor is valid.
    is-open: func(desc: descriptor) -> bool

    /// Closes a descriptor.
    close: func(desc: descriptor) -> result<_, errno>
}
```

# wasi-log

The `wasi-log` interface provides facilities for logging. It is mostly
unmodified from other proposals except with the addition of the `sink`
descriptor so that logs always have a context.

```wit
interface wasi-log {
    use { descriptor, errno } from wasi-core

    /// A log level, describing a kind of message.
    enum level {
        /// Describes messages about the values of variables and the flow of control
        /// within a program.
        trace,

        /// Describes messages likely to be of interest to someone debugging a program.
        debug,

        /// Describes messages likely to be of interest to someone monitoring a program.
        info,

        /// Describes messages indicating hazardous situations.
        warn,

        /// Describes messages indicating serious errors.
        error,
    }

    /// Returns whether or not the descriptor is a sink.
    is-sink: func(sink: descriptor) -> bool

    /// Emit a log message.
    ///
    /// A log message has a `level` describing what kind of message is being sent,
    /// a context, which is an uninterpreted string meant to help consumers group
    /// similar messages, and a string containing the message text.
    log: func(sink: descriptor, level: level, context: string, message: string) -> result<_, errno>
}
```

# wasi-clock

Clocks are defined as descriptors. They alwasy return a `duration` type. The
`duration` type represents the time that has elapsed since some previous time.

  * In the case of a monotonic clock, that time is undefined but is likely to
    be something like "boot time."

  * In the case of a realtime clock, that time is defined as the Unix Epoch.

```wit
interface wasi-clock {
    use { descriptor, errno } from wasi-core

    /// A duration since some time in the past.
    record duration {
        seconds: u64,
        nanoseconds: u32,
    }

    /// Returns whether or not the descriptor is a monotonic clock.
    is-clock-monotonic: func(clock: descriptor) -> bool

    /// Returns whether or not the descriptor is a realtime clock.
    is-clock-realtime: func(clock: descriptor) -> bool

    /// Query the resolution (precision) of the clock.
    ///
    /// The nanoseconds field of the output is always less than 1000000000.
    get-resolution: func(clock: descriptor) -> result<duration, errno>

    /// Read the current value of the clock.
    ///
    /// If this clock is monotonic, calling this function repeatedly will produce
    /// a sequence of non-decreasing durations from some undefined time in the past.
    ///
    /// If this clock is not monotonic, calling this function repeatedly will
    /// produce a sequence of possibly-decreasing durations from
    /// 1970-01-01T00:00:00Z. This is known as [POSIX's Seconds Since the Epoch] or
    /// [Unix Time].
    ///
    /// The nanoseconds field of the output is always less than 1000000000.
    ///
    /// [POSIX's Seconds Since the Epoch]: https://pubs.opengroup.org/onlinepubs/9699919799/xrat/V4_xbd_chap04.html#tag_21_04_16
    /// [Unix Time]: https://en.wikipedia.org/wiki/Unix_time
    get-duration: func(clock: descriptor) -> result<duration, errno>
}
```

# wasi-random

All functions take a `random` descriptor which represents a source of entropy.
Callers can request a random `u64` or `list<u8>`. The latter allows for large
reads but with the cost of allocation. The former allows for fast random values
without allocation.

```wit
interface wasi-random {
    use { descriptor, errno, usize } from wasi-core

    /// A value containing 128 random bits.
    ///
    /// This is a value import, which means it only provides one value, rather
    /// than being a function that could be called multiple times. This is intented
    /// to be used by source languages to initialize hash-maps without needing the
    /// full `getrandom` API.
    ///
    /// This value is not required to be computed from a CSPRNG, and may even be
    /// entirely deterministic. Host implementatations are encouraged to provide
    /// random values to any program exposed to attacker-controlled content, to
    /// enable DoS protection built into many languages' hash-map implementations.
    insecure-random: tuple<u64, u64>

    /// Returns whether or not the descriptor is an entropy source.
    is-random: func(random: descriptor) -> bool

    /// Return a random `u64`.
    ///
    /// This function must produce data from an adaquately seeded CSPRNG, so it
    /// must not block, and the returned data is always unpredictable.
    get-random: func(random: descriptor) -> result<u64, errno>

    /// Return `len` random bytes.
    ///
    /// This function must produce data from an adaquately seeded CSPRNG, so it
    /// must not block, and the returned data is always unpredictable.
    get-random-bytes: func(random: descriptor, len: usize) -> result<list<u8>, errno>
}
```

# wasi-poll

This interface is mostly unmodified from previous interfaces. The one minor
exception is that `poll-oneoff()` can return an error. I'm not sure why this
wasn't already possible. So either it was overlooked or I have missed something.

```
interface wasi-poll {
    use { descriptor, errno } from wasi-core
    use { duration } from wasi-clock

    /// The type of event to subscribe to.
    record subscription {
        /// Information about the subscription.
        info: subscription-info,

        /// The value of the `userdata` to include in associated events.
        userdata: u64,
    }

    /// Information about events to subscribe to.
    variant subscription-info {
        /// Set a monotonic clock timer.
        monotonic-clock-timeout(monotonic-clock-timeout),

        /// Set a wall clock timer.
        wall-clock-timeout(wall-clock-timeout),

        /// Wait for a writeable stream to be ready to accept data.
        write(descriptor),

        /// Wait for a readable stream to have data ready.
        read(descriptor),
    }

    /// Information about a monotonic clock timeout.
    record monotonic-clock-timeout {
        /// An absolute or relative timestamp.
        timeout: duration,

        /// Specifies an absolute, rather than relative, timeout.
        is-absolute: bool,
    }

    /// Information about a wall clock timeout.
    record wall-clock-timeout {
        /// An absolute or relative timestamp.
        timeout: duration,

        /// Specifies an absolute, rather than relative, timeout.
        is-absolute: bool,
    }

    /// An event which has occurred.
    record event {
        /// The value of the `userdata` from the associated subscription.
        userdata: u64,

        /// Information about the event.
        info: event-info,
    }

    /// Information about an event which has occurred.
    variant event-info {
        /// A monotonic clock timer expired.
        monotonic-clock-timeout,

        /// A wall clock timer expired.
        wall-clock-timeout,

        /// A readable stream has data ready.
        read(read-event),

        /// A writable stream is ready to accept data.
        write(write-event),
    }

    /// An event indicating that a readable stream has data ready.
    record read-event {
        /// The number of bytes ready to be read.
        nbytes: u64,

        /// Indicates the other end of the stream has disconnected and no further
        /// data will be available on this stream.
        is-closed: bool,
    }

    /// An event indicating that a writeable stream is ready to accept data.
    record write-event {
        /// The number of bytes ready to be accepted
        nbytes: u64,

        /// Indicates the other end of the stream has disconnected and no further
        /// data will be accepted on this stream.
        is-closed: bool,
    }

    poll-oneoff: func(in: list<subscription>) -> result<list<event>, errno>
}
```

# wasi-file

This interface represents reading or writing a file to disk. It borrows heavily
from previous interfaces but commits to having no runtime-stored position. This
move was started in a previous iteration by @sunfishcode; I just completed it.

Note that the biggest change to this interface is that it split out `wasi-dir`
into a separate interface.

```wit
interface wasi-file {
    use { descriptor, errno, usize } from wasi-core
    use { duration } from wasi-clock

    /// Non-negative file size or length of a region within a file.
    type filesize = u64

    /// File flags.
    ///
    /// Note: This was called `fd-flags` in earlier versions of WASI.
    flags %flags {
        /// Read mode: Data can be read.
        read,

        /// Write mode: Data can be written to.
        write,

        /// Append mode: Data written to the file is always appended to the file's
        /// end.
        append,

        /// Write according to synchronized I/O data integrity completion. Only the
        /// data stored in the file is synchronized.
        dsync,

        /// Non-blocking mode.
        nonblock,

        /// Synchronized read I/O operations.
        rsync,

        /// Write according to synchronized I/O file integrity completion. In
        /// addition to synchronizing the data stored in the file, the
        /// implementation may also synchronously update the file's metadata.
        sync,
    }

    /// File or memory access pattern advisory information.
    enum advice {
        /// The application has no advice to give on its behavior with respect to the specified data.
        normal,

        /// The application expects to access the specified data sequentially from lower offsets to higher offsets.
        sequential,

        /// The application expects to access the specified data in a random order.
        random,

        /// The application expects to access the specified data in the near future.
        will-need,

        /// The application expects that it will not access the specified data in the near future.
        dont-need,

        /// The application expects to access the specified data once and then not reuse it thereafter.
        no-reuse,
    }

    /// When setting a timestamp, this gives the value to set it to.
    variant new-timestamp {
        /// Leave the timestamp set to its previous value.
        no-change,

        /// Set the timestamp to the current time of the system clock associated
        /// with the filesystem.
        now,

        /// Set the timestamp to the given value as time since the Unix epoch.
        timestamp(duration),
    }

    /// Returns whether or not the descriptor is a file.
    is-file: func(file: descriptor) -> bool

    /// Provide file advisory information on a file.
    ///
    /// This is similar to `posix_fadvise` in POSIX.
    advise: func(
        file: descriptor,

        /// The offset within the file to which the advisory applies.
        offset: filesize,

        /// The length of the region to which the advisory applies.
        length: filesize,

        /// The advice.
        advice: advice
    ) -> result<_, errno>

    /// Force the allocation of space in a file.
    ///
    /// Note: This is similar to `posix_fallocate` in POSIX.
    allocate: func(
        file: descriptor,

        /// The offset at which to start the allocation.
        offset: filesize,

        /// The length of the area that is allocated.
        length: filesize
    ) -> result<_, errno>

    /// Get flags associated with a file.
    ///
    /// Note: This returns similar flags to `fcntl(fd, F_GETFL)` in POSIX.
    ///
    /// Note: This was called `fdstat_get` in earlier versions of WASI.
    get-flags: func(file: descriptor) -> result<%flags, errno>

    /// Read bytes from a file.
    ///
    /// Note: This is similar to `pread` in POSIX.
    read-at: func(
        /// The file from which to read.
        file: descriptor,

        /// The offset within the file at which to read.
        offset: filesize,

        /// The maximum number of bytes to read.
        length: usize,
    ) -> result<list<u8>, errno>

    /// Adjust the size of an open file. If this increases the file's size, the
    /// extra bytes are filled with zeros.
    ///
    /// Note: This was called `fd_filestat_set_size` in earlier versions of WASI.
    set-size: func(file: descriptor, size: filesize) -> result<_, errno>

    /// Adjust the timestamps of an open file or directory.
    ///
    /// Note: This is similar to `futimens` in POSIX.
    ///
    /// Note: This was called `fd_filestat_set_times` in earlier versions of WASI.
    set-times: func(
        file: descriptor,

        /// The desired values of the data access timestamp.
        atim: new-timestamp,

        /// The desired values of the data modification timestamp.
        mtim: new-timestamp,
    ) -> result<_, errno>

    /// Synchronize the data and metadata of a file to disk.
    ///
    /// Note: This is similar to `fsync` in POSIX.
    sync: func(file: descriptor) -> result<_, errno>

    /// Synchronize the data of a file to disk.
    ///
    /// Note: This is similar to `fdatasync` in POSIX.
    sync-data: func(file: descriptor) -> result<_, errno>

    /// Write bytes to a file.
    ///
    /// Note: This is similar to `pwrite` in POSIX.
    write-at: func(
        /// The file into which to write.
        file: descriptor,

        /// The offset within the file at which to write.
        offset: filesize,

        /// Data to write
        buf: list<u8>,
    ) -> result<usize, errno>
}
```

# wasi-dir

```wit
interface wasi-dir {
    use { %flags, new-timestamp, filesize } from wasi-file
    use { descriptor, errno } from wasi-core
    use { duration } from wasi-clock

    /// Identifier for a device containing a file system. Can be used in combination
    /// with `inode` to uniquely identify a file or directory in the filesystem.
    type device = u64

    /// Filesystem object serial number that is unique within its file system.
    type inode = u64

    /// Flags determining the method of how paths are resolved.
    flags at-flags {
        /// As long as the resolved path corresponds to a symbolic link, it is expanded.
        symlink-follow,
    }

    /// Open flags used by `open-at`.
    flags o-flags {
        /// Create file if it does not exist.
        create,

        /// Fail if not a directory.
        directory,

        /// Fail if file already exists.
        excl,

        /// Truncate file to size 0.
        trunc,
    }

    /// The type of a filesystem object referenced by a descriptor.
    ///
    /// Note: This was called `filetype` in earlier versions of WASI.
    enum %type {
        /// The type of the descriptor or file is unknown or is different from
        /// any of the other types specified.
        unknown,

        /// The descriptor refers to a block device inode.
        block-device,

        /// The descriptor refers to a character device inode.
        character-device,

        /// The descriptor refers to a directory inode.
        directory,

        /// The descriptor refers to a named pipe.
        fifo,

        /// The file refers to a symbolic link inode.
        symbolic-link,

        /// The descriptor refers to a regular file inode.
        regular-file,

        /// The descriptor refers to a socket.
        socket,
    }

    /// File attributes.
    ///
    /// Note: This was called `filestat` in earlier versions of WASI.
    record stat {
        /// Device ID of device containing the file.
        device: device,

        /// File serial number.
        inode: inode,

        /// File type.
        %type: %type,

        /// Number of hard links to the file.
        nlink: u64,

        /// For regular files, the file size in bytes. For symbolic links, the length
        /// in bytes of the pathname contained in the symbolic link.
        size: filesize,

        /// Last data access.
        atime: duration,

        /// Last data modification.
        mtime: duration,

        /// Last file status change .
        ctime: duration,
    }

    /// Permissions mode used by `open-at`, `change-permissions-at`, and similar.
    flags mode {
        /// True if the resource is considered readable by the containing
        /// filesystem.
        readable,

        /// True if the resource is considered writeable by the containing
        /// filesystem.
        writeable,

        /// True if the resource is considered executable by the containing
        /// filesystem. This does not apply to directories.
        executable,
    }

    /// A directory entry.
    record dirent {
        /// The serial number of the file referred to by this directory entry.
        inode: inode,

        /// The length of the name of the directory entry.
        namelen: u64,

        /// The type of the file referred to by this directory entry.
        %type: %type,
    }

    /// Returns whether or not the descriptor is a directory.
    is-directory: func(dir: descriptor) -> bool

    /// Returns the guest path the descriptor represents.
    get-mount-path: func(dir: descriptor) -> result<string, errno>

    /// Read directory entries from a directory.
    ///
    /// When successful, the contents of the output buffer consist of a sequence of
    /// directory entries. Each directory entry consists of a `dirent` object,
    /// followed by `dirent::d_namlen` bytes holding the name of the directory
    /// entry.
    ///
    /// This function fills the output buffer as much as possible, potentially
    /// truncating the last directory entry. This allows the caller to grow its
    /// read buffer size in case it's too small to fit a single large directory
    /// entry, or skip the oversized directory entry.
    readdir: func(dir: descriptor) -> result<list<u8>, errno>

    /// Create a directory.
    ///
    /// Note: This is similar to `mkdirat` in POSIX.
    create-directory-at: func(dir: descriptor, path: string) -> result<_, errno>

    /// Return the attributes of a file or directory.
    ///
    /// Note: This is similar to `fstatat` in POSIX.
    ///
    /// Note: This was called `fd_filestat_get` in earlier versions of WASI.
    stat-at: func(
        dir: descriptor,

        /// Flags determining the method of how the path is resolved.
        at-flags: at-flags,

        /// The relative path of the file or directory to inspect.
        path: string,
    ) -> result<stat, errno>

    /// Adjust the timestamps of a file or directory.
    ///
    /// Note: This is similar to `utimensat` in POSIX.
    ///
    /// Note: This was called `path_filestat_set_times` in earlier versions of WASI.
    set-times-at: func(
        dir: descriptor,

        /// Flags determining the method of how the path is resolved.
        at-flags: at-flags,

        /// The relative path of the file or directory to operate on.
        path: string,

        /// The desired values of the data access timestamp.
        atim: new-timestamp,

        /// The desired values of the data modification timestamp.
        mtim: new-timestamp,
    ) -> result<_, errno>

    /// Create a hard link.
    ///
    /// Note: This is similar to `linkat` in POSIX.
    link-at: func(
        dir: descriptor,

        /// Flags determining the method of how the path is resolved.
        old-at-flags: at-flags,

        /// The relative source path from which to link.
        old-path: string,

        /// The base directory for `new-path`.
        new-descriptor: descriptor,

        /// The relative destination path at which to create the hard link.
        new-path: string,
    ) -> result<_, errno>

    /// Open a file or directory.
    ///
    /// The returned descriptor is not guaranteed to be the lowest-numbered
    /// descriptor not currently open/ it is randomized to prevent applications
    /// from depending on making assumptions about indexes, since this is
    /// error-prone in multi-threaded contexts. The returned descriptor is
    /// guaranteed to be less than 2**31.
    ///
    /// Note: This is similar to `openat` in POSIX.
    open-at: func(
        dir: descriptor,

        /// Flags determining the method of how the path is resolved.
        at-flags: at-flags,

        /// The relative path of the object to open.
        path: string,

        /// The method by which to open the file.
        o-flags: o-flags,

        /// Flags to use for the resulting descriptor.
        %flags: %flags,

        /// Permissions to use when creating a new file.
        mode: mode
    ) -> result<descriptor, errno>

    /// Read the contents of a symbolic link.
    ///
    /// Note: This is similar to `readlinkat` in POSIX.
    readlink-at: func(
        dir: descriptor,

        /// The relative path of the symbolic link from which to read.
        path: string,
    ) -> result<string, errno>

    /// Remove a directory.
    ///
    /// Return `errno::notempty` if the directory is not empty.
    ///
    /// Note: This is similar to `unlinkat(fd, path, AT_REMOVEDIR)` in POSIX.
    remove-directory-at: func(
        dir: descriptor,

        /// The relative path to a directory to remove.
        path: string,
    ) -> result<_, errno>

    /// Rename a filesystem object.
    ///
    /// Note: This is similar to `renameat` in POSIX.
    rename-at: func(
        dir: descriptor,

        /// The relative source path of the file or directory to rename.
        old-path: string,

        /// The base directory for `new-path`.
        new-descriptor: descriptor,

        /// The relative destination path to which to rename the file or directory.
        new-path: string,
    ) -> result<_, errno>

    /// Create a symbolic link.
    ///
    /// Note: This is similar to `symlinkat` in POSIX.
    symlink-at: func(
        dir: descriptor,

        /// The contents of the symbolic link.
        old-path: string,

        /// The relative destination path at which to create the symbolic link.
        new-path: string,
    ) -> result<_, errno>

    /// Unlink a filesystem object that is not a directory.
    ///
    /// Return `errno::isdir` if the path refers to a directory.
    /// Note: This is similar to `unlinkat(fd, path, 0)` in POSIX.
    unlink-file-at: func(dir: descriptor, path: string) -> result<_, errno>

    /// Change the permissions of a filesystem object that is not a directory.
    ///
    /// Note that the ultimate meanings of these permissions is
    /// filesystem-specific.
    ///
    /// Note: This is similar to `fchmodat` in POSIX.
    change-file-permissions-at: func(
        dir: descriptor,

        /// Flags determining the method of how the path is resolved.
        at-flags: at-flags,

        /// The relative path to operate on.
        path: string,

        /// The new permissions for the filesystem object.
        mode: mode,
    ) -> result<_, errno>

    /// Change the permissions of a directory.
    ///
    /// Note that the ultimate meanings of these permissions is
    /// filesystem-specific.
    ///
    /// Unlike in POSIX, the `executable` flag is not reinterpreted as a "search"
    /// flag. `read` on a directory implies readability and searchability, and
    /// `execute` is not valid for directories.
    ///
    /// Note: This is similar to `fchmodat` in POSIX.
    change-directory-permissions-at: func(
        dir: descriptor,

        /// Flags determining the method of how the path is resolved.
        at-flags: at-flags,

        /// The relative path to operate on.
        path: string,

        /// The new permissions for the directory.
        mode: mode,
    ) -> result<_, errno>
}
```

# Networking

The following set of interfaces extends WASI to support networking.

## wasi-net

This interface defines an opaque descriptor type that represents a particular
network. This enables context-based security for networking.

```wit
interface wasi-net {
    use { descriptor, errno } from wasi-core;

    /// Returns whether or not the descriptor is a network.
    is-network: func(network: descriptor) -> bool
}
```

## wasi-ip

This interface defines IP-address-related data structures. Note that I chose
to represent addresses as fixed tuples rather than other representations. This
avoids allocations and makes it trivial to convert the addresses to strings.

```wit
interface wasi-ip {
    type ipv4-address = tuple<u8, u8, u8, u8>
    type ipv6-address = tuple<u8, u8, u8, u8, u8, u8, u8, u8, u8, u8, u8, u8, u8, u8, u8, u8>

    enum ip-address-family {
        /// Similar to `AF_INET` in POSIX.
        ipv4,

        /// Similar to `AF_INET6` in POSIX.
        ipv6,
    }

    variant ip-address {
        ipv4(ipv4-address),
        ipv6(ipv6-address),
    }

    record ipv4-socket-address {
        address: ipv4-address,
        port: u16,
    }

    record ipv6-socket-address {
        address: ipv6-address,
        port: u16,
        flow-info: u32,
        scope-id: u32,
    }

    variant ip-socket-address {
        ipv4(ipv4-socket-address),
        ipv6(ipv6-socket-address),
    }
}
```

## wasi-dns

This interface allows callers to perform DNS resolution. Since the existing
definition in `wasi-sockets` uses currently unsupported `wit` features, I have
rewritten this interface to be usable on existing features.

First, you call `resolve-name()` to create a resolver descriptor. Then, you
call `resolve-next(resolver)` in a loop to fetch the IP addresses. Finally,
you destroy the resolver with `close()` (like for all descriptors). This
structure allows for non-blocking operation in conjunction with `poll-oneoff()`.

```wit
interface wasi-dns {
    use { ip-address-family, ip-address } from wasi-ip
    use { descriptor, errno } from wasi-core

    /// Resolution flags.
    flags resolver-flags {
        /// Equivalent to O_NONBLOCK.
        nonblock,
    }

    /// Returns whether or not the descriptor is a resolver.
    is-resolver: func(listener: descriptor) -> bool

    /// Starts resolving an internet host name to a list of IP addresses.
    ///
    /// This function returns a new resolver on success or an error if
    /// immediately available. For example, this function returns
    /// `errno::inval` when `name` is:
    ///   - empty
    ///   - an IP address
    ///   - a syntactically invalid domain name in another way
    ///
    /// The resolver should be destroyed with `close()` when no longer in use.
    resolve-name: func(
        network: descriptor,

        /// The name to look up.
        ///
        /// IP addresses are not allowed and will return `errno::inval`.
        ///
        /// Unicode domain names are automatically converted to ASCII using
        /// IDNA encoding.
        name: string,

        /// If provided, limit the results the specified address family.
        address-family: option<ip-address-family>,

        /// Flags controlling the behavior of the name resolution.
        %flags: resolver-flags
    ) -> result<descriptor, errno>

    /// Get the next address from the resolver.
    ///
    /// This function should be called multiple times. On each call, it will
    /// return the next address in connection order preference. If all
    /// addresses have been exhausted, this function returns `none`. If
    /// non-blocking mode is used, this function may return `errno::again`
    /// indicating that the caller should poll for incoming data.
    /// This function never returns IPv4-mapped IPv6 addresses.
    resolve-next: func(resolver: descriptor) -> result<option<ip-address>, errno>
}
```

## wasi-tcp

This interface represents the core of TCP networking. There are two types of
TCP descriptors:

  1. a listener, which produces new incoming connections; and ...
  2. a connection, which actually transfers data to remote parties.

Note that this interface does not implement `bind()`. Binding inputs happen
as part of the `listen()` or `connect()` calls.

Non-blocking connections should be supported by calling `connect()` with the
`nonblock` flag. Then you poll on the resulting connection descriptor for
**output**. Once poll returns, you can call `get-remote-address()` to determine
the connection status and `receive(connection, 1)` to collect the error status.
This follows the existing usage under POSIX.

```
interface wasi-tcp {
    use { descriptor, errno } from wasi-core;
    use { ip-socket-address } from wasi-ip

    /// Listener flags.
    flags listener-flags {
        /// Equivalent to O_NONBLOCK.
        nonblock,
    }

    /// Connection flags.
    flags connection-flags {
        /// Equivalent to SO_KEEPALIVE.
        keepalive,

        /// Equivalent to O_NONBLOCK.
        nonblock,

        /// Equivalent to TCP_NODELAY.
        nodelay,
    }

    /// Returns whether or not the descriptor is a TCP listener.
    is-tcp-listener: func(listener: descriptor) -> bool

    /// Returns whether or not the descriptor is a TCP connection.
    is-tcp-connection: func(connection: descriptor) -> bool

    /// Creates a new listener.
    ///
    /// If the IP address is zero (`0.0.0.0` in IPv4, `::` in IPv6), the
    /// implementation will decide which network address to bind to.
    ///
    /// If the TCP/UDP port is zero, the socket will be bound to an
    /// unspecified free port.
    ///
    /// The listener should be destroyed with `close()` when no longer in use.
    listen: func(
        network: descriptor,
        address: ip-socket-address,
        backlog: option<usize>,
        %flags: listener-flags
    ) -> result<descriptor, errno>

    /// Accepts a new incoming connection.
    ///
    /// When in non-blocking mode, this function will return `errno::again`
    /// when no new incoming connection is immediately available. This is an
    /// indication to poll for incoming data on the listener. Otherwise, this
    /// function will block until an incoming connection is available.
    accept: func(
        listener: descriptor,
        %flags: connection-flags
    ) -> result<tuple<descriptor, ip-socket-address>, errno>

    /// Connect to a remote endpoint.
    ///
    /// If the local IP address is zero (`0.0.0.0` in IPv4, `::` in IPv6), the
    /// implementation will decide which network address to bind to.
    ///
    /// If the local TCP/UDP port is zero, the socket will be bound to an
    /// unspecified free port.
    ///
    ///  References
    /// - https://pubs.opengroup.org/onlinepubs/9699919799/functions/bind.html
    /// - https://man7.org/linux/man-pages/man2/bind.2.html
    /// - https://pubs.opengroup.org/onlinepubs/9699919799/functions/connect.html
    /// - https://man7.org/linux/man-pages/man2/connect.2.html
    connect: func(
        network: descriptor,
        local-address: ip-socket-address,
        remote-address: ip-socket-address,
        %flags: connection-flags,
    ) -> result<descriptor, errno>

    /// Send bytes to the remote connection.
    ///
    /// This function may not successfully send all bytes. Check the number of
    /// bytes returned.
    ///
    /// Note: This is similar to `pwrite` in POSIX.
    send: func(connection: descriptor, bytes: list<u8>) -> result<usize, errno>

    /// Receive bytes from the remote connection.
    ///
    /// This function receives **at most** `length` bytes from the remote
    /// connection.
    ///
    /// Note: This is similar to `recv` in POSIX.
    receive: func(connection: descriptor, length: usize) -> result<list<u8>, errno>

    /// Get the current bound address.
    ///
    /// Note: this function can be called on either a listener or a connection.
    ///
    /// References
    /// - https://pubs.opengroup.org/onlinepubs/9699919799/functions/getsockname.html
    /// - https://man7.org/linux/man-pages/man2/getsockname.2.html
    get-local-address: func(socket: descriptor) -> result<ip-socket-address, errno>

    /// Get the remote address.
    ///
    /// References
    /// - https://pubs.opengroup.org/onlinepubs/9699919799/functions/getpeername.html
    /// - https://man7.org/linux/man-pages/man2/getpeername.2.html
    get-remote-address: func(connection: descriptor) -> result<ip-socket-address, errno>

    /// Get the flags set for the connection.
    get-flags: func(connection: descriptor) -> result<connection-flags, errno>

    /// Sets the flags for the connection.
    set-flags: func(connection: descriptor, %flags: connection-flags) -> result<_, errno>

    /// Gets the receive-buffer size.
    ///
    /// Note: this is only a hint. Implementations may internally handle this
    /// in any way, including ignoring it.
    ///
    /// Equivalent to SO_RCVBUF.
    get-receive-buffer-size: func(connection: descriptor) -> result<usize, errno>

    /// Gets the receive-buffer size.
    ///
    /// Note: this is only a hint. Implementations may internally handle this
    /// in any way, including ignoring it.
    ///
    /// Equivalent to SO_RCVBUF.
    set-receive-buffer-size: func(connection: descriptor, value: usize) -> result<_, errno>

    /// Gets the send-buffer size.
    ///
    /// Note: this is only a hint. Implementations may internally handle this
    /// in any way, including ignoring it.
    ///
    /// Equivalent to SO_SNDBUF.
    get-send-buffer-size: func(connection: descriptor) -> result<usize, errno>

    /// Sets the send-buffer size.
    ///
    /// Note: this is only a hint. Implementations may internally handle this
    /// in any way, including ignoring it.
    ///
    /// Equivalent to SO_SNDBUF.
    set-send-buffer-size: func(connection: descriptor, value: usize) -> result<_, errno>
}
```

# World

This world definition catalogs the interfaces available under preview2.

```wit
world wasi-snapshot-preview2 {
    export core: wasi-core
    export logging: wasi-log
    export clock: wasi-clock
    export random: wasi-random
    export poll: wasi-poll
    export file: wasi-file
    export dir: wasi-dir
    export net: wasi-net
    export ip: wasi-ip
    export tcp: wasi-tcp
}
```
