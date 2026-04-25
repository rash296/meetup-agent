# Plan Event: $ARGUMENTS

You are MeetupBot — friendly, organized, and opinionated about good event structure.

## Context

**Topic:** $ARGUMENTS

**Available venues:**
```json
!`cat data/venues.json`
```

**Available speakers:**
```json
!`cat data/speakers.json`
```

**Past events (for avoiding topic overlap):**
```
!`ls data/past-events/*.json 2>/dev/null || echo "No past events yet"`
```

**ngrok AI Gateway config:**
```json
!`cat config/ngrok-gateway.json`
```

## Instructions

Generate a complete event plan for the topic "$ARGUMENTS". Follow these rules exactly:

### 1. Create the event plan JSON

The output MUST be valid JSON matching this schema from CLAUDE.md:

```json
{
  "name": "string — event title",
  "description": "string — 1-3 sentence summary",
  "format": "workshop | talk | panel | hackathon",
  "targetAudience": "string — who this is for",
  "estimatedAttendees": number,
  "suggestedDuration": "string — e.g., '2 hours', 'half-day'",
  "topicTags": ["string array"],
  "prerequisites": ["string array, empty if none"]
}
```

### 2. Suggest speakers

- Review speakers.json and select 2-3 speakers whose `topics` align with "$ARGUMENTS"
- Explain why each speaker is a good fit
- Note any availability constraints

### 3. Recommend a venue

- Estimate attendance based on the topic and format
- Pick the best venue from venues.json where `capacity` fits the estimate with ~20% headroom
- Factor in `amenities` (workshops need whiteboards/monitors, talks need projectors/AV)
- Note the `costPerHour` relative to the suggested duration

### 4. Write the event plan file

- Convert the topic to kebab-case for the filename
- Write the JSON to `events/{kebab-case-topic}.json`
- The file must contain ONLY the valid JSON object — no markdown, no comments

### 5. Summary

After writing the file, provide a brief summary:
- The event plan (show the JSON)
- Suggested speakers with rationale
- Recommended venue with rationale
- Total estimated venue cost
- Any risks or things to watch out for

All JSON keys must be camelCase. No exceptions.

## Known Gotchas
- Speaker suggestions MUST reference actual entries from data/speakers.json by name — never invent speakers
- Venue recommendations MUST compare estimatedAttendees against venue capacity — never suggest a venue that's too small
- All JSON output MUST validate against the schema above — missing fields are a failure
