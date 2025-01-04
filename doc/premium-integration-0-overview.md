Premium XS Integration, Overview
================================

There are several competing philosophies for wrapping external C libraries.  One is that
the XS module should hide all the details of the library and provide a clean
"Perlish interface".  The opposite extreme is that the external C functions should be
exposed to Perl using an extremely minimal XS layer, or the Foreign Function Interface
(FFI) and all the logic for working with the library should be written in Perl.

I advocate something in the middle.  I think that a good interface should expose as
much of the low-level as possible (to make the most usage of that library possible by
other Perl modules) while "padding the sharp edges" so that it is difficult for Perl-side
usage to crash the program.  Higher level features can be provided in addition to the low
level API via XS, Perl modules, or both.

If you consider that the average C library is an awkward mess of state machines and
lightly-enforced state requirements that will segfault if not carefully obeyed,
wrapping that nicely for the Perl developer is going to require a lot of data translation
and runtime sanity checks.
If you skip those runtime sanity checks in your wrapper library, it drags down the
efficiency of your subsequent Perl development to the level of C development, which is to
say, sitting around scratching your head for hours wondering why the program keeps
segfaulting. (or attaching gdb to your debug build of perl)
If you write those runtime checks in Perl, like with the FFI approach, your runtime
performance can suffer significantly.
If you write those runtime checks in XS, you can actually do quite a lot of them before
there's any notable decrease in the performance of the script.

Meanwhile, C code runs an order of magnitude faster than Perl opcodes, so if you're going
to require the end user to use a compiled module already, I feel it makes sense to put as
much of the higher-level routines into XS as you have time for.  But, the higher level
routines shouldn't be at the expense of the lower-level ones, or else you limit what
people can do with the library.

This guide will explain all the tricks I know to write safe, fast, convenient, and
powerful XS libraries.
