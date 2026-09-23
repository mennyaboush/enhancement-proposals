---
title: netbox-inventory-backend
authors:
  - Menny Aboush
creation-date: 2026-09-10
last-updated: 2026-09-22
tracking-link: https://redhat.atlassian.net/browse/OSAC-4347
prd: prd.md
see-also: []
replaces: []
superseded-by: []
---

# NetBox Inventory Backend for Bare Metal as a Service

## Summary

This design adds NetBox as a pluggable inventory backend for bare-metal host allocation in OSAC, following the existing backend pattern. Cloud Infrastructure Admin configures the NetBox endpoint and credential material through Helm values; the chart mounts the token and optional CA as files and configures Metal3 management automatically. OSAC allocates hosts transparently from NetBox inventory without exposing backend details to tenants. See [PRD](prd.md) for detailed requirements.

## Motivation

OSAC's current inventory backends serve specific infrastructure patterns, but there are operators that manage their primary physical inventory in NetBox, a comprehensive infrastructure resource management system. Lack of NetBox integration forces operators to maintain a separate OSAC-specific inventory, creating data inconsistency and operational overhead during lifecycle changes (adding hosts, decommissioning, updating capability metadata).

The NetBox backend integrates into OSAC's existing pluggable inventory system (`inventory.Client` interface) without changing tenant-facing APIs or workflows. Cloud Infrastructure Admin configures the backend; allocation and deallocation remain transparent to tenants via standard `BareMetalInstance` API.

Deployment is in-tree (compiled into the operator) following established patterns for BCM and Metal3, with configuration exposed as Helm values that an existing Enclave Wizard pipeline may schema-validate. Direct Helm deployment remains supported. This approach avoids operational complexity of separate services while maintaining the option for future out-of-tree extraction per OSAC-3806.

### Goals

- Implement the `inventory.Client` interface for NetBox, following existing backend patterns.

- Prevent double-allocation of hosts via atomic assignment in NetBox with ETag-based optimistic locking (If-Match on PATCH; 412 Precondition Failed on conflict).
- Track allocation state in NetBox's native device status (`staged` available, `active` claimed), with `osac_instance_id` identifying the claiming BareMetalInstance for crash recovery and idempotency.
- Support `BareMetalInstanceType` host selection through pre-created NetBox custom fields, preserving selector keys and values and combining their equality filters server-side.
- Expose Helm configuration for the NetBox endpoint, credentials, and TLS certificates; an existing Enclave Wizard pipeline may schema-validate these values without exposing secrets in logs or error messages.
- Maintain tenant transparency — allocation/deallocation workflows identical across all backends; no NetBox-specific UX.

### Assumptions

- NetBox Community is an operator-managed external dependency. The integration uses its open-source REST API, custom fields, and conditional updates; no Enterprise feature, paid plugin, or direct database access is required. OSAC does not create devices or field definitions.
- The NetBox deployment permits OSAC to persist allocation state in the native device `status` and the `osac_instance_id` custom field. These are integration prerequisites, not fields created by OSAC at runtime.
- The existing OSAC API field `BareMetalInstanceType.host_label_selector` supplies the host-selection criteria. For NetBox, each key names a provider-created custom field on `dcim.device`, and each value specifies the required scalar value. The API field name remains unchanged; it does not mean that NetBox tags or Kubernetes labels are used for matching. The adapter adds only the REST query prefix `cf_`; it does not derive selectors from the instance type's hardware description or fetch the instance type during allocation.
- The API token is granted only the read/update permissions required for the documented device and custom-field operations.
- Enrolled device names meet the [host identity contract](#host-id-mapping): valid unchanged BMH names, unique in the operator-visible inventory, and aligned with the deployment's fabric hostnames. Administrators coordinate renaming, device replacement, and endpoint changes by draining affected instances first.

### Non-Goals

- NetBox catalog management (adding/removing devices or editing hardware metadata) — operator's responsibility; the backend only writes the documented allocation status and owner field.
- OS provisioning or image selection — orthogonal to inventory allocation; existing Metal3/BMH integration used.
- Provisioning status reporting back to NetBox — only the allocation state (`staged`/`active`) and owner identifier are recorded.
- Health checks on assigned nodes — if a node is deleted from NetBox while assigned, OSAC does not proactively detect it.
- Admin host-listing or inventory visibility in OSAC API.

## Proposal

NetBox backend is implemented in `bare-metal-fulfillment-operator/internal/inventory/netbox.go`, following the existing backend pattern. A new `NetBoxClient` reuses `baremetalhost.Manager` through the startup factory used by BCM. The internal inventory contract gains explicit per-instance context for safe claim and release; the controller preserves its normal lifecycle order and adds persisted recovery/cleanup checkpoints for NetBox. Shared allocation logs are also hardened so raw selectors, host IDs, and instance IDs are not emitted.

- **`FindFreeHost(ctx, matchExpressions)`** — Validate the selector against NetBox's field metadata, query unassigned pool devices using custom-field equality filters, and validate candidate data and naming; return `<metal3 namespace>/<device.name>` plus the separate numeric NetBox device ID, or nil when no device matches.
- **`AssignHost(ctx, inventoryHostID, instance, labels)`** — Revalidate the saved selector and pool eligibility, atomically claim with an ETag, and ensure the Metal3 resources; idempotent for crash recovery.
- **`UnassignHost(ctx, inventoryHostID, instance, labels)`** — Verify the requesting instance owns the claim, complete its Metal3 cleanup, then clear allocation state; idempotent.
- **`GetHostNICs(ctx, inventoryHostID)`** — Parse the deterministic BMH host ID and delegate to the existing `baremetalhost.Manager.GetHardwareNICs`; convert the returned MAC addresses to `inventory.HostNIC` values. Return `(nil, nil)` only when the BMH has not reported hardware NIC data yet. [Codebase: `osac/bare-metal-fulfillment-operator/internal/inventory/bcm.go`]

The implementation affects the BMF inventory clients/interface and mocks,
BMI/pool controller call sites, startup factory, and internal BMH manager; the BMF and
umbrella Helm charts; and the monorepo E2E suite under
`osac/tests/e2e/bmaas/`. Existing backends receive mechanical signature
updates and retain their current behavior. Fulfillment-service selector
projection gets regression coverage, without a new API or projection rule.
No CRD schema, gRPC, or tenant-facing API changes are added. The controller
also persists the existing `ExternalHostName` and a private device-ID annotation
before claiming a NetBox candidate; existing backends retain their behavior.

Configuration flows through the existing BMF inventory configuration Secret and the backend-specific Secrets rendered by Helm. Cloud Infrastructure Admin provides:

- NetBox API endpoint URL
- API token (rendered into a Kubernetes Secret with key `token`)
- Optional CA certificate (rendered into a Kubernetes Secret with key `ca.crt` for self-signed TLS)
- The required per-device BMC custom fields defined below

The backend uses NetBox's native device model and REST API. Host allocation state is tracked in the native device status: `staged` is available and `active` is claimed. The `osac_instance_id` custom field records which BareMetalInstance owns an active device; status alone cannot distinguish an idempotent retry from a different claimant. Credentials are never logged; API errors (401, 403) are permanent at the HTTP retry layer and produce generic messages.

### Workflow Description

#### Cloud Infrastructure Admin: Configure NetBox Backend

1. **Prerequisite:** Devices exist in NetBox with:
   - A nonempty, unique name accepted unchanged by the BMH naming contract below; for example `worker-rack3-07`. When the fabric uses hostname lookup, its server entry uses this same name.
   - Capability custom fields are pre-created on `dcim.device` with exact filtering and populated by the administrator, for example integer `cpu_cores=16`, text `gpu_model="a100"`, and integer `memory_gb=128`. The instance type selector uses those exact names and string values (`cpu_cores: "16"`). These names are examples, not a hardcoded hardware schema. OSAC does not write capability data.
   - The following exact integration custom fields on `dcim.device` (administrator-created, not built-in NetBox fields). BMC values are required on hosts assigned by OSAC; their definitions may use `required=false` so unrelated devices are unaffected:
     - `osac_bmc_username` — text; BMC login username.
     - `osac_bmc_password` — text; BMC login password. Access is controlled by NetBox RBAC; NetBox custom fields are not a Kubernetes Secret.
     - `osac_bmc_address` — text; complete Metal3-compatible BMC address, including protocol and Redfish system path when applicable.
     - `osac_boot_mac` — text; the boot NIC MAC address in canonical colon-separated form.
     - `osac_instance_id` — text, optional/nullable, with NetBox custom-field
       filtering set to `exact`; OSAC allocation owner, written and cleared by
       the backend. The exact filter setting enables the server-side
       `cf_osac_instance_id__empty=true` narrowing query.
     - `osac_managed` — boolean, optional, default `false`, exact filtering; the administrator explicitly sets it to `true` on devices in the OSAC pool. Missing, null, or false values exclude a device. This field records pool membership, not the allocation owner.
     Capabilities use separate scalar custom fields; no generic JSON `osac_labels` field or NetBox tag convention is required.
   - Status set to `staged` (available for allocation). The backend changes it to `active` when it claims the device and restores `staged` on release.

2. **Configure the canonical Helm values** under `bmf.netbox`, `bmf.metal3`,
   and `bmf.secrets` as shown in [Configuration via Helm Values and Enclave
   Wizard](#configuration-via-helm-values-and-enclave-wizard). Helm creates
   the backend credential and configuration Secrets before the operator starts;
   the names under `bmf.secrets` are the names assigned to those chart-owned
   Secrets, not references to Secrets created later by the operator. Supply
   token and CA with `--set-file` or an equivalent protected values mechanism.

3. **Deploy via osac-installer:**
   ```bash
   helm upgrade --install osac ./charts/osac -f values.yaml \
     --set-file bmf.netbox.token=/secure/path/netbox.token \
     --set-file bmf.netbox.caCert=/secure/path/netbox-ca.crt
   ```

4. **Operator starts** with the NetBox client constructed through the existing
   BMF startup factory and given the Metal3 `baremetalhost.Manager`. The operator
   performs the runtime connectivity, authentication, and required-custom-field
   checks described in the startup validation contract; Helm and Enclave Wizard
   do not probe NetBox. The optional existing installer hook checks Metal3
   prerequisites inside the cluster; Wizard validation and offline rendering
   do not perform that check.

#### Existing BareMetalInstance reconciliation

The existing `BareMetalInstanceReconciler` remains the caller of the inventory client. The NetBox adapter uses its existing discovery/assignment/deallocation boundaries, with the additional identity persistence and cleanup routing described below:

1. The fulfillment-service resolves the referenced `BareMetalInstanceType` `host_label_selector` (or the existing legacy template fallback when no instance type is referenced) into the CRD's immutable `spec.selector.hostSelector`. The reconciler clones that map and passes it unchanged to `FindFreeHost`; the tenant request does not supply a separate arbitrary capability map. The adapter validates custom-field names, types, and filtering, then sends each key/value pair as `cf_<key>=<value>` alongside the fixed pool, status, and empty-owner filters. Returned devices are checked for pool membership, availability, and the requested capability values.
   The adapter does not fetch the instance type or add a separate NetBox
   `device_type` filter, and it does not extract selectors from the instance
   type's hardware fields. Any hardware distinction needed for placement must
   be represented by a custom-field key/value pair in the resolved HostSelector.
2. `FindFreeHost` returns `<metal3 namespace>/<device.name>`, `Host.Name=device.name`, and a separate `Host.BackendID` containing the numeric NetBox ID. Before calling `AssignHost`, the reconciler persists `ExternalHostID`, `ExternalHostName`, and the private device-ID annotation together in one BMI update. When this binding exists, reconciliation skips `FindFreeHost` and passes the saved device ID to `AssignHost`; no BMH or in-memory cache is needed to recover after a restart.
3. `AssignHost` receives the instance's persisted selector and identity, rechecks eligibility on a fresh NetBox detail read, then claims ownership and delegates BMC/BareMetalHost setup to the Metal3 lifecycle integration. The reconciler continues through the existing provisioning and readiness conditions.
4. During deallocation, the reconciler calls `UnassignHost`; the adapter completes owner-checked Metal3 resource cleanup before clearing the NetBox ownership field.

The existing deletion path can initiate network-offboard power operations
before `UnassignHost`. Gate that shutdown on a nonempty assigned `HostClass`
and the presence of `BareMetalInstanceNetworkingFinalizer`, in addition to
the existing networking configuration checks. That finalizer is recorded
before network work starts and is retained for cleanup. A saved candidate ID
or the discovery-time `Allocated` condition alone is not ownership evidence.
Deletion before successful assignment therefore performs no power operation
or networking teardown on the candidate; inventory cleanup still verifies
the requesting UID. Existing finalizer-driven network/management cleanup for
successfully assigned instances remains in its current order.

The detailed retry, race, and crash-recovery behavior belongs with the individual client methods below and the existing controller retry policy. Tenant-facing BareMetalInstance APIs and provisioning state transitions remain unchanged.

#### Selector inputs and compatibility constraints

Existing context: fulfillment-service projects the instance type's
`host_label_selector.match_labels` into the immutable BMI
`spec.selector.hostSelector`. This proposal adds no new selector API.

The NetBox-specific change is passing that saved map to `AssignHost` through
`AllocationContext` so a persisted candidate can be revalidated after restart.
The existing assignment `labels` parameter comes from BMI inventory metadata
(`spec.inventoryLabels`, `spec.inventoryPersistentLabels`, and the pool ID),
not from `hostSelector`. It is separate from capability selection: NetBox
does not interpret that parameter as capabilities or write it to custom fields.
The release `labels` list likewise is not a custom-field deletion request.
The adapter does not fetch or reconstruct
the instance type. Fulfillment-service currently re-resolves the catalog during
reconciliation even for existing CRs. Because that can conflict with CRD
immutability, administrators must create a new instance type instead of
changing selectors on types still referenced by live or pending instances.
Retain that old catalog identity unchanged; do not delete/recreate it. The
same restriction applies to the legacy template selector source. Pool-created
BMIs use their persisted profile selector, including the existing
`managedBy=baremetal` default when present. NetBox adds no Metal3-style aliases:
`hostType` or `managedBy` must be compatible exact custom-field names with
populated values, or the administrator must configure another selector.
This feature does not rely on catalog edits safely changing only new requests.

For pool-created BMIs, extend the pool controller to propagate the trusted
parent `BareMetalPool` tenant annotation to its child BMI. A NetBox-backed
pool must be associated with its actual tenant before allocation; absence of
trusted tenant metadata is a configuration error, never an invented tenant
or permission to create unscoped resources. Existing non-NetBox backends keep
their current behavior. This metadata propagation is explicit new work; the
current pool controller does not populate it.

### API Extensions

No new CRDs or gRPC services. Tenant-facing APIs remain unchanged:

- **BareMetalInstance API** — no modifications. Tenants request hosts via standard API; backend transparent.
- **inventory.Client interface** — internal Go-only extension described below;
  all implementations and test doubles are updated in the same operator build.
- **Operator-managed BMI metadata** — the existing host ID/name fields and
  `osac.openshift.io/inventory-device-id` annotation form the NetBox recovery
  binding; `osac.openshift.io/inventory-cleanup` records cleanup progress.
  They are not tenant inputs or capability selectors.

The controller constructs a backend-neutral `AllocationContext` from the
persisted BMI for both assignment and release:

```go
type AllocationContext struct {
    InstanceID          string // Kubernetes BMI UID, never the API UUID
    InstanceName        string
    InstanceNamespace   string
    BackendID           string // persisted backend device ID; NetBox numeric ID
    CleanupState        string // controller-owned cleanup checkpoint, if any
    HostSelector        map[string]string // copy of spec.selector.hostSelector
    ResourceAnnotations map[string]string // trusted tenant and owner-reference only
}
```

`AssignHost` replaces its existing instance-ID string parameter with this
context; `UnassignHost` adds the context before its existing `labels []string`.
Assignment `labels map[string]string` retains its existing meaning. BCM,
Metal3, and OpenStack read `InstanceID` where applicable and otherwise keep
their current matching and label behavior. NetBox uses the saved selector
for new claims and the UID for both claims and release. No in-memory
selection cache or lookup of a mutable catalog entry is required. `Host`
gains the same optional `BackendID string` field; NetBox returns its device
ID, and existing backends leave it empty. Only the NetBox path requires and
persists it, as specified below.

NetBox additionally uses the controller-owned annotation
`osac.openshift.io/inventory-cleanup` for restart-safe cleanup. Its state is
copied into `AllocationContext.CleanupState`, never taken from assignment
labels. The controller and adapter use typed preparation-failure and
cleanup-checkpoint results as described under release; existing backends
do not emit them and keep their behavior.

Operational impact if controller is down:
- New BareMetalInstance requests pend until controller restarts
- Running hosts remain allocated (assignment recorded in NetBox)
- On controller restart: persisted cleanup resumes from `UnassignHost`; otherwise an existing complete host binding resumes `AssignHost` before any new `FindFreeHost` call

## UX Alignment

N/A — BareMetalInstance API is unchanged. NetBox backend is transparent to tenant-facing UX.

## Implementation Details/Notes/Constraints

### Configuration File Structure

NetBox backend configuration follows the existing `inventory.Config` pattern:

```go
type NetBoxOptions struct {
    Endpoint     string `json:"endpoint"`     // https://netbox.example.com
    TokenFile    string `json:"tokenFile"`    // required mounted token file
    CACertFile   string `json:"caCertFile"`   // optional mounted PEM CA file
}
```

The operator unmarshals the generic `inventory.Config` from the YAML in the Secret mounted at `OSAC_INVENTORY_CONFIG_PATH`, then reads `cfg.Options["netbox"]` into `NetBoxOptions`. The canonical rendered inventory configuration is:

```yaml
name: netbox-inventory
type: netbox
hostClass: metal3
options:
  netbox:
    endpoint: "https://netbox.example.com"
    tokenFile: "/etc/osac/secrets/osac-netbox-api-token/token"
    caCertFile: "/etc/osac/secrets/osac-netbox-ca/ca.crt" # optional
```

Helm renders this `inventory.yaml`, credential Secrets, read-only mounts,
and Metal3 management configuration. Secret names are chart inputs under
`bmf.secrets`; only the resulting file paths reach `NetBoxOptions`. The
operator loads those files at startup and does not fetch API-token/CA Secrets
through the Kubernetes API. Enclave Wizard only validates and passes the
configuration values through. [Locked: D2]

Inventory `type: netbox` chooses the discovery/claim client; `hostClass: metal3`
chooses the existing AAP provisioning workflow. The startup factory requires
management `type: metal3`, a nonempty management namespace, and inventory
`hostClass: metal3`; it constructs the BMH manager with that namespace.
There is no new `netbox` provisioning class or AAP role.
[Codebase: `osac/osac-aap/collections/ansible_collections/osac/templates/roles/bm_host_provisioning/vars/main.yaml`]

**Hardcoded Conventions:** Pool membership and allocation-state fields are fixed. Capability fields are administrator-managed and are not mutated by OSAC:

- Device pool: custom field `osac_managed=true`
- Allocation state: device status `staged` (available) or `active` (claimed)
- Allocation owner: `osac_instance_id` custom field (empty when unassigned)
- Capabilities: separate custom fields such as `cpu_cores` and `gpu_model`; the matching selector is sent as `cf_cpu_cores=16&cf_gpu_model=a100`. No mapping table, slug encoding, or per-capability code change is needed when field names match selector keys.

### Allocation Tracking and Device Filtering

The native device status and separate custom fields identify availability, ownership, and pool membership:

**1. Device Pool Selection via Custom Field**

Cloud Infrastructure Admin sets the boolean `osac_managed` field to `true`
on devices available to OSAC. The query requires `cf_osac_managed=true`;
the adapter also requires a JSON boolean `true` in returned device data.
Allocation and release preserve this administrator-owned field.

**2. Allocation State via Device Status**

OSAC uses the native device status as the allocation state: `staged` means available for selection, and `active` means claimed by OSAC. The adapter does not change status for any other device lifecycle purpose.

**3. Allocation Owner via Custom Field**

A custom field `osac_instance_id` (string, nullable) on each device stores the BareMetalInstanceID when allocated. Empty value means device is unassigned; populated value means device is owned by that BareMetalInstance.

### Host ID Mapping

Use the NetBox device's existing `name` unchanged as the BareMetalHost name,
following BCM. For device ID `42`, name `worker-rack3-07`, and namespace
`baremetal`, the binding is:

| Purpose | Value |
|---|---|
| `Host.InventoryHostID` / BMI `spec.externalHostID` | `baremetal/worker-rack3-07` |
| `Host.Name` / BMI `spec.externalHostName` | `worker-rack3-07` |
| BMH `metadata.name` | `worker-rack3-07` |
| Operator-managed BMC Secret name | `worker-rack3-07-bmc-secret` |
| `Host.BackendID` / BMI `osac.openshift.io/inventory-device-id` annotation | `42` |
| NetBox detail/claim/release endpoint | `/api/dcim/devices/42/` |

The namespace qualifies the Kubernetes object; it is not added to the NetBox
device name. Metal3 uses namespace/name, while NetBox REST operations use the
separately saved numeric ID. [User direction; Codebase:
`osac/bare-metal-fulfillment-operator/internal/inventory/bcm.go`]

**Name validation:** Require 1–63 lowercase ASCII letters, digits, or hyphens,
starting and ending with a letter or digit, matching BCM's conservative
hostname contract. Reject invalid or empty names rather than lowercasing,
truncating, adding a prefix, or replacing characters. This integration rule
is narrower than Kubernetes' general DNS-subdomain naming rules. NetBox itself
allows unnamed devices and uniqueness scoped by site/tenant, so it does not
guarantee this contract. Require unique names across the inventory visible to
the OSAC token, which must include all devices enrolled in this deployment.
[Community device naming](https://github.com/netbox-community/netbox/blob/v4.6.10/docs/models/dcim/device.md#name)

For each otherwise eligible candidate, query `/api/dcim/devices/?name=<name>`
with URL encoding and no availability/capability filter; inspect every page.
NetBox's `name` filter uses case-insensitive equality, so count case-equivalent
names as collisions, then require the sole result's exact unchanged name and
ID to equal the candidate. An invalid/ambiguous name is a configuration error,
not no capacity. Repeat this check before a new claim, then re-read the bound
device for name, eligibility, and ETag. Existing BMH/Secret name collisions
are subject to the UID ownership guards; foreign resources are never adopted.
This name check is read-only and does not replace custom-field host selection.
Administrators must not introduce duplicate names or rename devices while
allocation is in progress; a device ETag cannot lock another device's name.
[Community name filter](https://github.com/netbox-community/netbox/blob/v4.6.10/netbox/dcim/filtersets.py#L1319-L1321)

**Durable binding:** Persist the host ID, name, and numeric ID annotation in
one successful BMI API update before any NetBox claim or per-host resource
write. Validate the numeric ID as a positive canonical decimal int64. The
controller copies this operator-managed value into `AllocationContext.BackendID`,
separate from tenant/owner annotations and assignment labels. Fulfillment
reconciliation must preserve these operator-managed fields and annotation.
On confirmed race loss/reselection, clear all three together. Errors, uncertain
writes, and cleanup retries retain them. A partial/malformed binding fails
closed; never reconstruct the numeric ID from a name or a BMH that might not
yet exist. The configured endpoint/namespace must remain unchanged until all
pending and allocated instances are drained.

**Rename/replacement recovery:** Keep names unchanged from candidate selection
through release. If a detail read finds a renamed device, `AssignHost` returns
a configuration error, retains the binding/claim, and creates no replacement
BMH. Owner-checked release still addresses the original numeric ID and saved
BMH name, even after a rename or pool removal. If the numeric ID returns 404,
retain the binding/finalizer for administrator repair; never look up or touch
a replacement device that reuses the old name. This is recovery at lifecycle
boundaries, not proactive health monitoring.

**Networking and provisioning:** Persist `ExternalHostName` at selection, not
only after readiness. Existing network attachment/offboard jobs use it, and
their `ExternalHostID` suffix fallback produces the same unchanged name.
The provider must register that name in any hostname-based fabric inventory;
OSAC does not translate it. DHCP continues to prefer the inspected NIC MAC and
uses the same name only when its existing MAC-less fallback applies. An
already-allocated host is not renamed automatically. AAP routes `HostClass=metal3`
to its existing Metal3 provisioning role, which finds the BMH using
`ExternalHostID`. Test these existing consumers end to end; no new networking
backend or NetBox provisioning role is introduced.
[Codebase: `osac/osac-aap/playbook_osac_move_network_attachment.yml`,
`osac/osac-aap/playbook_osac_query_dhcp_lease.yml`]

**FindFreeHost Query:**

The query combines server-side device-pool filters with the resolved
`spec.selector.hostSelector` capability values:

```
GET /api/dcim/devices/?cf_osac_managed=true&cf_cpu_cores=16&cf_gpu_model=a100
&status=staged&cf_osac_instance_id__empty=true&limit=100&offset=0
```

Distinct query parameters are combined with AND semantics. The adapter
verifies returned device data before accepting a candidate:

| Filter/check                    | Purpose                                      |
|---------------------------------|----------------------------------------------|
| `cf_osac_managed=true`          | Administrator-authorized pool membership     |
| `cf_<hostSelector key>=<value>` (×N) | Capability equality across all requested fields |
| status=staged                   | Available allocation state                   |
| `cf_osac_instance_id__empty=true` | Server-side empty-owner narrowing          |
| Decoded pool, status, owner, and capability values | Candidate eligibility check in adapter |

Custom-field query parameters narrow the response; they do not prove that a
field exists or that its filter is enabled. Schema validation below prevents
unknown or disabled filters from silently broadening selection. The adapter
requires `osac_managed=true`, `status.value=staged`, an empty owner, and every
requested capability value. Missing/null capabilities or mismatched types do
not match. Additional unrelated fields do not disqualify a device. It continues
through pages until a candidate is found or `next` is null, validating that any
pagination URL stays on the configured HTTPS origin and device-list path.

The resolved `spec.selector.hostSelector` map is populated from the
`BareMetalInstanceType` by the fulfillment-service. NetBox uses the selector
keys as exact custom-field names, preserving meaningful values:
```
Input: {"cpu_cores": "16", "gpu_model": "a100"}
Query params: &cf_cpu_cores=16&cf_gpu_model=a100
```

**Advantages:**
- Server-side filtering: only matching pool and capability devices are returned by NetBox; reduced network payload
- The selector retains the existing key/value meaning; field metadata supplies its NetBox type
- Administrator-managed capability and pool fields remain unchanged by allocation PATCHes
- ETag protection covers the assignment PATCH; fields other than allocation status and owner are preserved

### NetBox REST API Interactions

**HTTP Client Pattern:**
- A small `net/http` adapter centralizes dynamic custom-field queries, version-compatible decoding, sanitized errors, bounded read retries, and conditional-write policy across the limited endpoint surface. The official generated SDK is also viable: it exposes HTTP responses for ETags and accepts a custom HTTP client, although conditional headers need customization. This choice avoids that extra wrapper; it is not a claim that the SDK cannot support the flow. [Official client configuration](https://github.com/netbox-community/go-netbox/blob/d187c662e8d4a9687197885d986ac3cd4f268b08/configuration.go#L76-L85)
- Base URL: the configured HTTPS origin (e.g., `https://netbox.example.com`)
  with no path other than an optional trailing slash; the adapter trims the
  trailing slash and appends `/api/` exactly once. A configured `/api` path is
  rejected so requests cannot become `/api/api/...`.
- Authentication: a NetBox v1 (classic) API token in the `Authorization: Token <token>` header; v2/Bearer tokens are outside this initial contract
- TLS validation: system CA bundle + optional custom CA cert (injected into
  the `http.Client` Transport); non-HTTPS endpoints are rejected before a
  request is sent
- Redirect policy: `http.Client.CheckRedirect` rejects any redirect that
  changes the scheme or authority from the configured origin, so the token is
  never sent to an HTTP endpoint or a different host
- Timeout: 30s per request
- Retry logic: read-only GET requests retry transient errors (5xx, network) up to 3 times. Conditional PATCH is never blindly replayed: timeout/network/5xx after submission is an uncertain outcome returned to reconciliation with the persisted host ID intact. The next detail read determines whether the claim/release committed. A direct 412 follows the method-specific race path; permanent 4xx errors fail fast.
- ETag/`If-Match` protection requires the Community releases and acceptance gates in [NetBox Version Compatibility](#netbox-version-compatibility).

**Key Endpoints:**
- `GET /api/extras/custom-fields/` — Read and paginate field definitions for schema validation
- `GET /api/dcim/devices/` — Select candidates using pool/capability custom fields, status, and `cf_osac_instance_id__empty=true`; separately query an exact `name` to validate uniqueness before selection/new claim
- `GET /api/dcim/devices/{id}/` — Fetch single device; response includes ETag header for optimistic locking
- `PATCH /api/dcim/devices/{id}/` — Set device status and update the owner custom field during assignment/unassignment. The body is a partial update under `custom_fields`, and the adapter preserves the other custom fields. It sends `If-Match` with the ETag from the prior GET to detect concurrent modifications (412 Precondition Failed on conflict).

For list/detail responses, the adapter compares the choice object's
`status.value` (`staged` or `active`) and treats a missing, null, or empty
`custom_fields.osac_instance_id` value as unassigned. PATCH requests send the
status slug (`staged` or `active`) and the nested custom-field value shown
below.

Assignment and release payloads are:

```json
{"status":"active","custom_fields":{"osac_instance_id":"<bare-metal-instance-id>"}}
{"status":"staged","custom_fields":{"osac_instance_id":null}}
```

These are the complete partial-update bodies. NetBox merges the supplied
custom-field entries with the stored map; the adapter sends only
`osac_instance_id`, preserving pool, capabilities, BMC data, and unrelated
metadata. It does not echo credentials into an allocation PATCH.
[Community custom-field serializer](https://github.com/netbox-community/netbox/blob/main/netbox/extras/api/customfields.py)

**Error Handling:**
- 401 Unauthorized — Secret validation failure; permanent at the HTTP retry layer; actionable message
- 403 Forbidden — Token lacks permissions; permanent at the HTTP retry layer; actionable message
- 4xx validation (excluding 412) — Invalid query/filter; permanent at the HTTP retry layer; report the operation and validation category without raw responses or selector values
- 412 Precondition Failed — The device changed since the eligibility read. AssignHost re-reads ownership: the same active owner resumes recovery, another owner or a clearly unassigned device permits `(nil, nil)`/reselection, and an unreadable/inconsistent state returns an error preserving the host ID. UnassignHost re-reads and rechecks ownership before retrying.
- 5xx server error — Transient; retry with backoff
- Network error (timeout, connection refused) — Transient; retry with backoff

No secrets or tenant data are logged. Error messages use generic "unable to contact inventory backend" when exposing details would leak information.

### NetBox Version Compatibility

NetBox Community 4.6.0 introduced the required ETag/`If-Match` support.
Its conditional single-device update takes a row lock and compares the fresh
ETag inside a database transaction before saving. The adapter requires the
exact detail-GET ETag, never `*` or an unconditional PATCH.
[Community conditional update](https://github.com/netbox-community/netbox/blob/v4.6.0/netbox/netbox/api/viewsets/__init__.py#L288-L302)

The initial compatibility targets are Community **4.6.10 and 4.7.1**, pinned
by image digest in the implementation's CI fixtures. Each becomes supported
only after the real-API and E2E acceptance suites pass; source inspection is
not a runtime compatibility result. No Enterprise feature or plugin is used.
Dependencies are device list/detail/partial-update, native status, and custom
fields. The decoder accepts selection values as strings (4.6) or
`{value,label}` objects (4.7), and validates active field status when present.
[4.7 selection serialization](https://github.com/netbox-community/netbox/blob/v4.7.1/netbox/extras/models/customfields.py#L387-L398)

No runtime version endpoint query or version-dependent code paths are needed. The
client requires the configured NetBox deployment to support ETag/`If-Match`;
every assignment/release detail read must provide a nonempty ETag before a
PATCH. An empty inventory can initialize successfully; startup cannot prove
conditional writes without a device or perform a destructive probe.
NetBox also exposes its API version in the `API-Version` response header, which
can be retained for diagnostics, but the client does not branch on it at
runtime.
Any later NetBox release remains unsupported until its pinned compatibility
image passes the same CI/E2E contract tests and is added to the matrix.

### Custom-Field Host Matching

**Approach: server-side custom-field equality with candidate validation**

The instance type selector supplies separate capability names and values.
For example, `cpu_cores: "16"` matches an integer field whose value is 16,
and `gpu_model: "a100"` matches that exact text or stored selection value.
NetBox combines the corresponding `cf_` filters with the fixed pool,
status, and empty-owner filters. The adapter verifies the decoded result
against the same predicates. [NetBox Community filtering](https://github.com/netbox-community/netbox/blob/main/docs/reference/filtering.md)

**Why custom fields:**

- They preserve OSAC's key/value selectors and provide typed, named values in
  NetBox's administrator interface. Values are not embedded in tag slugs.
- An existing capability field can be reused when its name, type, and exact
  filter meet this contract. New capability dimensions require administrator
  field definitions and data, but no hardcoded adapter mapping.
- Tags also support server-side filtering. Neither smaller responses nor a
  measured performance advantage is exclusive to either representation.
  This choice follows the selector semantics and administrator workflow.
- OSAC writes only the owner field and native allocation status. Pool,
  capability, BMC, and unrelated metadata remain administrator-managed.

**Field names and values:**

Keys are exact NetBox custom-field names: 1–50 ASCII letters, digits, or
underscores, excluding double underscores. The adapter rejects empty keys,
keys beginning with `cf_`, and the reserved integration fields
`osac_managed`, `osac_instance_id`, `osac_bmc_username`,
`osac_bmc_password`, `osac_bmc_address`, and `osac_boot_mac`.
This prevents selectors from replacing pool/owner filters or supplying
lookup suffixes such as `__empty` or `__gt`. Keys are never normalized or
derived from hardware fields. Every value must be non-empty.

The supported capability types are scalar `text`, `integer`, `boolean`,
and `select`, all with `filter_logic=exact`:

| NetBox field type | Selector value | Candidate comparison |
|---|---|---|
| Text / selection | Exact string or stored choice value, e.g. `"a100"`; no leading/trailing whitespace and no literal `"null"` | Exact, case-sensitive string equality; for selection fields decode the 4.6 string or the 4.7 `{value,label}` object's string `value`, never its display label |
| Integer | Canonical base-10 signed 64-bit integer string, e.g. `"16"`; no leading zeros or plus sign | JSON integer equality without float conversion |
| Boolean | Exactly `"true"` or `"false"` | JSON boolean equality; missing/null is neither true nor false |

Unsupported types, including JSON containers, multiselect, object references,
dates, and decimals, produce a selector configuration error. A provider can
represent a capability value as exact text when typed arithmetic is unnecessary.
The selector expresses equality; it does not infer minimum CPU/RAM quantities
or ranges. The shared OSAC selector remains `map<string,string>`.
Literal `"null"` is rejected for text/selection because Community filtering
uses it as a missing/null sentinel; surrounding whitespace is rejected because
string filters trim it. URL encoding cannot restore exact equality for either
case. These are explicit configuration errors, not an empty-capacity result.
[Community null filter](https://github.com/netbox-community/netbox/blob/v4.6.10/netbox/extras/filters.py#L56-L73)
[Community field definitions and filters](https://github.com/netbox-community/netbox/blob/main/netbox/extras/models/customfields.py)

**Schema validation and lifetime:**

At startup, the operator paginates
`GET /api/extras/custom-fields/?object_type=dcim.device`, checks the
returned `object_types`, and validates the six fixed integration field
definitions: pool, owner, and the four BMC fields. Pool and owner must permit
unset values (`required=false`) and exact filtering; pool must be boolean
with default false and owner must be text with no non-empty default. BMC
definitions must be text; actual per-device BMC values are validated during
assignment. The API represents `type` and `filter_logic` as choice objects;
the adapter reads their `value` members. When field metadata contains `status`
(introduced in 4.7), require `status.value=active`; provisioning or deleting
fields are not usable. These read-only checks work inside
the deployment network and need no access from Helm or Enclave Wizard.

Capability keys arrive with each resolved selector and are not all known at
startup. Each `FindFreeHost` call and each new-claim `AssignHost` refreshes the paginated field metadata,
revalidates pool/owner definitions, and validates every requested capability's
device association, supported type, and exact filtering before querying
devices. Metadata is reused within that call only; a new or corrected
instance type requires no operator restart. Failure to read or validate
metadata returns an error, with no fallback to a less restrictive query.
Missing fields, disabled/loose filtering, unsupported types, and invalid
values are configuration errors, distinct from a valid query with no hosts.
Administrators must coordinate schema edits with allocation; field-schema
changes are not covered by a device ETag.
[Community dynamic filter construction](https://github.com/netbox-community/netbox/blob/main/netbox/netbox/filtersets.py)

The adapter sends exactly one `cf_<key>` parameter per selector entry,
constructed with `url.Values.Set` and URL encoding. Extra device fields are
allowed, but every requested field must match. Neither a tag nor a generic
JSON capability map participates in matching. A schema change or unexpectedly broad
API response cannot make a mismatching returned device eligible; the local
typed checks reject it. These checks cannot compensate for a server query
that omits an eligible device, so real Community API tests must verify the
filters' completeness as well as their exclusion behavior.

### AssignHost Implementation

**Normal Path (Happy Case):**
0. Validate the nonempty `instance.InstanceID`, required resource context, and positive numeric `instance.BackendID`. Parse `inventoryHostID` as the configured Metal3 namespace plus unchanged device name; require the complete persisted binding described above.
1. Read the device by `instance.BackendID`; require its returned ID and name to match the binding and check assignment status. A missing or renamed device returns an error retaining the binding. Capture the nonempty ETag header; a missing ETag prevents any claim PATCH.
   - If status is `active` and `osac_instance_id` is the **same ID** → skip the NetBox claim PATCH and continue with the idempotent Secret/BMH ensure steps below
   - If the owner is this BMI but status is inconsistent → return an error and retain the recovery pointer for repair; do not reselect and abandon its claim
   - Otherwise, any non-empty owner, a status other than `staged`, or `osac_managed` not equal to JSON boolean `true` → return (nil, nil), with no writes
   - For an unclaimed device, refresh and validate field metadata for `instance.HostSelector` and recheck name uniqueness, then fetch the bound device again and retain that response's ETag. Recheck its ID/name and apply the same-owner recovery branch again if needed; otherwise recheck pool, status, empty owner, and every typed selector value together. An eligibility mismatch returns `(nil, nil)` so the controller clears the complete candidate binding and searches again; identity/schema/transport errors retain the binding and return an error. This also handles edits between discovery and assignment, including a restart before claiming.
2. Extract BMC data: read `osac_bmc_username`, `osac_bmc_password`, `osac_bmc_address`, and `osac_boot_mac` from the device custom fields. Validate the address and MAC before changing the allocation marker.
3. For an unassigned device, PATCH assignment: set `status = active` and `osac_instance_id = instance.InstanceID`. Include the exact `If-Match` ETag from the eligibility-checked detail response. On **412**, re-read and validate the bound device identity and ownership before deciding race loss: the same active owner with unchanged name resumes at step 2 using the newly read BMC data and skips another claim PATCH; another owner or a clearly unassigned device permits `(nil, nil)` only when identity still matches. A renamed, unreadable, or inconsistent state returns an error preserving the binding. An earlier timed-out claim can have committed while a retry was in flight. For an already-owned device, do not repeat the claim PATCH.
4. Ensure the namespace-scoped BMC Secret through `BMHLifecycleManager`, using the extracted username/password and trusted resource annotations. The Secret carries an operator-managed Kubernetes label and owner metadata identifying this BMI UID; refuse to overwrite an existing Secret owned by anyone else.
5. Create the BMH through `BMHLifecycleManager`, passing the validated BMC address, Secret name, boot MAC, annotations, and full BareMetalInstance consumer reference including UID. Idempotent reuse requires matching ownership, including the UID, and a non-terminating object.
6. Check readiness through the existing manager; return `Host.Ready = true/false`, `HostClass=metal3`, and the unchanged bound host name/IDs. The controller sets BMI `HostClass` only after readiness, preserving its existing gate. NetBox does not perform power control, inspection, OS provisioning, or readiness transitions.

If BMC Secret or BMH preparation fails after the claim, `AssignHost` returns
a typed `PreparationFailed` error without starting destructive compensation.
The controller persists cleanup state `compensating`, then routes subsequent
reconciliations through `UnassignHost` until rollback completes. A failed
checkpoint update permits no cleanup side effect. This prevents a restart or
pending deletion from re-entering the same-owner resource-creation path.
Cleanup removes only this BMI's BMH/Secret, observes disappearance, checkpoints
that fact, then conditionally clears its claim. After success, the controller
clears the cleanup state and full candidate binding in one update before a
new search. Errors retain the state/binding for safe retry. `Host.Ready=false`
and transient readiness-read errors are not preparation failures and retain
the normal assignment retry path. Failures after this adapter boundary use
the existing provisioning/deallocation lifecycle.

**Idempotency Contract (Safe for Unlimited Retries):**
- If persisted cleanup is in progress, the controller continues UnassignHost;
  AssignHost rejects nonempty CleanupState without preparing resources
- Otherwise every call starts by reading device and checking current assignment
- If status is `active` and already assigned to the same ID → skip the ownership write, but repeat the idempotent Secret/BMH ensure and readiness checks
- For a new claim, require pool membership, `staged`, an empty owner, and the persisted selector's typed capabilities on the same detail response used for the conditional PATCH
- Same-owner active retries and release remain possible even if an administrator removes pool membership after allocation
- This makes the entire flow idempotent across retries, transient failures, and crashes

**Race Condition Handling (Concurrent Requests with ETag Protection):**
- BareMetalInstance A and B both call `FindFreeHost` → same device returned to both
- A's `AssignHost` reads device (step 1), captures ETag_v1
- B's `AssignHost` reads device (step 1), captures ETag_v1
- A's PATCH includes `If-Match: ETag_v1` → succeeds (first writer wins); NetBox updates device and returns new ETag_v2
- B's PATCH includes `If-Match: ETag_v1` → fails with 412 Precondition Failed (device was modified since B's read)
- B re-reads and observes A's different owner; `AssignHost` returns `(nil, nil)` and the caller retries `FindFreeHost`
- Result: true first-writer-wins semantics; no silent overwrites; race detected at conditional PATCH time

**Transient Failure Recovery (PATCH Fails):**
- PATCH fails with 5xx, timeout, or network error
- Do not automatically resend the PATCH with its old ETag: a committed write whose response was lost would otherwise return 412 and be mistaken for another claimant
- Return an error and retain the complete host ID/name/numeric-ID binding; controller requeues with exponential backoff
- Next reconciliation retries `AssignHost`
- Step 1 finds the same owner → skips the claim write and resumes Secret/BMH ensure and readiness checks; there is no cached assignment result
- If that read precedes a delayed earlier commit, a subsequent PATCH may return 412. The ownership read in the 412 path recognizes this BMI's eventual claim and preserves it. A failed verification read returns an error, never permission to discard the recovery pointer.

**Crash Recovery (Operator Restart):**
- Crash after `FindFreeHost` but before persisting the host binding:
  - No state changes anywhere; reconciliation restarts from `FindFreeHost`
- Crash after persisting the host binding but before `AssignHost` PATCH:
  - CR has host ID/name and numeric-ID annotation set together; NetBox is still unassigned
  - On restart, the controller validates the complete binding, skips `FindFreeHost`, and calls `AssignHost` using the persisted numeric ID
  - If another request claimed the device: `AssignHost` returns (nil, nil); the controller clears all three binding values together before retrying `FindFreeHost`
- Crash after `AssignHost` PATCH succeeds:
  - Both CR (complete persisted binding) and NetBox (`status=active`, `osac_instance_id`) are consistent
  - On restart, `AssignHost` step 1 finds matching assignment → skips the ownership write, ensures the Secret/BMH, and reconciliation continues to provisioning

### UnassignHost Implementation

**Persisted cleanup checkpoints:** Before entering this method, the controller
records `releasing` for deletion or `compensating` for preparation rollback in
the BMI cleanup annotation. If deletion starts during compensation, retain
its progress and complete cleanup for deletion instead of allocating again.
The corresponding `releasing-ready` / `compensating-ready` states mean both
owned Kubernetes resources were observed absent before the NetBox release
attempt. These four values are the only accepted nonempty states.

**Normal Path (Happy Case):**
0. Validate the nonempty requesting `instance.InstanceID`, persisted numeric `BackendID`, saved namespace/name binding, and one of the four persisted cleanup states. Missing or unknown cleanup state cannot authorize destructive cleanup.
1. Read the device by the persisted numeric ID, never by name. A rename does
   not change the cleanup target. A nonempty owner must equal the requesting BMI UID;
   otherwise return an ownership-conflict error with no writes or deletes,
   except for the completed-cleanup recovery rule below.
   `staged` plus empty owner and no residual resources is already released.
   Other inconsistent status/owner combinations fail closed for operator
   repair. Pool membership and capability changes do not block release.
2. For an active claim owned by this BMI, inspect the existing BMH and BMC
   Secret. Require matching `ConsumerRef.UID` and Secret owner metadata before
   cleanup; an operator-managed Kubernetes label alone is insufficient. Missing objects
   are safe for crash recovery. Refuse foreign or ownerless residual objects.
3. Delete only that BMH with Kubernetes UID/resourceVersion preconditions.
   `DeleteBMH()` currently returns when DELETE is accepted, so add a
   direct API-server absence check: while the BMH exists or is terminating, return a retriable
   cleanup-pending error. Retain its credentials and the NetBox claim.
4. Once the BMH is absent, delete only its owned BMC Secret with object
   preconditions and verify absence using an uncached API-server read. A pending/failed deletion retains the
   NetBox claim and inventory finalizer for the next reconciliation.
5. Once both resources are absent, if the persisted state is not yet the
   corresponding `*-ready` state, return typed `CleanupCheckpointRequired`
   without releasing NetBox. The controller saves that state in the BMI and
   retries; write failure leaves the claim and finalizer intact.
6. Re-read the device and recheck the requesting owner and resource absence.
   If still owned by this
   BMI, PATCH `status=staged` and `osac_instance_id=null` with this fresh,
   nonempty ETag. A 412 retries from step 1; never replace the ETag without
   rechecking ownership. If already released and resources are absent,
   return success. Do not mutate a different owner's claim or resources.

**Release completed before a crash:** With a persisted `*-ready` checkpoint,
a retry may find staged/empty state, or an active device already claimed by
another BMI. Use direct Kubernetes reads to confirm no BMH or Secret at the
bound names is still owned by the releasing UID; missing resources or clearly
foreign-owned replacements require no deletion. Then report cleanup complete
without writing NetBox or touching replacements. Ownerless/inconsistent
resources or a reappearing resource owned by the old UID fail closed for
repair. Without the checkpoint, a different owner is still an ownership error.
The checkpoint proves prior resource cleanup, not permission to clear another
claim. This handles a successful or lost-response release followed by
reallocation before the old BMI's finalizer is removed.

The controller keeps the full binding/checkpoint until it removes the inventory
finalizer (normal deletion), or atomically clears them for a new search after
compensation. `PreparationFailed` and `CleanupCheckpointRequired` are typed
internal error results on the existing error returns, not new tenant APIs;
the controller handles them before its generic error path. Neither checkpoint
is written by the NetBox adapter directly to the BMI.

**Idempotency Contract:**
- Same read-first pattern as AssignHost
- Every retry uses persisted cleanup progress and checks device state
- Safe for multiple retries without duplicate writes
- ETag-based `If-Match` on PATCH prevents concurrent modification; 412 triggers re-read and retry (safe because unassignment is idempotent)
- The requesting UID is mandatory: agreement between a BMH consumer and a NetBox owner does not authorize a different BMI's deletion. A candidate selected but never claimed cannot release a subsequent claimant's host.

### Metal3 BareMetalHost Lifecycle

The adapter translates NetBox BMC metadata into `baremetalhost.Manager`
operations; Metal3 continues to handle power, inspection, provisioning, and
readiness (D4). The internal manager gains ownership-aware NetBox operations:
pass trusted annotations and a UID-bearing consumer reference on creation,
read ownership/deletion state, refuse foreign-resource updates, and delete
with object preconditions while observing completion. Ownership and absence
decisions use uncached API-server reads, including after uncertain creates;
the existing cached `BMHExists` result is insufficient. Existing helper behavior
does not by itself provide these guarantees; the implementation adds and tests
them without changing other backends' existing resource behavior.

**Key Properties:**

- **Coupled lifecycle** — Inventory allocation and BMH creation happen together (steps 4-6 in AssignHost); a preparation failure retains the claim while the controller checkpoints and completes compensation before another allocation attempt
- **Idempotent** — `EnsureBMCSecret()` and `CreateBMH()` are idempotent; crash recovery finds existing objects and reuses them
- **Ordered deallocation** — BMH and its operator-managed Secret are deleted and their absence checkpointed before the NetBox allocation marker is cleared
- **Readiness reporting** — the existing manager reads BMH status; `AssignHost` returns that status to the caller while the controller requeues until ready

**BMC Credential Security:**

- NetBox stores the configured per-device BMC metadata; it remains the source of truth for the adapter. Because NetBox custom fields are not a secret store, the NetBox API token must be restricted to the operator's device read/update permissions and NetBox roles must restrict who can view these fields.
- Credentials are fetched only during assignment and passed to the existing manager, which creates a namespace-scoped, operator-managed Kubernetes Secret for the BMH
- The Secret carries a Kubernetes label for cleanup and is deleted during deallocation; the NetBox adapter never logs or writes the credential values back to NetBox
- Reuses the existing Metal3/BMH integration without adding provisioning logic to the NetBox backend

### TLS and Credential Security

**Credential Management:**
- API token stored in the Helm-created Kubernetes Secret `osac-netbox-api-token`, key `token`
- Helm mounts key `token` read-only at `/etc/osac/secrets/<bmf.secrets.netboxToken>/token` and renders that full path as `tokenFile`; the operator reads the file, not the Kubernetes Secret API
- Operator constructs the HTTP client with the `Authorization: Token <token>` header
- Token never logged; errors sanitized to hide sensitive values

**TLS Configuration:**
- System CA bundle used by default
- Optional custom CA cert provided in the Helm-created `osac-netbox-ca` Secret, key `ca.crt`; Helm renders its mounted path as `caCertFile`
- Certificate pinning not supported in initial implementation
- Self-signed cert support: Cloud Infrastructure Admin provides CA cert; operator adds to `http.Client.Transport.TLSClientConfig`

**Validation at Startup:**
- Operator reads configuration on startup
- Parses the configured URL, reads the required `tokenFile` and optional `caCertFile`, and builds the authenticated HTTP client
- Missing/unreadable/empty token files and explicitly configured unreadable/invalid PEM CA files fail initialization. An omitted CA path uses system trust; a bad configured CA never silently falls back. Credential paths are file paths, never interpreted as Secret names. The client loads them once; rotation requires an explicit operator restart.
- Performs read-only NetBox API requests that validate connectivity, authentication, and the fixed integration field definitions described in [Schema validation and lifetime](#custom-field-host-matching)
- Fails backend initialization on invalid URL, TLS/connectivity failure, 401/403, or missing required custom field; no allocation is attempted with an unvalidated backend
- Permission to perform the allocation PATCH is verified by the first real assignment; the token is documented with the required read/update device permissions

### Configuration via Helm Values and Enclave Wizard

NetBox uses Metal3 for power management and host provisioning. Enabling
`bmf.netbox.enabled` selects the complete combination automatically; a second
enable flag is not required. `bmf.metal3.namespace` remains the single namespace
setting for management, the injected BMH manager, and its BMC Secret Role.

The BMF chart rejects NetBox combined with another inventory selection:
`metal3.enabled`, `bcm.enabled`, or the umbrella's fake Metal3 test backend.
These flags select inventories, not independent management add-ons. For
valid values, render NetBox inventory when `netbox.enabled`, Metal3 inventory
when `metal3.enabled`, and Metal3 management when either is enabled. An omitted
or false Metal3 flag therefore works with NetBox without duplicate configuration
Secrets. Existing non-NetBox rendering remains unchanged.
Enable NetBox token/CA mounts and the namespace-scoped BMC Secret Role for
NetBox independently of `metal3.enabled`.

```yaml
bmf:
  secrets:
    inventoryConfig: "osac-inventory-config"
    managementConfig: "osac-management-config"
    netboxToken: "osac-netbox-api-token"
    netboxCA: "osac-netbox-ca"
  netbox:
    enabled: true
    endpoint: "https://netbox.example.com"
    token: ""                 # supplied with --set-file
    caCert: ""               # optional; supplied with --set-file
  metal3:
    namespace: "baremetal"
```

The chart creates `bmf.secrets.netboxToken` with key `token`, creates
`bmf.secrets.netboxCA` with key `ca.crt` when `caCert` is non-empty, and
renders `bmf.secrets.inventoryConfig` with the `options.netbox` file paths
shown above and fixed `hostClass: metal3`. There is no NetBox host-class
override. When no CA is supplied, its Secret, `caCertFile`, volume, and mount
are omitted. The Deployment mounts the backend Secrets read-only below a
deterministic path derived from each Secret name, following the existing BMF
certificate-volume pattern; the client reads the fixed `token` and `ca.crt`
files from those mounts. This follows the existing BMF `bcm.*` plus
`secrets.*` pattern rather than introducing a second `secretRef` convention,
and avoids asking an operator to create the backend credential Secrets
separately. The per-host BMC Secret is different: `AssignHost` reads the
BMC fields from the selected NetBox device and calls the existing
`baremetalhost.Manager.EnsureBMCSecret`; that namespace-scoped Secret is
created only when a host is assigned, then deleted by `UnassignHost`.

The same enablement renders this management configuration into
`bmf.secrets.managementConfig`:

```yaml
name: metal3-management
type: metal3
options:
  metal3:
    namespace: "baremetal"
```

The startup factory explicitly injects the BMH manager into the NetBox client,
as it does for BCM; a registry-only constructor cannot supply that dependency.
It parses the namespace from management configuration, not another NetBox
option. Extend the existing BCM-only BMC Secret Role/RoleBinding gate for
NetBox, retaining `get/create/update/delete` and uncached Secret operations.

Sensitive values are supplied with `--set-file` (or an equivalent protected values mechanism), for example:

```bash
helm upgrade --install osac ./charts/osac -f values.yaml \
  --set-file bmf.netbox.token=/secure/path/netbox.token \
  --set-file bmf.netbox.caCert=/secure/path/netbox-ca.crt
```

**Validation at deployment:**
- Standalone chart validation and the umbrella `values.schema.json` require an HTTPS endpoint, nonempty token, and Metal3 namespace when NetBox is enabled; conflicting backend selections fail rendering. The NetBox values, Secret names, descriptions, and optional CA are declared in the umbrella schema for Wizard consumption.
- Rendering creates the configuration/credential Secrets and mounts without contacting NetBox or the Metal3 runtime. If the existing installer pre-install validation hook is enabled, its Metal3 checks and corresponding `provisionings` RBAC are enabled by NetBox too, subject to BMaaS enablement. This is an in-cluster prerequisite check, not a Wizard or NetBox connectivity probe.
- Operator startup performs the runtime validation contract above
- After changing credentials or configuration with Helm, explicitly restart the BMF Deployment; a Secret-only Helm upgrade does not guarantee a rollout. The restarted operator rereads the files and repeats startup validation. Endpoint/namespace changes require a prior drain, because saved device IDs are tied to that inventory.

Optional Enclave Wizard integration (pre-deployment):

This is a values-schema integration contract, not a new Enclave Wizard
component. Deployments may use Helm directly.

1. Accepts and schema-validates the endpoint, token/CA inputs, and Metal3 namespace using the existing Enclave Wizard validation/playbook mechanisms
2. Generates the values input for Helm; it does not create Kubernetes Secrets itself and does not probe NetBox connectivity or Metal3 operator availability
3. Helm creates the Secrets and configuration described above
4. The operator performs runtime connectivity, authentication, and custom-field validation during initialization

**Startup validation contract (operator initialization):**
- Authentication failures (401, 403): detected at operator startup; backend initialization fails with an actionable message
- Connectivity failures: detected at operator startup; operator fails fast
- Fixed-schema failures (missing/incompatible pool, owner, or BMC field definitions on `dcim.device`): detected at operator startup; operator fails fast
- Capability definitions are validated per `FindFreeHost` and again before a new claim, when the resolved selector is known; invalid selectors fail before listing devices or writing a claim
- Invalid endpoint URL: detected at operator startup; operator fails fast
- Result: if the backend initializes successfully, NetBox is reachable, authenticated, and has the required schema; Enclave Wizard is not a runtime dependency

### Error Messages and Diagnostics

Cloud Infrastructure Admin receives clear, actionable messages:

- **At startup:** "NetBox backend initialized"
- **On connectivity failure:** "Failed to connect to NetBox API: request timeout after 30s. Check endpoint URL and network connectivity."
- **On auth failure:** "NetBox API authentication failed. Check the mounted token configured by tokenFile and its device read/update permissions."

## Security Considerations

### Tenant Isolation

No tenant-identifying data is recorded in NetBox. The assignment identifier is the Kubernetes BareMetalInstance UID; NetBox stores only that ID, not tenant name or namespace. [PRD: In Scope — no tenant-identifying data in NetBox]

Tenant users cannot see which backend is in use or any NetBox state; allocation is transparent. Network namespace or firewall rules may restrict NetBox API access to OSAC control plane only. [PRD: In Scope — tenant transparency]

### Input Validation

The cloud-admin-managed `BareMetalInstanceType.host_label_selector.match_labels`
is the source of the selector for a NetBox profile. The fulfillment-service
resolves that map into the immutable
`BareMetalInstance.spec.selector.hostSelector`, and the controller supplies the
resolved map to the inventory backend. When no instance type is referenced,
the existing legacy template fallback remains the source of the resolved
selector. The tenant-facing request selects the published type or catalog
entry through the existing API; the tenant does not need to know the NetBox
custom-field representation.

The adapter applies the [field-name, type, and schema rules](#custom-field-host-matching)
before constructing a device query. It adds only `cf_` to each validated
field name and uses the supplied value. Lookup suffixes and reserved
integration fields are rejected; encoding a value such as `a100&status=active`
must keep it inside one value and cannot change the fixed status or pool filter.
An unknown field is a configuration error; a known field with no matching
devices is ordinary capacity exhaustion.

NetBox queries constructed defensively:
```go
params := url.Values{}
params.Set("cf_osac_managed", "true")
params.Set("status", "staged")
params.Set("cf_osac_instance_id__empty", "true")
for key, value := range validatedSelector {
    params.Set("cf_"+key, value)
}
requestURL.RawQuery = params.Encode()
```

### Authorization

OSAC trusts NetBox API token to enforce authorization. Token should have read/update permissions on `dcim.device` and read access to the required custom-field metadata; no create/delete (the operator never removes devices or changes the NetBox schema). [Assumption: Cloud Infrastructure Admin configures NetBox token with least-privilege scopes]

## Failure Handling and Recovery

The per-method sections above are normative for request ordering, idempotency, race handling, and crash recovery. This section summarizes only the controller-level policy:

- The HTTP adapter retries transient GET failures up to three times. An uncertain PATCH outcome is returned without transport replay, preserving the recovery pointer until the next ownership read. The existing reconciliation lifecycle requeues with normal backoff. “Permanent” HTTP errors are not retried within the request, but runtime credential revocation still follows controller backoff; startup validation catches usual misconfiguration before allocation.
- `FindFreeHost` returning `(nil, nil)` is expected capacity exhaustion, not a NetBox error. The existing controller reports no matching hosts and polls again using `NoFreeHostsPollIntervalDuration`.
- A 412 response follows the owner-checked method paths above: assignment confirms whether this BMI owns the device before selecting again; release re-reads without clearing another instance's assignment.
- Startup URL, TLS, authentication, and required-field failures prevent backend initialization. In this initial design, correcting the mounted Secret requires an operator restart so the client is rebuilt and startup validation runs again.
- `UnassignHost` clears the NetBox marker only after the existing Metal3 manager has completed BMH and operator-managed Secret cleanup. A cleanup failure therefore leaves the marker set and is retried safely.
- If adapter-side BMC Secret or BMH preparation fails after a claim, the controller persists compensation intent before cleanup. Retries finish that cleanup rather than recreating a BMH. Resource-absence checkpoints allow completion after a lost release response and subsequent reallocation, without changing the new owner's resources.
- All retry paths are idempotent: the assignment field, `ExternalHostID`, BMH, and BMC Secret are checked before repeating a write or create operation.
- Missing/malformed identity bindings, invalid/duplicate names, and renamed or missing bound devices are configuration/identity errors, not capacity exhaustion. Preserve the complete binding on errors; numeric-ID-based release cannot switch to a same-named replacement.

## RBAC / Tenancy

Existing tenant-facing BareMetalInstance RBAC remains unchanged. The BMF
service account retains its existing cluster-scoped BMH/HardwareData access;
the adapter restricts operations to its configured Metal3 namespace. Extend
the existing namespace-scoped BMC Secret lifecycle Role to NetBox; no
cluster-wide Secret list/watch is added. File-based token/CA loading needs
no separate Kubernetes Secret-read grant. The BMC Role covers Secrets in its
namespace, so platform credential separation still depends on namespace/RBAC
placement, not on the file-loading choice alone.
Per-host BMHs and BMC Secrets carry `osac.openshift.io/tenant` and
`osac.openshift.io/owner-reference` annotations supplied by the controller.
It copies the trusted BMI tenant annotation and sets the owner-reference
annotation to the BMI's Kubernetes UID, rather than copying a possibly absent
owner annotation or accepting the separate assignment metadata as authority. The BMH's full
`ConsumerRef` additionally records the BMI name, namespace, kind, and UID;
cross-namespace Kubernetes ownerReferences are not used. Missing or
conflicting ownership metadata prevents resource creation/reuse. The manager
must propagate these annotations; its current helpers do not do so.
Chart-owned token/CA/configuration Secrets are platform-scoped and remain
Helm-owned, never per-tenant resources. No tenant metadata is sent to NetBox.
The BMI device-ID annotation is written by the trusted controller from the
inventory result and preserved on subsequent reconciliation, not populated
from tenant requests or assignment labels. It supplies a lookup target, never
ownership authority: the requesting BMI UID is still checked before writes.
The NetBox token authorization boundary is covered under [Authorization](#authorization).

## Observability and Monitoring

### Metrics

New Prometheus metrics emitted by NetBox backend:

| Metric | Type | Prometheus labels | Meaning |
|--------|------|--------|---------|
| `osac_netbox_assignment_attempts_total` | counter | `result` | Total AssignHost attempts; result = success/race/error |
| `osac_netbox_api_errors_total` | counter | `error_type` | Total API errors by type (401, 403, 5xx, timeout) |

Pool-wide capacity/usage monitoring is deferred to a separate cross-backend
metrics feature. This backend does not scan the entire pool to publish a
capacity gauge; `FindFreeHost` can stop at the first eligible candidate.

Cross-backend host-search and assignment latency metrics are not defined by
this backend-specific design. They should be added at the shared controller
boundary so BCM, OpenStack, Metal3, and NetBox expose the same measurements.

### Structured Logs

The NetBox adapter and the shared allocation-path logs include:
- `host_selector_key_count` — number of keys in the resolved `spec.selector.hostSelector`
- `query_capability_filter_count` — number of capability custom-field filters, excluding fixed pool/status/owner filters
- `matching_devices_count` — count of inspected candidates passing the checks in this search, not total pool capacity
- `host_selected` — whether a candidate was selected
- `netbox_api_error` — error type/code (never token value or full response)

Selector keys/values, full query URLs, host IDs, BareMetalInstance UIDs, tenant data,
and credential details are not logged. The implementation must replace the
generic controller's existing raw `matchExpressions` and `InventoryHostID`
fields with these safe counts/booleans/error codes on the NetBox allocation
path; this is a logging-only hardening change and does not alter reconciliation
behavior.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| NetBox API incompatibility across versions | Features broken in upgrades | Require NetBox 4.6+ for ETag/`If-Match`; document and test the supported point-release matrix in CI |
| Credential exposure via error messages | Security breach | Audit all error paths; sanitize NetBox error responses; never log API responses |
| NetBox network partition | Allocation blocked until recovery | Graceful degradation: timeout after 30s; requeue BareMetalInstance; no tenant-visible difference |
| External status edit on an OSAC-managed device | A claimed device can look available or an available device can be hidden from allocation | Require pool membership, `staged`, and empty owner for new claims; preserve owner under `If-Match` |
| Capability schema or filter changes | A query can become invalid or less selective | Refresh field metadata per search, reject incompatible schemas, validate decoded candidates, and coordinate administrator schema edits with allocation |
| Race between AssignHost and UnassignHost | Device simultaneously assigned and deallocated; inconsistent state | Require the requesting UID to own NetBox and Kubernetes resources; use Kubernetes object preconditions, wait for deletion, then reread owner and conditionally release with `If-Match` |
| Duplicate/invalid names or rename during allocation | BMH collision or broken fabric lookup | Validate unchanged names and uniqueness before new claims, persist numeric identity before writes, require stable names until drain, and fail closed without adopting foreign resources |

All mitigations are concrete and testable.

## Drawbacks

**NetBox version floor** — ETag/`If-Match` support makes NetBox 4.6 the minimum for the race-safe implementation. Mitigation: publish the supported point-release matrix and test it in CI.

**Inventory preparation** — Administrators must create pool, ownership, BMC, and capability fields, and populate device values. Tags require fewer field definitions; custom fields instead provide typed values matching the existing selector contract. Fixed fields are validated at startup and capability fields per search.

**Name contract** — Unchanged device names preserve BCM-style host IDs and
fabric lookups, but exclude unnamed, invalid, or duplicate-named devices.
Name-uniqueness checks add read requests. Persisting a separate numeric ID is
required for safe retries before a BMH exists; renaming requires a drain.

**Schema lookup and query cost** — Each search reads field metadata before querying devices; custom-field filters use NetBox's JSON-backed storage. No query-latency advantage over tags is assumed. Real Community API and large-pool tests establish the baseline before implementation-specific caching or indexing is considered.

**Read-before-write assignment flow** — AssignHost and UnassignHost must read the device before the conditional PATCH to determine its current state and capture an ETag. A concurrent external edit causes the PATCH to fail safely with 412 rather than silently overwriting the device. Mitigation: follow the method-specific race path and retry reconciliation.

**Small custom HTTP adapter** — Choosing a hand-maintained adapter instead of the viable generated SDK makes OSAC responsible for its decoding and request policy. Keep its endpoint surface narrow and test both supported Community releases; reassess the SDK if the integration grows.

## Alternatives (Not Implemented)

### Alternative 1: Status-Only Assignment Tracking

Use only the built-in `status` field (for example, `staged` for available and `active` for claimed), without an owner custom field.

**Pros:**
- No owner custom field required; simpler NetBox schema
- Native status field; operators already familiar with it

**Cons:**
- Cannot distinguish a retry for the same BareMetalInstance from a different claimant
- Other systems managing NetBox state may overwrite status
- A status transition alone does not preserve the assignment owner for crash recovery

**Rejection:** D3 requires using the native status for allocation state, so this design uses status together with `osac_instance_id`; status alone is insufficient for idempotency and ownership checks.

### Alternative 2: Out-of-Tree Backend (Separate Sidecar Service)

Deploy NetBox backend as a gRPC sidecar service instead of in-tree implementation.

**Pros:**
- Loose coupling; backend updates don't require operator recompilation
- Language flexibility; backend could be written in Python (NetBox native)

**Cons:**
- Operational overhead; manage separate service, routing, TLS
- Introduces network latency and failure modes (sidecar unavailable)
- Enclave Wizard setup more complex (manage additional service deployment)

**Rejection:** In-tree is simpler and aligns with the existing pattern. [Locked: D2] Out-of-tree extraction supported via clean internal types (future OSAC-3806).

### Alternative 3: Sync NetBox State Back to OSAC API

Periodically query NetBox; expose available/claimed host counts in OSAC API for Admin visibility.

**Pros:**
- Admin can see inventory utilization without leaving OSAC

**Cons:**
- Increases complexity; requires new API (AdminInventoryStatus or similar)
- Eventual consistency issues; stale data
- Out of scope for current feature

**Rejection:** [PRD: Out of Scope — Admin host-listing / inventory visibility in the OSAC API] Deferred to future enhancement. Current design allows future extension without changes to NetBox backend.

### Alternative 4: Capability and Pool Tags

Pre-create a pool tag and one slug for each capability/value combination.
Repeated `tag=` filters provide AND matching and allow arbitrary independent
capability markers without defining a field for each capability dimension. However, exact
slug keys with placeholder values depart from OSAC's key/value selector;
encoding pairs into slugs requires a collision-safe naming convention. Tags
also allow conflicting values for a dimension unless administrators prevent
them. Separate scalar custom fields provide typed values and direct equality
filters. Both approaches can narrow results server-side, and no benchmark
establishes a performance winner. Custom fields are selected for the agreed
key/value contract. [Locked: D5]

### Alternative 5: One JSON Custom Field for All Capabilities

A generic `osac_labels` map avoids a field definition for every dimension.
However, arbitrary JSON custom fields do not expose the same scalar `cf_`
equality filters as separately defined fields in the Community API. A design
using that map would need a different query mechanism or broader retrieval
with client-side matching. Separate scalar fields make the query and its
validation explicit. [Community filter generation](https://github.com/netbox-community/netbox/blob/main/netbox/extras/models/customfields.py)

### Alternative 6: Generate BMH Names from Numeric Device IDs

An ID-derived name avoids NetBox name validity and uniqueness prerequisites
and embeds the REST lookup key. However, it differs from the inventory/fabric
hostname and requires explicit translation for hostname-based consumers.
The selected approach follows the requested BCM-style unchanged name, with
the numeric ID persisted separately for recovery. [User direction]

## Test Plan

[The detailed test plan](testplan.md) is the scenario inventory and requirement
coverage source. The checks below summarize its layers without duplicating
scenario counts. These are planned acceptance tests, not executed results.

### Unit Tests

Use Go table-driven tests, a TLS HTTP test server, and mocked Metal3 manager:

- Decode configuration and fixed field metadata; validate exact field names,
  supported scalar types, reserved keys, lookup suffixes, and encoded values.
  Invalid selectors/schema produce errors before listing devices.
- Translate `{"cpu_cores":"16","gpu_model":"a100"}` into distinct
  `cf_cpu_cores=16&cf_gpu_model=a100` parameters. Verify unchanged projection
  separately in fulfillment-service; hardware descriptions supply no filters.
- Check AND equality and supersets, missing/null values, wrong JSON types,
  empty ownership, and Boolean pool membership. Paginate past ineligible
  results and validate pagination URLs without introducing a pool-monitoring
  query.
- Exercise conditional assignment/release, missing ETags, conflicting owners,
  concurrent updates, restart recovery, BMC validation, resource cleanup
  ordering, and preservation of administrator metadata.
- Assert generic errors and safe controller logs, bounded retry behavior,
  TLS rejection, HTTP redirect rejection, and token/BMC secret non-disclosure.
- Validate unchanged host names, paginated uniqueness checks, independent
  numeric IDs, rename/missing-device recovery, mounted-file failures, and
  the NetBox-inventory/Metal3-provisioning startup contract.

### Integration Tests

BMF controller-runtime envtest tests create BareMetalInstance CRs directly and
use a TLS NetBox mock. They verify persisted host IDs, later reconciliation,
same-owner recovery, race loss/reselection, finalizers, Metal3 ownership,
pending BMH deletion, atomic host-ID/name/device-ID persistence, clearing on
race loss, persisted compensation/resource-absence checkpoints, completion
after release followed by reallocation, and Helm-rendered Secret/configuration wiring. Envtest
simulates Metal3 status; it does not run fulfillment-service or provision a
physical server.

A separate suite runs against each pinned **NetBox Community** image with
synthetic devices, field schemas, and credentials. It verifies real scalar
filter semantics (including case-sensitive equality), ignored/disabled filter
safety, empty-owner completeness, metadata pagination, schema changes after
startup, partial-update preservation, and concurrent conditional PATCHes.
A mock cannot establish those NetBox guarantees. Real API cases also verify
name filtering, duplicate names across sites/tenants, and recovery by numeric
ID after a rename.

### E2E Tests

Use the monorepo `osac/tests/e2e/bmaas/` pytest patterns and a real Community
NetBox deployment plus Metal3-accessible hosts. Fixtures contain
`osac_managed=true`, exact scalar capability fields, BMC metadata, and
`status=staged`. No pool or capability tags are needed.

Exercise tenant create/provision/delete, independent tenant claims, restart
recovery, preservation of pool/capability/unrelated metadata, capacity
exhaustion and recovery, and concurrent allocation. Measure capacity from
NetBox before provisioning: BMHs do not exist until assignment. Repeat the
existing tenant workflow under separate backend configurations to verify API
transparency. Disconnected coverage uses mirrored dependencies and private
NetBox connectivity, without public-network access.

Provisioning asserts `HostClass=metal3` and a BMH named exactly like the
NetBox device. Where fabric networking is configured, test attachment,
offboarding, and DHCP discovery with both NIC-MAC matching and the existing
name fallback, including restart with the persisted host binding.

## Graduation Criteria

- **Dev Preview:** all unit and envtest scenarios in the detailed plan pass;
  Helm validation passes; races produce one owner and wrong-requester cleanup
  cannot release another instance; log assertions find no sensitive values.
- **Tech Preview:** all real Community API and E2E scenarios pass against every
  pinned supported image. Twenty concurrent requests against ten eligible
  hosts produce exactly ten assignments and no orphaned BMHs or BMC Secrets.
- **GA:** the complete plan passes across the supported matrix, with no
  critical allocation or credential-exposure defects. Administrator setup,
  migration, operations, and recovery procedures are published in osac-docs.

## Upgrade / Downgrade Strategy

The shared OSAC API and CRD schema are unchanged. NetBox preparation and
selector compatibility still require an explicit migration plan.

**First deployment / existing custom fields:** Reuse compatible fields and
selector pairs directly. Define missing capability fields and `osac_managed`,
set exact filtering, and explicitly enroll pool devices with the boolean
value true. Adding a new capability dimension requires a NetBox field and
device values; the adapter discovers it without a code change. Do not rename
existing fields used by other consumers merely to match the examples.
Validate names and fabric correspondence before enrollment. The device-ID
annotation and host ID/name are internal controller-owned state, not fields
the tenant must supply.

**Earlier tag-based prototype:** Pause new allocation and drain existing
prototype NetBox instances using the old backend before switching contracts.
Explicitly convert each old capability slug into a field/value pair, and each
reviewed pool member into `osac_managed=true`; reject ambiguous or conflicting
tag data. Create new instance-type identities for new requests; retain old
referenced definitions unchanged until drained. Existing CRD `hostSelector`
maps are immutable; changing the source catalog entry can cause rejected
reconciliation patches and is not a migration. Pending resources using old tag
keys must be cancelled/recreated
through the normal API lifecycle. Verify cleanup of all persisted host IDs,
owners, BMHs, and BMC Secrets before enabling the new backend. Unrelated tags
may stay in NetBox and are ignored by OSAC; no automatic conversion is built
into the adapter.

The same drain requirement applies to any experimental deployment using
ID-generated BMH names or lacking the durable numeric-ID annotation. Do not
rewrite live `ExternalHostID` values or rename existing BMHs in place. This
design does not infer a missing device binding from a hostname.

**Another inventory backend:** Compatible key/value selectors may be reusable,
but preserving their shape does not migrate hosts, assignment state, BMC data,
or backend-specific host IDs. Drain the old backend before changing it, prepare
the target inventory, and validate selectors against its naming/type rules.

**Downgrade:** Before downgrading to a version without the NetBox backend, stop
new NetBox allocations, delete or otherwise drain every NetBox-backed
BareMetalInstance, and wait for BMH/BMC Secret cleanup. Verify that no
OSAC-managed NetBox device still has a non-empty `osac_instance_id`; only then
switch the Helm values to the alternative backend. If the pool cannot be
drained, keep the NetBox backend enabled until it can be.

Deploy matching operator and chart revisions together; the runtime options
and NetBox startup factory must agree with the rendered configuration.

## Version Skew Strategy

The NetBox client and its internal interface/controller changes ship in one
BMF binary. Its chart must render file-path options, the device-name identity
contract, and Metal3 management as described here. Existing AAP Metal3 and
hostname consumers are reused; their behavior is covered by E2E tests.
Do not mix prototype and revised NetBox operators against live instances;
drain before switching identity/configuration contracts.

If future out-of-tree migration (OSAC-3806) separates backend into sidecar, version skew strategy will be defined then. Current design supports clean extraction (backend implements `inventory.Client`; no operator-internal types leaked).

## Support Procedures

### Detect Failures

**Symptoms of misconfiguration:**
- Operator logs: "NetBox API authentication failed"
- BareMetalInstance status reports the existing allocation failure condition
- Prometheus metric `osac_netbox_api_errors_total{error_type="401"}` increasing

**Symptoms of connectivity issues:**
- Operator logs: "NetBox API request timeout"
- BareMetalInstance has `HostConditionAllocated=False` with reason
  `NoMatchingHosts` when the pool is empty; transient API errors are retried
- Tenant-facing status remains the generic "No hosts available" contract and
  does not expose NetBox-specific errors
- Prometheus metric `osac_netbox_api_errors_total{error_type="timeout"}` increasing

**Symptoms of schema mismatch:**
- Operator startup logs: "Custom field osac_instance_id not found in NetBox"
- Operator exits or marks itself unhealthy
- Capability-schema errors fail the affected allocation attempt before querying devices; verify field names, supported scalar types, device association, and exact filtering
- A valid FindFreeHost query with zero candidates: verify `osac_managed=true`, `staged`, empty owner, and the requested capability values on devices

### Disable the Feature

To disable NetBox backend and switch to Metal3:

1. Stop new NetBox allocations and drain every BareMetalInstance currently
   allocated from NetBox; allow finalizers to remove its BMH and BMC Secret.
2. Verify that no OSAC-managed NetBox device still has a non-empty
   `osac_instance_id`.
3. Update Helm values: disable `bmf.netbox` and enable the desired alternative
   backend (for example, `bmf.metal3`).
4. Run `helm upgrade osac ./charts/osac -f values.yaml`, then explicitly
   restart the BMF Deployment to load the new backend configuration.

If the pool cannot be drained, do not disable NetBox: the configured backend is
also used for cleanup, so switching first can orphan NetBox assignments.

**Impact on cluster health:** After the drain, no active NetBox-backed
BareMetalInstance remains; new allocations use Metal3 and do not interact with
NetBox. Skipping the drain is unsupported because it can leave assignments or
Secrets orphaned.

**Impact on new workloads:**
- New BareMetalInstance requests allocate from Metal3, not NetBox

### Re-Enable and Recovery

To re-enable NetBox backend after disabling:

1. Verify NetBox status/custom-field conventions are intact (`staged` for available, `active` plus `osac_instance_id` for claimed); verify `osac_managed=true` and administrator-maintained capability field values on pool devices
2. Supply valid token/CA input files and the intended Metal3 namespace
3. Set `bmf.netbox.enabled: true` and disable conflicting inventory backends;
   Metal3 management is included automatically
4. Helm upgrade, then explicitly restart the BMF Deployment
5. New allocations use NetBox again. Recreate or resubmit any drained
   BareMetalInstance resources after the backend is enabled.

**Consistency guarantee:** Re-enable is safe after the drain because no active
BareMetalInstance depends on the disabled backend. NetBox allocation state is
durable and is revalidated before new allocations.

## Infrastructure Needed

None for NetBox itself. NetBox is an externally managed dependency owned by the
Cloud Infrastructure Admin; OSAC does not provision, manage, or own its
infrastructure.

Testing infrastructure:
- In-process TLS `httptest.Server` for unit and envtest integration tests
- A pinned NetBox container exposed through a Kubernetes Service for full Kind
  E2E tests

---

## Provenance

Authored: revise @ design 0.11.3 - 9b25062, workspace feat/OSAC-4742-netbox-tags-etag-design @ f0a8211
Phases: revise, revise, revise

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.3","ai_workflows":"9b25062","source_repo":"f0a8211","source_repo_branch":"feat/OSAC-4742-netbox-tags-etag-design","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["revise","revise","revise"],"authoring_modes":["skill"],"context_changed":false,"origin_untracked":true} -->
