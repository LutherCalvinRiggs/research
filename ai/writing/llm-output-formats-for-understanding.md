# Better LLM Outputs: ASD-STE100, Diagrams, Web Pages, and Explainer Videos

**Source:** Andrej Karpathy (Twitter/X post, pasted text, no URL provided) + attached ASD-STE100 overview infographic
**Saved:** 2026-10-02
**Tags:** ai, writing, llm, asd-ste100, explainers, oversight

---

## TL;DR
Karpathy suggests changing the *format* of LLM output so it is easier for humans to understand: controlled writing (ASD-STE100), then diagrams, then interactive HTML pages, then custom explainer videos. As AI does more of the work, our job shifts to oversight and understanding, and cheap, throwaway software artifacts can help.

## Key Concepts & Terms
- **ASD-STE100 (Simplified Technical English)**: A controlled-language spec made for aircraft maintenance documents. It has Part 1 (nine sections of writing rules) and Part 2 (a dictionary of about 900 approved words).
- **Approved vs. unapproved words**: Each word has one approved meaning and part of speech. "Close" is approved as a verb ("close the valve") but not as an adjective ("near").
- **Noun cluster**: A string of nouns used as one name. The limit is 3 words.
- **Procedural vs. descriptive sentence**: Procedural sentences give instructions (max 20 words). Descriptive sentences explain (max 25 words).
- **"80% of the way to STE"**: Karpathy's softened prompt, because the full spec is very strict.
- **Discardable software artifacts**: Custom web apps or videos made for one topic that would never have been worth building by hand.
- **3b1b-style explainer**: A video in the style of 3Blue1Brown, with animated math and visual explanations.

## Main Arguments & Takeaways
- We will spend much more time *understanding* model output, so output format matters.
- Step 1, Writing: ask the LLM to explain something in ASD-STE100. Models know it well, and its constraints produce clean, readable prose.
- The spec is stringent, so asking for "80% of the way" is a practical compromise.
- Step 2, Diagrams: a diagram is often easier to process and parse than text.
- Step 3, Web pages: ask for output "in HTML" to get interactive pages, animations, and polished layouts.
- Step 4, Explainer videos: the format he is most bullish on. Example prompt: "Create a 3b1b style video explainer on X. Use my ElevenLabs API key for audio narration."
- Narration needs an API key, or free alternatives that run on local compute, which the LLM can find.
- Big picture: LLMs will do more legwork on their own, and human work moves up to oversight and understanding.
- Because intelligence and code are becoming abundant, large custom throwaway artifacts are now practical.

## What the STE infographic adds
- **Sentence limits:** procedural max 20 words, descriptive max 25, paragraphs max 6 sentences, noun clusters max 3 words, one instruction per sentence (except simultaneous actions).
- **Core rules:** use the same word for the same thing, keep "the", "a", and "this", use the active voice in procedures, use vertical lists for complex text, write one topic per paragraph.
- **Verb forms:** approved are imperative, simple present, past, future, infinitive, and past participle as an adjective. Not approved are progressive (-ing), perfect, and passive in procedures.
- **Word swaps:** ensure → MAKE SURE, prior to → BEFORE, replenish → FILL, utilize → USE, approximately → ABOUT, in order to → TO, commence → START.
- **Safety instructions:** give a clear, simple command first, then the risk. WARNING means risk of injury and CAUTION means risk of damage.
- **History:** 1979 AECMA works on controlled English for airlines. 1986 the first Simplified English Guide is published. 2005 it becomes ASD-STE100. Today it is a free download, revised by the STEMG maintenance group.

## Notable Quotes
> "80% of the way to ASD-STE100" — his softened version of the spec, because it is "quite stringent."

> "you can ask for large, custom, discardable software artifacts (e.g. web apps, video explainers) that would have never made sense to create before."

> "Create a 3b1b style video explainer on X. Use my ElevenLabs API key for audio narration."

## Questions & Gaps
- No evidence beyond personal experience that STE improves comprehension. Is there research on this?
- STE was built for procedures. Does it work for abstract or conceptual topics, or does it flatten nuance?
- Generated diagrams, pages, and videos can look convincing while being wrong. How do we verify them?
- What is the cost and effort of the video workflow (APIs, rendering, local TTS alternatives)?
- Which of the four formats fits which kind of content?

## Next steps to explore
- Build a reusable "80% STE" prompt and test it on dense notes.
- Try the same topic as text, diagram, and HTML page and compare how well you retain it.

## Related Notes
- [Comprehension Debt — Addy Osmani](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/comprehension-debt-addy-osmani.md): the risk Karpathy's "oversight and understanding" framing is trying to reduce, since AI output you cannot easily follow becomes understanding you never built.
- [Cognitive Surrender — Addy Osmani](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/cognitive-surrender-addy-osmani.md): the failure mode of accepting AI output without judging it; more readable formats make real review easier.
- [Own the Outer Loop — Agentic Accountability](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/own-the-outer-loop-agentic-accountability.md): the same shift of human work toward oversight, applied to agent-built software.
