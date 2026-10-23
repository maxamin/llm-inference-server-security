# LLM Inference Server Security

> Exposed model servers, multi-tenant KV-cache leakage and side channels in vLLM/Ollama/Triton-style serving stacks.

## Contents

<!-- toc:start -->
1. [The Problem: Model Servers as Shared Infrastructure](chapters/01-problem-and-threat-model.md)
2. [Serving Architecture: Batching, Paged Attention and Prefix Caching](chapters/02-serving-architecture.md)
3. [Exposed and Unauthenticated Endpoints](chapters/03-exposed-endpoints.md)
4. [Cross-Tenant Leakage via Shared Caches and Timing](chapters/04-cross-tenant-leakage.md)
5. [Resource Exhaustion and Model/Adapter Loading Risks](chapters/05-dos-and-model-loading.md)
6. [Hardening, Monitoring and a Security Programme](chapters/06-hardening-monitoring-programme.md)
<!-- toc:end -->

## How to read this

Read chapters in order, or jump to what you need. Every chapter ends with exercises and further reading. See [RESOURCES.md](RESOURCES.md) and [GLOSSARY.md](GLOSSARY.md).

## Build

```bash
mdbook serve   # optional: renders SUMMARY.md as a website
```

## License

Text: CC BY-SA 4.0. Code snippets: MIT. See [LICENSE](LICENSE).
