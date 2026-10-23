# Exposed and Unauthenticated Endpoints

The most common serious finding against inference servers is not a clever side channel. It is a server reachable from a network it should not be reachable from, with no authentication, exposing management functions next to inference. This chapter maps the default attack surface of the major engines and shows how to audit your own.

## Defaults matter more than features

| Engine | Default listen | Built-in auth | Notable non-inference routes |
|---|---|---|---|
| Ollama | `127.0.0.1:11434` on a native Linux install; containers usually publish it | none | model pull, push, create, copy, delete; list and show models |
| vLLM OpenAI-compatible server | configurable host/port (8000 common) | optional `--api-key` | metrics, health, tokenizer, pause/resume and other routes, depending on version |
| NVIDIA Triton | HTTP 8000, gRPC 8001, metrics 8002 | none by default; restricted-API options | model repository load/unload, shared-memory registration, statistics |
| SGLang | configurable | optional API key | cache flush, weight update and other admin routes, depending on version |
| llama.cpp server | `127.0.0.1:8080` by default | optional API key | slots, props, metrics when enabled |

Defaults change between releases, so treat this table as a checklist of questions, not a set of answers. Check each value against the documentation for the version you run.

Two facts from primary sources illustrate why this matters:

- **Ollama.** Wiz Research's write-up of CVE-2024-37032 ("Probllama", a path-traversal flaw fixed in 0.1.34, May 2024) notes that the default Linux install binds to localhost, while Docker deployments expose the API publicly. Wiz counted over 1,000 exposed instances in June 2024. With no authentication, anyone who reaches the port can list models, pull new ones (consuming disk and bandwidth), delete models and run inference.
- **vLLM.** The project's security documentation states that `--api-key` protects only the OpenAI-compatible routes (newer versions also cover some `/v2` and `/inference` prefixes), lists routes that stay unauthenticated even when a key is set, and warns that some control routes, such as `/pause`, cause denial of service. It explicitly says not to rely on `--api-key` alone.

## The management plane

Management routes are where exposure turns into compromise:

- **Model load/pull/create.** Triton's model control API (when started in explicit mode) and Ollama's pull and create endpoints let a caller make the server fetch and parse new artefacts. Parsing untrusted model files is code-execution territory (chapter 5).
- **Shared-memory registration.** Triton lets clients register system or CUDA shared-memory regions to avoid copying tensors. Wiz's 2025 research on CVE-2025-23319 describes a chain in Triton's Python backend that starts with an information leak and ends in remote code execution by an unauthenticated attacker; NVIDIA fixed it in release 25.07. When a feature lets clients name memory regions, validation bugs have high impact.
- **Weight updates and cache flushes.** Routes added for reinforcement-learning pipelines and operations let callers swap weights or clear caches. Exposed, they enable model tampering or cheap DoS.
- **Tokenizer and model-info routes.** Usually harmless, but they reveal the exact model, version and context length, which helps an attacker choose exploits and jailbreaks.

## Metrics and logs

Prometheus endpoints are often published on a separate port and forgotten. They reveal model names, queue depth, GPU memory, cache hit rates and sometimes LoRA adapter names (which may encode customer names). Cache hit-rate metrics are a side channel by themselves (chapter 4). Request logs can contain whole prompts when debug logging is on.

## Why "it's behind the VPN" fails

Berta's analysis of the big-data stack shows the typical failure: components assumed to be internal (ZooKeeper on 2181, Hadoop RPC, dashboards) are reached by anyone who lands inside the network. The Microsoft *Kubernetes Best Practices* e-book (2023) states plainly that Kubernetes environments "aren't completely safe for hostile multi-tenant usage". A pod in the same cluster, a compromised notebook server or an SSRF in a front-end app can all reach a server that binds to `0.0.0.0` inside the cluster.

## Audit method

Run this against systems you own or have written authorisation to test.

1. **Discover.** From each network vantage point (internet, corporate LAN, cluster pod network), scan for known ports: 11434, 8000–8002, 8080, 30000, plus whatever your manifests declare.
2. **Fingerprint.** Request version, model list and health routes. Record the exact version.
3. **Enumerate routes.** Compare the route list in the engine's source for that version with what the proxy forwards. Many engines publish an OpenAPI schema (`/openapi.json` on FastAPI-based servers).
4. **Test authentication per route,** not per server: unauthenticated request, wrong key, right key.
5. **Check internal sockets** (RPC, KV transfer, distributed executor) with `ss -ltnp` on each node.
6. **Map to fixes:** bind to loopback or a private interface, put an authenticating gateway in front, allow-list routes, and apply NetworkPolicies.

## Worked example

> **Execution status:** Step 1 was run on the author's workstation against Ollama 0.40.7 bound to loopback; the observation is quoted. Steps 2–4 were not executed in this book's CI; verify locally.

**Step 1: confirm unauthenticated management reads.** Against the local Ollama:

```bash
curl -s http://127.0.0.1:11434/api/tags | jq -c '.models[] | {name, size, digest}'
```

Observed: one model returned with name, size (about 4.7 GB) and digest, no credential required. The safe property here is the loopback bind, not authentication.

**Step 2: route allow-list in a gateway.** An nginx front end that exposes only inference and requires a bearer token:

```nginx
map $http_authorization $authok { default 0; "Bearer CHANGE-ME-LONG-RANDOM" 1; }
server {
  listen 443 ssl;
  location = /api/chat     { if ($authok = 0) { return 401; } proxy_pass http://127.0.0.1:11434; }
  location = /api/generate { if ($authok = 0) { return 401; } proxy_pass http://127.0.0.1:11434; }
  location / { return 404; }
}
```

In production use a real identity layer (OIDC, mTLS) rather than a static token in config.

**Step 3: verify from outside.** `curl -i https://host/api/tags` should return 404, `/api/generate` without a token 401.

**Step 4: vLLM route check.** Start vLLM with `--api-key`, fetch `/openapi.json`, and for every path send an unauthenticated request. Every 2xx on a non-health route is a finding to close at the gateway.

## Common pitfalls

- **Setting an API key and assuming the server is protected.** Keys may cover only some routes.
- **Publishing container ports with `-p 11434:11434`.** That binds to all host interfaces; use `-p 127.0.0.1:11434:11434`.
- **Forgetting the metrics port.** It leaks model names and cache statistics.
- **Allow-listing by prefix.** `/v1/` may include admin-like routes; allow-list exact paths and methods.
- **Scanning third-party servers "to see".** Internet-wide probing of other people's model servers without authorisation is unlawful in many jurisdictions. Report exposures you stumble on through the owner or a CERT.

## Exercises

1. For your engine version, produce a table of every HTTP route, whether it requires auth with your configuration, and whether your gateway forwards it.
2. Write a Kubernetes NetworkPolicy that allows only the gateway pods to reach the inference pods on the API port, and test it from a debug pod.
3. Add a CI check that fails if any manifest or compose file publishes an inference port on `0.0.0.0`.

## Further reading

- vLLM security documentation: <https://docs.vllm.ai/en/stable/usage/security.html>
- Wiz Research, "Probllama" (CVE-2024-37032, 2024): <https://www.wiz.io/blog/probllama-ollama-vulnerability-cve-2024-37032>
- Wiz Research, "Breaking NVIDIA Triton: CVE-2025-23319" (2025): <https://www.wiz.io/blog/nvidia-triton-cve-2025-23319-vuln-chain-to-ai-server>
- Sheila A. Berta, *The Unbelievable Insecurity of the Big Data Stack* (conference talk).
- Microsoft, *Kubernetes Best Practices* e-book (2023), multi-tenancy chapters.
- Ollama FAQ on `OLLAMA_HOST` and network exposure: <https://github.com/ollama/ollama/blob/main/docs/faq.md>
