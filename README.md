# Self-audit of my personal automation stack — 2026-11-20

I run my own automation on one Windows workstation: crypto exchange bots, market-data collectors,
equity-data scripts, small web/game projects and media tooling. Nobody else maintains these systems,
so defending them is my job. This is the record of an audit I ran with Claude (Claude Code) on
2026-11-20, what it found, what was fixed the same day, and what is still open.

Trading logic, strategies, venues/accounts and secret values are deliberately left out of this note.

## Scope

| Area | What was checked | Tooling |
|---|---|---|
| Secrets on disk | ~45 project folders: hard-coded keys, `.env`/config files, file ACLs | `detect-secrets` 1.5.0 (keyword/token plugins), manual review of every hit |
| Secrets in git history | Every git repository: secret-like files ever committed, `.gitignore` coverage | `git log --diff-filter=A`, `git ls-files`, `git check-ignore` |
| Python dependencies | Global interpreter (197 pkgs), 2 isolated bot/collector venvs, 6 `requirements.txt` | `pip-audit` 2.10.1 (PyPI advisory DB / OSV) |
| JavaScript dependencies | 12 projects with `package-lock.json` | `npm audit` |
| Running processes | Which interpreters/venvs live services use, so fixes would not break them | process + loaded-module inspection |

Secret values were never printed: scans reported only file, line and type, and key files were inspected by key **name** only.

## Findings and status

| # | Finding | Severity | Status |
|---|---|---|---|
| 1 | Global Python had **102 known vulnerabilities in 9 packages** (aiohttp, urllib3, certifi, pillow, pypdf, filelock, brotli, setuptools, soupsieve) | High | **Fixed** — upgraded to the minimum fixed versions; `pip check` clean; import smoke test passed (incl. CUDA torch); re-audit: 0 known vulns |
| 2 | 6 JS projects had high-severity advisories (vite, undici, wrangler/miniflare, puppeteer-core, zod, sharp, @expo/cli …) | High | **Fixed** — `npm audit fix` (no forced majors): 6 projects now 0; 1 project has 3 moderate left that need a breaking upgrade |
| 3 | Isolated venvs used by live bots/collectors | — | Audited: **0 known vulns**; left untouched while running |
| 4 | A trading bot's `.env` holds wallet/API private keys in plaintext, and AI-agent sandbox groups have Modify rights on it | High | Open — restrict ACL to my user, move keys to Windows Credential Manager (one of my bots already uses `keyring`) |
| 5 | An old `config.json` (2025-11) holds LLM provider API keys in plaintext | Medium | Open — revoke/rotate the keys, delete the file |
| 6 | A service password stored in a plaintext `.env` | Medium | Open — move to a password manager / Credential Manager |
| 7 | One git repo (no commits yet) has no `.gitignore`, so its `.env` would be committed on the first commit | Medium | Open — add `.gitignore` before first commit |
| 8 | Git history of all repos | — | Checked: **no secret files ever committed**; only `.env.example` templates are tracked |
| 9 | 40-odd scanner hits in vendored third-party code (pgAdmin, Nakama docs, pandas tests, a wasm bundle) | — | Reviewed: false positives |

## Still to do (owner actions)

- Exchange API keys: confirm withdrawal permission is off and IP allow-listing is on for every key; rotate on a schedule.
- Phishing-resistant MFA (passkey / security key) on GitHub, Google and Anthropic accounts.
- CI: GitHub Actions → Anthropic via OIDC (Workload Identity Federation) instead of static API keys.
- Re-run this audit monthly (`pip-audit`, `npm audit`, `detect-secrets`).

## How I use Claude for this

Reviewing my own code and dependency advisories, triaging scanner output, checking unfamiliar
packages/scripts before I run them, and hardening how my bots store credentials. I only test systems I own.
