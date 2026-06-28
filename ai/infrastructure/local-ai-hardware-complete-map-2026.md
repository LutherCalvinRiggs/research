# Local AI Hardware — The Complete Map (2026)

**Source:** https://x.com/starmexxx/status/2063992615721435154 (pasted content by @starmexxx)
**Saved:** 2026-06-27
**Tags:** ai, infrastructure, tools, economics, technology

> **Note:** Mac Mini coverage overlaps with `ai/tools/mac-mini-local-ai-server-3-dollars-month.md`. This note covers the full hardware landscape including Jetson, RTX 3090, GMKtec EVO-X2, and the GPU rental model — which that note doesn't cover.

---

## TL;DR
Four forces converged in late 2025/early 2026 making local AI inference genuinely competitive with cloud subscriptions: models got smaller and smarter (Qwen 3.6 27B beats Claude on vision benchmarks), hardware caught up (Apple M4 UMA, AMD Strix Halo 128GB unified memory, NVIDIA Jetson at $249), subscriptions got more expensive ($200/month each for Claude Code Max and ChatGPT Pro), and open source won (Llama, Qwen, DeepSeek, Mistral — all free). The software stack is identical across all hardware: Ollama + Open WebUI + Claude Code pointed at localhost.

---

## The Subscription Bill Being Replaced

| Subscription | Monthly | Annual |
|---|---|---|
| Claude Code Max (20x) | $200 | $2,400 |
| ChatGPT Pro | $200 | $2,400 |
| Gemini Advanced | $20 | $240 |
| GitHub Copilot | $19 | $228 |
| Cursor Pro | $20 | $240 |
| **Heavy user total** | **$459** | **$5,508** |

Hardware path vs. subscription path over time:

| Year | Subscription | Hardware ($599–1,700 + ~$50/yr electricity) |
|------|-------------|------|
| 1 | $5,508 | $649–1,750 |
| 2 | $11,016 | +$100 |
| 3 | $16,524 | +$100 |
| 5 | $27,540 | +$200 |

By year 3, even the most expensive device has paid for itself 6–10x over.

---

## The Five Devices

### Level 1 — Jetson Orin Nano Super ($249)

| Spec | Value |
|------|-------|
| Price | $249 one-time |
| AI performance | 67 TOPS |
| RAM | 8GB unified |
| Max model | 7B (Llama 3.2, Mistral 7B, Qwen 2.5 7B) |
| Power | 7–25W |
| Electricity (24/7) | ~$2/month |
| Size | Smaller than a wallet |

Best for: users paying $20/month for ChatGPT Plus who want to stop. 7B models handle ~80% of daily use (drafting, summarizing, coding scripts, Q&A). Doesn't handle complex multi-step reasoning, large context windows, or frontier-level tasks.

---

### Level 2 — Mac Mini M4 ($599–1,399)

Covered in depth at `ai/tools/mac-mini-local-ai-server-3-dollars-month.md`. Summary:

- $599 (16GB): 8B models comfortably
- $799 (32GB): 14B models including Qwen 3.6 14B, DeepSeek R1 14B
- $1,399 M4 Pro (48GB): Llama 3.3 70B — closest local equivalent to GPT-4

Key advantage: Apple Silicon unified memory architecture. CPU and GPU share one memory pool — model loads once, both processors read from the same place. A $599 Mac Mini outruns $1,500 Windows AI machines for inference.

---

### Level 3 — Used RTX 3090 ($700)

The best value-per-dollar device on the map. Key insight: for local AI, VRAM matters more than chip generation.

| GPU | VRAM | Price | Best local model |
|-----|------|-------|-----------------|
| RTX 5090 (new) | 32GB | $3,800+ | 70B models |
| RTX 4090 (used) | 24GB | $2,000+ | 70B models |
| **RTX 3090 (used)** | **24GB** | **$650–800** | **27B models** |
| RTX 4070 (new) | 12GB | $599 | 14B models only |
| RTX 3060 (used) | 12GB | $200 | 14B models only |

A 2020 card with 24GB beats a 2024 card with 12GB for this specific workload. The RTX 3090 isn't just cheap — it's actively better than newer smaller-VRAM cards.

**The model that makes it worth it: Qwen 3.6 27B**

| Benchmark | Qwen 3.6 27B (local, free) | Claude 4.5 Opus ($200/month) |
|-----------|---------------------------|------------------------------|
| RealWorldQA (vision) | 84.1 | 77.0 |
| IFBench (instructions) | 76.5 | 58.0 |
| AIME 2026 (math) | 91.3 | 93.3 |
| MMLU (knowledge) | 83.2% | ~82% |

Free, locally-runnable, beats Anthropic's flagship on vision by 7 points and on instructions by 18.

**Buying used RTX 3090**: eBay sellers with 98%+ feedback. Request GPU-Z screenshots showing VRAM health. Avoid cards described as coming from mining rigs.

---

### Level 4 — GMKtec EVO-X2 ($1,700)

| Spec | Value |
|------|-------|
| Chip | AMD Ryzen AI Max+ 395 (Strix Halo) |
| Unified memory | Up to 128GB |
| Usable VRAM (Linux) | Up to 110GB |
| AI performance | 126 TOPS combined |
| Electricity (24/7) | ~$9/month |
| Notable | AMD CEO personally signed a unit at Shanghai AI Developer Day |

The first x86 chip that can run a 200B+ parameter model on a single piece of silicon. AMD claimed 3x+ performance advantage over RTX 5080 on DeepSeek R1 inference at CES 2026.

| Model | VRAM needed | Result |
|-------|-------------|--------|
| Qwen3-235B | ~110GB | Runs fully |
| DeepSeek-V3 | ~100GB | Runs comfortably |
| Llama 3.3 70B | ~42GB | Fast, headroom to spare |
| Qwen 3.6 27B | ~16GB | Very fast daily driver |

Break-even: ~9–10 months vs. full subscription stack. Three-year savings vs. subscriptions: ~$13,000.

---

### Level 5 — GPU Rental Farm (Earn Instead of Save)

The inverted model: instead of running AI locally to avoid paying for cloud, rent your compute to the cloud and earn from it.

| GPU | Mining ($/month) | AI rental ($/month) | Multiplier |
|-----|-----------------|---------------------|-----------|
| RTX 3090 | $40–90 | $200–400 | 4–5x |
| RTX 4090 | $80–150 | $500–1,000 | 5–7x |
| RTX 5090 | $120–200 | $700–1,400 | 5–7x |
| A100 80GB | n/a | $1,200–2,500 | — |
| H100 | n/a | $2,500–5,000 | — |

**Platforms:** Vast.ai, Clore.ai, io.net, RunPod, Akash, Salad. Cut: 15–25%. Paid in dollars or stablecoins.

**Scaling:**

| Scale | Net monthly (RTX 4090) |
|-------|------------------------|
| 1 card | $400–800 |
| 4 cards | $1,600–3,200 |
| 8 cards | $3,200–6,400 |
| 16 cards | $6,400–12,800 |

The mechanism: OpenAI and Anthropic buy compute from these farms cheaply and resell it at $200/month. GPU farm operators capture the spread.

---

## The Universal Software Stack

Identical setup on every device:

```bash
# Step 1 — Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Step 2 — Pull the largest model your RAM allows
ollama pull qwen3.6:27b      # for RTX 3090 / EVO-X2
ollama pull qwen3.6:14b      # for Mac Mini 32GB
ollama pull qwen3.6:8b       # for Mac Mini 16GB / Jetson

# Step 3 — Point Claude Code at local Ollama
ANTHROPIC_BASE_URL=http://localhost:11434/v1 claude
```

| Layer | Tool |
|-------|------|
| Runtime | Ollama (free, open source) |
| Browser UI | Open WebUI (private ChatGPT interface) |
| Coding agent | Claude Code → localhost |
| Recommended models | Qwen 3.6 27B, DeepSeek R1, Llama 3.3 70B, Mistral 7B |

---

## The Decision Tree

| Situation | Recommendation |
|-----------|---------------|
| Paying $20/month (ChatGPT Plus) | Jetson Orin Nano $249 |
| Paying $200+/month on AI APIs | Mac Mini M4 $599 |
| Heavy Claude Code user | Mac Mini Pro $1,399 or RTX 3090 $700 |
| Need 200B+ models | GMKtec EVO-X2 $1,700 |
| Already have PC with 4090 | Add RTX 3090 or use existing GPU |
| Want to earn instead of save | GPU rental farm |
| Best value per dollar | Used RTX 3090 + existing PC |
| Zero setup, just works | Mac Mini M4 |
| Privacy-critical work | Any device — all local |
| Best of both worlds | Any device + keep one $20/month subscription |

**The hybrid strategy** (what most users end up with): local hardware handles 80% of daily tasks for free, one $20/month subscription covers the remaining 20% of genuinely frontier-level reasoning. Total: ~$23/month vs. $459.

---

## The Four Forces That Made This Possible (2025–2026)

1. **Models got smaller and smarter** — Qwen 3.6 27B beats Claude 4.5 Opus on vision. DeepSeek R1 14B handles reasoning at 60+ tokens/second on consumer hardware. Llama 3.3 70B runs on a $1,400 Mac Mini Pro.

2. **Hardware caught up** — Apple M4 unified memory bandwidth beats discrete GPUs for inference. AMD Strix Halo brought 128GB unified memory to x86. NVIDIA dropped capable AI hardware to $249.

3. **Subscriptions got more expensive** — Claude Code Max at $200/month. ChatGPT Pro at $200/month. The "professional tier" became the new baseline for serious users.

4. **Open source won** — Llama (Meta), Qwen (Alibaba, Apache 2.0), DeepSeek (free), Mistral (free). Every model in this article downloads in 15 minutes at zero cost.

---

## Questions & Gaps
- Benchmark comparisons (Qwen 3.6 27B vs. Claude) are cited without a source or methodology. Independent reproduction would be valuable before relying on these numbers for capability planning.
- GPU rental income figures are estimates with wide ranges — actual earnings depend heavily on utilization rate, which varies by platform, GPU model, and market conditions.
- The EVO-X2 "110GB usable VRAM on Linux" figure — what's the actual headroom in practice? Memory management overhead may reduce this for very large models.
- Token generation speed isn't provided for most devices. Knowing tokens/second per device is important for interactive use cases — a model that runs but generates at 2 tokens/second isn't usable for real-time coding.

## Related Notes
- [Mac Mini Local AI Server ($3/month)](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/mac-mini-local-ai-server-3-dollars-month.md) — deep dive on Mac Mini M4 specifically: Ollama v0.14.0 Anthropic API compatibility, OpenClaw integration, Open WebUI Docker setup. Read alongside this note.
- [Unit Economics of Running Cattle](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/unit-economics-of-cattle.md) — Jed's cost governance framework for NEEDLE fleets. Local Ollama as the agent backend eliminates the API spend layer entirely for commodity beads, radically changing the economics described there.
- [NEEDLE Implementation Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Implementation-Guide.md) — `ANTHROPIC_BASE_URL=http://localhost:11434/v1` in worker environments redirects all NEEDLE workers to local Ollama. A Mac Mini or RTX 3090 as the inference backend for commodity beads is a direct application of this hardware map.
- [AI Company Analysis Forensic Screener](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/ai-company-analysis-forensic-screener.md) — the GPU rental farm model (Vast.ai, RunPod etc.) represents an interesting equity analysis angle: the infrastructure layer between GPU hardware and frontier model providers.
