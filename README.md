# Understanding

Things I broke, debugged, or got curious about — written down before I forget how I actually understood them.
 
None of this starts from a textbook. It starts from something I'm actually using and not fully understanding yet — then I work backward, or just get curious and follow it.

Some of it:
 
- Auth started with JWT because that's what I had at work. Working backward led to the older schemes, then forward again to JWE. 
- Indexes started with "add an index, query gets faster" — working backward led to how data sits on disk, which I still don't fully have. 
- Timeseries started because I use Mongo's TSDB feature and got curious why it's structured differently. 
- Idempotency started from Kafka — producer-side vs consumer-side idempotency are not the same problem.
 
So this repo is not "implemented and mastered." It's closer to: here's the depth I've reached on this, going backward from what I use. Some docs are near-complete. Some stop at "I understand the shape of it, not the internals yet".
 
Two rules for anything that goes in here:
1. If I can't explain it without jargon, I don't understand it yet.
2. Say what I actually know, not what sounds complete.

## Format
 
Every doc here follows a similar map:
 
1. **Mental Model** — the one image/analogy that makes the rest click
2. **Historical Evolution** — why it exists, what it replaced
3. **What Broke** — the failure that forced the idea into existence
4. **Core Mechanism** — how it actually works, no hand-waving
5. **Tradeoffs** — what you give up to get it
6. **Production Usage** — where I've actually seen or used it (if I haven't, the doc says so instead of faking it)
7. **Common Failure Modes** — how it breaks, and how that looks in practice
8. **Practical Guidelines** — when to reach for it, when not to
9. **One-line Principle** — the takeaway if you remember nothing else

## Why this exists
  
So six months from now I don't have to relearn this from scratch.
