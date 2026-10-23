# Resources

## Serving engines and their security documentation

- [vLLM](https://github.com/vllm-project/vllm), [security guide](https://docs.vllm.ai/en/stable/usage/security.html), [prefix caching design](https://docs.vllm.ai/en/stable/design/prefix_caching.html)
- [SGLang](https://github.com/sgl-project/sglang) (RadixAttention prefix cache)
- [Ollama](https://github.com/ollama/ollama) and its [FAQ on network exposure](https://github.com/ollama/ollama/blob/main/docs/faq.md)
- [llama.cpp server](https://github.com/ggml-org/llama.cpp)
- [NVIDIA Triton Inference Server](https://github.com/triton-inference-server/server)
- [Hugging Face Text Generation Inference](https://github.com/huggingface/text-generation-inference)

## Papers (systems)

- Kwon et al., *Efficient Memory Management for Large Language Model Serving with PagedAttention*, SOSP 2023
- Yu et al., *Orca: A Distributed Serving System for Transformer-Based Generative Models*, OSDI 2022
- Wu et al., *Fast Distributed Inference Serving for Large Language Models* (FastServe), 2023
- Miao et al., *SpecInfer*, 2023
- Xiao et al., *Efficient Streaming Language Models with Attention Sinks* (StreamingLLM), 2023

## Papers and advisories (security)

- Wu et al., *I Know What You Asked: Prompt Leakage via KV-Cache Sharing in Multi-Tenant LLM Serving*, NDSS 2025
- Timing side channels in LLM serving systems, [arXiv:2409.20002](https://arxiv.org/abs/2409.20002)
- Weiss et al., *What Was Your Prompt? A Remote Keylogging Attack on AI Assistants*, USENIX Security 2024
- [Wiz: Probllama, CVE-2024-37032](https://www.wiz.io/blog/probllama-ollama-vulnerability-cve-2024-37032)
- [Wiz: Triton CVE-2025-23319 chain](https://www.wiz.io/blog/nvidia-triton-cve-2025-23319-vuln-chain-to-ai-server)
- [NVD: CVE-2025-32444](https://nvd.nist.gov/vuln/detail/CVE-2025-32444) (vLLM Mooncake pickle over ZeroMQ)
- [Trail of Bits: LeftoverLocals](https://blog.trailofbits.com/2024/01/16/leftoverlocals-listening-to-llm-responses-through-leaked-gpu-local-memory/)

## Books and talks

- Wen Ming Liu and Lingyu Wang, *Preserving Privacy Against Side-Channel Leaks: From Data Publishing to Web Applications* (Springer)
- Sheila A. Berta, *The Unbelievable Insecurity of the Big Data Stack* (talk)
- Microsoft, *Kubernetes Best Practices* e-book (2023)
- Julien Vehent, *Securing DevOps* (Manning, 2018)
- Mark Summerfield, *Programming in Python 3* (pickle security)

## Tools

- [modelscan](https://github.com/protectai/modelscan), [picklescan](https://github.com/mmaitre314/picklescan)
- [safetensors](https://github.com/huggingface/safetensors)
- [k6](https://k6.io) and [hey](https://github.com/rakyll/hey) for load tests
- [nmap](https://nmap.org) for authorised exposure checks

## Guidance

- [OWASP GenAI Security Project / Top 10 for LLM Applications](https://genai.owasp.org)
- [NVIDIA product security bulletins](https://www.nvidia.com/en-us/security/)
