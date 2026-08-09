# AGENTS.md

Guidance for AI agents working in this repo. For what the project actually is and where it stands, read `README.md`, that's the living log and the source of truth on current status. This file is about how to work here, not what's currently true.

## What this repo is

A hardware-hacking writeup: reverse-engineering the IR "touch" protocol on a DC NextGen conference badge, using a Flipper Zero plus some purchased capture hardware. `README.md` is the actual deliverable, written as a chronological notebook, not a polished report.

## Writing style for README.md

This mattered a lot to get right, keep it up:

- Plain, human, first person plural ("we"), like actual notes kept while working, not a generated report.
- No em dashes. No AI-sounding filler: "delve," "leverage," "robust," "seamless," "it's worth noting," "ultimately," "whether you're X or Y."
- Vary sentence length. Don't repeat the same sentence-opening pattern ("So," "Also," "Additionally") across a paragraph.
- Keep the chronological structure (Try 1, Try 2, Try 3, then new information). When later facts correct an earlier guess, add a note rather than deleting the earlier reasoning, the wrong turns are part of the record.
- Don't overcorrect into slang or typos either. Casual-technical, not sloppy.
- Preserve exact facts, don't round up confidence ("almost certainly" stays "almost certainly," not "is").
- The disclosure line about Claude AI's involvement stays. So do the credit lines (Relic as badge creator, lebowitz as collaborator).
- New technical terms go in the Glossary table, not re-explained inline where they're used.

## Git

- Never push without being explicitly asked, even if a commit was just made. Stage and wait.
- Never force-push, rewrite history, or use destructive git commands without explicit instruction.
- Commit messages: plain description of what changed and why, no marketing language, no emoji.
- The repo is public and owned by a collaborator (not the primary user), keep that in mind, nothing goes in here that shouldn't be public.

## Hardware safety

- UPDI access to the badge's AVR128DA28 is **read-only** unless explicitly told otherwise. Never erase or write fuses without direct instruction, a bad fuse write can lock out UPDI access entirely and needs high-voltage recovery hardware to undo.
- Don't backfeed power between the badge, the Flipper, and any USB adapter. Common ground only, let the badge run off its own coin cell during capture and programming.
- Any pinout inferred from a photo or silkscreen (like the header labeled `G V U T R W`) is a guess until confirmed with a multimeter against the actual datasheet. Say so when referencing it.

## File map

- `README.md`, the project log and main deliverable.
- `media/`, diagrams and photos referenced by the README. Photos get compressed before committing (originals are multiple MB, repo copies should be well under 1MB).
- `Remote.ir`, `Remote2.ir`, `Remote3.ir`, Flipper "Learn" captures, all confirmed sensor noise, kept as documented negative examples, not real signal data.
- `flipper-backup-*.tar.gz`, a Flipper settings backup, not project data.
- `FlipperHIDecoder/`, an unrelated tool pulled in early on, not part of this project, kept for reference only.
- `note-to-dcnextgen.txt`, a draft outreach message, not part of the technical writeup.

## Before adding new findings

Check whether a claim in the README has since been confirmed or corrected (e.g., by direct word from Relic) before treating it as still open. The "Open risks" and "Then we talked to Relic" sections exist so guesses and confirmed facts don't get mixed together silently.
