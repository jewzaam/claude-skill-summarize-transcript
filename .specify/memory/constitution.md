<!--
Sync Impact Report
- Version change: N/A → 0.1.0
- Added principles:
  - I. Accuracy & Impartiality
  - II. Source Fidelity
  - III. Transparency
  - IV. Graceful Degradation
  - V. User Control
- Added sections: Governance
- Removed sections: SECTION_2, SECTION_3 (not needed for this project)
- Templates requiring updates:
  - .specify/templates/plan-template.md — ✅ no changes needed
    (Constitution Check section is generic; gates are filled per-feature)
  - .specify/templates/spec-template.md — ✅ no changes needed
    (Template is domain-agnostic; principles apply at review time)
  - .specify/templates/tasks-template.md — ✅ no changes needed
    (Task structure is generic; principle compliance is a review concern)
- Follow-up TODOs: none
-->

# Summarize-Transcript Skill Constitution

## Core Principles

### I. Accuracy & Impartiality

The skill MUST produce output that contains only information
clearly present in the source transcript(s). No context,
assumptions, interpretations, or conclusions may be added
beyond what participants stated.

- Speaker positions in disagreements MUST be attributed to
  the correct speaker.
- Strong language MUST be preserved verbatim — never softened.
- Direct quotes MUST be accurate.
- Speculative or uncertain attributions MUST be marked
  (e.g., `(likely "word")`, `(unclear term)`).
- Action items and decisions MUST trace to specific statements
  in the source material.

Rationale: Fabricated or editorialized content erodes trust
and can misrepresent participants' positions.

### II. Source Fidelity

The skill MUST preserve the texture and character of the
original discussion. Summaries are not sanitized minutes —
they are faithful records.

- Uncertainty markers ("I don't know", "maybe", incomplete
  thoughts) MUST be preserved in the Timeline section.
- Rambling speech patterns, run-on sentences, and exploratory
  dialogue MUST NOT be over-organized into clean bullet points.
- Meeting confusion, scheduling issues, and process problems
  MUST be included — they are context, not noise.
- Transcription artifact cleanup (merged fillers, spurious
  spaces) is permitted because it corrects engine errors, not
  speaker intent.

Rationale: Over-polishing a transcript creates a false
impression of structure that did not exist. Readers need to
calibrate how firm a decision was based on how it was discussed.

### III. Transparency

Every editorial addition, limitation, and correction MUST be
visible to the reader. The skill MUST NOT silently alter
meaning or attribution.

- Section headers added by the skill MUST use `[Editorial: ...]`
  format.
- Post-summary corrections MUST use square brackets
  (`[corrected word]`) to distinguish them from original content.
- Source quality limitations (closed-caption fidelity, missing
  timestamps, incomplete attendee lists) MUST be disclosed in
  the Document Status section.
- When multiple sources are used, each MUST be listed with its
  type (primary/supplementary) and any conflicts between sources
  MUST be noted — never silently resolved.

Rationale: Readers cannot assess trustworthiness without
knowing what the skill added, what it could not verify, and
where the source material was unreliable.

### IV. Graceful Degradation

The skill MUST produce useful output regardless of input
completeness. Missing optional inputs reduce output quality
but MUST NOT cause failure.

- Missing roster file: proceed without roster-based name
  resolution.
- Missing metadata.json: infer meeting title and date from
  transcript content.
- Malformed roster file: report the parsing error clearly,
  then proceed without roster-based resolution.
- Missing glossary file: skip technical term correction
  without error.
- Single recognized file in a directory: behave identically
  to single-file input mode.
- No recognized files in a directory: inform the user and
  stop — do not produce an empty summary.

Rationale: Meeting transcripts arrive in unpredictable
combinations. A skill that fails on missing optional inputs
forces users into manual workarounds that defeat the purpose
of automation.

### V. User Control

The user is the final authority on names, corrections, and
output content. The skill provides defaults and suggestions
but MUST NOT make irreversible decisions without user input.

- Roster files are user-maintained artifacts — the skill reads
  them but MUST NOT modify them without explicit user consent.
- Post-hoc name corrections MUST be offered interactively, not
  applied silently.
- Unresolved names MUST be flagged for user review rather than
  guessed at.
- The skill MUST offer to persist corrections back to the
  roster but MUST NOT do so without confirmation.

Rationale: Automated name resolution will always have edge
cases. Users who maintain their own rosters and approve
corrections retain confidence in the output.

## Governance

This constitution is the authoritative guide for the
summarize-transcript skill's behavior and design decisions.
All specifications, plans, and implementations MUST comply
with its principles.

- **Amendments** require documentation of the change, the
  rationale, and a version bump following semantic versioning:
  - MAJOR: Principle removal or backward-incompatible
    redefinition.
  - MINOR: New principle added or existing principle
    materially expanded.
  - PATCH: Clarifications, wording fixes, non-semantic
    refinements.
- **Compliance review**: Each spec and plan MUST include a
  Constitution Check verifying alignment with these principles.
- **Conflict resolution**: If a design decision conflicts with
  a principle, the decision MUST either be revised or the
  constitution amended — silent non-compliance is not permitted.

**Version**: 0.1.0 | **Ratified**: 2026-03-24 | **Last Amended**: 2026-03-24
