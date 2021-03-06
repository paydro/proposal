# Proposal: Zip-based Go package archives

Author: Russ Cox

Last updated: February 2016

Discussion at https://golang.org/issue/14386.

## Abstract

Go package archives (the `*.a` files manipulated by `go tool pack`) use the old Unix ar archive format.

I propose to change both Go package archives and Go object files to use the more standard zip archive format.
In contrast to ar archives, zip archives admit efficient random access to individual files within the archive
and also allow decisions about compression on a per-file basis.
The result for Go will be cleaner access to the parts of a package archive
and the ability later to add compression of individual parts
as appropriate.

## Background

### Go object files and package archives

The Go toolchain stores compiled packages in archives
written in the Unix ar format used by traditional C toolchains.

Before continuing, two notes on terminology:

 - An archive (a `*.a` file), such as an ar or zip file, is a file that contains other files.
   To avoid confusion, this design document uses the term _archive_
   for the archive file itself and reserves the term _file_ exclusively
   for other kinds of files, including the files inside the archive.

 - An _object file_ (a `*.o` file) holds machine code corresponding to a source file;
    the linker merges multiple object files into a final executable.
    Examples of object files include the ELF, Mach-O, and PE
    object files used by Linux, OS X, and Windows systems, respectively.
    We refer to these as _system object files_.
    Go uses its own object file format, which we refer to as _Go object files_;
    that format is unchanged by this proposal.
    
In a traditional C toolchain, an archive contains a file
named `__.SYMDEF` and then one or more object files (`.o` files)
containing compiled code; each object file corresponds to a different C or assembly source file.
The `__.SYMDEF` file is a symbol index a mapping from symbol name (such as `printf`)
to the specific object file containing that symbol (such as `print.o`).
A traditional C linker reads the symbol index to learn which of the
object files it needs to read from the archive; it can completely ignore
the others.

Go has diverged over time from the C toolchain way of using ar archives.
A Go package archive contains
package metadata in a file named `__.PKGDEF`,
one or more Go object files,
and zero or more system object files.
The Go object files are generated by the compiler
(one for all the Go source code in the package)
and by the assembler
(one for each assembly source file).
The system object files are generated by the system C compiler
(one for each `*.c` file in the package directory, plus a few for
C source files generated by cgo),
or (less commonly) are direct copies of `*.syso` files in the package source directory.
Because the Go linker does dead code elimination at a symbol level rather than
at the object file level, a traditional C symbol index is not useful
and not included in the Go package archive.

Long before Go 1, the Go compiler read a single Go source file
and wrote a single Go object file, much like a C compiler.
Each object file contained a fragment of package metadata
contributed by that file.
After running the compiler separately on each Go source file
in a package, the build system (`make`) invoked the archiver
(`6ar`, even on non-amd64 systems) to create an archive
containing all the object files.
As part of creating the archive, the archiver copied and merged
the metadata fragments from the many Go object files into
the single `__.PKGDEF` file.
This had the effect of storing the package metdata in the archive twice,
although the different copies ended up being read by different tools.
The copy in `__.PKGDEF` was read by later compilations importing
the package, and the fragmented copy spread across the Go object files
was read by the linker (which needed to read the object files anyway)
and used to detect version skew (a common problem due to the
use of per-directory makefiles).

By the time of Go 1, the Go compiler read all the Go source files for a package
together and wrote a single Go object file.
As before, that object file contained (now complete) package metadata,
and the archiver (now `go tool pack`) extracted that metadata into the
`__.PKGDEF` file.
The package still contained two copies of the package metadata.
Equally embarassing, most package archives
(those for Go packages with no assembly or C)
contained only a single `*.o` file, making the archiving step
a mostly unnecessary, trivial copy of the data through the file system.

Go 1.3 added a new `-pack` option to the Go compiler, directing it to
write a Go package archive containing `__.PKGDEF` and a `_go_.o` _without_
package metadata.
The go command used this option to create the initial package archive.
If the package had no assembly or C sources, there was no need for
any more work on the archive.
If the package did have assembly or C sources, those additional objects
needed to be appended to the archive, which could be done without
copying or rewriting the existing data.
Adopting `-pack` eliminated the duplicate copy of the package metadata,
and it also removed from the linker the job of detecting version skew,
since the package metadata was no longer in the object files the linker read.

The package metadata itself contains multiple sections used by different programs:
a unique build ID, needed by the go command;
the package name, needed by the compiler during import, but also needed by the linker;
detailed information about exported API, needed by the compiler during import;
and directives related to cgo, needed by the linker.
The entire metadata format is textual, with sections separated by `$$` lines.

Today, the situation is not much different from that of Go 1.3.
There are two main problems.

First, the individual package metadata sections are difficult to
access independently, because of the use of ad-hoc framing
inside the standard ar-format framing.
The inner framing is necessary in the current system in part
because metadata is still sometimes (when not using `-pack`)
stored in Go object files,
and those object files have no outer framing.

The line-oriented nature of the inner framing is a hurdle
for converting to a more compact binary format for the export data.

In a cleaner design, the different metadata sections would be stored
in different files in the Go package archive, eliminating the inner framing.
Cleaner separation would allow different tools to access only the
data they needed without needing to process unrelated data.
The go command, the compiler, and the linker all read `__.PKGDEF`,
but only the compiler needs all of it.

Distributed build systems can also benefit from splitting a package
archive into two halves, one used by the compiler to satisfy imports
and one used by the linker to generate the final executable.
The build system can then ship just the compiler-relevant
data to machines running the compiler and just the linker-relevant
data to machines running the linker.
In particular, the compiler does not need the Go object files,
and the linker does not need the Go export data;
both savings can be large.

Second, there is no simple way to enable compression for certain
files in the Go package archive.
It could be worthwhile to compress the Go export data 
and Go object files, to save disk space as well as I/O time
(not just disk I/O but potentially also network I/O,
when using network file systems or distributed build systems).

### Archive file formats

The ar archive format is simplistic: it begins with a distinguishing 8-byte header (`!<arch>\n`)
and then contains a sequence of files.
Each file has its own 60-byte header giving the file name (up to 16 bytes),
modification time, user and group IDs, permission bits, and size.
That header is followed by size bytes of data.
If size is odd, the data is followed by a single padding byte
so that file headers are always 16-bit aligned within the archive.
There is no table of contents: to find the names of all files in the archive,
one must read the entire archive (perhaps seeking past file content).
There is no compression.
Additional file entries can simply be appended to the end of an existing archive.

The zip archive format is much more capable, but only a little more complex.
A zip archive consists of a sequence of files followed by a table of contents.
Each file is stored as a header giving metadata such as the file name and data encoding,
followed by encoded file data,
followed by a file trailer.
The two standard encodings are “store” (raw, uncompressed)
and “deflate” (compressed using the same algorithm as zlib and gzip).
The table of contents at the end of the zip archive is a contiguous list of file headers
including offsets to the actual file data, making it efficient to access a particular
file in the archive.
As mentioned above, the zip format supports but does not require compression.
Appending to a zip archive is simple, although not as trivial as appending to an ar archive.
The table of contents must be saved, then new file entries are written
starting where the table of contents used to be, and then a new, expanded
table of contents is written.
Importantly, the existing files are left in place during this process,
making it about as efficient as adding to an ar format archive.

## Proposal

To address the problems described above,
I propose to change Go package archives
to use the zip format instead of the current ar format,
at the same time separating the current `__.PKGDEF` metadata file
into multiple files according to what tools use process the data.

To avoid the need to preserve the current custom framing in
Go object files, I propose to stop writing Go object files at all,
except inside Go package archives.
The toolchain would still generate `*.o` files at the times it does today,
but the bytes inside those files would be identical to those inside
a Go package archive.

Although the bytes stored in the `*.a` and `*.o` files would be changing,
there would be no visible changes in the rest of the toolchain.
In particular, the file names would stay the same,
as would the commands used to manipulate and inspect archives.
The only differences would be in the encoding used within the file.

A Go package archive would be a zip-format archive containing the following files:

	_go_/header
	_go_/export
	_go_/cgo
	_go_/*.obj
	_go_/*.sysobj

The `_go_/header` file is required, must be first in the archive, must be uncompressed,
and is of bounded size.

The header content is a sequence of at most four textual metadata lines. For example:

	go object darwin amd64 devel +8b5a9bd Tue Feb 2 22:46:19 2016 -0500 X:none
	build id "4fe8e8c8bc1ea2d7c03bd08cf3025e302ff33742"
	main
	safe

The `go object` line must be first and identifies the operating system, architecture,
and toolchain version (including enabled experiments) of the package archive.
This line is today the first line of `__.PKGDEF`, and its uses remain the same:
the compiler and linker both refuse to use package archives with an unexpected
`go object` line.

The remaining lines are optional, but whichever ones are present must appear
in the order given here.

The `build id` line specifies the build ID, an opaque hash used by the build system
(typically the go command) as a version identifier, to help detect when a package
must be rebuilt.
This line is today the second line of `__.PKGDEF`, when present.

The `main` line is present if the package archive is `package main`, making it a valid
top-level input for the linker. The command `go tool link x.a` will refuse to build
a binary from `x.a` if that package's header does not have a `main` line.

The `safe` line is present if the code was compiled with `-u`, indicating that it has
been checked for safety. When the linker is invoked with `-u`, it refuses to use
any unsafe package archives during the link.
This mode is experimental and carried forward from earlier versions of Go.

The `main` and `safe` lines are today derived from the first line of the export data,
which echoes the package statement from the Go source code, followed by the
word `safe` for safe packages.
The new header omits the name of non-main packages entirely in order
to ensure that the header size is bounded no matter how long a package name
appears in the package's source code.

More header lines may be added to the end of this list in the future,
always being careful to keep the overall header size bounded.

The `_go_/export` file is usually required (details below), must be second in the archive,
and holds a description of the package's exported API
for use by a later compilation importing the package. 
The format of the export data is not specified here,
but as mentioned above part of the motivation for this design
is to make it possible to use a binary export data format
and to apply compression to it.
The export data corresponds to the top of the `__.PKGDEF` file,
excluding the initial `go object` and `build id` lines and stopping at the first `$$` line.

The `_go_/cgo` file is optional and holds cgo-related directives for the linker.
The format of these directives is not specified here.
This data corresponds to the end of the Go object file metadata,
specifically the lines between the third `$$` line and the terminating `!` line.

Each of the `_go_/*.obj` files is a traditional Go object file,
holding machine code, data, and relocations processed by the linker.

Each of the `_go_/*.sysobj` files is a system object file,
either generated during the build by the system C compiler
or copied verbatim from a `*.syso` file in the package source directory
(see the [go command documentation](https://golang.org/cmd/go/#hdr-File_types)
for more about `*.syso` files).

It is valid today and remains valid in this proposal for
multiple files within an archive to have the same name.
This simplifies the generation and combination of package files.

## Rationale

### Zip format

As discussed in the background section, the most fundamental problem
with the current archive format as used by Go is that all package metadata is
combined into the single `__.PKGDEF` file.
This is done for many reasons, all addressed by the use of zip files.

One reason for the single `__.PKGDEF` file is that there is
no efficient random access to files inside ar archives. 
The first file in the archive is the only one that can be accessed
with a fixed number of disk I/O operations, and so it is often
given a distinguished role.
The zip format has a contiguous table of contents,
making it possible to access any file in the archive in a
fixed number of disk I/O operations.
This reduces the pressure to keep all important data in the first file.

It is still possible, however, to read a zip file from the beginning of the file,
without first consulting the table of contents.
The requirements that `_go_/header` be first, be uncompressed,
and be bounded in size exist precisely to make it possible to read
the package archive header by reading nothing but a prefix of the file
(say, the first kilobyte).
The requirement that `_go_/export` be second also makes it possible
for a compiler to read the header and export data without using
any disk I/O to read the table of contents.

As mentioned above, another reason for the single `__.PKGDEF` file
is that the metadata is stored not just in Go package archives but also
in Go object files, as written by `go tool compile` (without `-pack`) or `go tool asm`,
and those object files have no archive framing available.
Changing `*.o` files to reuse the Go package archive format
eliminates the need for a separate framing solution for metadata in `*.o` files.

Zip also makes it possible to make different compression decisions for
different files within an archive. This is important primarily because
we would like the option of compressing the export data and Go object files
but likely cannot compress the system object files, because reading them
requires having random access to the data.
It is also useful to be able to arrange that the header can be read
without the overhead of decompression.

We could take the current archive format and add a table of contents
and support for per-file compression methods, but why reinvent the wheel?
Zip is a standard format, and the Go standard library already supports it well.

The only other standard archive format in the Go standard library is the Unix tar format.
Tar is a marvel: it adds significant complexity to the ar format
without addressing any of the architectural problems that make ar unsuitable for our purposes.

In some circles, zip has a bad reputation.
I speculate that this is due to zip's historically strong association with MS-DOS
and historically weak support on Unix systems.
Reputation aside, the zip format is clearly documented,
is well designed, and avoids the architectural problems of ar and tar.

It is perhaps worth noting that Java .jar files also use the zip format internally,
and that seems to be working well for them.

### File names

The names of files within the package archives all begin with `_go_/`.
This is done so that Go packages are easier to distinguish from other zip files
and also so that an accidental `unzip x.a` is easier to clean up.

Distinguishing Go object files from system object files by name is new in this proposal.
Today, tools assume that `__.PKGDEF` is the only non-object file in the
package archive, and each file must be inspected to find out what kind
of object file it is (Go object or system object).
The suffixes make it possible to know both that a particular file
is an object file and what kind it is, without reading the file data.
The suffixes also isolate tools from each other, making it easier
to extend the archive with new data in new files.
For example, if some other part of the toolchain needs to add a
new file to the archive, the linker will automatically ignore it
(assuming the file name does not end in `.obj` or `.sysobj`).

### Compression

Go export data and Go object files can both be quite large.

I ran experiment on a large program at Google, built with Go 1.5.
I gathered all the package archives linked into that program
corresponding to Go source code generated from protocol buffer definitions
(which can be quite large), and I ran the standard `gzip -1` 
(fastest, least compression) on those files.
That resulted in a 7.5x space savings for the packages.

Clearly there are significant space improvements available
with only modest attempts at compression.

I ran another experiment on the main repo toward the end of the Go 1.6 cycle.
I changed the existing package archive format to force 
compression of `__.PKGDEF` and all Go object files,
using Go's compress/gzip at compression level 1,
when writing them to the package archive,
and I changed all readers to know to decompress them when 
reading them back out of the archive.
This resulted in a 4X space savings for packages on disk:
the $GOROOT/pkg tree after make.bash shrunk from about 64 MB to about 16 MB.
The cost was an approximately 10% slowdown in make.bash time:
the roughly two minutes make.bash normally took on my laptop
was extended by about 10 seconds.

My experiment was not as efficient in its use of compression as it could be.
For example, the linker went to the trouble to open and decompress
the beginning of `__.PKGDEF` just to read the few bits it actually needed.

Independently, Klaus Post has been working on improving the
speed of Go's compress/flate package (used by archive/zip,
compress/gzip, and compress/zlib) at all compression levels,
as well as the efficiency of the decompressor.
He has also replaced compression level 1 by a port of
the logic from Google's Snappy (formerly Zippy) algorithm,
which was designed specifically for compression speed.
Unlike Snappy, though, his port produces DEFLATE-compatible
output, so it can be used by a compressor without requiring
a non-standard decompressor on the other side.

From the combination of a more careful separation of data within
the package archive and Klaus's work on compression speed,
I expect the slowdown in make.bash due to
compression can be reduced to under 5% (for a 4X space savings!).

Of course, if the cost of compression is determined to be not paid for
by the space savings it brings, it is possible to use zip with no 
compression at all.
The other benefits of the zip format still make this a worthwhile cleanup.

## Compatibility

The toolchain is not subject to the [compatibility guidelines](https://golang.org/doc/go1compat).

Even so, this change is intended to be invisible to any use case that does not actually open a 
package archive or object files and read the raw bytes contained within.

## Implementation

The implementation proceeds in these steps:

1. Implementation of a new package `cmd/internal/pkg`
for manipulating zip-format package archives.

2. Replacement of the ar-format archives with zip-format archives,
but still containing the old files (`__.PKGDEF` followed by any number
of object files of unspecified type).

3. Implementation of the new file structure within the archives:
the separate metadata files and the forced suffixes for Go object files
and system object files.

4. Addition of compression.

Steps 1, 2, and 3 should have no performance impact on build times.
We will measure the speed of make.bash to confirm this.

These steps depend on some extensions to the archive/zip package suggested by Roger Peppe.
He has implemented these and intends to send them early in the Go 1.7 cycle.

Step 4 will have a performance impact on build times.
It must be measured to make a proper engineering decision
about whether and how much to compress.

This step depends on the compress/flate performance improvements by Klaus Post described above.
He has implemented these and intends to send them early in the Go 1.7 cycle.

I will do this work early in the Go 1.7 cycle, immediately following Roger's and Klaus's work.
I have a rough but working prototype of steps 1, 2, and 3 already.
Enabling compression in the zip writer is a few lines of code beyond that.

Part of the motivation for doing this early in Go 1.7 is to make it possible
for Robert Griesemer to gather performance data for his new binary
export data format and enable that for Go 1.7 as well.
The binary export code is currently bottlenecked by the need to escape
and unescape the data to avoid generating a terminating `\n$$` sequence.
