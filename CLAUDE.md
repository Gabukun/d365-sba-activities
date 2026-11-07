# Instructions for AI-assisted work in this repository

This is a Dynamics 365 Finance & Operations X++ extension for Honegger: package `HOGsbaSubBillActivity`, prefix `HOGsba` (SBA = Subscription billing). Read `README.md` first.

## Before writing or changing anything

- Follow `docs/conventions/01-naming-and-extensions.md`, `02-testing.md` and `03-alm-and-source-control.md`. When a convention changes, update the convention file in the same change. These files will move to the central repository *youniform* (DEC-11); then that copy wins.
- **Never guess standard object, field or method names.** A name not confirmed on the dev VM is marked `VERIFY` and gets a discovery item (`D-nn` in doc 02 §4). Code against it only after confirmation. Microsoft Learn first, then community sources, and always say which.
- X++ in docs is a draft until it has been compiled on the VM. Say so in the doc.
- Never commit Microsoft standard source code, secrets, or customer production data. Object-name lists and short excerpts quoted in docs are fine.

## Working model

- The dev VM is reachable only over the user's Tailscale network. A cloud session can't reach it, so ask for results from the VM (call stacks, short excerpts, CSV object lists) or work in a session running on the VM.
- The user works on Windows: give **PowerShell** commands, not bash.
- Keep IDs stable and cross-referenced: FR (doc 01), D (doc 02), TC (doc 07), DEC/OP/R (doc 08), ADR. Commit messages and PRs reference these IDs, not issue numbers.
- Question the request and say where it may cause trouble later (upgrades, consolidation, posting, migration to the customer's Azure DevOps).

## Language

English for code, labels (en-US) and docs; German label translations in the label file. Plain, short sentences; tables for structured content.
