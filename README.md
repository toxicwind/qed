# QED

QED is a small JavaScript text-analysis toolkit: four "lens" modules that each
extract a different signal from text, plus a `LensOrchestrator` that
auto-discovers and runs them, with GitHub Actions CI exercising the lenses.

Plain Node.js (CommonJS), zero dependencies, no build step, no binary.

## Lenses (`src/_11ty/lenses/`)

Each lens is a single module exporting `{ name, description, analyze(data, meta) }`
and returning a plain JSON object with a `confidence` score.

| Lens | What it does |
|---|---|
| `tectonic` | Repository health/drift scoring: `healthScore = 100 − 2 × days-since-push`, stale-repo flag after 30 idle days, pass threshold 50 in CI mode / 70 otherwise |
| `osint` | Regex extraction of URLs, email addresses, and IPv4 addresses; deduped lists plus an IOC count |
| `stylometric` | Trigram fingerprinting (top 5), Shannon entropy, word count, unique-word ratio, average word length |
| `cryptographic` | Pattern detection for potential base64 keys (40+ chars), hex hashes (32–64 chars), and GitHub tokens (`ghp_`/`gho_`/`ghu_`/`ghs_`/`ghr_`); reports `CRITICAL` when tokens are found |

## Orchestrator (`lib/lens-orchestrator.js`)

`LensOrchestrator` auto-discovers every `lens_*.js` module in `src/_11ty/lenses/`
(the directory is configurable via the `lensDir` constructor option). Drop a new
`lens_*.js` file in and it's picked up — no registration step.

```js
const { LensOrchestrator } = require('./lib/lens-orchestrator');

const orch = new LensOrchestrator();
await orch.discover();                        // finds lens_*.js modules

const one = await orch.analyze('osint', text);       // run a single lens
const all = await orch.analyzeAll(text, { path });   // run every lens
```

## CI

- [`.github/workflows/agentic-lens-ci.yml`](.github/workflows/agentic-lens-ci.yml)
  — "Agentic Lens-First CI": runs on push to `main`, pull requests, and manual
  dispatch. Discovers the lenses, then asserts: stylometric confidence ≥ 0.8,
  cryptographic lens reports `clean`, OSINT lens finds at least one URL. The
  final MCP smoke step is a no-op — there is no `services/mcp-stack` in the tree.
- [`.github/workflows/tectonic-drift.yml`](.github/workflows/tectonic-drift.yml)
  — daily 06:00 UTC cron (plus manual dispatch): runs the tectonic lens and fails
  if the repo health score drops below threshold.

## Configuration

Copy `.env.example` to `.env` and fill in values. Never commit real secrets.

| Key | Purpose |
|---|---|
| `GITHUB_TOKEN` | GitHub API token for drift checks (CI uses the built-in `${{ secrets.GITHUB_TOKEN }}`) |
| `LENS_ENABLED` | Enable/disable lens runs |
| `LENS_SWARM_CONCURRENCY`, `SWARM_MAX_CONCURRENCY` | Parallelism knobs |
| `LENS_AUTO_COMMIT` | Whether automation may commit |
| `SWARM_TIMEOUT_MS` | Per-run timeout (default 30000) |
| `OPHEL_VAULT_PATH`, `MODELBEATS_API_ENDPOINT`, `ZEDRA_DAEMON_HOST`, `MUSEPOOL_CDN_PRIMARY`, `MCP_TRANSPORT` | Integration endpoints (unset by default) |

## License

No license file is present in this repository yet — all rights reserved until one
is added.
