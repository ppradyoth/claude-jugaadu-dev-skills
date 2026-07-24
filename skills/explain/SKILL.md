---
name: explain
description: >
  Explain what a piece of code actually does — a function, a file, a regex, a gnarly one-liner —
  in plain English, fast. Use when the user says "explain this", "wtf does this do", "what is this
  code doing", "walk me through this file", "explain this regex", or invokes /explain. Reads the
  real code first; never guesses from the name. Top-down: the point before the line-by-line.
---

# Explain

Someone is staring at code they don't understand. Maybe it's someone else's. Maybe it's theirs from
six months ago. Either way they want the *gist* first, then the detail — not a line-by-line crawl
that buries the point.

This reads the actual code and explains it top-down. The one-sentence "what is this for" comes
first. The mechanics come after, and only as deep as the thing warrants.

## Steps

1. **Read the real thing.** If the user pointed at a file or symbol, open it. Don't explain from the
   name — names lie. If they pasted a snippet, use that. If it calls something non-obvious in the
   same repo, glance at that too (one hop, not a spelunk).

2. **Lead with the purpose.** One sentence: what this code is *for*, in the caller's terms — what
   you'd put in a docstring, not a restatement of the syntax.
   > "Validates a JWT and returns the user id, or throws if the token is expired or tampered with."

3. **Then the shape**, sized to the input:
   - **A function** → inputs → what it does → outputs → side effects (I/O, mutation, network). Call
     out anything surprising: early returns, swallowed errors, hidden state.
   - **A file/module** → its job in the system, the handful of exports that matter, and how they
     fit together. Skip the boilerplate.
   - **A regex** → say what it matches in words, then break it into named chunks. Give one string it
     matches and one it doesn't.
   - **A dense one-liner** → unfold it into the 3–4 plain steps it's secretly doing.

4. **Flag the sharp edges.** If something is a bug magnet, a footgun, or just non-obvious — an
   off-by-one, an unhandled `null`, an `await` missing in a loop, O(n²) hiding in a `.includes` —
   say so. This is often the real reason they asked.

5. **Match the depth to the ask.** "what does this do" → a tight paragraph. "walk me through it" →
   the full structured pass. Don't write an essay for a three-line function.

## Rules

- Purpose first, always. If the reader stops after one sentence, they should still know what it does.
- Plain English over jargon. "Runs once per item" beats "iterates the collection applying the callback."
- Quote the specific line when you reference behavior — `file.ts:42` so they can jump to it.
- Never invent behavior you didn't read. If a called function is opaque and you didn't open it, say
  "this calls `X`, which I haven't traced" rather than guessing what `X` does.
- No "this code is a function that..." filler. State what it accomplishes.
- If the code is genuinely simple, say so in a line and stop. Over-explaining simple code is noise.
- If you spot a real bug while explaining, surface it — but keep it separate from the explanation so
  the answer to "what does this do" stays clean.
