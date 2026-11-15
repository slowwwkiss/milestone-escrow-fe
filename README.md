# Milestone Escrow — trust-minimized freelance payments in Fe

**Pitch:** A freelancer escrow in Fe where the client funds milestones in ETH, releases payment per milestone, and claws back whatever is unreleased after the deadline — no oracle, no dispute committee, no token: the deadline is the only arbitrator.

Entry for the [Fe Builders Challenge](https://fe-lang.org/contest/) (Argot Collective, $5,000 prize pool). Built by Achmad Reza Baehaqi (Bogor, Indonesia), solo.

## The problem

Freelance work runs on trust: the client fears paying for work that never arrives, the freelancer fears delivering work that never gets paid. Existing escrow solutions add a trusted third party (a platform, a multisig committee, an arbitrator) or a governance token — each one a new party to trust, a new fee, and a new failure mode. For small freelance jobs, that overhead kills the deal.

## Approach

One escrow contract per job, deployed with the client, the freelancer, and a lock period. The deadline splits the contract's life into two phases:

- **Before the deadline** — the client funds the escrow (`fund`, payable, any number of top-ups) and pays out milestones (`release(amount)`) as work is delivered. Only the client can fund or release.
- **At / after the deadline** — the release window closes and the client can claw back the unreleased remainder (`refund(to)`). The freelancer's window to be paid has ended.

That is the whole mechanism. No dispute flow is needed because the client never loses the ability to recover unreleased funds, and the freelancer is paid the moment the client approves a milestone — payment is one transaction, not an invoice cycle.

Accounting is explicit: `funded`, `released`, and `refunded` are tracked separately, and `available() = funded - released - refunded`. Every state change happens **before** the ETH transfer (checks-effects-interactions), so a reentrant call finds nothing left to take. Funding closes at the deadline instant (`timestamp <= deadline`), release requires strictly before (`timestamp < deadline`), refund requires at or after (`timestamp >= deadline`).

## Fe highlight

Fe's **effect system** makes the security argument visible in the code. Each message handler declares exactly which storage it touches in its `uses` clause — the compiler rejects a handler that reads or writes anything it did not declare:

```fe
Release { amount } -> Wei uses (
    ctx, client, mut call, freelancer, deadline,
    mut released, funded, refunded, mut log,
) {
    assert!(ctx.caller() == client, "Only the client can release")
    assert!(ctx.timestamp() < deadline, "Deadline has passed")
    assert!(amount <= funded - released - refunded, "Amount exceeds available balance")

    // Effects before interaction: a reentrant call finds the
    // balance already spent.
    released = released + amount
    assert!(
        call.send_value(addr: freelancer, gas: 0, value: amount).success(),
        "Transfer failed",
    )
    log.emit(Released { to: freelancer, amount, total_released: released })
    amount
}
```

The `uses` clause is the audit surface: `mut released` is the only mutation, `call` is the only external interaction, and the ordering (mutation, then send) is right there in the source. The same discipline applies to `init` — its parameters must not shadow storage fields, so `client_addr` / `freelancer_addr` are bound explicitly and assigned to storage. Solidity-style selectors (`#[selector = sol("release(uint256)")]`) keep the contract callable from any EVM tooling with a standard ABI.

## Architecture

Single ingot, two source files:

| Path | Contents |
| --- | --- |
| `fe.toml` | Ingot manifest (`milestone_escrow` v0.1.0). |
| `src/lib.fe` | `MilestoneEscrow` contract: `EscrowMsg` message enum (11 messages), three events (`Funded`, `Released`, `Refunded`), six storage fields. |
| `src/tests.fe` | 13 tests in Fe's own test framework (`fe test`), using `evm.create2` / `evm.call` against the contract. |
| `.github/workflows/ci.yml` | Checks formatting, runs tests, builds — with Fe 26.6.0 on every push. |
| `LICENSE` | Apache-2.0 (from the official starter). |

Messages: `fund()` (payable), `release(uint256)`, `refund(address)`, plus view functions `client()`, `freelancer()`, `deadline()`, `funded()`, `released()`, `refunded()`, `available()`. `refund` takes a payout address (normally the client's own wallet), mirroring the official starter Vault's `withdraw(address)` pattern.

## Build and test

```sh
fe --version   # fe 26.6.0 (5b3f96a)
fe fmt --check
fe test        # 13 passed; 0 failed
fe build       # out/MilestoneEscrow.{bin,runtime.bin,abi.json}
```

Install Fe with [feup](https://fe-lang.org/getting-started/installation/) (Fe 26.5.0 or later is allowed for the contest; this project was built and tested with 26.6.0):

```sh
curl -fsSL https://raw.githubusercontent.com/argotorg/fe/master/feup/feup.sh | bash -s -- --version v26.6.0
source ~/.fe/env
```

No other tools are needed. No mainnet deployment is required (and none was made) — the contract is verified entirely by `fe test` locally.

## Demo

Reproducible CLI example — the happy path, from the test suite:

```sh
fe test fund_then_release_milestones
```

What it proves: deploy with a 7-day lock, fund 1 ETH + 2 ETH, release milestone 1 (1 ETH) and milestone 2 (2 ETH) to the freelancer, and end with escrow balance 0 and `available() == 0`. The refund path is covered by `refund_after_deadline_returns_remainder` (zero lock = deadline is "now": fund 2 ETH, refund 2 ETH to a payout address, escrow ends empty). Run the full suite with `fe test`.

## Limitations

- **One job per contract.** There is no factory and no multi-job registry; deploying one escrow per freelance job is the intended use. A factory would be the natural next step.
- **No partial-dispute flow.** If the client and freelancer disagree mid-job, the contract offers no arbitration — the client simply stops releasing, and the remainder is refunded after the deadline. This is a deliberate design choice (deadline as the only arbitrator), not an oversight, but it means a malicious client can stall a milestone until the deadline and claw the funds back. A commit/reveal delivery hash or a mutual-agreement early-refund would mitigate that.
- **Deadline semantics at the boundary.** Funding is allowed at the exact deadline instant (`timestamp <= deadline`) while release requires strictly before it. This is what makes the post-deadline path testable in Fe's test framework (see NOTES.md); on a live chain the same rules apply.
- **Test-framework limits (not contract bugs).** The Fe 26.6 test EVM keeps `block.timestamp` constant within a test and cannot deploy with value (`create2` with nonzero value reverts), so there is no true "wait past the deadline" test — the deadline-crossing tests use a zero lock period instead. The `fe test` EVM also cannot receive a plain value transfer at the test account address, which is why `refund` takes a payout address. Details in NOTES.md.
- **Not audited.** The contract is small and the invariants are covered by tests, but it has not been professionally audited. Do not use it for large amounts without one.

## Declaration

- **Pre-existing work:** Started from the official [Fe Builders Challenge starter](https://github.com/fe-lang/builders-challenge-starter) (commit `f9d9aa92a03af82c2bbb0dc2e0810244eb13e686`) — its `fe.toml`, CI workflow, `LICENSE`, README skeleton, and the example Vault contract. The Vault was fully replaced by `MilestoneEscrow`; no Vault code remains. All escrow design, code, and tests were written during the contest window (Nov 15, 2026).
- **Third-party code:** Only the starter template above (Apache-2.0) and the Fe standard library (`std::evm`). No other dependencies.
- **AI assistance:** Built with an AI coding assistant (Muse): the assistant wrote the contract and tests, ran `fe test` / `fe build`, and debugged the failures documented in NOTES.md. The human (Achmad Reza Baehaqi) is the responsible contact and reviewed the result.

## Team

Achmad Reza Baehaqi (solo) — Bogor, Indonesia. Contact: erjeganteng3@gmail.com. GitHub: slowwwkiss.

## License

Apache-2.0. See `LICENSE`.
