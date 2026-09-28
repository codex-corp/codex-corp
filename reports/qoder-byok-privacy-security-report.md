# Independent Privacy & Network Analysis of Qoder BYOK / Custom Models

> **Status:** Independent user-led forensic analysis; updated after a synchronized live-trace test  
> **Test date:** 2026-09-28  
> **Scope:** Qoder Desktop on Windows using a custom OpenAI-compatible model endpoint  
> **Focus:** Network behavior, telemetry, local storage, BYOK routing, and practical privacy hardening

## Executive summary

This report documents an independent hands-on analysis of how Qoder behaves when a **custom / BYOK model** is selected.

The most important finding is that, in the tested configuration, the actual custom-model inference request was sent directly to the configured local OpenAI-compatible endpoint rather than to Qoder's hosted inference endpoint.

However, Qoder continued to communicate with Qoder-controlled cloud services for control-plane functions, account/licensing, telemetry, tracking, and other product features. Runtime logs and static inspection of the distributed JavaScript bundle showed several telemetry/reporting operations, including events tied to AI turns and sessions.

A particularly important privacy observation was that a human-readable **task name derived from the user's prompt** was present in an outbound `businessFinish.report` event. I did **not** find evidence that the complete custom-model prompt/context JSON was duplicated to Qoder's hosted inference service during the tested BYOK flow.

Qoder also stored substantial local session and telemetry data, including structured conversation records, tool-call data, prompt previews, file-context metadata, and file names.

This is therefore best described as a **privacy and data-minimization issue**, not evidence of malicious behavior or a claim that Qoder sends every prompt or source file to its servers.

---

## Why I tested this

BYOK often creates an intuitive expectation:

```text
IDE -> my custom endpoint -> my chosen model provider
```

The question was whether using a custom model also meant that Qoder stopped sending AI-session-related information to its own infrastructure.

The answer from this test was:

```text
                    +-> Local custom model gateway -> chosen provider
                    |
Qoder Desktop ------+-> Qoder control plane
                    +-> telemetry / tracking
                    +-> account / licensing services
                    +-> optional cloud-backed features
```

So **custom inference routing and product telemetry are separate data paths**.

---

## Test setup

The test intentionally avoided publishing machine-specific identifiers, private repositories, credentials, customer data, or proprietary source code.

The environment used:

- Windows desktop installation of Qoder
- A custom OpenAI-compatible model
- A local HTTP gateway bound to loopback
- The local gateway forwarded model requests to the selected upstream provider
- Native Windows process/network inspection
- Qoder runtime logs
- Static inspection of Qoder's distributed JavaScript runtime bundle
- Reversible Windows Firewall tests

No TLS man-in-the-middle certificate was installed.

---

## Finding 1 — Custom-model inference was sent directly to the configured local endpoint

Qoder's runtime log recorded the custom-provider request as a direct HTTP POST to the configured local OpenAI-compatible endpoint:

```text
[qoder-server-request] -->
operation=customProvider.openai.chat
method=POST
url=http://localhost:<LOCAL_PORT>/v1/chat/completions
```

and the corresponding successful response:

```text
[qoder-server-request] <--
operation=customProvider.openai.chat
host=localhost:<LOCAL_PORT>
path=/v1/chat/completions
status=200
```

The same routing behavior was observed for at least one automatically spawned internal evaluation/sub-agent turn.

### What this proves

For the tested custom model, the primary inference request was routed directly to the configured local endpoint.

### What this does **not** prove

It does not prove that Qoder sends no session-related data elsewhere. Telemetry and control-plane traffic are separate from the model request itself.

---

## Finding 2 — Qoder maintained separate cloud endpoints even while a custom model was active

The runtime elected and used Qoder-controlled service endpoints including:

```text
center.qoder.sh
api2.qoder.sh
openapi.qoder.sh
```

Observed roles included:

| Endpoint | Observed role |
|---|---|
| `center.qoder.sh` | endpoint discovery, policy/control-plane requests, turn tracking |
| `api2.qoder.sh` | telemetry ingestion, AI-turn reporting, hosted inference capability |
| `openapi.qoder.sh` | account, campaign/license-related requests |
| Sentry ingest endpoint | crash/error reporting infrastructure |

The tested installation also used an internal HTTPDNS mechanism, which means relying only on the Windows DNS cache is not sufficient to enumerate all destinations.

---

## Finding 3 — Active AI-turn telemetry/reporting was present

### `businessFinish.report`

A runtime operation named:

```text
businessFinish.report
```

was sent to a Qoder-controlled endpoint under:

```text
/algo/api/v2/service/business/finish
```

Static inspection of the distributed runtime bundle showed an event structure containing session/business identifiers and a `business.name` field.

A runtime event in the test contained a human-readable task name that was derived from the user's prompt.

A controlled benign test made this behavior directly observable. A prompt beginning with:

```text
explain how ...
```

produced an outbound business-reporting field:

```json
"name": "explain ho"
```

The same prefix-style behavior was observed in more than one turn. Machine identifiers, session identifiers, and private prompt text are intentionally omitted from this public report.

### Privacy implication

This demonstrates that "telemetry" was not limited to anonymous counters such as latency or success/failure. At least one reporting path included human-readable text derived from user input.

This still does **not** establish that the full prompt was sent in this event.

### Controlled live-trace confirmation

A synchronized trace correlated one benign prompt with Qoder's local inference and cloud reporting paths.

Observed sequence:

```text
User submits prompt
        |
        v
Qoder -> localhost:<LOCAL_PORT>/v1/chat/completions
        |
        | 200 OK / streamed model response
        v
Turn completes
        |
        +-> api2.qoder.sh
        |    businessFinish.report
        |    includes prompt-derived "name" prefix
        |
        +-> center.qoder.sh
             backFlowAgentQueryFinish.report
             session/turn operational metadata
```

The local inference request and the two cloud reporting operations had separate operation names and destinations. No duplicate hosted-model inference request was observed during this trace.

---

## Finding 4 — Additional turn/session tracking was present

Another observed reporting operation was:

```text
backFlowAgentQueryFinish.report
```

to a tracking endpoint under Qoder's control.

Static inspection showed fields such as:

```text
session_id
task_id
request_set_id
prompt_id
entry
product
client_type
duration_ms
loop_iteration_count
terminal_reason
business_state_final
```

Runtime/log instrumentation also contained tracking hooks and messages such as:

```text
ai-code-tracking-post-tool-use
ai-code-tracking-query-end
ai-code-tracking-session-end

[TraceTelemetryService] Sent event: session/prompt
[ChatContextTelemetry] Context added: type=file, name=...
```

These strings prove the presence of session/context instrumentation.

During the synchronized trace, `backFlowAgentQueryFinish.report` was observed immediately after the turn and contained operational fields such as session/task identifiers, duration, loop count, and terminal state. In the inspected payload, raw prompt text and source code were not present.

They do **not**, by themselves, prove that every associated field or file name was transmitted to a remote server.

---

## Finding 5 — OpenTelemetry logging was configured to a Qoder endpoint

The application initialized an OpenTelemetry log destination:

```text
https://api2.qoder.sh/otel/v1/logs
```

This is separate from the custom-model inference path.

The synchronized single-turn trace did not observe a distinct OpenTelemetry POST burst tied to that exact turn. Because the remote payload is TLS-protected, this test cannot prove that OpenTelemetry never contains additional user- or project-derived fields.

---

## Finding 6 — Significant session data was stored locally

The tested installation wrote extensive local records, including:

- structured session/turn JSONL files
- user and assistant conversation content
- tool-call inputs and outputs
- prompt previews and prompt lengths
- code-context/indexing logs
- file-context metadata
- file names
- outbound request metadata
- runtime operation names, URLs, status codes, durations, and request IDs

Representative locations followed patterns similar to:

```text
<USER_HOME>/.qoder/logs/runs/.../qodercli.log
<USER_HOME>/.qoder/logs/sessions/.../segments/*.jsonl
<USER_HOME>/.qoder/logs/qoder-context.log
<APP_DATA>/Qoder/logs/...
<APP_DATA>/com.qoder.app.stable/logs/...
```

In one test, context telemetry recorded names resembling sensitive environment/configuration files.

### Important distinction

A file name appearing in a local telemetry log is **not proof** that the file contents were uploaded.

But local plaintext session storage still matters for workstation security, backups, malware exposure, endpoint collection tools, and retention policies.

---

## Finding 7 — Blocking all Qoder Internet access broke custom-model availability

A Windows Firewall test blocked outbound Internet access for the main Qoder executable while leaving loopback/local networking available.

Result:

- Qoder itself still launched
- the local model endpoint remained reachable at the OS level
- **the custom model disappeared from the model selector**

This indicates that custom/BYOK model availability depends on Qoder's online control plane/catalog even though the actual inference request can be routed directly to a local custom endpoint.

This behavior is consistent with Qoder's public documentation, which says that available BYOK providers/models are determined by the catalog available to the current account.

---

## Finding 8 — Blocking only the observed `api2.qoder.sh` addresses preserved custom inference in the test

A narrower firewall experiment blocked the then-current network addresses used by `api2.qoder.sh` while leaving the rest of Qoder's control plane available.

The addresses blocked in this point-in-time test were:

```text
8.223.13.163
47.57.188.188
```

Result:

- custom model remained visible
- local custom-model inference continued working
- the controlled inference request still reached the local OpenAI-compatible endpoint successfully
- `businessFinish.report` attempted to use `api2.qoder.sh` and failed while the block was active
- a periodic model-catalog refresh to `api2.qoder.sh` also failed
- no fallback hosted-model inference request was observed

This strongly suggests that the tested custom inference path did not require Qoder's hosted `api2` inference/telemetry endpoint.

### Important limitation

This is **not a durable firewall strategy** by itself.

Qoder uses HTTPDNS and cloud infrastructure whose addresses can change. IP-specific blocking can therefore fail open after endpoint rotation or failover.

A domain/SNI/application-aware egress policy or controlled proxy is preferable for long-term enforcement.

Broad blocking of entire cloud-provider address ranges is **not** recommended from this test alone: those ranges may host unrelated services and are wider than the evidence supports.

---

## What was confirmed vs. not confirmed

### Confirmed in the tested build

- Custom model requests were sent directly to the configured local OpenAI-compatible endpoint.
- Qoder communicated separately with its own cloud infrastructure.
- Qoder initialized a remote OpenTelemetry endpoint.
- AI-turn and session reporting operations were present.
- A prompt-derived human-readable task name appeared in an outbound business-reporting event.
- A synchronized trace confirmed that this prompt-derived prefix was generated after the local model turn and sent through `businessFinish.report`.
- The same synchronized trace observed separate turn-completion metadata sent through `backFlowAgentQueryFinish.report`.
- No duplicate Qoder-hosted model inference request was observed during the controlled trace.
- Qoder stored extensive session/tool/context information locally.
- Full Internet blocking broke custom-model discovery/availability.
- Narrow blocking of the observed `api2` addresses did not break local custom inference during the test.

### Not confirmed

- No evidence was found that the **complete custom-model prompt/context body** was duplicated to Qoder's hosted inference endpoint.
- Because TLS-protected OpenTelemetry payloads were not decrypted, this test cannot prove that no additional prompt- or project-derived fields ever appear in those telemetry records.
- The test did not prove that full source-file contents were uploaded through `ChatContextTelemetry`.
- TLS-protected cloud payloads were not decrypted.
- This report does not establish how every Qoder plan, platform, region, or future build behaves.
- This report makes no claim about malicious intent.

---

## Public privacy-policy context

Qoder's current public privacy policy defines chat, coding, and agentic inputs/outputs as **User Content**.

It also states that de-identified User Content may be used to research, develop, and improve the service, with an opt-out through **Share & Improve** depending on the user's plan.

The same policy separately describes BYOK transmission to the third-party provider selected by the user.

Relevant public references:

- Qoder Privacy Policy: https://qoder.com/privacy-policy
- Qoder Custom Models (IDE): https://docs.qoder.com/qoder/custom-models
- Qoder Custom Models (CLI): https://docs.qoder.com/cli/custom-models

This report's forensic observations should be read alongside those published policies rather than as a replacement for them.

---

## Practical hardening recommendations

### 1. Do not treat BYOK as equivalent to "offline" or "no vendor telemetry"

BYOK can control the **model inference destination** while the IDE still uses vendor services for telemetry, catalog, account, licensing, security, or product features.

Model routing and IDE telemetry should be evaluated separately.

### 2. Use least-privilege egress

For sensitive projects, the strongest practical control is network-layer enforcement.

The experiments showed why an all-or-nothing block is not ideal: blocking all Internet access caused the custom-model entry to disappear, while narrower blocking of the observed `api2.qoder.sh` addresses preserved custom inference in the tested session.

Prefer:

```text
ALLOW   local custom-model gateway
ALLOW   only the minimum Qoder control-plane services required
BLOCK   optional telemetry / crash-reporting destinations where operationally acceptable
BLOCK   unused hosted-model paths
MONITOR newly introduced endpoints after every Qoder update
```

Avoid depending permanently on a static list of IP addresses.

A sensible staged hardening workflow is:

```text
1. Keep loopback/local custom-model traffic allowed.
2. Block or suppress optional telemetry paths where they can be isolated.
3. Keep only the minimum account/control-plane access needed for the product to function.
4. Re-test after a cold restart, not only in an already-authenticated session.
5. Monitor HTTPDNS/failover behavior for replacement addresses.
6. Verify that no hosted-model fallback occurs when telemetry/inference endpoints are blocked.
```

In the tested environment, blocking the observed `api2.qoder.sh` addresses was compatible with custom inference. Blocking all Qoder Internet access was not.

The editor setting `telemetry.telemetryLevel: "off"` is also worth testing and verifying against runtime traffic. Treat the setting as effective only after confirming that the corresponding network events stop; this report has not yet established that it suppresses every Qoder-specific tracking path.

### 3. Treat `.qoderignore` as an indexing control, not a security boundary

Exclude obvious secrets and sensitive data, for example:

```text
.env
.env.*
*.pem
*.key
*.p12
*.pfx
credentials/
secrets/
customer-data/
production-data/
database-dumps/
backups/
```

But do not assume ignore rules prevent every agent/tool path from reading a file.

Highly sensitive secrets should ideally live outside the working tree and outside the agent's accessible workspace.

### 4. Audit local Qoder logs

Because session transcripts and tool data can be stored locally:

- understand retention
- protect workstation backups
- restrict endpoint-management access
- avoid syncing sensitive Qoder state to unnecessary cloud storage
- review what survives after deleting a conversation
- re-check storage locations after upgrades

### 5. Harden the custom-model gateway too

A local gateway can see the entire plaintext model request.

For sensitive work:

- disable raw prompt/response logging unless required
- use short retention
- disable semantic caches if they persist sensitive payloads
- protect gateway logs
- redact secrets before upstream transmission where possible

### 6. Audit MCP / computer-use integrations separately

MCP servers, computer-use helpers, search/indexing services, and plugins can create additional network paths that are independent of the main LLM request.

A privacy review that looks only at the model endpoint is incomplete.

### 7. Verify telemetry controls instead of assuming they cover product-specific tracking

The tested build exposed a VS Code-style setting:

```json
"telemetry.telemetryLevel": "off"
```

Because Qoder also has product-specific operations such as `businessFinish.report` and `backFlowAgentQueryFinish.report`, disabling a generic editor telemetry setting should not be assumed to suppress all of them without a follow-up network trace.

### 8. Re-test after every substantial Qoder update

Endpoint behavior, telemetry fields, helper processes, and catalog dependencies can change.

A lightweight regression test should confirm:

```text
custom inference destination
Qoder cloud destinations
new helper executables
new telemetry operations
local log contents
firewall behavior
```

---

## A safer architecture for sensitive development

A practical compromise is:

```text
                         +----------------------+
                         | required control     |
                         | plane only           |
                         +----------^-----------+
                                    |
Developer -> Qoder -----------------+
             |
             | custom inference
             v
      Local model gateway
             |
             v
      Selected provider

Network policy:
- explicit local model route
- minimum required Qoder cloud access
- optional telemetry blocked where possible
- gateway content logging disabled
- secrets kept outside agent-accessible workspace
```

For environments where **no source code, prompt text, metadata, or file names may leave the workstation except to a specifically approved model endpoint**, a fully local/offline coding stack remains the cleaner security boundary.

---

## Reproducibility notes

Useful investigation techniques included:

### Enumerate Qoder processes

```powershell
Get-CimInstance Win32_Process |
  Where-Object {
    $_.Name -match 'qoder' -or
    $_.CommandLine -match 'qoder'
  } |
  Select-Object ProcessId, ParentProcessId, Name, ExecutablePath, CommandLine
```

### Inspect active sockets

```powershell
Get-NetTCPConnection |
  Select-Object OwningProcess, LocalAddress, LocalPort,
                RemoteAddress, RemotePort, State
```

### Search runtime logs for network operations

Search for terms such as:

```text
customProvider.openai.chat
businessFinish.report
backFlowAgentQueryFinish.report
api2.qoder.sh
center.qoder.sh
openapi.qoder.sh
/otel/v1/logs
ChatContextTelemetry
TraceTelemetryService
```

### Inspect distributed runtime code

Search the installed JavaScript runtime bundles for the same operation names and inspect the nearby event-building code.

Do not publish credentials, machine tokens, authorization headers, private repository names, workstation paths, or customer data while doing this.

---

## Responsible interpretation

This report is intentionally conservative.

It distinguishes:

- **observed network behavior**
- **locally visible runtime/log evidence**
- **static inspection of shipped application code**
- **public policy statements**
- **things that remain unproven because TLS payloads were not intercepted**

The key takeaway is not that "Qoder uploads everything."

The evidence supports a narrower conclusion:

> **In the tested BYOK configuration, custom-model inference was routed directly to the configured local endpoint and no duplicate hosted-model inference was observed. Qoder nevertheless maintained independent telemetry/control-plane data flows, and a synchronized trace confirmed that at least one outbound AI-turn reporting field contained a prefix derived directly from user prompt text.**

That distinction matters for developers evaluating whether a BYOK coding workflow meets their organization's privacy requirements.

---

## Disclosure / update notes

This is an independent technical report based on one tested environment and one point in time.

If Qoder changes the relevant behavior, adds clearer controls, or documents these data flows more precisely, this report should be updated accordingly.
