# Calendar Context

## Immediate Temporal Context (Calendar — Optional)

This section is ONLY included in the report if Calendar access (ie Google Calendar)
is available and succeeds. If the calendar fetch fails, is unavailable,
or returns no events, omit this section entirely without comment.

### Instructions

Fetch the following from Calendar:
- The most recently completed event (past)
- The next upcoming event (future)

Calculate precise relative times from the current moment:
- Under 60 minutes: use minutes ("in 34 minutes", "23 minutes ago")
- Under 24 hours: use hours and minutes ("in 2 hours 15 minutes")
- Under 7 days: use days and hours ("3 days ago", "in 2 days")
- Beyond 7 days: use days only ("in 14 days", "11 days ago")

Present the two events as a single sentence or short paired lines
in the Bureau's dry, matter-of-fact voice. Use the actual event
title from the calendar — do not paraphrase or summarize it.

### Placement in Report

Insert as a new section between TEMPORAL BEARINGS and WORK WEEK STATUS,
under the heading:

  IMMEDIATE VICINITY

### Examples of good phrasing

  "Your last event (Dentist) was 2 days ago.
   Your next event (Meeting with Daisy) is in 34 minutes."

  "You departed your last known waypoint (Lunch with Bob) 
   3 hours ago. Your next scheduled waypoint (Sprint Planning) 
   arrives in 2 days."

  "The Bureau notes a gap in your schedule. Your last logged 
   event was 11 days ago (Haircut). Nothing is scheduled for 
   the next 14 days. This is either vacation or cause for concern."

### Edge Cases

- If no past events exist: omit the past half, present only upcoming
- If no upcoming events exist: "No further waypoints on record. 
  Either your schedule is admirably clear, or you have lost track 
  of it entirely."
- If an event is currently in progress: "You are currently 
  mid-event ({{EVENT_NAME}}), which began {{X}} minutes ago 
  and ends in {{Y}} minutes. The Bureau suggests you pay attention."