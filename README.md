# JL Rental Management System

- `web/` single-file front end. `template.html` has no renter data. Build locally: `python3 web/gen.py "MASTER RENTER LIST.xlsx" && python3 web/build.py`, then open `web/index.html`.
- `backend/` Node API for Google Drive storage, Google Sign-In and role enforcement. See `backend/README.md`. Tests: `cd backend && npm i && npm test`.
- `docs_drive_plan.md` architecture plan.

**Keep this repository private.** Renter data (names, contacts, rents) is deliberately excluded by `.gitignore`; never commit the workbook, `data.json` or the built `index.html`, and never commit `.env` or Google credentials.
