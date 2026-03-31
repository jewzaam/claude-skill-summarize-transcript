---
name: summarize-transcript
description: Create executive summary and detailed timeline from meeting transcripts. Handles Gemini transcripts, Google Meet closed captions, and chat logs — individually or combined.
argument-hint: <transcript-file-directory-or-google-doc-url> [output-file-path]
disable-model-invocation: true
allowed-tools: Read, Write, Agent, Glob, Bash
---

# Summarize Meeting Transcript

Create an executive summary and detailed timeline from meeting transcript(s) at `$0`.

## Input Modes

`$0` can be a single file, a directory, or a Google Docs URL:

- **Single file** — Summarize that one transcript
- **Directory** — Scan for all recognized transcript files and combine them into a single unified summary (see Knitting Multiple Sources below)
- **Google Doc URL** — When `$0` is a Google Docs URL (matches `docs.google.com/document/d/<ID>`), fetch the document, convert each tab to markdown, and write files to a working directory. Then proceed as a directory input. See the Google Doc Fetch & Convert section below.

To detect the input mode: first check if `$0` matches a Google Docs URL pattern (`docs.google.com/document/d/`). If so, use the Google Doc Fetch & Convert flow below. Otherwise, attempt to Read it. If Read fails (directories can't be read as files), treat it as a directory and use Glob to find all files in it. Classify each by filename (see Transcript Source Detection table). Process all recognized files together.

If a directory contains no recognized transcript or metadata files, inform the user and stop — there's nothing to summarize.

## Transcript Source Detection

Determine the transcript source type from each input filename. Match on the base name regardless of extension (`.md`, `.txt`, or other):

| Base Filename | Source Type | Notes |
|---|---|---|
| `closed-caption` | Google Meet Closed Captions | Scraped from DOM; lower quality (see below) |
| `chat` | Meeting Chat Log | Text chat, not spoken word |
| `metadata` (`.json`) | Meeting Metadata | Title, date, start time, meeting URL (see below) |
| `gemini-notes` | Gemini Meeting Notes | Supplementary; provides attendees, title, attachments, Gemini summary/details (see below) |
| Anything else | Gemini Transcript | Higher quality, includes timestamps and speaker identification |

### Meeting Metadata

When a `metadata.json` file is present in the directory, read it to populate the summary header. This avoids guessing meeting title and date from transcript content. Expected format:

```json
{
  "meetingTitle": "Meeting Name",
  "date": "YYYY-MM-DD",
  "startTime": "ISO-8601 timestamp",
  "meetingUrl": "https://meet.google.com/...",
  "meetingCode": "xxx-xxxx-xxx"
}
```

- Use `meetingTitle` as the `# [Meeting Title]` header
- Use `date` as the `## Meeting Date:` value
- Use `startTime` to establish the meeting's timezone context — compare it against timestamps in other sources to determine UTC offsets for chronological alignment
- `meetingUrl` and `meetingCode` can be included in the summary header if present
- Metadata is never the primary source — it supplements whatever transcript files are present

### Gemini Meeting Notes

When `gemini-notes.md` is present (from a Google Doc Notes tab or manually created), it provides meeting metadata and supplementary context.

**Metadata extraction:**
- Meeting title: from the first `HEADING_2` (`##`) in the notes
- Attendees: from the "Invited" section listing participants. Strikethrough names (`~~@Name~~`) indicate removed invitees. Role annotations (e.g., presenting, host) are preserved if present.
- Attachments: from "Attachments" and "Meeting records" sections containing links — these become links in the summary header

**Supplementary content:**
- Gemini's "Summary" and "Details" sections are available for cross-referencing against the transcript
- They are NOT echoed verbatim into the summary — the summary is built from the transcript
- They can help resolve ambiguous speaker attributions or fill gaps where the transcript is unclear

**Relationship to metadata.json:**
- `metadata.json` remains the metadata source for closed-caption workflows
- `gemini-notes` is the metadata source for Google Meet Gemini workflows
- They serve the same role (providing meeting context) but for different source pipelines
- If both are present (unlikely), `gemini-notes` takes precedence for conflicting fields

### Closed Caption and Chat Log Formats

See `references/source-formats.md` for detailed format specifications for closed caption files and chat logs, including quality limitations and timestamp handling.

### Knitting Multiple Sources

When multiple transcript sources are available from the same meeting (e.g., a directory containing both `closed-caption.md` and `chat.md`, or a Gemini transcript alongside a `chat.md`), they should be combined into a single unified summary.

**Knitting Rules:**

1. **Identify the primary source** — Use the highest-quality transcript as the backbone:
   - Gemini transcript (best) > Closed captions > Chat-only
   - When Gemini notes are present alongside a Gemini transcript, the transcript is primary and notes are supplementary (metadata + cross-reference only).
   - If both a Gemini transcript and closed captions exist, prefer the Gemini transcript for the spoken-word timeline. Use closed captions only to fill gaps or resolve ambiguities.
   - When all three are present (Gemini + CC + chat): use Gemini as primary, ignore CC unless it fills gaps in the Gemini transcript, and interleave chat entries into the timeline.

2. **Integrate chat log entries** — Chat messages provide context that complements the spoken transcript:
   - Interleave chat entries into the timeline at their approximate chronological position
   - Mark chat entries with a `[Chat]` prefix: `**[Chat] Speaker Name:** message text`
   - Chat messages often contain links, code snippets, or clarifications that speakers reference verbally — connect these when the reference is clear
   - Do not duplicate information: if a chat message restates something already captured in the spoken transcript, omit it unless the chat version adds detail (e.g., a URL)
   - Chat-sourced action items (links shared for follow-up, explicit assignments) belong in the Summary's Action Items list — attribute them as coming from chat

3. **Chronological alignment strategy** — Placing chat entries in the timeline requires matching them to the spoken discussion:
   - **Gemini + chat** (both have timestamps): align by time — place each chat message in the timeline section whose time range contains it
   - **CC + chat** (chat has UTC timestamps, CC has no per-entry timestamps): use chat timestamps to establish rough chronological anchors. Since CC entries are only sequentially ordered, place chat messages by topical context — insert them near the CC discussion they relate to. If `metadata.json` provides `startTime` (UTC) and CC attendee times are in local time, compute the UTC offset to help correlate timelines.
   - **CC only** (no chat): topical context is the only ordering mechanism
   - When placement is ambiguous, group the chat message at the end of the most relevant `[Editorial]` section with a note: `(Note: exact timing relative to discussion uncertain)`

4. **Cross-reference attendees** — Merge attendee lists from all sources. The chat log may reveal participants who never spoke (and thus don't appear in captions or transcript).

5. **Source attribution in output** — The summary header should note all source files used:
   ```markdown
   ### Sources
   - `transcript.md` — Gemini transcript (primary)
   - `gemini-notes.md` — Gemini meeting notes (supplementary — metadata)
   - `chat.md` — Meeting chat log (supplementary)
   ```
   List each file with its detected source type and whether it was primary or supplementary.

6. **Conflict resolution** — If sources disagree (e.g., different speaker attribution):
   - Prefer the higher-quality source
   - Note the discrepancy in the Known Issues section
   - Do not silently pick one version

**Knitted Timeline Example:**

```markdown
### [Editorial: Architecture Discussion] (00:12:30 - 00:18:45)

**Alice** explained the current architecture uses a monolithic deployment...
she noted "we've been talking about splitting this for months."

**[Chat] Bob:** https://github.com/org/repo/issues/42 — "this is the tracking issue for the split"

**Alice** acknowledged the link (Note: see chat message above) and said they should
start with the API layer...

**[Chat] Carol:** +1, API layer first makes sense. I can take the auth service piece.
```

**Chat Log Format:** See `references/source-formats.md` for chat log format details and timestamp handling.

### Chat-Only Summary Rules

When the only input is a chat log (no spoken-word transcript), the summary format adapts because chat is structured text, not speech:

- **No "rambling" to preserve** — Chat messages are composed text, not real-time speech. Don't apply the spoken-word preservation rules (trailing thoughts, run-on sentences, etc.)
- **Timeline is message-based** — The timeline is a chronological log of messages grouped by `[Editorial]` topic sections, not a verbatim record of speech
- **Timestamps are the primary ordering** — Use chat timestamps directly rather than inferring order
- **Attribution is reliable** — Chat identifies senders precisely, unlike speech-to-text which can misattribute
- **The Summary section still applies** — Organize by topic with bold lead-ins, cover decisions and action items, same executive summary format
- **Links and code snippets are first-class content** — These are often the most valuable parts of a chat log; preserve them fully in the timeline

The output contains two complementary sections:
- **Summary** — A concise, topically-organized executive summary a reader can consume in 2-3 minutes
- **Timeline** — A chronological, verbatim-style record preserving the full texture of the discussion

Output file: `$1` (defaults to `summary.md` in the same directory as the transcript).

## YAML Frontmatter

Every summary file MUST begin with a YAML frontmatter block. This structured metadata enables indexing, search, and cross-meeting queries. The frontmatter is extracted from the transcript content during summarization — do not fabricate any fields.

**Format:**

```yaml
---
title: "Meeting Title"
date: "YYYY-MM-DD"
meeting_code: "xxx-xxxx-xxx"
attendees:
  - "Alice Smith"
  - "Bob Jones"
topics:
  - "API migration progress"
  - "Release blocker discussion"
decisions:
  - "Defer GCP migration to Q3"
  - "Use new auth middleware for all services"
action_items:
  - owner: "Alice Smith"
    action: "Update SDP with new timeline"
  - owner: "Bob Jones"
    action: "File OCPBUGS ticket for auth regression"
references:
  - type: jira
    value: "OCPBUGS-1234"
  - type: jira
    value: "CNTRLPLANE-99"
  - type: anstrat
    value: "ANSTRAT-42"
keywords:
  - "hypershift"
  - "GCP"
  - "auth middleware"
---
```

**Field rules:**

| Field | Required | Source | Notes |
|---|---|---|---|
| `title` | Yes | `metadata.json`, `gemini-notes`, or inferred | Meeting title |
| `date` | Yes | `metadata.json` date field or directory name | YYYY-MM-DD format |
| `meeting_code` | No | `metadata.json` meetingCode field | Omit if unavailable |
| `attendees` | Yes | All sources merged | Full names, no role annotations (those go in the body) |
| `topics` | Yes | Extracted from summary | 3-8 short phrases capturing the major discussion areas |
| `decisions` | No | Extracted from summary | Concrete decisions made — omit if none |
| `action_items` | No | Extracted from summary | Each has `owner` and `action` — omit if none identified |
| `references` | No | Pattern-matched from all sources | Jira tickets (`PROJ-NNN`), ANSTRAT references, notable URLs shared in chat — omit if none found |
| `keywords` | Yes | Extracted from summary | 3-10 significant technical terms, project names, or domain concepts discussed — not generic words |

**Reference detection patterns:**
- **Jira tickets:** Match `[A-Z][A-Z0-9]+-\d+` patterns (e.g., `OCPBUGS-1234`, `CNTRLPLANE-99`, `MGMT-456`). Use `type: jira`.
- **ANSTRAT references:** Match `ANSTRAT-\d+` or references to specific ANSTRATs by number. Use `type: anstrat`.
- **Notable URLs:** URLs shared in chat or verbally spelled out that represent action items, tracking issues, or reference material. Do NOT include meeting URLs, calendar links, or attachment URLs already in the Sources section. Use `type: url`.

**Keyword extraction guidance:**
- Include project names (e.g., "HyperShift", "ROSA"), technology terms, team names, and domain concepts
- Exclude generic words like "meeting", "discussion", "team", "update"
- Prefer the canonical form of terms (e.g., "HyperShift" not "hypershift")

The frontmatter block must be the very first content in the file, before the `# [Meeting Title]` header.

## CRITICAL: Accuracy and Impartiality Requirements

These rules apply to **both** sections (Summary and Timeline):

1. **Do NOT add context or assumptions** — State only what is clearly present in the transcript
2. **Keep strong language** — Don't soften phrases (e.g., keep "power sucks" not "problematic")
3. **Accurate attribution** — Be careful about who said what, add clarifying notes when needed
4. **No interpretive conclusions** — Do not add conclusions or interpretations not stated by participants

### Summary Section Rules

The Summary is a traditional executive summary — concise, topically organized, and readable in 2-3 minutes.

- Organize by topic, not chronology
- Use short paragraphs with **bold lead-ins** for each major topic
- Cover: key decisions, open conflicts/disagreements, action items, and important context
- Attribute positions to speakers where it matters (e.g., disagreements)
- Use direct quotes sparingly — only for particularly notable or strong statements
- End with an **Action Items Discussed** list
- Condensing and organizing is expected — but do not add meaning that wasn't stated

### Timeline Section Rules

The Timeline is a verbatim-style chronological record that preserves the full texture of the discussion.

- **Preserve uncertainty** — Include phrases like "I don't know", "maybe", incomplete thoughts, trailing dialogue
- **Preserve confusion** — Meeting confusion, scheduling issues, participants being lost are important context
- **Preserve rambling nature** — Don't over-organize rambling explanations into clean bullet points
- **Mark editorial additions** — Use `[Editorial: ...]` format for section headers or organizational structure you add
- **Include process issues** — Meeting logistics, confusion about agenda, people joining late, etc.

**What to include in the Timeline:**

- All significant statements and exchanges in chronological order
- Uncertainty markers and incomplete thoughts
- Strong language and self-aware statements (e.g., "politically incorrect answer")
- Meeting confusion and scheduling issues
- Rambling speech patterns (run-on sentences are OK)
- Statements showing working session nature

**What to avoid in the Timeline:**

- Over-organizing rambling content into clean structures
- Smoothing over uncertainty or disagreement
- Softening strong language
- Creating impression of more structure than existed
- Clean bullet points that misrepresent exploratory discussion

### Transcription Artifact Cleanup

Before summarizing, apply these corrections to fix common speech-to-text garbling patterns:

1. **Conjunction + filler merged** — Transcription often merges conjunctions/words with filler sounds:
   - "You'veum" → "You've um"
   - "Butum" → "But um"
   - "Soum" → "So um"
   - Pattern: word immediately followed by "um", "uh", "ah" with no space

2. **Spurious spaces within words** — Extra spaces inserted mid-word:
   - "Fo lks" → "Folks"
   - "H elen" → "Helen"
   - "E vent" → "Event"
   - Pattern: single capital or lowercase letter separated from the rest of the word by a space

3. **Spurious spaces around punctuation** — Extra spaces around hyphens and compound terms:
   - "E vent - driven" → "Event-driven"
   - "re - deploy" → "re-deploy"
   - Pattern: spaces around hyphens in compound words

These corrections should be applied silently (no bracket notation needed) since they are fixing transcription engine errors, not changing what was said. If a correction is ambiguous, preserve the original and add `(likely "corrected form")`.

### Technical Term Correction via Glossary

If `~/source/handbook/The Ansible Engineering Handbook/Glossary/glossary.md` exists, read it and use it to correct mistranscribed technical terms:

- Match glossary terms, acronyms, and their AKAs against the transcript
- Fix phonetic mistranscriptions (e.g., "PDTs" when "PDT" is the correct acronym, "SDPs" vs the actual term)
- Apply corrections silently when the glossary match is unambiguous
- Use `(likely "[glossary term]")` when the match is plausible but uncertain
- Do NOT add glossary definitions or context — only use it for spelling/term correction

If the glossary file is not available, skip this step without error.

### Handling Post-Summary Corrections

**CRITICAL:** If corrections are made after the summary is created (e.g., user corrects transcription errors):

1. **In Quotes:** Use square brackets `[corrected word]` to indicate the correction
   - Example: `"I know [Naveen] is out"` when transcript said "Lavin"
   - Example: `"lack of [subject matter] expertise"` when transcript said "sudden mother"

2. **Outside Quotes:** Square brackets are strongly recommended for transparency
   - Example: `**Massimo** knows [Naveen] is out.`
   - Shows the correction was made post-summary creation

3. **Document in Known Issues:** Add note about what was corrected
   - Example: `Transcription misheard "Naveen" as "Lavin" (corrected with [brackets])`

**Rationale:** This maintains transparency and doesn't mislead readers about what was in the original transcript versus what was corrected. Without brackets, corrections in quotes violate the principle of not providing interpretation.

**Note:** Square brackets in Markdown don't require escaping - `[text]` renders as literal brackets since there's no `(url)` following it.

## Google Doc Fetch & Convert

When `$0` is a Google Docs URL, follow the procedure in `references/google-doc-fetch.md` to fetch the document, convert tabs to markdown files, and write them to a working directory. Then proceed to the Process steps below with `$0` set to that directory.

## Process

### 1. Read the Transcript(s), Detect Source Types, and Prepare Reference Material

**If `$0` was a Google Doc URL:** The Google Doc Fetch & Convert section above has already converted the document tabs to markdown files in a working directory. `$0` is now that directory. Proceed with directory-mode detection below.

**Determine input mode:**
- If `$0` is a directory, use Glob to find files in it (e.g., `*.md`, `*.txt`, `*.json`). Classify each by base filename (see Transcript Source Detection table). Identify the primary source (highest quality transcript) and supplementary sources.
- If `$0` is a single file, classify it by filename as before.

**Read metadata first** — If `metadata.json` is present, read it before the transcripts. This establishes the meeting title, date, and timezone context needed for chronological alignment.

**Read all transcript files** to understand:
- Who the participants are (merge attendee lists across all sources)
- What the meeting was about
- The overall flow and any confusion
- Transcription quality issues (accents, technical terms, etc.)
- How chat log entries relate to the spoken discussion (if chat is present)

**For closed captions specifically:**
- Ignore DOM index numbers in the input — they are scraping artifacts
- Note that the participant list may be incomplete
- Watch for out-of-order or interleaved speaker text caused by caption processing delays
- Expect lower speech-to-text fidelity than Gemini transcripts

**When multiple sources are present:**
- Read the primary transcript first to establish the narrative arc
- Then read supplementary sources (chat, secondary transcript) and note where they add context
- Build a mental map of how chat messages align chronologically with the spoken discussion

Also attempt to read `~/source/handbook/The Ansible Engineering Handbook/Glossary/glossary.md`. If available, use it as a reference for correcting technical terms throughout the summary. If unavailable, proceed without it.

While reading, mentally apply the transcription artifact cleanup rules (merged fillers, spurious spaces, compound term spacing) to understand what was actually said.

### 2. Create Initial Summary

Create the summary file with:

**Header Structure:**

Use `metadata.json` values for title and date when available. Otherwise infer from transcript content. The file begins with YAML frontmatter (see YAML Frontmatter section above), followed by the markdown header:

```markdown
---
title: "Meeting Title"
date: "YYYY-MM-DD"
[... full frontmatter as specified above ...]
---

# [Meeting Title from metadata or inferred]
## Meeting Date: [Date from metadata or inferred]

### Attendees
[List of attendees — from CC attendee header, transcript speakers, and/or chat senders]
[Include role annotations from CC if available: (presenting), (host)]

---
```

Attendee notes (add whichever apply):
- For closed captions: add `**Note:** This list may be incomplete. Closed captions only capture speakers whose names appeared in the caption stream during recording.`
- For multi-source: add `Attendee list compiled from all available sources (transcript, chat, closed captions).`

When multiple source files are used, add a `### Sources` section after Attendees listing each file, its type, and whether it's primary or supplementary:
```markdown
### Sources
- `transcript.md` — Gemini transcript (primary)
- `chat.md` — Meeting chat log (supplementary)
```
Omit this section for single-file input.

```markdown
---

## Summary

[A concise but thorough executive summary of the meeting. This is NOT a timeline — it is a
traditional summary that a reader can consume in 2-3 minutes to understand what happened
without reading the full timeline. Organize by topic, not chronology.]

[Cover: key decisions, open conflicts/disagreements, action items, and important context.
Use short paragraphs with bold lead-ins for each major topic. Include an "Action Items
Discussed" list at the end.]

[IMPORTANT: The summary must still only contain information stated in the transcript.
Do not add interpretation or conclusions not present in the discussion. Attribute positions
to speakers where it matters (e.g., disagreements). Use direct quotes sparingly — only
for particularly notable or strong statements.]

---

## IMPORTANT NOTES ABOUT THIS SUMMARY

**Nature of the Meeting:**
[Describe if it was exploratory, structured, confused, etc.]

**Editorial Choices:**
- Section headers (e.g., "[Editorial: Topic]") are organizational additions - NOT explicit topics identified by participants
- The original transcript contains [describe quality issues]
- This summary preserves [what you're preserving]
- No content has been fabricated, but organizational structure has been added for readability

---

## Timeline of Discussion

**For Gemini transcripts:** "Note: Time ranges are approximate based on [sparse/available] timestamps in transcript"
**For closed captions:** "Note: No timestamps are available. Sections reflect the sequential order of captions as captured. Due to closed caption processing delays, the order may not perfectly reflect the actual conversation flow."
```

**Timeline Sections:**

For Gemini transcripts, use format: `### [Editorial: Descriptive Topic] (HH:MM:SS - HH:MM:SS)`
For closed captions (no timestamps available), use format: `### [Editorial: Descriptive Topic]`

For each speaker statement:
- Use bold for speaker name: `**Name**`
- Include full context and nuance
- Preserve rambling speech with ellipses for trailing thoughts
- Add `(likely "word")` for unclear transcription
- Add `(unclear term)` for unintelligible parts
- Use quotes for direct quotes
- Add clarifying parentheticals when needed: `(Note: ...)`

For chat log entries interleaved into the timeline:
- Use `**[Chat] Name:**` prefix to distinguish from spoken word
- Place at the approximate chronological position relative to the spoken discussion
- Include the full message text, preserving any URLs or code snippets
- If a speaker verbally references a chat message, add a `(Note: see chat message above/below)` parenthetical

**Footer Structure:**
```markdown
---

## Document Status and Limitations

**Source Accuracy:**
- All content is based solely on statements in the transcript
- No fabricated content has been added
- Direct quotes are preserved where possible, including uncertain speech and incomplete thoughts
- [For closed captions:] Source is Google Meet closed captions scraped from the DOM, which are lower fidelity than Gemini transcripts

**Known Issues:**
- [List transcription quality issues]
- [Note accent-related misinterpretations]
- [For Gemini: Note sparse timestamps or other limitations]
- [For closed captions: Note lack of timestamps, potential out-of-order text, incomplete participant list, and DOM index artifacts that were ignored]
- [For multi-source: Note any conflicts between sources and how they were resolved]
- [For multi-source: Note any chat messages that could not be reliably placed chronologically]

**Editorial Choices Made:**
- Section headers in brackets [Editorial: ...] are organizational additions not stated by participants
- [For Gemini:] Time ranges are approximate based on [available data]
- [For closed captions:] No timestamps available; section ordering reflects caption sequence which may not perfectly match conversation flow
- Rambling speech patterns preserved where they occurred
- Uncertainty markers ("I don't know", "maybe", incomplete thoughts) deliberately preserved
- Strong language preserved (e.g., [examples])
- Meeting confusion and process issues included

**What This Document Is:**
- An executive summary distilling the key topics, decisions, and action items
- A timeline capturing what was said with minimal interpretation
- An attempt to preserve the exploratory, uncertain nature of the working session
- A verbatim-style timeline acknowledging transcription quality issues

**What This Document Is Not:**
- A definitive technical specification or decision record
- Free from potential misinterpretations due to transcript errors

---

*Generated By: Claude Code (<model>)*
```

### 3. Verify Accuracy and Impartiality

Use the Agent tool to launch a general-purpose agent to verify the summary. Pass the paths to all source files and the summary file. Read `agents/verifier.md` and include its contents as the agent's instructions.

### 4. Review and Revise

If the verification agent identifies issues:
1. Read the verification report carefully
2. Revise the summary to address all identified issues
3. Consider whether a second verification pass is needed

### 5. Present Results

Show the user:
- Summary file location
- Key findings from verification
- Any remaining concerns or limitations
- Offer to revise if they identify transcription corrections or other issues

## Additional Notes

- Transcription quality varies — watch for accent-related errors and technical term misinterpretations
- The Summary section should be concise and organized; the Timeline section should be verbatim-style and preserve full texture
- When in doubt, preserve more detail rather than less in the Timeline
- Use the Agent tool with a general-purpose subagent for verification to get thorough review
- Closed captions are a fundamentally lower-quality source than Gemini transcripts — the output should clearly communicate this to readers so they calibrate their trust accordingly
- Source type detection is filename-based and will evolve as new transcript sources are added
- When knitting multiple sources, chat messages add valuable context (links, clarifications) that spoken transcripts miss — prioritize including them where they clearly relate to the discussion
- A directory input with only a single file should behave identically to passing that file directly
