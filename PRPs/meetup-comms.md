# PRP: Communications and Scheduling System

## Context

MeetupBot currently generates event plan JSON via `/plan-event`. This PRP adds three new slash commands — `/build-schedule`, `/draft-announcement`, and `/draft-speaker-outreach` — that consume that event plan JSON to produce schedules, announcements, and speaker outreach emails. All three commands extend the existing slash-command pattern established by `plan-event.md`.

### Relevant Files

- `CLAUDE.md` — agent personality, JSON schema, naming conventions, tone rules
- `.claude/commands/plan-event.md` — existing slash command pattern to follow (uses `$ARGUMENTS`, injects data via `!cat`, includes Known Gotchas)
- `examples/sample-event-plan.json` — reference event plan shape that all new commands consume as input
- `data/speakers.json` — speaker names, topics, bios, availability (used by schedule + outreach)
- `data/venues.json` — venue capacity, amenities, cost (used by schedule)
- `data/past-events/.gitkeep` — past events directory (empty, but referenced for future use)

### Key Constraints from CLAUDE.md

- All event data MUST be structured JSON; freeform text only for explanations
- JSON keys use camelCase — no exceptions
- File naming: kebab-case for files, `{topic}-{format}-{YYYY-MM}.json` for past events
- Agent tone: friendly, organized, opinionated, direct — lead with the recommendation
- Event plan schema fields: `name`, `description`, `format`, `targetAudience`, `estimatedAttendees`, `suggestedDuration`, `topicTags`, `prerequisites`

## Implementation Plan

### Command 1: `/build-schedule`

**File**: `.claude/commands/build-schedule.md`
**Purpose**: Generate a time-slot schedule from an event plan JSON, assigning speakers to slots with breaks.
**Input**: `$ARGUMENTS` = path to an event plan JSON file (e.g., `events/claude-code-workshop.json`)

**Steps**:
1. Read the event plan JSON from the path provided in `$ARGUMENTS`
2. Inject `data/speakers.json` and `data/venues.json` via `!cat` (same pattern as `plan-event.md`)
3. Parse `suggestedDuration` from the event plan to determine total available time
4. Allocate time slots:
   - Each talk/session: 30-45 minutes (depending on format)
   - Breaks: 15-minute break every 90 minutes
   - Setup time: 15 minutes at start
   - Teardown/networking: 15 minutes at end
5. Match speakers from `data/speakers.json` whose `topics` align with the event's `topicTags`
6. Validate: no speaker appears in overlapping time slots (double-booking check)
7. Validate: speaker `availability` field is compatible with the event timing (weekday-evenings, weekends, flexible)
8. Output a **markdown table** showing the schedule (time, session, speaker, duration)
9. Output a **JSON file** to `schedules/{kebab-case-event-name}-schedule.json`

**Output files**:
- `schedules/{kebab-case-event-name}-schedule.json`

**Schedule JSON schema**:
```json
{
  "eventName": "string",
  "totalDuration": "string",
  "slots": [
    {
      "startTime": "string — relative offset, e.g., '0:00', '0:45'",
      "endTime": "string",
      "type": "setup | talk | workshop | break | networking",
      "title": "string",
      "speaker": "string | null",
      "duration": "string"
    }
  ]
}
```

**Known Gotchas** (to include in command file):
- Speaker suggestions MUST reference actual entries from `data/speakers.json` by name
- No speaker may appear in overlapping time slots
- Break every 90 minutes is mandatory — don't skip it for short events
- `$ARGUMENTS` must point to a valid event plan JSON file

### Command 2: `/draft-announcement`

**File**: `.claude/commands/draft-announcement.md`
**Purpose**: Draft an event announcement email tailored to the target audience from the event plan.
**Input**: `$ARGUMENTS` = path to an event plan JSON file

**Steps**:
1. Read the event plan JSON from the path provided in `$ARGUMENTS`
2. Inject `data/speakers.json` and `data/venues.json` via `!cat` for speaker highlights and venue details
3. Draft an announcement email that includes:
   - Event name (from `name`)
   - Date placeholder: `[EVENT DATE]`
   - Venue name and address (recommend the best-fit venue from `data/venues.json` based on `estimatedAttendees`)
   - Speaker highlights: 2-3 sentences about featured speakers, referencing their bios
   - Target audience hook: opening line tailored to `targetAudience`
   - What attendees will learn/do (derived from `description` and `topicTags`)
   - Prerequisites (from `prerequisites`, if any)
   - RSVP link placeholder: `[RSVP LINK]`
4. Match the tone from CLAUDE.md: friendly, direct, opinionated
5. Write the email to `comms/{kebab-case-event-name}-announcement.md`

**Output files**:
- `comms/{kebab-case-event-name}-announcement.md`

**Known Gotchas** (to include in command file):
- Speaker names MUST match actual entries from `data/speakers.json`
- Venue MUST be selected based on `estimatedAttendees` with ~20% capacity headroom
- Use placeholders for date and RSVP link — don't invent them
- Tone must match CLAUDE.md: friendly, organized, opinionated — not generic marketing copy

### Command 3: `/draft-speaker-outreach`

**File**: `.claude/commands/draft-speaker-outreach.md`
**Purpose**: Draft personalized outreach emails to speakers, referencing their topics and asking them to present.
**Input**: `$ARGUMENTS` = path to an event plan JSON file

**Steps**:
1. Read the event plan JSON from the path provided in `$ARGUMENTS`
2. Inject `data/speakers.json` via `!cat`
3. Identify speakers from `data/speakers.json` whose `topics` overlap with the event's `topicTags`
4. For each matched speaker, draft a personalized email that includes:
   - Greeting using their name
   - Reference to their specific topics/expertise (from `topics` and `bio`)
   - Brief event description (from `name` and `description`)
   - Why they'd be a great fit (connect their topics to the event)
   - Logistics: suggested duration, format, estimated audience size
   - Clear ask: "Would you be interested in presenting?"
   - Date placeholder: `[EVENT DATE]`
5. Write all outreach emails to a single file: `comms/{kebab-case-event-name}-speaker-outreach.md`
   - Separate each email with a markdown heading (`## Outreach: {Speaker Name}`)

**Output files**:
- `comms/{kebab-case-event-name}-speaker-outreach.md`

**Known Gotchas** (to include in command file):
- Each email MUST reference the speaker's actual `topics` and `bio` from `data/speakers.json`
- Never invent speaker details — only use what's in the data
- Note speaker `availability` constraints in the logistics section
- Tone: professional but warm, matching CLAUDE.md personality

## Directory Structure (New)

```
meetup-agent/
├── .claude/commands/
│   ├── plan-event.md              (existing)
│   ├── generate-prp.md            (existing)
│   ├── build-schedule.md          (NEW)
│   ├── draft-announcement.md      (NEW)
│   └── draft-speaker-outreach.md  (NEW)
├── schedules/                     (NEW — created on first /build-schedule run)
├── comms/                         (existing, currently empty)
└── PRPs/
    └── meetup-comms.md            (this PRP)
```

## Validation Gates

### Gate 1: Files Exist and Follow Patterns
- [ ] `.claude/commands/build-schedule.md` exists
- [ ] `.claude/commands/draft-announcement.md` exists
- [ ] `.claude/commands/draft-speaker-outreach.md` exists
- [ ] Each command file uses `$ARGUMENTS` for the event plan file path
- [ ] Each command file injects data files via `!cat` (same pattern as `plan-event.md`)
- [ ] Each command file has a Known Gotchas section

### Gate 2: Commands Produce Expected Output
- [ ] `/build-schedule events/hands-on-claude-code-workshop.json` writes a valid JSON schedule to `schedules/`
- [ ] `/build-schedule` output includes a markdown table of the schedule
- [ ] `/draft-announcement events/hands-on-claude-code-workshop.json` writes an email to `comms/`
- [ ] `/draft-speaker-outreach events/hands-on-claude-code-workshop.json` writes outreach emails to `comms/`

### Gate 3: Constraints Are Enforced
- [ ] Schedule has no double-booked speakers
- [ ] Schedule includes breaks every 90 minutes
- [ ] Schedule includes setup and teardown time
- [ ] Schedule JSON uses camelCase keys
- [ ] Announcement references a real venue with capacity >= `estimatedAttendees` * 1.2
- [ ] Announcement includes all required elements (name, venue, speakers, audience hook, RSVP placeholder)
- [ ] Speaker outreach references actual speaker data (names, topics, bios) — nothing invented
- [ ] All output file names are kebab-case

### Gate 4: Integration
- [ ] Running `/plan-event` then piping its output file to `/build-schedule` works end-to-end
- [ ] Running `/plan-event` then piping its output file to `/draft-announcement` works end-to-end
- [ ] Running `/plan-event` then piping its output file to `/draft-speaker-outreach` works end-to-end

## Success Criteria

1. A user can run `/plan-event Claude Code Workshop`, then pass the resulting JSON file to `/build-schedule` to get a time-slotted schedule with speakers assigned
2. A user can run `/draft-announcement` with an event plan to get a ready-to-send announcement email with real speaker and venue details
3. A user can run `/draft-speaker-outreach` with an event plan to get personalized outreach emails for every relevant speaker
4. All three commands consume the same event plan JSON — no re-entry of event details

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Event plan JSON missing fields that commands expect | Commands should validate required fields and report clear errors if missing |
| Speaker topics don't overlap with event topicTags | Commands should flag when no speakers match and suggest broadening tags |
| Schedule duration math errors (slots exceed total time) | Build-schedule should validate total slot time <= suggestedDuration |
| Tone drift in emails (too formal, too casual) | Command files explicitly reference CLAUDE.md personality and include tone guidance |
| Venue capacity mismatch in announcements | Announcement command must check `estimatedAttendees * 1.2 <= capacity` |
| Speaker availability not checked against event timing | Build-schedule must note availability conflicts as warnings |

## Open Questions

None — INITIAL.md is specific enough to proceed. The only implicit assumption is that event plan JSON files live in `events/` (established by `/plan-event`).

## Estimated Effort

| Component | Complexity | Notes |
|-----------|-----------|-------|
| `/build-schedule` command | Medium | Schedule math + double-booking validation + dual output (markdown + JSON) |
| `/draft-announcement` command | Low | Template-driven, straightforward data substitution |
| `/draft-speaker-outreach` command | Low-Medium | Multiple personalized emails, speaker matching logic |
| **Total** | **Medium** | ~30-45 min with `/execute-prp` |
