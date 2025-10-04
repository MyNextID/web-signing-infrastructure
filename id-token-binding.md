# Binding a PassKey to an ID Token Using OpenID Connect (OIDC)

**Version:** 0.2
**Status:** Draft

## Abstract

This specification defines a mechanism for binding a **WebAuthn credential**
(“PassKey”) to an **OpenID Connect (OIDC) ID Token**. The binding allows a
Relying Party (RP) to cryptographically associate a user’s verified identity
(attested by an Identity Provider) with the cryptographic key pair used for
signing operations. The binding is achieved without modifying IDP
infrastructure, using only the existing OIDC `nonce` parameter.

## 1. Introduction

OpenID Connect (OIDC) enables an **Identity Provider (IDP)** to issue a signed
**ID Token** containing verifiable user claims.  **PassKeys**, based on the
FIDO2/WebAuthn standard, are hardware-backed, phishing-resistant credentials
that require user presence verification.

This document specifies a mechanism to cryptographically bind a user’s PassKey
to an OIDC ID Token using the `nonce` field. The result is a verifiable linkage
between **identity assertions** (from the IDP) and **key control** (from the
PassKey), without requiring any modifications to IDPs.

## 2. Terminology

The key words **MUST**, **SHOULD**, and **MAY** are to be interpreted as described in [RFC 2119].

| Term                 | Definition                                                                  |
| -------------------- | --------------------------------------------------------------------------- |
| **IDP**              | Identity Provider supporting OIDC and issuing signed ID Tokens.             |
| **RP**               | Relying Party or client application consuming the ID Token.                 |
| **PassKey**          | A WebAuthn public key credential, as defined in [WebAuthn Level 3].         |
| **PassKey ID (PID)** | The JWK thumbprint (RFC 7638) of the PassKey’s public key.                  |
| **ID Token (IDT)**   | A JWT issued by an IDP containing user identity assertions (OIDC Core 1.0). |
| **Nonce**            | A client-defined challenge included in an OIDC request to prevent replay.   |

## 3. Protocol Overview

Pre-requisite: End-user registered her PassKey with the RP.
The binding protocol operates as follows:

```
 End-User        RP (Client)           IDP
    |                |                  |
    |--(1) Select IDP-----------------> |
    |                |                  |
    |<-(2) Authn Request (nonce=PID+rand) 
    |                |                  |
    |                |<-(3) Authenticate|
    |                |                  |
    |<-(4) ID Token---------------------|
    |                |                  |
    |--(5) Verify ID Token & PassKey--> |
```

Note: the flow can be also implemented on the IDP side which would make the
trust model stronger.

## 4. Nonce Construction

The `nonce` binds the ID Token to a specific PassKey. It encodes:

* A version identifier
* The PassKey’s JWK thumbprint (`PID`)
* A random challenge

Encoding:

```
version <- 1
rand    <- random_bytes(16)      # at least 128 bits
nonce   <- <version>.<BASE64URL(PID)>.<BASE64URL(rand)>
```

**Example:**

```
1.qYSgm25L-aHPK-KTA7Rn7Xr91khUTnkKa7rqyEWU2WA.NzY3d2VmcWRmZ3dlcQ
```

**Requirements:**

* The `version` field **MUST** be present and set to `1`.
* `rand` **MUST** contain at least 128 bits of entropy.
* The `PID` **MUST** be computed per RFC 7638.
* The IDP **MUST** return the nonce unchanged in the ID Token.
* The ID Token **MUST** be signed by the IDP.

## 5. Procedures

### 5.1 PassKey Registration

When registering:

1. RP initiates WebAuthn registration (`navigator.credentials.create()`).
2. Extract the PassKey’s public key.
3. Compute the PID = JWK thumbprint.
4. Store the PID and public key with the user’s RP account.

### 5.2 OIDC Authentication Request

RP sends the OIDC request with a constructed nonce:

```
GET /authorize?
  response_type=code
  &scope=openid%20profile%20email
  &client_id=s6BhdRkqt3
  &state=af0ifjsldkj
  &nonce=1.<PID>.<rand>
  &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
```

### 5.3 Verification

On receiving an ID Token, the RP:

1. Validates ID Token signature.
2. Extracts nonce → `version`, `PID`, `rand`.
3. Verifies:

   * Nonce matches request.
   * `rand` has not been replayed (RP MUST track state or TTL).
   * `PID` matches registered PassKey’s public key.

If successful, the RP can assert that the ID Token identity is bound to the PassKey.

## 6. Security Considerations

* **Replay Protection:** Each `rand` MUST be unique and unexpired. RPs MUST reject reused or stale values.
* **Binding Strength:** Assurance derives from both the ID Token signature (by IDP) and PassKey cryptography.
* **Nonce Entropy:** Nonces MUST use cryptographically secure random sources.
* **Malicious RP Injection:** If an RP constructs nonces dishonestly, users might be misled. Clients MAY display nonce contents for transparency.
* **Cross-RP Collisions:** Because nonces are arbitrary strings echoed by the IDP, implementers MUST prevent sharing or leakage across unrelated RPs.
* **Multi-device PassKeys:** When PassKeys sync across devices, PIDs remain stable (same keypair), but implementers SHOULD account for device migration.
* **Audience Binding:** ID Tokens MUST be validated against the RP’s `client_id` to prevent token replay at another RP.

## 7. References

* [RFC 7638] JSON Web Key (JWK) Thumbprint
* [RFC 2119] Key words for use in RFCs
* [OIDC Core 1.0] OpenID Connect Core 1.0 (with errata set 1)
* [WebAuthn Level 3] W3C Web Authentication: An API for accessing Public Key Credentials

## Appendix A: Example End-to-End Flow

1. **Registration:** User registers PassKey → RP computes and stores PID.
2. **Authentication:** RP sends OIDC request with nonce `1.<PID>.<rand>`.
3. **ID Token:** IDP issues signed ID Token with nonce echoed.
4. **Verification:** RP verifies ID Token signature, checks nonce freshness, validates PID.
5. **Result:** RP can assert that the PassKey corresponds to the identity asserted in the ID Token.
