# Feather Assets

Authoritative registry of persistent, individually identifiable world assets
(horses, wagons, boats) and their ownership for the Feather Framework. Assets
owns asset identity, type, canonical model, owner interest, status, provenance,
lifecycle, versioned metadata, and the audit facts for all of them.

It does not own spawning, AI, stable or garage UI, feed, fuel, condition,
handling, XP, customization presentation, or theft gameplay. Those stay with the
gameplay resources (`bcc-stables`, `bcc-wagons`, `bcc-boats`), which call this
service for ownership truth. Native Feather; no legacy compatibility layer.

Implements phases A0–A2 of the Assets master plan (`feather-framework-docs`,
`feather-assets/Feather_Assets_Master_Plan.md`) on the Contract 1 service
kernel. The first A3 runtime adapter, `bcc-stables` 2.0.0, is implemented
and running on the development server; several live flows still need validation.
Wagon and boat adapters remain planned. A4 (second-type validation) happens during
the `bcc-wagons` port. A5 (Access decision) belongs to `feather-access`.

Shipped defaults use `Config.DevMode=false` with service-policy authorization
enabled. Development commands are not registered. The permanent server-console
`AssetsReleaseContractSmokeTest` is read-only.

## Status

| Gate | State |
|---|---|
| Static check (lua-language-server, error level) | Passed 2026-10-03 |
| `AssetsReleaseContractSmokeTest` (9 checks) | Passed 9/9 on the server 2026-10-13 |
| `AssetsLifecycleSmokeTest` (21 checks, DevMode) | Not yet run on the server |
| First consumer (`bcc-stables` Feather port) | Registration and metadata revision passed live 2026-10-03; transfer and retirement not yet exercised (see `bcc-stables/FEATHER_PORT.md`) |

## Dependencies

```text
feather-mysql
feather-core
```

At runtime, character holders resolve through the `character-profile` provider
(feather-character). Organization holders are validated through
feather-organizations, which lists `feather-assets` in its `trustedReaders`.
Service policy is decided by the default Core policy provider (feather-admin).

## Startup order

Start Assets after Core, Character, Organizations and Authority (and Jobs when
installed), and before Admin and asset consumers:

```text
ensure feather-mysql
ensure feather-core
ensure feather-character
ensure feather-organizations
ensure feather-authority
ensure feather-assets
ensure feather-admin
```

A fresh install applies four checksummed migrations and logs
`event=startup.ready migrationsApplied=4 routes=1`. Later starts apply none.
Migration `004_asset_event_references` adds the event reference columns.

## Installation and first acceptance

Place in `resources/[feather]/feather-assets`, add it to `server.cfg` in the
order above, restart feather-admin (to load the `['feather-assets']` policy
grants) and feather-organizations (to load the reader trust), then run in the
server console:

```text
refresh
ensure feather-assets
AssetsReleaseContractSmokeTest
```

Expect 9/9 with the shipped configuration.

The mutating acceptance test needs `Config.DevMode=true` on an isolated
development server. Enable it, restart feather-assets, and run with two
different characters (server IDs of loaded players, or character UUIDs):

```text
AssetsLifecycleSmokeTest <serverId|characterUuid> <otherServerId|otherCharacterUuid>
```

Expect 21/21. The test mutates data: it registers a uniquely provenanced smoke
horse for the first character, races two concurrent transfers to the second
(exactly one must win), revises metadata and status, and retires the horse. Set
DevMode back to `false` and restart feather-assets afterwards.

If the first mutation returns `authorization_denied` with
`policyCode=service_forbidden`, restart feather-admin so it loads the
`['feather-assets']` service-policy grants.

## A0 consumer inventory

The original VORP-era BCC resources (BryceCanyonCounty upstream) stored assets
in their own tables, keyed by the VORP `charid` integer. This inventory describes
the legacy starting point; the Feather stables port now uses Assets UUIDs:

| Resource | Table | Ownership fields | Type-specific state |
|---|---|---|---|
| `bcc-stables` | `player_horses` | `identifier`, `charid` | `name`, `components`, `gender`, `xp`, `captured`, `born`, `health`, `stamina`, `writhe`, `dead`, `selected` |
| `bcc-wagons` | `player_wagons` | `identifier`, `charid` | `name`, `condition`, `selected` |
| `bcc-boats` | `boats` | `identifier`, `charid` | `name`, `fuel`, `condition` |

All three share the same core: one owner, one model, one display name. That is
what Assets owns. The split for the ports:

| Field | Owner after the port |
|---|---|
| Asset identity, type, `model` | Assets (`assetId`, `assetType`, `modelKey`) |
| `charid` / `identifier` | Assets owner interest (`holderType='character'`, character UUID) |
| `name`, `components`, `gender`, coat | Assets metadata (consumer-versioned JSON) |
| `dead` | Assets `RetireAsset` (reason such as `stables.horse_died`) |
| `xp`, `health`, `stamina`, `writhe`, `condition`, `fuel`, `selected` | The gameplay resource's own table, keyed by `assetId` |

## Ownership and decisions

The master plan's key decisions, as implemented:

| Decision | Choice |
|---|---|
| One service for horses and wagons? | Yes. The A0 inventory shows identical ownership invariants. Types are configuration (`Config.AssetTypes`); the base service has no type-specific branches. |
| Customization | Generic versioned metadata: one bounded JSON object per asset plus a consumer-owned `metadataVersion`. |
| Stolen possession | Gameplay-only fact in this release. Theft does not change ownership; a consumer may mark an asset `missing`. |
| Liens and insurance | Deferred. Only the `owner` interest type is supported until a real consumer needs more. Delegated use (keys, shared mounts) belongs to `feather-access`. |

Further rules:

- **One owner.** A unique active slot per asset and interest type makes double
  ownership impossible at the database level.
- **Duplicate registration.** The managing resource's provenance
  (`provenanceType` + `provenanceId`) is unique. Registering the same purchase,
  capture or legacy row twice returns `asset_exists` with the existing
  `assetId`.
- **Concurrent sale.** Transfers lock the asset row and require
  `expectedRevision`. Of two concurrent sales, one commits and the other gets
  `revision_conflict`.
- **Statuses.** `active`, `held` (impounded, collateral) and `missing` are set
  with `ChangeAssetStatus`. `retired` is terminal and only set by `RetireAsset`,
  which also ends the owner interest. Only an `active` asset can change owner.
- **Managing resource.** The registering resource manages the asset and is the
  only caller (besides privileged managers) that may mutate it.
- **Survives entity deletion.** Records never reference a runtime entity or
  server source ID.
- **Offline owners.** Allowed. Character UUIDs resolve through the Character
  provider whether or not the character is online.

## Server exports

All exports return the Contract 1 envelope `{ ok = true, value = ... }` or the
flat `{ ok = false, code, message, details }`. The caller is the actual Cfx
invoking resource and must be listed in the matching `Config.Access` group. No
client may supply a trusted resource, account, or character identity.

`GetCapabilities()`, `GetHealth()`, and `AwaitReady(timeoutMs)` (default 30000,
integer 0–60000) report lifecycle.

Every mutation takes a stable `requestId` (at most 128 characters) and a
namespaced `reasonCode` such as `stables.purchase`. It also accepts an optional
`actorSource` (the player who triggered it; Assets derives that player's account
and character from Core), an optional `correlationId`, and an optional
`referenceType` + `referenceId` pair naming the consumer record behind the
change (for example `stables_order` and the order UUID). The reference is stored
on the audit event and published with it, so a workflow can be traced across
Assets and Economy. Retrying a request ID
with the same payload replays the stored result with `replayed = true`. Reusing
it with a different payload returns `idempotency_conflict`.

### Mutations

| Export | Fields | Policy action |
|---|---|---|
| `RegisterAsset` | `assetType`, `modelKey`, `holderType`, `holderId`, `provenanceType`, `provenanceId`, optional `metadataVersion`, `metadata` | `assets.asset.manage` |
| `TransferOwnership` | `assetId`, `expectedRevision`, `toHolderType`, `toHolderId`, optional `fromHolderType` + `fromHolderId` | `assets.ownership.transfer` |
| `ChangeAssetStatus` | `assetId`, `expectedRevision`, `status` (`active`, `held`, `missing`) | `assets.asset.manage` |
| `UpdateAssetMetadata` | `assetId`, `expectedRevision`, `metadataVersion`, `metadata` (object, or `false` to clear) | `assets.asset.manage` |
| `RetireAsset` | `assetId`, `expectedRevision` | `assets.asset.manage` |

The caller becomes the managing resource on `RegisterAsset`. Every mutation
returns the full asset snapshot with its new `revision`.

### Reads

| Export | Fields | Returns |
|---|---|---|
| `GetAsset` | `assetId` | Asset snapshot with `owner` and `metadata` |
| `ListAssetsByHolder` | `holderType`, `holderId`, optional `assetType`, `status`, `includeMetadata`, `limit`, `cursor` | `{ items, nextCursor }`, newest first |
| `CheckInterest` | `assetId`, `holderType`, `holderId`, optional `interestType` (`owner`) | `{ matched, interestId?, assetType, assetStatus }` |
| `FindOperation` | `requestId` | The caller's stored result, for timeout recovery |
| `InspectAssetHistory` | `assetId`, optional `limit`, `cursor` | Audit trail; managers read their own assets, privileged auditors read all |

`CheckInterest` returns facts only. The consuming resource still decides
whether those facts permit an action; a `held` horse is still owned, but a
stable may refuse to release it.

### Asset snapshot

```lua
{
    assetId = 'uuid',
    assetType = 'horse',
    modelKey = 'A_C_Horse_Arabian_White',
    status = 'active',
    managerResource = 'bcc-stables',
    provenanceType = 'stable_purchase',
    provenanceId = 'purchase-uuid',
    metadataVersion = 1,
    metadata = { name = 'Silver', gender = 'female', components = { ... } },
    revision = 3,
    createdAt = 1759500000000,
    updatedAt = 1759500100000,
    retiredAt = nil,
    owner = {
        interestId = 'uuid',
        holderType = 'character',
        holderId = 'character-uuid',
        revision = 1,
        validFrom = 1759500100000
    }
}
```

### Metadata limits

Metadata is an object of identifier keys whose values are strings, finite
numbers, booleans, arrays or objects. It is bounded by `Config.Limits`: 8192
encoded bytes, depth 4, 256 nodes, 1024-character strings. Violations return
`invalid_metadata`. Metadata bodies are never copied into events; only the
`metadataVersion` travels with the fact.

### Failure codes

| Code | Meaning |
|---|---|
| `authorization_denied` | Caller not trusted, not the manager, or policy denied |
| `policy_unavailable` | Policy provider missing or indeterminate (Assets fails closed) |
| `revision_conflict` | Stale `expectedRevision`; `details.revision` holds the current value |
| `idempotency_conflict` | Request ID reused with a different payload |
| `asset_exists` | Provenance already registered; `details.assetId` |
| `asset_not_found`, `asset_retired`, `asset_unavailable` | Missing, terminal, or not `active` for a transfer |
| `holder_mismatch` | `fromHolder*` is not the current owner |
| `holder_not_found`, `holder_inactive`, `holder_type_not_allowed`, `holder_provider_unavailable` | Holder validation |
| `invalid_metadata`, `invalid_input`, `no_change` | Request validation |
| `transaction_failed` | Retry with the same request ID, or call `FindOperation` first |

## Contract

```lua
local Assets = exports['feather-assets']

-- Purchase: the stable has taken payment and stored a pending purchase record.
local registered = Assets:RegisterAsset({
    requestId = ('purchase:%s'):format(purchaseId),
    assetType = 'horse',
    modelKey = 'A_C_Horse_Arabian_White',
    holderType = 'character',
    holderId = characterId,
    provenanceType = 'stable_purchase',
    provenanceId = purchaseId,
    metadataVersion = 1,
    metadata = { name = 'Silver', gender = 'female' },
    reasonCode = 'stables.purchase',
    actorSource = source
})

-- Sale between players: assert the seller so a stale view is rejected.
local asset = registered.value
Assets:TransferOwnership({
    requestId = ('sale:%s'):format(saleId),
    assetId = asset.assetId,
    expectedRevision = asset.revision,
    fromHolderType = 'character',
    fromHolderId = sellerCharacterId,
    toHolderType = 'character',
    toHolderId = buyerCharacterId,
    reasonCode = 'stables.sale',
    actorSource = source
})

-- Before spawning: facts only, the stable decides.
local check = Assets:CheckInterest({
    assetId = assetId,
    holderType = 'character',
    holderId = characterId
})
```

## Client route

`assets.owned.list.v1` takes an empty payload or `{ assetType = 'horse' }` and
returns the calling player's own active assets (`assetId`, `assetType`,
`modelKey`, `status`), at most `Config.Limits.maxClientAssets`. It excludes
manager, provenance and metadata. It requires an active character and is rate
limited. Clients cannot mutate anything.

## Event publication

Committed facts are written to the `feather_asset_events` audit table and the
`feather_asset_outbox` in the same transaction as the change, then published
through Core:

```text
assets.asset.registered.v1
assets.ownership.transferred.v1
assets.status.changed.v1
assets.metadata.revised.v1
assets.asset.retired.v1
```

Payloads carry `assetId`, `assetType`, the current holder, `revision`, the
source resource, request, reason, actor and correlation fields, plus
event-specific details (previous holder, previous status, metadata versions).
They are bounded to 4096 bytes.

## Configuration and access

| Setting | Purpose |
|---|---|
| `Config.AssetTypes` | Registrable types and the holder types each accepts. Keys are stable; never rename one once assets exist. Only `horse` is enabled; `wagon` is enabled with the bcc-wagons port (A4), then `boat`. |
| `Config.HolderTypes` | `character` (Character provider), `organization` (feather-organizations); `validator = false` trusts the manager. |
| `Config.Limits` | List page size, client route bound, metadata bounds. |
| `Config.Authorization` | Policy actions `assets.asset.manage` and `assets.ownership.transfer`. |
| `Config.Access` | `trustedReaders`, `trustedManagers` (with `privilegedManagers`), `trustedAuditors` (with `privilegedAuditors`). A privileged entry must also be in its trusted group. |

Each consumer that mutates assets additionally needs its policy grants in
`Config.servicePolicy['feather-assets'][<resource>]` of feather-admin. Restart
feather-admin after changing it.

Current trusted resources are `feather-assets`, `feather-admin` and `bcc-stables`
in the reader, manager and auditor groups. Only feather-admin is privileged.
The development server's feather-admin configuration grants `bcc-stables` both
Assets policy actions; trust entries alone do not grant mutation permission.
Add `bcc-wagons` and `bcc-boats` trust and policy entries as each port lands.
Read-only consumers need reader trust, without mutation policy grants.

## Database access

Only Assets writes `feather_asset_records`, `feather_asset_interests`,
`feather_asset_receipts`, `feather_asset_events`, `feather_asset_outbox`, and
`feather_asset_schema_migrations`. Consumers must use the exports above and
never query these tables. Applied migrations are checksummed and must not be
modified; a mismatch fails readiness.

## Remaining work

| Phase | Work |
|---|---|
| A3 runtime adapters | `bcc-stables` 2.0.0 implements registration, ownership checks, metadata, transfer and retirement, with gameplay state keyed by `assetId`. Purchase, tack and saddlebags passed live on 2026-10-03. Validate the remaining flows, including trade, capture and retirement; see the port's `FEATHER_PORT.md`. |
| A4 second type | Port `bcc-wagons` and confirm wagons fit without type-specific branches before freezing Contract 1. Then `bcc-boats`. |
| Legacy migration | Needed only for servers with legacy data; no import ships with the stables port because this development server had no `player_horses` table. An import must map `charid` to character UUIDs and register each legacy row with `provenanceType = 'legacy_<table>'`, `provenanceId = <old id>` (idempotent by provenance). |
| A5 access decision | Keys, shared mounts and organization-pool use go to `feather-access`, not new interest types here. |
