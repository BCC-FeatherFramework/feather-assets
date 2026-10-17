# Feather Assets: implementation plan

**Status:** A0–A5 done on 2026-10-17; [Contract 1](CONTRACT.md) proposed for freezing (0.2.0). Release 10/10 and lifecycle 23/23 for horse, wagon and boat passed on 2026-10-17 (section 6).
**Master plan:** [Feather Assets Master Plan](https://github.com/FeatherFramework/feather-framework-docs/blob/main/feather-assets/Feather_Assets_Master_Plan.md) (architecture proposal, "accept for detailed discovery").
**First consumer:** bcc-stables (horses). Wagons (bcc-wagons) and boats (bcc-boats) are the second and third types.
**Pattern:** the same Contract 1 service kernel as `feather-property` and `feather-contracts`.

## 1. Scope from the master plan

Assets owns asset identity, asset type, canonical model reference, owner interest,
status, provenance, lifecycle and the domain metadata revision. It does not own
spawning, AI, stable or garage UI, fuel or feed, handling, customization
presentation or theft gameplay.

Initial contract: `RegisterAsset`, `GetAsset`, `ListAssetsByHolder`,
`TransferOwnership`, `ChangeAssetStatus`, `UpdateAssetMetadata`, `RetireAsset`,
`CheckInterest`. Events: registered, ownership transferred, status changed,
metadata revised, retired. All are implemented.

## 2. A0: consumer inventory

Taken from the upstream BCC sources (`player_horses`, `player_wagons`, `boats`).

| Use case | Consumer | Assets facts | Stays in the consumer |
|---|---|---|---|
| Buy a horse, wagon or boat | stables, wagons, boats | `RegisterAsset` (provenance = the consumer's order) | price, shop menus, the money step (Economy) |
| Tame and register a wild horse | stables | `RegisterAsset`, provenance `tamed_capture` | taming gameplay, registration fee |
| Craft a boat | boats (`SaveNewCraft`) | `RegisterAsset`, provenance = the craft record | recipe, crafting |
| Name, gender, tack | stables | metadata `{ name, gender, components }` | component catalog and preview |
| Wagon or boat name | wagons, boats | metadata `{ name }` | |
| Selected horse, wagon or boat | all three (`selected` column) | none | per-character selection table |
| XP, health, stamina, condition, fuel | all three | none | consumer table keyed by asset UUID |
| Wrecked boat (condition 0) | boats | none (gameplay state, the boat is still owned) | condition |
| Owner limit (`maxPlayerWagons`, `maxPlayerHorses`) | all three | `ListAssetsByHolder` with `assetType` and `limit` = cap + 1 | the cap |
| Trade to another player | all three (`UPDATE charid`) | `TransferOwnership`, seller asserted | distance check, menus |
| Sell back to the shop | all three (`DELETE`) | `RetireAsset` | payout (Economy) |
| Permanent death | stables (`dead` flag) | `RetireAsset` | death detection |
| Saddlebags, wagon and boat storage | all three (`horse_<id>`, `wagon_<id>`) | none | feather-inventory container keyed by asset UUID |
| Portable canoe (an inventory item that spawns a boat) | boats | `RegisterAsset` with provenance `inventory_item` + the item UUID, retired when the item is consumed or lost | the item (feather-inventory owns it) |
| Impound, collateral | admin, future justice | status `held` | impound gameplay |
| Stolen | future theft | status `missing` (the owner does not change) | theft gameplay |
| Shared mounts, keys | future | delegation epoch (see section 5) | feather-access |
| Liens, insurance | none named | not in Contract 1 | |

Holder types for v1: `character` (UUID through the character-profile provider) and
`organization` (feather-organizations). A wainwright or boatman job wagon owned by a
business is an organization holder.

## 3. Key decisions (master plan "Key decisions")

| Question | Decision | Reason |
|---|---|---|
| Do horses and wagons share sufficient invariants for one service? | Yes: one service, types are configuration | Every row in section 2 maps to the same facts. The service has no type-specific code: `grep -n horse server/*.lua` finds only comments and the test model table. |
| Customization: generic metadata or type-specific provider state? | Generic versioned metadata: one bounded JSON object plus `metadataVersion` owned by the managing resource | Horse tack and wagon or boat names fit; nothing needs Assets to understand the content. |
| Stolen possession: ownership change, custody interest or gameplay fact? | Gameplay fact; the consumer may mark the asset `missing` | Theft does not move legal ownership; a recovered horse returns to its owner without a transfer. |
| Do liens or insurance justify first-release interest types? | No | No named consumer. Only `owner` exists; new interest types can be added as capability features later. |

## 4. Phases

| Phase | State |
|---|---|
| A0 Consumer inventory | Done (section 2) |
| A1 Narrow type (horses) | Done 2026-10-03 |
| A2 Ownership: registration, query, transfer, retirement | Done; release 9/9 and lifecycle 21/21 on 2026-10-13 |
| A3 Runtime adapters: stables | Done: bcc-stables registers, trades, sells and retires horses through Assets (offline flow test 2026-10-16, see `bcc-stables/FEATHER_PORT.md`) |
| A4 Second type | Done at the domain level: `wagon` and `boat` are enabled and the same lifecycle test passed 23/23 for each type on 2026-10-17 (`AssetsLifecycleSmokeTest <a> <b> wagon`, `... boat`). The bcc-wagons and bcc-boats ports are gameplay work and use only the exports listed in the contract. |
| A5 Access decision | Done: section 5 |

## 5. A5: delegation and the Access integration request (2026-10-14)

Repeated delegation needs exist (shared horses, wagon keys, boat crew), so delegation
belongs in feather-access, not in Assets. Assets supplies the target facts Access
asked for and keeps its own ownership path separate.

What Assets provides in Contract 1:

- **Target context** (trusted readers, server only): `GetAsset` returns the asset
  UUID, `assetType`, `status`, the owner interest (`interestId`, holder) and
  `delegationEpoch`; `CheckInterest` returns `matched`, `assetStatus` and
  `delegationEpoch`. Grantor authority is "is the current owner", answered by
  `CheckInterest`.
- **Delegation epoch:** a per-asset integer, stored on the asset row
  (migration 005). It starts at 1 and rises by exactly 1 in the same commit as
  `TransferOwnership` and `RetireAsset`. Status and metadata changes never change
  it, so a `held` or `missing` asset that returns to `active` keeps its epoch (the
  Access lifecycle matrix: "Resume: same-epoch grants may resume"). It never goes
  down and is never reused, including when an asset returns to a previous owner.
  `assets.ownership.transferred.v1` and `assets.asset.retired.v1` carry the new
  value; events are for reconciliation, not for authorization.
- **Lifecycle answers for the matrix:** owner change advances the epoch;
  `held` and `missing` assets should deny delegated use (Assets reports the status;
  Access or the consumer denies); `retired` is terminal and the asset UUID is never
  reused; a stopped or failed Assets returns `not_ready` or `dependency_unavailable`,
  which Access must treat as deny.

Not in Contract 1, waiting for Access Phase 0:

- An Access target-provider registration (`RegisterTargetProvider`) and the exact
  capability vocabulary (proposal: `asset.retrieve`). Capability feature
  `accessProvider` stays `0`.
- Grant preservation by the new owner. Default clearing works through the epoch
  now; a preservation workflow needs the new owner's authenticated consent per
  transfer and will be added as an optional feature.
- A reservation or fencing protocol for strict ordering between a consuming
  mutation and a concurrent transfer. Without it, the guarantee is: a check that
  starts after a committed transfer or retirement sees the new epoch.

Delegated use never grants transfer, retirement or management: those remain the
managing resource's exports under Assets policy.

## 6. Acceptance gates (master plan)

| Gate | Evidence |
|---|---|
| Duplicate registration and concurrent sale cannot create double ownership | Unique provenance per manager (`asset_exists`); unique active owner slot; lifecycle test "duplicate provenance" and "concurrent sale one winner"; release test "one owner per live asset" |
| Transfer is idempotent and fully audited | Receipts per caller and request ID; "idempotent replay", "idempotency conflict", "audit history" |
| Records survive entity deletion and restart | No entity or server ID is stored; all state is in MariaDB |
| Gameplay resources can be replaced without losing ownership truth | Consumers keep only gameplay rows keyed by asset UUID; ownership is read from Assets |
| A second type fits without type-specific conditionals | Section 3; the lifecycle test per type |

Server runs for 0.2.0 (offline, two character UUIDs):

| Test | Expect | Result |
|---|---|---|
| `AssetsReleaseContractSmokeTest` | 10/10 | Passed 10/10 on 2026-10-17 |
| `AssetsLifecycleSmokeTest <a> <b>` (horse) | 23/23 | Passed 23/23 on 2026-10-17 |
| `AssetsLifecycleSmokeTest <a> <b> wagon` | 23/23 | Passed 23/23 on 2026-10-17 |
| `AssetsLifecycleSmokeTest <a> <b> boat` | 23/23 | Passed 23/23 on 2026-10-17 |
| bcc-stables `StablesFlowTest` (regression) | 9/9 | Passed 9/9 twice on 2026-10-17 |
