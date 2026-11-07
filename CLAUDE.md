# Instructions for AI-assisted work in this repository

This is a Dynamics 365 Finance & Operations X++ extension for Honegger: package `HOGsbaSubBillActivity`, prefix `HOGsba` (SBA = Subscription billing). Read `README.md` first.

## Before writing or changing anything

- Follow the conventions. **Master:** [youniform](https://github.com/youserpath/youniform) `conventions/` (read it if it is cloned next to this repo, e.g. `../youniform`). **Snapshot:** `docs/conventions/` in this repo, kept so the repo is complete at handover (DEC-11). Never change the snapshot alone: change youniform first, then refresh the snapshot in the same piece of work.
- **Never guess standard object, field or method names.** A name not confirmed on the dev VM is marked `VERIFY` and gets a discovery item (`D-nn` in doc 02 §4). Code against it only after confirmation. Microsoft Learn first, then community sources, and always say which.
- X++ in docs is a draft until it has been compiled on the VM. Say so in the doc.
- Never commit Microsoft standard source code, secrets, or customer production data. Object-name lists and short excerpts quoted in docs are fine.

## Working model

- Discovery, builds and best-practice checks run in the Claude session **on the dev VM** (DEC-13). Cloud sessions can't reach the VM (Tailscale only); they do research, docs and review, and exchange results through this repo.
- On the VM: read standard metadata in `PackagesLocalDirectory`, but **don't query business data in the database** until OP-16 is closed. Never change Microsoft or ISV packages; only our own packages under `Metadata/`.
- The user works on Windows: give **PowerShell** commands, not bash.
- Keep IDs stable and cross-referenced: FR (doc 01), D (doc 02), TC (doc 07), DEC/OP/R (doc 08), ADR. Commit messages and PRs reference these IDs, not issue numbers.
- Question the request and say where it may cause trouble later (upgrades, consolidation, posting, migration to the customer's Azure DevOps).

## Language

English for code, labels (en-US) and docs; German label translations in the label file. Plain, short sentences; tables for structured content.
