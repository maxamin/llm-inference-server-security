# Glossary

- **Prefill:** the phase that processes all prompt tokens in parallel and builds the KV cache.
- **Decode:** the phase that generates one token per step, attending to cached keys and values.
- **KV cache:** stored attention keys and values for earlier tokens, reused during decoding.
- **PagedAttention:** vLLM's method of storing the KV cache in fixed-size blocks mapped through a block table, like virtual memory pages.
- **Block table:** per-sequence mapping from logical KV blocks to physical GPU memory blocks.
- **Prefix caching:** reusing KV blocks across requests whose prompts share a block-aligned prefix.
- **Cache salt:** a secret mixed into prefix-cache hashes so that only requests with the same salt share blocks.
- **Continuous (iteration-level) batching:** admitting and retiring requests at every decode step rather than per batch.
- **TTFT (time to first token):** latency until the first output token; drops on prefix-cache hits.
- **Semantic cache:** an application cache that returns a stored completion for a sufficiently similar prompt.
- **Token-length side channel:** inferring content from the sizes of encrypted streamed token packets.
- **Management plane:** server routes that change state (load, pull, delete, pause, update weights), as opposed to inference routes.
- **Disaggregated serving:** running prefill and decode on different workers and transferring KV caches between them.
- **LoRA adapter:** a small low-rank weight delta that specialises a base model; multi-LoRA servers select one per request.
- **safetensors:** a tensor file format with no executable code path, used instead of pickle-based checkpoints.
- **Model DoS:** exhausting serving capacity with expensive requests (OWASP LLM04).
