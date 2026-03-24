# Feature Specification: People Name Resolution

**Feature Branch**: `001-people-name-resolution`
**Created**: 2026-03-24
**Status**: Draft
**Input**: Bootstrap from Plan-people-names.md - feature for identifying and resolving people names in transcript summaries

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Resolve Names from Roster File (Priority: P1)

As a user summarizing transcripts for a recurring team, I want to maintain a roster file that maps first names, garbled variants, and full names so that the skill automatically produces summaries with correct, unambiguous full names on first mention.

**Why this priority**: Name resolution is the core value of this feature. A roster file provides the highest-fidelity, user-controlled source of truth and handles the most common case: recurring meetings with the same team.

**Independent Test**: Can be tested by placing a roster file alongside a transcript containing garbled and first-name-only references, running the skill, and verifying the summary uses correct full names on first mention.

**Acceptance Scenarios**:

1. **Given** a roster file exists in the transcript directory with entries for known attendees, **When** the skill generates a summary, **Then** all first-name-only references are expanded to full names on first mention in each section, and garbled variants listed in the roster are replaced with the correct full name.
2. **Given** a roster file exists but a name in the transcript does not match any roster entry or variant, **When** the skill generates a summary, **Then** the unmatched name appears as-is and is flagged for user review in the post-summary output.
3. **Given** no roster file exists in the transcript directory or any parent directory, **When** the skill generates a summary, **Then** the skill proceeds without roster-based resolution and relies on the post-hoc correction flow.

---

### User Story 2 - Resolve "You" to Transcript Owner (Priority: P1)

As a user who records meetings via Google Meet closed captions, I want "You" references in the transcript to be automatically replaced with my full name so that the summary correctly attributes my statements.

**Why this priority**: Every closed-caption transcript contains "You" as the speaker label for the meeting owner. Without resolution, the summary is immediately ambiguous about a primary participant.

**Independent Test**: Can be tested by providing a transcript containing "You" speaker labels alongside a transcript owner declaration, running the skill, and verifying "You" is replaced with the owner's full name.

**Acceptance Scenarios**:

1. **Given** a transcript owner is declared in the roster file or metadata, **When** the skill encounters "You" as a speaker, **Then** "You" is replaced with the transcript owner's full name.
2. **Given** no transcript owner is declared anywhere, **When** the skill encounters "You" as a speaker, **Then** the skill uses the existing behavior (prompt-based or passthrough) and flags "You" for user review.

---

### User Story 3 - Post-Hoc Name Correction (Priority: P2)

As a user summarizing a meeting with new or unfamiliar participants, I want the skill to present all detected names after generating the summary so I can correct garbled names, provide full names, and optionally update the roster for future use.

**Why this priority**: Rosters cannot cover every scenario — new participants, ad-hoc meetings, and unpredictable garbling all produce names the roster does not contain. Post-hoc correction provides a safety net.

**Independent Test**: Can be tested by running the skill on a transcript with names not in any roster file, verifying the skill presents a list of detected names with confidence indicators, and confirming corrections are applied to the summary.

**Acceptance Scenarios**:

1. **Given** the skill has generated a summary containing names not found in the roster, **When** the summary is complete, **Then** the skill presents a list of all unique names found in the transcript with indicators of which were resolved via roster and which are unresolved.
2. **Given** the user provides corrections (e.g., "cup its" -> "Stephen Cuppet"), **When** corrections are applied, **Then** the summary is updated with the corrected names.
3. **Given** the user provides corrections, **When** the user confirms, **Then** the skill offers to add the new name mappings to the roster file (creating it if it does not exist).

---

### User Story 4 - Full-Name-on-First-Mention Convention (Priority: P2)

As a reader of meeting summaries, I want full names used on the first mention of each person within each section (Summary, Timeline) so that the summary is unambiguous without being cluttered with repeated full names.

**Why this priority**: This is a formatting convention that directly improves readability for audiences beyond the original attendees, which is the stated use case for these summaries.

**Independent Test**: Can be tested by verifying that each section of the generated summary uses a person's full name on first mention and first name only for subsequent references within the same section.

**Acceptance Scenarios**:

1. **Given** a person's full name is known (from roster or user correction), **When** the summary is generated, **Then** the full name appears on the first mention in each major section, with first name used for subsequent references within that section.
2. **Given** a person's full name is not known, **When** the summary is generated, **Then** the name appears as found in the transcript (no fabricated full names).

---

### User Story 5 - Roster File Discovery (Priority: P3)

As a user who works with multiple teams, I want the skill to search for roster files in parent directories so that I can maintain a single roster per team rather than duplicating it for every meeting transcript.

**Why this priority**: Reduces roster maintenance burden for users with recurring teams. Useful but not critical for the feature to function.

**Independent Test**: Can be tested by placing a roster file in a parent directory, running the skill on a transcript in a subdirectory, and verifying the roster is discovered and used.

**Acceptance Scenarios**:

1. **Given** no roster file exists in the transcript directory but one exists in a parent directory, **When** the skill runs, **Then** the parent directory roster is used for name resolution.
2. **Given** roster files exist in both the transcript directory and a parent directory, **When** the skill runs, **Then** the transcript-directory roster takes precedence, with parent roster entries used as fallback for names not found locally.

---

### Edge Cases

- What happens when the same first name maps to multiple people in the roster (e.g., two "Dan" entries)? The roster must distinguish them via variants or context; if ambiguous, the name is flagged for user review.
- What happens when a garbled variant listed in the roster matches a real word in the transcript that is not a name (e.g., "leaser" as a variant for "Eleazar" but also a legitimate word)? Matching should only occur in speaker-label positions, not in general transcript content.
- What happens when the transcript contains names of people who are discussed but not present (e.g., "Florian mentioned this last week")? These names are not in the attendee list or roster. The skill should still attempt roster-based resolution and flag unresolved names in the post-hoc step.
- What happens when a roster file has invalid or malformed content? The skill reports a clear error identifying the roster file and the parsing issue, then proceeds without roster-based resolution.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The skill MUST support a roster file format that maps full names to known first-name variants and garbled transcription variants.
- **FR-002**: The skill MUST search for a roster file in the transcript directory, then walk up parent directories until one is found or the filesystem root is reached.
- **FR-003**: The skill MUST resolve "You" speaker labels to the transcript owner's full name when a transcript owner is declared.
- **FR-004**: The skill MUST use full names on first mention of each person within each major section of the summary, then may use first names for subsequent references in that section.
- **FR-005**: The skill MUST present a list of all detected names after summary generation, indicating which were resolved via roster and which remain unresolved.
- **FR-006**: The skill MUST accept user corrections for unresolved or incorrectly resolved names and apply them to the generated summary.
- **FR-007**: The skill MUST offer to persist user-provided name corrections back to the roster file.
- **FR-008**: The skill MUST match roster variants only against speaker labels and speaker-attributed name positions, not against general transcript content.
- **FR-009**: The skill MUST gracefully handle missing, malformed, or empty roster files by reporting the issue and continuing without roster-based resolution.
- **FR-010**: The transcript owner MUST be declarable in either the roster file or the transcript metadata.

### Key Entities

- **Roster**: A file containing person entries, each with a full name, a list of known name variants (including garbled transcription forms), and optional metadata such as role or team. Also optionally declares the transcript owner.
- **Person Entry**: A single record within a roster mapping one individual's full name to their known variants.
- **Transcript Owner**: The identity of the person whose speech appears as "You" in closed-caption transcripts.
- **Name Reference**: Any occurrence of a person's name in a transcript, whether as a speaker label or an in-text mention.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Summaries generated with a populated roster file contain zero first-name-only references for rostered people on their first mention per section.
- **SC-002**: All "You" speaker labels are resolved to the transcript owner's full name when a transcript owner is declared, with zero residual "You" references in the output.
- **SC-003**: 100% of names detected in a transcript appear in the post-summary name review list.
- **SC-004**: User-provided corrections are applied to the summary within the same skill session — no manual re-editing required.
- **SC-005**: Roster file discovery completes without user-perceptible delay.

## Assumptions

- Users primarily work with Google Meet closed-caption transcripts and Gemini-generated transcripts, which are the formats already supported by the skill.
- The roster file is a user-maintained artifact; the skill does not auto-generate rosters from external sources (calendar, participant lists) in this version.
- Garbled variant matching is explicit (exact match against listed variants), not fuzzy. Fuzzy matching is deferred to avoid false positives.
- The skill already has a working summary generation pipeline; this feature adds a name-resolution layer before and after that pipeline.
- The post-hoc correction flow is interactive and requires user presence; fully automated (non-interactive) runs skip the post-hoc step.
- Role and team metadata in roster entries are informational context for the user; the skill does not use them for disambiguation logic in this version.
