# Using GC to prevent exhaustion of other resources too

If I understand correctly, the GC is invoked regularly after a fixed amount of memory allocations.  Why not also do
the same for other fixed resources (such as file handles) too?  That is, say, every 100 file handle allocations,
trigger a GC cycle by calling `System.Mem.performGC`.

This is something I've been wonder for a number of years (mostly when I run into the pain associated with manually
tracking file handles) and every now and then I try and google it.  Usually I don't find anything.  This time
around, I found this Conal Elliot also raised the same question on Haskell cafe a number of years ago, which gives
me the feeling it is a reasonable question to be asked

- [Conal Elliott suggests using GC to track other
  resources](https://mail.haskell.org/pipermail/haskell-cafe/2004-October/007228.html)
- [Conal Elliott responds to issues of promptness and whether bracketing performs
  better](https://mail.haskell.org/pipermail/haskell-cafe/2004-October/007317.html)
- [Conal Elliott claifies that collection occurs when the resource and not memory is
  low](https://mail.haskell.org/pipermail/haskell-cafe/2004-October/007233.html)

The potential issues that were floated to Conal focused on either the handle still being used, closing it having
side effects, or the fact the GC happens on memory getting low and not the resource.

- With regard to still being used, Conal seems to be suggesting per-type collectors (see third response back
  above), but I would think, so long as each resource handle only occurs in a single piece of memory (handles are
  an opaque type that reference the actual handler with an associated finalizer) then the regular GC will only reap
  them when they are no longer in use anywhere in the program.

- Other side effects (e.g., releasing locks on the file) seem like pretty special cases that should be treated as
  such and not be handicap us in the general case.  If we argue otherwise, we should also concede that regular GC
  is no go because it also has side effects that we sometimes care about too (e.g., it introduces unpredictable
  delays that can be bad for real-time programs).

- The final objection that the resources may can be consumed before memory is low enough for GC is a failure to
  understand that the suggestion is that GC would also be triggered upon a fixed number of allocations of other
  resources, such as file handles, in addition to memory, so they could not get low without triggering a GC.

Given the fact that there are library calls to trigger GC, it would seem this might be something that could be
implemented entirely in the libraries?  Code that returns a handle could increment shared (or per-thread for
performance?) counters, calling the GC when they pass a given number (allocating a handle means you are fixed for a
kernel trip anyway, so this shouldn't be too bad), not expose the actual OS handle in a way that it could be
duplicated, and associate it with a closing finalizer.
