This is a simple configuration script use for some
[BSD.lv](https://www.bsd.lv) project sources.
Its mission is to provide [OpenBSD](https://www.openbsd.org) portability
functions and feature testing.

It allows easy porting to Linux (glibc and musl), FreeBSD, NetBSD, Mac
OS X, SunOS, and OmniOS (illumos).
The continuity of this portability is maintained by BSD.lv's
[continuous integration](https://kristaps.bsd.lv/cgi-bin/minci.cgi/index.html?project-name=oconfigure)
system.  Other systems may also be supported: please let us know if they are.

See [versions.md](versions.md) for version information.

To use:

1. copy
[configure](https://raw.githubusercontent.com/kristapsdz/oconfigure/master/configure),
[compats.c](https://raw.githubusercontent.com/kristapsdz/oconfigure/master/compats.c),
and
[tests.c](https://raw.githubusercontent.com/kristapsdz/oconfigure/master/tests.c)
into your source tree
2. have `include Makefile.configure` at the top of your Makefile
3. have `#include "config.h"` as the first inclusion in your sources
4. read over the documentation below in case you need to guard header inclusion
5. compile compats.o and link with it

Source users run `./configure` prior to running `make`.  The `configure`
script will check for common features as noted in the test files, e.g.,
[pledge(2)](https://man.openbsd.org/pledge.2), and also provide
compatibility for other functions, e.g.,
[strlcpy(3)](https://man.openbsd.org/strlcpy.3).

The `./configure` script may be executed in a cross-compiling
environment with the compiler and linker set appropriately.

If you have Makefile flags you'd like to set, set them when you invoke
`configure` as key-value pairs on the command-line, e.g.,

```
./configure PREFIX=/opt
```

The following flags are recognised and accepted: `LDADD`, `LDFLAGS`,
`CPPFLAGS`, `DESTDIR`, `PREFIX`, `MANDIR`, `LIBDIR`, `BINDIR`,
`SHAREDIR`, `SBINDIR`, and `INCLUDEDIR`.  Un-recognised flags are
discarded and warned about.

If you want to use an alternative CC or CFLAGS, specify them as an
environmental variable.  If the compiler is not found, **oconfigure**
will try to locate `clang` and `gcc` before giving up.

```
CC=musl-gcc ./configure
```

Using **oconfigure** requires some work within your sources to node
compatibility areas, then some in your build environment:

```c
#include "config.h" /* required inclusion */

#if HAVE_ERR /* sometimes err.h exists, sometimes not */
# include <err.h>
#endif
#include <stdio.h>
#include <unistd.h>

int main(void) {
#if HAVE_PLEDGE /* do we have pledge? */
	if (pledge("stdio", NULL) == -1)
		err(EXIT_FAILURE, NULL);
#endif
	warnx("hello, world!"); /* compat provides this */
	return 0;
}
```

And then...

```sh
./configure
cc -o config.o -c config.c
cc -o main.o -c main.c
cc config.o main.o
```

It's better to build this into your Makefile, as the output
*Makefile.configure* will set compiler, compiler flags, installation
utilities, and so on.

This framework was inspired by [mandoc](https://mandoc.bsd.lv)'s
`configure` script written by Ingo Schwarze.

What follows is a description of the features and facilities provided by
the package when included into your sources.

The compatibility layer is generally provided by the excellent portable
[OpenSSH](https://www.openssh.com/).  All copyrights are in their
respective sources.

## b64\_ntop

This and its partner `b64_pton` are sometimes declared but not defined.
The following will guard against that in your sources.

```c
#if HAVE_B64_NTOP
# include <netinet/in.h>
# include <resolv.h>
#endif
```

Some systems (Linux in particular) with `HAVE_B64_NTOP` need `-lresolv`
during linking.  If so, set `LDADD_B64_NTOP` in *Makefile.configure* to
`-lresolv`.  Otherwise it is empty.

If the functions are not found, provides compatibility functions
`b64_ntop` and `b64_pton`.

Since these functions are always poorly documented, the following
demonstrates usage for `b64_ntop`, which translates `src` *into* an
encoded NUL-terminated string `dst` and returns the string length or -1.
The `dstsz` is the maximum size required for encoding.

```c
srcsz = strlen(src);
dstsz = ((srcsz + 2) / 3 * 4) + 1;
dst = malloc(dstsz);
if (b64_ntop(src, srcsz, dst, dstsz) == -1)
	goto bad_size;
```

`b64_pton` reverses this situation from an encoded NUL-terminated string
`src` into the decoded and NUL-terminated string `dst` (it's common not
to need the NUL-terminator for the decoded string, which is meant to be
binary).  The `dstsz` is the maximum size required for decoding.

```c
srcsz = strlen(src);
dstsz = srscsz / 4 * 3;
dst = malloc(dstsz + 1); /* NUL terminator */
if ((c = b64_pton(src, dst, dstsz)) == -1)
	goto bad_data;
dst[c] = '\0'; /* NUL termination */
```

## Capsicum

Tests for [FreeBSD](https://www.freebsd.org)'s
[Capsicum](https://www.freebsd.org/cgi/man.cgi?capsicum(4)) subsystem,
defining the `HAVE_CAPSICUM` variable with the result.
Does not provide any compatibility.

```c
#if HAVE_CAPSICUM
# include <sys/resource.h>
# include <sys/capsicum.h>
#endif
```

The guard is required for systems without these headers.

## endian.h

On most operating systems (Linux, OpenBSD), *endian.h* provides the
POSIX.1 endian functions, e.g.,
[htole32(3)](https://man.openbsd.org/htole32.3),
[be32toh(3)](https://man.openbsd.org/be32toh.3), etc.
On FreeBSD, however, these are in *sys/endian.h*.
Mac OS X and SunOS have their own functions in their own places.

The required invocation to use the endian functions is:

```c
#if HAVE_ENDIAN_H
# include <endian.h>
#elif HAVE_SYS_ENDIAN_H
# include <sys/endian.h>
#elif HAVE_OSBYTEORDER_H
# include <libkern/OSByteOrder.h>
#elif HAVE_SYS_BYTEORDER_H
# include <sys/byteorder.h>
#endif
```

Compatibility for the Mac OS X and SunOS functions to the usual
`htole32` style is provided.

To make this easier, the `COMPAT_ENDIAN_H` is also defined:

```c
#include COMPAT_ENDIAN_H
```

This will paste the appropriate location.

## err.h

Tests for the [err(3)](https://man.openbsd.org/err.3) functions,
defining `HAVE_ERR` variable with the result.  If not found, provides
compatibility functions `err`, `errx`, `warn`, `warnx`, `vwarn`,
`vwarnx`.

```c
#if HAVE_ERR
# include <err.h>
#endif
```

The *err.h* header needs to be guarded to prevent systems using the
compatibility functions for failing, as the header does not exist.

## expat(3)

Tests for expat(3) compilation and linking.  Defines `HAVE_EXPAT` if
found.  Does not provide any compatibility.

The `LDADD_EXPAT` value provided in *Makefile.configure* will be set to
`-lexpat` if found. Otherwise it is empty.

## explicit\_bzero(3)

Tests for [explicit\_bzero(3)](https://man.openbsd.org/explicit_bzero.3)
in *string.h*, defining `HAVE_EXPLICIT_BZERO` variable with the result.

```c
#include <string.h> /* explicit_bzero */
```

If not found, provides a compatibility function.  The compatibility
layer will use
[memset\_s](http://en.cppreference.com/w/c/string/byte/memset), if
found.  `HAVE_EXPLICIT_BZERO` shouldn't be directly used in most
circumstances.

## getprogname(3)

Tests for [getprogname(3)](https://man.openbsd.org/getprogname.3) in
*stdlib.h*, defining `HAVE_GETPROGNAME` with the result.  Provides a
compatibility function if not found.

```c
#include <stdlib.h> /* getprogname */
```

The compatibility function tries to use `__progname`,
`program_invocation_short_name`, or `getexecname()`.  If none of these
interfaces may be found, it will emit a compile-time error.
`HAVE_GETPROGNAME` shouldn't be directly used in most circumstances.

## INFTIM

Tests for `INFTIM` in [poll(2)](https://man.openbsd.org/poll.2),
defining `HAVE_INFTIM` with the result.  Provides a compatibility
value if not found.

```c
#include <poll.h> /* INFTIM */
```

Since a compatibility function is provided, `HAVE_INFTIM` shouldn't be
directly used in most circumstances.

## libsocket

On IllumOS-based distributions, all socket functions
([bind(2)](https://man.openbsd.org/bind.2),
[listen(2)](https://man.openbsd.org/listen.2),
[socketpair(2)](https://man.openbsd.org/socketpair.2), etc.)
require linking to the `-lsocket` and `-lnsl` libraries.

If this is required, the `LDADD_LIB_SOCKET` variable in *Makefile.configure*
will be set to the required libraries.

## md5.h

Tests for the standalone [md5(3)](https://man.openbsd.org/md5.3)
functions, defining `HAVE_MD5` with the result.

If not found, provides a full complement of standalone (i.e., not
needing any crypto libraries) MD5 hashing functions.  These are
`MD5Init`, `MD5Update`, `MD5Pad`, `MD5Transform`, `MD5End`, and
`MD5Final`.  The preprocessor macros `MD5_BLOCK_LENGTH`,
`MD5_DIGEST_LENGTH`, and `MD5_DIGEST_STRING_LENGTH` are also defined.  

These differ ever-so-slightly from the OpenBSD versions in that they use
C99 types for greater portability, e.g., `uint8_t` instead of
`u_int8_t`.

If using these functions, you'll want to guard an inclusion of the
system-default.  Otherwise a partial *md5.h* may conflict with results,
or a missing *md5.h* may terminate compilation.

```c
#if HAVE_MD5
# include <sys/types.h>
# include <md5.h>
#endif
```

On some systems (FreeBSD in particular) with `HAVE_MD5`, you'll
also need to add `-lmd` when you compile your system, else it will
fail with undefined references.

The `LDADD_MD5` value provided in *Makefile.configure* will be set to
`-lmd` if it's required. Otherwise it is empty.

## memmem(3)

Tests for [memmem(3)](https://man.openbsd.org/memmem.3) in *string.h*,
defining `HAVE_MEMMEM` with the result.  Provides a compatibility
function if not found.

```c
#include <string.h> /* memmem */
```

Since a compatibility function is provided, `HAVE_MEMMEM` shouldn't be
directly used in most circumstances.

## memrchr(3)

Tests for [memrchr(3)](https://man.openbsd.org/memrchr.3) in *string.h*,
defining `HAVE_MEMRCHR` with the result.  Provides a compatibility
function if not found.

```c
#include <string.h> /* memrchr */
```

Since a compatibility function is provided, `HAVE_MEMRCHR` shouldn't be
directly used in most circumstances.

## minor(2)

[major(2)](https://man.openbsd.org/major.2),
[minor(2)](https://man.openbsd.org/minor.2),
and
[makedev(2)](https://man.openbsd.org/makedev.2)
all live in different places on different systems.

```c
#if HAVE_SYS_MKDEV_H
# include <sys/types.h> /* dev_t */
# include <sys/mkdev.h> /* minor/major/makedev */
#elif HAVE_SYS_SYSMACROS_H
# include <sys/sysmacros.h> /* minor/major/makedev */
#else
# include <sys/types.h> /* minor/major/makedev */
#endif
```

This can be made much easier as follows, where `COMPAT_MAJOR_MINOR_H` is
set to one of the above.  *sys/types.h* may be included twice.

```c
#include <sys/types.h>
#include COMPAT_MAJOR_MINOR_H
```

## mkfifoat(2)

Tests for the [mkfifoat(3)](https://man.openbsd.org/mkfifoat.3)
function, defining `HAVE_MKFIFOAT` with the result.
Provides a compatibility function if not found.

This is *not* a direct replacement, as the function is not atomic: it
internally gets a reference to the current directory, changes into the
"at" directory, runs the function, then returns to the prior current.

Upon errors, it makes a best effort to restore the current working
directory to what it was.

## mknodat(2)

Tests for the [mknodat(3)](https://man.openbsd.org/mknodat.3)
function, defining `HAVE_MKNODAT` with the result.
Provides a compatibility function if not found.

This is *not* a direct replacement, as the function is not atomic: it
internally gets a reference to the current directory, changes into the
"at" directory, runs the function, then returns to the prior current.

Upon errors, it makes a best effort to restore the current working
directory to what it was.

## PATH\_MAX

Tests for the `PATH_MAX` variable in *limits.h*, defining
`HAVE_PATH_MAX` with the result.  If not found, defines the `PATH_MAX`
macro to be 4096.

```c
#include <limits.h> /* PATH_MAX */
```

Since a compatibility value is provided, `HAVE_PATH_MAX` shouldn't be
directly used in most circumstances.

## pledge(2)

Test for [pledge(2)](https://man.openbsd.org/pledge.2), defining `HAVE_PLEDGE`
with the result.  Does not provide any compatibility.

```c
#include <unistd.h> /* pledge */
```

The `HAVE_PLEDGE` guard is not required except around the function invocation.

## readpassphrase(3)

Tests for the [readpassphrase(3)](https://man.openbsd.org/readpassphrase.3)
function, defining `HAVE_READPASSPHRASE` with the result.
Provides a compatibility function if not found.
The `<readpassphrase.h>` header inclusion needs to be
guarded for systems that include it by default; otherwise, the
definitions are provided in the generated `config.h`:

```c
#if HAVE_READPASSPHRASE
# include <readpassphrase.h>
#endif
```

If using this function, makes sure you explicitly zero the passphrase
buffer as described in
[readpassphrase(3)](https://man.openbsd.org/readpassphrase.3).

## reallocarray(3)

Tests for [reallocarray(3)](https://man.openbsd.org/reallocarray.3) in
*stdlib.h*, defining `HAVE_REALLOCARRAY` with the result.  Provides a
compatibility function if not found.

```c
#include <stdlib.h> /* reallocarray */
```

Since a compatibility function is provided, `HAVE_REALLOCARRAY` shouldn't be
directly used in most circumstances.

## recallocarray(3)

Tests for [recallocarray(3)](https://man.openbsd.org/recallocarray.3) in
*stdlib.h*, defining `HAVE_RECALLOCARRAY` with the result.  Provides a
compatibility function if not found.

```c
#include <stdlib.h> /* recallocarray */
```

Since a compatibility function is provided, `HAVE_RECALLOCARRAY` shouldn't be
directly used in most circumstances.

## sandbox\_init(3)

Tests for
[sandbox\_init(3)](https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man3/sandbox_init.3.html),
defining `HAVE_SANDBOX_INIT` with the result.
Does not provide any compatibility.

## seccomp-filter(3)

Tests for Linux's
[prctl(2)](http://man7.org/linux/man-pages/man2/prctl.2.html) function,
which is the gateway for
[seccomp(2)](http://man7.org/linux/man-pages/man2/seccomp.2.html).
Defines `HAVE_SECCOMP_FILTER` if found.
Does not provide any compatibility.

*Note: this test does not mean that the sandboxing is enabled.* You'll
need to perform a run-time check for `prctl`'s return value in your
sources.

## SOCK\_NONBLOCK

Tests for [socketpair(2)](https://man.openbsd.org/socketpair.2) in
*sys/socket.h* supporting the `SOCK_NONBLOCK` mask as found on OpenBSD.
Defines the `HAVE_SOCK_NONBLOCK` variable.

```c
#if HAVE_SOCK_NONBLOCK
	socketpair(AF_UNIX, flags|SOCK_NONBLOCK, 0, fd);
#else
	socketpair(AF_UNIX, flags, 0, fd);
	fcntl(fd[0], F_SETFL, 
	      fcntl(fd[0], F_GETFL, 0)|O_NONBLOCK);
	fcntl(fd[1], F_SETFL, 
	      fcntl(fd[1], F_GETFL, 0)|O_NONBLOCK);
#endif
```

The guard is not required only around the variable usage, not header
inclusion.  However, the above example could have the *fcntl.h* header
guarded by `!HAVE_SOCK_NONBLOCK`.

## strlcat(3)

Tests for [strlcat(3)](https://man.openbsd.org/strlcat.3) in *string.h*,
defining `HAVE_STRLCAT` with the result.  Provides a compatibility function if
not found.

```c
#include <string.h> /* strlcat */
```

Since a compatibility function is provided, `HAVE_STRLCAT` shouldn't be
directly used in most circumstances.

## strlcpy(3)

Tests for [strlcpy(3)](https://man.openbsd.org/strlcpy.3) in *string.h*,
defining `HAVE_STRLCPY` with the result.  Provides a compatibility function if
not found.

```c
#include <string.h> /* strlcpy */
```

Since a compatibility function is provided, `HAVE_STRLCPY` shouldn't be
directly used in most circumstances.

## strndup(3)

Tests for [strndup(3)](https://man.openbsd.org/strndup.3)
in *string.h*, defining `HAVE_STRNDUP` with the result.  Provides a
compatibility function if not found.

```c
#include <string.h> /* strndup */
```

Since a compatibility function is provided, `HAVE_STRNDUP` shouldn't be
directly used in most circumstances.

## strnlen(3)

Tests for [strnlen(3)](https://man.openbsd.org/strnlen.3) in *string.h*,
defining `HAVE_STRNLEN` with the result.  Provides a compatibility function if
not found.

```c
#include <string.h> /* strnlen */
```

Since a compatibility function is provided, `HAVE_STRNLEN` shouldn't be
directly used in most circumstances.

## strtonum(3)

Tests for [strtonum(3)](https://man.openbsd.org/strtonum.3) in *stdlib.h*,
defining `HAVE_STRTONUM` with the result.  Provides a compatibility function if
not found.

```c
#include <stdlib.h> /* strtonum */
```

Since a compatibility value is provided, `HAVE_STRTONUM` shouldn't be directly
used in most circumstances.

## sys/queue.h

Tests for the [queue(3)](https://man.openbsd.org/queue.3) header,
*sys/queue.h*.  Defines `HAVE_SYS_QUEUE` if found and provides all of
the queue macros if not.  To use these macros, make sure to guard
inclusion:

```c
#if HAVE_SYS_QUEUE
# include <sys/queue.h>
#endif
```

This uses `TAILQ_FOREACH_SAFE` as a basis for determining whether the
header exists and is well-formed.
This is because glibc provides a skeleton *sys/queue.h* without this
critical macro.

## sys/tree.h

Tests for the [tree(3)](https://man.openbsd.org/tree.3) header,
*sys/tree.h*.  Defines `HAVE_SYS_TREE` if found and provides all of the
tree macros if not.  To use these macros, make sure to guard
inclusion:

```c
#if HAVE_SYS_TREE
# include <sys/tree.h>
#endif
```

## systrace(4)

Tests for the deprecated systrace(4) interface.  Defines `HAVE_SYSTRACE` if
found.  Does not provide any compatibility.

This function is never "found".

## unveil(2)

Test for [unveil(2)](https://man.openbsd.org/unveil.2), defining `HAVE_UNVEIL`
with the result.  Does not provide any compatibility.

```c
#include <unistd.h> /* unveil */
```

The `HAVE_UNVEIL` guard is not required except around the function invocation.

## WAIT\_ANY

Tests for `WAIT_ANY` in [waitpid(2)](https://man.openbsd.org/waitpid.2),
defining `HAVE_WAIT_ANY` with the result.  Provides a compatibility
value for both `WAIT_ANY` and `WAIT_MYPGRP` if not found.

```c
#include <sys/wait.h> /* WAIT_ANY, WAIT_MYPGRP */
```

Since a compatibility function is provided, `HAVE_WAIT_ANY` shouldn't be
directly used in most circumstances.

## zlib(3)

Tests for zlib(3) compilation and linking.  Defines `HAVE_ZLIB` if
found.  Does not provide any compatibility.

The `LDADD_ZLIB` value provided in *Makefile.configure* will be set to
`-lz` if found. Otherwise it is empty.
