# Cross-Tenant Leakage via Shared Caches and Timing

Chapter 2 showed that prefix caching turns the KV cache into a global, content-addressed store whose hit or miss status changes time-to-first-token (TTFT). This chapter treats that as a side channel, surveys the published attacks, and shows how to measure and close it.

## The basic oracle

Suppose a server caches KV blocks for any block-aligned prefix it has seen. A co-tenant sends a guess `G` and measures TTFT. If another tenant recently sent a prompt starting with `G`, the guess hits the cache and prefill is skipped for those blocks, so TTFT is lower. The attacker now has a membership oracle: "has anyone recently sent a prompt starting with this text?"

With token-by-token extension, membership becomes extraction. Guess the first block; once it hits, try candidates for the next block, and so on. Real attacks must handle block granularity (a hit appears only when a whole block matches), noise from batching, and eviction. Published work shows these are engineering problems, not barriers:

| Work | Year / venue | Target | Finding (as reported by authors) |
|---|---|---|---|
| Wu et al., *I Know What You Asked: Prompt Leakage via KV-Cache Sharing in Multi-Tenant LLM Serving* (PromptPeek) | NDSS 2025 | vLLM, SGLang | reconstructs other users' prompts by probing shared KV cache |
| Timing-leakage study, arXiv:2409.20002 | 2024 preprint | SGLang, LangChain, GPTCache | system-prompt recovery token by token via cache timing; semantic caches also leak |
| CVE-2025-46570 | 2025 advisory | vLLM before 0.9.0 | chunk-based prefix caching gave a distinguishable timing difference; the advisory reports near-perfect separation (AUC about 0.99) at 8-token prefixes |

Semantic caches (which return a stored answer when a new prompt is similar enough to an old one) are worse: a hit returns another user's completion, not just a timing signal.

## Other channels in the same family

- **Token-length side channel.** Weiss et al., *What Was Your Prompt? A Remote Keylogging Attack on AI Assistants* (USENIX Security 2024), showed that streaming one token per encrypted packet reveals token lengths, from which a model can reconstruct much of a response. This is the LLM version of the packet-size leaks catalogued by Liu and Wang in *Preserving Privacy Against Side-Channel Leaks*, which also analyses padding as the countermeasure.
- **Queueing and batch interference.** With continuous batching, one tenant's long generation slows others. That reveals load and, coarsely, activity patterns.
- **Speculative decoding acceptance.** Draft-token acceptance depends on content; latency per token can reveal how predictable an output is.
- **Metrics.** A world-readable prefix-cache hit counter is the oracle without the timing noise.
- **GPU memory remanence.** If GPU scratch memory is not cleared between kernels, data can leak at hardware level. LeftoverLocals (Trail of Bits, 2024) is the example; see **gpu-security-ai-infrastructure**.

## Threat model for cache attacks

The attacker needs: (1) to share a cache with the victim (same server, same model, same salt or no salt); (2) to send requests and time them, or read metrics; (3) a candidate space. System prompts and templated RAG context are the most exposed, because they are long, shared and predictable. Free-text user questions are harder but not safe: PromptPeek-style attacks exploit predictable templates around user content.

## Defences

| Defence | How it works | Cost | Residual risk |
|---|---|---|---|
| Disable prefix caching | no shared blocks | throughput loss for shared system prompts | none from this channel |
| Per-tenant cache salt | mix a tenant secret into the first block's hash (vLLM `cache_salt`) | lose cross-tenant reuse only | tenants sharing a salt can still probe each other |
| Per-tenant server pools | separate engine instances | GPU cost | lowest |
| Cache only operator-defined prefixes | original PagedAttention model | engineering | prefixes themselves can be probed if secret |
| Timing padding / response bucketing | delay TTFT to a fixed bucket | latency | statistical leakage over many queries |
| Token batching in streams | send several tokens per packet, pad lengths | perceived latency | coarse lengths still leak |
| Restrict metrics | authenticate `/metrics`, drop per-tenant cache labels | none | — |

The vLLM documentation presents salting as preventing timing attacks between groups with different salts. That is the vendor's claim; verify it with your own measurement (worked example below) before you rely on it, and note that it does nothing within a trust group.

The decision rule is simple: **the cache sharing domain must not be larger than the trust domain.** If two tenants must not learn each other's prompts, they must not share cache blocks.

## Worked example

> **Execution status:** Step 1 was run on the author's CPU-only workstation (Ollama 0.40.7, qwen2.5-coder 7B Q4) and the numbers are quoted; it did *not* isolate a clean cache effect, which is itself the lesson. Steps 2–4 were not executed in this book's CI; verify locally on a GPU vLLM lab instance.

**Step 1: naive measurement on a noisy host.** Five requests with two canary prefixes (`ACME-CANARY-7731`, `ZETA-CANARY-0042`) of about 440 tokens, `num_predict=1`:

```text
B-cold           prompt_eval_ms 55379
A-cold           prompt_eval_ms  9869
A-shared-prefix  prompt_eval_ms  8847
B-after-A        prompt_eval_ms  8581
A-after-B        prompt_eval_ms  7675
```

The first request was slow because of warm-up; after that, run-to-run noise on a busy CPU host swamped any prefix effect. A single-shot timing comparison proves nothing either way. Attacks and audits both need many samples and statistics.

**Step 2: controlled experiment.** On an isolated vLLM instance with prefix caching enabled, have a "victim" script send a prompt beginning with a canary system prompt every few seconds. An "attacker" script sends 200 probes for the true prefix and 200 for a wrong prefix of the same length, recording TTFT for each.

**Step 3: quantify.** Compute the ROC AUC separating the two distributions:

```python
from sklearn.metrics import roc_auc_score
y = [1]*len(hit_ttft) + [0]*len(miss_ttft)
s = [-t for t in hit_ttft + miss_ttft]   # lower TTFT = more likely hit
print("AUC", roc_auc_score(y, s))
```

An AUC near 0.5 means no usable signal; near 1.0 means a reliable oracle.

**Step 4: apply a control and re-measure.** Give victim and attacker different `cache_salt` values (or disable prefix caching) and repeat steps 2–3. Record both AUCs in your security test suite so a version upgrade cannot silently reintroduce the channel.

## Common pitfalls

- **Concluding "no leak" from a handful of requests.** Noise hides signals that thousands of probes reveal.
- **Salting per request instead of per tenant.** You lose all caching benefit; salt per trust domain.
- **Forgetting semantic caches** in the application layer (LangChain, GPTCache and similar), which can return other users' answers.
- **Leaving cache metrics public** while fixing timing.
- **Assuming encryption hides streams.** Packet sizes and timing still leak token lengths.

## Exercises

1. Implement steps 2–4 on a lab GPU and report AUC with caching on, salted and off.
2. Audit your application for semantic caching. Is the cache key scoped by tenant and user?
3. Capture a streaming response in Wireshark from your own server and plot packet sizes. Then enable token batching or padding and compare.

## Further reading

- Wu et al., *I Know What You Asked: Prompt Leakage via KV-Cache Sharing in Multi-Tenant LLM Serving* (NDSS 2025): <https://www.ndss-symposium.org/wp-content/uploads/2025-1772-paper.pdf>
- Timing side channels in LLM serving systems, arXiv:2409.20002: <https://arxiv.org/abs/2409.20002>
- Weiss, Ayzenshteyn, Amit and Mirsky, *What Was Your Prompt? A Remote Keylogging Attack on AI Assistants* (USENIX Security 2024).
- Wen Ming Liu and Lingyu Wang, *Preserving Privacy Against Side-Channel Leaks* (Springer), chapters on traffic padding.
- vLLM prefix caching design (cache salting): <https://docs.vllm.ai/en/stable/design/prefix_caching.html>
- Trail of Bits, "LeftoverLocals" (2024): <https://blog.trailofbits.com/2024/01/16/leftoverlocals-listening-to-llm-responses-through-leaked-gpu-local-memory/>
