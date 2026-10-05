# Sumcoin Explorer Work — September 28, 2026

## Purpose

This file records the production Sumcoin Insight explorer changes identified and prepared for upstreaming on September 28, 2026.

The live explorer was compared against clean GitHub clones rather than blindly copying npm-installed directories.

Production explorer:

`/root/explorer/sumcoin-explorer`

Live package examined:

`/root/explorer/sumcoin-explorer/node_modules/bitcore-node-sumcoin`

Installed version:

`3.1.15`

Branch:

`explorer-production-fixes-2026`

Pull request:

`sumcoinlabs/bitcore-node-sumcoin#1`

## Audit method

The production npm package was compared against a clean clone of the GitHub repository using recursive file diffs.

Differences caused only by npm installation metadata, lockfiles, `.gitignore` files, or packaging were separated from actual production source changes.

The important production changes in this repository were narrowed to exactly:

- `lib/services/bitcoind.js`
- `lib/services/web.js`

## Coinstake / Proof-of-Stake fix

File:

`lib/services/bitcoind.js`

The old code treated every non-coinbase transaction fee as:

`inputSatoshis - outputSatoshis`

That is wrong for a Sumcoin Proof-of-Stake coinstake transaction because the output includes newly minted staking reward.

The production code now detects a coinstake using the Sumcoin/Core transaction structure:

- not coinbase
- contains inputs
- at least two outputs
- first output has zero satoshis
- first output has an empty script

When detected it sets:

`tx.coinstake = true`

and:

`tx.stakeRewardSatoshis = tx.outputSatoshis - tx.inputSatoshis`

and:

`tx.feeSatoshis = 0`

This prevents a staking reward from incorrectly appearing as a negative transaction fee.

Normal non-coinbase transaction fee calculation remains unchanged.

## Localhost web binding

File:

`lib/services/web.js`

The production explorer changed:

`self.server.listen(self.port);`

to:

`self.server.listen(self.port, '127.0.0.1');`

This is intentional.

The Insight backend runs behind Nginx on the production explorer. Port 3001 should therefore listen only on localhost instead of being directly exposed on all interfaces.

Do not casually remove this unless the deployment architecture is intentionally changed.

## Related insight-sum-api dependency

These new fields:

- `transaction.coinstake`
- `transaction.stakeRewardSatoshis`

are consumed by the corresponding production changes in:

`sumcoinlabs/insight-sum-api#4`

That API layer exposes the stake reward and reports a zero transaction fee for coinstake transactions.

Preferred merge order:

1. `bitcore-node-sumcoin#1`
2. `insight-sum-api#4`
3. `insight-sum-ui` production synchronization

## Changes intentionally NOT copied

The live npm-installed `package.json` contains npm-generated metadata and differs significantly from repository source.

Those npm installation differences were intentionally excluded.

Repository-only files such as lockfiles and `.gitignore` were also left alone.

## Validation

Before the PR was created:

`git diff --check`

passed.

Syntax validation also passed:

`node --check lib/services/bitcoind.js`

`node --check lib/services/web.js`

The PR initially contained exactly two source files with:

51 additions and 3 deletions.

These source versions were already being used by the live production explorer.

## Other related work

A separate production audit identified three source changes in `insight-sum-api`:

- `lib/currency.js`
- `lib/status.js`
- `lib/transactions.js`

A much larger `insight-sum-ui` production audit was also completed.

The UI changes include currency presentation, status display, transaction/coinstake handling, address/payment UX, payment sounds, branding, Spanish pages, homepage changes, CSS, icons and SEO-related work.

The UI repository still needs a carefully scoped PR. Generated files and runtime SEO artifacts should not simply be copied wholesale.
