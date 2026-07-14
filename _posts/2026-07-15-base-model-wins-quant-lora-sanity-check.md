# When Fine-Tuning Makes Things Worse: A Quick Quant/LoRA Sanity Check

I've been running a Qwen3.6-27B setup locally — base Q8, an Unsloth-dynamic UD-Q6_K_XL quant, a couple of my own LoRA fine-tunes, and (out of curiosity) Tess-4-27B, a community reasoning fine-tune of the same base model. Instead of trusting vibes, I ran all of them through the same coding task and compared the actual output.

## The Test

The task: build a single-file HTML tool implementing a line-sweep algorithm (find the maximum overlap among time intervals, and later, a weighted version that flags when concurrent resource usage exceeds a capacity threshold). Same prompt, same constraints, different models/quants.

I didn't just eyeball the UI — I hand-traced the actual algorithm logic against a handful of adversarial test cases:

- **Basic example** — a standard 4-interval overlap case
- **Simple tie-break** — intervals that touch exactly at a boundary (`[1,5)` and `[5,10)` should *not* overlap)
- **Two-digit tie-break** — same idea, but with coordinates in the double digits, which exposes bugs in comparison logic that only fail lexicographically (e.g. comparing arrays with `<`, which silently falls back to string comparison in JS)
- **Weighted sweep with capacity overflow** — multiple overlapping "over capacity" segments that need to be correctly separated, not merged
- **Negative coordinates** — a basic robustness check

## Results

| Model | Result |
|---|---|
| Base Q8 | Passed every test, every round |
| Base UD-Q6_K_XL | Passed every test, every round — indistinguishable from Q8 |
| My "thinkcap" LoRA | Wrong bottleneck logic on the very first test |
| My "qwopus" LoRA | Runtime crash (a destructuring bug: `const [x] = Math.min(...)`, where `Math.min` returns a number, not an array) |
| Tess-4-27B | Failed the two-digit tie-break test (array comparison bug) in round 1; passed round 2's core logic but shipped a UI where the displayed "usage" numbers came from a separate, buggy helper function that returned different values than the ones actually used for the real computation |

The base model — at both Q8 and Q6_K_XL — was the only one that came out clean across every round.

## The Real Finding: How Fine-Tuned Models Fail

It's not that fine-tuning is universally bad. It's *how* the fine-tuned/derivative models failed. Every single failure traced back to the same root cause: **the model tried to solve the problem more than once inside the same file.**

- Dead code from an abandoned first attempt left sitting next to the "real" logic
- A second helper function recomputing a value that was already computed correctly elsewhere, and returning a different (wrong) answer
- Multiple "let me try a cleaner approach" blocks stacked on top of each other, with only the last one actually wired up — except sometimes the wrong one got wired up

The base models, by contrast, computed each value once and reused it. Boring, but correct.

If I had to guess: reasoning-heavy fine-tuning that's trained to visibly deliberate can end up leaking that deliberation into the code itself — multiple attempts get left in the artifact instead of being collapsed into one clean solution. It's not that the underlying model can't reason about the problem; it's that the "show your work" habit doesn't always get cleaned up before the final answer ships.

## The Quant Takeaway

The UD-Q6_K_XL quant was essentially free: 2x throughput, larger context window, zero quality loss on this test suite. If you're running models locally and haven't tried a solid Q6 quant yet, there's no reason not to.

## Caveats

This is one algorithm family (line-sweep variants), a handful of adversarial test cases, and a small sample per model. It tells me these specific models have a specific failure mode on this specific kind of task — not that any of them are broadly bad. I'd want to run a different problem shape (state machines, tree/graph traversal, multi-file refactors) before drawing a bigger conclusion.

For now, though: the boring baseline won every round, the quantized version was free, and the "improved" versions all introduced bugs the base model didn't have. Worth remembering the next time a fine-tune or a fancy new release looks tempting — run the sanity check first.