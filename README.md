# Standalone MTC ACME interoperability

Implementation evidence for ACME issuance and retrieval of standalone Merkle Tree Certificates, and for storage and installation of the standalone certificate and the later landmark-relative certificate.

This repository is the joint note from [TrustAsia](trust-asia.md) (issuance) and [CertLocker](certlocker.md) (storage and installation).

Readers are assumed to know ACME ([RFC 8555](https://www.rfc-editor.org/rfc/rfc8555)) and the PLANTS Merkle Tree Certificates work.

## Specifications

| Document | Role in this report |
| --- | --- |
| [draft-ietf-plants-merkle-tree-certs-08](https://datatracker.ietf.org/doc/draft-ietf-plants-merkle-tree-certs/) | Protocol. ACME issuance and the `acme-optional-alternate` retrieval pattern are Section 9. CA identifiers are Section 5.1. Standalone and landmark-relative certificate construction are Sections 6.3 and 6.4. |
| [draft-ietf-acme-profiles](https://datatracker.ietf.org/doc/draft-ietf-acme-profiles/) | How the client names the MTC product on `newOrder`. |
| [draft-elkins-plants-enterprise-mtc-deployment-01](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01) | The deployment draft that asked for this evidence. |

The results below are implementation evidence for these parts of the deployment draft:

- [Section 5.1](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01#section-5.1), issuance and retrieval of the complete standalone artifact, and retention of that artifact by the certificate manager.
- [Section 5.3.11.3](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01#section-5.3.11.3), a vendor result with an explicit support level, versions, and dependencies, from the parties who ran the test.
- The issuance and retrieval portion of Test 1 in [Section 11](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01#section-11): enroll a standalone MTC from a compatible CA, and preserve the artifact bytes and metadata in the certificate manager.

## Reports

| Party | Scope | Report |
| --- | --- | --- |
| [![TrustAsia](images/trustasia-logo.svg)](trust-asia.md) | ACME server, issuance log, cosignatures, standalone certificate, later landmark-relative certificate | [trust-asia.md](trust-asia.md) |
| [![CertLocker](images/certlocker-logo.svg)](certlocker.md) | Profile selection, retrieval, byte preservation, paired records, installation of both certificates. Contact [mtc@certlocker.io](mailto:mtc@certlocker.io) | [certlocker.md](certlocker.md) |

## What was exercised

In August 2026, CertLocker completed an ACME order against TrustAsia's MTC staging CA for `mtc.certlocker.dev`, using the profile name `mtc-47d`. The workflow recorded the standalone certificate, waited, then recorded the landmark-relative certificate and linked the two records. That run is a staging interoperability result. The staging CAs are in the Chrome Quantum-resistant Root Program under `UNTRUSTED_VALIDATION_ONLY`.

On 18 November 2026 the same staging directory advertises `mtc-7d-07` and `mtc-47d-07` for new orders. The August profile name is retained for orders already accepted. Details and the directory document are in the TrustAsia report.

[mtcdemo.trustasia.cn](https://mtcdemo.trustasia.cn/) handles Merkle Tree Certificates at the TLS level. The server holds a classical certificate, a standalone MTC, and a landmark-relative MTC, and selects one of them from `trust_anchors` in the ClientHello. CertLocker's application Trust probes that handshake. When the handshake will support an MTC, Trust serves the MTC.

## Support level

Categories follow Section 5.3.11.3 of the deployment draft.

| Capability | Level | Where it is described |
| --- | --- | --- |
| TrustAsia staging CA: ACME order, domain control validation, issuance of a standalone MTC, later landmark-relative certificate on the same order | Tested interoperability, staging only | [trust-asia.md](trust-asia.md) |
| CertLocker: select the directory profile, store the standalone artifact, wait, store the landmark-relative artifact, keep the two records linked | Tested interoperability, staging only | [certlocker.md](certlocker.md) |
| Preservation of certificate bytes and issuance metadata in the certificate manager | Tested as paired inventory records. A byte-level sample is not in this repository yet | [certlocker.md](certlocker.md) |
| Relying-party validation of the inclusion proof and cosignatures | A certificate-manager storage result. Validation remains the relying party's job | [certlocker.md](certlocker.md) |
| TLS handshake selection among a classical certificate, a standalone MTC, and a landmark-relative MTC | Tested on [mtcdemo.trustasia.cn](https://mtcdemo.trustasia.cn/) | [trust-asia.md](trust-asia.md) |
| Trust probes the handshake and serves the MTC when the handshake supports it | Tested interoperability, staging demonstration | [certlocker.md](certlocker.md) |
| Renewal at 75% of certificate lifespan, with an expiry warning | Default behavior of Trust. The probe compares the stored thumbprint with the certificate being served | [certlocker.md](certlocker.md) |
| Multi-node rollout | CertLocker already installs one certificate on every selected host with a single automation | [certlocker.md](certlocker.md) |
| Revocation, emergency withdrawal, failure recovery | Further testing relative to Tests 6, 7, and 12 | Both reports |

## Further testing

Trust is the live handshake check against the demonstration host: it probes, it serves the MTC when that handshake will support one, and it compares thumbprints to confirm the installation. By default, Trust renews a certificate at 75% of its lifespan and warns as expiry approaches. Revocation and failure recovery remain further testing under Tests 6, 7, and 12 in Section 11.
