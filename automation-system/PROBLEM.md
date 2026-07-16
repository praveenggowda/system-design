# Automation System — Problem Statement

Design the monday.com Automation System.

Users can create rules on their boards:

- WHEN a task status changes to Done → THEN notify the manager
- WHEN a task is created → THEN assign it to a specific user
- WHEN a due date arrives → THEN move the item to Overdue

## Constraints

- Tens of millions of boards globally
- 10,000 events per second average, 50,000 at peak
- A single event can match multiple automation rules
- Automation effect should fire within 2 to 5 seconds of the triggering action
- Idempotency required — same event must not trigger the same automation twice
- Audit trail retained for 6 months
- Scope: three trigger types, three action types above

## Trigger Types

1. Board event — status changed, item created, item moved, column value updated
2. Time-based — due date arrives

## Action Types

1. Notify a user or manager
2. Assign a user to an item
3. Move an item to a group (e.g. Overdue)
