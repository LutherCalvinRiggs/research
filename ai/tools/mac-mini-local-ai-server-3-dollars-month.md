# Run Claude Code on a Mac Mini for $3/Month — Local AI Server Guide

**Source:** https://x.com/hey_madni/status/2063992615721435154 (pasted content by @hey_madni)
**Saved:** 2026-06-10
**Tags:** ai, tools, infrastructure, productivity, tools, economics

---

## TL;DR
Since Ollama v0.14.0 (Jan 2026) added native Anthropic Messages API compatibility, you can point Claude Code at `localhost:11434` instead of `api.anthropic.com` and run the full agentic loop locally — file edits, tool calls, terminal commands — against a local model for $0 in API costs. A Mac Mini M4 (16GB, $599) handles 70% of daily AI work; the M4 with 32GB ($799) runs 14B models for serious coding. Hybrid strategy: local for 80%, one $20/mo subscription for the hard 20% = $23/month instead of $459.

## Key Concepts & Terms
- **Unified Memory Architecture (UMA)**: Apple Silicon's core hardware advantage for AI inference. CPU and GPU share one memory pool — the model loads once and both read from it directly. On a standard PC, data constantly copies between system RAM and GPU VRAM, which kills inference speed. A $599 Mac Mini runs inference faster than a $1,500 Windows machine with a discrete GPU as a result.
- **Memory bandwidth**: The true determinant of token generation speed — not chip generation or marketing specs. M4: 120 GB/s. More bandwidth = faster responses.
- **Ollama**: Free open-source runtime for local LLMs. As of v0.14.0 (Jan 2026), speaks the Anthropic Messages API natively, meaning Claude Code can be pointed at it directly with no other changes.
- **`ANTHROPIC_BASE_URL`**: The environment variable Claude Code uses to find its backend. Setting it to `http://localhost:11434/v1` redirects all Claude Code calls to the local Ollama instance.
- **`OLLAMA_KEEP_ALIVE=-1`**: Keeps the model loaded in memory indefinitely rather than unloading after 5 minutes of inactivity. Critical for responsive agentic workflows.
- **Open WebUI**: Dockerized web interface that puts a ChatGPT-style browser UI on top of local Ollama. Nothing leaves your machine.
- **Hybrid strategy**: Local Mac Mini for 80% of daily tasks (drafting, summarizing, standard coding); one cloud subscription for the hard 20% (complex reasoning, frontier-level tasks). Total: ~$23/month vs. $459/month for full cloud stack.

## The Hardware Math

| Spec | M4 base ($599) | M4 with 32GB ($799) | M4 Pro ($1,399) |
|------|---------------|---------------------|-----------------|
| RAM | 16GB | 32GB | 48–64GB |
| Power under load | 10–20W | 10–20W | 20–30W |
| Electricity/month | ~$2–3 | ~$2–3 | ~$3–5 |
| Largest model | Up to 14B (slow) | Up to 14B (comfortable) | Up to 70B |
| Best use | 70% of daily work | Serious coding, analysis | Frontier-level local |

Mac Mini vs. Windows AI box:
- Mac Mini: 10–20W under load = $2–3/month electricity
- Windows AI box: 300–500W = $30–50/month electricity
- Over a year: $360–$600 extra just in power for the Windows box, before hardware cost

## What Actually Runs (Honest Breakdown)

| Model | Params | RAM needed | Best for |
|-------|--------|-----------|---------|
| Gemma 4 4B | 4B | 16GB | Quick tasks, email, drafts |
| Qwen 3.6 8B | 8B | 16GB | Coding, writing |
| Mistral 7B | 7B | 16GB | General use |
| Qwen 3.6 14B | 14B | 32GB | Serious coding, analysis |
| DeepSeek R1 14B | 14B | 32GB | Reasoning, math, logic |
| Llama 3.3 70B | 70B | 64GB | Frontier-level tasks |

**Honest caveat from the article**: The $599/16GB base model handles 70% of daily work — drafting, summarizing, coding scripts, Q&A. It is not a replacement for Claude Opus on complex agentic workflows. Be honest about that.

XDA Developers (April 2026) tested 32GB + Qwen 3.6 14B as a Claude Pro replacement and concluded "productivity didn't drop a bit."

## The Three-Command Setup

```bash
# Step 1 — Install Ollama (2 minutes)
curl -fsSL https://ollama.com/install.sh | sh

# Step 2 — Pull a model
ollama pull qwen3.6:14b

# Step 3 — Connect Claude Code to local model
ANTHROPIC_BASE_URL=http://localhost:11434/v1 claude
```

**Keep model loaded in memory** (add to `~/.zshrc`):
```bash
export OLLAMA_KEEP_ALIVE="-1"
```

**Add a browser interface** (Open WebUI — private ChatGPT on your local network):
```bash
docker run -d -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  ghcr.io/open-webui/open-webui:main
# → open localhost:3000
```

## The Full Local Stack

| Component | Tool | Cost |
|-----------|------|------|
| Hardware | Mac Mini M4 | $599 one-time |
| Runtime | Ollama v0.14.0+ | Free |
| Browser interface | Open WebUI | Free |
| Coding agent | Claude Code → localhost | $0 API |
| AI agent | OpenClaw → Ollama backend | $0 API |
| Recommended models | Qwen 3.6 14B, DeepSeek R1 14B, Gemma 4 4B | Free |
| Power | ~$3/month electricity | — |

## Connecting OpenClaw to Local Ollama

```json
// ~/.openclaw/openclaw.json
{
  "vllm": {
    "baseUrl": "http://localhost:11434/v1",
    "apiKey": "VLLM_API_KEY",
    "api": "openai-completions"
  }
}
```

One Mac Mini then runs simultaneously:
- Claude Code → local Ollama (zero API cost)
- OpenClaw agent on Telegram/WhatsApp (24/7, zero API cost)
- Open WebUI (private ChatGPT for local network)

## The Cost Comparison

**Before (full cloud stack):**
| Subscription | Monthly | Annual |
|-------------|---------|--------|
| Claude Code Max (20x) | $200 | $2,400 |
| ChatGPT Pro | $200 | $2,400 |
| Gemini Advanced | $20 | $240 |
| GitHub Copilot | $19 | $228 |
| Cursor Pro | $20 | $240 |
| **Total** | **$459** | **$5,508** |

**After (hybrid strategy):**
- Mac Mini: $599 one-time
- Electricity: $3/month
- One cloud subscription for hard 20%: $20/month
- **Total ongoing: ~$23/month — saves $5,232/year**

**Enterprise scale context**: Uber rolled out Claude Code to 5,000 engineers. Per-person bills climbed to $500–$2,000/month. They burned through their entire 2026 AI budget ($3.4B) in four months.

## Who Should Actually Buy This

| Situation | Verdict |
|-----------|---------|
| Paying $200+/month on AI APIs | Buy it — pays off in 3 months |
| Privacy-critical work | Buy it — data never leaves |
| Heavy Claude Code user ($6+/day) | Buy it — ROI in 30 days |
| Casual ChatGPT user ($20/month) | Skip — math doesn't work |
| Need frontier models (Opus, GPT-5 level) | Keep subscription + use local for 80% |

## Questions & Gaps
- The Ollama v0.14.0 Anthropic Messages API compatibility — how complete is it? Are there Claude Code features (tool use schemas, specific API parameters) that don't translate cleanly to local model inference?
- The "70% of daily work" claim for 16GB/8B models — is this validated against actual agentic coding workloads, or subjective daily use? Complex multi-file refactors or NEEDLE-style headless workers likely need at least 14B.
- Token generation speed at 120GB/s bandwidth for 14B models — what's the actual tokens/second number? The article doesn't give a concrete benchmark.
- The article assumes OpenClaw works correctly against Ollama's OpenAI completions API — is this validated or does it require specific model instruction-following capability?
- For NEEDLE fleet use: could local Ollama serve as the agent backend for bead workers, replacing Claude API costs? The `ANTHROPIC_BASE_URL` redirect is exactly the same mechanism NEEDLE would use.

## Related Notes
- [I Stopped Hitting Claude's Usage Limits](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/stopped-hitting-claude-usage-limits-10-things.md) — token optimization habits at the session level; this note is the infrastructure-level solution to the same problem.
- [Unit Economics of Running Cattle](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/unit-economics-of-cattle.md) — Jed's cost governance framework for NEEDLE fleets. Local Ollama as the agent backend would eliminate the API spend layer entirely for commodity tasks, changing the fleet economics significantly.
- [Self-Hosted AI Sandboxes Guide](https://github.com/LutherCalvinRiggs/research/blob/main/ai/infrastructure/self-hosted-ai-sandboxes-guide-2026.md) — the Mac Mini local setup is self-hosted inference; the sandbox guide covers the isolation layer needed when those local models are executing code.
- [Benchmarks Measure a Model You Aren't Running](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/benchmarks-measure-a-model-you-arent-running.md) — the claim "Qwen 3.6 14B handles real coding tasks" should be read through the lens of that note: benchmarks for these models were run at near-empty context, which is rarely the deployment condition.
- [Meta/AWS Graviton5/Firecracker](https://github.com/LutherCalvinRiggs/research/blob/main/ai/infrastructure/meta-aws-graviton5-firecracker-agentic-ai.md) — the CPU-to-GPU parity shift described there (orchestration becoming the bottleneck, not inference) suggests the Mac Mini's advantage is primarily on the inference side; orchestration overhead still applies.
