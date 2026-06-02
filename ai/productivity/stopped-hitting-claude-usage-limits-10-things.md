# I Stopped Hitting Claude's Usage Limits — 10 Things I Changed

**Source:** https://x.com/0x_kaize/status/2061475678143365248?s=42
**Fetched from:** https://x.com/0x_kaize/status/2038286026284667239 (X article by @0x_kaize, indexed version)
**Saved:** 2026-06-02
**Tags:** ai, productivity, claude, token-optimization, prompting, llm

---

## TL;DR
Claude doesn't count messages — it counts tokens. This X article by @0x_kaize lays out 10 practical habits to use tokens more efficiently, so you hit usage limits far less often and often get better results in the process.

## Key Concepts & Terms
- **Tokens**: The unit Claude uses to measure usage — not messages. Every word, character, and piece of context costs tokens. The full conversation history is re-read on every single turn.
- **Context window**: Claude's working memory per session. The longer a chat grows, the more tokens each new reply costs because all prior messages are included.
- **Projects (Claude feature)**: A built-in way to persist shared context (like uploaded PDFs or system instructions) across multiple chats without re-uploading and re-tokenizing every time.

## Main Arguments & Takeaways
- **Don't send correction messages** — when Claude misunderstands, avoid sending a separate "No, I meant X" follow-up. Every additional message adds to the conversation history that Claude re-reads every turn, burning tokens on context that didn't help.
- **Batch your questions into one prompt** — three separate prompts equals three full context reloads. One prompt with three tasks equals one context load. Bundling saves tokens twice: fewer reloads, and you stay further from your limit. Instead of "Summarize this," then "List the main points," then "Suggest a headline," write it all in one message.
- **Summarize and restart long chats** — when a chat gets long, ask Claude to summarize everything, copy the summary, open a new chat, and paste it as the first message. This resets the context clock without losing continuity.
- **Use Projects for repeated documents** — uploading the same PDF to multiple chats forces Claude to re-tokenize it every single time. The Projects feature stores that context persistently, eliminating repeated overhead.
- **The compounding cost problem**: Token cost per message = all previous messages + your new one. This compounds fast. The root insight is that most people lose tokens not from what they ask, but from how conversation history accumulates around it.

## Notable Quotes
> "Claude doesn't count messages. It counts tokens." — @0x_kaize

> "Three separate prompts = three context loads. One prompt with three tasks = one context load." — @0x_kaize

## Questions & Gaps
- The article covers 10 habits but only a subset are captured in public search snippets — what are the remaining tips?
- Does the "summarize and restart" strategy meaningfully reduce response quality compared to keeping the full history intact?
- Are these optimizations equally effective across Pro, Max, and API tiers, or do the economics differ significantly?
- How much do different content types (code blocks, images, long documents) vary in token cost — and does the article address this?
- As Anthropic iterates on pricing and session limit models, how durable are these workarounds over time?