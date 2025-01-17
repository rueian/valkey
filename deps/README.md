This directory contains all Valkey dependencies, except for the libc that
should be provided by the operating system.

* **Jemalloc** is our memory allocator, used as replacement for libc malloc on Linux by default. It has good performances and excellent fragmentation behavior. This component is upgraded from time to time.
* **hiredis** is the official C client library for Redis. It is used by redis-cli, redis-benchmark and Redis Sentinel. It is part of the Redis official ecosystem but is developed externally from the Redis repository, so we just upgrade it as needed.
* **linenoise** is a readline replacement. It is developed by the same authors of Valkey but is managed as a separated project and updated as needed.
* **lua** is Lua 5.1 with minor changes for security and additional libraries.
* **hdr_histogram** Used for per-command latency tracking histograms.
* **fast_float** is a replacement for strtod to convert strings to floats efficiently. 

How to upgrade the above dependencies
===

Jemalloc
---

Jemalloc is modified with changes that allow us to implement the Valkey
active defragmentation logic. However this feature of Valkey is not mandatory
and Valkey is able to understand if the Jemalloc version it is compiled
against supports such Valkey-specific modifications. So in theory, if you
are not interested in the active defragmentation, you can replace Jemalloc
just following these steps:

1. Remove the jemalloc directory.
2. Substitute it with the new jemalloc source tree.
3. Edit the Makefile located in the same directory as the README you are
   reading, and change the --with-version in the Jemalloc configure script
   options with the version you are using. This is required because otherwise
   Jemalloc configuration script is broken and will not work nested in another
   git repository.

However note that we change Jemalloc settings via the `configure` script of Jemalloc using the `--with-lg-quantum` option, setting it to the value of 3 instead of 4. This provides us with more size classes that better suit the Valkey data structures, in order to gain memory efficiency.

If you want to upgrade Jemalloc while also providing support for
active defragmentation, in addition to the above steps you need to perform
the following additional steps:

5. In Jemalloc tree, file `include/jemalloc/jemalloc_macros.h.in`, make sure
   to add `#define JEMALLOC_FRAG_HINT`.
6. Implement the function `je_get_defrag_hint()` inside `src/jemalloc.c`. You
   can see how it is implemented in the current Jemalloc source tree shipped
   with Valkey, and rewrite it according to the new Jemalloc internals, if they
   changed, otherwise you could just copy the old implementation if you are
   upgrading just to a similar version of Jemalloc.

#### Updating/upgrading jemalloc

The jemalloc directory is pulled as a subtree from the upstream jemalloc github repo. To update it you should run from the project root:

1. `git subtree pull --prefix deps/jemalloc https://github.com/jemalloc/jemalloc.git <version-tag> --squash`<br>
This should hopefully merge the local changes into the new version.
2. In case any conflicts arise (due to our changes) you'll need to resolve them and commit.
3. Reconfigure jemalloc:<br>
```sh
rm deps/jemalloc/VERSION deps/jemalloc/configure
cd deps/jemalloc
./autogen.sh --with-version=<version-tag>-0-g0
```
4. Update jemalloc's version in `deps/Makefile`: search for "`--with-version=<old-version-tag>-0-g0`" and update it accordingly.
5. Commit the changes (VERSION,configure,Makefile).

Hiredis
---

Hiredis is used by Sentinel, `valkey-cli` and `valkey-benchmark`. Like Valkey, uses the SDS string library, but not necessarily the same version. In order to avoid conflicts, this version has all SDS identifiers prefixed by `hi`.

1. `git subtree pull --prefix deps/hiredis https://github.com/redis/hiredis.git <version-tag> --squash`<br>
This should hopefully merge the local changes into the new version.
2. Conflicts will arise (due to our changes) you'll need to resolve them and commit.

Linenoise
---

Linenoise is rarely upgraded as needed. The upgrade process is trivial since
Valkey uses a non modified version of linenoise, so to upgrade just do the
following:

1. Remove the linenoise directory.
2. Substitute it with the new linenoise source tree.

Lua
---

We use Lua 5.1 and no upgrade is planned currently, since we don't want to break
Lua scripts for new Lua features: in the context of Valkey Lua scripts the
capabilities of 5.1 are usually more than enough, the release is rock solid,
and we definitely don't want to break old scripts.

So upgrading of Lua is up to the Valkey project maintainers and should be a
manual procedure performed by taking a diff between the different versions.

Currently we have at least the following differences between official Lua 5.1
and our version:

1. Makefile is modified to allow a different compiler than GCC.
2. We have the implementation source code, and directly link to the following external libraries: `lua_cjson.o`, `lua_struct.o`, `lua_cmsgpack.o` and `lua_bit.o`.
3. There is a security fix in `ldo.c`, line 498: The check for `LUA_SIGNATURE[0]` is removed in order to avoid direct bytecode execution.
4. In `lstring.c`, the luaS_newlstr function's hash calculation has been upgraded from a simple hash function to MurmurHash3, implemented within the same file, to enhance performance, particularly for operations involving large strings.

Hdr_Histogram
---

Updated source can be found here: https://github.com/HdrHistogram/HdrHistogram_c
We use a customized version based on master branch commit e4448cf6d1cd08fff519812d3b1e58bd5a94ac42.
1. Compare all changes under /hdr_histogram directory to upstream master commit e4448cf6d1cd08fff519812d3b1e58bd5a94ac42
2. Copy updated files from newer version onto files in /hdr_histogram.
3. Apply the changes from 1 above to the updated files.

fast_float
---
The fast_float library provides fast header-only implementations for the C++ from_chars functions for `float` and `double` types as well as integer types.  These functions convert ASCII strings representing decimal values (e.g., `1.3e10`) into binary types. The functions are much faster than comparable number-parsing functions from existing C++ standard libraries.

Specifically, `fast_float` provides the following function to parse floating-point numbers with a C++17-like syntax (the library itself only requires C++11):

    template <typename T, typename UC = char, typename = FASTFLOAT_ENABLE_IF(is_supported_float_type<T>())>
    from_chars_result_t<UC> from_chars(UC const *first, UC const *last, T &value, chars_format fmt = chars_format::general);

To upgrade the library,
1. Check out https://github.com/fastfloat/fast_float/tree/main
2. cd fast_float
3. Invoke "python3 ./script/amalgamate.py --output fast_float.h"
4. Copy fast_float.h file to "deps/fast_float/".
