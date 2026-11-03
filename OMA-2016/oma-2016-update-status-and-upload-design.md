# OMA-2016: appliance update status and image upload

Status: proposed design.

## Scope and ownership

[OMA-2016](https://redhat.atlassian.net/browse/OMA-2016) adds two agent endpoints:

- `GET /api/v2/update`: read update status.
- `PUT /api/v2/update/image`: upload and validate an image for a disconnected appliance.

Uploading prepares an image; it does not install it or restart the agent.

[OMA-3031](https://redhat.atlassian.net/browse/OMA-3031) owns
`POST /api/v2/update`, connected downloads, activation in both modes, persistence
of the auto-update preference, and automatic updates. It extends the service
introduced here instead of adding another preparation path.

[Migration Planner PR #1883](https://github.com/kubev2v/migration-planner/pull/1883)
provides host activation and outcome reporting. The host watches `updates/*.tar`;
publishing an archive there triggers installation. This design changes the
original OMA-2016 contract: successful upload must no longer rename an archive
into that watched directory. Update OMA-2016 and OMA-3031 together to reflect this
boundary. The host watcher and its result format remain unchanged.

## GET /api/v2/update

Return `200 OK` with an `UpdateStatus` resource. Example for a disconnected
appliance with a prepared image:

```json
{
  "currentVersion": "v0.31.0",
  "connected": false,
  "autoUpdate": null,
  "updateAvailable": null,
  "latestVersion": null,
  "state": "ready",
  "preparedImage": {
    "version": "v0.32.0",
    "digest": "sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
  },
  "lastResult": null,
  "lastError": null
}
```

### Field retrieval and persistence

GET assembles the response from runtime state and persisted data. It does not
contact the SaaS, inspect an OCI archive, or hash image layers on each request.

| Field | Meaning | Storage/source | Work performed on GET | Restart behavior |
| --- | --- | --- | --- | --- |
| `currentVersion` | Version of the currently running agent. | Version configuration supplied by the running agent image; same source as `/api/v2/version`. | Read the configured value. Historical host results must not override it. | Supplied again by the running image; not restored from historical update metadata. |
| `connected` | Whether the agent is configured for connected mode, regardless of temporary connection failures. | Existing `AgentMode` setting in the main database. | Read the configured mode through the console service. | The persisted mode survives restart. |
| `autoUpdate` | Nullable boolean controlling automatic future updates; relevant only in connected mode. | OMA-2016 uses `false` when connected and `null` when disconnected. OMA-3031 adds a preference in the main database. | Return the mode-appropriate default; after OMA-3031, read the preference. | The preference survives restart once implemented by OMA-3031. |
| `updateAvailable` | Nullable boolean indicating whether SaaS reports an available update. | Latest successful SaaS availability response, cached in memory by OMA-3031. | Read the cached value; return `null` when disconnected or unknown. OMA-2016 always returns `null`. | Cache is cleared; wait for a fresh SaaS response before reporting availability. |
| `latestVersion` | Nullable latest version reported by SaaS. | Same in-memory SaaS response as `updateAvailable`. | Read the cached version; return `null` when disconnected or unknown. OMA-2016 always returns `null`. | Cache is cleared; wait for a fresh SaaS response. |
| `state` | Current update operation state, described below. | Current `UpdateService` operation state in memory. | Read a snapshot protected by a mutex. | Reconstructed from the recovered candidate; OMA-3031 adds recovery for activation in progress. |
| `preparedImage` | Nullable version and OCI manifest digest of a complete, validated candidate; no internal filesystem paths are exposed. | `.staging/prepared.json` and its referenced archive, loaded into memory after preparation or startup recovery. | Read the cached version and manifest digest. | Persisted metadata and archive survive; verify them during startup before reporting `ready`. |
| `lastResult` | Nullable outcome of the last host update attempt. | Host-owned `<DataFolder>/updates/status.json`. | Read and parse the file on every GET. | The persistent file survives restart. |
| `lastError` | Nullable string describing the latest preparation failure; cleared when a new preparation begins. Host failures belong in `lastResult`. | Latest preparation error in memory. | Read the cached error. | Previous preparation errors are lost; startup recovery can set a new error if the persisted candidate is invalid. |

All fields are present; unavailable values use explicit JSON `null`. An uploaded
candidate does not set `updateAvailable`: that field describes connected-mode
SaaS availability only.

The service updates `state` as preparation progresses; GET does not infer it by
scanning files. Startup recovery verifies the referenced prepared archive once
before exposing it as ready and removes interrupted uploads. GET does not repeat
that potentially expensive validation.

Read `lastResult` from disk on each GET because the host updates it independently
of the agent. The host publishes the file atomically, so readers see a complete
record without requiring an agent-side file watcher. Missing and malformed-file
behavior is specified below.

`connected` describes configured operating mode, not current network reachability.
In OMA-3031, availability remains the last successfully reported value during a
temporary connection failure and can therefore become stale. Clear availability
on a transition to disconnected mode; a subsequent transition to connected mode
starts with unknown availability until a fresh response arrives. As a follow-up
for OMA-3031, consider `lastCheckedAt` so the UI can show the age of the cached
availability. That field is not part of the OMA-2016 response contract.

### Operation state and host outcomes

OMA-2016 produces `idle`, `uploading`, `validating`, and `ready`. Reserve
`downloading` and `applying` in the schema for OMA-3031. The normal upload flow is:

```text
idle or ready -> uploading -> validating -> ready
```

On failure, return to `ready` if a previous candidate remains valid, otherwise
`idle`. Keep the failure in `lastError`. A previously prepared image can remain
visible while its replacement is uploading or validating; `state` identifies
the active operation.

`lastResult` preserves the host contract, including `file`, `result`, `time`,
`error`, `error_stage`, `running_image`, and `last_successful_update`. Results are
`applied`, `rolled-back`, `failed`, or `unchanged`. Accept older records that
contain only `file`, `result`, and `time`, and ignore unknown fields. A missing
file means `lastResult: null`; an unreadable or malformed file returns `500`
and is logged, rather than silently hiding a host-reporting problem.

An `applied` result confirms host restart/readiness, not database recovery or
the readability of every saved collection. GET has no preparation or activation
side effects and does not contact the SaaS.

## PUT /api/v2/update/image

Accept `multipart/form-data` containing exactly one `file` part. The original
filename is not trusted and does not determine the candidate version or path.
This endpoint is available only in disconnected mode.

Stream the file into persistent storage, validate it, and return `200 OK` with
`UpdateStatus` and `state: "ready"` after preparation completes. The response
confirms preparation only. Validation remains part of the request; a cancelled
request cancels preparation and removes the incomplete candidate.

Use streaming multipart parsing rather than `FormFile`, which may spool a large
upload to a separate temporary filesystem before service validation begins.
Limit the file payload to 3 GiB, with a separate bounded allowance for multipart
headers and boundaries. Reject extra file parts, unexpected fields, and malformed
multipart input. Check the payload limit while reading, even without a declared
content length.

| HTTP status | Condition |
| --- | --- |
| `200` | Image validated and prepared; no activation occurred. |
| `400` | Missing/invalid multipart input, invalid OCI image, missing version label, digest mismatch, wrong architecture, or unsupported upgrade. |
| `409` | Connected mode, collection/inspection running, or another update operation holding the reservation. |
| `413` | File payload exceeds 3 GiB, or multipart framing exceeds its limit. |
| `507` | Insufficient data-folder space, including an out-of-space write failure. |
| `500` | Unexpected storage or internal failure. |

Use the existing API error shape and typed domain errors with constructor and
checker helpers. Apply the existing authentication middleware to both routes.
Customer enablement remains blocked by OMA-2934, as required by the ticket.

## Preparation service and persistent files

Introduce `UpdateService` with one shared preparation entry point:

```go
Prepare(ctx context.Context, r io.Reader, size int64, expectedDigest string) (PreparedImage, error)
Status(ctx context.Context) (UpdateStatus, error)
```

`size` is the archive payload size, or unknown; never trust it instead of counting
bytes. Upload passes an empty expected digest. OMA-3031 will pass the downloaded
image's expected OCI manifest digest after normalizing its ETag. This must be the
manifest digest, not a hash of the transport tar file.

Use the persistent data filesystem for both temporary and prepared files:

```text
<DataFolder>/updates/
  .staging/
    <uuid>.part       # incomplete upload, never eligible for activation
    <uuid>.tar        # validated candidate, still outside the host watch glob
    prepared.json    # atomically published candidate metadata and internal filename
  status.json        # host-owned last outcome
  <version>.tar      # activation handoff; written only by OMA-3031
```

Preparation proceeds as follows:

1. Reserve the update operation non-blockingly and check mode and active work.
   Return `409` on conflict. Hold the reservation through cleanup/publication;
   protect status snapshots separately from long-running file I/O.
2. Create `.staging/` with mode `0750` and a unique `.part` file with mode `0600`.
3. Check free space on the data filesystem. Require at least three times the
   archive size available at the start of preparation. For unknown sizes, enforce
   the equivalent budget incrementally: initial available bytes must cover three
   times bytes received, with current-space checks during streaming. Count bytes
   and handle write-time `ENOSPC` regardless of the preflight check.
4. Stream with context cancellation and the payload limit, sync, and close the
   file before validation. Reject a declared-size mismatch.
5. Validate the archive and upgrade compatibility.
6. Rename the validated file to its unique `.staging/<uuid>.tar` path. Publish
   `prepared.json` using a synced temporary file and atomic rename, then sync the
   containing directory. Metadata includes version, manifest digest, actual byte
   size, and internal candidate filename.
7. Once the new candidate is committed, remove the previous candidate. Failures
   before commit remove only the new files and preserve the previous candidate.

Always remove partial files on failure. A publication failure must never expose
the new image as prepared. Failure to remove an obsolete candidate after commit
is logged and retried during recovery; it does not invalidate the new candidate.

At startup, recover only the candidate referenced by valid metadata. Verify that
its archive exists and revalidate its contents before reporting `ready`. Remove
orphaned `.part` files and unreferenced candidate archives. Invalid metadata or a
missing/corrupt candidate produces `idle` with `lastError`; never infer readiness
from an arbitrary tar file. Preparation errors are retained in memory; surviving
restarts is required for the prepared candidate and host outcome, not an upload
error history.

## OCI validation

Inspect the archive without extracting its entries into the filesystem:

- Require a valid OCI layout, index, selected manifest, config, and all referenced
  layer blobs. Verify descriptor sizes and content digests, including layers.
- Konflux currently publishes Docker schema-2 images. Export the appliance with
  `skopeo copy --override-os linux --override-arch <appliance-arch> --format oci
  docker://<image-reference> oci-archive:<filename>` before uploading. The outer
  archive must be an OCI layout, with OCI manifest/config/layer media types;
  `docker save` / `docker-archive` is not the upload format. The prepared digest
  identifies the exported OCI manifest; format conversion can change it from
  the registry's Docker manifest digest. Any expected digest must refer to that
  exported manifest.
- Support a single runnable image for the appliance platform. Reject ambiguous
  multi-image indexes and a mismatched OS/architecture instead of selecting an
  arbitrary first entry.
- Read `org.opencontainers.image.version` and optional
  `min-upgrade-version` from the image config labels. This is the exact label
  emitted by `agent/Containerfile` using `build-config/min-upgrade-version`.
- Require a nonempty version. Derive the eventual host filename using the shared
  `[A-Za-z0-9._-]` sanitization rule; reject an empty result or a reserved dot name.
  The displayed version remains the original label value.
- Enforce the minimum using the environment-specific compatibility rules below.
  Never silently skip a supplied, nonempty minimum because a version is not
  semver. An omitted or empty minimum remains optional for older images.
- Compare a provided expected digest with the selected manifest digest.
- Bound archive entry count, metadata sizes, and total retained metadata. Stream
  layer hashing with bounded memory; do not keep every small blob in a map.
  Reject duplicate/conflicting metadata entries and unsafe archive paths.

### Minimum-version comparison

The running version comes from the agent's configured appliance version. The
minimum comes from the uploaded image's label, not from the candidate version.
Calculate compatibility during upload and startup recovery; GET uses the cached
prepared result and does not repeat the comparison.

| Environment | Accepted running/minimum format | Comparison |
|-------------|---------------------------------|------------|
| Production | `v<major>.<minor>.<patch>` | Compare numeric semver components; accept equality or a newer version. Prereleases, build suffixes, and leading zeros are not canonical build versions. |
| Dev | `main-release-<major>.<minor>-<sha>` | Compare major/minor numerically within `main`. Equal or newer releases are accepted regardless of SHA; older releases are rejected. |
| Stable (stage) | `stable-release-<major>.<minor>-<sha>` | Apply the dev rule within `stable`. |

For dev/stable, numeric components must have no leading zeros and SHAs must be
7–40 lowercase hexadecimal characters, matching the Containerfile contract.
Different channels are not comparable. Reject malformed minimums, unknown
running versions such as `DEV` or `on-pr-<sha>`, and incompatible version formats
with `400` and an explanation when a minimum is supplied.

Dev/stable compatibility is defined at release granularity. SHAs are validated
for format but ignored in the comparison; different SHAs in the same release
pass. For example, both `main-release-0.32-1e77135` and
`main-release-0.31-fffffff` meet minimum `main-release-0.31-212e974`.
`main-release-0.30-fffffff` fails. No commit ordering or ancestry check is needed.

Tests include a committed archive generated by the actual skopeo OCI exporter,
with platform and update labels captured from a published Konflux appliance.
Its filesystem layer is replaced with a tiny valid fixture layer. This exercises
the real exported layout, gzip layer descriptors, digests, and label contract
without downloading the full appliance during tests. It does not establish that
the complete appliance boots. Table tests cover production/dev/stable ordering,
the minimum boundary, different SHAs within the same release, invalid versions,
and channel mismatches.

These checks establish integrity and structural compatibility. Digest validation
alone does not authenticate the publisher; authentication/release trust policy
must not be represented as implemented by this ticket.

## Contract for OMA-3031

OMA-2016 implements preparation and status only. OMA-3031 adds activation to the
same service, with these requirements:

- Disconnected `POST /update` activates the prepared image. Connected POST
  downloads and calls the same `Prepare()` before activation.
- Activation verifies that the candidate still matches its metadata, persists
  the accepted operation, and atomically renames the archive to
  `updates/<sanitized-version>.tar` on the same filesystem. Never overwrite an
  archive being processed by the host.
- Upload, download, preparation, and activation use the same reservation. It
  remains held through host processing and is reconciled on startup. Mode
  changes and destructive data operations must not race with these operations.
- The host outcome is correlated with the pending activation; an old
  `status.json` must not finish a new attempt. OMA-3031 defines durable attempt
  tracking and recovery before publishing an archive.
- Activation and job starts share an operation gate. A one-time busy check does
  not prevent a collection or inspection from starting just before restart.
  Upload preparation itself never authorizes a later restart.
- Auto-update defaults to false, persists across restart, applies only in
  connected mode, and uses this same preparation/activation flow.
- Adding the preference must preserve it in existing configuration writes:
  console startup and `SetMode()` currently save configuration containing only
  `AgentMode` and must not inadvertently reset `autoUpdate`.

No `POST /update`, settings mutation endpoint, connected download, automatic
activation, or auto-update database migration is implemented in OMA-2016.

## Implementation and validation

| Area | Changes |
| --- | --- |
| API | Add paths and schemas in `agent/api/v2/openapi.yaml`; run `make generate`. |
| Models | Add status, prepared-image, and backward-compatible host-result models in `internal/models/update.go`. |
| Service | Add preparation/status in `internal/services/update.go` and archive validation in `update_archive.go`. |
| Wiring | Add service access through `ServiceManager` and `ServiceProvider`; construct it in `agent/app/app.go`. Apply equivalent handler coverage in RVTools mode. |
| Handlers | Add thin GET/PUT handlers in `agent/handlers/v2/update.go`; leave validation and filesystem work in the service. |
| Errors | Add typed update conflict, invalid-image, payload-size, and insufficient-storage errors in `pkg/errors`. |

Test successful preparation, every rejection above, unknown-length uploads,
cancellation, concurrent calls, bounded archive parsing, replacement failure,
and restart recovery. Test nullable fields, configured mode versus connectivity,
and old/new/missing/malformed host results. Use temporary directories and injected
filesystem/free-space seams; follow the repository's Ginkgo conventions.

The central acceptance test is that a successful upload reports `ready` while
leaving **no `updates/*.tar` installation trigger**, and every failed upload
preserves any previously prepared candidate. Include an integration check with
the host watcher confirming that preparation causes no restart.

Run `make generate`, `make validate-all`, `make test`, and `make build` for the
implementation. This document introduces no application changes.
