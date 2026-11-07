# Calendar deep-links (Google/Outlook + .ics)

- **category**: other
- **provider**: Google Calendar / Outlook (deep-link only) + .ics (RFC 5545)
- **reusable**: yes — no-SDK, no-auth deep-link/`.ics`-generation pattern, trivial to lift for any app that needs "add to calendar" without a full OAuth Calendar integration.
- **docs**: https://www.rfc-editor.org/rfc/rfc5545 (iCalendar) and https://support.google.com/calendar/thread/2716393 (Google Calendar render URL params)

## Overview
Lets a user add an event to their own calendar via one-click "Add to Google Calendar" / "Add to Outlook" links, or by downloading a generated `.ics` file — with no server call, no OAuth, and no third-party SDK. Both calendar providers expose an unauthenticated URL scheme that pre-fills a "new event" compose screen; the `.ics` file is a plain-text format any calendar client can import.

## Playbook

### Prerequisites
- None — no account, API key, or SDK required. Just event data (title, time, location, description) already available client- or server-side.

### Setup steps
1. Compute the event's start/end as ISO 8601 strings with a known duration/timezone; pick a sensible default end time (e.g. start + 3 hours) if the event has no explicit end.
2. Convert times to the compact UTC basic format required by both the Google deep-link and `.ics` (`YYYYMMDDTHHMMSSZ`).
3. Build the Google Calendar link: `https://calendar.google.com/calendar/render` with `action=TEMPLATE`, `text`, `dates=START/END`, `details`, `location` as query params.
4. Build the Outlook link: `https://outlook.live.com/calendar/0/deeplink/compose` with `path=/calendar/action/compose`, `rru=addevent`, `subject`, `body`, `location`, `startdt`, `enddt` as query params.
5. For the offline/other-clients case, generate a valid `.ics` string (`BEGIN:VCALENDAR` ... `BEGIN:VEVENT` ... `END:VEVENT` ... `END:VCALENDAR`) with CRLF line endings and RFC 5545 text escaping, and offer it as a downloadable file.
6. Render all three as plain links/buttons — no server round-trip needed for any of them.

### Core pattern
```ts
// calendar-links.ts — Google/Outlook one-click "add to calendar" deep-links
function isoToUtcCompact(iso: string): string {
  return new Date(iso).toISOString().replace(/[-:]/g, "").replace(/\.\d{3}Z$/, "Z");
}

type CalendarEventInput = {
  title: string;
  description: string;
  location: string;
  dateTime: string;      // ISO with offset
  endDateTime?: string;
};

function defaultEnd(dateTime: string): string {
  return new Date(new Date(dateTime).getTime() + 3 * 60 * 60 * 1000).toISOString();
}

export function googleCalendarLink(e: CalendarEventInput): string {
  const start = isoToUtcCompact(e.dateTime);
  const end = isoToUtcCompact(e.endDateTime ?? defaultEnd(e.dateTime));
  const params = new URLSearchParams({
    action: "TEMPLATE",
    text: e.title,
    dates: `${start}/${end}`,
    details: e.description,
    location: e.location,
  });
  return `https://calendar.google.com/calendar/render?${params.toString()}`;
}

export function outlookCalendarLink(e: CalendarEventInput): string {
  const params = new URLSearchParams({
    path: "/calendar/action/compose",
    rru: "addevent",
    subject: e.title,
    body: e.description,
    location: e.location,
    startdt: e.dateTime,
    enddt: e.endDateTime ?? defaultEnd(e.dateTime),
  });
  return `https://outlook.live.com/calendar/0/deeplink/compose?${params.toString()}`;
}
```

```ts
// ics.ts — generic .ics file generator, importable by any calendar client
type IcsEventInput = {
  title: string;
  description: string;
  location: string;
  dateTime: string;
  endDateTime?: string;
  uid: string; // must be globally unique per event
};

// RFC 5545 §3.3.11 — escape commas, semicolons, backslashes, and newlines.
function escapeIcsText(value: string): string {
  return value.replace(/\\/g, "\\\\").replace(/;/g, "\\;").replace(/,/g, "\\,").replace(/\n/g, "\\n");
}

// CRLF line endings are required by RFC 5545 — LF-only is silently rejected by some clients.
export function buildIcs(e: IcsEventInput): string {
  const lines = [
    "BEGIN:VCALENDAR",
    "VERSION:2.0",
    "PRODID:-//your-app//EN",
    "CALSCALE:GREGORIAN",
    "BEGIN:VEVENT",
    `UID:${e.uid}`,
    `DTSTAMP:${isoToUtcCompact(new Date().toISOString())}`,
    `DTSTART:${isoToUtcCompact(e.dateTime)}`,
    `DTEND:${isoToUtcCompact(e.endDateTime ?? defaultEnd(e.dateTime))}`,
    `SUMMARY:${escapeIcsText(e.title)}`,
    `DESCRIPTION:${escapeIcsText(e.description)}`,
    `LOCATION:${escapeIcsText(e.location)}`,
    "END:VEVENT",
    "END:VCALENDAR",
  ];
  return lines.join("\r\n");
}
```

### Env vars
none — plain URL construction and text generation, no API keys or config.

### Gotchas
- Google's and Outlook's deep-link query param names/formats differ (`dates=START/END` combined vs. separate `startdt`/`enddt`) — don't assume one shape covers both.
- `.ics` requires CRLF (`\r\n`) line endings per RFC 5545; using plain `\n` is silently rejected or mis-parsed by some calendar clients (notably some versions of Outlook desktop).
- Escape `,`, `;`, `\`, and newlines in every free-text `.ics` field (`SUMMARY`, `DESCRIPTION`, `LOCATION`) — unescaped delimiter characters corrupt the parsed event.
- `UID` in the `.ics` file must be unique per event; reusing an event's own ID or a per-invite-link ID is a good source. Some clients treat an update with the same `UID` as replacing a previously-imported event, which can be desirable or surprising depending on intent.
- This pattern has no way to know whether the user actually added the event — there is no callback, confirmation, or API to query success. It's fire-and-forget by design; do not build features that depend on confirming the add happened.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace, to let invited guests add an event to their own calendar via one-click Google/Outlook links or a downloadable `.ics` file, with no server call or auth involved.
