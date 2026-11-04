# VCAP Enterprise Interface (VCAP-EI) 1.0

*Editor's Draft — 4 November 2026*

**Specification identifier:** VCAP-EI-1.0  
**Editor's Draft URL:** https://vlei-auth.github.io/vcap-ei/  
**Status:** Editor's Draft  
**Editors:**
  * Fan Dang (ACentauri Technology Limited)

**Scope:** The interface between a platform (wallet or enterprise web page) and an enterprise service that issues Tickets, archives signatures, publishes KERI material and relays Agent access for vLEI Key authenticators.

**Companion documents:** [VAF 1.0](https://vlei-auth.github.io/vaf/) (architecture and obligations), [VCAP 1.0](https://vlei-auth.github.io/vcap/) (authenticator protocol). This document uses the data structures of VCAP §9 unchanged.

**Abstract:** VCAP defines what the authenticator accepts; VAF defines what the enterprise must check. Neither defines how a platform asks an enterprise for a Ticket, submits a result or recovers a lost one. This document defines that interface, so that a platform written by one party can use an enterprise service written by another.

---

## Contents

1. Introduction
2. Conformance
3. Terminology
4. Overview
5. HTTP Binding
6. Common Structures
7. Operations
8. WebAuthn Route Operations
9. Procedures
10. Enterprise Service Requirements
11. Security and Privacy Considerations
- Appendix A Operation Table
- Appendix B Purpose Registry
- Appendix C Wallet Operation Profiles
- Appendix D References

---

## 1. Introduction

### 1.1 Purpose

A vLEI Key signs only with a Ticket from the enterprise (VCAP §1.1 rule 2). Every signature therefore involves two exchanges: platform ↔ authenticator (VCAP) and platform ↔ enterprise (this document). VAF §1.1 states that "any KERI client that implements VCAP and obtains enterprise Tickets can use the same authenticator"; this interface is what makes that statement testable.

### 1.2 Scope

In scope:

- the enrollment record of a device the employee holds;
- single-use challenges for StateProof;
- Ticket requests for KEL and non-KEL kinds;
- archiving device results, and retrieving archived originals and signatures for recovery;
- submitting signed EXN / RPY messages for delivery;
- relaying the employee's Agent requests under the bearer token, without the authenticator;
- reading publication, acceptance and business status;
- the enterprise endpoints used by the WebAuthn route (VCAP §12.9).

**Agent profile.** The Agent that a CLIENT AID controls (VAF §3.2) is reached through an agent profile. VCAP-EI 1.0 defines one, the **KERIA agent profile**: the enterprise creates a KERIA agent from the CLIENT inception (purpose 1), the CLIENT approves its delegation (purpose 2, `agent-material`), and the platform reaches it through `agent-session` under the employee's bearer token, limited by a wallet operation profile (Appendix C). Agent requests produce no slot signature and do not use the authenticator. A service that offers neither of purposes 1 and 2 does not depend on the agent implementation. Other agent profiles are added in later versions.

Out of scope: employee authentication itself (OAuth 2.0 / OIDC, §5.4), enterprise approval workflows, the PDP, agent and witness protocols, key custody of the enterprise's role keys, and the manufacturing and IT procedures of VCAP §12.5 steps 1–3, and recovery of operations that span several EXN messages (for example IPEX grant and admit; VAF §1.2). The platform learns the device identifier, reset epoch, configuration digest and slots of an authenticator from its StateProof.

### 1.3 Design Principles

1. **The platform is untrusted** (VAF §2.2). No request field grants authority; authority comes from the authenticated employee and the enterprise's own records.
2. **The enterprise never signs for the device and never asks for a re-signature** when an outcome is unknown. Every write is idempotent on a stable key and every unknown outcome is resolved by read-only reconciliation (VCAP §12.3).
3. **Outcomes are reported per stage**: device signed, enterprise archived, network accepted, business authorized (VAF §2.3). No response collapses them. The further VAF stage **credential verified** applies only to presentations of a credential (IPEX grant and offer, purposes 10 and 12): it is the receiving verifier's result, reported separately from delivery (§6.3), and is not reported for any other purpose.

---

## 2. Conformance

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED** and **MAY** in this document are to be interpreted as described in BCP 14 [RFC 2119] [RFC 8174] when, and only when, they appear in all capitals, as shown here.

Conformance classes:

- **Enterprise Service:** implements all operations of Section 7 marked *required*, the operations marked *conditionally required* whose condition it meets, and the requirements of Section 10. Section 8 is provisional and not required.
- **Platform:** uses the operations it needs in the order of Section 9, and meets the platform obligations of VCAP §12 and VAF §8.1.

---

## 3. Terminology

Terms of VCAP §3 and VAF §2 apply. Additionally:

| Term | Meaning |
|---|---|
| enterprise service | The service implementing this interface; it combines the Ticket issuer, archive, publisher and gateway roles of VAF §2.1 |
| principal | The tenant-qualified employee identifier the enterprise service derives from the bearer token |
| scope | The (device, resetEpoch, slot, configurationDigest) tuple a request concerns (§6.1) |
| purpose | An enterprise business operation identifier that maps to exactly one (owner, kind) pair (Appendix B) |
| operation | One durable enterprise-side action, identified by an `operationId`, created when a signed message is admitted |
| receiver | The identifier of the destination an EXN / RPY result is delivered to, chosen by the enterprise for each purpose and returned with the Ticket (§7.5) |
| ticketId | The first 16 bytes of `H(ticket_bstr)` (VCAP §3.2), identifying one issued Ticket |

---

## 4. Overview

### 4.1 Roles

```
Employee ──PIN, touch──▶ Authenticator ◀──VCAP──▶ Platform ◀──VCAP-EI (HTTPS, CBOR)──▶ Enterprise service ──▶ KERIA / witnesses / recipients
                                                     ▲                                       │
                                                     └──── OAuth 2.0 bearer token ◀── IdP ◀──┘
```

### 4.2 A Normal Signature

```
Platform                                 Enterprise service
  |-- [challenge] (only if state unknown) ------>|
  |<-- challenge ---------------------------------|
  |   (GetState on authenticator → StateProof)    |
  |-- kel-ticket / message-ticket (original) ---->|  verify state, store original, approve, sign Ticket
  |<-- Ticket ------------------------------------|
  |   (Outbox; Sign on authenticator → Signed)    |
  |-- kel-archive / gateway-submit (Signed) ------------->|  verify, archive, queue publication or delivery
  |<-- archived / operation state ----------------|
  |-- kel-status / gateway-submit (repeat) ------>|  network accepted? business authorized?
```

---

## 5. HTTP Binding

### 5.1 Requests and Responses

1. Every operation is an HTTP `POST` to `{base}/v1/vcap/{operation}` over TLS 1.2 or later (TLS 1.3 RECOMMENDED). `{base}` is the enterprise service origin.
2. Request and response bodies are deterministic CBOR (VCAP §6.1) with media type `application/cbor`. In addition to VCAP §6.1, the simple values `true` and `false` are permitted in fields typed `_bool`. A request body is a CBOR map with unsigned-integer keys; the keys of each operation are listed in Section 7, and a request with extra or missing keys MUST be rejected. Platforms MUST reject responses that do not have the structure given in Section 7. Any added field therefore requires a new version path (`/v2/`).
3. A request body MUST NOT exceed 32 768 bytes; larger requests receive `413`. This accommodates a 16 384-byte KERI message (VCAP Appendix A) with its proof and chain.
4. Responses carry `Cache-Control: no-store`.
5. An enterprise service that serves browser pages from other origins MUST restrict CORS to an explicit list of origins and MUST NOT use `*`.

### 5.2 Status Codes

| HTTP | Meaning | Platform action |
|---|---|---|
| 200 | Success; body is the operation result | Continue |
| 400 | Malformed request, invalid device evidence, or policy rejection of content | Do not retry the same request |
| 401 | Missing or invalid bearer token (`WWW-Authenticate: Bearer`) | Re-authenticate the employee |
| 403 | The principal is not authorized for this operation, asset or slot now | Stop; show the user |
| 404 | The requested record is not retained | Follow §9.3 |
| 409 | An existing operation for this request is in progress or in doubt | **Stop and reconcile** (§9.3); never obtain a new signature to get past it |
| 413 / 415 | Body too large / wrong media type | Fix the client |
| 429 | Rate limit; carries `Retry-After` | Repeat the same request after the indicated time |
| 5xx | Server or network failure; outcome unknown | Retry the **same** request with backoff; writes are idempotent (§10.2) |

Every response other than 200 carries an error body:

```
error = {1: code, ? 2: detail_tstr}
```

`detail` is informative text for logs and MUST NOT be used for decisions. Platforms decide on the HTTP status and `code`:

| code | HTTP | Meaning | Platform action |
|---:|---|---|---|
| 1 | 400 | Malformed request | Fix the client |
| 2 | 400 | Device evidence invalid (StateProof, Signed or challenge) | Obtain new evidence; do not repeat unchanged |
| 3 | 400 | Content rejected by enterprise policy | Show the user; do not repeat |
| 4 | 403 | Principal not authorized for this operation, asset or slot | Stop; show the user |
| 5 | 403 | Scope does not match the current enrollment record (§6.1) | Obtain a new StateProof; call `registration` |
| 6 | 404 | Record not retained | Follow §9.3 |
| 7 | 409 | StateProof required (§7.3) | `challenge` → GetState → repeat with a StateProof |
| 8 | 409 | An operation for this request is in progress or in doubt | Stop and reconcile (§9.3); never obtain a new signature to get past it |
| 9 | 409 | Conflicts with a stored request (different original, challenge reuse) | Reconcile (§9.3) |
| 10 | 429 / 5xx | Temporary failure | Repeat the same request |

VCAP §2 (error precedence unspecified) applies equally here.

### 5.3 Configuration

In 1.0 the platform obtains `{base}`, the IdP (issuer, client registration) and, for browser pages, the RP ID out of band, for example through enterprise device management (MDM) or a configuration file installed by IT. This document defines no discovery document.

### 5.4 Authentication

1. Every request carries `Authorization: Bearer <token>` (RFC 6750). Token issuance (for example OIDC with the enterprise IdP) is out of scope. A native platform SHOULD obtain the token as RFC 8252 specifies, with the authorization code grant and PKCE (RFC 7636) in an external user agent, and SHOULD keep any refresh token in the operating system's protected key store.
2. The enterprise service derives a principal from the token and MUST re-read the principal's current authority over the asset and slot for every request, including on completion of long operations, so that revocation takes effect on the next request (VAF §8.2).
3. The bearer token is independent of the authenticator PIN token. A platform MUST NOT send the PIN, pinHash, shared secret or pinUvAuthToken to the enterprise.

---

## 6. Common Structures

### 6.1 Scope

Every operation except `registration` begins with the scope keys:

```
1: device_b32               ; device identifier (VCAP §6.3)
2: resetEpoch               ; uint
3: slot                     ; 0..3
4: configurationDigest_b32  ; H(TrustConfig envelope) the platform expects
```

The enterprise service MUST reject a request whose (device, resetEpoch, configurationDigest) does not match its current record for an asset assigned to the principal.

### 6.2 Device Evidence

| Name | Encoding | Source |
|---|---|---|
| StateProof | VCAP §9.3 | GetState, CreateIdentity, PrepareRotation |
| Ticket | VCAP §9.2 envelope, as bytes | This interface |
| Signed | VCAP §9.4 map | Sign, UploadEnd, Recover |
| original | The exact message bytes signed | Platform Outbox / enterprise archive |

### 6.3 Operation State

Operations created by `gateway-submit` report one of:

| State | Meaning |
|---|---|
| `QUEUED` | Signature archived; delivery or execution scheduled |
| `COMPLETED` | The enterprise-side action finished; `result` holds its output |
| `IN_DOUBT` | The action may or may not have taken effect; the service is reconciling. The platform MUST NOT request a new signature for the same intent |
| `REJECTED` | Permanently refused by the enterprise or an authority (VCAP §12.8 item 6) |

`COMPLETED` for delivery means the message was handed to the receiver, not that the receiver accepted it; business acceptance is read separately (§7.10).

---

## 7. Operations

Each operation lists its request keys after the scope (§6.1) and its result. *Required* operations are mandatory for an Enterprise Service.

### 7.1 `registration` (required)

Returns the enterprise record of a device the principal holds, so the platform can verify StateProofs and build inception events.

- **Request:** `{1:device_b32, 2:resetEpoch}` (no slot, no digest).
- **Result:** `[1, device_b32, tenant_b32, resetEpoch, enterpriseRoot_b32, attestationPublic_b32, trustConfig_bstr, clientWitnesses[*aid_t44], clientToad, workWitnesses[*aid_t44], workToad]`.
- `clientWitnesses` / `clientToad` and `workWitnesses` / `workToad` are the witness lists and thresholds the enterprise requires in the `b` and `bt` fields of CLIENT and WORK `icp` events.

The platform MUST check that `H(trustConfig)` equals the configurationDigest in the device's StateProof, and that the StateProof verifies under `attestationPublic`, before using any other field.

### 7.2 `challenge` (required)

Issues a single-use challenge for a StateProof (VCAP §12.2 step 0).

- **Request:** scope.
- **Result:** `challenge_b32`.
- The challenge is bound to the principal and the scope; it expires after at most 300 s and is consumed by the first Ticket request that uses it (§10.1).

### 7.3 `kel-ticket` (required)

Requests a Ticket for ICP, IXN or ROT.

- **Request:** scope + `5: StateProof / null`, `6: purpose`, `7: original_bstr`.
- **Result:** `Ticket_bstr`.
- A StateProof in key 5 makes it the state basis; its challenge MUST be one issued by `challenge` (§7.2) for this principal and scope, and the service verifies it with the enrolled attestation public key. A null key 5 selects the enterprise's archived head and counter as the state basis (VCAP §12.8 item 1).
- An enterprise service MAY require a StateProof for every Ticket, and MAY require one for some requests only (for example the first Ticket of a session, or after any `409`). A service that requires one and receives a null key 5 MUST return `409` with error code 7 (§5.2). A platform MUST then obtain a challenge (§7.2), call GetState and repeat the request with key 5 present.
- A service that accepts null keys MUST use only its own archived head and counter as the state basis, never values supplied by the platform.
- A service that implements §8 SHOULD accept a null key 5 for a slot whose archived state is current and not in doubt. On the WebAuthn route each StateProof costs the employee a separate browser call with its own PIN entry and UP, while `webauthn-create-result` and every `webauthn-sign-result` already keep the archived state current (VCAP §12.9).
- The service MUST perform VCAP §12.8 and VAF §8.2 in full before signing, including SAID recomputation, prior-event and key checks, witness policy and, for ROT, `signerKeyRef = KeyRef(old.next)`.
- **Idempotency:** a repeated request with identical bytes returns the same Ticket. A different original for a (scope, sequence) that already has a Ticket returns `409` with error code 9.

### 7.4 `kel-ticket-retry` (optional)

Requests reissue for an **existing frozen original** after a fresh StateProof shows the earlier Ticket was not used (VCAP §12.3 KEL step 4). Same request as §7.3 with key 5 present. The enterprise MUST authorize this operation separately from ordinary issuance and MUST NOT accept a different original.

### 7.5 `message-ticket` (required)

Requests a Ticket for EXN or RPY.

- **Request:** as §7.3.
- **Result:** `[Ticket_bstr, receiver_tstr]`. `receiver` is the destination the enterprise has configured for the purpose; the platform passes it to `gateway-submit` (§7.9).
- The service MUST verify the route, recipients, attachments, embedded material and validity (`dt`) per VAF §5.3 and §7.
- For purpose 11 the service is the verifier (VAF §8.3) of the credential carried by the grant being admitted, and MUST complete the credential trust checks of VAF §8.2 before issuing the Ticket.
- For purpose 15 the original MUST be an `exn` whose `r` is `/challenge/response`, whose `i` is the slot's AID, whose `rp` is the AID of the party that issued the challenge, whose `p` is empty, whose `e` is empty or absent, and whose `a` is exactly `{"words": [...]}` with 12 or 24 words of the BIP-39 English word list. The words reach the employee out of band, so the service cannot check where they came from: its approval establishes only that the employee answers this recipient now. The service SHOULD refuse a second Ticket for the same recipient and words once a response with them has been signed.
- **Idempotency:** a repeated request with identical bytes returns the same result.

### 7.6 `kel-archive` (required)

Archives the device result of a KEL signature and queues its publication.

- **Request:** scope + `5: sequence`, `6: Signed`.
- **Result:** `null`.
- The service MUST verify `Signed` against the archived original and the expected public key (VCAP §9.4), check that `signCounter` matches an issued Ticket, and store the signature and the publication task in one transaction. Repeating the request with the same Signed succeeds; a different Signed for the same (scope, sequence) is `400`.

### 7.7 `kel-original` (required)

Returns the archived original and, if archived, the device result (VCAP §12.3 KEL step 3).

- **Request:** scope + `5: said_t44`, `6: originalDigest_b32`.
- **Result:** `[original_bstr, Signed_bstr / null]`; `404` if not retained.

### 7.8 `kel-status` (required)

- **Request:** as §7.7.
- **Result:** `[networkAccepted_bool, businessAuthorized_bool]`.
- `networkAccepted` follows VAF §6.4 and the acceptance conditions of the KERI binding (witness receipts ≥ `bt`, and for ROT the commitment check).
- `businessAuthorized` is true once the enterprise-side action that the purpose names (Appendix B, for example agent creation for purpose 1 or delegation approval for purpose 2) has completed. For a purpose without such an action it equals `networkAccepted`.

### 7.9 `gateway-submit` (required)

Submits a signed EXN or RPY message for delivery or execution.

- **Request:** scope + `5: purpose`, `6: original_bstr`, `7: ticketId_b16`, `8: Signed`, `9: receiver_tstr`.
- **Result:** `[operationId_b32, state, result / null]`.
- Repeating the identical request returns the current state of the same operation; this is also how the platform reads the outcome of an `IN_DOUBT` operation if the service does not offer §7.10.
- `receiver` MUST equal the value returned by `message-ticket` (§7.5); the service rejects any other value, so a redirection of the platform becomes visible.
- The service MUST archive the signature before any delivery side effect, and MUST deliver the **same** signed message on every retry. An `IN_DOUBT` operation is never resolved by a new signature.

### 7.10 `operation-status` (optional)

- **Request:** scope + `5: operationId_b32`.
- **Result:** `[state, result / null, businessAuthorized_bool / null]`.

### 7.11 `agent-session` (conditionally required: if the service offers purposes 1 and 2)

Relays one request to the employee's Agent, per the KERIA agent profile (§1.2). The request is authorized by the bearer token alone (§5.4); it needs no authenticator, PIN or touch, and a platform MUST NOT require the authenticator for it.

- **Request:** scope + `5: walletProfile_tstr`, `6: clientAid_t44`, `7: agentAid_t44`, `8: path_tstr`, `9: method_tstr`, `10: body_bstr / null`, `11: headers`, `12: requestId_b16 / null`.
  - `path` is the request path, optionally followed by `?` and a query string of `name=value` pairs joined by `&`, percent-encoded.
  - `method` is the uppercase HTTP method.
  - `headers = [* [name_tstr, value_tstr]]`, names in lowercase.
- **Result:** `[1, status, contentType_tstr / null, body_bstr, contentRange_tstr / null]` from the Agent response, at most 65 536 bytes in total (Appendix C.1).
- `walletProfile` names a wallet operation profile of Appendix C that the enterprise has enabled for the principal.
- The service MUST check, on every request, current employee authority over the asset and slot (§5.4 item 2), that `clientAid` is the CLIENT AID archived for this slot with completed business acceptance and `agentAid` the Agent created from it, and that the request matches the named wallet operation profile (Appendix C). Any other request is `403`.
- Writes MUST carry a `requestId`; a repeated `requestId` returns the stored outcome, and a write whose outcome is unknown returns `409` rather than being re-dispatched. Reads MUST NOT carry one.

### 7.12 `agent-material` (conditionally required: if the service offers purposes 1 and 2)

Returns the material the platform needs to approve Agent delegation (VCAP §12.5 step 8).

- **Request:** scope + `5: clientAid_t44`.
- **Result:** `[clientIcp_bstr, agentDip_bstr, [*agentSignature_tstr]]`.

### 7.13 `aid-status`, `reset-reason` and `reset-report` (optional)

- `aid-status`: scope + `5: aid_t44` → `active_bool`. Lets a platform stop using an AID the enterprise has deactivated (VCAP §12.7).
- `reset-reason`: scope → `[state_tstr, reason_bstr] / null`. Tells the platform why the enterprise requires a Reset, if it does.
- `reset-report`: `{1:device_b32, 2:bindingProof}` (no scope) → `[state_tstr, resetEpoch]`, where `state` is `"recorded"` (a newer epoch was recorded now), `"known"` (the epoch was already on record) or `"pending-personalization"` (recorded, and IT has not yet issued a TrustConfig for it). Reports a Reset per §9.5. A BindingProof that does not verify, or whose tenant or root digest is not this enterprise's, is `403`. A report with an epoch lower than the record is `409`, because an epoch never decreases. The service MUST be idempotent per (device, resetEpoch).

### 7.14 `message-original` (optional)

Returns the archived original of a non-KEL message and, if archived, its device result, for recovery when the platform Outbox is lost (§9.3).

- **Request:** scope + `5: ticketId_b16`.
- **Result:** `[original_bstr, Signed_bstr / null]`; `404` if not retained.

---

## 8. WebAuthn Route Operations (Provisional)

**This section is provisional and non-normative in 1.0.** The VCAP WebAuthn route is an optional authenticator feature (VCAP §2.1, §7.14) which is provisional (VCAP §7.14); a change there may change this section. An Enterprise Service MAY implement this section; conformance to 1.0 does not require it.

Results that carry WebAuthn options are CBOR maps whose keys are the WebAuthn member names as text strings, an exception to §5.1 item 2, so that a page can pass them to the WebAuthn API after conversion.

These operations support VCAP §12.9. The page is served from an RP ID in `webauthnRpIds`; the enterprise service is the Outbox on this route (VAF §8.1).

### 8.1 `webauthn-create-options`

- **Request:** scope (configurationDigest from `registration`) + `5: owner`.
- **Result:** `PublicKeyCredentialCreationOptions` fields as CBOR: `rp.id`, `user.id = "VC1" ‖ CBOR({1:slot, 2:owner})`, `challenge_b32`, `pubKeyCredParams = [-65700]`, `attestation = "none"`, `userVerification = "required"`.

### 8.2 `webauthn-create-result`

- **Request:** scope + `5: clientDataJSON_bstr`, `6: attestationObject_bstr`.
- **Result:** `[slotHandle_bstr, currentPublic_b32, nextPublic_b32]`.
- The service verifies per VCAP §12.9.1 step 3 and records the slot handle (step 4).

### 8.3 `webauthn-sign-options`

- **Request:** scope + `5: purpose`, `6: original_bstr`.
- **Result:** `PublicKeyCredentialRequestOptions` fields: `rpId`, `challenge_b32`, `allowCredentials = [VCAPREQ]`, `userVerification = "required"`.
- The service performs the full Ticket issuance of §7.3 / §7.5 and places the Ticket inside VCAPREQ (op 1). If VCAPREQ would exceed 1023 bytes it returns `400` and the platform uses the native route.
- The state basis is the archived state of the slot (§7.3, null key 5): the StateProof verified by `webauthn-create-result` for the slot's first Ticket, and thereafter the result of the last `webauthn-sign-result` or native archive. The page calls no GetState first. When the slot is in doubt, the service returns `409` with error code 7 and the page reconciles through §8.5 before repeating the request.

### 8.4 `webauthn-sign-result`

- **Request:** scope + `5: clientDataJSON_bstr`, `6: authenticatorData_bstr`, `7: signature_bstr`.
- **Result:** as §7.6 (KEL) or §7.9 (non-KEL).
- The service verifies `deviceSig` with the enrolled attestation key, the Signed result, and that `signCount` equals the Ticket's `signCounter` (VCAP §12.9.2 step 3), then archives and continues as for the native route.

### 8.5 `webauthn-state-options`

Returns request options for a wrapped GetState (op 2) or Recover (op 4) with a fresh challenge, for the unknown-outcome procedure of VCAP §12.9.3.

- **Request:** scope + `5: op` (2 or 4), `6: originalDigest_b32` (optional for op 2, required for op 4).
- **Result:** `PublicKeyCredentialRequestOptions` fields as in §8.3, with VCAPREQ carrying the op, the slot handle and, when given, key 6.
- For a slot in doubt, the service SHOULD request op 2 with the `originalDigest` of the in-doubt Ticket. The result carries the StateProof and, if the signature was made, the Signed record, so one browser call resolves the outcome (VCAP §7.14.5). The service then archives the Signed record, or, if nothing was signed, keeps the Ticket valid for a new `webauthn-sign-options` request started by the user.
- The page submits the result of a call made with these options to `webauthn-sign-result` (§8.4). The service recognizes it by the single-use challenge it issued here, which records the op and, when given, the `originalDigest`; it verifies `deviceSig` and the StateProof with the enrolled attestation key and processes the result per VCAP §12.9.3, never as a new signature.

---

## 9. Procedures

### 9.1 Onboarding (VCAP §12.5 steps 4–10)

1. GetState with a challenge the platform generates itself, to learn device, resetEpoch and configurationDigest; `registration` → verify the TrustConfig digest and the StateProof (§7.1). Before the PIN, a platform identifies the device as follows. If status-mode GetState reports ENTERPRISE_BOUND or PERSONALIZED_NO_PIN, binding-mode GetState (VCAP §7.4.3) with a challenge the platform generates gives device, resetEpoch and configurationDigest, which the platform checks against `registration` and the registered `attestationPublic` before asking for the PIN. A PIN_READY device reports no device identifier without a PIN, so the platform confirms which device it is only from the StateProof of this step, after the PIN, and MUST NOT request a Ticket before that StateProof verifies.
2. `challenge` → CreateIdentity(CLIENT) with that challenge → `kel-ticket` (purpose 1) with the returned StateProof and an `icp` using `clientWitnesses` / `clientToad`; Sign; `kel-archive`. The enterprise creates the agent from the archived ICP; `kel-status` reports `businessAuthorized` when it has.
3. `agent-material` → build CLIENT IXN; `kel-ticket` (purpose 2); Sign; `kel-archive`.
4. `challenge` → CreateIdentity(WORK) → `kel-ticket` (purpose 3) with an `icp` using `workWitnesses` / `workToad`; Sign; `kel-archive`; wait for `kel-status` network acceptance.
5. Build RPY end-role; `message-ticket` (purpose 14); Sign; `gateway-submit`.

Steps 1–5 run in one context: one PIN entry and four UPs, one per signature (VCAP §12.5). After step 3 the platform reads the Agent with `agent-session` and needs the authenticator only for later signatures.

The device arrives in PERSONALIZED_NO_PIN (VCAP §12.5); in step 1 the platform asks the employee to choose a PIN (ClientPIN setPIN, at least the TrustConfig `minPinChars`) after the BindingProof checks and before any other command.

The registration of the device and employee by the enterprise is the approval of onboarding. For purposes 1, 2, 3 and 14 the service SHOULD issue Tickets and record business acceptance without a further human decision once the request passes the checks of §7.3 and §7.5: the witness policy of `registration`, the Agent material of `agent-material`, a WORK inception whose controlling CLIENT is the one archived for this asset, and an end-role that names the Agent created for this employee. A service that requires manual review MUST say so in the `kel-status` / `operation-status` result rather than leaving the platform waiting without a state.

### 9.2 Signing

As VCAP §12.2, with steps 0, 1 and 7 implemented by `challenge`, `kel-ticket` / `message-ticket`, and `kel-archive` / `gateway-submit`.

### 9.3 Recovery

| Situation | Platform action |
|---|---|
| Ticket request returned 5xx | Repeat the identical request |
| Ticket request returned 409, code 7 | `challenge` → GetState → repeat the request with the StateProof |
| Ticket request returned 409, code 8 or 9 | `challenge` → GetState → compare with expectation; `kel-ticket-retry` only if the Ticket was not used |
| Sign response lost (KEL) | GetState; if head advanced: `kel-original` (if Outbox lost) → Recover → `kel-archive` |
| Sign response lost (non-KEL) | GetState; if counter advanced and no later signature: `message-original` (if Outbox lost) → Recover → `gateway-submit` |
| `kel-archive` / `gateway-submit` returned 5xx | Repeat with the same Signed |
| `gateway-submit` returned `IN_DOUBT` | Repeat the identical `gateway-submit` (or `operation-status`) with backoff; never re-sign |

### 9.4 Rotation (VCAP §12.6)

PrepareRotation → `kel-ticket` (purpose 5) with the returned StateProof → Sign(ROT) → `kel-archive` → `kel-status`.

### 9.5 Reset (VCAP §12.7)

After Reset the device has a new resetEpoch; the platform calls `registration` with it. Requests for the old epoch return `403`. `aid-status` reports the old AIDs as inactive once the enterprise has deactivated them.

When the platform finds the device in ENTERPRISE_BOUND or PERSONALIZED_NO_PIN (status-mode GetState), or `registration` returns `403` or `404` for the epoch the platform last used, the platform obtains a BindingProof with binding-mode GetState (VCAP §7.4.3) and a challenge it generates itself, and submits it with `reset-report`. The enterprise:

1. verifies the BindingProof under the device's enrolled attestation key and checks the tenant and enterpriseRootDigest;
2. if the reported resetEpoch is higher than its record, records the Reset, deactivates the AIDs of the earlier epochs and stops issuing Tickets for them (VCAP §12.7 step 1, VAF §6.5), whether or not it ordered the Reset;
3. returns what the platform should tell the user: wait for re-personalization by IT, or contact IT.

The platform MUST NOT guess or increment a reset epoch itself.

---

## 10. Enterprise Service Requirements

### 10.1 Challenges

1. Challenges MUST be 32 bytes from a cryptographically secure RNG, bound to (principal, scope), valid for at most 300 s, and consumed by exactly one Ticket request.
2. The service MUST record the request bound to a challenge before it signs a Ticket for it; a second, different request with the same challenge is `409` with error code 9.

### 10.2 Idempotency and Unknown Outcomes

1. Every write is keyed by a stable identifier: (scope, sequence) for KEL, `ticketId` (§3) for non-KEL, `requestId` for Agent writes.
2. A repeated identical request MUST return the stored outcome. A request that conflicts with a stored one MUST be refused.
3. When the outcome of a Ticket signature, an agent call or a delivery is unknown, the service MUST record `IN_DOUBT`, reconcile read-only, and MUST NOT re-dispatch the effect or ask the platform for a new device signature.

### 10.3 Ticket Issuance

The service MUST meet VCAP §12.8 and VAF §5.3 and §8.2. In addition:

1. It MUST select the role key itself from TrustConfig by the purpose's capability bit (Appendix B); the request carries no key choice.
2. It MAY apply an enterprise-side issuance window to unused Tickets (for example refusing to deliver a stored Ticket after a time). Such a window is enterprise policy; the authenticator enforces the signature counter (VCAP §13.2 item 2).

### 10.4 Archive

The original and its Signed result MUST be stored durably before the service reports "archived", and MUST be retrievable by (device, resetEpoch, slot, said, originalDigest) for the platform's authorized recovery.

---

## 11. Security and Privacy Considerations

1. **Bearer token theft** lets an attacker request Tickets for content the enterprise approves, but not produce signatures: those still need the device, PIN and touch. Enterprises SHOULD bind tokens to the client (for example DPoP, RFC 9449) and keep them short-lived.
2. **Content approval** is the main control against induced signing (VCAP §13.2 item 1). The approval UI SHOULD present the decoded message, not just its purpose.
3. **StateProof and BindingProof disclosure**: StateProofs carry all four slots and a persistent device identifier, and BindingProofs carry the device identifier, tenant and reset epoch (VAF §9.3). The service MUST NOT forward them to Agents, receivers or third parties.
4. **Agent relay** is limited to the requests of the declared wallet operation profile (§7.11), which keeps `agent-session` from acting as a general proxy to the Agent. It is authorized by the bearer token only, so a stolen token exposes the Agent's data and its metadata writes until revoked, but no slot signature. Native platforms SHOULD protect the refresh token with the operating system key store and local user verification, and SHOULD rotate it on every use.
5. **Receiver echo** (§7.9) prevents a compromised platform from redirecting delivery without detection, but the enterprise remains responsible for choosing receivers.

---

## Appendix A Operation Table

| Operation | Class | Request keys after scope | Result |
|---|---|---|---|
| registration | required | (1:device, 2:resetEpoch only) | enrollment record |
| challenge | required | — | challenge |
| kel-ticket | required | 5 StateProof?, 6 purpose, 7 original | Ticket |
| kel-ticket-retry | optional | as kel-ticket, 5 required | Ticket |
| message-ticket | required | as kel-ticket | [Ticket, receiver] |
| kel-archive | required | 5 sequence, 6 Signed | null |
| kel-original | required | 5 said, 6 originalDigest | [original, Signed?] |
| kel-status | required | 5 said, 6 originalDigest | [networkAccepted, businessAuthorized] |
| gateway-submit | required | 5 purpose, 6 original, 7 ticketId, 8 Signed, 9 receiver | [operationId, state, result] |
| operation-status | optional | 5 operationId | [state, result, businessAuthorized] |
| agent-session | conditionally required (purposes 1, 2) | 5 walletProfile, 6–12 | Agent response |
| agent-material | conditionally required (purposes 1, 2) | 5 clientAid | [clientIcp, agentDip, agentSigs] |
| aid-status | optional | 5 aid | active |
| reset-reason | optional | — | [state, reason] / null |
| reset-report | optional | (1:device, 2:BindingProof only) | [state, resetEpoch] |
| message-original | optional | 5 ticketId | [original, Signed?] |
| webauthn-* | provisional | Section 8 | Section 8 |

## Appendix B Purpose Registry

| Purpose | Operation | Owner | Kind | Capability bit |
|---:|---|---|---|---:|
| 1 | CLIENT inception and agent creation | CLIENT | ICP | 0 |
| 2 | Agent delegation approval | CLIENT | IXN | 1 |
| 3 | WORK inception | WORK | ICP | 8 |
| 4 | WORK interaction (seal anchoring) | WORK | IXN | 9 |
| 5 | WORK rotation | WORK | ROT | 10 |
| 10 | IPEX grant | WORK | EXN | 11 |
| 11 | IPEX admit | WORK | EXN | 11 |
| 12 | IPEX offer | WORK | EXN | 11 |
| 13 | Remote-signing response (EXN sent on behalf of the WORK AID at a peer's request) | WORK | EXN | 11 |
| 14 | Agent end-role (`/end/role/add`) | WORK | RPY | 12 |
| 15 | Challenge response (`/challenge/response` EXN answering a peer's identity challenge, §7.5) | WORK | EXN | 11 |

Ranges:

| Range | Use |
|---|---|
| 1..127 | Assigned by this document. Values 6–9 and 16–18 are unassigned; 19 (formerly the AGENT session) MUST NOT be reassigned; 20–127 are reserved |
| 128..65535 | Enterprise-defined. An enterprise MUST map each such value to exactly one (owner, kind) pair of VCAP Appendix B, and a platform MUST NOT assume interoperability for them |

A purpose determines the capability bit the service checks and the role key it selects (§10.3 item 1); it is not sent to the authenticator. The route, recipients and embedded material of an EXN are checked by the service against the purpose.

## Appendix C Wallet Operation Profiles

An `agent-session` request names a profile (§7.11 key 5). The profile fixes exactly which Agent requests `agent-session` relays. A request outside the profile MUST be refused. A profile MUST NOT include KEL events, RPY, EXN, agent creation, credential issuance or controller operations, because those need a slot signature and a Ticket.

### C.1 Profile `"vaf"`

The Agent requests of a holder wallet, in the forms that signify-ts at commit `62a646f1eb67c30c8f1aecfe628d81dd693b60f2` sends them, including KERI challenge–response, which a holder uses to prove control of its AID to a peer such as a QVI. Path segments in braces are single percent-decoded segments that MUST NOT be empty, `.`, `..`, or contain `/`, `\`, `%` or NUL.

**Reads** (no `requestId`):

| Method | Path | Allowed query parameters |
|---|---|---|
| GET | `/identifiers` | — |
| GET | `/identifiers/{name}` | — |
| GET | `/identifiers/{name}/oobis` | `role` |
| GET | `/operations/{name}` | — |
| GET | `/config` | — |
| GET | `/contacts` | `group`, `filter_field`, `filter_value` |
| GET | `/contacts/{pre}` | — |
| GET | `/notifications` | — |
| GET | `/exchanges/{said}` | — |
| GET | `/credentials/{said}` | — |
| GET | `/registries/{ri}/{said}` | — |
| GET | `/schema/{said}` | — |
| GET | `/challenges` | `strength`: `128` or `256` |
| POST | `/credentials/query` | body: JSON with exactly `filter` (object), `sort` (array of strings, each 1..256), `skip` (0..2^31−1), `limit` (1..100); ≤ 8192 bytes |

**Writes** (`requestId` required):

| Method | Path | Body |
|---|---|---|
| PUT | `/identifiers/{name}` | JSON with exactly `name` (string, 1..256) |
| PUT | `/contacts/{pre}` | JSON, ≤ 8192 bytes |
| DELETE | `/contacts/{pre}` | none |
| POST | `/oobis` | JSON with `url` (http/https, no userinfo or fragment) and optional `oobialias` |
| DELETE | `/credentials/{said}` | none; `{said}` is a 44-character `E` SAID |
| PUT | `/notifications/{said}` | none |
| POST | `/challenges_verify/{pre}` | JSON with exactly `words` (array of 12 or 24 strings, each 1..16 characters) |
| PUT | `/challenges_verify/{pre}` | JSON with exactly `said` (a 44-character `E` SAID) |

**Headers:** at most two; `accept` with value `application/json` or `application/json+cesr`, and, for GET `/identifiers` and `/notifications` only, `range` of the form `aids=a-b` or `notes=a-b` with `b − a < 100`. Write requests carry no headers and no query string.

**Response:** `[1, status, contentType / null, body_bstr, contentRange / null]`, at most 65 536 bytes; larger results MUST be paginated, never truncated.

GET `/challenges` returns words for the employee to give to a peer. POST `/challenges_verify/{pre}` asks the Agent to confirm that `pre` has sent a signed response containing those words; it returns a long-running operation read with GET `/operations/{name}`. PUT `/challenges_verify/{pre}` records the response `said` as accepted. The employee's own response to a peer's challenge is not an Agent request: it is an EXN signed under purpose 15 and delivered with `gateway-submit` (§7.9).

### C.2 Registering a Profile

A new profile is added to this appendix with a new name. Its entry lists every (method, path, query, header, body) form it allows, in the format of C.1. Changing an existing profile's list requires a new name.

## Appendix D References

### D.1 Normative References

- [BCP14] RFC 2119 and RFC 8174 — <https://www.rfc-editor.org/info/bcp14>
- [VCAP] VCAP 1.0 — <https://vlei-auth.github.io/vcap/>
- [VAF] VAF 1.0 — <https://vlei-auth.github.io/vaf/>
- [RFC6750] The OAuth 2.0 Authorization Framework: Bearer Token Usage — <https://www.rfc-editor.org/rfc/rfc6750>
- [RFC8949] CBOR — <https://www.rfc-editor.org/rfc/rfc8949>
- [RFC9110] HTTP Semantics — <https://www.rfc-editor.org/rfc/rfc9110>
- [RFC8446] TLS 1.3 — <https://www.rfc-editor.org/rfc/rfc8446>; [RFC5246] TLS 1.2 — <https://www.rfc-editor.org/rfc/rfc5246>

### D.2 Informative References

- [RFC9449] OAuth 2.0 Demonstrating Proof of Possession (DPoP) — <https://www.rfc-editor.org/rfc/rfc9449>
- [RFC8252] OAuth 2.0 for Native Apps — <https://www.rfc-editor.org/rfc/rfc8252>
- [RFC7636] Proof Key for Code Exchange (PKCE) — <https://www.rfc-editor.org/rfc/rfc7636>
- [WEBAUTHN] W3C, Web Authentication Level 3 — <https://www.w3.org/TR/webauthn-3/>
- [SIGNIFY-TS] signify-ts, source of the request forms of the `"vaf"` profile (Appendix C.1) — <https://github.com/WebOfTrust/signify-ts>
