# Struct Module #

Larc uses the struct module for reading and writing packed data. The module
was originally created by [Roberto Ierusalimschy](http://www.inf.puc-rio.br/~roberto/struct/).

## Packed Data ##

Numbers and strings can be packed into a binary format. The conversion is
done based on a format string. The format is a series of letters, where each
letter may be followed by a decimal number, and may include spaces but not
invalid letters. The number is optional except for setting the alignment.
There is no number when setting the endian mode.

#### Format Codes ####
| **Code** | **Meaning**  |
|:---------|:-------------|
| `>`      | Sets the conversion to big-endian mode. Numbers after this will be packed with the most significant bytes first.   |
| `<`      | Sets the conversion to little-endian mode. Numbers after this will be packed with the least significant bytes first.  |
| `!`      | Changes the maximum alignment, which must be a power of 2. Pad bytes are inserted before numbers so they will begin at a multiple of either the maximum alignement or the size of the number, whichever is lower.  |
| `x`      | Skip the given number of bytes. NULL bytes are inserted when packing. If the number is 0, adds enough bytes so the next conversion will be aligned (which may be 0). Padding bytes themselves are never aligned.  |
| `b`/`B`  | A signed (lower-case) or unsigned (upper-case) 8-bit number.  |
| `h`/`H`  | A signed (lower-case) or unsigned (upper-case) 16-bit number.  |
| `l`/`L`  | A signed (lower-case) or unsigned (upper-case) 32-bit number.  |
| `q`/`Q`  | A signed (lower-case) or unsigned (upper-case) 64-bit number. A Lua number does not have enough precision to represent all 64-bit numbers. When necessary, a large integer userdata will be returned.  |
| `i`/`I`  | A signed (lower-case) or unsigned (upper-case) in the given number of bytes. The default size is that of the native machine integer.  |
| `f`      | A single-precision floating point number. The exact size is machine-dependant but is usually 4 bytes.  |
| `d`      | A double-precision floating point number. The exact size is machine-dependant but is usually 8 bytes.  |
| `c`      | A string with a given number of characters. Unpacked NULL bytes in the string will be preserved.  |
| `s`      | A string with no more than the given number of characters. Unpacked NULL bytes are removed.  |
| `u`      | UTF-16 encoded string. The Lua string will be UTF-8 encoded.  |
| `U`      | Unicode string with 32-bit characters. The Lua string will be UTF-8 encoded.  |

#### Repetition and Width ####
For the plain numbers, `f`, `d`, `b`, `h`, `l`, `q`, and their unsigned
equivalents, the number after the format code is a repeat count. Consecutive
numbers will be packed together. It is the same as typing the code multiple
times. The other formats use the number as a width specifier and only convert a
single number or string (except for `x` which converts nothing).

#### String Conversion ####
Both `s` and `c` will convert a single string, but do so differently.
Unpacking a `c` format returns all of the bytes in the string, including
trailing NULL bytes. The `s` format terminates the string when it encounters
a NULL byte. Both formats are the same when packing with a width greater
than zero. They will truncate the string if it is too long, and add trailing
NULL bytes if it is too short. When the packing width is zero, the entire
string is packed, and the `s` format will add a terminating NULL.

Unpacking a `c` format with zero width expects the code to be previously
unpacked value to be a number. The number will be popped from the stack
and used as the width of the string. So a Pascal-style string can be read
using `"Bc0"` and a single string with up to 255 characters will be
returned.

Unpacking a `s` format with zero width will read up to the next NULL byte
in the packed string. The string without terminating NULL is returned, and
conversion continues after the NULL byte. This is the default mode for `s`
conversions. The default width of a `c` conversion is 1.

Unicode formats act like `s`. The string being packed must be encoded as
UTF-8. Invalid encodings will be rejected by the struct module. Otherwise,
malformed UTF could be used as a target vector for insertion attacks.
(Such as the invalid UTF-8 code "0xC0 0x80" being decoded to a NULL character.)
Unicode strings will align to multiples of 2 for `u`, or multiples of 4
for `U`.

### struct.pack ###
| **Arguments** | format, ...  |
|:--------------|:-------------|
| **Returns**   | string       |

Call `struct.pack` with the format string and the values to be packed.
There must be enough values, and of the correct types, that the format
string requires. The result will be a string with the packed bytes.

### struct.unpack ###
| **Arguments** | format, string, number _(opt)_  |
|:--------------|:--------------------------------|
| **Returns**   | ..., number                     |
| **On error**  | nil, error message              |

Call `struct.unpack` with the format string and a packed string. The
optional third arguments is the position in the string to begin conversion
at. The original values passed to `struct.pack` will be recreated when
the same format string is used with the packed string. After the unpacked
values, a number is returned which is the position in the string where
conversion stopped. If the entire string was converted, this number will
be larger than the length of the packed string. Or it points to the first
character that was not converted. If the conversion cannot succeed, usually
because the packed string is too short, then a `nil` is returned and a
message describing the error.

## Large Integers ##

The standard Lua platform cannot handle 64-bit integers. The struct module
uses a userdata type represent integers that are too large for a Lua number.
A signed 64-bit integer is stored in an 8 byte block of memory using native
word order. Each userdata is assigned a metatable that is registered with
the name `"large integer"`. The metatable allows large integers to be used
with numbers in expressions. The result of an operation with a large integer
and a number s always a large integer.

### struct.largeinteger ###
| **Arguments** | large integer or number or string  |
|:--------------|:-----------------------------------|
| **Returns**   | large integer                      |

Creates a new large integer from the argument. Conversion from a number
or another large integer is trivial. A string will be interpreted as
a base-10 number. If the string begins with `"0x"` or `"0X"`, then a
base-16 conversion is attempted. Or if it begins with `"0"` then
base-8 is tried. The conversion fails if there are invalid characters
in the string.

### largeinteger:tonumber ###
| **Arguments** | large integer  |
|:--------------|:---------------|
| **Returns**   | number         |

Convert a large integer to a number. The result is undefined if the integer
is too large to be represented by a Lua number.

### largeinteger:tostring ###
| **Arguments** | large integer, base _(opt)_, width _(opt)_ |
|:--------------|:-------------------------------------------|
| **Returns**   | string                                     |

Convert a large integer to a string representation of the number. The default
conversion will print the number in base-16 with a `"0x"` prefix. Negative
values are printed with a `"-"` before the number.

A base between 2 and 62 inclusively can be given to print the number using
the alphabet `[0-9A-Za-z]`. The number is printed as an unsigned value. The
third argument will add the digit `"0"` to the left of the number so the
string is at least that many characters long, but may be longer.

The special bases 32, 64, and 85 will convert the number to encodings
defined in RFC4648, or the ASCII85 encoding. RFC4648 also defines
base16, but this is equivalent to the general encoding scheme for other
bases. The special bases ignore the padding width.

The number is always printed with the most significant bits first.

## Variable Length Integers ##

Packed formats can save space by storing integers in a variable-length
encoding.

#### VLI ####
A variable-length encoding used by the XZ file format. The integer is
divided into 7-bit units and the most significant units with all 0 bits
are discarded. Each unit is packed into an 8-bit byte with the least
significant unit first. All bytes except for the last byte have the
high bit set. Up to 9 bytes are used, so a maximum of 63 bits can be
encoded.

#### MBI ####
A multi-byte integer encoding used by the 7z file format. If the integer
uses less than 8 bits, only a single byte is needed. Otherwise, the
first byte in the encoding will have 1, 2, or up to 8 of the highest bits
set, followed by a 0 bit (except when all 8 bits are set). If there are
available bits remaining in the byte, then they are filled using the most
significant bits from the integer. This byte is packed first, then the
number of extra bytes is the number of high bits set in the first byte.
Each successive byte is filled using bits from the integer in least
significant byte order, and excluding any bits that were used in the first
byte. Up to nine bytes are used and all 64 bits can be encoded, but it
is more complicated to encode and decode than VLI.

### struct.packvli ###
| **Arguments** | large integer  |
|:--------------|:---------------|
| **Returns**   | string         |

Convert a number or large integer to a VLI packed string. Negative numbers
cannot be packed using VLI.

### struct.unpackvli ###
| **Arguments** | string or function  |
|:--------------|:--------------------|
| **Returns**   | number, position    |

Decode a VLI packed string. The argument can be a function that will be called
with the number of bytes needed. The function should not return more than the
requested number of bytes. If there are no bytes available, it should return
`nil`. If the decoded integer is too large for a Lua number, then a large
integer may be returned. The number of bytes read is also returned. If the string
is empty, or the callback function returns `nil`, then `nil` is returned.

### struct.packmbi ###
| **Arguments** | large integer  |
|:--------------|:---------------|
| **Returns**   | string         |

Convert a number or large integer to a multi-byte integer.

### struct.unpackmbi ###
| **Arguments** | string or function  |
|:--------------|:--------------------|
| **Returns**   | number, position    |

Decode a multi-byte integer. The argument can be a function that will be called
with the number of bytes needed. The function should not return more than the
requested number of bytes. If there are no bytes available, it should return
`nil`. If the decoded integer is too large for a Lua number, then a large
integer may be returned. The number of bytes read is also returned. If the string
is empty, or the callback function returns `nil`, then `nil` is returned.