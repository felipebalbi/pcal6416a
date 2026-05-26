# AGENTS.md — `pcal6416a`

Canonical guidance for AI coding assistants (and humans) working on this
repository. This file is the source of truth for project conventions,
build/test commands, and code-style rules. `.github/copilot-instructions.md`
intentionally defers to this file.

## What this crate is

`pcal6416a` is a platform-agnostic, `#![no_std]` Rust driver for the NXP
[PCAL6416A](https://www.nxp.com/docs/en/data-sheet/PCAL6416A.pdf) 16-bit I²C
I/O expander. It is built on top of the [`embedded-hal`] and
[`embedded-hal-async`] 1.0 traits and uses the [`device-driver`] crate to
generate the register interface from `device.yaml`.

Key facts (see `Cargo.toml`):

- Package: `pcal6416a` `0.3.0`, MIT-licensed, Rust **edition 2024**.
- Categories: `embedded`, `hardware-support`, `no-std`.
- Lints in the manifest: `unsafe_code = "forbid"`, `missing_docs = "deny"`,
  clippy `correctness`/`suspicious`/`perf`/`style = "forbid"`,
  `pedantic = "deny"`. Do not weaken these.
- MSRV enforced in CI: **Rust 1.85** (see `.github/workflows/check.yml`
  `msrv` job). The README still says `1.81`; treat the CI value as
  authoritative until the README is fixed.
- Single optional feature: `defmt` (enables `defmt` derives and forwards
  `device-driver/defmt-03`).
- Patched dependencies: `embedded-hal` and `embedded-hal-async` are pinned
  to `git = "https://github.com/rust-embedded/embedded-hal.git"` via
  `[patch.crates-io]`. Expect `Cargo.lock` churn on `cargo update`.

[`embedded-hal`]: https://docs.rs/embedded-hal
[`embedded-hal-async`]: https://docs.rs/embedded-hal-async
[`device-driver`]: https://docs.rs/device-driver

## Repository layout

```
.
├── AGENTS.md                       ← you are here
├── Cargo.toml                      ← crate manifest, lint config, features
├── README.md                       ← short user-facing intro
├── CONTRIBUTING.md                 ← licensing/DCO note
├── CODE_OF_CONDUCT.md
├── CODEOWNERS
├── LICENSE                         ← MIT
├── SECURITY.md
├── deny.toml                       ← cargo-deny config
├── rustfmt.toml                    ← nightly-only options (see below)
├── build.rs                        ← rebuild trigger for device.yaml
├── device.yaml                     ← register map consumed by device-driver
└── src/
    └── lib.rs                      ← entire driver + unit tests (#[cfg(test)])
└── .github/
    ├── copilot-instructions.md     ← minimal pointer, see end of this file
    └── workflows/
        ├── check.yml               ← fmt, clippy, semver, doc, hack, deny, msrv
        └── nostd.yml               ← cargo check on thumbv8m.main-none-eabihf
```

There are no `examples/`, `tests/`, or `benches/` directories. All tests live
in the `#[cfg(test)] mod tests` block at the bottom of `src/lib.rs`.

## Building and testing

All commands below have been verified against `main` on Windows with a
standard `rustup` install. They match what CI runs in
`.github/workflows/check.yml` and `nostd.yml`.

### Default build

```sh
cargo build
```

### Unit tests

The unit tests use `embedded-hal-mock` and `tokio` (for `#[tokio::test]`
async tests):

```sh
cargo test
```

There are currently 59 unit tests + 1 ignored doctest. All run on the host
(they do not require hardware).

### Formatting

```sh
cargo fmt --check
```

`rustfmt.toml` sets `group_imports = "StdExternalCrate"` and
`imports_granularity = "Module"`, which are nightly-only options. On stable
`cargo fmt --check` prints warnings like
`unstable features are only available in nightly channel` but still exits 0.
CI runs `cargo fmt --check` on the **nightly** toolchain (see the `fmt`
job), so before submitting you should match that:

```sh
cargo +nightly fmt --check
```

### Clippy

CI matches `dtolnay/rust-toolchain@master` for `stable` and `beta` and runs
clippy via `giraffate/clippy-action@v1` with the explicit flags below. To
reproduce locally:

```sh
cargo clippy -- -F clippy::suspicious -F clippy::correctness -F clippy::perf -F clippy::style
```

Note that `Cargo.toml` already promotes those four groups to `forbid` and
`pedantic` to `deny`, so a plain `cargo clippy` will also fail on the same
issues; the explicit `-F` flags above mirror CI exactly.

### Documentation

```sh
RUSTDOCFLAGS="--cfg docsrs" cargo doc --no-deps --all-features
```

PowerShell equivalent:

```powershell
$env:RUSTDOCFLAGS = "--cfg docsrs"
cargo doc --no-deps --all-features
```

### Feature-powerset check (cargo-hack)

```sh
cargo install cargo-hack    # one-time
cargo hack --feature-powerset check
```

This iterates over `{}` and `{defmt}` (the only feature).

### MSRV check

```sh
rustup toolchain install 1.85
cargo +1.85 check
```

### `no_std` target check

```sh
rustup target add thumbv8m.main-none-eabihf
cargo check --target thumbv8m.main-none-eabihf --no-default-features
```

### Supply-chain checks (cargo-deny, cargo-semver-checks)

Both run in CI but are not required locally:

```sh
cargo install cargo-deny
cargo deny --manifest-path ./Cargo.toml check --all-features

cargo install cargo-semver-checks
cargo semver-checks
```

## Code conventions

- **Edition:** Rust 2024. Use 2024-edition idioms (e.g. `let … else`,
  expression-position `match` cleanups). Do not regress to 2021.
- **`no_std`:** The crate is `#![cfg_attr(not(test), no_std)]`. Anything in
  the non-test path must not use `std`. Tests may use `std` (and do — they
  use `tokio` and `Vec`).
- **Unsafe:** `unsafe_code = "forbid"` at the crate level. Do not introduce
  `unsafe` blocks; if you genuinely need one, that is an architectural
  discussion, not a local change.
- **Missing docs:** `missing_docs = "deny"`. The crate-level
  `#![allow(missing_docs)]` opt-out exists only because the generated
  `device-driver` items would otherwise trip it. New hand-written public
  items should still carry doc comments.
- **Clippy:** Treat `pedantic` as a hard requirement (it is `deny`). Use
  `#[must_use]` on pure accessors (`AddrPinState::address`, `Pin::bit`,
  `Pin::number`, etc.), prefer `const fn` where possible, and avoid the
  patterns clippy-pedantic flags (unnecessary clones, needless borrows,
  shadowed names, etc.).
- **Formatting:** `max_width = 120`, `imports_granularity = "Module"`,
  `group_imports = "StdExternalCrate"`. Use `cargo +nightly fmt` before
  pushing so the import grouping actually takes effect.
- **Error type:** A single generic `Pcal6416aError<E>` wraps the I²C error
  type and implements `embedded_hal::digital::Error`. New fallible APIs
  should return `Result<_, Pcal6416aError<I2c::Error>>`, not bespoke error
  types.
- **Sync vs async:** The crate provides both
  `device_driver::RegisterInterface` (blocking, `embedded_hal::i2c::I2c`)
  and `device_driver::AsyncRegisterInterface` (async,
  `embedded_hal_async::i2c::I2c`) impls for `Pcal6416aDevice<I2c>`. When
  adding a new operation, provide both flavours and add tests for each
  (look for the `_async` suffix convention on test functions).

## Driver / HAL specifics

- **Register map:** `device.yaml` is the single source of truth for the
  PCAL6416A register layout. The `device_driver::create_device!` macro in
  `src/lib.rs` generates the `Device` type and per-register accessors from
  it. Prefer editing `device.yaml` over hand-rolling register code.
  `build.rs` emits `cargo:rebuild-if-changed=device.yaml` so edits trigger
  a rebuild.
- **YAML conventions:** `default_byte_order: LE`, `default_bit_order: LSB0`,
  `defmt_feature: defmt`. Per-pin fields are named `<reg-prefix>_<port>_<bit>`
  (e.g. `I0_3`, `O1_7`, `PE0_0`, `PUD1_5`). Keep that pattern when adding
  fields so the generated accessor names stay consistent (`i_0_3()`, etc.).
- **I²C address:** `IOEXP_ADDR_LOW = 0x20` and `IOEXP_ADDR_HIGH = 0x21`,
  selected by the `AddrPinState` (`Low`/`High`) field on `Pcal6416aDevice`.
- **Register width:** `LARGEST_REG_SIZE_BYTES = 2`. Both `write_register`
  impls allocate a `[u8; 1 + LARGEST_REG_SIZE_BYTES]` stack buffer and
  write `&buf[..=data.len()]` so 1-byte writes don't accidentally clobber
  the next register. Preserve this if you change the I²C transaction code.
- **Shared device / pin splitting:** `SharedDevice<I2c, M>` wraps a
  `Device` inside `embassy_sync::mutex::Mutex<M, _>` and exposes per-pin
  `IoPin<'a, I2c, M>` handles via `split()`. The `RawMutex` parameter `M`
  is supplied by the caller (e.g. `CriticalSectionRawMutex`,
  `NoopRawMutex`). `IoPin` implements the
  `embedded-hal-async` digital traits.
- **`defmt`:** Public error/enum types use
  `#[cfg_attr(feature = "defmt", derive(defmt::Format))]`. Mirror that
  pattern on any new public enum or error.

## Commit & PR conventions

Verified against `git log --pretty=%s` on `main`:

- Subject line: imperative mood, capitalised, ≤ 50 characters where
  practical. The repo accepts both Conventional-Commits style
  (`feat: add GPIO pin interface…`) and plain imperative subjects
  (`More registers in device.yaml`, `Added blocking implementation`).
  When in doubt, prefer Conventional Commits.
- Blank line between subject and body; wrap body at 72 columns; the body
  should explain **what** and **why**, not how.
- PR titles are typically suffixed with the PR number by GitHub's
  squash-merge (e.g. `… (#21)`). You do not need to add this yourself.
- **AI attribution (required):** every commit that includes AI-generated
  or AI-assisted work must carry an `Assisted-by` trailer:

  ```
  Assisted-by: <AGENT_NAME>:<MODEL_VERSION> [TOOL1] [TOOL2]
  ```

  Example:

  ```
  Assisted-by: GitHub Copilot:claude-opus-4.7
  ```

  AI agents must verify their own identity (agent name and model version)
  before composing this trailer — do not hard-code a value from a previous
  session.
- **No `Signed-off-by` from AI agents.** Only humans may certify the DCO.
- See `.github/copilot-instructions.md` for the canonical AI-attribution
  wording.

## What not to do

- Don't introduce `unsafe` (lint is `forbid`).
- Don't downgrade or `allow(...)`-away the clippy `forbid`/`deny` lints in
  `Cargo.toml` to make code compile. Fix the underlying issue.
- Don't add a `std` dependency to the non-test compilation path.
- Don't hand-edit register accessor code that ought to come from
  `device.yaml` via `device_driver::create_device!`.
- Don't change the I²C wire format in
  `Pcal6416aDevice::{read,write}_register` without preserving the
  "write only `&buf[..=data.len()]` bytes" behaviour described above.
- Don't force-push to shared branches and don't open PRs from an AI
  agent without a human reviewer.

## How to find more context

- **Datasheet:** <https://www.nxp.com/docs/en/data-sheet/PCAL6416A.pdf>
- **`device-driver` crate docs:** <https://docs.rs/device-driver>
- **`embedded-hal` 1.0:** <https://docs.rs/embedded-hal/1.0.0>
- **`embedded-hal-async` 1.0:** <https://docs.rs/embedded-hal-async/1.0.0>
- **`embassy-sync`:** <https://docs.rs/embassy-sync>
- **CI definitions:** `.github/workflows/check.yml`,
  `.github/workflows/nostd.yml` — always the source of truth for what
  "green" means.

---

## Incorporated from `.github/copilot-instructions.md`

The following is the full content of `.github/copilot-instructions.md` at
the time of writing, kept here so this file is a strict superset:

### Commit Messages
- Subject line: capitalized, 50 characters or less, imperative mood
  (e.g., "Fix bug" not "Fixed bug")
- Separate subject from body with a blank line
- Wrap body text at 72 characters
- Use the body to explain *what* and *why*, not *how*

### AI Attribution
Every commit that includes AI-generated or AI-assisted work **must**
contain an `Assisted-by` trailer in the commit message:

```
Assisted-by: AGENT_NAME:MODEL_VERSION [TOOL1] [TOOL2]
```

Where:
- `AGENT_NAME` is the name of the AI tool or framework
  (e.g., `GitHub Copilot`)
- `MODEL_VERSION` is the specific model version used
  (e.g., `claude-opus-4.6`)
- `[TOOL1] [TOOL2]` are optional specialized analysis tools used
  (e.g., `coccinelle`, `sparse`, `smatch`, `clang-tidy`)

Basic development tools (git, cargo, editors) should not be listed.

AI agents **must** verify their own identity (agent name and model
version) before composing the `Assisted-by` trailer — do not assume or
hard-code a model name from a previous session.

AI agents **MUST NOT** add `Signed-off-by` tags. Only humans can certify
the Developer Certificate of Origin.
