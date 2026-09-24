---
name: english-technical-message
description: Use when improving a short English technical message such as a PR comment, GitHub issue, Slack message, or recruiter reply. Mention "improve my message", "fix my english", "polish this comment" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# English Technical Message

## TL;DR checklist

- [ ] Identify the author's intent and audience.
- [ ] Correct grammar and phrasing without changing the technical facts.
- [ ] Return one ready-to-send version in the original tone.

## Key read-only checks

- Read the exact draft and surrounding communication context supplied by the user.

## Common pitfalls

- Do not turn a short operational message into an essay.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [Workflow](#workflow) below.

Last verified: unverified

## When to use

Use when the user has drafted a short English technical message — a PR comment, GitHub issue,
Slack message, or reply to a recruiter — and wants it improved before sending.

## Goal

Make the message natural, clear, and correctly worded at a B2-friendly level, while preserving
the original technical meaning and the author's intent — not turning it into something overly
formal or robotic.

## Workflow

1. Read the draft and identify the actual intent (report a bug, ask a question, push back
   politely, decline something, confirm a decision).
2. Fix grammar, word choice, and phrasing errors without changing the technical content.
3. Keep sentences short and direct — this is a technical message, not an essay; avoid
   over-formalizing wording that was already clear.
4. Preserve the author's tone (casual Slack message stays casual; a formal recruiter reply stays
   professional) rather than flattening everything to the same register.
5. If the message could be read as ambiguous or accidentally blunt/rude in English, flag that
   specifically.
6. Offer one improved version, and — if the original was notably complex — a simpler
   alternative phrasing as well.
7. Briefly explain the key corrections made, so the user learns the pattern for next time.

## Safety rules

- Do not change the technical meaning of the message (e.g. don't soften a real blocker into a
  vague comment).
- Do not over-formalize casual messages (Slack messages don't need to read like a legal memo).
- Do not invent additional content the user didn't say.

## Output format

```
## Improved version
<the corrected message>

## Simpler version (optional)
<only if useful — a more concise/simpler alternative>

## Key corrections
- <original phrase> → <correction> — <why>
```

## Quality checklist

- [ ] Technical meaning is unchanged from the original
- [ ] Tone/register matches the original context (Slack vs. formal reply)
- [ ] Corrections are explained briefly, not just silently applied
- [ ] No content was invented beyond what the user wrote
- [ ] Wording stays at a natural, B2-friendly level, not artificially advanced
