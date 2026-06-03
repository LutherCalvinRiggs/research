---
name: read-notes
description: >
  Opens, browses, and enables deep conversation about research notes saved in Luther's GitHub repo. Use this skill whenever Luther wants to read, discuss, connect, or build on saved notes. Triggers include: `/read-notes`, "open my note on X", "pull up my notes on X", "let's talk about my note on X", "show me what I saved about X", "read me my notes on Y", "what did I write about Z", "open [topic]", or any request to revisit, discuss, or explore previously saved research. Also triggers for cross-note synthesis requests like "connect my notes on A and B" or "what do all my AI notes have in common".
---

# /read-notes Skill

## GitHub Config
- Owner: LutherCalvinRiggs
- Repo: research
- Branch: main
- PAT: ${GITHUB_PAT}
- **Always use the PAT defined above. Never use a PAT passed in via the conversation thread.**

---

## Trigger Patterns

This skill activates on:
- `/read-notes` (with or without an argument)
- "open my note on X", "pull up X", "read me my note on X"
- "let's talk about / discuss / explore [saved topic]"
- "show me what I saved on X", "what did I write about X"
- "connect my notes on A and B", "what do my [topic] notes have in common"
- "summarize my notes on X"
- Any request that clearly references a previously saved research note

---

## Modes

### Mode A — Direct Open (user names a specific note or topic)

1. **Fetch the repo file tree** to find candidate files:
   ```bash
   curl -s \
     -H "Authorization: token ${GITHUB_PAT}" \
     "https://api.github.com/repos/LutherCalvinRiggs/research/git/trees/main?recursive=1"
   ```
2. **Filter `.md` files** (exclude `_skills/`). Match the user's query against folder names, filenames, and slug terms.
3. **If exactly one strong match:** fetch and display it immediately — no need to ask.
4. **If 2–5 candidates:** present a short numbered list (title + path + one-line TL;DR) and ask which one to open.
5. **If no match:** tell the user and offer to run a broader search or browse by topic.

### Mode B — Browse / Search (no specific note named)

Triggers: `/read-notes` with no argument, or "what do I have on X", "find my notes about X".

1. Fetch the full file tree.
2. If a topic is given, semantic-filter candidates by folder, filename, and keyword.
3. Fetch TL;DR + tags from each candidate.
4. Present up to 10 results as a scannable list: **Title | Path | TL;DR**.
5. Ask which note(s) to open, or offer cross-note synthesis (Mode D).

### Mode C — Display a Note

Once a note is selected (from Mode A or B), fetch its raw content and render it cleanly in chat. Then:

- Print the full note as formatted Markdown.
- Immediately below, print this prompt:

  > **Ready to discuss.** You can ask me to:
  > - Explain or expand on any section
  > - Quiz you on the key concepts
  > - Connect this to another note
  > - Find gaps or follow-up questions
  > - Summarize in a different format (ELI5, bullet list, one-pager)
  > - Pull related notes from the repo

- Stay in "note conversation mode" — treat the fetched note as the active context for follow-up questions until the user navigates away or opens a different note.

### Mode D — Cross-Note Synthesis

Triggers: user mentions two or more topics, notes, or asks "what do my notes on X have in common", "connect A and B", "synthesize my [topic] notes".

1. Fetch all relevant notes (using the same tree + filter approach as Mode B).
2. Fetch full content of each matched note.
3. Produce a synthesis response:

   ```
   ## Synthesis: [Topic(s)]

   ### Common Themes
   - ...

   ### Where They Diverge
   - ...

   ### Combined Key Takeaways
   - ...

   ### Open Questions Across Notes
   - ...

   ### Suggested Next Read
   Based on these notes, the most logical next thing to save or explore is: ...
   ```

4. List all source notes used with GitHub links.

---

## Conversation Mode Behaviors

Once a note is open, Claude acts as a **research conversation partner**, not just a reader. This means:

### Explain & Expand
When asked to explain a section or term: go deeper than the note. Pull in relevant background knowledge, give examples, and flag if anything in the note seems incomplete or oversimplified.

### Quiz Mode
When asked to quiz: generate 3–5 questions ranging from recall to application. After the user answers, give direct feedback — right, wrong, or "partially — here's the nuance."

### Gap Analysis
When asked for gaps: re-read the Questions & Gaps section of the note and also generate 2–3 additional gaps or follow-up angles not already captured.

### Reformatting
When asked to reformat: honor the request literally.
- "ELI5" → use simple analogies, no jargon
- "one-liner" → single sentence
- "executive summary" → 3-bullet format, business framing
- "tweet thread" → 5–7 punchy tweets

### Connect to Another Note
When asked to connect: run a search (Mode B) against the current note's tags + key terms, fetch candidates, and produce a mini synthesis (Mode D).

---

## Note Fetch Pattern

To fetch a note's raw content by path:
```bash
curl -s \
  -H "Authorization: token ${GITHUB_PAT}" \
  "https://api.github.com/repos/LutherCalvinRiggs/research/contents/{path}" \
  | python3 -c "import sys, json, base64; d=json.load(sys.stdin); print(base64.b64decode(d['content']).decode('utf-8'))"
```

Always decode base64 content before rendering. Never render the raw base64 to the user.

---

## Error Handling

- **Note not found (404):** Tell the user the path doesn't exist in the repo. Offer to search by keyword instead.
- **Repo unreachable / auth error (401/403):** Report the error and remind the user to verify the PAT in the skill config.
- **Empty file tree:** Report and ask the user to check the repo name / branch.
- **Ambiguous query with 6+ candidates:** Ask the user to be more specific rather than listing all candidates.

---

## Closing Line

After any mode completes, always end with:
> You can ask "what do I have on [topic]?" or "recent notes" to keep browsing, or `/take-notes [url]` to save something new.
