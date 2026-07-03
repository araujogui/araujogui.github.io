---
title: "A benchmark can be right and still lie to you"
date: 2026-07-02
description: "Why a real 1135% speedup was also a misleading one."
---

Earlier this year, after adding hex color support to Node.js's `styleText` API
in [#61556](https://github.com/nodejs/node/pull/61556), I added a small
optimization to the same function. I ran the benchmark, and one line of the
results said **1135% faster**: more than twelve times as fast. I remember
reading that row twice, because a number like that doesn't happen to you very
often.

A couple of weeks later I opened another pull request that undid part of the
same optimization.

Undoing part of an optimization that just posted a 1135% speedup sounds strange,
but the number was right, and it still managed to mislead me. It wasn't a bug or
a measurement fluke: the optimized code really did get 1135% faster. But it only
described one narrow situation, and I hadn't yet discovered how narrow. Once I
did, I could see which part of the optimization was worth keeping and which part
wasn't helping. Here's the whole thing, from the start.

## What `styleText` even is

If you've ever printed colored text to a terminal, you've used ANSI escape codes
whether you knew it or not. The terminal doesn't have a "make this red" API.
Instead you wrap your text in invisible little control sequences: bytes the
terminal reads, acts on, and never shows you.

Node exposes this through `util.styleText`:

```js
import { styleText } from "node:util";

console.log(styleText("red", "hello"));
```

Under the hood, that `'red'` turns into something like `\x1b[31mhello\x1b[39m`.
Think of it as `<span style="color:red">hello</span>`, except instead of HTML
tags the browser parses, it's raw bytes your terminal eats off the byte stream.
`\x1b[31m` means "switch the foreground to red," and `\x1b[39m` means "put the
foreground back to default." Everything between them comes out red.

A while back I added hex color support to it:

```js
console.log(styleText("#ff0000", "hello"));
```

Hex colors use a 24-bit truecolor sequence with the raw RGB channels:
`\x1b[38;2;255;0;0m`. To support hex, `styleText` does a small pipeline on every
call: parse `#ff0000` into `(255, 0, 0)`, then format and concatenate those
numbers into `38;2;r;g;b`.

That pipeline is where this whole story starts.

## The itch

Here's the thing that bugged me. Most CLIs don't use a thousand different
colors. They pick a small, fixed palette and reuse it constantly. Picture a
logger, painting a few thousand lines from the same handful of hex colors:

```js
for (const line of lines) {
  const color = line.level === "error" ? "#ff5555" : "#00b3ff";
  console.log(styleText(color, line.text, { validateStream: false }));
}
```

That `validateStream: false` is the "I know this stream supports colors, skip
the per-call check" flag you might use in a hot loop. It'll matter later. For
now, focus on the color: Node re-parses each one, re-derives its `(r, g, b)`,
and re-builds the exact same escape sequence it built for the last line at that
level. A tiny set of inputs, the same tiny set of outputs, computed from scratch
over and over.

That's the textbook shape of a thing you memoize. Pure function, a small pool of
repeated inputs, throw a cache in front of it. I'd explain it to a junior dev in
one sentence. It felt too obvious _not_ to do.

## Building the cache

So I built it (this is PR [#62999](https://github.com/nodejs/node/pull/62999),
if you want to read the real thing). Two pieces.

First, a `Map` keyed on the hex string: check it, return the cached sequence on
a hit, build and store it on a miss. Bounded at 256 entries, evicting the oldest
when it fills, because the color is whatever the caller passes in. If that's an
endless stream of distinct values, an unbounded cache is a memory leak you
haven't noticed yet.

Second, a fast path. `styleText` handles arrays of formats (`styleText(['red',
'bold'], ...)`) and different types of text (number, boolean, etc.). But the
common call is "one format, one string" so I added a short-circuit at the top
for exactly that case. The rest of the function, the part that already handled
arrays and other types, is the slow path. The cache backs both: the new fast
path and the existing slow path each check it before building a hex sequence
from scratch.

Then I made the benchmark harder to fool. The existing benchmark only tested one
fixed hex color, which the cache would trivially hit every time. That's not a
fair test. It only shows the happy path. So I added a rotating case that cycles
through 1000 distinct hex colors, more than the cache can hold, to hammer the
cache under a worst-case/high-miss workload and see if the bookkeeping cost
hurt.

## The number that stood out

I ran it. Here's the row that jumped out:

```csv
validateStream=0 format='#ff0000' messageType='string'   ***   1135.88 %   ±31.19% ±42.04% ±55.81%
```

More than twelve times as fast. Three stars, statistically significant.

The rest of the table was quiet. Really quiet. Here's roughly what the other
forty-odd rows looked like:

```csv
validateStream=0 format='#ff0000' messageType='boolean'   -0.71 %   ±1.04% ±1.38% ±1.80%
validateStream=0 format='#ff0000' messageType='number'    -0.59 %   ±0.93% ±1.24% ±1.62%
validateStream=0 format='red'     messageType='string'     4.92 %   ±6.11% ±8.17% ±10.72%
validateStream=1 format='italic'  messageType='string'     1.47 %   ±3.09% ±4.12% ±5.36%
...
```

Low single-digit deltas, some negative, all comfortably inside their own error
bars. The only clear signal was the giant one, and it showed up under one very
specific combination of inputs. That's the thread worth pulling.

## Why it was real in exactly one place

Look at two rows from that table. Same format, same message type. The only thing
that changes is `validateStream`:

```csv
validateStream=0 format='#ff0000' messageType='string'   ***   1135.88 %
validateStream=1 format='#ff0000' messageType='string'         -0.27 %
```

Same job from the caller's point of view: colorize a string with a hex color.
One is more than twelve times faster. The other didn't move at all. Neither
benchmark was wrong; they were measuring two very different code paths.

With `validateStream=0`, a single format (not an array), and a string message,
`styleText` takes the fast path. On this path, there is very little surrounding
work, so building the hex escape sequence is a large part of the total cost.
Replace that work with a cache lookup, and the relative win looks enormous. You
removed nearly all of a tiny thing, so the percentage goes to the moon. Twelve
times "barely any work" is still barely any work, but the ratio is real.

Change any of those conditions and the picture changes. `validateStream=1`, for
example (the default value for that option), takes the slow path. On this path,
building the escape sequence is just one small step among larger fixed costs, so
saving it disappears into the noise. It's coupon-clipping next to a mortgage
payment. Technically you saved money. Nobody can tell.

"Does caching help styleText with hex colors?" was too broad a question, mixing
two code paths that behave nothing alike. Asking on which code path the cache
pays for itself gets you an answer that's right there in the quiet rows.

## Deleting it

So I pulled the cache out of the slow path. This is PR
[#63706](https://github.com/nodejs/node/pull/63706). On the slow path, it now
just recomputes each sequence from scratch: no `Map`, no "is the cache full"
branch, no cache at all.

Then I benchmarked it, to confirm removing the cache hadn't cost anything. The
results were exactly as unremarkable as that reasoning predicted: a scatter of
1-3% deltas with wide error bars, statistically indistinguishable from having
changed nothing. Validation dominates the cost on this path, so the lookup
savings were negligible all along.

What survived: the fast-path cache stayed. That's the one place the measurement
justified it, so that's the one place it lives. A cache belongs where the thing
it caches is the thing that's expensive, and on the slow path it wasn't.

## What this is actually about

Both cache PRs were correct. The cache made the fast path more than twelve times
faster and shipped. Pulling it from the slow path was also right, because there
it saved nothing. The lesson isn't that the win was small. It's that one number
can't speak for a function with multiple paths that behave nothing alike.

None of this is really about caches, or about Node. It's the same trap as a
`useMemo` that only helps one render path, or a database index that only speeds
up one shape of query. The win is real, but it lives in a corner of the input
space, and a headline number won't tell you which corner. Real and narrow are
not in conflict.

So a few things I check now, whatever I'm optimizing:

- When a benchmark averages over branches, split it. One number for a function
  with multiple code paths is one number too few.
- Benchmark the path real callers take, not the fastest one I can construct.
- Be suspicious of a giant relative win on something that was already cheap. A
  huge percentage of almost nothing is still almost nothing.

## The trail

If you want to watch this happen in slow motion, the whole thing is public:

- [#61556](https://github.com/nodejs/node/pull/61556): adding hex color support
- [#62999](https://github.com/nodejs/node/pull/62999): the cache and the fast path
- [#63706](https://github.com/nodejs/node/pull/63706): taking it back out of the slow path

The two cache PRs point in opposite directions, and both are right. Honestly,
that's most of what performance work actually looks like: you add the thing, you
measure hard enough to find out where it really helps, and then you keep exactly
that much and let the rest go.
