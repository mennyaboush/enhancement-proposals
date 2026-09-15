# Test Plan — OSAC-1610: NetBox Inventory Backend

## Test Strategy Overview

The NetBox backend is a pluggable `inventory.Client` implementation within bare-metal-fulfillment-operator. Testing follows three layers:

1. **Unit tests** — Configuration parsing, label matching, idempotency logic, error handling (table-driven tests against **mock HTTP server**)
2. **Integration tests** — Full allocation workflow against **mock NetBox HTTP server in Kind cluster** (envtest + httptest.Server); failure recovery; lifecycle management
3. **E2E tests** — Tenant workflows against **real NetBox container instance**; allocation, provisioning, deallocation; cross-backend transparency; inventory exhaustion recovery

All tests follow Ginkgo/Gomega conventions used in bare-metal-fulfillment-operator.

## Testing Infrastructure & Patterns

### References to Existing Backend Implementations

The NetBox backend testing patterns follow the same conventions as existing inventory backends:

- **BCM Backend** (`osac/bare-metal-fulfillment-operator/internal/inventory/bcm/`):
  - `client_test.go` — Configuration parsing, error handling patterns
  - `fixtures.go` — Mock BMC credential structure (reference for NetBox Secret patterns)
  - `wait_for_*` helpers — Retry logic for transient failures (pattern to follow for transient failure tests)

- **Metal3 Backend** (`osac/bare-metal-fulfillment-operator/internal/inventory/metal3/`):
  - `client_test.go` — Idempotency contract tests for FindFreeHost/AssignHost/UnassignHost
  - BareMetalHost lifecycle management (reference for BMH creation/deletion timing)

### Test Fixtures and Provisioning

**Unit Tests:**
- Mock HTTP server provisioned via `httptest.Server` (standard Go testing package; no external dependencies)
- Mock Kubernetes Secrets via `fakeclient.NewClientBuilder()` (controller-runtime test utilities)
- NetBox schema responses hardcoded for each test case (no external NetBox instance needed)

**Integration Tests:**
- Mock NetBox HTTP server: `httptest.Server` running in-process within Kind cluster envtest
- Real Kubernetes cluster: provided by Kind + envtest (standard bare-metal-fulfillment-operator test infrastructure)
- Mock NetBox custom field schema: provided by test setup; validated by client startup

**E2E Tests:**
- Real NetBox container: provisioned via `docker run netbox:latest` or via test infrastructure container stack
- Kind cluster: same as integration tests; accessible to NetBox container via network
- Real BareMetalInstance CRDs: created via fulfillment-service API or kubectl apply
- Metal3 BareMetalHost lifecycle: managed by operator; verified via Kubernetes API watch

### Startup Validation Testing

The design requires both `inventory.type=netbox` AND `management.type=metal3` to be set. This constraint is enforced at operator startup:

- **Test location:** Unit / Configuration and Initialization section
- **Test case:** "Validate Co-Requirement: NetBox Requires Metal3 Management"
  - Setup: Operator config with `inventory.type=netbox` but `management.type` unset or non-Metal3
  - Action: Operator startup
  - Expected: Operator logs error "NetBox backend requires management.type=metal3" and exits with non-zero status

---

## Unit Tests

**Structure:** All unit tests use table-driven Ginkgo/Gomega tests with mock HTTP server (httptest.Server) and fake Kubernetes client. No external dependencies; runs in <5 seconds.

### Configuration and Initialization

**Test: Parse Valid NetBox Configuration**
- **Setup:** YAML config with endpoint, tokenSecret, caCertSecret (optional)
- **Action:** Call NewNetBoxClient() with valid config
- **Expected:** Client initialized; no error; endpoint stored correctly
- **Location:** netbox_test.go / Describe("Configuration")

**Test: Reject Missing Endpoint URL**
- **Setup:** Config with empty endpoint
- **Action:** NewNetBoxClient()
- **Expected:** Error: "endpoint required"

**Test: Load API Token from Kubernetes Secret**
- **Setup:** Mock Secret reader; Secret named "netbox-api-token" with key "token" = "test-token-123"
- **Action:** Client initialization; read token
- **Expected:** Token loaded; not exposed in logs or error messages

**Test: Load CA Certificate from Optional Secret**
- **Setup:** Config with caCertSecret; Secret contains PEM-encoded cert
- **Action:** Client initialization; TLS config built
- **Expected:** CA cert added to http.Client transport; valid TLS handshake with cert-signed server

**Test: Validate Custom Field Exists in NetBox Schema**
- **Setup:** Mock HTTP server returns device schema without osac_instance_id
- **Action:** Client startup calls /api/dcim/devices/schema/ → validates custom field
- **Expected:** Error: "custom field osac_instance_id not found; please create it in NetBox"

**Test: TLS Validation Failure on Invalid Certificate**
- **Setup:** NetBox endpoint uses self-signed cert; no CA provided
- **Action:** Client attempts connection
- **Expected:** TLS handshake fails; error message: "failed to verify TLS certificate"

---

### Startup Validation

**Test: Validate Co-Requirement — NetBox Requires Metal3 Management**
- **Setup:** Operator config with `inventory.type=netbox` but `management.type` not set to "metal3"
- **Action:** Operator initializes NetBox backend during startup
- **Expected:** Initialization fails with error: "NetBox backend requires management.type=metal3"; operator startup blocked; admin must fix config before retry
- **Note:** Design constraint (see [design.md](design.md#non-goals)); ensures BMH lifecycle is managed by Metal3 controller, not NetBox backend

---

### HTTP Client and Error Handling

**Test: Bearer Token Header Correctly Set**
- **Setup:** Mock HTTP server; expects Authorization header
- **Action:** FindFreeHost() call
- **Expected:** HTTP request includes `Authorization: Token test-token-123`; no plaintext token in logs

**Test: Handle 401 Unauthorized — Permanent Failure**
- **Setup:** Mock server returns 401 on device list request
- **Action:** FindFreeHost()
- **Expected:** Error returned (not retried); error message: "NetBox API authentication failed"; no token in error

**Test: Handle 403 Forbidden — Permanent Failure**
- **Setup:** Mock server returns 403 (insufficient permissions)
- **Action:** FindFreeHost()
- **Expected:** Error returned; message: "NetBox API permissions insufficient"

**Test: Handle 404 Not Found — Permanent Failure**
- **Setup:** Mock server returns 404 on invalid endpoint path
- **Action:** FindFreeHost()
- **Expected:** Error returned; message includes field/resource name; no exposure of full API response

**Test: Handle 5xx Server Error — Transient Failure with Retry**
- **Setup:** Mock server returns 500 on first two calls, 200 on third
- **Action:** FindFreeHost()
- **Expected:** Retries 3 times; eventually succeeds; no error returned

**Test: Handle Network Timeout — Transient Failure with Retry**
- **Setup:** Mock server delays response indefinitely; client timeout = 500ms
- **Action:** FindFreeHost()
- **Expected:** Timeout error; retries up to 3 times; returns error if all fail

**Test: Handle Connection Refused — Transient Failure with Retry**
- **Setup:** Endpoint unreachable
- **Action:** FindFreeHost()
- **Expected:** Connection error; retries; returns error after retries exhausted

**Test: Error Messages Never Expose Credentials**
- **Setup:** Config with token "secret-token-xyz"
- **Action:** Trigger 401 error; capture error message and logs
- **Expected:** Error message and logs do not contain "secret-token-xyz" or any recognizable token pattern

---

### FindFreeHost Label Matching

**Test: Match Device with All Required Labels**
- **Setup:** Tenant requests {cpu_cores: "16", memory_gb: "64"}; NetBox device has {cpu_cores: "16", memory_gb: "64", architecture: "x86_64"}
- **Action:** FindFreeHost({cpu_cores: "16", memory_gb: "64"})
- **Expected:** Device returned as candidate

**Test: No Match — Device Missing Required Label**
- **Setup:** Tenant requests {cpu_cores: "16", memory_gb: "64"}; device has only {cpu_cores: "16"}
- **Action:** FindFreeHost({cpu_cores: "16", memory_gb: "64"})
- **Expected:** Device filtered out; nil returned if no other candidates

**Test: Match — Device Has Extra Labels**
- **Setup:** Tenant requests {cpu_cores: "16"}; device has {cpu_cores: "16", memory_gb: "64", architecture: "x86_64"}
- **Action:** FindFreeHost({cpu_cores: "16"})
- **Expected:** Device matches (permissive matching)

**Test: Reserved Key Exclusion (osac_instance_id)**
- **Setup:** Device query results include osac_instance_id in labels; tenant request does not filter by it
- **Action:** Matching logic
- **Expected:** osac_instance_id ignored in matching; device still evaluated by other labels

**Test: Reserved Key Exclusion (managedBy)**
- **Setup:** Device has managedBy: "osac"; tenant request does not filter by managedBy
- **Action:** Matching logic
- **Expected:** managedBy not considered in matching; device candidate if other labels match

**Test: Empty Label Selector — All Unlabeled Devices Match**
- **Setup:** FindFreeHost({}) [no labels requested]
- **Action:** Query returns 5 devices, none with labels
- **Expected:** All 5 devices are candidates (permissive default)

**Test: Permissive Label Value Validation**
- **Setup:** Device label value contains space: {name: "compute node 1"}
- **Action:** Match against {name: "compute node 1"}
- **Expected:** Match succeeds (spaces allowed; not strict K8s label syntax)

**Test: Case-Sensitive Label Matching**
- **Setup:** Device has {cpu_cores: "16"}; tenant requests {CPU_CORES: "16"}
- **Action:** FindFreeHost({CPU_CORES: "16"})
- **Expected:** No match (labels are case-sensitive key=value)

---

### FindFreeHost tags and Status Filtering

**Test: Query Returns Only Staged Devices with Managed Tag**
- **Setup:** Mock HTTP response includes 10 devices; 6 with status=staged + tag managed_by=osac, 2 status=offline, 2 status=staged but wrong tag
- **Action:** FindFreeHost()
- **Expected:** Query includes filters: `tag=managed_by:osac&status=staged&osac_instance_id__empty=true`; only 6 devices match

**Test: No Candidates — All Active Devices Already Assigned**
- **Setup:** NetBox returns 5 active devices; all have osac_instance_id set
- **Action:** FindFreeHost()
- **Expected:** nil returned (no error); controller retries

**Test: No Candidates — Empty Device Pool**
- **Setup:** NetBox returns 0 devices matching pool
- **Action:** FindFreeHost()
- **Expected:** nil returned; no error

**Test: Pagination of Large Device Lists**
- **Setup:** NetBox has 250 devices; API limits response to 100 per page
- **Action:** FindFreeHost() queries all pages
- **Expected:** All 250 evaluated; pagination handled transparently; single candidate returned

---

### AssignHost Idempotency and Race Conditions

**Test: Successful Assignment**
- **Setup:** Device found unassigned; AssignHost called with instanceID "inst-123"
- **Action:** AssignHost(inventoryHostID="dev-1", bareMetalInstanceID="inst-123")
- **Expected:** Device PATCH updates osac_instance_id to "inst-123"; read-after-write confirms; returns (host, nil)

**Test: Idempotent Retry — Same Assignment ID**
- **Setup:** Device already has osac_instance_id = "inst-123"
- **Action:** AssignHost(inventoryHostID="dev-1", bareMetalInstanceID="inst-123") called again
- **Expected:** Read returns existing assignment; matches requested ID; returns (host, nil) without PATCH

**Test: Race Condition — Device Assigned to Different Instance**
- **Setup:** Device already has osac_instance_id = "inst-999" (another instance claimed it)
- **Action:** AssignHost(inventoryHostID="dev-1", bareMetalInstanceID="inst-123")
- **Expected:** Read-then-compare detects mismatch; returns (nil, nil) — race lost

**Test: Crash Recovery — Assignment Persisted**
- **Setup:** Device has osac_instance_id = "inst-123" (from previous AssignHost call)
- **Action:** Operator crashes and restarts; reconciliation retries AssignHost with same ID
- **Expected:** Idempotent: returns (host, nil); reconciliation proceeds; no double-assignment


---

### UnassignHost Idempotency

**Test: Successful Deassignment**
- **Setup:** Device has osac_instance_id = "inst-123"
- **Action:** UnassignHost(inventoryHostID="dev-1")
- **Expected:** Device PATCH clears osac_instance_id; read-after-write confirms nil; returns nil

**Test: Idempotent Retry — Already Unassigned**
- **Setup:** Device already has osac_instance_id = nil (unassigned)
- **Action:** UnassignHost(inventoryHostID="dev-1") called
- **Expected:** Read returns nil; already unassigned; returns nil without PATCH

**Test: Idempotent Retry — Different Assignment ID**
- **Setup:** Device has osac_instance_id = "inst-999" (belongs to different instance)
- **Action:** UnassignHost(inventoryHostID="dev-1") [attempting to unassign inst-123]
- **Expected:** Read returns different ID; does not touch it; returns nil (not ours to unassign)


---

### GetHostNICs (Unsupported in Initial Implementation)

**Test: GetHostNICs Returns (nil, nil)**
- **Setup:** Call GetHostNICs on any device ID
- **Action:** GetHostNICs(inventoryHostID="dev-1")
- **Expected:** (nil, nil) returned (not an error; indicates NIC discovery unsupported)
- **Note:** Future enhancement; no API call made to NetBox in current implementation

---

## Integration Tests

**Structure:** Integration tests use envtest (Kubernetes API server + etcd in-memory) with **mock NetBox HTTP server** (httptest.Server). Tests run against real operator reconciliation loop but with mocked external dependencies. Runs in ~30 seconds.

**Pattern:** Follow existing integration tests in BCM backend (`osac/bare-metal-fulfillment-operator/internal/inventory/bcm/integration_test.go`) for envtest setup, DeferCleanup patterns, and context usage.

### Full Allocation Workflow Against Kind Cluster

**Test: Allocate Host via BareMetalInstance**
- **Setup:** Kind cluster with bare-metal-fulfillment-operator; **Mock NetBox HTTP server** (httptest.Server); Kubernetes Secret with API token
- **Prerequisites:** 
  - `osac_instance_id` custom field exists in mock NetBox schema
  - 5 devices in mock NetBox with status=staged, managed_by="osac" tag, labels {cpu_cores: "16"}
- **Action:**
  1. Create BareMetalInstance: `name: test-bmi, labels: {cpu_cores: "16"}`
  2. Wait for operator to reconcile
- **Expected:**
  1. BareMetalInstance status.externalHostID set (device ID)
  2. Device in NetBox now has osac_instance_id set to BareMetalInstance UID
  3. No double-allocation event in logs

**Test: Deallocate Host via BareMetalInstance Deletion**
- **Setup:** BareMetalInstance allocated to device; host ready
- **Action:**
  1. Delete BareMetalInstance
  2. Wait for operator to reconcile
- **Expected:**
  1. Device in NetBox has osac_instance_id cleared
  2. Host returns to available pool
  3. BareMetalInstance deleted cleanly

**Test: Recovery from Transient Failure**
- **Setup:** BareMetalInstance allocated; NetBox becomes temporarily unreachable
- **Action:**
  1. Create allocation in progress
  2. Mock server returns 5xx for 5 seconds
  3. Wait for recovery
- **Expected:**
  1. Allocation retries after timeout
  2. On recovery: allocation succeeds
  3. No observer-visible disruption (requeue transparent to user)

**Test: Permanent Auth Failure**
- **Setup:** Secret contains invalid API token
- **Action:**
  1. Create BareMetalInstance
  2. Operator attempts allocation
- **Expected:**
  1. BareMetalInstance status condition: "InventoryAllocationFailed"
  2. Operator logs: "NetBox API authentication failed"
  3. Event emitted to BareMetalInstance
  4. Admin sees actionable message: "Check Secret netbox-api-token"

**Test: Schema Validation Failure on Startup**
- **Setup:** NetBox mock schema missing osac_instance_id custom field
- **Action:**
  1. Deploy operator with NetBox config
- **Expected:**
  1. Operator logs: "Custom field osac_instance_id not found"
  2. Operator exits or marks readiness = false
  3. Admin must create field before operator can start

**Test: Configuration Reload — Updated Endpoint**
- **Setup:** Operator running with one endpoint; Helm values updated with new endpoint
- **Action:**
  1. Update ConfigMap with new endpoint URL
  2. Restart operator
  3. Create BareMetalInstance
- **Expected:**
  1. Operator picks up new endpoint
  2. Allocation queries new endpoint
  3. Succeeds if new endpoint is reachable

**Test: Cross-Backend Non-Regression**
- **Setup:** Test cluster configured with Metal3 backend; deploy second cluster with NetBox backend
- **Action:**
  1. Create BareMetalInstance in Metal3 cluster
  2. Create BareMetalInstance in NetBox cluster
  3. Verify both allocate successfully
- **Expected:**
  1. Metal3 cluster: allocation uses Metal3 BareMetalHost
  2. NetBox cluster: allocation uses NetBox devices
  3. No cross-contamination

---

## E2E Tests (osac-test-infra)

**Structure:** E2E tests run against a **real NetBox container instance** and a real Kind cluster with all OSAC components deployed (fulfillment-service, operator, Metal3). Tests follow pytest patterns in osac-test-infra. Runs in ~5 minutes per test.

**Prerequisites:** Real NetBox container accessible from Kind cluster; Keycloak for API auth; fulfillment-service API running.

**Reference:** See `osac-test-infra/tests/` for existing E2E patterns (gRPC client, K8s client, wait_for_* helpers, pytest fixtures).

### Tenant Workflow Transparency

**Test: Allocate and Provision Host Against NetBox**
- **Setup:** Real NetBox instance accessible from Kind cluster; BareMetalInstance with labels
- **Action:**
  1. Tenant (via fulfillment-service API) creates BareMetalInstance with label selectors
  2. Operator allocates from NetBox
  3. Metal3 BareMetalHost created; power on / inspection / provisioning proceeds
  4. Wait for BareMetalInstance status = Ready
- **Expected:**
  1. ComputeInstance or VirtualMachine can SSH into provisioned host
  2. Status conditions show Provisioning → Provisioned → Ready
  3. Tenant sees no backend-specific details (NetBox transparent)

**Test: Deallocate and Reuse Host**
- **Setup:** Host allocated and provisioned from NetBox pool
- **Action:**
  1. Delete BareMetalInstance
  2. Wait for cleanup
  3. Create new BareMetalInstance with same label selectors
- **Expected:**
  1. Host deallocated cleanly (power off, BMH deleted)
  2. Host returned to NetBox available pool
  3. New request allocates same or different host
  4. No orphaned state

**Test: Label Selector Matching in Tenant Workflow**
- **Setup:** NetBox has pools: [16-core nodes] and [8-core nodes]
- **Action:**
  1. Tenant requests cpu_cores: "16"
  2. Request processed; allocation searches NetBox
  3. Verify allocation matched correctly
- **Expected:**
  1. Host allocated from 16-core pool
  2. 8-core nodes not allocated
  3. Tenant sees successful allocation

### Inventory Exhaustion Edge Case

**Test: Allocate, Exhaust, Recover (Inventory Overflow Handling)**
- **Setup:** Kind cluster with N available devices; observe initial count
- **Action:**
  1. Create N BareMetalInstance objects (consume all inventory)
  2. Wait for all to reach Ready
  3. Create BMI N+1 (overflow; no devices left)
  4. Assert N+1 does not reach Ready (stalls in allocation)
  5. Delete one allocated BMI to free a device
  6. Assert N+1 recovers to Ready
- **Expected:**
  1. N succeed; N+1 stalls (no available NetBox devices)
  2. N+1 requeues internally; does not create zombie BMH
  3. After device freed: N+1 eventually reaches Ready
  4. No corruption; all allocations idempotent

### Observability and Diagnostics

**Test: Prometheus Metrics Emitted**
- **Setup:** Operator running; Prometheus scraping operator metrics; 8 available NetBox devices with host_class="compute"
- **Action:**
  1. Create 10 BareMetalInstance objects with host_class="compute"
  2. 8 succeed (allocated); 2 race (no hosts left, requeue)
  3. Query Prometheus metrics
- **Expected:**
  1. `osac_netbox_hosts_available{host_class="compute"}` = 0 (all 8 devices allocated)
  2. `osac_netbox_assignment_attempts_total{result="success"}` = 8
  3. `osac_netbox_assignment_attempts_total{result="race"}` = 2
  4. No API errors recorded (success path; no errors)

**Test: Kubernetes Events Emitted**
- **Setup:** BareMetalInstance allocating
- **Action:**
  1. Observe events on BareMetalInstance via `kubectl describe`
- **Expected:**
  1. Event: "HostAllocated" on successful assignment
  2. Event: "NoHostsAvailable" if FindFreeHost returns nil after retries
  3. Event: "HostAllocated" on success
  4. Event: "HostDeallocated" on deletion

**Test: Structured Logs**
- **Setup:** Operator running; logs captured
- **Action:**
  1. Create allocation
  2. Grep logs for relevant entries
- **Expected:**
  1. Logs include: host_search_label_selector, selected_host_id, assignment_id
  2. No credentials, secrets, or tenant names in logs
  3. Error logs include diagnostic info (e.g., "custom_field not found") but no API responses

---

## Test Coverage Summary

| Layer | Component | Test Scenarios | Coverage |
|-------|-----------|--------|----------|
| **Unit** | Config parsing | 6 scenarios | Endpoint validation, Secret loading, TLS cert validation, schema validation |
| **Unit** | Startup validation | 1 scenario | NetBox+Metal3 co-requirement constraint |
| **Unit** | HTTP client | 8 scenarios | Auth headers, error codes (401/403/404/5xx), timeouts, connection errors, credential safety |
| **Unit** | FindFreeHost label matching | 8 scenarios | Required labels, missing labels, extra labels, reserved keys, case sensitivity |
| **Unit** | FindFreeHost filtering | 4 scenarios | Tag and status filtering, no candidates, pagination |
| **Unit** | AssignHost | 4 scenarios | Success, idempotency, race condition, crash recovery |
| **Unit** | UnassignHost | 3 scenarios | Success, idempotency (same ID), idempotency (different ID) |
| **Unit** | GetHostNICs | 1 scenario | Returns (nil, nil) — unsupported in initial implementation |
| **Integration** | Allocation workflow | 6 scenarios | Allocate via BareMetalInstance, deallocate via deletion, transient failure recovery, permanent auth failure, schema validation, config reload |
| **Integration** | Cross-backend | 1 scenario | Metal3 and NetBox backends coexist without cross-contamination |
| **E2E** | Tenant workflow | 3 scenarios | Allocate and provision, deallocate and reuse, label selector matching |
| **E2E** | Inventory exhaustion | 1 scenario | Overflow handling and recovery |
| **E2E** | Observability | 3 scenarios | Prometheus metrics, Kubernetes events, structured logs |

**Total test scenarios: 49**

---

## Test Case Reference (TC-IDs)

Each test is identified by a TC-ID for traceability in Jira subtasks and CI logs. Format: `TC-LAYER-NNN` (LAYER = UNIT/INT/E2E).

### Unit Tests (TC-UNIT-001 to TC-UNIT-035 — 35 total)

**Configuration and Initialization (TC-UNIT-001 to TC-UNIT-006):**
- TC-UNIT-001: Parse Valid NetBox Configuration
- TC-UNIT-002: Reject Missing Endpoint URL
- TC-UNIT-003: Load API Token from Kubernetes Secret
- TC-UNIT-004: Load CA Certificate from Optional Secret
- TC-UNIT-005: Validate Custom Field Exists in NetBox Schema
- TC-UNIT-006: TLS Validation Failure on Invalid Certificate

**Startup Validation (TC-UNIT-007):**
- TC-UNIT-007: Validate Co-Requirement — NetBox Requires Metal3 Management

**HTTP Client and Error Handling (TC-UNIT-008 to TC-UNIT-015):**
- TC-UNIT-008: Bearer Token Header Correctly Set
- TC-UNIT-009: Handle 401 Unauthorized — Permanent Failure
- TC-UNIT-010: Handle 403 Forbidden — Permanent Failure
- TC-UNIT-011: Handle 404 Not Found — Permanent Failure
- TC-UNIT-012: Handle 5xx Server Error — Transient Failure with Retry
- TC-UNIT-013: Handle Network Timeout — Transient Failure with Retry
- TC-UNIT-014: Handle Connection Refused — Transient Failure with Retry
- TC-UNIT-015: Error Messages Never Expose Credentials

**FindFreeHost Label Matching (TC-UNIT-016 to TC-UNIT-023):**
- TC-UNIT-016: Match Device with All Required Labels
- TC-UNIT-017: No Match — Device Missing Required Label
- TC-UNIT-018: Match — Device Has Extra Labels
- TC-UNIT-019: Reserved Key Exclusion (osac_instance_id)
- TC-UNIT-020: Reserved Key Exclusion (managedBy)
- TC-UNIT-021: Empty Label Selector — All Unlabeled Devices Match
- TC-UNIT-022: Permissive Label Value Validation
- TC-UNIT-023: Case-Sensitive Label Matching

**FindFreeHost Filtering (TC-UNIT-024 to TC-UNIT-027):**
- TC-UNIT-024: Query Returns Only Staged Devices with Managed Tag
- TC-UNIT-025: No Candidates — All Active Devices Already Assigned
- TC-UNIT-026: No Candidates — Empty Device Pool
- TC-UNIT-027: Pagination of Large Device Lists

**AssignHost (TC-UNIT-028 to TC-UNIT-031):**
- TC-UNIT-028: Successful Assignment
- TC-UNIT-029: Idempotent Retry — Same Assignment ID
- TC-UNIT-030: Race Condition — Device Assigned to Different Instance
- TC-UNIT-031: Crash Recovery — Assignment Persisted

**UnassignHost (TC-UNIT-032 to TC-UNIT-034):**
- TC-UNIT-032: Successful Deassignment
- TC-UNIT-033: Idempotent Retry — Already Unassigned
- TC-UNIT-034: Idempotent Retry — Different Assignment ID

**GetHostNICs (TC-UNIT-035):**
- TC-UNIT-035: GetHostNICs Returns (nil, nil)

### Integration Tests (TC-INT-001 to TC-INT-007)

**Allocation Workflow (TC-INT-001 to TC-INT-006):**
- TC-INT-001: Allocate Host via BareMetalInstance
- TC-INT-002: Deallocate Host via BareMetalInstance Deletion
- TC-INT-003: Recovery from Transient Failure
- TC-INT-004: Permanent Auth Failure
- TC-INT-005: Schema Validation Failure on Startup
- TC-INT-006: Configuration Reload — Updated Endpoint

**Cross-Backend (TC-INT-007):**
- TC-INT-007: Cross-Backend Non-Regression

### E2E Tests (TC-E2E-001 to TC-E2E-007 — 7 total)

**Tenant Workflow Transparency (TC-E2E-001 to TC-E2E-003):**
- TC-E2E-001: Allocate and Provision Host Against NetBox
- TC-E2E-002: Deallocate and Reuse Host
- TC-E2E-003: Label Selector Matching in Tenant Workflow

**Inventory Exhaustion Edge Case (TC-E2E-004):**
- TC-E2E-004: Allocate, Exhaust, Recover (Inventory Overflow Handling)

**Observability and Diagnostics (TC-E2E-005 to TC-E2E-007):**
- TC-E2E-005: Prometheus Metrics Emitted
- TC-E2E-006: Kubernetes Events Emitted
- TC-E2E-007: Structured Logs

---

## BMC/BMH Lifecycle Test Coverage

The NetBox backend does not manage BMC credentials or BareMetalHost (BMH) creation/deletion directly — these are handled by the operator's Metal3 integration, which is a separate concern:

- **BMC Credential Extraction:** NetBox backend reads BMC credentials from custom fields at assignment time; credentials are passed to Metal3 controller. Tested at **integration level** in TC-INT-001 and TC-INT-002 by verifying the full allocation workflow succeeds (which requires successful BMC credential retrieval and Metal3 BMH creation).

- **BMC Secret Creation:** NetBox backend returns credentials; operator creates Kubernetes Secret per Metal3 requirements. This is operator responsibility, not backend responsibility. Tested implicitly through operator reconciliation in TC-INT-001.

- **BMH Idempotency and Deletion:** Metal3 controller manages BMH lifecycle (create, inspect, provision, delete). NetBox backend is stateless regarding BMH state. Tested through operator reconciliation tests (TC-INT-001, TC-INT-002, TC-INT-003).

For explicit coverage of operator-level BMC Secret and BMH management, see `osac-operator/internal/controllers/` tests, which are out of scope for the NetBox backend test plan.

---

## Success Criteria

All tests must pass before OSAC-1610 is considered complete:

1. ✓ All 48 unit/integration/E2E test scenarios pass (TC-UNIT-001 to TC-UNIT-034, TC-INT-001 to TC-INT-007, TC-E2E-001 to TC-E2E-007)
2. ✓ No double-allocations under concurrent allocation (verified by TC-UNIT-030, TC-INT-001 with concurrent Creates)
3. ✓ No credential exposure in logs or error messages (TC-UNIT-015, TC-E2E-007)
4. ✓ Prometheus metrics collected and queryable (TC-E2E-005)
5. ✓ Cross-backend non-regression: Metal3 backend unaffected (TC-INT-007)
6. ✓ Startup validation enforces NetBox+Metal3 co-requirement (TC-UNIT-007)
6. ✓ Idempotency verified: crash recovery and concurrent operations safe
7. ✓ Tenant workflow transparency: NetBox allocation identical to other backends
