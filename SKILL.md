---
name: summarize-transcript
description: Create executive summary and detailed timeline from meeting transcripts. Use for Google Meet transcriptions or similar raw meeting records.
argument-hint: <transcript-file-path> [output-file-path]
disable-model-invocation: true
allowed-tools: Read, Write, Agent
---

# Summarize Meeting Transcript

Create an executive summary and detailed timeline from the meeting transcript at `$0`.

The output contains two complementary sections:
- **Summary** — A concise, topically-organized executive summary a reader can consume in 2-3 minutes
- **Timeline** — A chronological, verbatim-style record preserving the full texture of the discussion

Output file: `$1` (defaults to `summary.md` in the same directory as the transcript).

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

## Process

### 1. Read the Transcript and Prepare Reference Material

Read `$0` to understand:
- Who the participants are
- What the meeting was about
- The overall flow and any confusion
- Transcription quality issues (accents, technical terms, etc.)

Also attempt to read `~/source/handbook/The Ansible Engineering Handbook/Glossary/glossary.md`. If available, use it as a reference for correcting technical terms throughout the summary. If unavailable, proceed without it.

While reading, mentally apply the transcription artifact cleanup rules (merged fillers, spurious spaces, compound term spacing) to understand what was actually said.

### 2. Create Initial Summary

Create the summary file with:

**Header Structure:**
```markdown
# [Meeting Title]
## Meeting Date: [Date]

### Attendees
[List of attendees]

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

**Note: Time ranges are approximate based on [sparse/available] timestamps in transcript**
```

**Timeline Sections:**

Use format: `### [Editorial: Descriptive Topic] (HH:MM:SS - HH:MM:SS)`

For each speaker statement:
- Use bold for speaker name: `**Name**`
- Include full context and nuance
- Preserve rambling speech with ellipses for trailing thoughts
- Add `(likely "word")` for unclear transcription
- Add `(unclear term)` for unintelligible parts
- Use quotes for direct quotes
- Add clarifying parentheticals when needed: `(Note: ...)`

**Footer Structure:**
```markdown
---

## Document Status and Limitations

**Source Accuracy:**
- All content is based solely on statements in the transcript
- No fabricated content has been added
- Direct quotes are preserved where possible, including uncertain speech and incomplete thoughts

**Known Issues:**
- [List transcription quality issues]
- [Note accent-related misinterpretations]
- [Note sparse timestamps or other limitations]

**Editorial Choices Made:**
- Section headers in brackets [Editorial: ...] are organizational additions not stated by participants
- Time ranges are approximate based on [available data]
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

Use the Agent tool to launch a general-purpose agent to verify the summary:

```
Verify the accuracy and impartiality of the summary file against the original transcript.

Compare both files and check:

**Executive Summary section:**
1. Contains only information clearly stated in the transcript
2. No added context, assumptions, or interpretations beyond what was discussed
3. Key decisions, conflicts, and action items are accurately represented
4. Speaker positions in disagreements are correctly attributed
5. No significant topics omitted
6. Direct quotes (if any) are accurate

**Timeline section:**
7. Timeline accurately reflects the order of discussion
8. No misrepresentations or bias in how information is presented
9. No significant omissions
10. Speaker attributions are correct
11. Uncertainty, confusion, and rambling nature are preserved
12. Strong language is not softened
13. Editorial additions are properly marked

Provide a detailed verification report with:
- Overall assessment of accuracy and impartiality
- Specific instances where summary adds context not in transcript (if any)
- Specific instances of inaccuracies or misattributions (if any)
- Notable omissions (if any)
- Recommendations for corrections (if needed)

Be thorough and critical. The goal is to ensure the summary is factual and doesn't add interpretations or assumptions.
```

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
