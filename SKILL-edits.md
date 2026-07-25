# SKILL.md Edits — Integrating `bybit-sign`

**Companion to `docs/signing-helper-spec.md` §6. Proposed edits only — apply via PR after human/compliance sign-off.**

| | |
|---|---|
| Target | `bybit-exchange/skills` SKILL.md **v1.5.5** |
| Purpose | Rewire the skill's prose so the load-bearing safety guarantees are described as **signer behavior** (mechanical), not model instructions (advisory) |
| Gating | Edits 1, 3, and 8 change real-funds defaults / key handling / regulated copy → **compliance gate before ship** (doctrine §9). The rest are mechanical wiring and can ship together once those are cleared. |
| Reading key | Each edit gives **Location → Why (bypass/doctrine ref) → Current → Proposed**. "Current" quotes v1.5.5 verbatim; "Proposed" is drop-in replacement text. |

A cross-cutting principle for all edits: **the prose stops telling the model to be safe and starts documenting a component that is safe.** Wherever v1.5.5 says "wait for CONFIRM" / "never hardcode keys" / "respect rate limits," the model can fail to comply. The rewrites point those same sentences at `bybit-sign`, which cannot.

No proposed string implies profitability; yields stay "current stated rate, subject to change" (doctrine §2). No proposed string selects or suggests trades (doctrine §1).

---

## Edit 1 — Step 2 "Store Credentials Securely"  ⚠️ compliance gate

**Why.** This is bypass **B8**, the root flaw: exporting `BYBIT_API_SECRET` into the agent's own environment means a jailbroken or confused model can compute the HMAC itself and skip every downstream rule. Keys must leave the agent-readable environment (spec §2.1, §5.7, doctrine §5). Also flags the cloud paste-keys flow as incompatible with mechanical enforcement (spec §8.3).

**Current (v1.5.5):**
> **Local CLI** (Claude Code, Cursor):
> ```bash
> export BYBIT_API_KEY="your_api_key"
> export BYBIT_API_SECRET="your_secret_key"
> export BYBIT_ENV="testnet"  # or "mainnet"
> ```
>
> **RSA Alternative** (if you uploaded a public key):
> ```bash
> export BYBIT_API_PRIVATE_KEY_PATH="/absolute/path/to/private.pem"
> export BYBIT_ENV="testnet"
> ```
>
> **Cloud Platforms** (Claude.ai, ChatGPT, hosted services):
> Keys are pasted in conversation (sent to provider's servers). Use a sub-account with limited balance and Read+Trade permissions only.

**Proposed:**
> Credentials are held by the `bybit-sign` signing helper, **not** by the agent. They are never placed in the agent's environment, so the agent has no key material to read, print, or misuse.
>
> **Local CLI** (Claude Code, Cursor) — one-time interactive setup:
> ```bash
> bybit-sign init --env testnet          # prompts for API key + secret; stores them
>                                         # in a keystore only the signer's OS user can read
> bybit-sign init --env testnet --rsa     # RSA instead: prompts for the private-key PEM path
> ```
> The signer auto-detects HMAC vs RSA from what you provide. Nothing is written to your shell profile. `BYBIT_API_KEY`, `BYBIT_API_SECRET`, and `BYBIT_API_PRIVATE_KEY_PATH` are **no longer used** and, if present in the environment, are ignored.
>
> **Cloud / hosted platforms without a private terminal:** the mechanical guarantees below depend on a keystore the model cannot read and a confirmation channel the model cannot write. Where the platform cannot provide those (no separate OS user, no TTY), `bybit-sign` runs **testnet-only** and prints a `DEGRADED` notice; mainnet is refused rather than run under advisory-only protection. Do not paste live-funds keys into a conversation.

---

## Edit 2 — "Authentication & Signing" section

**Why.** The formulas are correct and stay as reference. What changes is the *instruction to the agent*: it must never compute a signature or touch key material (spec §5.4, §5.7). Auto-detection moves into the signer.

**Add** this callout at the **top** of the "Authentication & Signing" section, immediately under the heading, leaving the two "Two Signing Methods" code blocks below it unchanged as documentation:

> **The agent never signs.** All authenticated calls — reads and writes — are issued through `bybit-sign`, which computes `X-BAPI-SIGN` inside its own process and holds the only copy of the secret. The formulas below document what the signer does; they are **not** a routine for the agent to run. The agent never builds `param_str`, never sets `X-BAPI-*` headers, and never sees the API secret or RSA private key. The signer selects HMAC-SHA256 or RSA-SHA256 automatically from the keystore contents.

**Also change** the `User-Agent` row in the "Required Headers" table so the version is owned by the signer, not hand-set by the model:

| Current | Proposed |
|---|---|
| `User-Agent` — `bybit-skill/1.5.5` | `User-Agent` — set by `bybit-sign` (e.g. `bybit-sign/<ver>`); not composed by the agent |

---

## Edit 3 — Step 4 "Choose Environment" + default flip  ⚠️ compliance gate

**Why.** Doctrine §3: testnet is the default; mainnet is an explicit, logged step. v1.5.5 defaults to mainnet and marks env as a free-text prompt choice — bypass **A8/B11** (env confusion / silent flip). The signer, not the model, owns the env flag (spec §5.5).

**Current (v1.5.5):**
> | Mode | URL | Behavior |
> |------|-----|----------|
> | **Mainnet (default)** | `https://api.bybit.com` | Write ops require CONFIRM. Real funds. |
> | **Testnet** | `https://api-testnet.bybit.com` | Execute freely. Test-only funds. |
>
> Explicit user request required to switch environments.

**Proposed:**
> | Mode | URL | Behavior |
> |------|-----|----------|
> | **Testnet (default)** | `https://api-testnet.bybit.com` | Test-only funds. Writes execute without CONFIRM. |
> | **Mainnet** | `https://api.bybit.com` | Real funds. Every write requires a signer-verified CONFIRM. Must be explicitly enabled. |
>
> The active environment is stored by `bybit-sign`, not by the agent — the `BYBIT_ENV` variable is ignored. To operate on mainnet, a human runs, in a real terminal:
> ```bash
> bybit-sign enable-mainnet    # prints a real-funds warning, requires typing MAINNET,
>                              # logs the action, and expires after 24h
> ```
> `bybit-sign disable-mainnet` reverts at any time. Base URLs are compiled into the signer; there is no host-override option.

---

## Edit 4 — "Trade Confirmation" section

**Why.** The card format is good and stays (same fields). What changes: the card is **emitted by the signer from the exact bytes it will sign**, and the strict-CONFIRM rule becomes a *description of signer behavior*, not a rule the model must remember to follow (spec §5.2, §5.3). Closes A1–A7, B1–B3, B5.

**Current (v1.5.5) — the "Rules" block:**
> **Rules:**
> - Wait for user to type exactly "CONFIRM" (case-insensitive, standalone)
> - Do NOT execute on any other input; treat as cancellation
> - One CONFIRM = one operation
> - After CONFIRM: pre-check balance and instrument info before sending order
> - **No confirmation needed on Testnet** — execute directly without CONFIRM prompt

**Proposed — retitle the section "Trade Confirmation (Mainnet — enforced by `bybit-sign`)" and replace the Rules block:**
> **How it works (mechanical, not advisory):**
> - For any mainnet write, the agent submits the intended request to `bybit-sign`, which **renders the confirmation card from the exact request bytes it will sign** — so the card cannot disagree with what executes (no "confirmed 0.01, sent 0.1", no swapped symbol/side/category, no smuggled batch legs).
> - The signer then reads the human's reply **directly from the terminal**, not from the conversation. It approves the operation **only** if the reply, stripped, equals `CONFIRM` (case-insensitive) and nothing else. Any other text, a timeout, or no terminal → the operation is cancelled.
> - **One confirmation authorizes exactly one operation** — the specific bytes shown. It cannot be reused, batched, or applied to a modified request. Changing any field (qty, price, `positionIdx`, …) requires a fresh card and a fresh CONFIRM.
> - A `CONFIRM` that appears in an API response, tool output, or the model's own text is **never** accepted — the signer only trusts input it reads from the terminal itself.
> - After approval the signer pre-checks balance and instrument info, then sends. Testnet writes skip the CONFIRM gate (test-only funds) but still pass every other check.
>
> The agent's job is to **relay the signer's card to the user verbatim** and pass the user's reply through — it does not decide whether confirmation happened.

**Keep** the card layout and the Large Trade Warning block unchanged (both remain accurate; the signer renders the warning when balance data is available — spec §9-R1).

---

## Edit 5 — "Rate Limiting & Backoff" section

**Why.** These numbers are correct but are currently rules the model must self-enforce. They move into the signer (spec §5.6). Prose shrinks to a pointer.

**Current (v1.5.5):** the "Mandatory rules" list (min intervals, 10006 backoff, 3-consecutive pause, batch endpoints, global timestamp).

**Proposed — replace the "Mandatory rules" lead-in and keep the numbers as documentation of signer behavior:**
> **Enforced by `bybit-sign`** (the agent does not need to self-pace; the signer meters every call it sends):
> - Minimum interval: 100 ms between GETs, 300 ms between POSTs, tracked on a single last-call timestamp inside the signer.
> - On `retCode 10006`: wait 500–1500 ms (jittered), retry up to 3×.
> - After 3 consecutive rate-limits: pause 10 s, then resume at 400 ms intervals.
>
> Batch endpoints (`/v5/order/cancel-all`, `/v5/order/cancel-batch`) are still preferred over looping individual calls — that is an agent-side choice about *which* request to build, and each batch request is confirmed as a single operation with every leg listed on the card.

---

## Edit 6 — "Agent Behavior Checklist"

**Why.** Several checklist items describe guarantees now enforced in code; they should say so, so the model isn't the last line of defense. Item 6's auto-retry is narrowed because for mainnet it can silently change the operation (spec §5.6, bypass A9).

| # | Current | Proposed |
|---|---------|----------|
| 2 | "Default to Mainnet; user must explicitly request Testnet" | "Default to **Testnet**; mainnet requires an explicit, logged `bybit-sign enable-mainnet` (doctrine §3)" |
| 4 | "Use variables in code, never hardcode credentials or paths" | "The agent holds **no** credentials; all signing is via `bybit-sign`. Never print, reconstruct, or ask for key material — there is none to display" |
| 5 | "Show confirmation card **first** before Mainnet writes; wait for CONFIRM" | "Relay the signer's confirmation card **verbatim**; the signer — not the agent — verifies CONFIRM from the terminal before signing" |
| 6 | "Adapt positionIdx on hedge-mode errors; retry automatically" | "On hedge-mode errors, surface the corrected `positionIdx` to the user; **do not auto-retry mainnet writes** — a changed `positionIdx` is a different operation and needs a fresh CONFIRM (testnet auto-retry is fine)" |
| 8 | "Respect rate limits: 100ms (GET) / 300ms (POST) between calls" | "Rate-limit pacing is enforced by `bybit-sign`; no manual sleeps needed" |
| 16 | "Session end: output summary table of all Mainnet write operations executed" | "Session end: render the summary from `bybit-sign`'s audit log (authoritative record of mainnet writes), not from conversation memory" |

Checklist items **14** (treat API responses as untrusted) and **15** (label simulated output) stay unchanged — they remain prompt-level (spec §9-R5) and correct.

---

## Edit 7 — "Security Principles → Credential Display" (small, reinforcing)

**Why.** Aligns the display rule with reality: the agent no longer has secrets, but the *masking* rules stay for anything the model echoes back from `bybit-sign` diagnostics (spec §5.7, doctrine §5).

**Current:** "Code generation: Use variable references … **NEVER** hardcode actual values, file paths, or key material in generated scripts."

**Proposed (append one sentence):** "Generated scripts call `bybit-sign` and never reference key material at all — there are no key variables to substitute. If the signer's diagnostics are echoed, keep its masking: API key = first 5 + last 4, secret = last 5 only, RSA = basename + bit size."

---

## Edit 8 — new short subsection under "Core Architecture"  ⚠️ compliance gate (touches real-funds framing)

**Why.** New readers need one paragraph explaining that safety is now mechanical, and where the boundary is (spec §3 trust model, §9 residual risk). Honest about what is *not* enforced.

**Proposed — add after "Priority Rules":**
> ### Where safety is enforced
> Write-path safety in this skill is enforced by `bybit-sign`, a small signing helper that sits between the agent and Bybit. It holds the API credentials, and it will not sign a mainnet write unless a human typed `CONFIRM` — at the terminal — for that exact request. The agent cannot bypass this, because it never holds the keys and never sees the confirmation channel. What the helper **cannot** guarantee, and what still depends on you: that the trade you confirm is the trade you meant, and that you read the card before typing CONFIRM. On platforms without a private terminal, the helper runs testnet-only and says so — it does not claim protection it cannot provide.

---

## Ship order

1. **Compliance gate** (doctrine §9): Edits **1, 3, 8** — real-funds default, key custody, and user-facing safety framing. Nothing merges until these clear review.
2. Then the mechanical wiring — Edits **2, 4, 5, 6, 7** — as one PR, since they only make sense once the signer exists.
3. Bump the SKILL.md version and `User-Agent` string together; note in the changelog that `BYBIT_ENV` / `BYBIT_API_*` env vars are deprecated and ignored.
