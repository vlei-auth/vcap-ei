# VCAP Enterprise Interface (VCAP-EI) 1.0

*Candidate Specification — 11 November 2026*

**Specification identifier:** VCAP-EI-1.0  
**URL:** https://vlei-auth.github.io/vcap-ei/  
**Status:** Candidate Specification  
**Editors:**
  * Fan Dang (ACentauri Technology Limited)

**Scope:** The interface between a platform (a wallet or other VCAP client) and an enterprise service that issues Tickets, archives signatures, publishes KERI material and relays Agent access for VCAP authenticators.

**Companion documents:** [VAF 1.0](https://vlei-auth.github.io/vaf/) (architecture and obligations), [VCAP 1.0](https://vlei-auth.github.io/vcap/) (authenticator protocol). This document uses the data structures of VCAP §9 unchanged.

**Abstract:** VCAP defines what the authenticator accepts; VAF defines what the enterprise must check. Neither defines how a platform asks an enterprise for a Ticket, submits a result or recovers a lost one. This document defines that interface, so that a platform written by one party can use an enterprise service written by another.

---

## Status of this document

This is a **Candidate Specification** of VCAP-EI 1.0, dated 11 November 2026. Its features are frozen: until it becomes final it changes only to correct errors, remove ambiguity, or resolve findings of the review below, and every such change is listed in the change log. It becomes the final 1.0 when (1) at least one Enterprise Service and one Platform, written independently of each other's code, pass the VAF conformance tests including VAF BASE-T15–T20 on real authenticators; (2) an implementation written only from these documents reproduces the test vectors of VCAP Appendix E and completes a signature and its archiving; and (3) an independent security review of VCAP, VCAP-EI and VAF has been completed and its findings resolved. Comments and errata are submitted as issues at <https://github.com/vlei-auth/vcap-ei/issues>.

## License and Intellectual Property

Copyright © 2026 ACentauri Technology Limited and the contributors.

This document is licensed under the Creative Commons Attribution 4.0 International License (CC BY 4.0), <https://creativecommons.org/licenses/by/4.0/>. Code and data published with it, including test vectors and the scripts that generate them, are licensed under the Apache License, Version 2.0, <https://www.apache.org/licenses/LICENSE-2.0>.

As of the date of this document, the editors are not aware of any patent or patent application whose claims would necessarily be infringed by implementing it. Anyone who is aware of such a patent or application is requested to disclose it through the issue tracker above. The licensing terms for any such claims are stated before 1.0 is final.

Names of other specifications and organizations, including FIDO, CTAP, KERI, ACDC, vLEI and GLEIF, are used only to identify the specifications this document relies on. They do not imply endorsement, and implementing this document does not by itself entitle a product to any certification or trademark of those organizations.

## Contents

1. Introduction
2. Conformance
3. Terminology
4. Overview
5. HTTP Binding
6. Common Structures
7. Operations
8. (Reserved)
9. Procedures
10. Enterprise Service Requirements
11. Security and Privacy Considerations
- Appendix A Operation Table
- Appendix B Purpose Registry
- Appendix C Wallet Operation Profiles
- Appendix D References
- Appendix E Change Control

---

## 1. Introduction

### 1.1 Purpose

An authenticator signs only with a Ticket from the enterprise (VCAP §1.1 item 3). Every signature therefore involves two exchanges: platform ↔ authenticator (VCAP) and platform ↔ enterprise (this document). VAF §1.1 states that "any KERI client that implements VCAP and obtains enterprise Tickets can use the same authenticator"; this interface is what makes that statement testable.

### 1.2 Scope

In scope:

- the enrollment record of a device the employee holds;
- single-use challenges for StateProof;
- Ticket requests for KEL and non-KEL kinds, and, for authenticators with group member signing (VCAP §8.4.4), for member signatures on group AID events and messages;
- archiving device results, and retrieving archived originals and signatures for recovery;
- submitting signed EXN / RPY messages for delivery;
- relaying the employee's Agent requests under the bearer token, without the authenticator;
- reading publication, acceptance and business status.

**Agent profile.** The Agent that a CLIENT AID controls (VAF §3.2) is reached through an agent profile. VCAP-EI 1.0 defines one, the **KERIA agent profile**: the enterprise creates a KERIA agent from the CLIENT inception (purpose 1), the CLIENT approves its delegation (purpose 2, `agent-material`), and the platform reaches it through `agent-session` under the employee's bearer token, limited by a wallet operation profile (Appendix C). Agent requests produce no slot signature and do not use the authenticator. A service that offers neither of purposes 1 and 2 does not depend on the agent implementation. Other agent profiles are added in later versions.

Out of scope: employee authentication itself (OAuth 2.0 / OIDC, §5.4), enterprise approval workflows (only their interface behavior is specified, §5.2 and §10.3), the PDP, agent and witness protocols, key custody of the enterprise's role keys, and the manufacturing and IT procedures of VCAP §12.5 steps 1–3, and recovery of operations that span several EXN messages (for example IPEX grant and admit; VAF §1.2). The archiving, dispatch admission, delivery and termination of a single signed message are in scope (§10.4–§10.6). The platform learns the device identifier, reset epoch, configuration digest and slots of an authenticator from its StateProof.

### 1.3 Design Principles

1. **The platform is untrusted** (VAF §2.2). No request field grants authority; authority comes from the authenticated employee and the enterprise's own records.
2. **The enterprise never signs for the device and never asks for a re-signature** when an outcome is unknown. Every write is idempotent on a stable key and every unknown outcome is resolved by read-only reconciliation (VCAP §12.3).
3. **Archiving records a fact; dispatch is a decision.** A signature the authenticator has committed exists whether or not the enterprise still approves of it. The service therefore archives every device result that verifies against a Ticket it issued, independent of approvals, time windows or the principal's current authority (§10.4), and decides separately whether to deliver or execute it (§10.5). Time windows gate Ticket issuance and dispatch, never archiving.
4. **Outcomes are reported per stage**: device signed, enterprise archived, dispatch held or queued, delivered, network accepted, business authorized, and terminated (VAF §2.3). No response collapses them. The further VAF stage **credential verified** applies only to presentations of a credential (IPEX grant and offer, purposes 10 and 12): it is the receiving verifier's result, reported as `PEER_ACCEPTED` when the verifier answers with a signed message (§6.3, Appendix B), and is not reported for any other purpose. A group event or message also passes **threshold met** before it is network accepted or delivered (§7.17).

---

## 2. Conformance

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED** and **MAY** in this document are to be interpreted as described in BCP 14 [RFC 2119] [RFC 8174] when, and only when, they appear in all capitals, as shown here.

Conformance classes:

- **Enterprise Service:** implements all operations of Section 7 marked *required*, the operations marked *conditionally required* whose condition it meets, and the requirements of Section 10.
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
| operation | One durable enterprise-side record of a signed non-KEL message, identified by an `operationId`, created when the message is archived by `gateway-submit` (§7.9); each signed message has exactly one |
| dispatch admission | The service's decision, after archiving, whether a signed message may now be delivered or executed (§10.5) |
| material | Key state, endpoint authorizations, schemas, credential chains and status that the service needs to validate a message or deliver it (§7.18) |
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
  |-- [message-prepare] (purposes 10–12) ------->|  material ready? resolve what is missing
  |<-- ready / missing ---------------------------|
  |-- kel-ticket / message-ticket (original) ---->|  verify state, store original, decide by rule, sign Ticket
  |<-- Ticket  (or 202 code 12 while reviewed) ---|
  |   (Outbox; Sign on authenticator → Signed)    |
  |-- kel-archive / gateway-submit (Signed) ----->|  1. verify and archive (no time condition)
  |                                               |  2. dispatch admission: QUEUED, HELD or CANCELLED
  |<-- archived / operation state ----------------|
  |-- kel-status / operation-status ------------->|  delivered? peer accepted? network accepted? business authorized?
```

The platform needs the authenticator only between receiving the Ticket and archiving. Preparation, review and delivery do not need it, so the platform SHOULD ask the user to connect the authenticator only once a Ticket is available.

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
| 202 | A Ticket request awaits an enterprise review; carries `Retry-After` | Keep the Outbox entry; repeat the **same** request after the indicated time; never obtain a new signature |
| 400 | Malformed request, invalid device evidence, or rule rejection of content | Do not retry the same request |
| 401 | Missing or invalid bearer token (`WWW-Authenticate: Bearer`) | Re-authenticate the employee |
| 403 | The principal is not authorized for this operation, asset or slot now | Stop; show the user |
| 404 | The requested record is not retained | Follow §9.3 |
| 409 | An existing operation for this request is in progress or in doubt (code 8, 9), or a precondition is missing (code 7, 11, 13) | Act on the code; never obtain a new signature to get past it |
| 413 / 415 | Body too large / wrong media type | Fix the client |
| 429 | Rate limit; carries `Retry-After` | Repeat the same request after the indicated time |
| 5xx | Server or network failure; outcome unknown | Retry the **same** request with backoff; writes are idempotent (§10.2) |

Every response other than 200 carries an error body:

```
error = {1: code, ? 2: detail_tstr, ? 3: reviewId_bstr, ? 4: rule_tstr, ? 5: missing}
missing = [+ ("oobi" / "endpoint-auth" / "schema" / "credential-chain" / "tel" / "return-endpoint")]
```

`reviewId` (16 bytes) identifies a pending review and is present exactly with code 12. `rule` names the enterprise rule that produced code 3 or 12; it MAY be shown to the user. `missing` lists the material the service could not obtain and is present exactly with code 13 (§7.18). A rule that is enterprise discretion (VAF §5.3) SHOULD be identified as such in `detail`.

`detail` is informative text for logs and MUST NOT be used for decisions. Platforms decide on the HTTP status and `code`:

| code | HTTP | Meaning | Platform action |
|---:|---|---|---|
| 1 | 400 | Malformed request | Fix the client |
| 2 | 400 | Device evidence invalid (StateProof, Signed or challenge) | Obtain new evidence; do not repeat unchanged |
| 3 | 400 | Content rejected by an enterprise rule. Never used for an expired approval or time window | Show the user; do not repeat |
| 4 | 403 | Principal not authorized for this operation, asset or slot | Stop; show the user |
| 5 | 403 | Scope does not match the current enrollment record (§6.1) | Obtain a new StateProof; call `registration` |
| 6 | 404 | Record not retained | Follow §9.3 |
| 7 | 409 | StateProof required (§7.3) | `challenge` → GetState → repeat with a StateProof |
| 8 | 409 | An operation for this request is in progress or in doubt | Stop and reconcile (§9.3); never obtain a new signature to get past it |
| 9 | 409 | Conflicts with a stored request (different original, challenge reuse) | Reconcile (§9.3) |
| 10 | 429 / 5xx | Temporary failure | Repeat the same request |
| 11 | 409 | The request needs group coordination first: a WORK rotation that would leave the slot outside a group's next-key commitment (VCAP §12.8 item 7.7) | Complete the pending group rotation, then repeat |
| 12 | 202 | The request is retained and awaits an enterprise review | Repeat the same request after `Retry-After` |
| 13 | 409 | Material needed to validate or deliver the message is missing (§7.18) | Show `missing`; supply it or wait, then repeat the same request |

Only `kel-ticket`, `kel-ticket-retry`, `message-ticket` and `group-ticket` return code 12. While the review is pending, a repeated request with identical bytes returns code 12 with the same `reviewId`; after approval it returns the Ticket; after rejection it returns code 3. The service retains the request from the first code 12 response, so the review covers exactly those bytes; a request with different bytes is a new request. The service signs the Ticket when the review approves the request (§10.3 item 4), so the platform can wait for the review without the authenticator.

`gateway-submit` and `kel-archive` never return a 4xx status because an approval, a review or a Ticket window has expired: a verifiable result is archived and its dispatch is reported as an operation state (§6.3).

VCAP §2 (error precedence unspecified) applies equally here.

### 5.3 Configuration

In 1.0 the platform obtains `{base}` and the IdP (issuer, client registration) out of band, for example through enterprise device management (MDM) or a configuration file installed by IT. This document defines no discovery document.

### 5.4 Authentication

1. Every request carries `Authorization: Bearer <token>` (RFC 6750). Token issuance (for example OIDC with the enterprise IdP) is out of scope. A native platform SHOULD obtain the token as RFC 8252 specifies, with the authorization code grant and PKCE (RFC 7636) in an external user agent, and SHOULD keep any refresh token in the operating system's protected key store.
2. The enterprise service derives a principal from the token and MUST re-read the principal's current authority over the asset and slot for every request, including on completion of long operations, so that revocation takes effect on the next request (VAF §8.2). Archiving (§10.4) is the one exception: it requires only that the token authenticates the principal for whom the Ticket was issued, and a revoked authority changes only how the archived result is dispatched (§10.5).
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

The enterprise service MUST reject a request whose (device, resetEpoch, configurationDigest) does not match its current record for an asset assigned to the principal, except that `kel-archive`, `gateway-submit` and `group-archive` are accepted for a scope that matched the record when the Ticket was issued, so that results of a retired epoch or a reassigned asset can still be archived (§10.4).

### 6.2 Device Evidence

| Name | Encoding | Source |
|---|---|---|
| StateProof | VCAP §9.3; carries a `group` element when GetState was called with a group AID | GetState, CreateIdentity, PrepareRotation |
| Ticket | VCAP §9.2 envelope, as bytes | This interface |
| Signed | VCAP §9.4 map | Sign, UploadEnd, Recover |
| original | The exact message bytes signed | Platform Outbox / enterprise archive |

### 6.3 Operation State

`gateway-submit` (§7.9) and `operation-status` (§7.10) report one of these states for the operation of a signed non-KEL message:

| State | Meaning | Platform |
|---|---|---|
| `HELD` | Archived; dispatch is not permitted now (§10.5). A `reason` is given, and a `reviewId` when the reason is `review-required` | Wait; MUST NOT request a new signature for the same intent; MAY terminate with `message-close` |
| `QUEUED` | Archived and admitted for dispatch; no delivery attempt yet | Wait |
| `SENDING` | At least one delivery attempt has started; its result is not yet known | Wait |
| `IN_DOUBT` | An enterprise-side action may or may not have taken effect; the service is reconciling | Wait; MUST NOT request a new signature |
| `TRANSPORTED` | The receiver's endpoint acknowledged the message (for example HTTP 2xx). This is not acceptance by the receiver | Report "delivered" |
| `PEER_ACCEPTED` | The receiver answered with a signed message that references this message's SAID and accepts it (Appendix B) | Report "accepted by the receiver" |
| `COMPLETED` | For a purpose executed by the enterprise rather than delivered (for example purpose 14), the action finished; `result` holds its output | Report "done" |
| `CANCELLED` | Terminated without any delivery attempt; a `reason` is given (§10.6) | Report "terminated, not sent" |
| `REJECTED` | After delivery, the receiver or an authority refused the message (for IPEX, a `/ipex/spurn` referencing it), or VCAP §12.8 item 6 applies | Report the rejection |

Transitions:

- after archiving: `QUEUED`, `HELD` or `CANCELLED` (§10.5);
- `HELD` → `QUEUED` or `CANCELLED`;
- `QUEUED` → `SENDING`, or for a purpose executed by the enterprise `COMPLETED` or `IN_DOUBT`; and, on a revoking event before the first delivery attempt, `HELD` or `CANCELLED` (§10.5);
- `SENDING` → `TRANSPORTED` or `IN_DOUBT`;
- `IN_DOUBT` → `TRANSPORTED` once identical bytes are delivered again (§10.2 item 3), or `COMPLETED` once the enterprise-side action is found to have finished;
- `TRANSPORTED` → `PEER_ACCEPTED` or `REJECTED` when the receiver's answer arrives (Appendix B).

`PEER_ACCEPTED`, `COMPLETED`, `CANCELLED` and `REJECTED` are final. No state is left by obtaining a new signature, and `CANCELLED` is reachable only before the first delivery attempt.

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
- The service MUST perform VCAP §12.8 and VAF §8.2 in full before signing, including SAID recomputation, prior-event and key checks, witness policy and, for ROT, `signerKeyRef = KeyRef(old.next)`.
- For purpose 5 on a slot that holds group records, the service applies VCAP §12.8 item 7.7 and returns `409` with error code 11 when the rotation would leave the slot outside the latest next-key commitment of one of its groups.
- **Idempotency:** a repeated request with identical bytes returns the same Ticket. A different original for a (scope, sequence) that already has a Ticket returns `409` with error code 9.

### 7.4 `kel-ticket-retry` (optional)

Requests reissue for an **existing frozen original** after a fresh StateProof shows the earlier Ticket was not used (VCAP §12.3 KEL step 4). Same request as §7.3 with key 5 present. The enterprise MUST authorize this operation separately from ordinary issuance and MUST NOT accept a different original.

### 7.5 `message-ticket` (required)

Requests a Ticket for EXN or RPY.

- **Request:** as §7.3.
- **Result:** `[Ticket_bstr, receiver_tstr]`. `receiver` is the destination the enterprise has configured for the purpose; the platform passes it to `gateway-submit` (§7.9).
- For purposes 10, 11 and 12 the service MUST first establish that the material of §7.18 is available; if it is not, it returns `409` with error code 13 and the missing items, without retaining an approval or starting a review.
- The service decides each request by rule (VAF §5.3): it issues the Ticket, returns code 12 while a review is pending (§5.2), or rejects with code 3.
- The enterprise-side window for the unused Ticket (§10.3 item 2) starts when the Ticket is signed and MUST NOT be shortened by the remaining lifetime of an approval or review.
- The service MUST verify the route, recipients, attachments, embedded material and validity (`dt`) per VAF §5.3 and §7.
- For purpose 11 the service is the verifier (VAF §8.3) of the credential carried by the grant being admitted, and MUST complete the credential trust checks of VAF §8.2 before issuing the Ticket.
- For purpose 15 the original MUST be an `exn` whose `r` is `/challenge/response`, whose `i` is the slot's AID, whose `rp` is the AID of the party that issued the challenge, whose `p` is empty, whose `e` is empty or absent, and whose `a` is exactly `{"words": [...]}` with 12 or 24 words of the BIP-39 English word list. The words reach the employee out of band, so the service cannot check where they came from: its approval establishes only that the employee answers this recipient now. The service SHOULD refuse a second Ticket for the same recipient and words once a response with them has been signed.
- For purpose 6 the original MUST be an `exn` whose `r` is `/multisig/icp`, `/multisig/ixn`, `/multisig/rot`, `/multisig/exn` or `/multisig/rpy`, whose `i` is the slot's AID, whose embedded event or message is byte for byte an original for which the service has archived this slot's member signature (§7.16, or §7.9 for purposes 20 and 21), and whose recipients are members of the group in its accepted key state.
- For purposes 20 and 21 the service applies VCAP §12.8 item 7.6: for purpose 20 the original is an `exn` whose `i` is a group AID of which the slot is a member, and for purpose 21 an `rpy` for such a group AID. It checks the route and content as for the corresponding single-signer purpose (for example purpose 10 for an IPEX `grant`).
- **Idempotency:** a repeated request with identical bytes returns the same result.

### 7.6 `kel-archive` (required)

Archives the device result of a KEL signature and queues its publication.

- **Request:** scope + `5: sequence`, `6: Signed`.
- **Result:** `null`.
- The service MUST verify `Signed` against the archived original and the expected public key (VCAP §9.4), check that `signCounter` matches an issued Ticket, and store the signature and the publication task in one transaction. As for §7.9 step 1, archiving does not depend on approvals, Ticket windows or the principal's current authority (§10.4). Repeating the request with the same Signed succeeds; a different Signed for the same (scope, sequence) is `400`.

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

Archives a signed EXN or RPY message and submits it for delivery or execution.

- **Request:** scope + `5: purpose`, `6: original_bstr`, `7: ticketId_b16`, `8: Signed`, `9: receiver_tstr`.
- **Result:** `[operationId_b32, state, result / null]`.
- `receiver` MUST equal the value returned by `message-ticket` (§7.5); the service rejects any other value, so a redirection of the platform becomes visible.
- The service processes the request in two steps:
  1. **Archive.** It verifies that it issued the Ticket identified by `ticketId` for this scope, that `original` is the original it stored for that Ticket, and that `Signed` verifies under the signing key the Ticket's `signerKeyRef` designates with `signCounter` equal to the Ticket's. If so it archives the original, Ticket and Signed durably (§10.4). This step MUST NOT depend on any approval, review, Ticket window, message age or the principal's current authority; it requires only that the bearer token authenticates the principal for whom the Ticket was issued.
  2. **Dispatch admission.** It decides per §10.5 whether the message may be delivered or executed now, and records the result as `QUEUED`, `HELD` or `CANCELLED`.
- `operationId` is determined by `(tenant, ticketId, originalDigest)` alone, so each signed message has exactly one operation whatever approvals later apply to it.
- Repeating the identical request returns the current state of the same operation. A platform observes a change from `HELD` to `QUEUED` by repeating the request or with §7.10.
- The service MUST deliver the **same** signed message on every attempt. No state is ever resolved by a new signature.
- A different Signed for a `ticketId` that already has an archived Signed returns `409` with error code 9.
- For purposes 20 and 21 the Signed is one member signature: the service archives it under step 1 and delivers the message to `receiver` only once member signatures meeting the group's thresholds are collected; until then the operation is `QUEUED`.

### 7.10 `operation-status` (required)

- **Request:** scope + `5: operationId_b32`.
- **Result:** `[state, result / null, businessAuthorized_bool / null, detail]`, where

```
detail = {1: archivedAt_ms, ? 2: reason_tstr, ? 3: reviewId_bstr,
          4: deliveryAttempts_uint, ? 5: lastAttemptAt_ms, ? 6: peerEvidenceSaid_t44}
```

- `reason` is present in `HELD` and `CANCELLED`, with a value from §10.5 or §10.6. `reviewId` is present when `reason` is `review-required`; the review it names is the one that will decide dispatch. `peerEvidenceSaid` is the SAID of the receiver's answer in `PEER_ACCEPTED` and `REJECTED`.
- The operation is read only; the request has no effect.

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

### 7.13 `aid-status` (required), `reset-reason` and `reset-report` (optional)

- `aid-status`: scope + `5: aid_t44` → `active_bool`. Lets a platform stop using an AID the enterprise has deactivated (VCAP §12.7, VAF §6.5 item 3).
- `reset-reason`: scope → `[state_tstr, reason_bstr] / null`. Tells the platform why the enterprise requires a Reset, if it does.
- `reset-report`: `{1:device_b32, 2:bindingProof}` (no scope) → `[state_tstr, resetEpoch / null]`, where `state` is `"recorded"` (a Reset was recorded now), `"known"` (the Reset was already on record), `"pending-enrollment"` (recorded, and IT has not yet enrolled the device again) or `"none"` (the BindingProof shows the binding on record, so it shows no Reset); `resetEpoch` is the epoch the BindingProof reports, or null when it shows STOCK_UNBOUND. Reports a Reset per §9.5. A BindingProof that does not verify under the attestation key of a device this enterprise enrolled is `403`. A BindingProof with this enterprise's tenant and root and an epoch lower than the record is `409`, because an epoch never decreases. The service MUST be idempotent per (device, recorded resetEpoch), the epoch whose end the report records.

### 7.14 `message-original` (optional)

Returns the archived original of a non-KEL message and, if archived, its device result, for recovery when the platform Outbox is lost (§9.3).

- **Request:** scope + `5: ticketId_b16`.
- **Result:** `[original_bstr, Signed_bstr / null]`; `404` if not retained.

---

### 7.15 `group-ticket` (conditionally required: if the service offers purposes 7 or 16–18)

Requests a Ticket for a member signature on a group AID event (VCAP kinds 7 and 8, §12.10).

- **Request:** scope + `5: StateProof / null`, `6: purpose`, `7: original_bstr`.
- **Result:** `Ticket_bstr`.
- `purpose` is 16, 17 or 18 (kind GROUP) or 7 (kind GROUP_SUPERSEDE), and the original is a group `icp`, `ixn` or `rot` respectively (for purpose 7, `ixn` or `rot`, or `icp` to reuse the record of an abandoned inception, VCAP §8.4.4 item 6).
- A StateProof in key 5 MUST have been obtained with a challenge from §7.2 and with the event's `i` as group AID (VCAP §7.4.2), so that it carries the slot's group record. A null key 5 selects the service's archived group record for the slot and its own record of the group's accepted key state. For purpose 7, and for any request at a (group AID, `s`) the service has recorded as abandoned, key 5 MUST be present, because only the device knows for certain whether its record holds the abandoned event; with a null key 5 the service returns `409` with error code 7. A member whose group record holds an event other than the one the group's witnesses accepted at the same sequence number signs the accepted event under purpose 7 (VCAP §12.8 items 7.2 and 7.5(a)), with key 5 present; the service approves it only when the record's `seq` equals that sequence number and the original is byte for byte the accepted event.
- The service MUST perform VCAP §12.8 item 7 in full before signing. In particular it approves at most one original per (group AID, `s`) across all members it serves, and for purpose 7 it signs only under a role key that holds capability bit 15 and only after recording a determination on the grounds of VCAP §12.8 item 7.5.
- **Idempotency:** a repeated request with identical bytes returns the same Ticket. A different original for a (group AID, `s`) that already has an approved original returns `409` with error code 9, except for one replacement (VCAP §12.8 items 7.3 and 7.5): once the service has recorded that the approved original was abandoned, the first different original it approves for that (group AID, `s`) becomes the approved original for every member it serves. A member whose group record holds the abandoned event requests it under purpose 7; any other member, including when no member signed the abandoned event, requests it under purpose 17 or 18. Any further different original for that (group AID, `s`) returns `409` with error code 9, and so does a second purpose 7 request for the same (scope, groupAid, `s`), matching the authenticator, which replaces at most once per sequence number (VCAP §8.4.4 item 6).

### 7.16 `group-archive` (conditionally required: if the service offers purposes 7 or 16–18)

Archives the device result of a member signature on a group event.

- **Request:** scope + `5: groupAid_t44`, `6: sequence`, `7: Signed`.
- **Result:** `null`.
- The service MUST verify `Signed` against the archived original and the slot's current key, check that `signCounter` matches an issued Ticket, and store the signature and update its copy of the slot's group record in one transaction. The service does not itself deliver the signature to the other members; the platform does so with the coordination `exn` below.
- The archive entry is keyed by (scope, groupAid, sequence, `Signed.originalDigest`); the original it belongs to is the one whose Ticket carries that digest. A Signed for a (scope, groupAid, sequence) without an archived entry is archived. Otherwise the service processes a request as follows:
  1. The same Signed as an archived entry: success, with no further effect (retry).
  2. A different Signed with the same key as an archived entry: `400`. The authenticator never signs the same group event twice (VCAP §8.4.4 items 5–6).
  3. A Signed for an original approved under purpose 7 as the replacement at this (scope, groupAid, sequence), whose `signCounter` matches that replacement Ticket: archived as the replacement. An earlier entry at the same sequence stays archived and is marked superseded; the platform sends only the replacement in a coordination `exn`. This also applies when the earlier signature was never archived.
  4. Any other Signed for a (scope, groupAid, sequence) that already has an archived entry: `400`.
- The coordination `exn` (`/multisig/icp`, `/multisig/ixn` or `/multisig/rot`) that carries the signature to the other members is a separate signature under purpose 6 (§7.5), delivered with `gateway-submit`.

### 7.17 `group-original` and `group-status` (conditionally required: if the service offers purposes 7 or 16–18)

- `group-original`: scope + `5: groupAid_t44`, `6: said_t44`, `7: originalDigest_b32` → `[original_bstr, Signed_bstr / null]`; `404` if not retained.
- `group-status`: as `group-original` → `[state_tstr, thresholdMet_bool, networkAccepted_bool, replacementSaid_t44 / null]`. `state` is `"pending"`, `"threshold-met"`, `"accepted"` or `"abandoned"`. `thresholdMet` is true once the service holds member signatures that satisfy the event's current threshold and, for `rot`, the prior next threshold. `networkAccepted` follows §7.8. `"abandoned"` means the service has recorded a determination under VCAP §12.8 item 7.5 for this event; `replacementSaid` is then the SAID of the approved replacement once one is agreed, and null before.

### 7.18 `message-prepare` (required for purposes 10–12 if offered; optional otherwise)

Establishes, before any review or Ticket, that the service has the material it needs to validate and deliver a message.

- **Request:** scope + `5: purpose`, `6: original_bstr`.
- **Result:** `[ready_bool, missing]`, where `missing` is as in §5.2 and empty when `ready` is true.
- The service checks, as the purpose requires: the key state of the counterparty or credential issuer (`"oobi"`); that the counterparty's endpoint authorization (`/end/role/add` reply) has been **accepted** by the service's KERI processing, not merely that an OOBI resolution finished (`"endpoint-auth"`); the schema of each credential involved (`"schema"`); its issuer chain (`"credential-chain"`) and current status (`"tel"`); and the endpoint through which the receiver's answer returns (`"return-endpoint"`).
- The service SHOULD obtain missing material itself (for example from the grant's sender, from OOBIs the enterprise configures, or from the schema sources an issuer declares) before it answers.
- The request has no effect other than obtaining material, needs no authenticator, retains no request for review and starts no time window. Repeating it is always permitted.

### 7.19 `message-close` (required)

Terminates the operation of a non-KEL Ticket that cannot or will not be completed, keeping its record.

- **Request:** scope + `5: ticketId_b16`, `6: reason_tstr`, `7: StateProof / null`.
- **Result:** `[state, reason_tstr]`.
- The platform MAY give only the reasons `user-abandoned`, `not-signed` and `signed-unrecovered`; the service accepts each only in this situation:
  - `user-abandoned`: either the Signed is archived and the operation is `HELD` or `QUEUED` with no delivery attempt, or no Signed is archived; key 7 is null. In the second case the service does not yet know whether the Ticket was used, so it leaves the signature counter it relies on unchanged until a later archive or StateProof shows the outcome (§10.6). This lets a user give up without the authenticator.
  - `not-signed`: no Signed is archived, and key 7 is a StateProof, obtained with a challenge from §7.2, whose slot `signCounter` is lower than the Ticket's.
  - `signed-unrecovered`: no Signed is archived, key 7 is a StateProof as above whose slot `signCounter` is at least the Ticket's, and the platform has no Signed for the Ticket (VCAP §12.3).
- The service sets the state to `CANCELLED`, keeps the original, Ticket, any Signed and every decision about them (§10.6), reconciles the slot's signature counter (§10.6), and from then on issues Tickets for the slot as if the operation had completed.
- **Idempotency:** a repeated request returns the same result. A request for an operation in `SENDING` or a later state returns `409` with error code 8.

## 8. (Reserved)

This section number is not used.

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

As VCAP §12.2, with steps 0, 1 and 7 implemented by `challenge`, `kel-ticket` / `message-ticket`, and `kel-archive` / `gateway-submit`. For purposes 10–12 the platform calls `message-prepare` (§7.18) before `message-ticket`. While a request is under review (code 12) the platform does not need the authenticator; it establishes the context of VCAP §12.2 step 2 only once it holds the Ticket.

A platform MUST NOT request a Ticket for a slot while a non-KEL signature it obtained on that slot is not yet archived (VCAP §12.2): the authenticator's `Recover` returns only the slot's most recent signature, so a later signature would leave the platform's Outbox as the only copy.

### 9.3 Recovery

| Situation | Platform action |
|---|---|
| Ticket request returned 5xx | Repeat the identical request |
| Ticket request returned 409, code 7 | `challenge` → GetState → repeat the request with the StateProof |
| Ticket request returned 409, code 8 or 9 | `challenge` → GetState → compare with expectation; `kel-ticket-retry` only if the Ticket was not used |
| Sign response lost (KEL) | GetState; if head advanced: `kel-original` (if Outbox lost) → Recover → `kel-archive` |
| Sign response lost (non-KEL) | GetState; if counter advanced and no later signature: `message-original` (if Outbox lost) → Recover → `gateway-submit` |
| Sign response lost, counter not advanced | Nothing was signed. After the user confirms, Sign again with the same Ticket and original; never automatically. If the service no longer delivers the Ticket, repeat the identical `message-ticket` |
| Signature committed but neither in the Outbox, nor recoverable, nor archived | `message-close` with `signed-unrecovered` and a fresh StateProof |
| Ticket obtained but the user gives up before signing | `message-close` with `user-abandoned`; with the authenticator at hand, preferably `not-signed` with a fresh StateProof, which also settles the counter |
| `kel-archive` / `gateway-submit` returned 5xx or no response | Repeat with the same Signed; an expired approval or window does not prevent archiving |
| `gateway-submit` returned `HELD` | Wait and read `operation-status`; never re-sign. If the user gives up: `message-close` with `user-abandoned` |
| `gateway-submit` returned `QUEUED`, `SENDING` or `IN_DOUBT` | Read `operation-status` with backoff; never re-sign |
| `message-ticket` returned code 13 | Supply the missing material or wait; repeat the identical request |
| Sign response lost (group event) | GetState with the group AID; if the record holds the event: `group-original` (if Outbox lost) → Recover → `group-archive` |
| Sign response lost (GROUP_EXN) | As non-KEL |

### 9.4 Rotation (VCAP §12.6)

PrepareRotation → `kel-ticket` (purpose 5) with the returned StateProof → Sign(ROT) → `kel-archive` → `kel-status`. For a slot that holds group records, the rotation changes the slot's share in each group (VCAP §12.10 item 3), and `kel-ticket` may return `409` with error code 11 (§7.3).

### 9.5 Reset (VCAP §12.7)

After Reset the device is in STOCK_UNBOUND with a new resetEpoch and has no enterprise binding. Once IT has enrolled it again (VCAP §12.7 step 3), the platform calls `registration` with the new epoch. Requests for the old epoch return `403`, except archiving requests (§6.1, §10.4), which are archived and whose operations are `CANCELLED` with reason `aid-retired`. `aid-status` reports the old AIDs as inactive once the enterprise has deactivated them.

When the platform finds a device it used before in STOCK_UNBOUND, ENTERPRISE_BOUND or PERSONALIZED_NO_PIN (status-mode GetState), or `registration` returns `403` or `404` for the epoch the platform last used, the platform obtains a BindingProof with binding-mode GetState (VCAP §7.4.3) and a challenge it generates itself, and submits it with `reset-report`. The enterprise:

1. verifies the BindingProof under the device's enrolled attestation key;
2. if the BindingProof shows STOCK_UNBOUND, a tenant or enterpriseRootDigest other than its own, or a resetEpoch higher than its record, records the Reset, deactivates the AIDs of the earlier epochs and stops issuing Tickets for them (VCAP §12.7 step 1, VAF §6.5), whether or not it ordered the Reset;
3. returns what the platform should tell the user: wait for re-personalization by IT, or contact IT.

The platform MUST NOT guess or increment a reset epoch itself.

### 9.6 Group Participation (VCAP §12.10)

1. The platform reads the group request and the members' key states, for example through `agent-session` under the profile `"vaf-group"` (Appendix C.3), and builds or receives the exact group event bytes.
2. For a group `rot` in which the member rotates its share: PrepareRotation → `kel-ticket` (purpose 5) → Sign(ROT) → `kel-archive`, and wait for `kel-status` network acceptance of the member's own rotation.
3. `group-ticket` (purpose 16, 17 or 18) → Sign(GROUP) → `group-archive`.
4. Build the `/multisig/*` `exn` carrying the event and the indexed signature; `message-ticket` (purpose 6) → Sign(EXN) → `gateway-submit`.
5. Read `group-status` until `thresholdMet` and `networkAccepted`; the platform reports these stages separately from "member signed".
6. If `group-status` reports `"abandoned"` with a `replacementSaid`, obtain the replacement's bytes and repeat steps 3–4 for it, with purpose 7 and a StateProof carrying the group record when the slot's group record holds the abandoned event. A slot replaces at most once per sequence number (VCAP §8.4.4), so the service SHOULD have all members agree on a replacement's bytes before it issues the first replacement Ticket.
7. To sign a message of the group, for example an IPEX `grant` of a credential the group issues or `/end/role/add` for the group AID: `message-ticket` (purpose 20 or 21) → Sign(GROUP_EXN or RPY) → `gateway-submit`; then send the signature to the other members in a `/multisig/exn` or `/multisig/rpy` `exn` under purpose 6 (steps 4–5).

Steps 3 and 4 run in one context: one PIN entry and two touches. Lost responses are reconciled per §9.3, using `group-original` and a StateProof requested with the group AID in place of `kel-original` and a plain StateProof.

---

## 10. Enterprise Service Requirements

### 10.1 Challenges

1. Challenges MUST be 32 bytes from a cryptographically secure RNG, bound to (principal, scope), valid for at most 300 s, and consumed by exactly one Ticket request or `message-close` (§7.19).
2. The service MUST record the request bound to a challenge before it signs a Ticket for it; a second, different request with the same challenge is `409` with error code 9.

### 10.2 Idempotency and Unknown Outcomes

1. Every write is keyed by a stable identifier: (scope, sequence) for KEL; for group events (scope, groupAid, sequence) for `group-ticket`, with the single replacement of §7.15, and (scope, groupAid, sequence, originalDigest) for `group-archive` (§7.16); for non-KEL, `ticketId` (§3) for the Ticket, the archive and `message-close`, and `(tenant, ticketId, originalDigest)` for the operation (§7.9); `requestId` for Agent writes.
2. A repeated identical request MUST return the stored outcome. A request that conflicts with a stored one MUST be refused.
3. When the outcome of a Ticket signature, an agent call or a delivery is unknown, the service MUST record `IN_DOUBT`, reconcile read-only, and MUST NOT ask the platform for a new device signature. It MUST NOT repeat an effect that is not idempotent. It MAY deliver again the **identical signed bytes** of an EXN or RPY to a receiver that discards duplicates by SAID, as KERI receivers do; this is how `IN_DOUBT` delivery is resolved.
4. Before each delivery attempt the service MUST durably record that the attempt started. An operation with no recorded attempt has certainly not been delivered; only an operation with a recorded attempt and no result can be `IN_DOUBT`.
5. Every retry of an archive, admission or delivery uses the original, Ticket, Signed, receiver and attachments stored when the Ticket was issued or the result archived. The service MUST fix at Ticket issuance the attachments it will deliver with the message and the business objects the message refers to (for example the credential and the grant it admits).

### 10.3 Ticket Issuance

The service MUST meet VCAP §12.8 and VAF §5.3 and §8.2. In addition:

1. It MUST select the role key itself from TrustConfig by the purpose's capability bit (Appendix B); the request carries no key choice.
2. It MAY apply an enterprise-side window to unused Tickets (for example refusing to deliver a stored Ticket after a time). Such a window is enterprise policy; the authenticator enforces the signature counter (VCAP §13.2 item 2). The window starts when the Ticket is signed, MUST NOT be shortened by the remaining lifetime of an approval or review, and MUST NOT be used to refuse archiving (§10.4). Its length MAY differ per rule, for example minutes for automatically allowed requests and hours for reviewed ones.
3. It MUST record, for each Ticket decision, the rule that decided it, and for a reviewed request the reviewer. A reviewer MUST NOT be the principal that made the request.
4. When a review approves a request (code 12), the service signs the Ticket at that time and returns it on the next identical request. Before returning a stored Ticket it rechecks only facts that can revoke it: the principal's authority, the device's reset epoch, the AID, the status of each credential involved, and the message age of §10.5 item 5. If one fails, it does not return the Ticket and answers with code 3 or 4. The approval does not otherwise expire while its Ticket is unused within the window of item 2. The state basis of such a Ticket is the one retained with the request (§5.2) or, if newer, the service's archived head and counter; if the slot has signed since, the authenticator rejects the Ticket (`TICKET_INVALID`) without a signature and the platform reconciles per VCAP §12.2.
5. Decisions are append-only. A later decision about the same original or signed message adds a record that names the one it supersedes; it never overwrites an earlier one, and a Ticket stays bound to the decision under which it was issued.

### 10.4 Archive

The original and its Signed result MUST be stored durably before the service reports "archived", and MUST be retrievable by (device, resetEpoch, slot, said, originalDigest) for the platform's authorized recovery.

The service MUST archive every device result that verifies against a Ticket it issued (§7.6, §7.9 step 1, §7.16). It MUST NOT make archiving depend on an approval, a review, a Ticket window, the message's age or the principal's current authority. A result for a retired reset epoch, a deactivated AID or a principal whose authority was revoked is archived and marked as such; it serves audit and recovery only and is never dispatched (§10.5). When the principal can no longer authenticate at all, the enterprise MAY take the result from the platform through its own support procedure and archive it in the same way.

Dispatch admission (§10.5) applies to non-KEL results. The publication of an archived KEL event follows §7.6 and VAF §6.4.

### 10.5 Dispatch Admission

After archiving a non-KEL result, the service decides whether it may be delivered or executed now. It checks, in order:

1. the principal's current authority over the asset and slot; if absent: `HELD`, reason `authority-revoked`;
2. that the AID is active and the reset epoch is current; if not: `CANCELLED`, reason `aid-retired`;
3. the rule that decided the Ticket. If the result was archived within the Ticket's window (§10.3 item 2), the service MAY rely on the decision recorded at issuance. Otherwise it applies the rule set in force now to current facts: allow continues; review gives `HELD`, reason `review-required`, with a new review whose `reviewId` §7.10 reports, and a rejection by that review gives `CANCELLED`, reason `review-rejected`; deny gives `CANCELLED`, reason `rule-denied`;
4. the current status of each credential the message presents or admits; if revoked: `CANCELLED`, reason `credential-revoked`;
5. that the message's own validity (`dt`) is not older than the enterprise's maximum message age; if it is: `CANCELLED`, reason `message-stale`. The enterprise MUST set this maximum no larger than the age of `dt` that the receivers it delivers to accept, and SHOULD verify that limit against each receiver implementation it supports; a message older than its receiver accepts can be `TRANSPORTED` and then be dropped by the receiver without a response. This document defines no default;
6. the receiver echo of §7.9.

If all pass, the operation is `QUEUED`. The service re-evaluates a `HELD` operation when the condition that held it may have changed, for example when the review decides or the principal's authority is restored.

After admission:

- expiry of an approval or window does not stop delivery;
- a revoking event (items 1, 2 and 4, or an explicit withdrawal of the approval) takes effect before the next delivery attempt: with no attempt recorded the operation becomes `HELD` or `CANCELLED`; with an attempt recorded the service only reconciles and records that the revocation came after delivery started;
- after a restart the service resumes queued operations from durable state without a new decision, rechecking only revoking events.

### 10.6 Termination and Audit

An operation is `CANCELLED` by `message-close` (§7.19) or by the service, with one of the reasons `user-abandoned`, `not-signed`, `signed-unrecovered`, `review-rejected`, `rule-denied`, `message-stale`, `credential-revoked`, `aid-retired` or `enterprise-closed`. The service cancels only operations with no delivery attempt.

On termination the service:

1. keeps the original, the Ticket, any Signed, every decision and review, and every admission decision and delivery attempt, for the retention period the enterprise applies to credential evidence;
2. reconciles the slot's signature counter: when the signature was committed (archived, or `signed-unrecovered`), the counter it relies on for the next Ticket is at least the Ticket's `signCounter`; for `not-signed` it is unchanged. When in doubt it requires a fresh StateProof (VCAP §12.8 item 4);
3. treats the Ticket as finished, so that it no longer blocks Tickets for the slot.

The authenticator still accepts an unused Ticket until the slot's counter advances (VCAP §5.3), so a signature can arrive after its operation was cancelled. The service archives it (§10.4), reconciles the counter, and keeps the operation `CANCELLED`; it never dispatches it.

### 10.7 Lifecycle Events

| Event | Archive | Dispatch admission and queued operations | Operations with a delivery attempt | Reading results |
|---|---|---|---|---|
| Reset or device loss (VAF §6.5) | Accepted and marked | Refused; queued operations `CANCELLED` (`aid-retired`) | Reconciled only | Read only, as VAF §8.4 permits |
| WORK rotation | Accepted; verified under the key the Ticket designates | Queued operations continue with the establishment event used to sign. The service SHOULD refuse a ROT Ticket for a slot with `HELD` operations (`409`, code 8) until they are dispatched or cancelled | Unaffected | Unaffected |
| Principal's authority revoked | Accepted and marked | Refused; queued operations `HELD` (`authority-revoked`) | Reconciled only | Per VAF §8.4 |
| Credential involved revoked | Accepted | Refused; queued operations `CANCELLED` (`credential-revoked`) | Reconciled only; the order of revocation and delivery is recorded | Unaffected |
| Approval withdrawn | Unaffected | Queued operations `HELD` (`review-required`) | Reconciled only | Unaffected |

No event removes an archived result, and no event rolls back a delivery that has started.

---

## 11. Security and Privacy Considerations

1. **Bearer token theft** lets an attacker request Tickets for content the enterprise approves, but not produce signatures: those still need the device, PIN and touch. Enterprises SHOULD bind tokens to the client (for example DPoP, RFC 9449) and keep them short-lived.
2. **Content approval** is the main control against induced signing (VCAP §13.2 item 1). The approval UI SHOULD present the decoded message, not just its purpose.
3. **StateProof and BindingProof disclosure**: StateProofs carry all four slots and a persistent device identifier, and BindingProofs carry the device identifier, tenant and reset epoch (VAF §9.3). The service MUST NOT forward them to Agents, receivers or third parties.
4. **Agent relay** is limited to the requests of the declared wallet operation profile (§7.11), which keeps `agent-session` from acting as a general proxy to the Agent. It is authorized by the bearer token only, so a stolen token exposes the Agent's data and its metadata writes until revoked, but no slot signature. Native platforms SHOULD protect the refresh token with the operating system key store and local user verification, and SHOULD rotate it on every use.
5. **Receiver echo** (§7.9) prevents a compromised platform from redirecting delivery without detection, but the enterprise remains responsible for choosing receivers.
6. **Unconditional archiving** (§10.4) lets a platform store a signature after the enterprise has withdrawn its consent. This creates no effect outside the enterprise: the signature exists already, and dispatch admission (§10.5) still decides whether it is sent. Refusing to archive would only leave an unaudited signature on the platform.
7. **Delayed dispatch.** A signature dispatched long after it was made may no longer reflect the signer's current intent. The maximum message age (§10.5 item 5), the recheck of revoking facts, and recipients' own checks of `dt` (VAF §5.3) bound this.

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
| operation-status | required | 5 operationId | [state, result, businessAuthorized, detail] |
| message-prepare | required for purposes 10–12 if offered | 5 purpose, 6 original | [ready, missing] |
| message-close | required | 5 ticketId, 6 reason, 7 StateProof? | [state, reason] |
| agent-session | conditionally required (purposes 1, 2) | 5 walletProfile, 6–12 | Agent response |
| agent-material | conditionally required (purposes 1, 2) | 5 clientAid | [clientIcp, agentDip, agentSigs] |
| aid-status | required | 5 aid | active |
| reset-reason | optional | — | [state, reason] / null |
| reset-report | optional | (1:device, 2:BindingProof only) | [state, resetEpoch] |
| message-original | optional | 5 ticketId | [original, Signed?] |
| group-ticket | conditionally required (purposes 7, 16–18) | 5 StateProof?, 6 purpose, 7 original | Ticket |
| group-archive | conditionally required (purposes 7, 16–18) | 5 groupAid, 6 sequence, 7 Signed | null |
| group-original | conditionally required (purposes 7, 16–18) | 5 groupAid, 6 said, 7 originalDigest | [original, Signed?] |
| group-status | conditionally required (purposes 7, 16–18) | 5 groupAid, 6 said, 7 originalDigest | [state, thresholdMet, networkAccepted, replacementSaid?] |

## Appendix B Purpose Registry

| Purpose | Operation | Owner | Kind | Capability bit |
|---:|---|---|---|---:|
| 1 | CLIENT inception and agent creation | CLIENT | ICP | 0 |
| 2 | Agent delegation approval | CLIENT | IXN | 1 |
| 3 | WORK inception | WORK | ICP | 8 |
| 4 | WORK interaction (seal anchoring) | WORK | IXN | 9 |
| 5 | WORK rotation | WORK | ROT | 10 |
| 6 | Group coordination (`/multisig/icp`, `/multisig/ixn`, `/multisig/rot`, `/multisig/exn`, `/multisig/rpy` EXN carrying the member's signature, §9.6) | WORK | EXN | 11 |
| 7 | Group event replacement (§9.6 step 6) | WORK | GROUP_SUPERSEDE | 15 |
| 10 | IPEX grant | WORK | EXN | 11 |
| 11 | IPEX admit | WORK | EXN | 11 |
| 12 | IPEX offer | WORK | EXN | 11 |
| 13 | (Reserved) | — | — | — |
| 14 | Agent end-role (`/end/role/add`) | WORK | RPY | 12 |
| 15 | Challenge response (`/challenge/response` EXN answering a peer's identity challenge, §7.5) | WORK | EXN | 11 |
| 16 | Group inception (member signature on a group `icp`) | WORK | GROUP | 14 |
| 17 | Group interaction (member signature on a group `ixn`) | WORK | GROUP | 14 |
| 18 | Group rotation (member signature on a group `rot`) | WORK | GROUP | 14 |
| 20 | Group message (member signature on an `exn` of a group AID, for example an IPEX `grant`, §9.6 step 7) | WORK | GROUP_EXN | 16 |
| 21 | Group reply (member signature on an `rpy` of a group AID, for example `/end/role/add`, §9.6 step 7) | WORK | RPY | 12 |

Ranges:

| Range | Use |
|---|---|
| 1..127 | Assigned by this document. Values 8 and 9 are unassigned; 19 is reserved and MUST NOT be assigned; 22–127 are reserved. Purposes 6, 7, 16–18, 20 and 21 serve group AIDs; 7, 16–18 and 20 need an authenticator with group member signing for kinds 7, 8 and 9 (VCAP §2.1) |
| 128..65535 | Enterprise-defined. An enterprise MUST map each such value to exactly one (owner, kind) pair of VCAP Appendix B, and a platform MUST NOT assume interoperability for them |

A purpose determines the capability bit the service checks and the role key it selects (§10.3 item 1); it is not sent to the authenticator. The route, recipients and embedded material of an EXN are checked by the service against the purpose.

Completion of non-KEL purposes (§6.3):

| Purpose | Final state on success | Receiver's answer that gives `PEER_ACCEPTED` / `REJECTED` |
|---:|---|---|
| 6 | `TRANSPORTED`; the group event's progress, including `thresholdMet`, is read with `group-status` (§7.17) | None |
| 10 | `TRANSPORTED`, then `PEER_ACCEPTED` | `/ipex/admit` / `/ipex/spurn` whose `p` is the grant's SAID |
| 11 | `TRANSPORTED`; `result` reports whether the employee's Agent stored the credential | None: an admit ends the IPEX exchange |
| 12 | `TRANSPORTED`, then `PEER_ACCEPTED` | `/ipex/agree` / `/ipex/spurn` whose `p` is the offer's SAID |
| 13, 15 | `TRANSPORTED` | None |
| 14 | `COMPLETED` | None |
| 20 | As the corresponding single-signer purpose, once the threshold is met | As the corresponding single-signer purpose |
| 21 | As purpose 14 | None |

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

### C.3 Profile `"vaf-group"`

The requests of profile `"vaf"` (C.1), and in addition the following reads, which a member uses to learn a pending group request and the group's members (KERIA multisig endpoints):

| Method | Path | Allowed query parameters |
|---|---|---|
| GET | `/multisig/request/{said}` | — |
| GET | `/identifiers/{name}/members` | — |

The profile adds no write. Group inception, joining, every group event and every group message need a member signature and a Ticket (§§ 7.5, 7.15); the enterprise service registers the group with the employee's Agent after `group-archive`.

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
- [SIGNIFY-TS] signify-ts, source of the request forms of the `"vaf"` profile (Appendix C.1) — <https://github.com/WebOfTrust/signify-ts>

## Appendix E Change Control

**Change control.** This document follows these rules from the Candidate onward:

1. **Versions.** A 1.0.x revision only corrects errors: it changes no message bytes, no domain separator or code point, and no outcome for an implementation that followed the intended reading. A 1.y revision only adds: new optional operations, fields behind a new version path, or new registered values; an implementation of 1.0 remains conformant. Any other change requires 2.0.
2. **Section numbers.** Published section and appendix numbers are not reused. A removed section remains as a heading marked "(Reserved)".
3. **Registered values.** Operation names, purposes (Appendix B), error codes (§5.2), operation states and reasons (§6.3, §10.5, §10.6) and wallet operation profiles (Appendix C) are assigned only by a published version of the relevant document. A value is never reassigned; a withdrawn value is marked Reserved.
4. **Errata.** Errata are submitted as issues in the document's repository. An accepted erratum is published in a 1.0.x revision and listed in the change log. Until 1.0 is final, request and result structures may still change in the Candidate without a new version path when a review finding requires it; the rule of §5.1 item 2 applies from the final 1.0 onward. Purposes 8, 9, 13 and 19 are Reserved in 1.0.

**Change log.**

- **11 November 2026, Candidate Specification.** Archiving of a verified signature separated from dispatch admission and delivery (VCAP-EI §§ 6.3, 7.9, 7.18, 7.19, 10.4–10.7; VAF §§ 2.3, 5.3, 6.3; VCAP §12.2, §12.8 item 8). The WebAuthn route removed (formerly VCAP §7.14; VCAP §12.9 and VCAP-EI §8 are now Reserved); the TrustConfig field that listed its RP IDs is now `reserved` and is the empty array (VCAP §9.1); test vectors regenerated (VCAP Appendix E). Purpose 13 withdrawn (VCAP-EI Appendix B). The maximum message age bounded by what receivers accept (VCAP-EI §10.5 item 5). Conformance test GROUP-T14 for groups with an independent software member added (VAF Section 12). License, intellectual-property and change-control terms added.
