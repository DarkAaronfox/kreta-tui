# CLAUDE.md

Terminal client (`kreta`) for the Hungarian e-KRÉTA school system. Rust 2024, ratatui 0.30, tokio, reqwest (rustls).

## Language rules

- **Everything user-visible in the TUI is Hungarian**: labels, tab names, key hints, status and error messages, CLI `--help`.
- **Everything else is English**: code, identifiers, comments, docs (README, this file), commit messages.

## Commands

```sh
cargo run -- --demo                     # UI with made-up data, no login
cargo test                              # unit tests + render test of every tab
SHOW=1 cargo test renders -- --nocapture   # print each tab as text (130x40)
cargo test live -- --ignored            # hits the real IDP with fake credentials
cargo test browser -- --ignored         # headless Chromium code-capture test
cargo clippy --all-targets -- -D warnings && cargo fmt --check   # what CI runs
```

Release: bump `version` in `Cargo.toml`, commit, then `git tag vX.Y.Z && git push --tags`.
`.github/workflows/release.yml` builds `kreta-<target>.tar.gz` for x86_64/aarch64 Linux, and `install.sh` downloads the latest one.

## Layout

| File | Role |
| --- | --- |
| `src/main.rs` | arg parsing, terminal setup, event loop (input thread + tick task + API messages on one mpsc channel) |
| `src/app.rs` | `App` state, key handling, spawning API calls (`spawn` → `Msg`), `Data` derived from `RawData` |
| `src/ui.rs` | all rendering; one `draw_*` per tab, palette constants at the top |
| `src/stats.rs` | weighted averages, timelines, "how many 5s needed" maths (unit-tested) |
| `src/store.rs` | session (`~/.local/share/kreta-tui/session.json`), last school (`school.json`, kept on logout) and cache (`~/.cache/kreta-tui/data.json`), all written 0600 |
| `src/demo.rs` | fake data in the raw KRÉTA JSON shape, for `--demo` and tests |
| `src/api/auth.rs` | OAuth2 + PKCE: headless form login, code exchange, refresh, revoke |
| `src/api/browser.rs` | browser login: Chromium via the DevTools HTTP endpoint, else clipboard watching |
| `src/api/client.rs` | authenticated GETs with transparent token refresh; `JsonExt` lenient accessors |
| `src/api/models.rs` | typed views parsed leniently from `serde_json::Value` |

## KRÉTA API notes

- Login is OAuth2 authorization code + PKCE at `https://idp.e-kreta.hu`, client `kreta-ellenorzo-student-mobile-ios`, redirect `https://mobil.e-kreta.hu/ellenorzo-student/prod/oauthredirect`. The redirect URL cannot be served locally, so the code is captured instead:
  - **Headless**: GET the authorize URL, scrape `__RequestVerificationToken` and `ReturnUrl`, POST `/account/login`, and stop redirect-following at the redirect URI.
  - **Browser**: launch Chromium with `--remote-debugging-port=0` and a temp profile. Open the URL via `PUT /json/new` (some browsers ignore CLI URLs on a fresh profile) and poll `/json/list`. Kill the whole process group afterwards, because launchers are often wrapper scripts.
- School search: `GET https://idp.e-kreta.hu/logininstituteselector?searchValue=<OM id | name | code>` (≥3 chars) returns `<li>` items with `data-val="<institute code>"` and the text `Name (code - om)`.
- Brave Origin shows a native first-run welcome window that DevTools can't close. It's suppressed by seeding the temp profile's `Local State` with `brave.origin.free_tier_accepted` and `browser.first_run_finished`.
- Access-token JWT claims: `kreta:institute_code`, `kreta:user_name`, `role`.
- Data: `https://<institute>.e-kreta.hu/ellenorzo/V3/Sajat/{TanuloAdatlap,Ertekelesek,Mulasztasok,BejelentettSzamonkeresek,HaziFeladatok,Feljegyzesek,OrarendElemek}`. The timetable needs `datumTol`/`datumIg` as UTC ISO timestamps.
- Fields are often null or missing. Always go through `JsonExt` (`str_at`, `path_str`, `date_at`) and never deserialize into strict structs. Raw JSON is cached as-is, so model changes don't invalidate the cache.
- Averages count only `Tipus.Nev == "evkozi_jegy_ertekeles"` grades with `SzamErtek` 1–5, weighted by `SulySzazalekErteke`. The overall average is the mean of the subject averages.

## Conventions

- Never store the password; only tokens.
- UI must not block: network work goes through `App::spawn` and comes back as a `Msg`.
- Keep `demo.rs` in sync with the fields `models.rs` reads so `--demo` and the render test exercise them.
