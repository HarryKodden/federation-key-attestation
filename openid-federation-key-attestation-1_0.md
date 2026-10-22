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

This specification acts as an extension to [@!OpenID.Federation]. It defines
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

[@!OpenID.Federation] Section 11 describes Federation Key Rollover for Entity
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
and "Trust Chain" as defined in [@!OpenID.Federation].

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


# Relationship to OpenID Federation Key Rollover

[@!OpenID.Federation] Section 11.2 describes Trust Anchor key rollover by
publishing new keys in the Trust Anchor Entity Configuration while continuing
to sign with previously distributed keys. That model works for Trust Anchors
because relying parties obtain Trust Anchor keys out of band or from the Trust
Anchor Entity Configuration itself.

For Intermediates and Leaf Entities, consumers verify Entity Configuration
signatures using keys from the Immediate Superior's Subordinate Statement. A
new key published only in the Subordinate Entity Configuration is therefore not
sufficient for Trust Chain validation until the Superior attests it.

The recommended rollover sequence when using this specification is:

1. The Subordinate adds the Candidate Key to its Entity Configuration `jwks`
   Claim.
2. The Subordinate continues to sign its Entity Configuration with an Attested
   Key.
3. The Subordinate calls the Superior's Federation Key Attestation Endpoint.
4. The Superior issues an updated Subordinate Statement that includes the
   Candidate Key in `jwks`.
5. The Subordinate begins signing with the newly attested key and MAY later
   retire previous keys via `federation_historical_keys_endpoint`.


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
  "iss": "https://ta.oidf-pilot.edugain.org",
  "sub": "https://ta.oidf-pilot.edugain.org",
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
        "https://ta.oidf-pilot.edugain.org/fetch",
      "federation_list_endpoint":
        "https://ta.oidf-pilot.edugain.org/list",
      "federation_resolve_endpoint":
        "https://ta.oidf-pilot.edugain.org/resolve",
      "federation_key_attestation_endpoint":
        "https://ta.oidf-pilot.edugain.org/key_rollover"
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

At least one of `add_keys` or `remove_keys` MUST be present.

The following is a non-normative example of Key Attestation Request Claims:

```json
{
  "iss": "https://ta.poc2.dev.oidf.lab.surfconext.nl",
  "sub": "https://ta.poc2.dev.oidf.lab.surfconext.nl",
  "aud": "https://ta.oidf-pilot.edugain.org",
  "iat": 1792600000,
  "exp": 1792603600,
  "jti": "kar-20261022-01",
  "add_keys": [
    {
      "kty": "EC",
      "crv": "P-256",
      "use": "sig",
      "alg": "ES256",
      "kid": "ta-poc2-dev-oidf-lab-surfconext-nl-20260921T124313Z",
      "x": "nXW6y02ia5cgtgiSl7ywnOGLpocH8nnj9plc3VwbKSs",
      "y": "TxoiRaUH9MbQ8ce0y5ddMCuWwYVUMVj70T-lbGhyL8M"
    }
  ],
  "retain_keys": [
    "ta-poc2-dev-oidf-lab-surfconext-nl-20260320T162914Z"
  ],
  "remove_keys": [],
  "entity_configuration_uri":
    "https://ta.poc2.dev.oidf.lab.surfconext.nl/.well-known/openid-federation"
}
```

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
6. Verify `iat`, `exp`, and that `jti` has not been reused.
7. Fetch the Subordinate Entity Configuration from
   `entity_configuration_uri` or the default well-known URL. Verify that Entity
   Configuration with an Attested Key.
8. For each JWK in `add_keys`, verify that a matching key with the same `kid`
   is present in the Entity Configuration `jwks`. Implementations MUST compare
   at least the public key parameters required by the key type; comparing only
   `kid` strings is insufficient.
9. Construct the updated attested key set by applying `add_keys`,
   `retain_keys`, and `remove_keys` according to local policy. The resulting
   set MUST contain at least one key.
10. Issue a new Subordinate Statement about `sub` whose `jwks` Claim contains
    the updated attested key set, and publish it at the Superior's
    `federation_fetch_endpoint`.

It is RECOMMENDED that the Superior also require proof of possession of each
Candidate Key, for example a nested JWT signed with that key's corresponding
private key, or an equivalent challenge-response. Publishing a public JWK in an
Entity Configuration alone does not prove possession of the private key.

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
- `status_uri`: OPTIONAL. String. URL where the requester can obtain processing
  status.

Clients that receive `202` MUST later obtain the updated Subordinate Statement
from the Superior's `federation_fetch_endpoint`.

### Error Response

Error responses MUST use `Content-Type` of `application/json` and SHOULD use
the error response pattern defined by [@!OpenID.Federation] for federation
endpoints.

Defined error codes include:

- `invalid_request`: Malformed request or missing required Claims.
- `invalid_client` / `unknown_subordinate`: Subject is not an Immediate
  Subordinate of this Superior.
- `unattested_signer`: Request signature key is not in the current attested
  key set.
- `key_not_published`: A Candidate Key is absent from the Subordinate Entity
  Configuration.
- `invalid_key`: A Candidate Key is malformed or does not match the published
  key parameters.
- `approval_required`: Policy requires human approval and the Superior does not
  use `202` for that case.
- `server_error`: Unexpected failure.


# Examples

## Example HTTP Request

The following is a non-normative `curl` example. The JWT must be signed with the
already-attested key
`ta-poc2-dev-oidf-lab-surfconext-nl-20260320T162914Z` before posting.

```bash
#!/usr/bin/env bash
SUPERIOR="https://ta.oidf-pilot.edugain.org"
SUBORDINATE="https://ta.poc2.dev.oidf.lab.surfconext.nl"
NEW_KID="ta-poc2-dev-oidf-lab-surfconext-nl-20260921T124313Z"
OLD_KID="ta-poc2-dev-oidf-lab-surfconext-nl-20260320T162914Z"

# REQUEST_JWT is a compact JWS with header:
# {"alg":"ES256","kid":"<OLD_KID>","typ":"key-attestation-request+jwt"}
# and claims as defined in this specification.

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
Configurations with an Attested Key until step 4 of the rollover sequence has
completed.

## Policy and Automatic Approval

Federations MAY auto-approve Key Attestation Requests that pass the normative
checks in this specification. Federations with stricter operational controls
MAY require out-of-band approval while still using this endpoint as the
machine-readable request and response channel.

## Interaction with Enrollment

Initial onboarding MAY continue to use federation-specific registration or
enrollment protocols. This endpoint is intended for subsequent key set updates
for an already established Superior/Subordinate relationship.


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
public key. Proof of possession of Candidate Keys is therefore RECOMMENDED.

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

<reference anchor="OpenID.Federation" target="https://openid.net/specs/openid-federation-1_1-final.html">
  <front>
    <title>OpenID Federation 1.1</title>
    <author fullname="R. Hedberg, Ed.">
      <organization>independent</organization>
    </author>
    <author fullname="Michael B. Jones">
      <organization>Self-Issued Consulting</organization>
    </author>
    <author fullname="A. Solberg">
      <organization>Sikt</organization>
    </author>
    <author fullname="John Bradley">
      <organization>Yubico</organization>
    </author>
    <author fullname="Giuseppe De Marco">
      <organization>independent</organization>
    </author>
    <author fullname="Vladimir Dzhuvinov">
      <organization>Connect2id</organization>
    </author>
    <date day="28" month="April" year="2026"/>
  </front>
</reference>

<reference anchor="OpenID.Federation.SubordinateEvents" target="https://openid.net/specs/openid-federation-subordinate-events-1_0.html">
  <front>
    <title>OpenID Federation Subordinate Events Endpoint 1.0</title>
    <author fullname="OpenID Foundation">
      <organization>OpenID Foundation</organization>
    </author>
    <date day="1" month="January" year="2025"/>
  </front>
</reference>

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

-00

* Initial individual draft defining the Federation Key Attestation Endpoint,
  request JWT profile, processing rules, and examples.
