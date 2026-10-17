# Feather Assets: Contract 1

**Status:** proposed for freezing with feather-assets 0.2.0 (2026-10-17).

This document is the public contract of feather-assets: what other resources may
call and rely on. Once frozen, everything listed here keeps working, with the
same meaning, for as long as `Contract = 1`. The README explains the model and
setup; this file is the reference a consumer codes against.

## 1. Compatibility promise

Within Contract 1:

- **May be added** without a new contract version: optional request fields,
  result fields, exports, events, payload fields, error codes, capability
  features, configured asset types and holder types. Consumers must ignore
  unknown result fields and treat an unknown error code as a failure.
- **Never changes**: the name, required fields, result shape or meaning of
  anything below; an error code's meaning; an event's name or the meaning of
  its fields; the policy action names; idempotency, revision and delegation
  epoch rules.
- **Needs Contract 2**: removing or renaming anything, making an optional field
  required, or changing what a result or event means.

Consumers check `GetCapabilities().value.contract == 1` and the features they
use before relying on them.

## 2. Lifecycle

| Export | Result |
|---|---|
| `GetCapabilities()` | `{ resource, contract = 1, version, state, assetTypes, interestTypes, features }` |
| `GetHealth()` | `{ state, phase, ... }` (diagnostic; fields may be added) |
| `AwaitReady(timeoutMs)` | The health snapshot once ready; `not_ready` after the timeout, `startup_failed` if startup failed. `timeoutMs` 0 to 60000, default 30000 |

Features in Contract 1 (value `1`): `lifecycle`, `health`, `migrations`,
`registration`, `ownership`, `transfer`, `status`, `metadata`, `retirement`,
`interestCheck`, `revisions`, `idempotency`, `operationLookup`, `outbox`,
`eventPublication`, `auditHistory`, `references`, `holderValidation`,
`clientSnapshot`, `delegationEpoch`. `accessProvider`, `liens`, `custody` and
`legacyMigration` are `0`.

## 3. Calling rules

- **Results.** Every export returns `{ ok = true, value = ... }` or
  `{ ok = false, code, message, details }`. `message` is for logs; branch on
  `code`.
- **Caller.** The calling resource is taken from Cfx (`GetInvokingResource`);
  it must be in the matching `Config.Access` group. No request field can claim
  another resource, account or character.
- **Common mutation fields.** `requestId` (required, 1 to 128 characters,
  `^[A-Za-z0-9][A-Za-z0-9._:-]*$`), `reasonCode` (required, namespaced,
  `^[a-z][a-z0-9._:-]*$`, at most 64), optional `actorSource` (the connected
  player who triggered it; Assets derives the account and character from Core),
  optional `correlationId` (same pattern as `requestId`), optional
  `referenceType` (`^[a-z][a-z0-9_]*$`, at most 48) + `referenceId` (same
  pattern as `requestId`), both or neither: the caller's record behind the
  change. Unknown fields are rejected.
- **Idempotency.** A request ID is scoped to the calling resource. Repeating it
  with the same payload returns the stored result with `replayed = true`;
  reusing it with a different payload returns `idempotency_conflict`. A replay
  returns the snapshot as it was stored when the request first succeeded. After
  a timeout, retry with the same request ID or call `FindOperation`.
- **Revisions.** Every change to an existing asset takes `expectedRevision`, the
  asset's current `revision`. A stale value returns `revision_conflict` with
  `details.revision`. Every successful change returns the full asset snapshot
  with its new revision.
- **Manager.** The resource that registers an asset manages it; only it, or a
  privileged manager, may change the asset.
- **Policy.** Mutations are also checked through Core policy, with the caller as
  `subject.resource`:
  - `assets.asset.manage`: register, status, metadata, retire;
  - `assets.ownership.transfer`: transfer.

  A missing or failing policy provider denies (`policy_unavailable`).

## 4. Mutations

Callers must be in `trustedManagers`.

| Export | Fields (besides the common ones) |
|---|---|
| `RegisterAsset` | `assetType`, `modelKey`, `holderType`, `holderId`, `provenanceType`, `provenanceId`, optional `metadataVersion` (default 1), `metadata` |
| `TransferOwnership` | `assetId`, `expectedRevision`, `toHolderType`, `toHolderId`, optional `fromHolderType` + `fromHolderId` |
| `ChangeAssetStatus` | `assetId`, `expectedRevision`, `status` (`active`, `held`, `missing`) |
| `UpdateAssetMetadata` | `assetId`, `expectedRevision`, `metadataVersion`, `metadata` (object, or `false` to clear) |
| `RetireAsset` | `assetId`, `expectedRevision` |

Field formats: `assetType` is a configured type key; `modelKey` is a game model
name, `^[A-Za-z0-9][A-Za-z0-9_]*$`, at most 100; `holderId`, `assetId` are
UUIDs (compared in lower case); `provenanceType` is `^[a-z][a-z0-9_]*$`, at most
48; `provenanceId` follows the `requestId` pattern; `metadataVersion` is an
integer 1 to 65535 owned by the managing resource.

Rules that are part of the contract:

- An asset starts `active`, at revision 1 and delegation epoch 1, with one owner
  interest for the given holder.
- Provenance is unique per managing resource: registering the same
  `provenanceType` + `provenanceId` again returns `asset_exists` with
  `details.assetId`.
- An asset has exactly one owner while it is not retired. Only an `active` asset
  changes owner (`asset_unavailable` with `details.status` otherwise). With
  `fromHolder*`, the transfer is refused (`holder_mismatch`) unless that holder
  is the current owner. Transferring to the current owner returns `no_change`.
- `held` (impounded, collateral) and `missing` (stolen, lost) keep the owner.
  Setting the status the asset already has returns `no_change`.
- `retired` is terminal and is set only by `RetireAsset`. It ends the owner
  interest in the same commit. Every later mutation returns `asset_retired`.
- `asset_type`, `model_key`, the managing resource and provenance never change
  after registration.
- Holders are validated: characters through the `character-profile` provider,
  organizations through feather-organizations (per `Config.HolderTypes`), and
  the holder type must be allowed by the asset type. Offline holders are valid.
- **Delegation epoch.** `delegationEpoch` is a per-asset integer for delegated
  use (feather-access). It rises by exactly 1 in the same commit as
  `TransferOwnership` and `RetireAsset`, and never changes on status or metadata
  changes. It never decreases and is never reused, including when an asset
  returns to a previous owner. A grant bound to an older epoch must be treated as
  no longer effective.

### Metadata

An object with identifier keys (`^[A-Za-z_][A-Za-z0-9_]*$`, at most 64
characters) whose values are strings, finite numbers, booleans, arrays or
objects, within `Config.Limits` (shipped: 8192 encoded bytes, depth 4, 256
nodes, 1024-character strings). Anything else returns `invalid_metadata`.
Metadata is never copied into events.

## 5. Reads

Callers must be in `trustedReaders` (history: `trustedAuditors`).

| Export | Fields | Result |
|---|---|---|
| `GetAsset` | `assetId` | Asset snapshot with `metadata` |
| `ListAssetsByHolder` | `holderType`, `holderId`, optional `assetType`, `status` (`active`, `held`, `missing`), `includeMetadata`, `limit` (1 to 50, default 20), `cursor` | `{ items = { snapshot... }, nextCursor }`: assets the holder owns now, newest registration first; retired assets are never listed |
| `CheckInterest` | `assetId`, `holderType`, `holderId`, optional `interestType` (`owner`) | `{ matched, interestType, interestId?, assetType, assetStatus, delegationEpoch }` |
| `FindOperation` | `requestId` | `{ found = false }` or `{ found = true, operation, result }` for the caller's own request |
| `InspectAssetHistory` | `assetId`, optional `limit` (1 to 50, default 20), `cursor` | `{ items = { event... }, nextCursor }`, newest first; a manager reads its own assets, a privileged auditor any |

`CheckInterest` returns facts only. The calling resource decides whether they
allow an action (for example, a `held` horse is still owned, but a stable may
refuse to release it).

`FindOperation` operations: `asset.register`, `ownership.transfer`,
`asset.status`, `asset.metadata`, `asset.retire`.

History items: `{ eventId, eventType, assetId, assetType, holderType, holderId,
sourceResource, requestId, reasonCode, referenceType, referenceId, revision,
actorAccountId, actorCharacterId, correlationId, details, createdAt }`, where
`eventType` is the short form (`asset.registered`, `ownership.transferred`,
`status.changed`, `metadata.revised`, `asset.retired`) and `details` holds the
event's extra fields (section 7).

### Asset snapshot

```lua
{
    assetId = 'uuid',
    assetType = 'horse',
    modelKey = 'A_C_Horse_Arabian_White',
    status = 'active',              -- active | held | missing | retired
    managerResource = 'my-stables',
    provenanceType = 'stable_purchase',
    provenanceId = 'order-uuid',
    metadataVersion = 1,
    metadata = { ... },             -- reads with metadata only
    revision = 3,
    delegationEpoch = 2,
    createdAt = 1759500000000,      -- Unix ms (second precision)
    updatedAt = 1759500100000,
    retiredAt = nil,
    owner = {                       -- absent once retired
        interestId = 'uuid',
        holderType = 'character',
        holderId = 'character-uuid',
        revision = 1,
        validFrom = 1759500100000
    }
}
```

## 6. Error codes

| Code | Meaning |
|---|---|
| `invalid_input` | A field is missing, unknown or malformed |
| `not_ready` | Assets is starting or stopped |
| `startup_failed` | `AwaitReady`: Assets failed to start; `details.health` |
| `dependency_unavailable` | Core is unavailable |
| `authorization_denied` | Caller not in the access group, not the manager, or policy denied |
| `policy_unavailable` | The policy provider is missing or failed (fails closed) |
| `actor_unavailable` | `actorSource` has no active character session |
| `idempotency_conflict` | Request ID reused with a different payload |
| `revision_conflict` | Stale `expectedRevision`; `details.revision` is current |
| `transaction_failed` | Commit not confirmed; retry the same request ID or call `FindOperation` |
| `asset_exists` | Provenance already registered by this manager; `details.assetId` |
| `asset_not_found` | No such asset |
| `asset_retired` | The asset is retired |
| `asset_unavailable` | Not `active` for a transfer; `details.status` |
| `no_change` | The asset already has that status or owner |
| `invalid_metadata` | Metadata is not a bounded JSON object |
| `holder_mismatch` | `fromHolder*` is not the current owner |
| `holder_not_found`, `holder_inactive`, `holder_type_not_allowed`, `holder_provider_unavailable` | Holder validation |
| `invalid_cursor` | The cursor is not from this listing |
| `character_required` | Client route: no active character |
| `invalid_persistence`, `internal_error` | Stored data is invalid, or an unexpected failure (report it) |

## 7. Events

Published through Core after commit, from an outbox, so each committed fact is
published even across restarts. Delivery is at least once: consumers deduplicate
by `eventId`. Server-only.

| Event | When | Extra fields |
|---|---|---|
| `assets.asset.registered.v1` | An asset is registered | `modelKey`, `status`, `interestId`, `provenanceType`, `provenanceId`, `managerResource`, `metadataVersion`, `delegationEpoch` |
| `assets.ownership.transferred.v1` | Ownership is transferred | `modelKey`, `interestId`, `previousInterestId`, `previousHolderType`, `previousHolderId`, `delegationEpoch` |
| `assets.status.changed.v1` | Status changes (not retirement) | `status`, `previousStatus` |
| `assets.metadata.revised.v1` | Metadata changes | `metadataVersion`, `previousMetadataVersion` |
| `assets.asset.retired.v1` | An asset is retired | `status` (`retired`), `previousStatus`, `previousInterestId`, `delegationEpoch` |

Every payload carries `eventId`, `assetId`, `assetType`, `holderType`,
`holderId`, `revision`, `sourceResource`, `requestId`, `reasonCode`,
`referenceType`, `referenceId`, `actorAccountId`, `actorCharacterId` and
`correlationId`. The holder is the owner after the change (for a transfer, the
new owner; for a retirement, the last owner). Payloads are at most 4096 bytes and
never carry metadata.

## 8. Client route

`assets.owned.list.v1`, payload `{ assetType? }`, returns
`{ assets = { { assetId, assetType, modelKey, status } } }`: the calling
player's own assets, at most `Config.Limits.maxClientAssets`, without manager,
provenance or metadata. It requires an active character and is rate limited (5
calls per 5 seconds, 128-byte payload). Clients cannot change anything.

## 9. Configuration

A server may change:

- the access groups (`trustedReaders`, `trustedManagers`, `privilegedManagers`,
  `trustedAuditors`, `privilegedAuditors`);
- limits (`Config.Limits`) and outbox timing;
- an asset type's label and the holder types it accepts (removing one affects
  only new registrations and transfers);
- add asset types and holder types;
- `Config.Authorization` action names only together with matching policy grants.

A server must not, once data exists:

- rename or remove an asset type or holder type key (they are stored on every
  record);
- edit an applied migration (a checksum mismatch fails startup).

Shipped asset types: `horse`, `wagon`, `boat`. Shipped holder types:
`character`, `organization`.

## 10. Not in Contract 1

Interests other than `owner` (liens, custody, insurance), an Access target
provider and its capability vocabulary, grant preservation on transfer, and a
fencing protocol for delegated use are not part of this contract. They would
arrive as new interest types, exports or capability features, announced through
`GetCapabilities`.
