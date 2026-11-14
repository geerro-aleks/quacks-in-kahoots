# Quacks in Kahoots!

The app starts with the questions in `quiz.mjs`, one shared quiz room, and a Neon Postgres database for the PIN, game phase, questions, players, answers, and scores. The host can add, edit, or remove questions in the lobby; saved questions are kept for future games. The host can reset the finished quiz for a new group. The race character artwork is served from `assets/race-character.png`.

## Technology stack

- **Frontend:** a single static page, [index.html](index.html), in plain HTML, CSS, and vanilla JavaScript with no framework or build step. Styling uses Tailwind CSS (loaded from its CDN) plus custom CSS; fonts are Bricolage Grotesque and Figtree from Google Fonts. The race track is drawn on an HTML Canvas 2D element.
- **Backend:** Node.js 20+ with one request handler, [app.mjs](app.mjs), built on the standard Fetch API (`Request`/`Response`). On Vercel it runs as serverless functions (`api/`); locally, [server.cjs](server.cjs) wraps it in a plain `node:http` server.
- **Database:** Neon serverless Postgres, accessed through the `@neondatabase/serverless` driver (the only npm dependency). The schema is in [schema.sql](schema.sql).
- **Live updates:** clients poll `/api/state` every three seconds; there are no WebSockets.
- **Hosting:** Vercel (static page and API functions, configured in [vercel.json](vercel.json)).

## Game flow and limits

The host starts the quiz. In the lobby, they can manage 1–50 questions, each with a prompt, four choices, and one correct answer. Changes are saved to Neon for future games and cannot be made after a game starts. Players have 30 seconds to answer by default, followed by a six-second race, correct-answer reveal, and a 25-second break by default. In the lobby, the host can change the answer time and the break (5–120 seconds each) under **Game timing**. During a game, the host can **Pause quiz** to freeze every timer and block answers, then **Resume quiz** to continue where it stopped. Correct answers earn 100 points. The server checks deadlines and scores; the answer key is not sent to players until the reveal.

Players can join while the quiz is in progress. A latecomer can answer the current question if its timer is still open; otherwise they join in time for the next question. Joining is closed after the quiz finishes, until the host resets it.

Clients request updates every three seconds because Vercel Functions do not provide this app with a shared always-on process for pushing live state to all players. This produces many requests in a large room: 500 players for a full quiz could make roughly 130,000 state requests, in addition to answer and join requests. Free Vercel/Neon usage is limited, so this is **not yet load-tested or guaranteed for hundreds of simultaneous players**. Test with a small group first and check both providers' current limits. If usage requires a paid plan, stop before upgrading if you do not want charges.

The room state and scores persist in Neon across function restarts. The project is a prototype, not hardened against abuse or large public events.

## Changing the host key

Open host controls with your current key, then open **Change host key** (in the lobby or after a game finishes). Enter the current key again and choose a new one (8–200 characters). The new key is stored as a salted scrypt hash in the `quiz_settings` table, never as plain text. You stay signed in; any other open host sessions are signed out. The server creates the `quiz_settings` table and the timing and pause columns automatically on startup; [schema.sql](schema.sql) includes them too if you prefer to run it yourself.

The `HOST_KEY` environment variable keeps working as a recovery key, so you can still get in and set a new key if you forget the one you chose. To go back to using only `HOST_KEY`, run `DELETE FROM quiz_settings;` in the Neon SQL Editor.
