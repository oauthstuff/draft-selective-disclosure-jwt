%%%
title = "Selective Disclosure JWT (SD-JWT)"
abbrev = "SD-JWT"
ipr = "trust200902"
area = "Security"
workgroup = "Web Authorization Protocol"
keyword = ["security", "oauth2"]

[seriesInfo]
name = "Internet-Draft"
value = "draft-fett-oauth-selective-disclosure-jwt-01"
stream = "IETF"
status = "standard"

[[author]]
initials="D."
surname="Fett"
fullname="Daniel Fett"
organization="yes.com"
    [author.address]
    email = "mail@danielfett.de"
    uri = "https://danielfett.de/"


[[author]]
initials="K."
surname="Yasuda"
fullname="Kristina Yasuda"
organization="Microsoft"
    [author.address]
    email = "Kristina.Yasuda@microsoft.com"
        
    
%%%

.# Abstract 

This document specifies conventions for creating JSON Web Token (JWT)
documents that support selective disclosure of JWT claim values. 

{mainmatter}

# Introduction {#Introduction}

The JSON-based claims in a signed JSON Web Token (JWT) [@!RFC7519] document are
secured against modification using JSON Web Signature (JWS) [@!RFC7515] digital
signatures. A consumer of a signed JWT document that has checked the document's
signature can safely assume that the contents of the document have not been
modified.  However, anyone receiving an unencrypted JWT can read all of the
claims and likewise, anyone with the decryption key receiving an encrypted JWT
can also read all of the claims.

This document describes a format for signed JWTs that supports selective
disclosure (SD-JWT), enabling sharing only a subset of the claims included in
the original signed JWT instead of releasing all the claims to every verifier.
During issuance, an SD-JWT is sent from the issuer to the holder alongside an
SD-JWT Salt/Value Container (SVC), a JSON object that contains the mapping
between raw claim values contained in the SD-JWT and the salts for each claim
value. 

This document also defines a format for SD-JWT Releases (SD-JWT-R), which convey
a subset of the claim values of an SD-JWT to the verifier. For presentation, the
holder creates an SD-JWT-R and sends it together with the SD-JWT to the
verifier. To verify claim values received in SD-JWT-R, the verifier uses the
salts values in the SD-JWT-R to compute the hash digests of the claim values and
compare them to the ones in the SD-JWT.

One of the common use cases of a signed JWT is representing a user's identity
created by an issuer. As long as the signed JWT is one-time use, it typically
only contains those claims the user has consented to release to a specific
verifier. However, when a signed JWT is intended to be multi-use, it needs to
contain the superset of all claims the user might want to release to verifiers
at some point. The ability to selectively disclose a subset of these claims
depending on the verifier becomes crucial to ensure minimum disclosure and
prevent verifiers from obtaining claims irrelevant for the transaction at hand.

One example of such a multi-use JWT is a verifiable credential, a
tamper-evident credential with a cryptographically verifiable authorship that
contains claims about a subject. SD-JWTs defined in this document enable such
selective disclosure of claims. 

While JWTs for claims describing natural persons are a common use case, the
mechanisms defined in this document can be used for many other use cases as
well.

This document also describes holder binding, or the concept of binding SD-JWT to
key material controlled by the subject of SD-JWT. Holder binding is optional to
implement.


## Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 [@!RFC2119] [@!RFC8174] when, and only when, they
appear in all capitals, as shown here.

**base64url** denotes the URL-safe base64 encoding without padding defined in
Section 2 of [@!RFC7515].

# Terms and Definitions

Selective Disclosure JWT (SD-JWT) 
:  A JWT [@!RFC7515] created by the issuer, which is signed as a JWS [@!RFC7515], 
   that supports selective disclosure as defined in this document.

SD-JWT Salt/Value Container (SVC) 
:  A JSON object created by the issuer that contains mapping between 
   raw claim values contained in the SD-JWT and the salts for each claim value.

SD-JWT Release (SD-JWT-R) 
:  A JWT created by the holder that contains a subset of the claim values of an SD-JWT in a verifiable way. 

Holder binding 
:  Ability of the holder to prove legitimate possession of SD-JWT by proving 
   control over the same private key during the issuance and presentation. SD-JWT signed by the issuer contains
   a public key or a reference to a public key that matches to the private key controlled by the holder.

Issuer 
:  An entity that creates SD-JWTs (2.1).

Holder 
:  An entity that received SD-JWTs (2.1) from the issuer and has control over them.

Verifier 
:  An entity that requests, checks and extracts the claims from SD-JWT-R (2.2)

Note: discuss if we want to include Client, Authorization Server for the purpose of
ensuring continuity and separating the entity from the actor.

# Flow Diagram

~~~ ascii-art
           +------------+
           |            |
           |   Issuer   |
           |            |
           +------------+
                 |
               Issues
           SD-JWT and SVC
                 |
                 v
           +------------+
           |            |
           |   Holder   |
           |            |
           +------------+
                 |
              Presents
         SD-JWT-R and SD-JWT
                 |
                 v
           +-------------+
           |             |+
           |  Verifiers  ||+
           |             |||
           +-------------+||
            +-------------+|
             +-------------+
~~~
Figure: SD-JWT Issuance and Presentation Flow

# Concepts

In the following, the contents of SD-JWTs and SD-JWT Releases are described at a
conceptual level, abstracting from the data formats described afterwards.

## Creating an SD-JWT

An SD-JWT, at its core, is a digitally signed document containing hash digests over the claim values with unique random salts and other metadata. 
It MUST be digitally signed using the issuer's private key.

```
SD-JWT-DOC = (METADATA, SD-CLAIMS)
SD-JWT = SD-JWT-DOC | SIG(SD-JWT-DOC, ISSUER-PRIV-KEY)
```

`SD-CLAIMS` can be a simple object with claim names mapped to hash digests over the claim values with unique random salts:
```
SD-CLAIMS = (
    CLAIM-NAME: HASH(SALT | CLAIM-VALUE)
)*
```

`SD-CLAIMS` can also be nested deeper to capture more complex objects, as will be shown later.

`SD-JWT` is sent from the issuer to the holder, together with the mapping of the plain-text claim values, the salt values, and potentially some other information. 

## Creating an SD-JWT Release

To disclose to a verifier a subset of the SD-JWT claim values, a holder creates a JWT such as the
following:

```
SD-JWT-RELEASE-DOC = (METADATA, SD-RELEASES)
SD-JWT-RELEASE = SD-JWT-RELEASE-DOC
```


`SD-RELEASES` follows the structure of `SD-CLAIMS` and can be a simple object with claim names mapped to values and salts:

```
SD-RELEASES = (
    CLAIM-NAME: (DISCLOSED-SALT, DISCLOSED-VALUE)
)
```

Just as `SD-CLAIMS`, `SD-RELEASES` can be more complex as well.

`SD-JWT-RELEASE` is sent together with `SD-JWT` from the holder to the
verifier.

## Optional Holder Binding

Some use-cases may require holder binding. 

If holder binding is desired, `SD-JWT` must contain information about key material controlled by the holder:

```
SD-JWT-DOC = (METADATA, HOLDER-PUBLIC-KEY, SD-CLAIMS)
```

Note: How the public key is included in SD-JWT is out of scope of this document. It can be passed by value or by reference. Examples in this document use `sub_jwk` Claim to include raw public key by value in SD-JWT.

With holder binding, the `SD-JWT-RELEASE` is signed by the holder using its private key. It therefore looks as follows:

```
SD-JWT-RELEASE = SD-JWT-RELEASE-DOC | SIG(SD-JWT-RELEASE-DOC, HOLDER-PRIV-KEY)
```


## Verifying an SD-JWT Release

A verifier checks that 

 * for each claim in `SD-JWT-RELEASE`, the hash digest `HASH(DISCLOSED-SALT | DISCLOSED-VALUE)` 
 matches the one under the given claim name in `SD-JWT`.
 * if holder binding is used, the `SD-JWT-RELEASE` was signed by
 the private key belonging to `HOLDER-PUBLIC-KEY`.

The detailed algorithm is described below.

# Data Formats

This section defines data formats for SD-JWTs (containing hash digests of the salted
claim values), SD-JWT Salt/Value Containers (containing the mapping of the
plain-text claim values and the salt values), and SD-JWT Releases (containing a
subset of the same mapping).

## Format of an SD-JWT

An SD-JWT is a JWT that MUST be signed using the issuer's private key. The
payload of an SD-JWT MUST contain the `sd_digests` and `hash_alg` claims
described in the following, and MAY contain a holder's public key or a reference
thereto, as well as further claims such as `iss`, `iat`, etc. as defined or
required by the application using SD-JWTs.

A `typ` parameter in the JOSE header of an SD-JWT MUST be set to the value `sd+jwt`.
### `sd_digests` Claim (Digests of Selectively Disclosable Claims)

An SD-JWT MUST include hash digests of the salted claim values that are included by the issuer
under the property `sd_digests`. 

The issuer MUST choose a unique and cryptographically random salt value
for each claim value. Each salt value
SHOULD contain at least 128 bits of pseudorandom data, making it hard for an
attacker to guess. The salt value MUST then be encoded as a string. It is
RECOMMENDED to base64url-encode the salt value.

The issuer MUST build the digests by hashing over a string that is formed by
JSON-encoding an ordered array containing the salt and the claim value, e.g.:
`["6qMQvRL5haj","Peter"]`. The digest value is then base64url-encoded. Note that
the precise JSON encoding can vary, and therefore, the JSON encodings MUST be
sent to the holder along with the SD-JWT, as described below. 


#### Flat and Structured `sd_digests` objects

The `sd_digests` object can be a 'flat' object, directly containing all claim
names and hashed claim values without any deeper structure. The `sd_digests`
object can also be a 'structured' object, where some claims and their respective
hash digests are contained in places deeper in the structure. It is at the issuer's
discretion whether to use a 'flat' or 'structured' `sd_digests` SD-JWT object,
and how to structure it such that it is suitable for the use case. 

Example 1 below is a non-normative example of an SD-JWT using a 'flat'
`sd_digests` object and Example 2 in the appendix shows a non-normative example
of an SD-JWT using a 'structured' `sd_digests` object. The difference between
the examples is how the `address` claim is disclosed.

Appendix 2 shows a more complex example using claims from eKYC (todo:
reference).

### Hash Function Claim

The claim `hash_alg` indicates the hash algorithm used by the Issuer to generate
the hashes of the salted claim values. The hash algorithm identifier MUST be a
value from the "Hash Name String" column in the IANA "Named Information Hash
Algorithm" registry [IANA.Hash.Algorithms]. SD-JWTs with hash algorithm
identifiers not found in this registry are not considered valid and MUST NOT be
accepted by verifiers.

### Holder Public Key Claim

If the issuer wants to enable holder binding, it MAY include a public key
associated with the holder, or a reference thereto. 

It is out of the scope of this document to describe how the holder key pair is
established. For example, the holder MAY provide a key pair to the issuer, 
the issuer MAY create the key pair for the holder, or
holder and issuer MAY use pre-established key material.

Note: need to define how holder public key is included, right now examples are using `sub_jwk` I think.

## Example 1: SD-JWT

This example and Example 2 in the appendix use the following object as the set
of claims that the Issuer is issuing:

{#example-simple-sd-jwt-claims}
```json
{
  "sub": "6c5c0a49-b589-431d-bae7-219122a9ec2c",
  "given_name": "John",
  "family_name": "Doe",
  "email": "johndoe@example.com",
  "phone_number": "+1-202-555-0101",
  "address": {
    "street_address": "123 Main St",
    "locality": "Anytown",
    "region": "Anystate",
    "country": "US"
  },
  "birthdate": "1940-01-01"
}
```

The following non-normative example shows the payload of an SD-JWT. The issuer
is using a flat structure, i.e., all of the claims the `address` claim can only
be disclosed in full.

{#example-simple-sd-jwt-payload}
```json
{
  "iss": "https://example.com/issuer",
  "sub_jwk": {
    "kty": "RSA",
    "n": "pm4bOHBg-oYhAyPWzR56AWX3rUIXp11_ICDkGgS6W3ZWLts-hzwI3x65659kg4hVo9dbGoCJE3ZGF_eaetE30UhBUEgpGwrDrQiJ9zqprmcFfr3qvvkGjtth8Zgl1eM2bJcOwE7PCBHWTKWYs152R7g6Jg2OVph-a8rq-q79MhKG5QoW_mTz10QT_6H4c7PjWG1fjh8hpWNnbP_pv6d1zSwZfc5fl6yVRL0DV0V3lGHKe2Wqf_eNGjBrBLVklDTk8-stX_MWLcR-EGmXAOv0UBWitS_dXJKJu-vXJyw14nHSGuxTIK2hx1pttMft9CsvqimXKeDTU14qQL1eE7ihcw",
    "e": "AQAB"
  },
  "iat": 1516239022,
  "exp": 1516247022,
  "hash_alg": "sha-256",
  "sd_digests": {
    "sub": "z4xgEco94diTaSruISPiE7o_wtmcOfnH_8R7X9Pa578",
    "given_name": "PvU7cWjuHUq6w-i9XFpQZhjT-uprQL3GH3mKsAJl0e0",
    "family_name": "H-Relr4cEBMlenyK1gvyx16QVpnt4MEclT5tP0aTLFU",
    "email": "ET2A1JQLF85ZpBulh6UFstGrSfR4B3KM-bjQVllhxqY",
    "phone_number": "SJnciB2DIRVA5cXBrdKoH6n45788mZyUn2rnv74uMVU",
    "address": "0FldqLfGnERPPVDC17od9xb4w3iRJTEQbW_Yk9AmnDw",
    "birthdate": "-L0kMgIbLXe3OEkKTUGwz_QKhjehDeofKGwoPrxLuo4"
  }
}
```

The SD-JWT is then signed by the issuer to create a document like the following:

{#example-simple-sd-jwt-encoded}
```
eyJhbGciOiAiUlMyNTYiLCAia2lkIjogImNBRUlVcUowY21MekQxa3pHemhlaUJhZzBZUk
F6VmRsZnhOMjgwTmdIYUEifQ.eyJpc3MiOiAiaHR0cHM6Ly9leGFtcGxlLmNvbS9pc3N1Z
XIiLCAic3ViX2p3ayI6IHsia3R5IjogIlJTQSIsICJuIjogInBtNGJPSEJnLW9ZaEF5UFd
6UjU2QVdYM3JVSVhwMTFfSUNEa0dnUzZXM1pXTHRzLWh6d0kzeDY1NjU5a2c0aFZvOWRiR
29DSkUzWkdGX2VhZXRFMzBVaEJVRWdwR3dyRHJRaUo5enFwcm1jRmZyM3F2dmtHanR0aDh
aZ2wxZU0yYkpjT3dFN1BDQkhXVEtXWXMxNTJSN2c2SmcyT1ZwaC1hOHJxLXE3OU1oS0c1U
W9XX21UejEwUVRfNkg0YzdQaldHMWZqaDhocFdObmJQX3B2NmQxelN3WmZjNWZsNnlWUkw
wRFYwVjNsR0hLZTJXcWZfZU5HakJyQkxWa2xEVGs4LXN0WF9NV0xjUi1FR21YQU92MFVCV
2l0U19kWEpLSnUtdlhKeXcxNG5IU0d1eFRJSzJoeDFwdHRNZnQ5Q3N2cWltWEtlRFRVMTR
xUUwxZUU3aWhjdyIsICJlIjogIkFRQUIifSwgImlhdCI6IDE1MTYyMzkwMjIsICJleHAiO
iAxNTE2MjQ3MDIyLCAiaGFzaF9hbGciOiAic2hhLTI1NiIsICJzZF9kaWdlc3RzIjogeyJ
zdWIiOiAiejR4Z0Vjbzk0ZGlUYVNydUlTUGlFN29fd3RtY09mbkhfOFI3WDlQYTU3OCIsI
CJnaXZlbl9uYW1lIjogIlB2VTdjV2p1SFVxNnctaTlYRnBRWmhqVC11cHJRTDNHSDNtS3N
BSmwwZTAiLCAiZmFtaWx5X25hbWUiOiAiSC1SZWxyNGNFQk1sZW55SzFndnl4MTZRVnBud
DRNRWNsVDV0UDBhVExGVSIsICJlbWFpbCI6ICJFVDJBMUpRTEY4NVpwQnVsaDZVRnN0R3J
TZlI0QjNLTS1ialFWbGxoeHFZIiwgInBob25lX251bWJlciI6ICJTSm5jaUIyRElSVkE1Y
1hCcmRLb0g2bjQ1Nzg4bVp5VW4ycm52NzR1TVZVIiwgImFkZHJlc3MiOiAiMEZsZHFMZkd
uRVJQUFZEQzE3b2Q5eGI0dzNpUkpURVFiV19ZazlBbW5EdyIsICJiaXJ0aGRhdGUiOiAiL
Uwwa01nSWJMWGUzT0VrS1RVR3d6X1FLaGplaERlb2ZLR3dvUHJ4THVvNCJ9fQ.TSxRFVMl
Nlt_drWLqQuzz9Sc19Bo5r-cgXZ0to8PfadImEJuPbhfkxCSt1xCT5IWQJrD6p_47h8Ac-
5yOtuu8s-wxrQzCUAU7OS2ThQGKiDKsGv0oiH3KQX4QyCAOIIaWOLxO6VQRxI7Kn6RFT0M
5jQxBNFmL582n_uJR31ANBO45Jc_pgp4iSTn2vf1lxxwU7B3bgrad0vHmvwmv1lu17Ov79
SGmT_fTjoybIJfHavTq61xCJ3UrMAGCEqm-_laOTxB1HRbPF38va0iuhqSSk2QaUpfDsVb
9VsPBVIx_MsGlK3kYUWcHzkJ0BBPIVu0xnVcLkg1_SO_5gPSxFWZ4g
```

(Line breaks for presentation only.)


## Format of a SD-JWT Salt/Value Container (SVC)

Besides the SD-JWT itself, the holder needs to learn the raw claim values that
are contained in the SD-JWT, along with the precise input to the hash
calculation, and the salts. There MAY be other information the issuer needs to
communicate to the holder, such as a private key if the issuer selected the
holder key pair.

A SD-JWT Salt/Value Container (SVC) is a JSON object containing at least the
top-level property `sd_release`. Its structure mirrors the one of `sd_digests` in
the SD-JWT, but the values are the inputs to the hash calculations the issuer
used, as strings.

The SVC MAY contain further properties, for example, to transport the holder
private key.

## Example: SVC for the Flat SD-JWT in Example 1

The SVC for Example 1 is as follows:

{#example-simple-svc-payload}
```json
{
  "sd_release": {
    "sub": "[\"2GLC42sKQveCfGfryNRN9w\", \"6c5c0a49-b589-431d-bae7-219122a9ec2c\"]",
    "given_name": "[\"eluV5Og3gSNII8EYnsxA_A\", \"John\"]",
    "family_name": "[\"6Ij7tM-a5iVPGboS5tmvVA\", \"Doe\"]",
    "email": "[\"eI8ZWm9QnKPpNPeNenHdhQ\", \"johndoe@example.com\"]",
    "phone_number": "[\"Qg_O64zqAxe412a108iroA\", \"+1-202-555-0101\"]",
    "address": "[\"AJx-095VPrpTtN4QMOqROA\", {\"street_address\": \"123 Main St\", \"locality\": \"Anytown\", \"region\": \"Anystate\", \"country\": \"US\"}]",
    "birthdate": "[\"Pc33JM2LchcU_lHggv_ufQ\", \"1940-01-01\"]"
  }
}
```

Important: As described above, hash digests are calculated over the string formed by
serializing a JSON array containing the salt and the claim value. This ensures
that issuer and verifier use the same input to their hash functions and avoids
issues with canonicalization of JSON values that would lead to different hash
digests. The SVC therefore maps claim names to JSON-encoded arrays. 

## Sending SD-JWT and SVC during Issuance

For transporting the SVC together with the SD-JWT from the issuer to the holder,
the SVC is base64url-encoded and appended to the SD-JWT using a period character `.` as the
separator. 

The SVC and SD-JWT are implicitly linked through the hash values of the claims
in the SVC that is included in the SD-JWT. To ensure that the correct SVC and 
SD-JWT pairings are being used, the holder SHOULD verify the binding between
SVC and SD-JWT as defined in the Verification Section of this document.

For Example 1, the combined format looks as follows:

{#example-simple-combined-sd-jwt-svc}
```
eyJhbGciOiAiUlMyNTYiLCAia2lkIjogImNBRUlVcUowY21MekQxa3pHemhlaUJhZzBZUk
F6VmRsZnhOMjgwTmdIYUEifQ.eyJpc3MiOiAiaHR0cHM6Ly9leGFtcGxlLmNvbS9pc3N1Z
XIiLCAic3ViX2p3ayI6IHsia3R5IjogIlJTQSIsICJuIjogInBtNGJPSEJnLW9ZaEF5UFd
6UjU2QVdYM3JVSVhwMTFfSUNEa0dnUzZXM1pXTHRzLWh6d0kzeDY1NjU5a2c0aFZvOWRiR
29DSkUzWkdGX2VhZXRFMzBVaEJVRWdwR3dyRHJRaUo5enFwcm1jRmZyM3F2dmtHanR0aDh
aZ2wxZU0yYkpjT3dFN1BDQkhXVEtXWXMxNTJSN2c2SmcyT1ZwaC1hOHJxLXE3OU1oS0c1U
W9XX21UejEwUVRfNkg0YzdQaldHMWZqaDhocFdObmJQX3B2NmQxelN3WmZjNWZsNnlWUkw
wRFYwVjNsR0hLZTJXcWZfZU5HakJyQkxWa2xEVGs4LXN0WF9NV0xjUi1FR21YQU92MFVCV
2l0U19kWEpLSnUtdlhKeXcxNG5IU0d1eFRJSzJoeDFwdHRNZnQ5Q3N2cWltWEtlRFRVMTR
xUUwxZUU3aWhjdyIsICJlIjogIkFRQUIifSwgImlhdCI6IDE1MTYyMzkwMjIsICJleHAiO
iAxNTE2MjQ3MDIyLCAiaGFzaF9hbGciOiAic2hhLTI1NiIsICJzZF9kaWdlc3RzIjogeyJ
zdWIiOiAiejR4Z0Vjbzk0ZGlUYVNydUlTUGlFN29fd3RtY09mbkhfOFI3WDlQYTU3OCIsI
CJnaXZlbl9uYW1lIjogIlB2VTdjV2p1SFVxNnctaTlYRnBRWmhqVC11cHJRTDNHSDNtS3N
BSmwwZTAiLCAiZmFtaWx5X25hbWUiOiAiSC1SZWxyNGNFQk1sZW55SzFndnl4MTZRVnBud
DRNRWNsVDV0UDBhVExGVSIsICJlbWFpbCI6ICJFVDJBMUpRTEY4NVpwQnVsaDZVRnN0R3J
TZlI0QjNLTS1ialFWbGxoeHFZIiwgInBob25lX251bWJlciI6ICJTSm5jaUIyRElSVkE1Y
1hCcmRLb0g2bjQ1Nzg4bVp5VW4ycm52NzR1TVZVIiwgImFkZHJlc3MiOiAiMEZsZHFMZkd
uRVJQUFZEQzE3b2Q5eGI0dzNpUkpURVFiV19ZazlBbW5EdyIsICJiaXJ0aGRhdGUiOiAiL
Uwwa01nSWJMWGUzT0VrS1RVR3d6X1FLaGplaERlb2ZLR3dvUHJ4THVvNCJ9fQ.TSxRFVMl
Nlt_drWLqQuzz9Sc19Bo5r-cgXZ0to8PfadImEJuPbhfkxCSt1xCT5IWQJrD6p_47h8Ac-
5yOtuu8s-wxrQzCUAU7OS2ThQGKiDKsGv0oiH3KQX4QyCAOIIaWOLxO6VQRxI7Kn6RFT0M
5jQxBNFmL582n_uJR31ANBO45Jc_pgp4iSTn2vf1lxxwU7B3bgrad0vHmvwmv1lu17Ov79
SGmT_fTjoybIJfHavTq61xCJ3UrMAGCEqm-_laOTxB1HRbPF38va0iuhqSSk2QaUpfDsVb
9VsPBVIx_MsGlK3kYUWcHzkJ0BBPIVu0xnVcLkg1_SO_5gPSxFWZ4g.eyJzZF9yZWxlYXN
lIjogeyJzdWIiOiAiW1wiMkdMQzQyc0tRdmVDZkdmcnlOUk45d1wiLCBcIjZjNWMwYTQ5L
WI1ODktNDMxZC1iYWU3LTIxOTEyMmE5ZWMyY1wiXSIsICJnaXZlbl9uYW1lIjogIltcImV
sdVY1T2czZ1NOSUk4RVluc3hBX0FcIiwgXCJKb2huXCJdIiwgImZhbWlseV9uYW1lIjogI
ltcIjZJajd0TS1hNWlWUEdib1M1dG12VkFcIiwgXCJEb2VcIl0iLCAiZW1haWwiOiAiW1w
iZUk4WldtOVFuS1BwTlBlTmVuSGRoUVwiLCBcImpvaG5kb2VAZXhhbXBsZS5jb21cIl0iL
CAicGhvbmVfbnVtYmVyIjogIltcIlFnX082NHpxQXhlNDEyYTEwOGlyb0FcIiwgXCIrMS0
yMDItNTU1LTAxMDFcIl0iLCAiYWRkcmVzcyI6ICJbXCJBSngtMDk1VlBycFR0TjRRTU9xU
k9BXCIsIHtcInN0cmVldF9hZGRyZXNzXCI6IFwiMTIzIE1haW4gU3RcIiwgXCJsb2NhbGl
0eVwiOiBcIkFueXRvd25cIiwgXCJyZWdpb25cIjogXCJBbnlzdGF0ZVwiLCBcImNvdW50c
nlcIjogXCJVU1wifV0iLCAiYmlydGhkYXRlIjogIltcIlBjMzNKTTJMY2hjVV9sSGdndl9
1ZlFcIiwgXCIxOTQwLTAxLTAxXCJdIn19
```

(Line breaks for presentation only.)

## Format of an SD-JWT Release

SD-JWT-R contains claim values and the salts of the claims that the holder 
has consented to release to the Verifier. This enables the Verifier to verify 
the claims received from the holder by computing the hash digests of the claim
values and the salts revealed in the SD-JWT-R using the hashing algorithm 
specified in SD-JWT and comparing them to the hash digests included in SD-JWT.

For each claim, an array of the salt and the claim value is contained in the
`sd_release` object. The structure of `sd_release` object in the SD-JWT-R is the same as in SD-JWT. 

The SD-JWT-R MAY contain further claims, for example, to ensure a binding
to a concrete transaction (in the example the `nonce` and `aud` claims).

When the holder sends the SD-JWT-R to the Verifier, the SD-JWT-R MUST be a JWS 
represented as the JWS Compact Serialization as described in 
Section 7.1 of [@!RFC7515].

If holder binding is desired, the SD-JWT-R is signed by the holder. If no
holder binding is to be used, the `none` algorithm is used, i.e., the document
is not signed. TODO: Change to plain base64 to avoid alg=none issues

## Example: SD-JWT Release for Example 1

The following is a non-normative example of the contents of an SD-JWT-R for Example 1:

{#example-simple-release-payload}
```json
{
  "nonce": "XZOUco1u_gEPknxS78sWWg",
  "aud": "https://example.com/verifier",
  "sd_release": {
    "given_name": "[\"eluV5Og3gSNII8EYnsxA_A\", \"John\"]",
    "family_name": "[\"6Ij7tM-a5iVPGboS5tmvVA\", \"Doe\"]",
    "address": "[\"AJx-095VPrpTtN4QMOqROA\", {\"street_address\": \"123 Main St\", \"locality\": \"Anytown\", \"region\": \"Anystate\", \"country\": \"US\"}]"
  }
}
```

For each claim, an array of the salt and the claim value is contained in the
`sd_release` object. 

Again, the SD-JWT-R follows the same structure as the `sd_digests` in the SD-JWT. 

Below is a non-normative example of a representation of the SD-JWT-R JWS Compact
Serialization:

{#example-simple-release-encoded}
```
eyJhbGciOiAiUlMyNTYiLCAia2lkIjogIkxkeVRYd0F5ZnJpcjRfVjZORzFSYzEwVThKZE
xZVHJFQktKaF9oNWlfclUifQ.eyJub25jZSI6ICJYWk9VY28xdV9nRVBrbnhTNzhzV1dnI
iwgImF1ZCI6ICJodHRwczovL2V4YW1wbGUuY29tL3ZlcmlmaWVyIiwgInNkX3JlbGVhc2U
iOiB7ImdpdmVuX25hbWUiOiAiW1wiZWx1VjVPZzNnU05JSThFWW5zeEFfQVwiLCBcIkpva
G5cIl0iLCAiZmFtaWx5X25hbWUiOiAiW1wiNklqN3RNLWE1aVZQR2JvUzV0bXZWQVwiLCB
cIkRvZVwiXSIsICJhZGRyZXNzIjogIltcIkFKeC0wOTVWUHJwVHRONFFNT3FST0FcIiwge
1wic3RyZWV0X2FkZHJlc3NcIjogXCIxMjMgTWFpbiBTdFwiLCBcImxvY2FsaXR5XCI6IFw
iQW55dG93blwiLCBcInJlZ2lvblwiOiBcIkFueXN0YXRlXCIsIFwiY291bnRyeVwiOiBcI
lVTXCJ9XSJ9fQ.j5jtArW1QDK1BNM13sJbQaE00GsAhhiPRYi6oK-iJRtLWE6DAgcWxDir
TvxTnuo7Mbb6gSqGTmdEEtmscWxweFfGQoddObPTDiapjWiR1bUMMqPDKNNkRe0CBkU-pW
ieYWN-fQxlRa4BKUqs18jcvGtTGA8Ye-i6t2xLROeXf2U_Seko8b7MQWIFHdbc0LvEFt_-
JAcqeshH5wjVkwhHofVuZq1vGLlINKBveKA2dmn6wuEzi6XRceTwFrG_hTECagfobdO-bY
MF3FSiCQM2KxC_6_aLApYo0aH3zjBv9rm0qNmnL_JGN5FIu6YqwhvPzfdsfkjMd68o8LTW
d7F6kQ
```

(Line breaks for presentation only.)

## Sending SD-JWT and SD-JWT-R during Presentation

The SD-JWT and the SD-JWT-R can be combined into one document using period character `.` as a separator (here for Example 1):

{#example-simple-combined-sd-jwt-sd-jwt-release}
```
eyJhbGciOiAiUlMyNTYiLCAia2lkIjogImNBRUlVcUowY21MekQxa3pHemhlaUJhZzBZUk
F6VmRsZnhOMjgwTmdIYUEifQ.eyJpc3MiOiAiaHR0cHM6Ly9leGFtcGxlLmNvbS9pc3N1Z
XIiLCAic3ViX2p3ayI6IHsia3R5IjogIlJTQSIsICJuIjogInBtNGJPSEJnLW9ZaEF5UFd
6UjU2QVdYM3JVSVhwMTFfSUNEa0dnUzZXM1pXTHRzLWh6d0kzeDY1NjU5a2c0aFZvOWRiR
29DSkUzWkdGX2VhZXRFMzBVaEJVRWdwR3dyRHJRaUo5enFwcm1jRmZyM3F2dmtHanR0aDh
aZ2wxZU0yYkpjT3dFN1BDQkhXVEtXWXMxNTJSN2c2SmcyT1ZwaC1hOHJxLXE3OU1oS0c1U
W9XX21UejEwUVRfNkg0YzdQaldHMWZqaDhocFdObmJQX3B2NmQxelN3WmZjNWZsNnlWUkw
wRFYwVjNsR0hLZTJXcWZfZU5HakJyQkxWa2xEVGs4LXN0WF9NV0xjUi1FR21YQU92MFVCV
2l0U19kWEpLSnUtdlhKeXcxNG5IU0d1eFRJSzJoeDFwdHRNZnQ5Q3N2cWltWEtlRFRVMTR
xUUwxZUU3aWhjdyIsICJlIjogIkFRQUIifSwgImlhdCI6IDE1MTYyMzkwMjIsICJleHAiO
iAxNTE2MjQ3MDIyLCAiaGFzaF9hbGciOiAic2hhLTI1NiIsICJzZF9kaWdlc3RzIjogeyJ
zdWIiOiAiejR4Z0Vjbzk0ZGlUYVNydUlTUGlFN29fd3RtY09mbkhfOFI3WDlQYTU3OCIsI
CJnaXZlbl9uYW1lIjogIlB2VTdjV2p1SFVxNnctaTlYRnBRWmhqVC11cHJRTDNHSDNtS3N
BSmwwZTAiLCAiZmFtaWx5X25hbWUiOiAiSC1SZWxyNGNFQk1sZW55SzFndnl4MTZRVnBud
DRNRWNsVDV0UDBhVExGVSIsICJlbWFpbCI6ICJFVDJBMUpRTEY4NVpwQnVsaDZVRnN0R3J
TZlI0QjNLTS1ialFWbGxoeHFZIiwgInBob25lX251bWJlciI6ICJTSm5jaUIyRElSVkE1Y
1hCcmRLb0g2bjQ1Nzg4bVp5VW4ycm52NzR1TVZVIiwgImFkZHJlc3MiOiAiMEZsZHFMZkd
uRVJQUFZEQzE3b2Q5eGI0dzNpUkpURVFiV19ZazlBbW5EdyIsICJiaXJ0aGRhdGUiOiAiL
Uwwa01nSWJMWGUzT0VrS1RVR3d6X1FLaGplaERlb2ZLR3dvUHJ4THVvNCJ9fQ.TSxRFVMl
Nlt_drWLqQuzz9Sc19Bo5r-cgXZ0to8PfadImEJuPbhfkxCSt1xCT5IWQJrD6p_47h8Ac-
5yOtuu8s-wxrQzCUAU7OS2ThQGKiDKsGv0oiH3KQX4QyCAOIIaWOLxO6VQRxI7Kn6RFT0M
5jQxBNFmL582n_uJR31ANBO45Jc_pgp4iSTn2vf1lxxwU7B3bgrad0vHmvwmv1lu17Ov79
SGmT_fTjoybIJfHavTq61xCJ3UrMAGCEqm-_laOTxB1HRbPF38va0iuhqSSk2QaUpfDsVb
9VsPBVIx_MsGlK3kYUWcHzkJ0BBPIVu0xnVcLkg1_SO_5gPSxFWZ4g.eyJhbGciOiAiUlM
yNTYiLCAia2lkIjogIkxkeVRYd0F5ZnJpcjRfVjZORzFSYzEwVThKZExZVHJFQktKaF9oN
WlfclUifQ.eyJub25jZSI6ICJYWk9VY28xdV9nRVBrbnhTNzhzV1dnIiwgImF1ZCI6ICJo
dHRwczovL2V4YW1wbGUuY29tL3ZlcmlmaWVyIiwgInNkX3JlbGVhc2UiOiB7ImdpdmVuX2
5hbWUiOiAiW1wiZWx1VjVPZzNnU05JSThFWW5zeEFfQVwiLCBcIkpvaG5cIl0iLCAiZmFt
aWx5X25hbWUiOiAiW1wiNklqN3RNLWE1aVZQR2JvUzV0bXZWQVwiLCBcIkRvZVwiXSIsIC
JhZGRyZXNzIjogIltcIkFKeC0wOTVWUHJwVHRONFFNT3FST0FcIiwge1wic3RyZWV0X2Fk
ZHJlc3NcIjogXCIxMjMgTWFpbiBTdFwiLCBcImxvY2FsaXR5XCI6IFwiQW55dG93blwiLC
BcInJlZ2lvblwiOiBcIkFueXN0YXRlXCIsIFwiY291bnRyeVwiOiBcIlVTXCJ9XSJ9fQ.j
5jtArW1QDK1BNM13sJbQaE00GsAhhiPRYi6oK-iJRtLWE6DAgcWxDirTvxTnuo7Mbb6gSq
GTmdEEtmscWxweFfGQoddObPTDiapjWiR1bUMMqPDKNNkRe0CBkU-pWieYWN-fQxlRa4BK
Uqs18jcvGtTGA8Ye-i6t2xLROeXf2U_Seko8b7MQWIFHdbc0LvEFt_-JAcqeshH5wjVkwh
HofVuZq1vGLlINKBveKA2dmn6wuEzi6XRceTwFrG_hTECagfobdO-bYMF3FSiCQM2KxC_6
_aLApYo0aH3zjBv9rm0qNmnL_JGN5FIu6YqwhvPzfdsfkjMd68o8LTWd7F6kQ
```

(Line breaks for presentation only.)

# Verification

## Verification by the Holder when Receiving SD-JWT and SVC

The holder SHOULD verify the binding between SD-JWT and SVC by performing the following steps:
 1. Check that all the claims in the SVC are present in the SD-JWT and that there are no claims in the SD-JWT that are not in the SVC
 2. Check that the hashes of the claims in the SVC match those in the SD-JWT

## Verification by the Verifier when Receiving SD-JWT and SD-JWT-R

Verifiers MUST follow [@RFC8725] for checking the SD-JWT and, if signed, the
SD-JWT Release.

Verifiers MUST go through (at least) the following steps before
trusting/using any of the contents of an SD-JWT:

 1. Determine if holder binding is to be checked for the SD-JWT. Refer to (#holder_binding_security) for details.
 2. Check that the presentation consists of six period-separated (`.`) elements; if holder binding is not required, the last element can be empty.
 3. Separate the SD-JWT from the SD-JWT Release.
 4. Validate the SD-JWT:
    1. Check that the `typ` JOSE header parameter has the value sd+jwt
    2. Ensure that a signing algorithm was used that was deemed secure for the application. Refer to [@RFC8725], Sections 3.1 and 3.2 for details.
    3. Validate the signature over the SD-JWT. 
    4. Validate the issuer of the SD-JWT and that the signing key belongs to this issuer.
    5. Check that the SD-JWT is valid using `nbf`, `iat`, and `exp` claims, if provided in the SD-JWT.
    6. Check that the claim `sd_digests` is present in the SD-JWT.
    7. Check that the `hash_alg` claim is present and its value is understand and the hash algorithm deemed secure.
 5. Validate the SD-JWT Release:
    1. If holder binding is required, validate the signature over the SD-JWT using the same steps as for the SD-JWT plus the following steps:
       1. Determine that the public key for the private key that used to sign the SD-JWT-R is bound to the SD-JWT, i.e., the SD-JWT either contains a reference to the public key or contains the public key itself.
       2. Determine that the SD-JWT-R is bound to the current transaction and was created for this verifier (replay protection). This is usually achieved by a `nonce` and `aud` field within the SD-JWT Release.
    2. For each claim in the SD-JWT Release:
       1. Ensure that the claim is present as well in `sd_release` in the SD-JWT.
          If `sd_release` is structured, the claim MUST be present at the same
          place within the structure.
       2. Compute the base64url-encoded hash of a claim revealed from the Holder
          using the claim value and the salt included in the SD-JWT-R and 
          the `hash_alg` in SD-JWT.
       3. Compare the hash digests computed in the previous step with the one of the same claim in the SD-JWT. 
          Accept the claim only when the two hash digests match.
       4. Ensure that the claim value in the SD-JWT-R is a JSON-encoded
          array of exactly two values.
       5. Store the second of the two values. 
    3. Once all necessary claims have been verified, their values can be
       validated and used according to the requirements of the application. It
       MUST be ensured that all claims required for the application have been
       released.

If any step fails, the input is not valid and processing MUST be aborted.


# Security Considerations {#security_considerations}

## Mandatory hash computation of the revealed claim values by the Verifier

ToDo: add text explaining mechanisms that should be adopted to ensure that 
  verifiers validate the claim values received in SD-JWT-R by calculating the
  hashes of those values and comparing them with the hashes in the SD-JWT: 
  - create a test suite that forces hash computation by the Verifiers, 
    and includes negative test cases in test vectors
  - use only implementations/libraries that are compliant to the test suite 
  - etc.

## Mandatory signing of the SD-JWT

The SD-JWT MUST be signed by the issuer to protect integrity of the issued
claims. An attacker can modify or add claims if an SD-JWT is not signed (e.g.,
change the "email" attribute to take over the victim's account or add an
attribute indicating a fake academic qualification).

The verifier MUST always check the SD-JWT signature to ensure that the SD-JWT
has not been tampered with since its issuance. If the signature on the SD-JWT
cannot be verified, the SD-JWT MUST be rejected. 

## Entropy and Uniqueness of the salt

The security model relies on the fact that the salt is not learned or guessed by
the attacker. It is vitally important to adhere to this principle. As such, the
salt MUST be created in such a manner that it is cryptographically random,
long enough and has high entropy that it is not practical for the attacker to
guess. Each salt value MUST be unique.

## Minimum length of the salt

The length of the randomly-generated portion of the salt MUST be at least 128 bits.

## Choice of a hash function

For the security of this scheme, the hash function is required to be preimage and collision
resistant, i.e., it is infeasible to calculate the salt and claim value that result in
a particular digest, and it is infeasible to find a different salt and claim value pair that
result in a matching digest, respectively.

Furthermore the hash algorithms MD2, MD4, MD5, RIPEMD-160, and SHA-1 
revealed fundamental weaknesses and they MUST NOT be used.

## Holder Binding {#holder_binding_security}
TBD

# Privacy Considerations {#privacy_considerations}

## Claim Names

Claim names are not hashed in the SD-JWT and are used as keys in a key-value pair, where the value is the hash.
This is because SD-JWT already reveals information about the issuer and the schema,
and revealing the claim names does not provide any additional information.

## Unlinkability 

Colluding issuer/verifier or verifier/verifier pairs could link issuance/presentation or two presentation sessions
to the same user on the basis of unique values encoded in the SD-JWT
(issuer signature, salts, digests, etc.). More advanced cryptographic schemes, outside the scope of
this specification, can be used to prevent this type of linkability.

# Acknowledgements {#Acknowledgements}
      
We would like to thank ...

# IANA Considerations {#iana_considerations}

TBD

<reference anchor="OIDC" target="https://openid.net/specs/openid-connect-core-1_0.html">
  <front>
    <title>OpenID Connect Core 1.0 incorporating errata set 1</title>
    <author initials="N." surname="Sakimura" fullname="Nat Sakimura">
      <organization>NRI</organization>
    </author>
    <author initials="J." surname="Bradley" fullname="John Bradley">
      <organization>Ping Identity</organization>
    </author>
    <author initials="M." surname="Jones" fullname="Mike Jones">
      <organization>Microsoft</organization>
    </author>
    <author initials="B." surname="de Medeiros" fullname="Breno de Medeiros">
      <organization>Google</organization>
    </author>
    <author initials="C." surname="Mortimore" fullname="Chuck Mortimore">
      <organization>Salesforce</organization>
    </author>
   <date day="8" month="Nov" year="2014"/>
  </front>
</reference>

<reference anchor="VC_DATA" target="https://www.w3.org/TR/vc_data">
  <front>
    <title>Verifiable Credentials Data Model 1.0</title>
    <author fullname="Manu Sporny">
      <organization>Digital Bazaar</organization>
    </author>
    <author fullname="Grant Noble">
      <organization>ConsenSys</organization>
    </author>
    <author fullname="Dave Longley">
      <organization>Digital Bazaar</organization>
    </author>
    <author fullname="Daniel C. Burnett">
      <organization>ConsenSys</organization>
    </author>
    <author fullname="Brent Zundel">
      <organization>Evernym</organization>
    </author>
    <author fullname="David Chadwick">
      <organization>University of Kent</organization>
    </author>
    <date day="19" month="Nov" year="2019" />
  </front>
</reference>

{backmatter}

# Additional Examples

## Example 2 - Structured SD-JWT
This non-normative example is based on the same claim values as Example 1, but
this time the issuer decided to create a structured object for the hashes. This
allows for the release of individual members of the address claim separately.

{#example-simple_structured-sd-jwt-payload}
```json
{
  "iss": "https://example.com/issuer",
  "sub_jwk": {
    "kty": "RSA",
    "n": "pm4bOHBg-oYhAyPWzR56AWX3rUIXp11_ICDkGgS6W3ZWLts-hzwI3x65659kg4hVo9dbGoCJE3ZGF_eaetE30UhBUEgpGwrDrQiJ9zqprmcFfr3qvvkGjtth8Zgl1eM2bJcOwE7PCBHWTKWYs152R7g6Jg2OVph-a8rq-q79MhKG5QoW_mTz10QT_6H4c7PjWG1fjh8hpWNnbP_pv6d1zSwZfc5fl6yVRL0DV0V3lGHKe2Wqf_eNGjBrBLVklDTk8-stX_MWLcR-EGmXAOv0UBWitS_dXJKJu-vXJyw14nHSGuxTIK2hx1pttMft9CsvqimXKeDTU14qQL1eE7ihcw",
    "e": "AQAB"
  },
  "iat": 1516239022,
  "exp": 1516247022,
  "hash_alg": "sha-256",
  "sd_digests": {
    "sub": "Iuc2ft9sZwaw2yi_DZJKipiHMR_HbZP-5-WY8EKNqoA",
    "given_name": "TemA6ZTvHYgNfC8nzPC1x8q-yQxp5FpZjz-GQlSuJm4",
    "family_name": "Ro-RcqHFcflmKJ-YTnpL_roVgljLLJZuMoKiT714TpM",
    "email": "61L69IeXAGZxaWJocXvcou9-07PS8Ows4HNFO1tWqcg",
    "phone_number": "7WMDBVSk7vrJ4lBG0U6ze_5aCsK9N1UZD5Pwc1knOGY",
    "address": {
      "street_address": "O3YCyDFQkM4JdjRHWL78j_Z9Voi3RSYw3VBU0VJ4Lkk",
      "locality": "BhZ5ZsxM39vej8aWP1l3_VdK6ErWIgaRsOBhiigsarc",
      "region": "sns2ezJN5JHUPAvzSwxFIwPj_PrQbeiBGZujbwOYuJo",
      "country": "rf-11mZmiYQOM7jRKqgNqqk_Zk1RbQJv7VVi9q2uU80"
    },
    "birthdate": "_gsBVx70--mLOaPOrl8POf5Vr3Coubg2yiz5pMrOLDc"
  }
}
```

The SVC for this SD-JWT is as follows:

{#example-simple_structured-svc-payload}
```json
{
  "sd_release": {
    "sub": "[\"G02NSrQfjFXQ7Io09syajA\", \"6c5c0a49-b589-431d-bae7-219122a9ec2c\"]",
    "given_name": "[\"lklxF5jMYlGTPUovMNIvCA\", \"John\"]",
    "family_name": "[\"nPuoQnkRFq3BIeAm7AnXFA\", \"Doe\"]",
    "email": "[\"5bPs1IquZNa0hkaFzzzZNw\", \"johndoe@example.com\"]",
    "phone_number": "[\"5a2W0_NrlEZzfqmk_7Pq-w\", \"+1-202-555-0101\"]",
    "address": {
      "street_address": "[\"y1sVU5wdfJahVdgwPgS7RQ\", \"123 Main St\"]",
      "locality": "[\"HbQ4X8srVW3QDxnIJdqyOA\", \"Anytown\"]",
      "region": "[\"C9GSoujviJquEgYfojCb1A\", \"Anystate\"]",
      "country": "[\"kx5kF17V-x0JmwUx9vgvtw\", \"US\"]"
    },
    "birthdate": "[\"H3o1uswP760Fi2yeGdVCEQ\", \"1940-01-01\"]"
  }
}
```

An SD-JWT-R for the SD-JWT above that discloses only `region` and `country` of
the `address` property:

{#example-simple_structured-release-payload}
```json
{
  "nonce": "XZOUco1u_gEPknxS78sWWg",
  "aud": "https://example.com/verifier",
  "sd_release": {
    "given_name": "[\"lklxF5jMYlGTPUovMNIvCA\", \"John\"]",
    "family_name": "[\"nPuoQnkRFq3BIeAm7AnXFA\", \"Doe\"]",
    "birthdate": "[\"H3o1uswP760Fi2yeGdVCEQ\", \"1940-01-01\"]",
    "address": {
      "region": "[\"C9GSoujviJquEgYfojCb1A\", \"Anystate\"]",
      "country": "[\"kx5kF17V-x0JmwUx9vgvtw\", \"US\"]"
    }
  }
}
```

## Example 3 - Complex Structured SD-JWT

In this example, a complex object such as those used for OIDC4IDA (todo reference) is used.

In this example, the Issuer is using a following object as a set of claims to issue to the Holder:

{#example-complex_structured-sd-jwt-claims}
```json
{
  "verified_claims": {
    "verification": {
      "trust_framework": "de_aml",
      "time": "2012-04-23T18:25Z",
      "verification_process": "f24c6f-6d3f-4ec5-973e-b0d8506f3bc7",
      "evidence": [
        {
          "type": "document",
          "method": "pipp",
          "time": "2012-04-22T11:30Z",
          "document": {
            "type": "idcard",
            "issuer": {
              "name": "Stadt Augsburg",
              "country": "DE"
            },
            "number": "53554554",
            "date_of_issuance": "2010-03-23",
            "date_of_expiry": "2020-03-22"
          }
        }
      ]
    },
    "claims": {
      "given_name": "Max",
      "family_name": "Meier",
      "birthdate": "1956-01-28",
      "place_of_birth": {
        "country": "DE",
        "locality": "Musterstadt"
      },
      "nationalities": [
        "DE"
      ],
      "address": {
        "locality": "Maxstadt",
        "postal_code": "12344",
        "country": "DE",
        "street_address": "An der Weide 22"
      }
    }
  },
  "birth_middle_name": "Timotheus",
  "salutation": "Dr.",
  "msisdn": "49123456789"
}
```

The following shows the resulting SD-JWT payload:

{#example-complex_structured-sd-jwt-payload}
```json
{
  "iss": "https://example.com/issuer",
  "sub_jwk": {
    "kty": "RSA",
    "n": "pm4bOHBg-oYhAyPWzR56AWX3rUIXp11_ICDkGgS6W3ZWLts-hzwI3x65659kg4hVo9dbGoCJE3ZGF_eaetE30UhBUEgpGwrDrQiJ9zqprmcFfr3qvvkGjtth8Zgl1eM2bJcOwE7PCBHWTKWYs152R7g6Jg2OVph-a8rq-q79MhKG5QoW_mTz10QT_6H4c7PjWG1fjh8hpWNnbP_pv6d1zSwZfc5fl6yVRL0DV0V3lGHKe2Wqf_eNGjBrBLVklDTk8-stX_MWLcR-EGmXAOv0UBWitS_dXJKJu-vXJyw14nHSGuxTIK2hx1pttMft9CsvqimXKeDTU14qQL1eE7ihcw",
    "e": "AQAB"
  },
  "iat": 1516239022,
  "exp": 1516247022,
  "hash_alg": "sha-256",
  "sd_digests": {
    "verified_claims": {
      "verification": {
        "trust_framework": "ulse4qfN0Jsw6MJaDRpE7s4QfC9jl5xRsAdtLu5y0Nw",
        "time": "5erOXGVITtW2jAYqqZjgSKv9BDV64-ij7oxm70PH5mw",
        "verification_process": "R2IWSNyqDsKcTv8wbJvHacrXvxH41M2o2XyBPXDdBMs",
        "evidence": [
          {
            "type": "60D_WEYdd9fVIg4Yp5LYkF_bfm-CrPO0MD4afV53QxE",
            "method": "k8rl0LZgxWhSeIY-Pgx7ElPbrThBEqqgEphCWOepQoY",
            "time": "Dv2iv4WN2l-W5TjjRFGuD7q5-cpUg5h_QmD6oIHac6c",
            "document": {
              "type": "bboBy937YbIm5bteK-7RqM-8sC6v80bzJOjOHfnmSsc",
              "issuer": {
                "name": "Fd3FDJu1SNMUbUzRZzbANGoIlKvcK-nJJNlsO-Jcfzs",
                "country": "Oe1_kHEITWW2XL0_mAZkdwKdr4oxE00fkDjYHHPb7Zk"
              },
              "number": "EVf75JZoNwyubkCbGd3yg5TGOp3PFNPPd8IBJr4iC58",
              "date_of_issuance": "H0dpfU5qe-oqAfqqLPYIb0uPm6x2ARIyVsOcOtsvfxg",
              "date_of_expiry": "Qf39B8Li53nPLPfFCTiNP8fiSl3cVUL5nmVSgYG8bC4"
            }
          }
        ]
      },
      "claims": {
        "given_name": "YdDYUguv9oG-G8qrtwFQKEBpCqk6qcfgAEINzvuHcuI",
        "family_name": "7meD-4eVxg9gtdgS4uFi5n2sCjY2szjLPJVJDFm5Zuk",
        "birthdate": "iEPf4UDcUu-JYKSkuOQ50eL5sivCEvsVunHrywK8FcU",
        "place_of_birth": {
          "country": "gOL8yL-y0eICAytKN13yijyivVf-7Tej4KULwXG_JaQ",
          "locality": "pXVczldRi8FsPm5WuVaF-4Sw3E3WgcPtHuLOmIhMyYU"
        },
        "nationalities": "Nb6-_rHdYcq-4pfEvkZFNVa3FHn1LUiMrTeSyJ_d9rk",
        "address": "-EfqJRS0_hpL031rOyszFeDrbEurLQb-oGD2kjeaKiE"
      }
    },
    "birth_middle_name": "I51fdBdsVsfbH4FWgI3db4yVEfOgf2FSIjl8EDBfzl8",
    "salutation": "YygVZ2yM6udZml4xl-APnKVhbOAVn38otn8btB-eFDU",
    "msisdn": "4Abm9luJeisejT3a80C9BGXWX3MG1XF3UK94cVl4Mvg"
  }
}
```

The SD-JWT is then signed by the issuer to create a document like the following:

{#example-complex_structured-sd-jwt-encoded}
```
eyJhbGciOiAiUlMyNTYiLCAia2lkIjogImNBRUlVcUowY21MekQxa3pHemhlaUJhZzBZUk
F6VmRsZnhOMjgwTmdIYUEifQ.eyJpc3MiOiAiaHR0cHM6Ly9leGFtcGxlLmNvbS9pc3N1Z
XIiLCAic3ViX2p3ayI6IHsia3R5IjogIlJTQSIsICJuIjogInBtNGJPSEJnLW9ZaEF5UFd
6UjU2QVdYM3JVSVhwMTFfSUNEa0dnUzZXM1pXTHRzLWh6d0kzeDY1NjU5a2c0aFZvOWRiR
29DSkUzWkdGX2VhZXRFMzBVaEJVRWdwR3dyRHJRaUo5enFwcm1jRmZyM3F2dmtHanR0aDh
aZ2wxZU0yYkpjT3dFN1BDQkhXVEtXWXMxNTJSN2c2SmcyT1ZwaC1hOHJxLXE3OU1oS0c1U
W9XX21UejEwUVRfNkg0YzdQaldHMWZqaDhocFdObmJQX3B2NmQxelN3WmZjNWZsNnlWUkw
wRFYwVjNsR0hLZTJXcWZfZU5HakJyQkxWa2xEVGs4LXN0WF9NV0xjUi1FR21YQU92MFVCV
2l0U19kWEpLSnUtdlhKeXcxNG5IU0d1eFRJSzJoeDFwdHRNZnQ5Q3N2cWltWEtlRFRVMTR
xUUwxZUU3aWhjdyIsICJlIjogIkFRQUIifSwgImlhdCI6IDE1MTYyMzkwMjIsICJleHAiO
iAxNTE2MjQ3MDIyLCAiaGFzaF9hbGciOiAic2hhLTI1NiIsICJzZF9kaWdlc3RzIjogeyJ
2ZXJpZmllZF9jbGFpbXMiOiB7InZlcmlmaWNhdGlvbiI6IHsidHJ1c3RfZnJhbWV3b3JrI
jogInVsc2U0cWZOMEpzdzZNSmFEUnBFN3M0UWZDOWpsNXhSc0FkdEx1NXkwTnciLCAidGl
tZSI6ICI1ZXJPWEdWSVR0VzJqQVlxcVpqZ1NLdjlCRFY2NC1pajdveG03MFBINW13IiwgI
nZlcmlmaWNhdGlvbl9wcm9jZXNzIjogIlIySVdTTnlxRHNLY1R2OHdiSnZIYWNyWHZ4SDQ
xTTJvMlh5QlBYRGRCTXMiLCAiZXZpZGVuY2UiOiBbeyJ0eXBlIjogIjYwRF9XRVlkZDlmV
klnNFlwNUxZa0ZfYmZtLUNyUE8wTUQ0YWZWNTNReEUiLCAibWV0aG9kIjogIms4cmwwTFp
neFdoU2VJWS1QZ3g3RWxQYnJUaEJFcXFnRXBoQ1dPZXBRb1kiLCAidGltZSI6ICJEdjJpd
jRXTjJsLVc1VGpqUkZHdUQ3cTUtY3BVZzVoX1FtRDZvSUhhYzZjIiwgImRvY3VtZW50Ijo
geyJ0eXBlIjogImJib0J5OTM3WWJJbTVidGVLLTdScU0tOHNDNnY4MGJ6Sk9qT0hmbm1Tc
2MiLCAiaXNzdWVyIjogeyJuYW1lIjogIkZkM0ZESnUxU05NVWJVelJaemJBTkdvSWxLdmN
LLW5KSk5sc08tSmNmenMiLCAiY291bnRyeSI6ICJPZTFfa0hFSVRXVzJYTDBfbUFaa2R3S
2RyNG94RTAwZmtEallISFBiN1prIn0sICJudW1iZXIiOiAiRVZmNzVKWm9Od3l1YmtDYkd
kM3lnNVRHT3AzUEZOUFBkOElCSnI0aUM1OCIsICJkYXRlX29mX2lzc3VhbmNlIjogIkgwZ
HBmVTVxZS1vcUFmcXFMUFlJYjB1UG02eDJBUkl5VnNPY090c3ZmeGciLCAiZGF0ZV9vZl9
leHBpcnkiOiAiUWYzOUI4TGk1M25QTFBmRkNUaU5QOGZpU2wzY1ZVTDVubVZTZ1lHOGJDN
CJ9fV19LCAiY2xhaW1zIjogeyJnaXZlbl9uYW1lIjogIllkRFlVZ3V2OW9HLUc4cXJ0d0Z
RS0VCcENxazZxY2ZnQUVJTnp2dUhjdUkiLCAiZmFtaWx5X25hbWUiOiAiN21lRC00ZVZ4Z
zlndGRnUzR1Rmk1bjJzQ2pZMnN6akxQSlZKREZtNVp1ayIsICJiaXJ0aGRhdGUiOiAiaUV
QZjRVRGNVdS1KWUtTa3VPUTUwZUw1c2l2Q0V2c1Z1bkhyeXdLOEZjVSIsICJwbGFjZV9vZ
l9iaXJ0aCI6IHsiY291bnRyeSI6ICJnT0w4eUwteTBlSUNBeXRLTjEzeWlqeWl2VmYtN1R
lajRLVUx3WEdfSmFRIiwgImxvY2FsaXR5IjogInBYVmN6bGRSaThGc1BtNVd1VmFGLTRTd
zNFM1dnY1B0SHVMT21JaE15WVUifSwgIm5hdGlvbmFsaXRpZXMiOiAiTmI2LV9ySGRZY3E
tNHBmRXZrWkZOVmEzRkhuMUxVaU1yVGVTeUpfZDlyayIsICJhZGRyZXNzIjogIi1FZnFKU
lMwX2hwTDAzMXJPeXN6RmVEcmJFdXJMUWItb0dEMmtqZWFLaUUifX0sICJiaXJ0aF9taWR
kbGVfbmFtZSI6ICJJNTFmZEJkc1ZzZmJINEZXZ0kzZGI0eVZFZk9nZjJGU0lqbDhFREJme
mw4IiwgInNhbHV0YXRpb24iOiAiWXlnVloyeU02dWRabWw0eGwtQVBuS1ZoYk9BVm4zOG9
0bjhidEItZUZEVSIsICJtc2lzZG4iOiAiNEFibTlsdUplaXNlalQzYTgwQzlCR1hXWDNNR
zFYRjNVSzk0Y1ZsNE12ZyJ9fQ.p0sdyAUJPsuJGhwglGnk0wfREearQjwa3xoPv62fsBCw
sxNrwtrOcFWigKLeIcsTJmdzCiJ-rknQFLtXBTsyZtrn_6bUq8lMQm4XB2VuXi6y5Dnz7r
vvF5dwZZqhVQLD-sbTGMq5_kRus8c6lMgxVq2ndGq6PHcwA2FG7e8aK5DDyJj5RiD_4Z7R
gwdlggT3uHNIMdXKu7oNyBXjwah7T9wv1jz8zrmAIDiEd2UyF31wbT3kMgMpi0bxqS4Tsa
Obo9wzvmLvGJ3KBdjEDdseuNM3fKwUobcvKEsJqFG8a4u1uGkcg411tTSUM3RlpBfjZWSF
BZTS65JaR6QQeRMBlA.eyJzZF9yZWxlYXNlIjogeyJ2ZXJpZmllZF9jbGFpbXMiOiB7InZ
lcmlmaWNhdGlvbiI6IHsidHJ1c3RfZnJhbWV3b3JrIjogIltcIk9CS2xUVmx2TGctQWR3c
VlHYlA4WkFcIiwgXCJkZV9hbWxcIl0iLCAidGltZSI6ICJbXCJNMEpiNTd0NDF1YnJrU3V
5ckRUM3hBXCIsIFwiMjAxMi0wNC0yM1QxODoyNVpcIl0iLCAidmVyaWZpY2F0aW9uX3Byb
2Nlc3MiOiAiW1wiRHNtdEtOZ3BWNGRBSHBqcmNhb3NBd1wiLCBcImYyNGM2Zi02ZDNmLTR
lYzUtOTczZS1iMGQ4NTA2ZjNiYzdcIl0iLCAiZXZpZGVuY2UiOiBbeyJ0eXBlIjogIltcI
mVLNW81cEhmZ3VwUHBsdGoxcWhBSndcIiwgXCJkb2N1bWVudFwiXSIsICJtZXRob2QiOiA
iW1wiajdBRGRiMFVWYjBMaTBjaVBjUDBld1wiLCBcInBpcHBcIl0iLCAidGltZSI6ICJbX
CJXcHhKckZ1WDh1U2kycDRodDA5anZ3XCIsIFwiMjAxMi0wNC0yMlQxMTozMFpcIl0iLCA
iZG9jdW1lbnQiOiB7InR5cGUiOiAiW1wiYXRTbUZBQ1lNYkpWS0QwNW8zSmd0UVwiLCBcI
mlkY2FyZFwiXSIsICJpc3N1ZXIiOiB7Im5hbWUiOiAiW1wiNEt5UjMyb0ladC16a1d2RnF
iVUxLZ1wiLCBcIlN0YWR0IEF1Z3NidXJnXCJdIiwgImNvdW50cnkiOiAiW1wiY2hCQ3N5a
HloLUo4NkktYXdRRGlDUVwiLCBcIkRFXCJdIn0sICJudW1iZXIiOiAiW1wiZmxOUDFuY01
6OUxnLWM5cU1Jel85Z1wiLCBcIjUzNTU0NTU0XCJdIiwgImRhdGVfb2ZfaXNzdWFuY2UiO
iAiW1wib3ZvVjFya2hfQU5tODYxcVVBQTJBd1wiLCBcIjIwMTAtMDMtMjNcIl0iLCAiZGF
0ZV9vZl9leHBpcnkiOiAiW1widDhFQS10S3NoNXdaTUI2YnBqTGZUUVwiLCBcIjIwMjAtM
DMtMjJcIl0ifX1dfSwgImNsYWltcyI6IHsiZ2l2ZW5fbmFtZSI6ICJbXCJSN0Q2TGhsNTJ
1eGxvQlFGUnVsenpBXCIsIFwiTWF4XCJdIiwgImZhbWlseV9uYW1lIjogIltcInloM2NRU
0tuaGRHbXBWZ2QzeWRIMlFcIiwgXCJNZWllclwiXSIsICJiaXJ0aGRhdGUiOiAiW1wiQkF
vS05LNUNqbER5WGZDUjZOa0syQVwiLCBcIjE5NTYtMDEtMjhcIl0iLCAicGxhY2Vfb2ZfY
mlydGgiOiB7ImNvdW50cnkiOiAiW1widl9hem02ZC10cVRuZGFOdlg5LUpMUVwiLCBcIkR
FXCJdIiwgImxvY2FsaXR5IjogIltcIk5XQ1dTZ0lqSmtWVlZzcGV0eGRXeHdcIiwgXCJNd
XN0ZXJzdGFkdFwiXSJ9LCAibmF0aW9uYWxpdGllcyI6ICJbXCJuZ2tLUlNrbS01VktYR1g
5akNGTEhRXCIsIFtcIkRFXCJdXSIsICJhZGRyZXNzIjogIltcImVyczk3d3hPTGR1RnVoS
k5aOVZVVEFcIiwge1wibG9jYWxpdHlcIjogXCJNYXhzdGFkdFwiLCBcInBvc3RhbF9jb2R
lXCI6IFwiMTIzNDRcIiwgXCJjb3VudHJ5XCI6IFwiREVcIiwgXCJzdHJlZXRfYWRkcmVzc
1wiOiBcIkFuIGRlciBXZWlkZSAyMlwifV0ifX0sICJiaXJ0aF9taWRkbGVfbmFtZSI6ICJ
bXCJhaHNaai1oN2VWYlh6UG5QVng5NkhRXCIsIFwiVGltb3RoZXVzXCJdIiwgInNhbHV0Y
XRpb24iOiAiW1wiczM5dENVMVZ2S19rSi1zcW9KQmd6Z1wiLCBcIkRyLlwiXSIsICJtc2l
zZG4iOiAiW1wiLWYyakZoRE9GVEtfT0E5aUFobGtqZ1wiLCBcIjQ5MTIzNDU2Nzg5XCJdI
n19
```

(Line breaks for presentation only.)

A SD-JWT-R for some of the claims:

{#example-complex_structured-release-payload}
```json
{
  "nonce": "XZOUco1u_gEPknxS78sWWg",
  "aud": "https://example.com/verifier",
  "sd_release": {
    "verified_claims": {
      "verification": {
        "trust_framework": "[\"OBKlTVlvLg-AdwqYGbP8ZA\", \"de_aml\"]",
        "time": "[\"M0Jb57t41ubrkSuyrDT3xA\", \"2012-04-23T18:25Z\"]",
        "evidence": [
          {
            "type": "[\"eK5o5pHfgupPpltj1qhAJw\", \"document\"]"
          }
        ]
      },
      "claims": {
        "given_name": "[\"R7D6Lhl52uxloBQFRulzzA\", \"Max\"]",
        "family_name": "[\"yh3cQSKnhdGmpVgd3ydH2Q\", \"Meier\"]",
        "birthdate": "[\"BAoKNK5CjlDyXfCR6NkK2A\", \"1956-01-28\"]",
        "place_of_birth": {
          "country": "[\"v_azm6d-tqTndaNvX9-JLQ\", \"DE\"]"
        }
      }
    }
  }
}
```

## Example 4 - W3C Verifiable Credentials Data Model

This example illustrates how the artifacts defined in this specification can be
represented using W3C Verifiable Credentials Data Model as defined in
[@VC_DATA].

Below is a non-normative example of an SD-JWT represented as a verifiable credential 
encoded as JSON and signed as JWS compliant to [@VC_DATA].

SVC sent alongside this SD-JWT as a JWT-VC is same as in Example 1.

```json
{
  "sub": "did:example:ebfeb1f712ebc6f1c276e12ec21",
  "jti": "http://example.edu/credentials/3732",
  "iss": "https://example.com/keys/foo.jwk",
  "nbf": 1541493724,
  "iat": 1541493724,
  "exp": 1573029723,
  "vc": {
    "@context": [
      "https://www.w3.org/2018/credentials/v1",
      "https://www.w3.org/2018/credentials/examples/v1"
    ],
    "type": [
      "VerifiableCredential",
      "UniversityDegreeCredential"
    ]
  },
  "sd_digests": {
    "given_name": "fUMdn88aaoyKTHrvZd6AuLmPraGhPJ0zF5r_JhxCVZs",
    "family_name": "9h5vgv6TpFV6GmnPtugiMLl5tHetHeb5X_2cKHjN7cw",
    "birthdate": "fvLCnDm3r4VSYcBF3pIlXP4ulEoHuHOfG_YmFZEuxpQ"
  }
}
```

Below is a non-normative example of an SD-JWT-R represented as a verifiable presentation
encoded as JSON and signed as a JWS compliant to [@VC_DATA].

```json
{
  "iss": "did:example:ebfeb1f712ebc6f1c276e12ec21",
  "aud": "s6BhdRkqt3",
  "nbf": 1560415047,
  "iat": 1560415047,
  "exp": 1573029723,
  "nonce": "660!6345FSer",
  "vp": {
    "@context": [
      "https://www.w3.org/2018/credentials/v1"
    ],
    "type": [
      "VerifiablePresentation"
    ],
    "verifiableCredential": ["eyJhb...npyXw"]
  },
  "sd_release": {
    "given_name": "[\"6Ij7tM-a5iVPGboS5tmvVA\", \"John\"]",
    "family_name": "[\"eI8ZWm9QnKPpNPeNenHdhQ\", \"Doe\"]",
    "birthdate": "[\"5bPs1IquZNa0hkaFzzzZNw\", \"1940-01-01\"]"
  }
}
```

# Document History

   [[ To be removed from the final specification ]]

   -01
   
   *  Editorial fixes
   *  Added hash_alg claim
   *  Renamed `_sd` to `sd_digests` and sd_release
   *  Added descriptions on holder binding - more work to do
   *  Clarify that signing the SD-JWT is mandatory

   -00

   *  Renamed to SD-JWT (focus on JWT instead of JWS since signature is optional)
   *  Make holder binding optional
   *  Rename proof to release, since when there is no signature, the term "proof" can be misleading
   *  Improved the structure of the description
   *  Described verification steps
   *  All examples generated from python demo implementation
   *  Examples for structured objects

