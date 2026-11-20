---
title: Project loops (P&A mini prompt library)
updated: 2026-11-19
sources: Megatron + Alex fleet experience, Nov 2026; format inspired by Forward-Future/loopy (card #1277)
---

# Project loops

Bounded, reusable agent prompts. Each one says what to do, how to check it, and when to stop.
Every loop ends in a named state: **success, no-op, blocked, approval required, exhausted, or no progress.** An error is never success.
Paste the prompt into any agent (Claude, Codex, local Qwen). Fill the `<angle brackets>`.

---

## 1. The PR-to-mergeable loop
Takes one PR from "open" to "ready for a human to merge", fixing one blocker per pass, and stops when it is ready or stuck.

Prompt:
> For PR <url>, read its current state: CI, review bot comments, open threads, mergeability. Pick the single highest-severity blocker, fix it with a new commit, and get a reviewer who did not write it to approve that exact commit. Re-check the PR. Stop when everything is green and resolved (send me the URL), or when the same blocker survives two passes (tell me why). Never merge, approve, or force-push.

Evidence: shipft tick, gold-fox #30 (one Codex P1 sat for 3 ticks), claudeclaw #92/#101/#110.

## 2. The HOLD-to-SHIP fold loop
Turns a reviewer's HOLD into a SHIP one finding at a time, and stops when the review converges or the design is the problem.

Prompt:
> Take the latest review of <artifact> at <sha>. Fix the highest-severity finding first, with a test that fails before the fix and passes after. Send the new sha back to the same reviewer. Stop at SHIP. If a round produces as many new High findings as it closed, stop patching and tell me the design needs to change, with the pattern the findings share.

Evidence: #1194 spec went r1 to r5, #1231 guard went r1 to r3 with new Highs each round.

## 3. The engine-differential guard loop
Hardens a security check by testing it against the real tool it guards, not a model of that tool.

Prompt:
> For the guard <file>, build an attack list for <protected thing> and run every case through the guard AND through the real engine (<binary and flags the tool actually uses>). Any case the engine reaches but the guard allows is a bypass. Close one bypass class per pass, add its cases to the suite, re-run everything. Stop at zero bypasses plus zero false positives on a normal-work set, or if two passes each find a new class, recommend deny-by-default instead.

Evidence: #1231. fnmatch vs ripgrep globset divergence, comma-split globs, case-insensitive APFS.

## 4. The blocked-mission re-packet loop
When an agent comes back blocked, fix the packet, not the model.

Prompt:
> Mission <id> ended blocked. Classify why: design, execution, environment (cage, auth, missing tool), or changed goal, citing the report line. If environment, gather the missing fact or access yourself, record it in the packet, and re-dispatch with the changed packet. Never re-dispatch an identical packet or switch to a stronger model for the same packet. Stop when it runs, or when the missing fact needs a person (mark it NEEDS ALEX with one question).

Evidence: gold-fox #30 (cage denied .git and GitHub auth; worked once main staged a writable clone), PROCESSES 3C.

## 5. The local-model bake-off loop
Decides whether a new local model or quant replaces the current one, using a held-out set it was not tuned on.

Prompt:
> Compare <candidate model/quant> against the current <role> model on <task set>. Tune prompts only on the working set; score the decision only on the separate held-out set, same settings, recorded. One change per pass (model, quant, or prompt). Keep a change only if it beats the current model on the held-out set without raising false flags. Stop when two passes in a row change nothing, or the candidate wins (then ask me before swapping the live port).

Evidence: Gemma-4 8-bit vs dense 27B reviewer (73% vs 47% recall), 4-bit's floor was the quant. #1104 bake-off.

## 6. The advisory burndown loop
Works a security advisory or upgrade list down one item at a time without breaking the fleet.

Prompt:
> List installed runtimes and tools with a published advisory or newer patched release, each with its source URL. Pick the highest severity item. Upgrade it, then prove it still works: native modules load, a smoke test passes, the version is confirmed. Restarts, credential changes and production touches need my GO first. Stop when the list is empty, or when every remaining item is waiting on my GO (list them with a recommendation).

Evidence: #1162 node 22.23.5 (7 High CVEs, ABI check, rolling restart), #1185 Claude Code 2.1.329.

## 7. The flaky-test root-cause loop
Kills a flaky test by finding the cause, not by retrying it.

Prompt:
> Run <test> <N> times under load (several busy processes) and record the failure rate. Form one hypothesis about the timing or state that decides pass vs fail and make a standalone repro that flips it on demand. Fix the cause, keep the original assertions, and re-run the same N under load. Stop at 0 failures in N, or after two disproved hypotheses (report what you ruled out). Never weaken an assertion to pass.

Evidence: #1170. Racy-clean git index made a filter test fail 1 in 20; pinned mtime, 0/50 under load.

---

## How to add a loop
Only save a loop that (a) fresh feedback can change, (b) has a check someone else could re-run, and (c) names its stop. If another pass would learn nothing new, it is a one-shot checklist, not a loop. Keep each prompt under ~80 words.
