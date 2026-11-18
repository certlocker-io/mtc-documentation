# TrustAsia — ACME issuance of standalone Merkle Tree Certificates

TrustAsia operates the staging certification authority. This file is the issuance report. Storage, byte preservation, and installation are in the [CertLocker report](certlocker.md). The repository index is the [README](README.md).

![TrustAsia](images/trustasia-logo.svg)

## 1. Implementation and test scope

| Item | Value |
| --- | --- |
| Role | ACME server and MTC CA for the staging interoperability environment |
| Directory observed | `https://acme.mtc-staging.trustasia.com/acme/v2/directory` |
| Observation date for the directory and profile page | 18 November 2026 |
| Client interoperability run | August 2026, with [CertLocker](certlocker.md), identifier `mtc.certlocker.dev`, profile name `mtc-47d` |
| Protocol text cited for section numbers | [draft-ietf-plants-merkle-tree-certs-08](https://www.ietf.org/archive/id/draft-ietf-plants-merkle-tree-certs-08.html) (1 November 2026) |
| Profile negotiation | [draft-ietf-acme-profiles](https://datatracker.ietf.org/doc/draft-ietf-acme-profiles/) |
| Evidence target | Section 5.1, Section 5.3.11.3, and the issuance half of Test 1 in Section 11 of [draft-elkins-plants-enterprise-mtc-deployment-01](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01) |

Two public descriptions bracket the August run:

- On 18 July 2026 TrustAsia described the staging environment against [draft-ietf-plants-merkle-tree-certs-05](https://datatracker.ietf.org/doc/draft-ietf-plants-merkle-tree-certs/). The same description is in the PQWS workshop note [slides-pqws-s26-00](https://www.ietf.org/slides/slides-pqws-s26-00.pdf).
- The public demonstration at [mtcdemo.trustasia.cn](https://mtcdemo.trustasia.cn/), fetched on 18 November 2026, states that the TLS demonstration implements the draft-07 X.509 envelope. The directory fetched the same day offers the draft-07 profile names `mtc-7d-07` and `mtc-47d-07`.

The August CertLocker order therefore ran against the server deployed at that time, which TrustAsia had described relative to draft-05, and it selected the unqualified profile name `mtc-47d`. The directory document in Section 7 is the server as observed on 18 November 2026. Those are different revisions of the same staging service.

Support level for Section 5.3.11.3: **tested interoperability, staging only**. The staging CAs and the TrustAsia mirror are listed in the Chrome Quantum-resistant Root Program under `UNTRUSTED_VALIDATION_ONLY`. Enrollment against that realm is the result this report records. Production deployment, in the sense of Section 5.3.11.3, waits on a root program that ships the anchors by default and on a supported product stack around them.

TrustAsia also maintains an MTC-aware fork of acme.sh at [`trustasia-com/acme.sh`, branch `feat/mtc-certificate-support`](https://github.com/trustasia-com/acme.sh/tree/feat/mtc-certificate-support). That client is independent of CertLocker. The August result is a second, separate ACME client against the same staging directory.

## 2. What the staging CA issues

The service has two test CAs. Each CA has its own issuance log. Landmark allocation and subtree cosignatures happen after the log append; the standalone certificate is the artifact that carries an inclusion proof and cosignatures, as in Section 6.3 of the MTC draft. The landmark-relative certificate for the same log entry is produced after a later landmark and is served from the same ACME order, as in Section 9.2.

The profile page gives the product distinction:

| Profile family | Validity | Revocation | CA and cosigner algorithms |
| --- | --- | --- | --- |
| `mtc-7d`, and the draft-07 name `mtc-7d-07` | 7 days | Exempt | ML-DSA-44 |
| `mtc-47d`, and the draft-07 name `mtc-47d-07` | 47 days | Required | ML-DSA-44 |

The algorithm line on the profile page is the CA and the cosigner. The end-entity public-key algorithm is a property of the issued certificate and is recorded with the sample PEM when that PEM is added.

The demonstration publishes these CA identifiers for the staging environment. They are the Section 5.1 CA IDs a relying party configures:

| Entity | OID |
| --- | --- |
| 7-day CA | `1.3.6.1.4.1.44494.3.1.3` |
| 47-day CA | `1.3.6.1.4.1.44494.3.1.4` |
| 7-day log 1 | `1.3.6.1.4.1.44494.3.1.3.0.1` |
| 47-day log 1 | `1.3.6.1.4.1.44494.3.1.4.0.1` |
| Mirror cosigner | `1.3.6.1.4.1.44494.3.2.1` |

Geomys cosigns both issuance logs, as stated in the PQWS note, which gives the standalone certificate an external cosignature in addition to the TrustAsia mirror. A decoded cosigner list from the August certificate is part of the sample PEM still to be added.

## 3. ACME workflow, and what changed for MTC

### Reused without an MTC-specific verb

Account registration, order creation, authorization, challenge validation, finalization, the order `certificate` URL, revocation, and key change are the RFC 8555 resources. The directory also publishes `renewalInfo` ([RFC 9773](https://www.rfc-editor.org/rfc/rfc9773)). Renewal scheduling is a later test; advertising the resource is recorded here so clients can see it, and it is not part of the August issuance result.

The directory `meta` object fetched on 18 November 2026 contains `termsOfService`, `website`, `caaIdentities` (`trustasia.com`), and `profiles`. It does not contain `externalAccountRequired`.

One additional directory member is present. Its value is the Let's Encrypt discussion of random directory entries, `https://community.letsencrypt.org/t/adding-random-entries-to-the-directory/33417`. Clients that require a closed set of directory keys will fail on this server. Clients that ignore unknown directory members interoperate. The member name is not a stable API.

### MTC-specific on the server

The client asks for an MTC by setting `profile` on `newOrder` to a name the directory currently advertises. The profile page states that an unknown or incompatible name produces `invalidProfile`. The landmark-relative certificate is the optional alternate of that same order.

Section 9.2 of the MTC draft is the server behavior this environment is implementing:

1. The order becomes `valid` when the standalone certificate is ready. The order `certificate` URL serves that standalone certificate.
2. The standalone response carries an `acme-optional-alternate` link to the landmark-relative certificate.
3. Until the next landmark is allocated, that URL responds with HTTP 202 and `Retry-After`.
4. After the landmark, the same URL serves the landmark-relative certificate.
5. A client that understands MTC sends `Accept: application/pem-certificate-chain-with-properties` and receives a `CertificatePropertyList`, including trust-anchor identifiers for the standalone certificate and for the landmark-relative certificate.

The August CertLocker run stored a standalone certificate and, later, a landmark certificate for one order. That outcome matches this shape. A captured `Link` header and a captured `CertificatePropertyList` are not in this repository yet, so this file does not quote those bytes. The directory and the profile document below were fetched and are quoted.

### Profile names are revision-qualified

The profile page documents four names:

- `mtc-7d` and `mtc-47d`, the names without a draft suffix.
- `mtc-7d-07` and `mtc-47d-07`, described as the draft-07 profiles, each with its own CA and log.

The page also states the transition rule: retained profiles continue to serve accepted orders and certificates, and new orders use only profiles advertised by the directory.

On 18 November 2026 the directory advertised only:

```text
mtc-7d-07
mtc-47d-07
```

The August interop selected `mtc-47d`. A client that still sends `mtc-47d` on a new order receives `invalidProfile`. A client that wants the same 47-day product sends `mtc-47d-07` and follows the directory rather than a cached name. The profile page's own `newOrder` example still shows `"profile": "mtc-7d"`, which is no longer an advertised name. The directory is authoritative; the example on that page is stale.

## 4. Artifact the server returns

Both artifacts are X.509 certificates whose proof is an MTC proof, as in Section 6.2 of the MTC draft.

| Artifact | When it is served | What it carries |
| --- | --- | --- |
| Standalone certificate | Order `certificate` URL, once the order is `valid` | Inclusion proof to a cosigned subtree, plus the cosignatures a relying party needs in order to accept that subtree. Section 6.3. |
| Landmark-relative certificate | `acme-optional-alternate` URL, after landmark allocation | Inclusion proof to a landmark subtree. Cosignatures are omitted because the relying party is expected to already have the landmark. Section 6.4. |

The standalone certificate is the complete artifact Section 5.1 of the deployment draft asks the client to retrieve. The landmark-relative certificate is a second encoding of the same issuance, returned on the same order. It is smaller, and it is useless to a relying party that does not already have the landmark.

The server's current profile page describes both the 7-day and the 47-day draft-07 profiles as having their own CA and log. A certificate issued under `mtc-47d` and a certificate issued under `mtc-47d-07` are not interchangeable artifacts from one log. Clients and inventory should store the profile name with the bytes.

No sample certificate is included in this revision. Publishing one requires a sanitized PEM from an accepted staging order, with account URLs and any subscriber data removed. The August order's bytes, if retained, would be served under the retained `mtc-47d` profile and would need a separate label from a certificate minted under `mtc-47d-07`.

## 5. TLS handshake on the demonstration host

The ACME service returns the certificates. [mtcdemo.trustasia.cn](https://mtcdemo.trustasia.cn/) handles Merkle Tree Certificates at the TLS level. The hostname `mtcdemo.trustasia.com` does not resolve; the demonstration that answers is the `.cn` host.

That host holds three credentials for one identity: a classical X.509 chain, a standalone MTC, and a landmark-relative MTC. On each TLS 1.3 handshake it reads `trust_anchors` in the ClientHello and selects one of them, as described in Section 8 of the MTC draft:

| ClientHello | Certificate the server sends |
| --- | --- |
| A landmark identifier or landmark group that covers this certificate | Landmark-relative certificate |
| The MTC CA identifier, with no usable landmark match | Standalone certificate, carrying the inclusion proof and cosignatures |
| No matching MTC anchor, including a ClientHello that omits `trust_anchors` | Classical X.509 chain |

The server sends one certificate for the connection. The page fetched on 18 November 2026 describes this selection against draft-07, and records that the demonstration sends the standalone certificate when the ClientHello carries `trust_anchors`.

CertLocker's application Trust probes this handshake. When the handshake will support an MTC, Trust serves the MTC. That behavior is described in the [CertLocker report](certlocker.md).

On 3 September 2026, Chrome Canary 155.0.8039.0 with the MTC test flags and `cosigners.json` 2.0.6 validated a standalone certificate carrying a Geomys cosignature, and a landmark-relative certificate, on this host.

## 6. Interoperability findings

| Finding | What implementers should do |
| --- | --- |
| Profile identifiers gained a draft suffix. `mtc-47d` issued the August certificate. New orders require `mtc-47d-07` or `mtc-7d-07`. | Read `meta.profiles` on every new order. Treat a cached profile name as a hint. Handle `invalidProfile` by refreshing the directory. |
| Retained profiles still serve old orders. | Keep using the order's existing `certificate` URL. Do not re-issue solely because the directory dropped the old name from `meta.profiles`. |
| The profile HTML example still says `mtc-7d`. | Prefer the directory over the example in the HTML. |
| The directory contains one unrecognized member, pointing at the random-directory-entry discussion. | Ignore unknown directory members. |
| `renewalInfo` is published. | Implement ARI separately from landmark retrieval. `Retry-After` on `acme-optional-alternate` is the delay for the second certificate of the current order. `renewalInfo` is the window for the next order. |
| 7-day and 47-day products differ in revocation, lifetime, CA, and log. | Store the profile and the CA ID with the certificate. A 7-day artifact is the revocation-exempt class. A 47-day artifact is the class that carries revocation information. |
| Draft-07 profiles use a distinct CA and log from the earlier names. | A trust-anchor configuration for the earlier staging CA does not automatically cover `mtc-47d-07`. Confirm the CA ID in the issued certificate. |
| The landmark is later than the standalone, on purpose. | Move the order to `valid` when the standalone certificate is ready. Serve the landmark as an optional alternate. An HTTP 202 on that URL is the specified waiting state. |

The August client selected `mtc-47d`, received a standalone certificate, waited, and received the landmark certificate. The rename finding is from the directory fetched after that run.

## 7. Reproducible server evidence

Collected 18 November 2026 with an unauthenticated `GET`. No account was created for this note.

### Directory

`GET https://acme.mtc-staging.trustasia.com/acme/v2/directory`

Response `200`, `content-type: application/json`. The random directory member is omitted; its value is the community URL cited in Section 3. Every other field is as returned:

```json
{
  "newNonce": "https://acme.mtc-staging.trustasia.com/acme/v2/new-nonce",
  "newAccount": "https://acme.mtc-staging.trustasia.com/acme/v2/new-account",
  "newOrder": "https://acme.mtc-staging.trustasia.com/acme/v2/new-order",
  "revokeCert": "https://acme.mtc-staging.trustasia.com/acme/v2/revoke-cert",
  "keyChange": "https://acme.mtc-staging.trustasia.com/acme/v2/key-change",
  "renewalInfo": "https://acme.mtc-staging.trustasia.com/acme/v2/renewal-info",
  "meta": {
    "termsOfService": "https://repository.trustasia.com/repo/sa/TrustAsia-SA_EN_V1.2.3.pdf",
    "website": "https://acme.mtc-staging.trustasia.com/",
    "caaIdentities": ["trustasia.com"],
    "profiles": {
      "mtc-7d-07": "https://acme.mtc-staging.trustasia.com/acme/v2/profiles#mtc-7d-07",
      "mtc-47d-07": "https://acme.mtc-staging.trustasia.com/acme/v2/profiles#mtc-47d-07"
    }
  }
}
```

### Profile document

`GET https://acme.mtc-staging.trustasia.com/acme/v2/profiles` returns `text/html` describing `mtc-7d`, `mtc-47d`, `mtc-7d-07`, and `mtc-47d-07`, and states that new orders follow the directory.

The page's illustration of `newOrder`, which uses a name the current directory does not advertise:

```json
{
  "profile": "mtc-7d",
  "identifiers": [{"type": "dns", "value": "www.example.org"}]
}
```

A new order against the directory observed on 18 November 2026 uses an advertised name:

```json
{
  "profile": "mtc-47d-07",
  "identifiers": [{"type": "dns", "value": "www.example.org"}]
}
```

`www.example.org` is a documentation identifier. It is not the name used in the August run. The August name was `mtc.certlocker.dev` with `"profile": "mtc-47d"`.

### Steps to repeat the server observation

1. `GET` the directory URL. Record `meta.profiles` and the date.
2. `GET` the profile URL linked from `meta.profiles`. Record validity, revocation class, and which names the HTML still documents.
3. Compare the advertised set with any profile name stored from an older order.
4. To repeat issuance, register an account, create an order with an advertised profile, complete the challenge for a name the tester controls, and finalize. Save the order URL, the `certificate` response including `Link` headers, the first response from the `acme-optional-alternate` URL, and the eventual landmark response. Strip account keys and subscriber contact data before committing them here.

### Known limitations of this issuance report

- The quoted server document is from 18 November 2026. The August certificate was issued under `mtc-47d`, before the directory was reduced to the `-07` names.
- A sanitized standalone PEM, its `Link` header, the HTTP 202 landmark response, and the landmark PEM are not in this repository yet.
- End-entity key algorithm, proof length, and cosigner set require that PEM. They are not inferred here.
- Handshake selection on the demonstration host is Section 5. ARI renewal, revocation, and failure recovery remain further tests.
- Contact for a staging account is `mtc@trustasia.com`.

## 8. References

- MTC draft: <https://datatracker.ietf.org/doc/draft-ietf-plants-merkle-tree-certs/>
- Section 9.2, ACME: <https://www.ietf.org/archive/id/draft-ietf-plants-merkle-tree-certs-08.html#section-9.2>
- Section 5.1, CA identifiers: <https://www.ietf.org/archive/id/draft-ietf-plants-merkle-tree-certs-08.html#section-5.1>
- ACME profiles: <https://datatracker.ietf.org/doc/draft-ietf-acme-profiles/>
- Deployment draft, Section 5.1, Section 5.3.11.3, and Section 11: <https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01>
- Staging directory: <https://acme.mtc-staging.trustasia.com/acme/v2/directory>
- Profile page: <https://acme.mtc-staging.trustasia.com/acme/v2/profiles>
- Demonstration: <https://mtcdemo.trustasia.cn/>
