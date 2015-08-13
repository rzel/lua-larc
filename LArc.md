# lArc Library #
A Lua library for reading and writing compressed files in a variety of formats.

## Release History ##
No releases have been made.
### Version 1 ###
The goals for version 1 are:
  * Zlib compression engine.
  * Bzip2 compression engine.
  * Lzma compression engine.
  * Read and write gzip files.
  * Read and write bzip2 files.
  * Read and write xv files.
  * Read and write zip archives with stored, deflate, bzip2, and lzma compression.
  * Read and write zip64 archives.
  * Read and write tar archives, including compressed files.
  * Function to identify and open a zip or (possibly compressed) tar archive.
  * Freeze the library API.
  * Compatible with Lua 5.1.4 and 5.2.

## Installation ##
Modify the `makefile` to satisfy your build requirements. In particular,
you will need to set the correct `PLAT` variable, and set the correct
paths to the external libraries. The external libraries will also need to
be built beforehand. When building for MinGW, you are encouraged to use
static libraries so that you won't have to worry about distributing many
DLLs with your application. Operating systems with real package management
can be free to use shared libraries and trust that the prerequisites will
be found.

Create the C modules.
```
make
```
Then copy all the modules to where Lua can find them.
```
make install
```

Or perhaps you can use [LuaRocks](http://www.luarocks.org)?

## Library Objects ##
The library deals with compression and decompression engines, compressed file
streams, and archive files. Each type of object is implemented consistently
so an application can use any available format easily.

### Compression Engines ###
An engine module will export two functions for compression and two functions
for decompression.

#### engine.compress ####
| **Arguments** | string, options  |
|:--------------|:-----------------|
| **Returns**   | string, number of bytes compressed, status  |
| **On error**  | nil, error message, status  |

Compresses a string and returns the result with the number of bytes that were
compressed (which should be the length of the original string). If there is
an error then `nil` is returned with an error message and the status.

Different engines may use different status codes, but errors will always be
less than zero, and a status greater than or equal to zero will not be an error.

> This may be changed to require all engines to use the same status for `OK`
> and `STREAM_END`.

Options to the engine are given in a table where the keys are specific to the
engine being used. All engines should accept the key `level` to adjust the
strength of the compression. The value of `level` can be 1 (fastest and
least effective) through 9 (greatest compression), or 0 to choose a reasonable
default. The option table can be omitted, and engines must ignore keys that
it doesn't recognize.

#### engine.compressor ####
| **Arguments** | options  |
|:--------------|:---------|
| **Returns**   | function  |
| **On error**  | nil, error message, status  |

Creates a function for incrementally compressing strings. The compressor
function accepts a string argument and will return a string with the bytes
that it has compressed so far. To finish compressing, call the function
with `nil` for the argument and it will flush the output to produce the
final string. To read from one file handle and compress to another, you
would do this:
```lua

function compress_stream(src, dst, level)
local engine = compressor{level=level or 9}
repeat
local buffer = src:read(1000)
dst:write(assert(engine(buffer)))
until not buffer
```

When the file handle reaches the end of the file, the `read` method will
return `nil` which tells the engine to finish compression. Notice the use
of `assert` to catch any errors from the compression engine.

Once you call the compression function with a `nil` or no argument, you
should not attempt to call it again with more data.

The options to `compressor` are the same as for `compress`.

#### engine.decompress ####
| **Arguments** | string, options  |
|:--------------|:-----------------|
| **Returns**   | string, number of bytes decompressed, status  |
| **On error**  | nil, error message, status  |

Decompresses a string. The options and status number are similar to
`compress`.

It is possible that a string may be partially decompressed before an
error occurs. The engine will return the incomplete string and indicate
that an error occurred by the status number. If the status is less than
zero, then the decompression was not successful. The second return value,
the number of bytes decompressed, can be used to check if the original
string was not completely read. Although this may not be because of an
error, but merely the presence of extra bytes in the string.

#### engine.decompressor ####
| **Arguments** | options  |
|:--------------|:---------|
| **Returns**   | function  |
| **On error**  | nil, error message, status  |

Creates a function for incrementally decompressing strings. Valid options are
the same as for `decompress` and usage is similar to `compressor`.

It is not necessary to pass a `nil` argument in order to flush data. Engines
should always return as many bytes as are available after each call. However,
passing `nil` is not harmful and the return value will be an empty string.

### Compressed Streams ###
A file or file handle that reads or writes compressed data can be created for
transparent compression.

#### stream.open ####
| **Arguments** | file, mode, level  |
|:--------------|:-------------------|
| **Returns**   | handle             |
| **On error**  | nil, error message  |

#### stream:read ####

#### stream:write ####

#### stream:close ####

#### stream:seek ####

### Archive Files ###

#### archive.open ####

#### archive:names ####

#### archive:getinfo ####

#### archive:read ####

#### archive:open ####

#### archive:extract ####

#### archive:close ####

## Zip Loader ##
The `ziploader` module adds a package loader to the module system that
will look for Lua modules inside of zip archives.
```lua

ziploader = require"larc.ziploader"
table.insert(ziploader.archives, "/usr/local/share/lua/modules.zip")
-- Looks for "mymodules/widget.lua" inside of modules.zip
require"mymodules.widget"
```