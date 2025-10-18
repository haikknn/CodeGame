## Quick context

CodeGame is a Node.js + MySQL + Redis web application that runs AI "tank" scripts in a sandboxed simulator. The server is Express + Sequelize and the game logic lives under `sandbox/`.

When editing code, prefer reading these files to understand the big picture:
- `app.js` — application entry, middleware and error handling
- `env.js` — global bootstrapping: loads models into `global`, constructs `Game()` helper and queuing
- `models/` — Sequelize model definitions and relationships are auto-imported in `models/index.js`
- `routes/` — route modules mounted by `routes/index.js`; session user lookup is performed there
- `sandbox/` — game simulation code (AI runtime) used by `Game()` and `services/` scripts
- `services/` — batch/maintenance scripts (`calc_rank.js`, `calc_tournament.js`, etc.)

## Primary developer workflows

- Install dependencies and build client bundle:

  npm install
  npm run build

- Run locally (development): copy `config/_sample.json` to `config/development.json`, edit DB/Redis credentials, then:

  node app

- Run maintenance tasks (examples):

  # calculate rank (production env expected)
  npm run rank

  # calculate tournament
  npm run tournament

Notes: `package.json` scripts rely on environment variables. On Windows PowerShell you may need to set environment variables differently (the repo historically assumes a Unix-like shell for inline env values).

## Architecture & data flow (concise)

- HTTP requests enter Express (`app.js`) and use a session backed by Redis. `routes/index.js` injects the currently logged-in `User` as `res.locals.me` using `User.find(...)`.
- Database models are defined per-file in `models/` and imported by `models/index.js`. Relationships (belongsTo, hasMany) are wired there.
- Game simulation is executed through `global.Game(mapId, code1, code2, options, callback)` (defined in `env.js`). It will:
  - attempt to return cached `Result` packed via `jsonpack`
  - otherwise load `Map`, run the `sandbox` game runner via an async queue, and store the packed replay in `Result`

- `services/*` scripts (e.g. `calc_rank.js`) run many head-to-head games using `runCodes` utilities and update `Code` rows. These scripts are intended to be run from the command line (often in production env).

## Project-specific conventions and patterns

- Models are loaded dynamically: each `models/<name>.js` exports a function that returns an array compatible with `sequelize.define`. See `models/index.js`.
- Globals: `env.js` merges `models` onto `global` and sets `global.$config`. Many files rely on those globals (e.g. `Map`, `Code`, `Result`, `User`), so when editing, consider that some references may not be explicitly required.
- JSHint predefs: `package.json` lists several globals used in client-side templates: `Map`, `Result`, `Tournament`, `Code`, `History`, `User`, `Game`, `Promise`.
- Client build: Gulp builds assets (look at `gulpfile.js` and `client/js/` + `client/css/`). Use `npm run build` or `npm run watch` during development.

## Integration points & external dependencies

- MySQL (configured under `config/*.json`) — Sequelize is used; `models/index.js` calls `sequelize.sync()` at startup.
- Redis — session store via `connect-redis` (configured through `config.redis` referenced in `app.js`).
- GitHub OAuth — `config/development.json` expects GitHub key/secret for account login (see README instructions).

## Editing guidance (practical tips for the agent)

- When adding or changing models, update `models/<name>.js` and rely on `models/index.js` dynamic import; no central registration needed, but confirm relationship wiring in `models/index.js` if adding associations.
- Avoid adding explicit require() for models in files that assume globals; instead, be consistent with existing code — either use global `Map`/`Result` or add local `require('..')` if you prefer clearer imports, but keep style consistent in nearby files.
- Long-running batch scripts (in `services/`) expect to be run with NODE_ENV=production in original package scripts — be cautious changing DB writes.
- The game runner (`sandbox/`) is synchronous from a single worker perspective but is invoked through an async queue in `env.js`. If you need to parallelize runs, modify the queue concurrency there and ensure DB writes remain safe.

## Examples from repo (patterns to follow)

- Caching result before running game: `env.js` checks `Result.find(...)` using md5 of code contents and stores `jsonpack.pack`ed replay after the run.
- Route mounting: `routes/index.js` uses `node-require-directory` to load route modules and mounts each under `/<routeName>`.

## Safety & testing notes

- There is no automated test suite in the repo. Before editing runtime-critical logic (game/sandbox or services), run a small manual smoke test by invoking `Game(mapId, code1, code2, {cache:false}, callback)` from a short script or the node REPL with `require('./env')` loaded.

## Files to inspect when stuck
- `env.js`, `sandbox/` (especially `sandbox/game.js` and `sandbox/replay.js`), `models/index.js`, `routes/index.js`, `services/calc_rank.js`, `gulpfile.js`, `package.json`, `README.md`.

If anything here looks incomplete or you want more detail on a specific area (models, services, client build), tell me which area and I'll expand the instructions.
