# nuon-ext-replay-stack

A [Nuon CLI extension](https://docs.nuon.co/guides/cli-extensions) that replays a missed install_stack phone-home.

## When to use

Sometimes a phone-home request from a customer install reaches the API and the payload gets stored in
`install_stack_version_runs`, but the corresponding `install_stack_versions` row never flips to `active` (e.g. the
update transaction got rolled back, the workflow signal was dropped, etc.). The customer's install hangs at the
"awaiting phone home" step indefinitely.

The recovery flow is:

1. Trigger a fresh reprovision of the install in the dashboard. This creates a **new** `install_stack_versions` row
   with a new `phone_home_url`, but it has zero runs yet.
2. Run this extension. It POSTs the previous run's payload (already in the DB) to the new `phone_home_url`, which
   flips the new stack version to active and lets the install complete.

## Install

```bash
nuon ext install <your-org>/nuon-ext-replay-stack
```

## Usage

```bash
# Uses NUON_INSTALL_ID from your current CLI context
nuon replay-stack

# Or pass explicitly
nuon replay-stack inlgosqg7mm06vukjlidehih49
```

Requires `nuon login` (org + token) and a selected install (or an explicit install ID arg).

## How it works

A single `GET /v1/installs/{id}/stack` returns the install's stack with the last 10 stack versions and each version's
last 10 runs preloaded. The extension:

1. Picks the newest stack version with `phone_home_url != ""` **and** `runs: []` — that's the post-reprovision SV
   awaiting its first phone home.
2. Picks the most recent run across all SVs — that's the prior SV's last run (the missed payload).
3. POSTs that payload (with `request_type: "Create"` injected) to the new `phone_home_url`.

The phone-home endpoint itself is unauthenticated (it's normally called from the customer's Lambda or Terraform
local-exec), so once we have the URL + payload, no further auth is needed.

## Development

```bash
# Local install (symlink) for iteration
nuon ext install $(pwd)

# Test
NUON_DEBUG=true nuon replay-stack <install_id>

# Uninstall
nuon ext remove replay-stack
```
