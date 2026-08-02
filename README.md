# Off-Ice Jump Training

A single-file training app for a figure skater working through an off-ice
strength progression. No build step, no server, no dependencies.

## Hosting it on GitHub Pages

1. Create a new repository (public is simplest — Pages on private repos needs a paid plan).
2. Drop `index.html` in the root and commit.
3. Repo → **Settings** → **Pages** → Source: *Deploy from a branch* → Branch: `main`, folder `/ (root)` → Save.
4. Wait a minute. The site appears at `https://<username>.github.io/<repo>/`.
5. On her phone, open that URL and use **Add to Home Screen** so it opens like an app.

## How the data works

Everything is stored in the browser's `localStorage` under the key `office-jump-v1`.

- Nothing leaves the device. There is no account and no server.
- Phone and laptop are **separate stores**. Settings → *Copy backup code* / *Paste backup code* moves history between them.
- Clearing browser data or using private browsing will wipe it. Copy a backup code occasionally.

## What's in it

- **Today** — daily greeting, weekly rotation meter, heavy-ice-week toggle, session start,
  jump test card when measure mode is on
- **Session wizard** — one exercise at a time, write-up + form notes, set logging,
  countdown timers for holds with beep and vibrate, 60s rest timer
- **Program** — full three-phase exercise library
- **Progress** — jump height trend, badges, jump-readiness checklist, session history
- **Settings** — skater name, four themes, weekly goal, phase, measure mode,
  backup codes, erase

## Measure mode

Off by default; turn it on in Settings. Uses `DeviceMotionEvent`, which requires
HTTPS — GitHub Pages provides that. On iOS the phone prompts for motion access
the first time.

**Jump height** comes from flight time. The phone reads close to zero g while
airborne, so a jump shows as a dip in acceleration magnitude; height is
`h = g·t²/8`. Resolution is about 1–2 cm at typical sampling rates, so the
number is worth comparing across months, not days. Implausible readings (under
3 cm, over 100 cm, flights outside 150–900 ms) are discarded.

**Balance score** comes from rotation rate during single-leg holds — RMS wobble
in deg/s, mapped to 0–100. It attaches to the set automatically when the hold
timer finishes.

If sensors are unavailable or permission is refused, both fall back cleanly and
nothing else in the app is affected.

Targets adapt on their own: complete every set at target without flagging the
session as tough, and the target nudges up next time. Flag it tough, or fall
well short, and it eases back. Floors at the starting value, caps at a sensible
ceiling per exercise.

## Editing the program

All content lives in the `EX` object and the `PHASES` array near the top of the
`<script>` block. Each exercise has `why`, `how`, `tips`, `mode` (`reps` or
`hold`), `sets`, `base`, `step`, `max`, and optional `perSide` / `jump` flags.
Adding an exercise means adding an entry and listing its id in a phase's `plan`.

## Note

General training guidance, not medical or coaching advice. Volume and jump work
should be cleared with her skating coach or a trainer who works with young
athletes.
