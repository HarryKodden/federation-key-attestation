%%%
title = "OpenID Federation Key Attestation Endpoint 1.0 - draft 01"
abbrev = "openid-federation-key-attestation"
ipr = "none"
workgroup = "OpenID Connect Working Group"
keyword = ["security", "openid", "federation", "key-rollover"]

[seriesInfo]
name = "Internet-Draft"
value = "openid-federation-key-attestation"
status = "standard"

[[author]]
initials="H."
surname="Kodden"
fullname="Harry Kodden"
organization="SURF"
    [author.address]
    email = "harry.kodden@surf.nl"

%%%

.# Abstract

This specification acts as an extension to OpenID Federation. It defines
an additional federation endpoint that allows a Subordinate Entity to request
that its Immediate Superior update the attested Federation Entity Keys in the
Subordinate Statement. The request is authenticated by a signature made with a
key the Superior already attests, and the candidate keys MUST already appear in
the Subordinate's Entity Configuration.

{mainmatter}

# Introduction

OpenID Federation defines how Federation Entity Keys are published in an Entity
Configuration and how an Immediate Superior attests a Subordinate's keys in a
Subordinate Statement. Trust Chains verify signatures using keys from the next
Entity Statement in the chain. For a Subordinate that is not a Trust Anchor,
consumers rely on the `jwks` Claim of the Subordinate Statement issued by the
Superior, not solely on the Subordinate's own Entity Configuration.

OpenID Federation Section 11 describes Federation Key Rollover for Entity
Configurations and for Trust Anchors. It does not define a standard protocol by
which a Subordinate asks its Immediate Superior to update the attested key set
in the Subordinate Statement. Implementations today depend on out-of-band
operations, proprietary enrollment flows, or opportunistic refresh at statement
expiry. Those mechanisms do not guarantee that newly published keys are copied
into the attested Subordinate Statement.

This specification defines the Federation Key Attestation Endpoint. A
Subordinate posts a signed Key Attestation Request JWT to its Immediate
Superior. The request is signed with a key already attested by that Superior.
The Superior verifies that each candidate key is present in the Subordinate's
Entity Configuration, then issues an updated Subordinate Statement and makes it
available from its `federation_fetch_endpoint`.

## Rollover Sequence

The endpoint enables the following recommended key rollover sequence for
Intermediates and Leaf Entities:

1. The Subordinate adds the Candidate Key to its Entity Configuration `jwks`
   Claim.
2. The Subordinate continues to sign its Entity Configuration with an Attested
   Key.
3. The Subordinate `POST`s a Key Attestation Request, signed with that Attested
   Key, to the Superior's Federation Key Attestation Endpoint.
4. The Superior publishes an updated Subordinate Statement that includes the
   Candidate Key in its `jwks` Claim.
5. The Subordinate begins signing with the newly attested key and MAY later
   retire previous keys via `federation_historical_keys_endpoint`.

## Rationale and Benefits

The following properties motivate this extension:

- Closes the attestation gap: Publishing a key in an Entity Configuration does
  not by itself make that key usable in a Trust Chain that depends on a
  Superior-issued Subordinate Statement.
- Continuity of control: Signing the request with an already-attested key binds
  the update to the entity the Superior already trusts.
- Independent verification: Requiring candidate keys to appear in the
  Subordinate Entity Configuration lets the Superior re-fetch and verify those
  keys before attestation.
- Compatible with historical keys: After the Superior attests the new key, the
  Subordinate can switch signing keys and later publish retired keys at
  `federation_historical_keys_endpoint` as defined by OpenID Federation.
- Complements Subordinate Events: Where a Superior publishes Subordinate Events,
  a successful attestation update SHOULD be recorded as a `jwks_update` event.

## Requirements Notation and Conventions

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 [@!RFC2119] [@!RFC8174]
when, and only when, they appear in all capitals, as shown here.


# Terminology

This specification uses the terms "Entity", "Entity Configuration", "Entity
Statement", "Federation Entity", "Federation Entity Key", "Trust Anchor",
"Intermediate", "Immediate Superior", "Subordinate", "Subordinate Statement",
and "Trust Chain" as defined in OpenID Federation.

This specification also defines the following terms:

Attested Key
: A Federation Entity Key present in the `jwks` Claim of a Subordinate
  Statement issued by an Immediate Superior about a Subordinate.

Candidate Key
: A Federation Entity Key that a Subordinate requests to be added to the
  attested key set. A Candidate Key MUST already appear in the Subordinate's
  Entity Configuration `jwks` Claim.

Key Attestation Request
: A signed JWT posted to the Federation Key Attestation Endpoint that requests
  an update to the attested key set for the Subordinate identified by the JWT
  `sub` Claim.

Key Attestation Proof
: A nested JWT signed with a Candidate Key's private key and included in the
  Key Attestation Request `key_proofs` Claim. It proves possession of that
  Candidate Key and is bound to the parent request via `attestation_jti`.


# Relationship to OpenID Federation Key Rollover

OpenID Federation Section 11.2 describes Trust Anchor key rollover by
publishing new keys in the Trust Anchor Entity Configuration while continuing
to sign with previously distributed keys. That model works for Trust Anchors
because relying parties obtain Trust Anchor keys out of band or from the Trust
Anchor Entity Configuration itself.

For Intermediates and Leaf Entities, consumers verify Entity Configuration
signatures using keys from the Immediate Superior's Subordinate Statement. A
new key published only in the Subordinate Entity Configuration is therefore not
sufficient for Trust Chain validation until the Superior attests it. The
recommended sequence that closes that gap is described in
(#rollover-sequence).


# Key Attestation Endpoint

The Federation Key Attestation Endpoint is an OPTIONAL endpoint that MAY be
published by Trust Anchors and Intermediate Entities. Leaf Entities MUST NOT
publish this endpoint. The endpoint URL MUST use the `https` scheme and MAY
contain port, path, and query parameter components. It MUST NOT contain a
fragment component.

## Endpoint Location

The location of the Federation Key Attestation Endpoint is published in the
`federation_entity` metadata using the `federation_key_attestation_endpoint`
parameter.

The following is a non-normative example of an Entity Configuration payload for
a Trust Anchor that includes the endpoint. The path component `/key_rollover`
is illustrative; any path MAY be used.

```json
{
  "iss": "https://ta.example.com",
  "sub": "https://ta.example.com",
  "iat": 1792503828,
  "exp": 1792590228,
  "jwks": {
    "keys": [
      {
        "kty": "EC",
        "crv": "P-256",
        "kid": "xcXdyJ2_7cOd05QIqfpdrb3j5-mYFw8dqdcqzEh0lUw",
        "x": "hh5u_VrRXLaX...",
        "y": "..."
      }
    ]
  },
  "metadata": {
    "federation_entity": {
      "federation_fetch_endpoint":
        "https://ta.example.com/fetch",
      "federation_list_endpoint":
        "https://ta.example.com/list",
      "federation_resolve_endpoint":
        "https://ta.example.com/resolve",
      "federation_key_attestation_endpoint":
        "https://ta.example.com/key_rollover"
    }
  }
}
```

## Key Attestation Request

### Request Format

The request MUST be an HTTP POST. The HTTP request body MUST be a Key
Attestation Request JWT. The `Content-Type` of the request MUST be
`application/key-attestation-request+jwt`.

Clients SHOULD set the `Accept` header to prefer
`application/entity-statement+jwt` on success and `application/json` on error.

### JWT Header

The Key Attestation Request is a JWS [@!RFC7515] signed JWT. The JOSE Header
MUST include:

- `alg`: REQUIRED. A signature algorithm suitable for Federation Entity Keys.
  `ES256` is RECOMMENDED.
- `kid`: REQUIRED. Key ID of the Attested Key used to sign the request. The
  value MUST identify a key in the current Subordinate Statement `jwks` for the
  subject.
- `typ`: REQUIRED. MUST be `key-attestation-request+jwt`.

### JWT Claims

The Key Attestation Request JWT Claims are:

- `iss`: REQUIRED. String. Entity Identifier of the Subordinate making the
  request.
- `sub`: REQUIRED. String. Entity Identifier of the Subordinate whose attested
  keys are to be updated. MUST equal `iss`.
- `aud`: REQUIRED. String. Entity Identifier of the Immediate Superior operating
  the endpoint.
- `iat`: REQUIRED. Number. Time at which the request was issued, expressed as
  Seconds Since the Epoch.
- `exp`: REQUIRED. Number. Expiration time of the request.
- `jti`: REQUIRED. String. Unique identifier for the request. Superiors MUST
  reject reused `jti` values within the request lifetime.
- `add_keys`: OPTIONAL. Array of JWK objects [@!RFC7517]. Candidate Keys to add
  to the attested key set. Every JWK MUST contain a `kid`. If present, the
  array MUST NOT be empty.
- `retain_keys`: OPTIONAL. Array of strings. Key IDs that the Subordinate asks
  the Superior to keep in the attested key set. If omitted, the Superior SHOULD
  retain all currently attested keys that are not listed in `remove_keys`.
- `remove_keys`: OPTIONAL. Array of strings. Key IDs that the Subordinate asks
  the Superior to remove from the attested key set.
- `entity_configuration_uri`: OPTIONAL. String. URL of the Subordinate Entity
  Configuration. If omitted, the Superior MUST derive
  `{sub}/.well-known/openid-federation`.
- `key_proofs`: CONDITIONAL. Object mapping Candidate Key `kid` values to
  compact JWTs. REQUIRED when `add_keys` is present. Each value MUST be a Key
  Attestation Proof JWT for the corresponding Candidate Key, as defined in
  (#key-attestation-proof). Keys listed only in `remove_keys` or
  `retain_keys` do not require proofs.

At least one of `add_keys` or `remove_keys` MUST be present.

The following is a non-normative example of Key Attestation Request Claims.
The `key_proofs` values are shown abbreviated; on the wire each value is a
complete compact JWS.

```json
{
  "iss": "https://subordinate.example.com",
  "sub": "https://subordinate.example.com",
  "aud": "https://ta.example.com",
  "iat": 1792600000,
  "exp": 1792603600,
  "jti": "kar-20261022-01",
  "add_keys": [
    {
      "kty": "EC",
      "crv": "P-256",
      "use": "sig",
      "alg": "ES256",
      "kid": "subordinate-20260921T124313Z",
      "x": "nXW6y02ia5cgtgiSl7ywnOGLpocH8nnj9plc3VwbKSs",
      "y": "TxoiRaUH9MbQ8ce0y5ddMCuWwYVUMVj70T-lbGhyL8M"
    }
  ],
  "retain_keys": [
    "subordinate-20260320T162914Z"
  ],
  "remove_keys": [],
  "key_proofs": {
    "subordinate-20260921T124313Z":
      "eyJhbGciOiJFUzI1NiIsImtpZCI6InN1Ym9yZGluYXRlLTIwMjYwOTIxVDEyNDMxM1oiLCJ0eXAiOiJrZXktYXR0ZXN0YXRpb24tcHJvb2Yrand0In0.eyJpc3MiOiJodHRwczovL3N1Ym9yZGluYXRlLmV4YW1wbGUuY29tIiwiYXVkIjoiaHR0cHM6Ly90YS5leGFtcGxlLmNvbSIsImlhdCI6MTc5MjYwMDAwMCwiZXhwIjoxNzkyNjAzNjAwLCJqdGkiOiJrYXAtMDEiLCJhdHRlc3RhdGlvbl9qdGkiOiJrYXItMjAyNjEwMjItMDEifQ.signature"
  },
  "entity_configuration_uri":
    "https://subordinate.example.com/.well-known/openid-federation"
}
```

### Key Attestation Proof

Publishing a public JWK in an Entity Configuration alone does not prove
possession of the corresponding private key. When `add_keys` is present, the
Subordinate MUST include a proof of possession for every Candidate Key in the
`key_proofs` Claim.

A Key Attestation Proof is a nested JWS [@!RFC7515] signed with the Candidate
Key's private key. Binding to the parent Key Attestation Request is achieved
by including that request's `jti` in the proof; this profile does not use a
server-issued nonce or challenge.

#### Proof JWT Header

The Key Attestation Proof JOSE Header MUST include:

- `alg`: REQUIRED. A signature algorithm suitable for the Candidate Key.
  `ES256` is RECOMMENDED.
- `kid`: REQUIRED. MUST equal the Candidate Key `kid` that this proof covers.
- `typ`: REQUIRED. MUST be `key-attestation-proof+jwt`.

#### Proof JWT Claims

The Key Attestation Proof JWT Claims are:

- `iss`: REQUIRED. String. Entity Identifier of the Subordinate. MUST equal the
  parent request `iss`.
- `aud`: REQUIRED. String. Entity Identifier of the Immediate Superior. MUST
  equal the parent request `aud`.
- `iat`: REQUIRED. Number. Time at which the proof was issued.
- `exp`: REQUIRED. Number. Expiration time of the proof.
- `jti`: REQUIRED. String. Unique identifier for this proof.
- `attestation_jti`: REQUIRED. String. MUST equal the parent Key Attestation
  Request `jti`.

The following is a non-normative example of Key Attestation Proof Claims for
the Candidate Key in the request example above:

```json
{
  "iss": "https://subordinate.example.com",
  "aud": "https://ta.example.com",
  "iat": 1792600000,
  "exp": 1792603600,
  "jti": "kap-01",
  "attestation_jti": "kar-20261022-01"
}
```

#### Proof Verification

For each JWK in `add_keys`, the Superior MUST:

1. Locate the proof JWT in `key_proofs` under that JWK's `kid`. Absence of a
   proof is an error.
2. Verify that the proof JOSE Header `typ` is `key-attestation-proof+jwt` and
   that Header `kid`, when present, equals the Candidate Key `kid`.
3. Verify that proof Claims `iss` and `aud` match the parent request, and that
   `attestation_jti` equals the parent request `jti`.
4. Verify proof `iat` and `exp` for freshness, applying the same clock-skew
   tolerance used for the parent request.
5. Verify the proof signature using the Candidate Key public parameters from
   `add_keys` (not the Attested Key that signed the parent request).

Remove-only requests (`remove_keys` without `add_keys`) do not require
`key_proofs`.

### Superior Processing Rules

Upon receiving a Key Attestation Request, the Superior MUST:

1. Verify that the HTTP `Content-Type` is
   `application/key-attestation-request+jwt`.
2. Parse the JWT and verify that `typ` is `key-attestation-request+jwt`.
3. Verify that `aud` identifies this Superior.
4. Verify that `iss` equals `sub` and that the subject is an Immediate
   Subordinate of this Superior.
5. Obtain the current Subordinate Statement about `sub` and verify the request
   signature using the key identified by the request header `kid`. That key
   MUST be present in the Subordinate Statement `jwks`.
6. Verify `iat`, `exp`, and that `jti` has not been reused. Superiors SHOULD
   allow a small clock skew (for example 300 seconds) when validating `iat`.
7. Fetch the Subordinate Entity Configuration from
   `entity_configuration_uri` or the default well-known URL. Verify that Entity
   Configuration with an Attested Key.
8. For each JWK in `add_keys`, verify that a matching key with the same `kid`
   is present in the Entity Configuration `jwks`. Implementations MUST compare
   at least the public key parameters required by the key type (`crv`, `x`,
   and `y` for EC; `crv` and `x` for OKP; `n` and `e` for RSA); comparing only
   `kid` strings is insufficient.
9. When `add_keys` is present, verify proof of possession for each Candidate
   Key as specified in (#proof-verification).
10. Construct the updated attested key set by applying `add_keys`,
    `retain_keys`, and `remove_keys` according to local policy. The resulting
    set MUST contain at least one key.
11. Issue a new Subordinate Statement about `sub` whose `jwks` Claim contains
    the updated attested key set, and publish it at the Superior's
    `federation_fetch_endpoint`.

After a baseline attested key set exists for a Subordinate, the Superior's
`federation_fetch_endpoint` response for that Subordinate MUST reflect the
attested set, not a live mirror of the Subordinate Entity Configuration
`jwks`. Establishing the initial baseline MAY occur at enrollment or on first
successful attestation.

Superiors MAY require manual approval. In that case they return `202 Accepted`
as described below and MUST complete processing only after approval.

## Key Attestation Response

### Success Response

On immediate success, the Superior MUST respond with HTTP status code `200` and
`Content-Type` of `application/entity-statement+jwt`. The body MUST be the
newly issued Subordinate Statement about the requesting Subordinate.

On acceptance for asynchronous processing, the Superior MAY respond with HTTP
status code `202` and `Content-Type` of `application/json`. The JSON object
SHOULD include:

- `status`: String. Value `accepted`.
- `sub`: String. Entity Identifier of the Subordinate.
- `request_id`: OPTIONAL. String. Opaque identifier for this queued request.
- `status_uri`: OPTIONAL. String. URL where the requester can obtain processing
  status (for example by supplying `request_id` as a query parameter).

When `status_uri` is provided, a successful poll response SHOULD include
`request_id`, `sub`, and `status` with one of the values `pending`,
`approved`, or `denied`. A `denied` response MAY include `reason`.

Clients that receive `202` MUST later obtain the updated Subordinate Statement
from the Superior's `federation_fetch_endpoint` after the request is approved.

### Error Response

Error responses MUST use `Content-Type` of `application/json` and SHOULD use
the error response pattern defined by OpenID Federation for federation
endpoints.

Defined error codes include:

- `invalid_request`: Malformed request or missing required Claims.
- `invalid_client` / `unknown_subordinate`: Subject is not an Immediate
  Subordinate of this Superior.
- `unattested_signer`: Request signature key is not in the current attested
  key set.
- `key_not_published`: A Candidate Key is absent from the Subordinate Entity
  Configuration.
- `invalid_key`: A Candidate Key is malformed, does not match the published
  key parameters, or its proof of possession is missing or invalid.
- `approval_required`: Policy requires human approval and the Superior does not
  use `202` for that case.
- `server_error`: Unexpected failure.


# Examples

## Example HTTP Request

The following is a non-normative `curl` example. The JWT must be signed with the
already-attested key
`subordinate-20260320T162914Z` before posting, and MUST include a
`key_proofs` entry for `NEW_KID` signed with that Candidate Key.

```bash
#!/usr/bin/env bash
SUPERIOR="https://ta.example.com"
SUBORDINATE="https://subordinate.example.com"
NEW_KID="subordinate-20260921T124313Z"
OLD_KID="subordinate-20260320T162914Z"

# REQUEST_JWT is a compact JWS with header:
# {"alg":"ES256","kid":"<OLD_KID>","typ":"key-attestation-request+jwt"}
# claims as defined in this specification, including key_proofs[NEW_KID]
# (typ key-attestation-proof+jwt, attestation_jti = request jti).

curl -sS -X POST "${SUPERIOR}/key_rollover" \
  -H "Content-Type: application/key-attestation-request+jwt" \
  -H "Accept: application/entity-statement+jwt, application/json" \
  --data-binary "${REQUEST_JWT}"
```

## Example Success Response Header

```http
HTTP/1.1 200 OK
Content-Type: application/entity-statement+jwt
```

The response body is a Subordinate Statement whose `iss` is the Superior, whose
`sub` is the Subordinate, and whose `jwks` contains both the retained Attested
Key and the newly added Candidate Key.


# Implementation Considerations

## Overlap Period

During rollover, Superiors SHOULD keep previously attested keys in the
Subordinate Statement until Subordinates and relying parties have had time to
obtain the updated statement. Subordinates SHOULD continue signing Entity
Configurations with an Attested Key until step 4 of
(#rollover-sequence) has completed.

When a Subordinate has more than one Immediate Superior, it SHOULD keep signing
with the previously Attested Key until every Immediate Superior has accepted
the Key Attestation Request. Only then SHOULD the Subordinate activate the
newly attested signing key. If any Superior denies the request or acceptance
times out, the previously Attested Key remains the active signer.

## Policy and Automatic Approval

Federations MAY auto-approve Key Attestation Requests that pass the normative
checks in this specification. Federations with stricter operational controls
MAY require out-of-band approval while still using this endpoint as the
machine-readable request and response channel.

## Interaction with Enrollment

Initial onboarding MAY continue to use federation-specific registration or
enrollment protocols. This endpoint is intended for subsequent key set updates
for an already established Superior/Subordinate relationship.

## Relation to Other Proof Mechanisms

The Key Attestation Proof profile defined here is specific to Federation Entity
Key updates. It is not hardware key attestation, not OAuth DPoP, and not the
OpenID for Verifiable Credential Issuance proof-of-possession mechanism that
uses `c_nonce` or confirmation claims such as `cnf`.


# Security Considerations

## Signer Must Be Attested

Accepting a Key Attestation Request signed with a key that is only present in
the Subordinate Entity Configuration, but not in the Superior's Subordinate
Statement, would allow an attacker who can publish a replacement Entity
Configuration to expand the attested key set. Superiors MUST reject such
requests.

## Candidate Keys Must Be Published and Matched

Superiors MUST fetch the Subordinate Entity Configuration and match Candidate
Keys to published JWKs by public key material, not only by `kid`. Relying on
`kid` alone enables key substitution if an attacker can choose colliding key
identifiers.

## Proof of Possession

Without proof of possession, a Subordinate (or an attacker controlling its
Entity Configuration publication) could request attestation of an arbitrary
public key that the Subordinate does not control. Superiors MUST therefore
require Key Attestation Proofs for every Candidate Key in `add_keys`, as
specified in (#key-attestation-proof).

Proofs are bound to a single parent request through `attestation_jti`. A
captured proof cannot authorize a different Key Attestation Request. Superiors
MUST still enforce parent-request `jti` uniqueness so that a captured request
and its proofs cannot be replayed.

## Replay

The combination of `jti`, `iat`, and `exp` is intended to limit replay. Superiors
MUST enforce freshness and `jti` uniqueness for the lifetime of the request.

## Authorization Boundary

This endpoint updates attestation for Immediate Subordinates only. It MUST NOT
be used to modify Trust Anchor keys distributed out of band, nor to attest keys
for entities that are not Immediate Subordinates of the responding Superior.


# Privacy Considerations

Key Attestation Requests reveal that a Subordinate is rotating Federation
Entity Keys and which Superior attests them. Deployments SHOULD protect the
endpoint with normal TLS practices and SHOULD avoid logging full JWTs
indefinitely.


{backmatter}

# Notices


Copyright (c) 2026 The OpenID Foundation.

The OpenID Foundation invites you to participate in this draft. Feedback and
discussion are encouraged on the OpenID Connect Working Group mailing list.


# Acknowledgements

This draft was motivated by operational experience rolling Federation Entity
Keys for Intermediate Trust Anchors whose keys are attested by a Superior Trust
Anchor. In particular, publishing new keys in an Entity Configuration while a
Superior Subordinate Statement continues to attest only the previous key leaves
Trust Chains unable to validate signatures made with the new key.


# Document History

-01

* Defined normative proof of possession: `key_proofs`, JOSE typ
  `key-attestation-proof+jwt`, and `attestation_jti` binding to the parent
  request.
* Clarified public-parameter matching by key type, attested-set locking at
  `federation_fetch_endpoint`, multi-superior rollover behavior, `202`
  `request_id` / status polling, and distinction from DPoP and OpenID4VCI
  proofs.

-00

* Initial individual draft defining the Federation Key Attestation Endpoint,
  request JWT profile, processing rules, and examples.
