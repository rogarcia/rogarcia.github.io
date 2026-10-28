---
layout: post
title: "Old caches, new tricks: an HPC engineer learns LLM serving"
---

<style>
/* Block-accounting figure. Every rule is scoped to .msf so the blog theme is untouched. */
.msf{--m-panel:#f5f7f6;--m-ink:#1a1d1c;--m-ink-2:#545a57;--m-rule:#d5dad7;--m-ctx-bg:#e3edfa;--m-ctx-edge:#2a78d6;--m-ctx-ink:#1d4f91;--m-st-bg:#fde6db;--m-st-edge:#eb6834;--m-st-ink:#9a3a12;font-family:system-ui,-apple-system,"Segoe UI",Roboto,sans-serif;font-size:15px;line-height:1.45;color:var(--m-ink);margin:1.75rem 0}
@media (prefers-color-scheme:dark){:root:not([data-theme="light"]) .msf{--m-panel:#1d2120;--m-ink:#eceeed;--m-ink-2:#aab1ad;--m-rule:#3a403d;--m-ctx-bg:#1b2f4d;--m-ctx-edge:#3987e5;--m-ctx-ink:#a9c8f0;--m-st-bg:#3d2416;--m-st-edge:#d95926;--m-st-ink:#f5b896;color-scheme:dark}}
:root[data-theme="dark"] .msf{--m-panel:#1d2120;--m-ink:#eceeed;--m-ink-2:#aab1ad;--m-rule:#3a403d;--m-ctx-bg:#1b2f4d;--m-ctx-edge:#3987e5;--m-ctx-ink:#a9c8f0;--m-st-bg:#3d2416;--m-st-edge:#d95926;--m-st-ink:#f5b896;color-scheme:dark}
.msf *{box-sizing:border-box}
.msf figure{margin:0;padding:1.1rem 1.1rem 0.9rem;background:var(--m-panel);border:1px solid var(--m-rule);border-radius:6px}
.msf .m-h{margin:0 0 0.2rem;font-size:1rem;font-weight:700;line-height:1.3;text-wrap:balance;color:var(--m-ink)}
.msf .m-sub{margin:0 0 0.4rem;color:var(--m-ink-2);font-size:0.875rem}
.msf .m-part{margin:0.9rem 0 0.5rem;font-size:0.85rem;font-weight:600;color:var(--m-ink)}
.msf .m-rows{display:grid;grid-template-columns:minmax(5rem,8.5rem) minmax(0,1fr) 2.5rem;gap:0.45rem 0.75rem;align-items:center}
.msf .m-lab{font-size:0.85rem;line-height:1.2;color:var(--m-ink-2)}
.msf .m-cells{display:grid;grid-template-columns:repeat(3,minmax(0,5.5rem));gap:0.35rem}
.msf .m-b{height:1.9rem;display:flex;align-items:center;justify-content:center;font-size:0.78rem;border:1.5px solid;border-radius:4px;white-space:nowrap}
.msf .m-ctx{background:var(--m-ctx-bg);border-color:var(--m-ctx-edge);color:var(--m-ctx-ink)}
.msf .m-part-b{background:linear-gradient(to right,var(--m-ctx-bg) 80%,transparent 80%);border:1.5px dashed var(--m-ctx-edge);color:var(--m-ctx-ink)}
.msf .m-st{background:var(--m-st-bg);border-color:var(--m-st-edge);color:var(--m-st-ink)}
.msf .m-n{text-align:right;font-size:0.85rem;font-variant-numeric:tabular-nums;color:var(--m-ink-2)}
.msf .m-big{font-weight:700;color:var(--m-ink)}
.msf .m-tot{padding-top:0.35rem;border-top:1px solid var(--m-ink-2);font-weight:700;color:var(--m-ink)}
.msf figcaption{margin-top:0.85rem;padding-top:0.65rem;border-top:1px solid var(--m-rule);color:var(--m-ink-2);font-size:0.8rem}
@media (max-width:480px){.msf .m-rows{grid-template-columns:4.6rem minmax(0,1fr) 2rem;gap:0.4rem 0.5rem}.msf .m-b{font-size:0.7rem}}
</style>

When I started [manifold-serve](https://github.com/rogarcia/manifold-serve) as a hands-on side project to catch up on production LLM inference by building (serve open models on rented GPUs, benchmark them, build a cache-aware router, then autoscaling), I expected my HPC background to be handy. Newsflash: it helped less than I thought, and in different places than I thought.

It has been over two months of on-and-off work, and there's no headline result yet. But I'm having a blast with this project, and I wanted to push this post out as a progress update: what transferred, what didn't, how fast this field moves, and a few findings worth sharing.

## What transferred from HPC

**Memory as a budget.** The KV cache (the GPU memory vLLM reserves to keep each conversation's attention state, so it isn't recomputed) is a capacity-planning problem. In one A/B, a 1.7x larger KV pool came from the memory budget (GPU memory utilization 0.92 to 0.95, plus a vLLM version that used less memory outside the cache), not from the fp8 KV cache I first credited. That bigger pool, not the new kernels, is the likeliest explanation for the 43% throughput gain in that run.

**Benchmark discipline.** Change one variable, repeat runs, expect variance from machine to machine. I saw two rented pods with the same GPU model, but a different host and driver, differ by 5 to 13% in throughput. Since then, I only compare runs from the same pod.

## What didn't transfer, and what I got wrong

I had to get comfortable with a bunch of serving-specific innovations such as PagedAttention, continuous batching, prefix caching and speculative decoding, plus how they interact with CUDA graph capture. Hybrid models add one more: some layers keep a recurrent state instead of a KV cache (more on that below).

Matching the CUDA toolchain to the GPU architecture should have been familiar territory, and I still got it wrong. My first round ran on fallback kernels because my pod image's CUDA toolchain was too old to build kernels for the RTX 5090. A current image fixed that, and on a 96 GB card the vLLM recipe ran unmodified. Twice I credited a gain to the wrong cause: the KV pool above, and a time-to-first-token (TTFT) gain from MTP that really came from CUDA graph capture sizes and showed up only on short prompts. My first custom load generator also undercounted speculative-decoding throughput, because it counted streamed chunks, not tokens. There's a lesson here: suspect the setup and the instrument before the result.

## How fast this field moves

vLLM went from 0.26 to 0.31 while I worked on this. Some releases shipped wheels built for CUDA 13, which my pod images couldn't run, and the models I used (Qwen3.8-27B-NVFP4 and Qwen3.5-9B) were new enough that the tooling around them was still settling. The flip side is how active the open-source community is. The vLLM recipes site gave me known-good serve commands, vLLM's multi-turn benchmark became my load generator for the router work, and my router follows the filter, score and pick design from llm-d.

## Two findings worth sharing

### Speculative decoding, measured properly

Speculative decoding lets a cheap drafter guess several tokens ahead, and the model checks all of them in one pass. MTP (multi-token prediction) is a drafter built into the model. My first attempt, squeezed onto a 32 GB card, showed 28 tok/s (undercounted, as it turned out) and looked like a failure. On one RTX PRO 6000 (96 GB) with the recipe unmodified, it looked very different:

- MTP drafting 5 tokens (k=5) gave 2 to 3x output throughput.
- The gain shrinks as the batch fills: on code prompts, 3.10x at concurrency 1 and 2.16x at concurrency 8.
- Acceptance falls with draft depth. On chat prompts, the first drafted token was kept 83% of the time and the fifth only 28%.
- Acceptance also falls with thinking on. On GSM8K (math), the tokens produced per verification step went from 4.83 to 3.91.

The scaffold was wrong, not the feature.

### Hybrid models change how the prefix cache behaves

This is where the HPC intuition paid off.

Each chat turn resends the whole conversation, and vLLM's prefix cache lets a replica skip recomputing a prefix it already holds. With two replicas there are two caches, so the router decides whether a turn lands where its history is cached.

Qwen3.5-9B, the model for this round, is a hybrid. Three of every four layers are [Gated DeltaNet](https://inferencex.semianalysis.com/glossary/gated-deltanet) layers, which keep a fixed-size recurrent state per conversation instead of a per-token KV cache. vLLM stores that state in the same pool as the KV cache. It splits the layers into four groups (one attention group and three DeltaNet groups) and sizes one block to hold either 528 tokens of attention cache or one state page. So a conversation with 2.8 blocks of context actually holds about 5.8 blocks:

<div class="msf">
<figure aria-labelledby="m-blk-h">
<p class="m-h" id="m-blk-h">One conversation's blocks, counted two ways</p>
<p class="m-sub">One block holds 528 tokens of attention cache or one DeltaNet state page.</p>
<p class="m-part">How I sized the pool (as if every layer were full attention)</p>
<div class="m-rows">
<div class="m-lab">context</div><div class="m-cells"><span class="m-b m-ctx">528 tok</span><span class="m-b m-ctx">528 tok</span><span class="m-b m-part-b">0.8</span></div><div class="m-n m-big">2.8</div>
</div>
<p class="m-part">What vLLM allocates on Qwen3.5-9B</p>
<div class="m-rows">
<div class="m-lab">attention group</div><div class="m-cells"><span class="m-b m-ctx">528 tok</span><span class="m-b m-ctx">528 tok</span><span class="m-b m-part-b">0.8</span></div><div class="m-n">2.8</div>
<div class="m-lab">DeltaNet group 1</div><div class="m-cells"><span class="m-b m-st">state</span></div><div class="m-n">1</div>
<div class="m-lab">DeltaNet group 2</div><div class="m-cells"><span class="m-b m-st">state</span></div><div class="m-n">1</div>
<div class="m-lab">DeltaNet group 3</div><div class="m-cells"><span class="m-b m-st">state</span></div><div class="m-n">1</div>
<div></div><div></div><div class="m-n m-tot">5.8</div>
</div>
<figcaption>Every live conversation holds one state block in each DeltaNet group, however short its context is. 2.8 blocks is the average context of a live conversation in the run (1,485 tokens).</figcaption>
</figure>
</div>

A conversation costs about twice what its token count predicts. I sized my "cache pressure" test in tokens, as if every layer were full attention, so the pool held 38% of the working set instead of 70%. The hit rate was 3 to 4% for every routing policy. That's a cache cliff that looks like classic LRU thrashing: when every conversation comes back after about the same delay, a cache that's too small evicts exactly what's needed next. Hits don't fall in proportion to the pool size. They fall off a cliff. (That explanation comes from how the benchmark is built. I haven't measured reuse distance directly.)

After sizing the pool in blocks (1,562 per replica, about 70% of the working set), routing made a difference. I compared three policies at 95% of the throughput knee:

- `least_load` sends each turn to the less busy replica.
- `session_hash` pins each conversation to one replica.
- `prefix_aware` prefers the replica that holds more of the prompt's cached blocks, then the less busy one.

| policy         | hit rate | TTFT p50 | TTFT p90 |
| -------------- | -------- | -------- | -------- |
| `least_load`   | 24%      | 158 ms   | 282 ms   |
| `session_hash` | 38%      | 148 ms   | 264 ms   |
| `prefix_aware` | 39%      | 144 ms   | 242 ms   |

TTFT is time to first token, p50 is the median and p90 the 90th percentile. Each row averages two seeds, and the p90 ranges of `least_load` and `prefix_aware` don't overlap.

Keeping each conversation on one replica is what raised the hit rate: `session_hash` isn't cache-aware and still got there. Prefix-aware routing added a tail-latency gain on top, because it balances load on turns with nothing cached yet, and about half of the prompts are shorter than one block.

## What's next

Next I want a control run with a full-attention model (Qwen3-8B) to see how much of this is specific to hybrid models, and a run where the cache fits, to measure what's left of the routing gain. Then I'll tackle autoscaling.

The run notes, scripts and charts, mistakes included, are in the repo: [github.com/rogarcia/manifold-serve](https://github.com/rogarcia/manifold-serve). So far this has cost a little over $50 in rented GPU time. If you've served a hybrid model with prefix caching, I'd like to hear how you sized the cache.
