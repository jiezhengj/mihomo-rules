<!-- PROJECT-SPEC-KIT-GOVERNANCE:START -->

# Spec Kit Governance

This repository uses the committed project-local Spec Kit governance package.

Read `docs/spec-kit/START_HERE.md` before substantive engineering work.

A conversational approval such as `the plan is acceptable` advances a direction into the upstream Spec Kit workflow; it does not authorize direct application-code edits before the current Spec Kit artifacts are aligned.

The governance package does not edit `.specify/**`, `specs/**`, or native Agent-generated integration files.

Do not replace the project baseline with personal global rules or a local Reference.

Project documentation language: `zh-CN`.

Write new and substantively rewritten project documentation, including Spec Kit artifacts, in this language unless an explicit user or more specific project instruction overrides it. Do not translate existing documentation solely because this setting was selected.

<!-- PROJECT-SPEC-KIT-GOVERNANCE:END -->
<!-- PROJECT-SPEC-KIT-REFERENCE-UPDATE-CHECK:START version=1 -->

# Spec Kit Reference update check

This check is active only when the current Agent has loaded the global Spec Kit Policy and that Policy provides a readable `SPEC_KIT_GOVERNANCE_SOURCE` absolute path.

When `.specify/` and the committed project governance package are present, run the local governance manager's read-only `check-update --source <central-reference-path>` once before the first substantive task in a new Agent session. If the Policy or source locator is absent, skip this check silently; do not scan the computer for a Reference directory.

If a verified Reference update is available, run the exact hash-bound `auto-upgrade` operation without waiting for project-owner approval. The sync may update only Reference-owned governance files and this managed block; it must never edit `.specify/**`, `specs/**`, native Agent files, or business code. After the governance sync, let the upstream Spec Kit workflow decide whether any specification, plan, or task artifacts need updating.

A missing source, unclean source, invalid verification, offline check, or timeout is non-blocking in normal project work and must not be presented as an available update.

<!-- PROJECT-SPEC-KIT-REFERENCE-UPDATE-CHECK:END -->
