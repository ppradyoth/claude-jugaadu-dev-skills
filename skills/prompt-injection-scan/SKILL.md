---
name: prompt-injection-scan
description: >
  Scan a whole codebase for prompt-injection risk — untrusted input flowing into an LLM prompt
  without a guardrail (OWASP LLM01). Use when the user says "scan for prompt injection",
  "is my LLM app safe", "check my prompts", "LLM security audit", "injection scan", or invokes
  /prompt-injection-scan. Shell-first: grep finds the risky call sites, AI only reasons about
  the ones grep surfaces. This is the repo-wide audit; /secure-diff is the fast pass on a diff.
---

# Prompt Injection Scan

Find the spots where untrusted input reaches a model prompt with nothing in between.

That's the whole bug class. User text, a tool result, a scraped web page, a filename — anything an attacker can influence — gets concatenated into a prompt and the model treats it as instructions. OWASP calls it LLM01. It's still the #1 LLM risk because it's so easy to write by accident.

This skill greps for the *shape* of that mistake across the repo, then reasons about the hits. It does not run your app and it does not send anything anywhere.

## Steps

1. Detect the SDK in use so you know what to grep for:

```bash
grep -rEl "anthropic|openai|langchain|llama_index|google\.generativeai|genai|cohere|mistralai|bedrock|ollama" \
  --include="*.py" --include="*.js" --include="*.ts" . 2>/dev/null | head -20
```

2. Find prompt construction sites — the lines that build the string the model sees:

```bash
# Python: f-strings / .format / % / + into a message field
grep -rnE "(system|user|prompt|content|messages|instruction)\s*=\s*(f\"|f'|.*\.format\(|.*%|.*\+)" \
  --include="*.py" . 2>/dev/null

# JS/TS: template literals into a message field — both `x = ` and `x: ` forms
grep -rnE "(system|user|prompt|content|messages|instruction)\s*[:=]\s*[\`\"'].*\\$\{" \
  --include="*.js" --include="*.ts" . 2>/dev/null
```

3. Find where untrusted input enters, so you can trace whether it reaches step 2:

```bash
# Web request bodies, query params, form fields
grep -rnE "request\.(json|form|args|body|data|query|params)|req\.(body|query|params)|\.get_json\(" \
  --include="*.py" --include="*.js" --include="*.ts" . 2>/dev/null

# Tool / function-call results fed back to the model (the injection vector people forget)
grep -rnE "tool_result|function_response|tool_outputs|observation|retrieved|search_results|page_content" \
  --include="*.py" --include="*.js" --include="*.ts" . 2>/dev/null
```

4. For each prompt-construction site from step 2, trace back one hop: does an untrusted value from step 3 flow into it? Read the ~15 lines around the hit — don't guess. Only flag a site when you can name the tainted variable.

5. Report findings in this format:

```
## Prompt Injection Findings

[HIGH]  path/to/file.py:42
  Untrusted `request.json["message"]` is concatenated into the system prompt.
  Model will follow instructions the user hides in that field.
  Fix: move user text to a user-role message, never the system prompt; add a delimiter
       and an instruction-hierarchy note; validate/limit length.

[MED]   path/to/rag.py:88
  Retrieved document `page_content` is inlined into the prompt with no framing.
  A poisoned document can hijack the response (indirect injection).
  Fix: wrap retrieved text in explicit delimiters, label it as untrusted data,
       and instruct the model to treat it as content, not commands.

## Not a finding
  path/to/x.py:10 — string is a hardcoded constant, no untrusted input reaches it.

## Verdict
  [one sentence: "2 real injection sinks, both in the RAG path — fix the system-prompt one first."]
```

## Rules

- Grep finds candidates. You confirm by reading the surrounding code — never flag a line you haven't traced to an untrusted source.
- Severity is about the *sink*, not the input: untrusted text in the **system** prompt or in a **tool-enabled** agent loop is HIGH; untrusted text in a plain user-role message with no tools is usually LOW.
- Indirect injection counts. Retrieved docs, tool results, scraped pages, filenames, and email bodies are attacker-influenced too — most scanners miss these; you shouldn't.
- Don't propose a WAF-style regex blocklist as "the fix." It isn't. The durable fixes are: keep untrusted text out of the system prompt, use role separation, delimit and label untrusted spans, constrain tool permissions, and validate output before it acts.
- Report the file:line so the user can jump straight to it.
- Zero false-confidence. If you can't trace the taint, say "needs manual review," don't call it clean.

## Gotchas

- A hit in step 2 is not a bug by itself — plenty of prompts are built from constants. The bug is only there when step 3 flows into it.
- Framework helpers hide the sink. LangChain `PromptTemplate`, LlamaIndex query engines, and agent wrappers concatenate for you — grep for the template *definition* and check what fills the variables.
- The most dangerous finding is usually the quietest: a tool result silently appended to the conversation. No user-facing input, full attacker control.
