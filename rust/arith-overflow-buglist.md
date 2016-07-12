List of Bugs uncovered in Rust via arithmetic overflow checking
===============================================================

This document is a list of bugs that were uncovered during the
implementation and deployment of arithmetic overflow checking.

This list is restricted solely to *legitimate* bugs. Cases
where the overflow was benign (e.g. the computed value is
unused), transient (e.g. the computed wrapped value is
guaranteed to be brought back into the original range, such as
in `unsigned - 1 + provably_positive`), or silly (random
non-functional code in the tests or documentation) are not
included in the list.

However, extremely rare or obscure corner cases are considered
legitimate bugs. (We begin with such a case.)

### `impl core::iter::RandomAccessIter for core::iter::Rev`

if one calls the `iter.idx(index)` with `index <= amt`,
then it calls the wrapped inner iterstor with a wrapped
around value. The contract for `idx` does say that it
does need to handle out-of-bounds inputs, so this
appeared benign at first, but there is the corner case
of an iterator that actually covers the whole range
of indices, which would then return `Some(_)` here when
(pnkfelix thinks) `None` should be expected.

reference:
https://github.com/rust-lang/rust/pull/22532#issuecomment-75168901

### `std::sys::windows::time::SteadyTime`

`fn ns` was converting a tick count `t` to nanoseconds
via the computation `t * 1_000_000_000 / frequency()`;
but the multiplication there can overflow, thus losing
the high-order bits.

Full disclosure: This bug was known prior to landing
arithmetic overflow checks, and filed as:

https://github.com/rust-lang/rust/issues/17845

Despite being filed, it was left unfixed for months,
despite the fact that the overflow would start
occurring after 2 hours of machine uptime, according to:

https://github.com/rust-lang/rust/pull/22788

pnkfelix included it on this list because having arithmetic
overflow forces such bugs to be fixed in some manner
rather than ignored.

### `std::rt::lang_start`

The runtime startup uses a fairly loose computation to
determine the stack extent to pass to
record_os_managed_stack_bounds (which sets up guard
pages and fault handlers to deal with call stack over-
or underflows).

In this case, the arithmetic involved was actually
*overflowing*, in this calculation:

```
let top_plus_20k = my_stack_top + 20000;
```

pnkfelix assumes that in practice this would lead to us
attempting to install a guard page starting from some
random location, rather than the actual desired
address range. While the lack of a guard page in the
right spot is probably of no consequence here (assuming
that the OS is already going to stop us from actually
attempting to write to stack locations resulting from
overflow if that ever occurs), attempting to install a
guard page on a random unrelated address range seems
completely bogus.
pnkfelix only observed this bug when building a 32-bit
Rust on a 64-bit Linux host via cross-compilation.

So, probably qualifies a rare bug.
reference:

https://github.com/rust-lang/rust/pull/22532#issuecomment-76927295

UPDATE: In hindsight, one might argue this should be
reclassified as a transient overflow, because the whole 
computation in context is:

```
let my_stack_bottom =
    my_stack_top + 20000 - OS_DEFAULT_STACK_ESTIMATE;
```

where `OS_DEFAULT_STACK_ESTIMATE` is a large value
(> 1mb).

However, my claim is that this code is playing guessing
games; do we really know that the stack is sufficiently
large that the computation above does not *underflow*?

So pnkfelix is going to leave it on this list, at least
for now. (pnkfelix subsequently changed the code to use
saturated arithmetic in both cases, though obviously
that could be tweaked a bit.)

### struct order of evaluation

There is an explanatory story here:

https://github.com/rust-lang/rust/issues/23112

In short, one of our tests was quite weak and not
actually checking the computed values. But
arithmetic-overflow checking immediately pointed
out an attempt to reserve a ridiculous amount
of space within a `Vec`. (This was on an experimental
branch of the codebase where we would fill with
a series of `0xC1` bytes when a value was dropped, rather
than filling with `0x00` bytes.)

It is actually quite likely that this test would still
have failed without the arithmetic overflow checking,
but it probably would have been much harder to diagnose
since the panic would have happened at some arbitrary
point later in the control flow.

### Invalid `Span` constructions:

https://github.com/rust-lang/rust/issues/23480

Experts in the syntax system probably already knew
about this bug, but overflow checking forced everyone
to be reminded of it, repeatedly, in:

https://github.com/rust-lang/rust/issues/23115

See in particular:

https://github.com/rust-lang/rust/issues/23115#issuecomment-82960834

We have not actually *fixed* issue #23480 at the time of
this writing. Instead we put in a workaround:

https://github.com/rust-lang/rust/pull/23489

### Overflow in benchmark iteration increment

The implementation of `#[bench]` was not prepared to deal
with an overflow in the iteration count.

The arithmetic overflow checking forced that to fail on a
benchmark like

```rust
#[bench]
fn foo(b: &mut test::Bencher) {}
```

rather than return bogus results.

The iteration count is stored in a `u64`, so overflow sounds
unlikely -- even though `n` is being multiplied by 2 on every
increment, so it sounds plausible that we could get to very large
numbers indeed, the fact that we have to actually *run* that many
iterations is what gives me pause.

At this point I am assuming that the compiler is doing something
to optimize away the looping when the benchmark code is empty:
keep in mind that the `#[bench]` code above is not calling the
`black_box` method of `b`.  Maybe we should be requring that?

Anyway, the chosen fix was to just return the results so far before
hitting the overflow:

https://github.com/rust-lang/rust/pull/23127
