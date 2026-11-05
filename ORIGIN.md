# Where webtyp comes from

webtyp was not designed on a whiteboard. It grew out of one clinic, one screen that JavaScript made painful, and
the moment that screen first ran as Go in the browser.

## The clinic system before it

A medical centre in Chile ran its whole day on **Pa100T**: patients, the professionals' schedules, bookings and the
clinical record of every visit, on one PostgreSQL database.

## The screen it could not do

Gastroscopy needed something Pa100T could not offer: capture video from the endoscope, move through it with a
joystick, keep the images and produce the report as a PDF in a fixed layout. A separate module was written for it,
and it was the first experiment with **Go compiled to WebAssembly** in the browser.

The pain that started everything was JavaScript: video capture, device input and PDF generation each meant another
library in another style, and most of that code could not be reused anywhere else. Seeing the same job done in Go,
in the browser, with types and one language from the database to the screen, showed it was possible.

## What it became

That module is now deprecated: its JavaScript dependencies broke and it cannot be maintained, so it runs in
production only as a compiled binary, reading the same database as Pa100T. The idea it proved became webtyp:

- **One language end to end.** The same Go models validate on the server and in the browser.
- **Reusable pieces, not one-off scripts.** Capture, PDF and forms are libraries any app composes, so the next
  specialty does not start from zero.
- **Built for people who are starting.** Every API, error and notice is written so a beginner understands it, in
  their own language.

The first app built on it, `mjosefa-cms`, replaces Pa100T for that same clinic; its data comes over with
`webtyp.com/dataimport`, and the gastroscopy screen returns as a webtyp module that renders its PDF in the browser.
