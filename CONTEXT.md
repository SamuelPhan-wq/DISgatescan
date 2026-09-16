# Check-in Queue — Diamond International School

A barcode/QR check-in system for EYP (Early Years Programme) and PYP (Primary
Years Programme). Guardians are issued scannable pickup cards; scanning a
card checks in every student linked to it, into a live queue for their
programme.

## Architecture

- **No custom backend/server.** The front-end talks directly to a Firebase
  Realtime Database from the browser.
- **Hosting:** static files on GitHub Pages.
- **Camera scanning:** the ZXing library (`@zxing/library` via unpkg),
  which reads both barcodes and QR codes across browsers.
- **Excel import/export:** the `xlsx` (SheetJS) library, for a two-sheet
  workbook (Students + Cards) used as a bulk-editing template.

All three HTML files below embed the same Firebase config and read/write the
same database, so they always stay in sync with each other.

## Files

- **`admin.html`** — the full console. Tabs: Scan station, EYP display, PYP
  display, Students, Cards, Records. This is where staff manage data.
- **`checkin-app.html`** — a stripped-down, display-only page: just the EYP
  and PYP queue boards (two tabs), meant for a wall-mounted screen. No scan
  input, no data management.
- **`security-scan.html`** — a scan-only page for the security desk, UI
  translated into Lao. No visible queues or roster; it scans a card, checks
  students in, and plays a loud alarm-style sound. Grade codes and school
  name are left in English on purpose.

## Data model (Firebase Realtime Database)

```
students/{studentId}        { name, code (a school ID, staff-assigned, NOT scanned), grade }
cards/{cardCode}             { guardian, family, studentIds: [studentId, ...] }
queues/EYP                   [ {studentId, code, name, grade, guardian, time}, ... ]  (max 10, newest first)
queues/PYP                   [ same shape ]
scanLog/{YYYY-MM}/{pushId}   { studentId, name, grade, program, guardian, code, date, time, ts }
```

Key design decisions:
- **Students vs. Cards are separate.** A student has no scan code of their
  own — a **card** is what gets scanned, held by a guardian, and lists one
  or more `studentIds` (siblings). Losing a card just means deleting that
  one card row; the student record is untouched.
- **Grades:** `Nursery, PreS, Pre-K, Kinder, G1, G2, G3, G4, G5`. Kinder–G5
  are PYP; everything else is EYP (see `programForGrade()` in each file —
  duplicated logic, kept in sync manually since these are static files with
  no shared module).
- **Duplicate prevention:** scanning a card checks each linked student
  against the current queue for their programme; if already present, it's
  skipped (shown as "already in queue"), not re-added.
- **Family grouping:** cards have an optional free-text `family` field
  (not an ID) purely so multiple guardians' cards for the same household can
  be tallied together in the Cards tab.
- **Tardy / late pick-up:** not stored as a flag — inferred purely from the
  scan's time of day. 8:30am–noon = tardy; ≥4:30pm = late pick-up. This is
  computed client-side in the Records tab from `scanLog`, not written back
  to the database.

## Notable UI features already built

- Excel-style column-header filter dropdowns (▾) on the Students and Cards
  tables in `admin.html` — click a column header, get a searchable checklist
  of that column's values, like an AutoFilter.
- Swipe-to-reveal-remove on individual queue slots (pointer events, works
  for touch and mouse) on both `admin.html` and `checkin-app.html`.
- Distinct sounds: a two-note chime per programme on successful check-in
  (plays on the scanning device), a louder "bell" per programme that plays
  only on a device with that programme's display tab open and active (for
  wall displays), and a separate loud alarm pattern in `security-scan.html`.
  All tones are generated with the Web Audio API (no audio files), gain set
  to 1.0 (max, to cut through noise) using square waves.
- Records tab: month picker, All/Tardy/Late toggle with a per-student tally,
  CSV export of whichever view is showing, and a "delete this month's
  records" button.

## Known constraints / past decisions worth knowing

- Originally planned as a Google Sheets + Apps Script system; moved off
  Apps Script entirely due to persistent CORS issues with cross-origin
  `fetch()` calls from GitHub Pages to a `script.google.com` deployment.
  Firebase was chosen specifically because it's built for this kind of
  client-side cross-origin access.
- A camera "snap photo, decode multiple codes from one still image" feature
  was built and then removed — it didn't work reliably. Multi-student
  check-in is instead handled properly through cards listing multiple
  `studentIds`, scanned live one card at a time.
- Firebase Realtime Database rules are fully open (`.read`/`.write: true`)
  — acceptable for this low-stakes internal tool, but worth knowing if
  scope ever expands.
- No test suite; all changes have been verified by manual testing in the
  browser after each edit.
