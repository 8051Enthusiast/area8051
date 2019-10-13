# at51

[![Build Status](https://travis-ci.org/8051Enthusiast/at51.svg?branch=master)](https://travis-ci.org/8051Enthusiast/at51)
[![Crates.io](https://img.shields.io/crates/v/at51)](https://crates.io/crates/at51)

A bunch of applications for the purpose of reverse engineering 8051 firmware.
Currently, there are four applications:
* `stat`, which gives blockwise statistical information about how similar a given file's opcode distribution is to normal 8051 code
* `base`, which determines the load address of a 8051 firmware file
* `libfind`, which reads library files and scans the firmware file for routines from those files (right now, only OMF51 files supported, which are used by PL/M-51 and more importantly C51)
* `kinit`, which reads a specific init data structure generated by the C51 compiler

The output of each subcommand can also be used in other programs via JSON.

### stat
This application does a square-chi (or a Kullback-Leibler) on the distribution of the would-be opcodes of a firmware image.
The lower the value, the more likely it is that a block is 8051 code.
For the blocksize, normally 512 or 256 is good (the opcodes are grouped into 16 buckets of similar frequency so that this does not create too much noise).

### base
This application tries to determine the load address of a firmware image (which in the best case is already reduced to the actual firmware that will be on the device).
It works by doing a circular cross-correlation between the target addresses of the `ljmp`/`lcall` instructions and the locations after `ret` instructions.
This is equivalent to counting for each possible load address how many jumps land behind `ret` instructions but can be efficiently done with some FFTs.

Normally, `acall`/`ajmp` are ignored since this introduces much noise by non-code data (1/16th of the 8051's instruction set is `acall`/`ajmp`) and can be enabled with a flag, but make sure that noise parts of the files (as detectable with entrpoy and the `stat` application) are zeroed-out.

Note that the file is assumed to wrap around in 16-bit space, which means that only the first 64k of the given file is loaded.
If you get an offset near the end of the space like `0xff80`, this probably means that there is a header of size `0x80` and the rest of the file is loaded at 0.

### libfind
This application loads some libraries given by the user and tries to find the functions inside the firmware.
Right now, OMF-51 libraries are supported, which is the format used by the C51 suite and seems to be how most 8051 firmware is actually created.

The relevant libraries are of the form C51\*.LIB (not C[XHD]51\*.LIB) and can currently be found on the internet just by searching for then, but you can of course also try to download the trial version of C51 to get the libraries from there.

Experimental support for sdld libraries (used by sdcc) is also included.
Note that there is more noise because the external symbol definition inside the object file doesn't define whether a symbol refers to code.

It is recommended to align the file to its load address before using this.

Example (on some random wifi firmware):

With `at51 libfind some_random_firmware /path/to/lib/dir/`:
```
Address | Name                 | Description
0x4220    ?C?CLDOPTR             char (8-bit) load from general pointer with offset
0x424d    ?C?CSTPTR              char (8-bit) store to general pointer
0x425f    ?C?CSTOPTR             char (8-bit) store to general pointer with offset
0x4281    ?C?IILDX              
0x4297    ?C?ILDPTR              int (16-bit) load from general pointer
0x42c2    ?C?ILDOPTR             int (16-bit) load from general pointer with offset
0x42fa    ?C?ISTPTR              int (16-bit) store to general pointer
0x4319    ?C?ISTOPTR             int (16-bit) store to general pointer with offset
0x4346    ?C?LOR                 long (32-bit) bitwise or
0x4353    ?C?LLDXDATA            long (32-bit) load from xdata
0x435f    ?C?OFFXADD            
0x436b    ?C?PLDXDATA            general pointer load from xdata
0x4374    ?C?PLDIXDATA           general pointer post-increment load from xdata
0x438b    ?C?PSTXDATA            general pointer store to xdata
0x4394    ?C?CCASE              
0x43ba    ?C?ICASE              
0x46f5    \[?C_START\]            
0x50e1    (MAIN)                
```

For some symbol names, which are in a general form, there are descriptions available.
Symbols embedded in parens are found only through references of other functions, symbols in brackets have some missing dependencies (meaning that functions they point to don't exist).

### kinit
This application is very specific to C51 generated code in that it decodes a specific data structure used to initialize memory values on startup.
The structure is read by the `?C_START` procedure and the location of the structure can therefore usually be found by running libfind and look at the two bytes after the start of `?C_START` (since it starts with a `mov dptr, #structure_address`).

Example:

With `at51 kinit -o offset some_random_firmware`:
```
bit 29.6 = 0
idata[0x5a] = 0x00
xdata[0x681] = 0x00
xdata[0x67c] = 0x00
xdata[0x692] = 0x00
xdata[0x6aa] = 0x01
xdata[0x46f] = 0x00
bit 27.2 = 0
bit 27.0 = 0
bit 26.3 = 0
bit 26.1 = 0
xdata[0x47d] = 0x00
xdata[0x40c] = 0x00
bit 25.3 = 0
xdata[0x46d] = 0x00
idata[0x5c] = 0x00
xdata[0x403..0x40a] = [0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]
xdata[0x467] = 0x00
```

## INSTALL
With cargo one can install it with `cargo install at51`.

Alternatively, to install from source, do
```
git clone 'https://github.com/8051Enthusiast/at51.git'
cargo install --path at51
```

