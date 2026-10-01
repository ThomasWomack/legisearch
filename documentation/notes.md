# Script Behavior
Maps local fields to legistar fields

## Field Mappings

### Events
| Local Field    | Legistar Field       | Transformation | Description |
| -------------- | -------------------- | -------------- | ----------- |
| id             | EventId              | None | Copied as is. |
| body_id        | EventBodyId          | None | Copied as is. |
| meeting_time   | datetime             | Built from EventDate + EventTime | `datetime` isn't a Legistar field. `format_event` creates it by joining the date part of EventDate with the time parsed from EventTime (via `dateutil`). If EventTime is empty or won't parse, it uses 12:00 noon. No timezone is attached. EventDate is parsed as ISO format, with `dateutil` as a fallback. If neither works, the event is skipped with a logged message, since meeting_time can't be NULL. |
| agenda_url     | EventAgendaFile      | Null → `''` | A missing or null agenda file is stored as an empty string. |
| minutes_url    | EventMinutesFile     | None | Copied as is. Stored as NULL if missing. |
| minutes_status | EventMinutesStatusId | Replaced with FAKEFINALSTATUS for nonfinal events > 21 days old | Only happens with `refetch_nonfinal`: if the status isn't FINALSTATUS (10) and the meeting was more than 21 days ago, it's stored as FAKEFINALSTATUS (-10) so the event isn't refetched again. Otherwise copied as is. |
| insite_url     | EventInSiteURL       | None | Copied as is. Stored as NULL if missing. |


### Items
| Local Field        | Legistar Field        | Transformation | Description |
| ------------------ | --------------------- | -------------- | ----------- |
| id                 | EventItemId           | Items filtered and merged | Items with no EventItemId are dropped. Items that share an agenda number with an earlier item are merged into that earlier item (see title/action_text), and only the first item's id is kept. Items before the first agenda number in an event have nothing to merge into, so each is kept as its own row. |
| event_id           | EventId               | Taken from parent event | Set from the parent event's EventId, not from the item record. |
| agenda_number      | EventItemAgendaNumber | Stripped, carried forward | Whitespace is stripped. An item with a blank agenda number gets the most recent non-blank one from earlier in the same event. |
| action_text        | EventItemActionText   | Merged across items | When items share an agenda number, each later item's action text is stripped and appended to the first item's text, separated by a blank line (`\n\n`). |
| title              | EventItemTitle        | Merged across items | Merged the same way as action_text. |
| full_text_lower    | (derived)             | Not stored | `format_event` builds `lower_text` by joining MatterType, AgendaNumber, Title and ActionText with newlines and lowercasing it, but the insert line is commented out, so this column is always NULL. |
| matter_id          | EventItemMatterId     | None | Copied as is. |
| matter_attachments | EventItemMatterAttachments | Converted to JSON | The attachment list is turned into a JSON object of `{MatterAttachmentName: MatterAttachmentHyperlink}`. Attachments with the same name overwrite each other. |
| matter_status      | EventItemMatterStatus | None | Copied as is. |
| matter_type        | EventItemMatterType   | None | Copied as is. |

## Search
- Returns Event Items
- Joins Event and Event Item fields
- Can filter by year & month
- Query Normalization: Lowercase transformation only (No whitespace or other normalization)
- Only finds whole match of query in text fields (no tokenization)
- Only Searches title (EventItemTitle). full_text_lower field not set in fetch.py, so no other fields searched
- Other fields that would be searched if full_text_lower set
  * EventItemMatterType
  * EventItemAgendaNumber
  * EvenItemActionText

### Searched Fields

Inactive:
- EventItemMatterType
- EventItemAgendaNumber
- 