# RFC XXXX: MCP Server Security Hardening

**Status:** Draft  
**Authors:** Open for community contributions  
**License:** CC BY 4.0  
**Terminology:** Uses RFC 2119/8174 terms (**MUST**, **SHOULD**, **MAY**).

---

## Abstract

Model Context Protocol (MCP) servers extend an agent’s reach into local and remote systems. Today they are powerful but under‑governed. This RFC defines minimum security requirements and recommended practices for MCP servers and their runtimes so agents don’t become the soft underbelly of an otherwise secure environment.

---

## Goals
- A crisp security baseline for MCP servers, clients, and operators.
- Interop: guidance that works across languages/runtimes.
- Practical defaults that reduce foot‑guns without blocking legit workflows.

## Non‑Goals
- Specifying the MCP wire protocol itself.  
- Mandating a specific auth provider or vendor.

---

## Threat Model
1. **Malicious/compromised server** escalates privileges, exfiltrates data, or pivots laterally.  
2. **Poisoned tool outputs** trick the client/agent into unsafe follow‑on actions.  
3. **Over‑broad capabilities** (filesystem, network, shell) allow unintended access.  
4. **Secrets exposure** via logs, crash dumps, or telemetry.  
5. **Supply‑chain risk** from tampered server packages or dependencies.  
6. **Multi‑tenant bleed** between projects/users on the same host.

---

## Deployment Modes

MCP servers appear in three broad modes. Baseline MUSTs apply to all; this section tightens or relaxes controls per mode.

### Modes (definitions)
- **Local‑Ephemeral:** same host as the client, short‑lived, single user.  
- **Local‑Resident:** same host, long‑lived daemon, may serve multiple projects for one user.  
- **Remote‑Service:** reachable over the network (intra‑org or internet), multi‑user or multi‑tenant.

### Delta Requirements by Mode

#### Identity & Trust
- **Local‑Ephemeral**
  - Clients **MUST** bind over OS‑auth channels (Unix domain socket/Named Pipe) with peer UID/PID verification.
  - **TOFU pinning** MAY be keyed to a short‑lived process identity; pin expiry **MUST** be ≤ process lifetime.
- **Local‑Resident**
  - Stable keypair or workload identity **MUST** be used and **pinned** across restarts.
  - Key rotation **SHOULD** be at least quarterly.
- **Remote‑Service**
  - **mTLS** with CA‑anchored identities **MUST** be used; per‑tenant certs **SHOULD** be supported.
  - Federation to enterprise IdP (OIDC/SPIFFE/etc.) **SHOULD** be supported.

#### Transport & Isolation
- **Local‑Ephemeral**
  - OS‑local transports **MUST** be preferred (no TCP listeners).
  - Process sandbox (**seccomp/AppArmor/SELinux/jail**) **SHOULD** be applied with a read‑only root FS and an explicit writable temp dir.
- **Local‑Resident**
  - Same as above, plus **distinct runtime users** per project **SHOULD** be used to avoid cross‑project bleed.
- **Remote‑Service**
  - Network listeners **MUST** bind on least‑exposed interfaces, behind authenticated proxies or service mesh where available.
  - **No privileged containers**, no host namespaces; per‑tenant sandboxing **MUST** be enforced.

#### Capability Scoping
- **Local‑Ephemeral**
  - Capability Manifest **MUST** default to filesystem: **read‑only**, network: **deny‑all**, exec: **deny**, unless the invoking request explicitly scopes otherwise.
  - Elevations **MUST** be **just‑in‑time approved** and expire with the process.
- **Local‑Resident**
  - Manifests **MUST** be persisted and diff‑checked on restart; changes require user approval.
- **Remote‑Service**
  - Manifests **MUST** be policy‑gated server‑side (policy‑as‑code) and auditable per tenant.

#### Secrets & Data Handling
- **Local‑Ephemeral**
  - Secrets **MUST NOT** be written to disk; memory‑only; zeroized on exit.
  - Temp files **MUST** be in a private directory with automatic deletion on process end.
- **Local‑Resident**
  - Secrets **SHOULD** use OS keychain or local KMS; caches **MUST** have TTLs.
- **Remote‑Service**
  - Central secret store (KMS/Vault) **MUST** be used; secrets at rest **MUST** be encrypted with scoped policies.

#### Network Egress
- **Local‑Ephemeral**
  - **Deny‑all egress** by default; offline mode **SHOULD** be standard.
- **Local‑Resident**
  - Allow‑listing per domain:port **SHOULD** be user‑approved and logged.
- **Remote‑Service**
  - Egress control **MUST** be enforced at both app and network layers (egress proxy), with per‑tenant policies.

#### Auditing
- **Local‑Ephemeral**
  - Minimal local audit **MUST** exist (operation, inputs hash, outcome). On process exit, logs **SHOULD** be flushed and pruned.
- **Local‑Resident**
  - Structured audit to local sink **MUST** be enabled; rotation and integrity (hash chain) **SHOULD** be on.
- **Remote‑Service**
  - Tamper‑evident audit to centralized sink **MUST** be on; tenant‑scoped views **MUST** be supported.

#### Supply‑Chain & Updates
- **Local‑Ephemeral**
  - Package signature verification **MUST** occur at install/launch; ephemeral images **SHOULD** be pulled from pinned digests.
- **Local‑Resident**
  - Auto‑update **SHOULD** respect pinned majors; provenance (SBOM + SLSA) **SHOULD** be published.
- **Remote‑Service**
  - Signed releases, reproducible builds where practical, and continuous attestation **SHOULD** be provided to tenants.

### Decision Matrix

| Control     | Local‑Ephemeral | Local‑Resident | Remote‑Service |
|-------------|------------------|----------------|----------------|
| Transport   | UDS/Named Pipe (MUST) | UDS (SHOULD) | mTLS (MUST) |
| Identity    | OS peer + TOFU  | Stable keypair (MUST) | CA/IdP identity (MUST) |
| FS Access   | RO by default (MUST) | Scoped per project (MUST) | Per‑tenant roots (MUST) |
| Egress      | Deny‑all (MUST) | Allow‑list (SHOULD) | Policy + proxy (MUST) |
| Secrets     | Memory‑only (MUST) | Keychain/KMS (SHOULD) | KMS/Vault (MUST) |
| Auditing    | Minimal local (MUST) | Structured local (MUST) | Central, tamper‑evident (MUST) |
| Sandbox     | seccomp/AppArmor (SHOULD) | Separate users (SHOULD) | Strong container/jail (MUST) |

---

## Normative Requirements

1. **Identity & Trust**  
   - Servers **MUST** present a stable identity (key pair or workload identity).  
   - Transport **MUST** be authenticated and confidential (mTLS or OS‑auth like Unix domain sockets with peer verification).  
   - Clients **MUST** implement **trust‑on‑first‑use (TOFU)** or explicit allow‑list for server identities.

2. **Capability Scoping**  
   - Servers **MUST** declare a **Capability Manifest** describing allowed filesystem paths, network egress domains/ports, and process execution rights.  
   - Clients **MUST** enforce the manifest at runtime.  
   - Servers **MUST NOT** escalate beyond the declared manifest.

3. **Sandboxing & Least Privilege**  
   - Servers **MUST** run as non‑root.  
   - Servers **SHOULD** be sandboxed (seccomp/bpf, AppArmor/SELinux, jail, or container) with minimal privileges.  
   - File operations **MUST** guard against path traversal and symlink attacks.

4. **Secrets Handling**  
   - Servers **MUST NOT** log secrets.  
   - Secrets **MUST** be sourced from a secure store (OS keychain, cloud KMS, Vault).  
   - Crash handlers **MUST** scrub memory dumps.

5. **Network Egress Control**  
   - Default egress **MUST** be deny‑all.  
   - Servers **SHOULD** support offline mode.  
   - DNS over secure transport **SHOULD** be used when egress is enabled.

6. **User Consent & Guardrails**  
   - For high‑risk ops (shell, package install, file delete), the client **MUST** require explicit **just‑in‑time approval** with a clear diff/plan.  
   - Clients **MUST** display provenance for results (which server, version, capabilities used).

7. **Auditing & Telemetry**  
   - Servers **MUST** emit structured audit events (request id, caller, capability invoked, inputs hash, outputs hash/size, decision result).  
   - Audit streams **MUST** be tamper‑evident (append‑only with sequence numbers and hash chaining).

8. **Package & Supply‑Chain Integrity**  
   - Binaries/images **MUST** be signed; clients/operators **MUST** verify signatures.  
   - Reproducible builds are **SHOULD** where practical.

9. **Update Strategy**  
   - Servers **MUST** surface version and build metadata.  
   - Auto‑update **MUST** respect pinned major versions and signature checks.

10. **Data Minimization & Retention**  
    - Servers **MUST** collect only the minimum data required.  
    - Local caches and temp files **MUST** have TTLs and secure deletion.

11. **Multi‑Tenant Isolation**  
    - Per‑tenant sandboxes **MUST** be enforced.  
    - Cross‑tenant data flows **MUST NOT** occur unless explicitly configured and audited.

12. **Safe Defaults**  
    - Shipping defaults **MUST** be secure: deny‑all, minimal FS access, auditing on, PII redaction on.  
    - “Demo mode” **MUST** be visibly labeled.

---

## Reference Capability Manifest (Example)

*Placeholder:* this section will include a non‑normative example manifest.  
Fields to illustrate:
- Server identity (e.g., SPKI pin or workload ID)
- Filesystem scopes (read‑only vs read‑write)
- Network egress allow‑list
- Exec allow‑list and argument schema
- Risk controls (approval requirements)
- Audit sink and integrity options

---

## Compliance Levels
- **Level 0 (Unsafe):** dev‑only, not for real data.  
- **Level 1 (Baseline):** all MUSTs satisfied.  
- **Level 2 (Hardened):** MUSTs + most SHOULDs + sandboxing.  
- **Level 3 (High Assurance):** formal policy, signed builds, reproducible builds, continuous attestation.

---

## Recommended Practices (Non‑Normative)
- **Policy as code** (Rego/Cedar/etc.).  
- **Human‑in‑the‑loop trails:** retain approval snapshots.  
- **Canary servers:** route a small % of calls to staging builds.  
- **Declassification steps:** explicit transforms before outputs leave the sandbox.  
- **Red‑team playbooks** for MCP (command injection, deserialization, prompt‑to‑shell pivots).

---

## Open Questions
- Standard for capability schemas (JSON Schema vs Protocol Buffers)?  
- Canonical attestation format (in‑band SBOM + SLSA provenance)?  
- Long‑term home: community spec vs standards body?

---

## Why Now
MCP adoption is growing fast. Servers are already being used in production to give agents deep hooks into sensitive environments. Security hardening has lagged behind usability. This RFC proposes a baseline to close that gap before real incidents force it.
