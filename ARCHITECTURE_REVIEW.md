# Architecture Review: sonic-alpine

## Overview

This is a SONiC (Software for Open Networking in the Cloud) platform implementation that
emulates a virtual switch using Alpine Linux and Google's Lemming dataplane. The architecture
bridges SONiC's control plane (syncd, Redis state DBs) with Lemming's dataplane via gRPC/SAI,
running in KVM or Kubernetes (KNE) environments.

---

## 1. Goroutine Error Loss in Packet Handler (HIGH)

**File:** `src/services/pkt-handler/main.go:76-93`

Two goroutines are launched for `ManagePorts` and `StreamPackets`, but only the first error
is ever read from the unbuffered channel:

```go
errCh := make(chan error)       // unbuffered
go func() { errCh <- ... }()   // goroutine A
go func() { errCh <- ... }()   // goroutine B
err = <-errCh                   // only reads ONE value, then main() exits
```

When one goroutine exits, the other is abandoned. If the surviving goroutine also tries to
send on `errCh`, it will block forever (goroutine leak). There's no coordinated shutdown —
`cancel()` is never called on the normal exit path, only on signal receipt.

**Fix:** Use a buffered channel (`make(chan error, 2)`), read both values, and call `cancel()`
after the first error so the second goroutine gets a context cancellation.

**Question:** Is it intentional that the process terminates when *either* RPC exits? Or should
it attempt reconnection/restart of the failed stream while keeping the other alive?

---

## 2. Hardcoded Dataplane Address — No Configuration Flexibility (HIGH)

**File:** `src/services/pkt-handler/main.go:37`

```go
const addr = "10.0.2.2:50000"
```

The Lemming dataplane address is a compile-time constant. This same `10.0.2.2:50000` address
appears to be assumed in `src/libsai-grpc/entrypoint.cc` as well. There's no way to change
it without recompiling.

**Fix:** Accept the address via a CLI flag or environment variable. The `--target_port` flag
already exists but isn't used for the main connection.

**Question:** Is this address always correct for both KVM (qemu guest to host) and KNE/Kubernetes
deployments? Different deployment modes likely need different addresses.

---

## 3. No gRPC Transport Security (MEDIUM)

**File:** `src/services/pkt-handler/main.go:44`

```go
grpc.WithTransportCredentials(insecure.NewCredentials())
```

All control-plane traffic between the packet handler and Lemming dataplane is unencrypted.
This includes port configuration commands and CPU packet streaming.

**Question:** Is this acceptable because the gRPC connection is always over a local/loopback
interface (KVM virtio), or will this ever traverse a real network in production Kubernetes
clusters?

---

## 4. Massive Code Duplication in Config Script (MEDIUM)

**File:** `src/services/config/alpinevs-config.sh`

This 194-line script contains 128 nearly-identical `redis-cli` invocations to set port IDs,
transceiver info, transceiver status, and DOM sensor data for 32 ports. Each "section" is a
copy-pasted block with only the port number and ID varying.

**Fix:** Replace with a loop:

```bash
id=1
for port in $(seq 0 4 124); do
    redis-cli -n 4 hmset "PORT|Ethernet${port}" "id" "${id}"
    redis-cli -n 6 hmset "TRANSCEIVER_INFO|Ethernet${port}" "parent" "1/${id}" \
        "type" "OSFP 8X Pluggable Transceiver"
    redis-cli -n 6 hmset "TRANSCEIVER_STATUS|Ethernet${port}" "status" "1"
    redis-cli -n 6 hmset "TRANSCEIVER_DOM_SENSOR|Ethernet${port}" "module_state" "ModuleReady"
    id=$((id + 1))
done
```

**Question:** Is the port count (32) and stride (4) always fixed, or should these be
configurable per platform variant?

---

## 5. Telemetry Config Merge Has Mutability and Complexity Issues (MEDIUM)

**File:** `src/sonic-platform-alpinevs/alpinevs-platform/sonic_platform/telemetry_device.py:35-63`

The `_modify_device_children` method mutates `source_config` in-place during iteration, uses
O(n^2) nested loops to find matching metrics by name, and calls `list.remove()` inside the
inner loop. This pattern is fragile:

- Mutating a list while iterating over a copy of it is a common source of subtle bugs.
- The `_convert_device_config` method also mutates its input — the caller's data is modified
  as a side effect.
- Every call to `get_device_info()` re-reads and re-merges the dynamic config file from disk.

**Fix:** Use dict-based metrics keyed by name instead of a list of tuples. The merge becomes
a simple `dict.update()`. Return new dicts instead of mutating in-place.

**Question:** How frequently is `get_device_info()` called? If polled regularly, the repeated
file I/O and merge could be a concern.

---

## 6. Python 2 Shebang on xcvrd Daemon (MEDIUM)

**File:** `src/platform/xcvrd/xcvrd.py:1`

```python
#!/usr/bin/env python2
```

The file has a Python 2 shebang, but the rest of the platform code uses modern Python 3.9+
features (`dict[str, str]`, `list[int]`). Python 2 reached end of life in January 2020.

**Fix:** Update the shebang to `#!/usr/bin/env python3` and verify compatibility.

**Question:** Is there a dependency on a Python 2-only version of `swsscommon` or
`sonic_py_common` that forces this?

---

## 7. xcvrd Always Reports "pass" for State Verification (LOW-MEDIUM)

**File:** `src/platform/xcvrd/xcvrd.py:78-82`

```python
verify_state_fvp = swsscommon.FieldValuePairs([
    ("status", "pass"),
    ("timestamp", channel_data),
    ("err_str", "")])
```

The state verification handler unconditionally responds with `status: "pass"`. There is no
actual verification logic.

**Question:** Is this intentional because this is a "fake xcvrd" for a virtual switch? If so,
a comment explaining this would help future maintainers.

---

## 8. No Unit Tests for Any Component (HIGH)

There are zero unit tests across the entire codebase:

- No Go tests (`_test.go` files) for the packet handler or VM launcher
- No Python tests (`test_*.py`) for platform, chassis, LED, telemetry, or xcvrd code
- No C++ tests for the SAI library bindings

The only testing is integration-level via OTG bash scripts requiring a full Kubernetes cluster.

**Fix:** The platform Python code (led_control, telemetry_device, chassis) is particularly
testable in isolation. The Go packet handler could test error channel / shutdown logic.

**Question:** Is there an existing SONiC test harness or mock framework for `swsscommon`?

---

## 9. No CI/CD Pipeline (MEDIUM)

No GitHub Actions, GitLab CI, or any CI configuration. All building, testing, and deployment
is manual:

- No automated PR validation
- No linting or formatting enforcement
- No automated image builds or artifact publishing
- No regression detection

**Question:** Is the project ready for automated build/test pipelines?

---

## 10. Inconsistent Error Handling Strategies Across Languages (LOW-MEDIUM)

| Component    | Language | Strategy                            |
|-------------|----------|-------------------------------------|
| pkt-handler | Go       | `log.Exit()` — immediate death      |
| xcvrd       | Python   | Log and silently continue           |
| led_control | Python   | Log and gracefully degrade          |
| pcie.py     | Python   | `print()` + `sys.exit()`           |
| chassis.py  | Python   | Catch exception, return empty       |

No consistent policy for fatal vs. recoverable errors. For a network platform where
availability matters, a common error handling policy would help.

---

## 11. Magic Redis Database Numbers (LOW)

**File:** `src/services/config/alpinevs-config.sh`

```bash
redis-cli -n 6 hmset ...    # STATE_DB?
redis-cli -n 4 hmset ...    # CONFIG_DB?
redis-cli -n 14 hmset ...   # APPL_STATE_DB?
```

Database numbers are bare integers with no explanation. SONiC has well-known DB ID assignments,
but newcomers to the codebase would not know what `-n 6` means.

**Fix:** Define variables: `STATE_DB=6`, `CONFIG_DB=4`, etc.

---

## Summary of Questions for the Team

1. **Packet handler resilience**: Should the packet handler attempt to reconnect when one gRPC
   stream fails, or is "exit and let systemd restart" the intended recovery model?
2. **Dataplane address**: Should the Lemming address be configurable for different deployment
   modes (KVM vs. Kubernetes)?
3. **Port count flexibility**: Is the 32-port / stride-4 layout fixed, or should the config
   script derive this from `config_db.json`?
4. **Python 2 constraint**: Is there a real dependency blocking the xcvrd Python 3 migration?
5. **Telemetry polling frequency**: How often is `get_device_info()` called, and does the
   repeated file I/O matter?
6. **Test strategy**: What's the expected unit test coverage target, and are there SONiC test
   mocks available?
7. **CI/CD readiness**: Is the project ready for automated build/test pipelines?
