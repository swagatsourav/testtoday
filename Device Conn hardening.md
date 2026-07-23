# Device Connection Hardening Plan

**Target file:** `confix/libs/device.py`
**Date:** 2026-07-23
**Status:** Proposed — awaiting implementation instruction

---

## Current Architecture

The connection flow is a two-hop SSH chain:

1. **Hop 1 — `_get_conn()`**: Netmiko `ConnectHandler` to the jumphost (RDNBOSS). Retries once (2 attempts total). Has a fallback that drops `ip` key if both `ip` and `host` are present.
2. **Hop 2 — `_connect_further()`**: Sends `ssh -o StrictHostKeyChecking=no user@device` as a shell command over the jumphost session, waits for a password prompt, sends credentials, then calls `redispatch()` to re-type the Netmiko connection for the target device type.

There is also `_copy_file_to_device()` which runs `scp -o StrictHostKeyChecking=no` over the jumphost to the target — similar retry pattern.

---

## What Currently Works

| Scenario | Handled? | How |
|---|---|---|
| Jumphost auth failure | ✅ | `ConnectHandler` raises `NetmikoAuthenticationException`, caught upstream in `action.py` |
| Jumphost timeout | ✅ | `ConnectHandler` raises `NetmikoTimeoutException`, caught upstream |
| Target wrong password | ✅ | `_password_issues` dict checks output for "incorrect password", "permission denied" |
| Target password expired | ✅ | Checked via "incorrect password" key |
| Target no password prompt after max attempts | ✅ | Raises after `total_attempts_dev` exhausted |
| Still on jumphost after login attempt | ✅ | Checks `lxmgt023` in prompt (line 283) |
| Dead connection after jumphost connect | ✅ | `is_alive()` check (line 208) |
| SCP file copy failures | ✅ | Checks for "Permission denied", "No such file", re-prompted password |

---

## Missing / Weak Scenarios

### 🔴 Critical — Will cause silent failures or hangs

#### 1. Known-hosts key mismatch on jumphost's `~/.ssh/known_hosts`

`StrictHostKeyChecking=no` is set for the `ssh` and `scp` commands from jumphost → target, so new keys are auto-accepted. But if the target's key **changed** (not new — already in `known_hosts` with a different fingerprint), OpenSSH prints `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` and **refuses to connect** with exit code 255. `-o StrictHostKeyChecking=no` alone does NOT override a conflicting entry — you also need `-o UserKnownHostsFile=/dev/null` or the stale entry must be removed.

**This is the fingerprint issue described.** The current code will hang waiting for a password prompt that never comes, eventually timeout, retry, and fail with an unhelpful "No password prompt found" error.

**Fix**: Detect `HOST IDENTIFICATION HAS CHANGED` / `Host key verification failed` in output. Run `ssh-keygen -R <host>` on the jumphost to clear the stale entry, then retry. Add `-o UserKnownHostsFile=/dev/null` as a fallback if removal fails.

#### 2. Jumphost `ConnectHandler` — no `known_hosts` policy at Paramiko level

`ConnectHandler` by default uses Paramiko's host key policy. If the jumphost itself changes keys, `paramiko.ssh_exception.SSHException: Host key ... doesn't match` or `BadHostKeyException` is raised. There's no handling for this.

**Fix**: Detect `BadHostKeyException` in `_get_conn()` and surface a clear error message. Optionally set `ConnectHandler(..., allow_auto_change=True)` or pass `ssh_config_file` with appropriate settings.

#### 3. Connection alive but channel dead (zombie session)

`is_alive()` checks the TCP socket, not whether the SSH channel is still responsive. The session can be alive but the shell channel closed (e.g., jumphost killed the session, idle timeout). Any subsequent `send_command()` will hang or raise a `socket.timeout` / `ReadException` / `WriteException`.

**Fix**: After `is_alive()`, do a quick `find_prompt()` or write/read test to confirm the channel is responsive. Add a connection-health check before critical operations.

#### 4. `_connect_further()` — no `read_timeout` on Ctrl+C send (line 271-272)

`send_command("\x03", expect_string=prompt)` has no `read_timeout`. If the jumphost is unresponsive, this hangs indefinitely.

**Fix**: Add `read_timeout=self.rto`.

#### 5. `disconnect()` — `_net_connect` may be `None`

If `connect()` fails before `_get_conn()` succeeds, `self._net_connect` is still `None`. The `finally` block in `action.py` calls `disconnect()`, which will raise `AttributeError: 'NoneType' object has no attribute 'disconnect'`.

**Fix**: Guard `disconnect()` with `if self._net_connect:`.

### 🟡 Important — Causes poor diagnostics or partial failures

#### 6. `_get_conn()` retry logic — silently swallows first attempt exception

The first failure is caught and discarded. If the real problem is auth or host-key, the retry just fails again. No logging on the first attempt failure.

**Fix**: Log the first-attempt exception at DEBUG level before retrying.

#### 7. `_connect_further()` — host key change detection is missing

The output from the `ssh` command may contain `REMOTE HOST IDENTIFICATION HAS CHANGED`, `ECDSA host key ... has changed`, `Offending key`, `Host key verification failed`. None of these are detected. The code just sees "no password prompt" and retries blindly.

**Fix**: Detect these strings in output, surface a specific error message identifying the key conflict, and optionally remove the offending `known_hosts` entry via `ssh-keygen -R <host>` on the jumphost, then retry.

#### 8. `_connect_further()` — no detection of SSH connection refused / unreachable

If the target device is down or SSH is not running, the output contains `Connection refused`, `Connection timed out`, `No route to host`, or `Network is unreachable`. These are not detected — the code just sees "no password prompt".

**Fix**: Detect these SSH error strings and raise specific exceptions with clear messages.

#### 9. `_connect_further()` — no detection of SSH banner / protocol errors

`Protocol mismatch`, `Connection reset by peer`, `kex_exchange_identification` errors from the target SSH server are not detected.

**Fix**: Add these to the detection set.

#### 10. `_redispatch()` — bare `except/raise` (line 290-295)

The try/except/raise is a no-op. It doesn't add context, log, or handle anything.

**Fix**: Either add useful context (device name, device type) to the error, or remove the try/except entirely.

#### 11. `_set_vendor_type()` — no fallback for unknown vendor

If `device_type` is not Juniper or Cisco XR, `self.vendor` is never set. Any subsequent access to `self.vendor` raises `AttributeError`.

**Fix**: Set a default vendor or raise early with a clear error.

#### 12. `connect()` — `_copy_file_to_device` runs before `_connect_further`

Line 189 copies files before line 191 connects further. The copy runs on the jumphost, targeting the device via SCP. If the device key has changed, the SCP fails with the same `known_hosts` issue. But worse: error from file copy may be confusing because the real issue is the key mismatch, not a file problem.

This ordering is intentional (copy must happen from jumphost before hopping), but key-mismatch errors from SCP need specific detection.

#### 13. No exponential backoff or jitter on retries

Retries happen immediately. Under load (many concurrent threads hitting the same jumphost), this creates a thundering herd.

**Fix**: Add a small delay with jitter between retry attempts.

#### 14. `_copy_file_to_device()` — `FileNotFoundError` used for auth failure

"Permission denied" during SCP raises `FileNotFoundError`, which is semantically wrong.

**Fix**: Use a custom exception or a generic `ConnectionError`/`PermissionError`.

### 🟢 Minor — Cleanup / robustness

#### 15. `_get_conn()` — `connection` referenced before assignment if both attempts fail

If `attempt == 1` raises, line 208 `connection.is_alive()` is reached on the code path (it isn't because of the raise, but it's fragile). The `break` makes it technically safe, but restructuring would be clearer.

#### 16. No timeout on `enable()` call (line 193)

`self._net_connect.enable()` has no explicit timeout. If the device doesn't respond to enable, it hangs.

#### 17. Hardcoded jumphost prompt check `lxmgt023` (line 283)

Only works for one specific jumphost hostname. If the environment uses different jumphosts, this check breaks silently.

**Fix**: Store the jumphost prompt from `find_prompt()` at connect time and compare against that.

#### 18. `disconnect()` for redispatched connections — commented-out code (lines 433-443)

The commented-out code suggests known issues with disconnecting redispatched connections. If the disconnect doesn't fully clean up, it may leak SSH sessions on the jumphost.

---

## Implementation Plan

Ordered by priority. Each item is a discrete, testable change.

### P1 — Auto-fix stale `known_hosts` on jumphost (Critical)

**What**: Detect `HOST IDENTIFICATION HAS CHANGED` / `Host key verification failed` / `Offending key` in SSH/SCP output. Run `ssh-keygen -R <host>` via `send_command_timing()` on the jumphost to clear the stale entry, then retry the connection. Add `-o UserKnownHostsFile=/dev/null` as a secondary flag only if `ssh-keygen -R` fails.

**Where**: `_connect_further()`, `_copy_file_to_device()`

**Scope-risk**: Narrow — SSH command strings only.

### P2 — Detect SSH connection errors from jumphost→target (Critical)

**What**: Check output for `Connection refused`, `Connection timed out`, `No route to host`, `Network is unreachable`, `Connection reset by peer`, `Protocol mismatch`, `kex_exchange_identification`. Raise specific, descriptive exceptions.

**Where**: `_connect_further()`, `_copy_file_to_device()`

**Scope-risk**: Narrow.

### P3 — Guard `disconnect()` against `None` connection (Critical)

**What**: Add `if self._net_connect:` check in `disconnect()`.

**Where**: `disconnect()`

**Scope-risk**: Trivial.

### P4 — Add `read_timeout` to Ctrl+C send (Critical)

**What**: Add `read_timeout=self.rto` to the `send_command("\x03", ...)` calls.

**Where**: `_connect_further()`, `_copy_file_to_device()`

**Scope-risk**: Trivial.

### P5 — Handle `BadHostKeyException` in `_get_conn()` (Important)

**What**: Catch Paramiko's `BadHostKeyException` separately and surface a clear error about the jumphost key change.

**Where**: `_get_conn()`

**Scope-risk**: Narrow.

### P6 — Log first-attempt failure in `_get_conn()` (Important)

**What**: Add `log.debug()` before retrying so first-attempt errors are visible in logs.

**Where**: `_get_conn()`

**Scope-risk**: Trivial.

### P7 — Set default vendor or raise early (Important)

**What**: Handle unknown `device_type` in `_set_vendor_type()` — either set `self.vendor = "Unknown"` or raise a clear error immediately.

**Where**: `_set_vendor_type()`

**Scope-risk**: Trivial.

### P8 — Remove no-op try/except in `_redispatch()` (Minor)

**What**: Add context (device name, device type) to the error, or remove the wrapper entirely.

**Where**: `_redispatch()`

**Scope-risk**: Trivial.

### P9 — Replace hardcoded `lxmgt023` with dynamic prompt (Important)

**What**: Store jumphost prompt from initial `find_prompt()` and compare against it instead of hardcoding `lxmgt023`.

**Where**: `_connect_further()`

**Scope-risk**: Narrow.

### P10 — Add retry backoff with jitter (Important)

**What**: Add `time.sleep(0.5 * attempt + random() * 0.5)` between retry attempts.

**Where**: `_connect_further()`, `_copy_file_to_device()`, `_get_conn()`

**Scope-risk**: Narrow.

### P11 — Fix `FileNotFoundError` misuse in SCP (Minor)

**What**: Use `PermissionError` or a custom exception for SCP auth failures.

**Where**: `_copy_file_to_device()`

**Scope-risk**: Trivial.

### P12 — Add connection health check (Important)

**What**: After `is_alive()`, validate channel with `find_prompt()` to confirm the channel is responsive.

**Where**: `_get_conn()`, optionally before operations.

**Scope-risk**: Narrow.

---

## Design Decisions

1. **Known-hosts auto-fix strategy**: Detect the error in output → run `ssh-keygen -R <host>` on the jumphost → retry. Do NOT blanket use `UserKnownHostsFile=/dev/null` as the primary approach (that disables all host-key verification, a security regression). Use it only as a fallback.

2. **Error detection should be centralized**: Create a helper method `_check_ssh_output(output, device_name)` that inspects SSH/SCP output for all known error patterns (key mismatch, connection errors, auth errors) and returns a classified result. Avoids duplicating detection logic.

3. **Retry structure**: Extract the retry-with-Ctrl+C pattern into a shared helper since `_connect_further()` and `_copy_file_to_device()` have nearly identical retry loops.
