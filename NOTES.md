# NOTES — Fe Builders Challenge: Milestone Escrow

Working notes for the main agent. Status as of 15 Nov 2026, ~20:30 WIB.

## Registration status: NOT DONE — needs a live browser

Registration is a Google Form. Programmatic submission was attempted 5 times
(plain curl + curl_cffi Chrome impersonation, with session cookies, fresh
`fbzx`, `fvv=1`, `pageHistory=0`, `submissionTimestamp=-1`,
`partialResponse`/`draftResponse`) — every attempt returned **HTTP 400**
with the form re-rendered and all required questions flagged empty. The form
is the modern JS-driven Google Forms: the HTML contains **no** `entry.*`
input names at all (fields exist only in `FB_PUBLIC_LOAD_DATA_`), and the
classic `/formResponse` POST path is rejected. This is a form-side block, not
a wrong-field problem — entry IDs were confirmed from the form's own data
(`1284094601` title, `894943470` idea, `1432028924` name, `50100267` email,
`2147442098` team, `333530346` repo URL, `2046107834` Zulip).

**What the main agent must do (browser task, ~2 minutes):**

1. Open https://docs.google.com/forms/d/e/1FAIpQLSehDUUbqfQt67Ecy21zIsOFN7pp4orx9Ko9DKCWcDkjhEUVBg/viewform
2. Fill in:
   - Project working title: `Milestone Escrow — trust-minimized freelance payments in Fe`
   - Project idea (1–3 sentences): `A Fe smart contract escrow for freelance work: a client funds milestones in ETH, the freelancer delivers, and funds release on client approval or auto-release after a deadline; unapproved work can be refunded after expiry. Built to show how Fe's type system and traits make escrow logic clear, safe, and auditable.`
   - Responsible contact name: `Achmad Reza Baehaqi`
   - Contact email: `erjeganteng3@gmail.com`
   - Team members / GitHub handles: `Achmad Reza Baehaqi (solo) — GitHub: slowwwkiss`
   - Repository URL (optional): leave blank for now, or the repo URL once the GitHub repo exists (see below)
   - Zulip contact (optional): blank
3. Submit. Confirmation arrives **by email** to erjeganteng3@gmail.com with
   the project ID and the **private submission link** — that link is required
   to submit the project. No sign-in is required to fill the form ("Sign in
   to Google" is only offered to save progress).
4. Deadline: **Nov 17, 2026, 23:59 UTC = Nov 18, 06:59 WIB. Firm, no
   exceptions** except if the organizers' side fails.

Submission form (for later, after registration): the starter README links
https://docs.google.com/forms/d/e/1FAIpQLSd2pqUkJzFpZMu5FVQP-NtetrIoB6nI-pHG1MLTOQhCpsLKqg/viewform
— but the real submission goes through the **private link from the
registration email**. Final submission deadline: **Nov 24, 2026, 23:59 UTC =
Nov 25, 06:59 WIB**. Winners announced Dec 1, 2026. Prizes paid in USDC on
Ethereum Mainnet to one payout address per winning project (requested by
email after winners are announced).

## Build status: DONE — 13/13 tests pass, contract compiles

- Fe **26.6.0 (5b3f96a)**, installed via feup at `~/.fe/bin/fe`.
- `fe fmt --check` — clean.
- `fe test` — **13 passed, 0 failed** (run Nov 15 ~20:26 WIB).
- `fe build` — emits `out/MilestoneEscrow.bin`, `.runtime.bin`, `.abi.json`
  (ABI verified: constructor `(address,address,uint256)`, `fund()` payable,
  `release(uint256)`, `refund(address)`, 7 views, 3 events).
- No mainnet deployment, no money spent. Local test only.

## What remains before submit (main agent)

1. **Register** (browser, see above) — BLOCKING. Without it, no submission.
2. **Create the public GitHub repo** `slowwwkiss/milestone-escrow-fe`
   (or another name) and push this directory. The local git remote is still
   the starter's (`fe-lang/builders-challenge-starter`); repoint it:
   `git remote set-url origin https://github.com/slowwwkiss/milestone-escrow-fe.git`,
   then push. The commit the jury reviews must be the pushed SHA.
3. **Commit the work** (not yet committed — see `git status`).
4. Submit via the private link with the repo URL + exact commit SHA before
   Nov 24, 23:59 UTC.

## Fe 26.6 findings (worth knowing for review / future work)

Probed empirically with throwaway ingots in `/tmp/fe-probe` (not part of
the submission):

1. **Ingot names must be identifiers** — `milestone-escrow` is rejected,
   `milestone_escrow` works.
2. **Init params may not shadow storage fields.** `init(client: Address,
   ...)` with a `client` field fails with "also bound by an effect" — the
   starter avoids this by naming the param `lock_period` vs field `lock`.
   Renamed to `client_addr` / `freelancer_addr`.
3. **Every function argument needs its label at the call site**
   (`deploy(lock: ..., client: ..., freelancer: ...)`); missing labels are
   compile errors. Same for struct-style message construction.
4. **Every field a handler touches must be in its `uses` clause** — the
   compiler names the missing one (e.g. `deadline` in `Fund`).
5. **`fe test` keeps `block.timestamp` constant inside a test** and
   **`create2` with a nonzero deployment value reverts.** Consequence:
   there is no true "wait past the deadline" test. Deadline-crossing is
   tested with a **zero lock period** (deadline == "now"): funding is still
   allowed at that instant (`timestamp <= deadline`), release is closed,
   refund is open. The contract logic is identical on a live chain; only
   the test's clock is fake.
6. **The `fe test` EVM cannot receive a plain value transfer at the test
   account address** (`evm.address()`): `send_value` to it fails with
   "Transfer failed", while the same call to a fresh address (e.g.
   `0x1234`) succeeds. Verified with a minimal probe contract. That is why
   `refund` takes a payout address (`refund(address)`), mirroring the
   starter Vault's `withdraw(address)` — and why the refund test pays out
   to `0x5678`, not to the test account. On a real chain, refunding to the
   client's own EOA works normally.
7. **No `warp` / `prank` cheatcodes** were found in the Fe 26.6 test
   framework (`fe doc` output + search). The test caller is always
   `evm.address()`; access-control tests therefore deploy with a *different*
   client address and assert the stranger's call reverts.
8. The starter's own 3 tests pass unmodified on Fe 26.6.0 (verified before
   replacing the Vault).

## Contest facts (verified Nov 15 from fe-lang.org/contest + blog)

- $5,000 total: 1st $2,500 / 2nd $1,500 / 3rd $1,000, USDC on Ethereum
  Mainnet, funded by Argot Collective. All three prizes guaranteed.
- Build window Nov 3–24, 2026. Registration closes Nov 17, 23:59 UTC.
  Submissions close Nov 24, 23:59 UTC. Winners ~Dec 1, 2026.
- Open worldwide, beginners welcome, solo or team, multiple projects
  allowed (register each separately). Argot members + judges excluded.
- Judging: technical quality 30%, Fe-specific value 25%, originality 20%,
  usefulness/insight 15%, documentation & demo 10%. No mainnet deploy
  needed; no video required; social reach does not affect score.
- AI assistance allowed, not scored negatively, must be disclosed (done in
  README). Core technical contribution must be in Fe. Public GitHub repo
  with OSI-approved license (Apache-2.0 kept from starter). Reproducible
  build + automated tests required.
- Judges include Austin Griffith (BuidlGuidl), Marius van der Wijden
  (Ethereum Foundation / Geth), Grant Wuerker + Micah Scopes (Argot / Fe
  core), Ahmed Castro, Philogy, Rahul Kothari, jistro.
