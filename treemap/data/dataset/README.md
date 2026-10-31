# Ethereum staking: attributed validator set

Who stands behind every active Ethereum validator, the execution-layer addresses identified along the way, and
the decisions, sources and methods each attribution rests on, so that the attributions can be used and checked.
This is the data behind the stake treemap at <https://pintail.xyz/staking/treemap/>. Generated 2026-10-31T04:45:18Z from a Lighthouse and a Reth archive
node; data through epoch 485,775 (2026-10-30T12:00:23Z). Code commit `489aba03956eaa11f0c04c0cf422ff76758f32a8`, registry fingerprint `eae080d9a897ca19f6c9f6dc80e05f55b56279b500ca9c3194b6c324c8c50505`
(manifest.json).

## Files

| File | Rows | What it holds |
|---|---:|---|
| validators.csv.gz | 815,146 | Active validators with their organisation, allocator, operator, principal and custodian |
| addresses.csv | 27,657 | Execution-layer addresses with an identified role and entity |
| entities.csv | 978 | The entities named in the files: name, kind, parent and organisation |
| boxes.csv | 1,254 | The treemap's boxes: what each rests on, and its stake now and at its peak |
| decisions.json | 324 | Every decision: statement, explanation, targets, evidence, sources and the stake resting on it |
| sources.json | 589 | The sources cited: link, publisher, the passage relied on, and the hash of the copy kept |
| methods.json | 60 | How claims are derived: each method's premises, heuristic and limits, and the anchors it uses |
| attributions.parquet | 2,968,135 | Each validator's treemap box over time, with its withdrawal credentials and clusters |
| stake.parquet | 2,651,030 | Each validator's effective balance over time, while active |
| nodes.parquet | 1,566 | The treemap's boxes: path, kind, entity, basis, method, the decisions each rests on |
| treemap.parquet | 114,579 | Each box's stake and validators at each frame, as the treemap shows them |

The CSV files are UTF-8 with a header row. Addresses are lower-case hex; read `credentials_prefix` as text.
All epochs are beacon-chain epochs (an epoch is 6.4 minutes; epoch 0 began 2020-12-01 12:00:23 UTC).

## Roles

Each validator can have up to four parties behind it, each recorded separately:

- **allocator**: whoever chose the operator for this stake (a staking service, a liquid staking protocol's
  module, an exchange's staking product). The treemap is drawn by allocator.
- **operator**: whoever runs the validator's keys and nodes.
- **principal**: whose ETH it is, where that differs from the allocator and is known.
- **custodian**: who holds the withdrawal address's keys on another's behalf, where known.

On chain, a custodian and its client cannot be told apart: whoever holds the keys acts for both. Principal and
custodian are therefore filled only where a recorded decision establishes them from other evidence (a fund's own
documents, payments to a client's own wallets); elsewhere they are empty, and the holder of a withdrawal address
may be the principal, a custodian, or both.

Each claim has a **basis**: `derived` (read mechanically from a protocol's own contracts, from anchors accepted by
decisions), `asserted` (by a recorded decision), or `inferred` (by a heuristic such as graffiti, fee recipients or
clustering, which can be wrong; methods.json gives each heuristic's failure modes).

## validators.csv.gz

One row per validator active at the latest frame.

| Column | Meaning |
|---|---|
| validator_index | the validator (any beacon node gives its public key: `/eth/v1/beacon/states/head/validators/<index>`) |
| effective_balance_eth | its effective balance (whole ETH) |
| credentials_prefix, withdrawal_address | `0x00` (BLS), `0x01` or `0x02`, and the address withdrawals are paid to |
| withdrawal_address_ens | the address's ENS primary name, where it resolves back to the address |
| organisation | the organisation behind the allocator (an entity id; empty if unattributed) |
| allocator, allocator_basis, allocator_method | the organisation's service or product that allocates the stake |
| operator, operator_basis, operator_method | who runs it, where known |
| operator_organisation | the organisation running the operator (the operator itself if it is one; for a protocol's registered operator, such as a Lido node operator, as a relation decision records) |
| operator_owner | that organisation's ultimate owner as of the data, where a relation decision records one (e.g. Bitwise for Attestant) |
| principal, principal_rests_on | whose ETH it is, and the decisions establishing it |
| custodian, custodian_rests_on | who holds the withdrawal address for another, and the decisions establishing it |
| box_id | the treemap box it sits in (boxes.csv, nodes.parquet): what its attribution rests on |

## addresses.csv

Execution-layer addresses with an identified role: one row per address, role and entity.

| Column | Meaning |
|---|---|
| address | the address |
| role | `withdrawal_address`, `deposit_account`, `fee_recipient`, `anchor` (a contract or account a method reads from, such as a batch deposit contract or protocol registry), or a tag's role (`hot_wallet`, `treasury`, `deposit_funder` and others) |
| relation | for a withdrawal address or deposit account: whether the entity is its validators' `allocator`, `principal` or `custodian` |
| entity, entity_name | the entity (entities.csv) |
| basis | as above; for tags, `first_party`, `on_chain`, `third_party` or `inferred` |
| ref | the decision, anchor or tag it comes from (decisions.json, methods.json), or, for a withdrawal address whose active validators all have one allocator, the box(es) they sit in |
| validators_now, eth_now | its active validators and their stake |
| ens_name | its ENS primary name, where verified |

## entities.csv

Every entity named in the other files: `entity` (a stable id such as `coinbase/prime`), `name`, `kind`, `parent`,
`organisation` (the top of its chain of parents) and `url` (its website). Entity ids are stable from one release to
the next; cluster ids (`cluster/…`, unnamed groups of validators) are not, and are left out.

## boxes.csv

The treemap's leaf boxes: each a set of validators attributed the same way. `rests_on` lists the decisions the
attribution depends on, `method` the method (methods.json) and `basis` how it is known; `eth_now`, `eth_peak` and
`peak_time` give its stake. To see what follows from rejecting a decision, filter on it:

```sql
SELECT box_id, path, eth_now, eth_peak FROM read_csv('boxes.csv') WHERE rests_on LIKE '%0226-%';
```

## decisions.json

Every decision, by id. An active decision gives what was decided (`decision`), a one-line `summary`, its
`grounds` (each with the kind of evidence it rests on: `on_chain`, `first_party` or `third_party`), what would
change it (`would_change`), where written, a fuller `explanation`, and its `targets` (the withdrawal addresses,
deposit accounts or validators it covers), `evidence` (transaction hashes and contract calls anyone can repeat on
their own node, each with a note on what it shows), `sources` and `leads` (third-party pages, such as explorer
labels, that pointed the research but are not relied on), and `stake`: the validators and ETH resting on it now
and at its peak. Decisions no longer in force are listed with their status and the decision replacing them.

## sources.json and methods.json

sources.json gives each source's link, publisher, whether it is the entity's own statement (`first_party`) or
another's, when it was retrieved, the sha256 of the copy kept, and the passage relied on. Explorer pages are cited
by link alone. methods.json gives each method's premises, conclusion, limits and, for heuristics, failure modes,
and the anchors (addresses accepted by decisions) methods read from.

## History

attributions.parquet, stake.parquet, nodes.parquet and treemap.parquet give every validator's box over time, so
the treemap can be rebuilt for any week since genesis.

### attributions.parquet
One row per validator per interval over which it sat in one treemap box, while active. An interval that spans
no frame's epoch (shorter than the week between frames, and falling between two) may be left out: the treemap
never shows it.

| Column | Meaning |
|---|---|
| validator_index | the validator |
| from_epoch, to_epoch | the interval |
| node_id | the box (a leaf of nodes.parquet) |
| path | the box's path, e.g. `Coinbase › Coinbase Prime (custody clients) › 765 named withdrawal addresses` |
| credentials_prefix | `0x00` (BLS), `0x01` or `0x02` over the interval |
| withdrawal_address | the address the credentials name (null for 0x00) |
| reward_cluster, deposit_cluster, withdrawal_cluster | the validator's inferred clusters, if any (ids are not stable from one day's data to the next) |

### stake.parquet

| Column | Meaning |
|---|---|
| validator_index | the validator |
| from_epoch, to_epoch | an interval over which it was active with this effective balance |
| effective_balance_eth | its effective balance (whole ETH), the measure the treemap sums |

### nodes.parquet

Every box of the treemap. `basis` says how its allocator is known: `derived` (read from a protocol's own
contracts, by a method), `asserted` (by a recorded decision), `inferred` (by a heuristic, e.g. clustering or
graffiti). `rests_on` lists the decisions (decisions.json) the box depends on; `method` names the method
(methods.json).

### treemap.parquet

Each leaf box's stake (`eth`) and validator count at each frame (`epoch`, `time`), exactly as in the treemap.

### Rebuilding the treemap

With [DuckDB](https://duckdb.org/), in this folder (or with the files' URLs in place of their names):

```sql
WITH frames AS (SELECT DISTINCT epoch FROM 'treemap.parquet')
SELECT f.epoch, a.node_id, sum(s.effective_balance_eth) AS eth, count(*) AS validators
FROM frames f
JOIN 'stake.parquet' s
  ON s.from_epoch <= f.epoch AND f.epoch < coalesce(s.to_epoch, 9e18)
JOIN 'attributions.parquet' a
  ON a.validator_index = s.validator_index AND a.from_epoch <= f.epoch AND f.epoch < coalesce(a.to_epoch, 9e18)
GROUP BY ALL
EXCEPT
SELECT epoch, node_id, eth, validators FROM 'treemap.parquet';
```

No rows back means the files agree with the treemap. To check the stake against your own node, compare
stake.parquet with the validators' effective balances at any epoch; to check an attribution, follow its box's
`rests_on` to the decision in decisions.json, its evidence and its sources.

Stake by organisation today, for example:

```sql
SELECT coalesce(organisation, '(unattributed)') AS organisation, sum(effective_balance_eth) AS eth
FROM read_csv('validators.csv.gz') GROUP BY 1 ORDER BY 2 DESC;
```

## Licence

The data and the decisions' text are published under the Creative Commons Attribution 4.0 International licence
(CC BY 4.0, <https://creativecommons.org/licenses/by/4.0/>): use them freely, crediting "Stakalytics
(pintail.xyz/staking)". Passages quoted from sources (`excerpt`) remain their publishers' and are quoted so the
decisions can be checked; explorer pages are cited by link only.

## Corrections

Attribution is evidence-based and can be wrong. Where you hold evidence that contradicts a decision, each one says
what would change it.
