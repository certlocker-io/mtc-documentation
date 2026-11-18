# CertLocker — storage and installation

简体中文：[certlocker.zh-CN.md](certlocker.zh-CN.md)

CertLocker is the ACME client and certificate manager. This file covers retrieval, preservation, and installation of the standalone certificate and the landmark-relative certificate, and the TLS handshake probe performed by CertLocker's application Trust. Issuance, the staging directory, and the profile catalogue are in the [TrustAsia report](trust-asia.md). The repository index is the [README](README.md).

![CertLocker](images/certlocker-logo.svg)

## 1. Implementation and test scope

| Item | Value |
| --- | --- |
| Role | ACME client, inventory, and installation of both certificates for one order |
| Staging directory | `https://acme.mtc-staging.trustasia.com/acme/v2/directory` |
| Run | August 2026. Public description: [CertLocker, 1 October 2026](https://certlocker.io/blog/wake-up-200-day-certificates-mtcs/) |
| Identifier | `mtc.certlocker.dev` |
| Profile sent on that order | `mtc-47d` |
| Workflow states recorded | Certificate Issued, then the landmark wait, then Landmark Issued, then Complete |
| Evidence target | The retrieval and preservation half of Test 1 in Section 11 of [draft-elkins-plants-enterprise-mtc-deployment-01](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01), together with the certificate-manager column of [Section 5.3.11.3](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01#section-5.3.11.3) |
| Contact | [mtc@certlocker.io](mailto:mtc@certlocker.io) |

Support level: **tested interoperability, staging only**. The order ran against TrustAsia's staging CA. The resulting certificates inherit that CA's `UNTRUSTED_VALIDATION_ONLY` status in the Chrome Quantum-resistant Root Program.

CertLocker's own ACME directory advertises `mtc7d` and `mtc47d` for certificates issued by CertLocker. The August order was sent to the TrustAsia staging directory, and the profile string was TrustAsia's `mtc-47d`.

The August server was the revision TrustAsia had described against draft-05. Section numbers in this file are from [draft-ietf-plants-merkle-tree-certs-08](https://www.ietf.org/archive/id/draft-ietf-plants-merkle-tree-certs-08.html).

## 2. What the client reuses, and what it does for MTC

Account, order, authorization, challenge, and finalization are unchanged RFC 8555. The MTC-specific client behavior on top of that order is:

1. Discover `meta.profiles` and send the selected name in `newOrder`, per [draft-ietf-acme-profiles](https://datatracker.ietf.org/doc/draft-ietf-acme-profiles/). The August order sent `mtc-47d`.
2. When the order is `valid`, download the certificate at the order's `certificate` URL. That response is the standalone certificate. CertLocker stores it as the standard certificate for the name.
3. Follow the landmark resource for the same order. While that resource is pending, keep the standalone record and keep polling. The August workflow shows this as the gap between Certificate Issued and Landmark Issued.
4. Store the landmark-relative certificate as a second record linked to the same order and the same name. Complete means both records exist and are linked.

Section 9.1 of the MTC draft defines the pending landmark as HTTP 202 with `Retry-After` on an `acme-optional-alternate` URL, and it tells the client to retry that URL independently of the standalone certificate. The August states match that pattern: the standalone record exists before the landmark record, and the landmark wait is part of the same workflow. The `Link` header and the 202 response from that run are not committed in this repository, so the header name is the draft's name for the pattern the workflow exhibited.

On download, a client that supports MTC sends `Accept: application/pem-certificate-chain-with-properties` (Section 9.2) so the response can carry a `CertificatePropertyList`. The property list is how trust-anchor identifiers travel with the bytes. A captured property list from the August order is not in this repository yet. Inventory for that order records the profile, the identifier, and the link between the two certificate records.

## 3. The two records

One issuance produces two certificates. CertLocker keeps them as two records and one workflow.

| Workflow state | Record | Draft artifact | Installation consequence |
| --- | --- | --- | --- |
| Certificate Issued | Standard certificate for `mtc.certlocker.dev` | Standalone certificate, Section 6.3. Inclusion proof and cosignatures are inside the certificate. | Eligible to install. A relying party with only the CA's cosigner configuration can use this artifact. |
| Landmark wait | Standalone record unchanged | `acme-optional-alternate` still pending | The wait extends the workflow. It does not withdraw the standalone record. |
| Landmark Issued | Second record, same order, same name | Landmark-relative certificate, Section 6.4. Inclusion proof to a landmark subtree. | Installed beside the standalone certificate. Section 6.4 says the authenticating party should deploy a standalone certificate alongside any landmark-relative certificate. |
| Complete | Both records linked | Same issuance, two proofs | Inventory can open either certificate from the workflow. |

The landmark certificate is an alternate proof of the log entry already stored. Replacing the standalone bytes with the landmark bytes would drop the cosignatures that a relying party without the landmark still needs. The August workflow linked the records; it left both in place.

Metadata kept with the pair, as shown by that workflow:

- the DNS identifier
- the profile name sent on the order (`mtc-47d` for this run)
- the order that produced both certificates
- which record is the standalone certificate and which is the landmark-relative certificate

A later order under `mtc-47d-07` is a different CA and log. The profile string is part of the record so the two generations stay distinct. See the profile transition in the [TrustAsia report](trust-asia.md).

## 4. Retrieval, preservation, and validation

Test 1 asks the certificate manager to preserve the artifact bytes and the metadata. That is a storage requirement. Checking the inclusion proof and the cosignatures is a relying-party requirement, in Sections 4.3 and 7.2 of the MTC draft.

| Operation | What CertLocker did in this test |
| --- | --- |
| Retrieval | Downloaded the standalone certificate from the order, then the landmark-relative certificate after the wait. |
| Byte preservation | Stored each response as its own certificate record. The landmark response did not overwrite the standalone record. |
| Metadata | Workflow history shows Certificate Issued, the wait, Landmark Issued, and Complete, with links from the workflow to both certificate records. |
| Handshake probe | Trust opens a TLS handshake against the endpoint and determines whether that handshake will support an MTC. When it will, Trust serves the MTC. The probe then compares the served thumbprint with the thumbprint stored in Trust. |
| Inclusion proof and cosignatures | Retained inside the stored certificate so a relying party can evaluate them. The manager result is the preserved artifact. Chrome, or another MTC-capable relying party, performs the cryptographic check. |

Section 5.1 of the deployment draft separates inventory from what a live handshake presents. A workflow in the Complete state means both artifacts are in inventory. Trust is the check against the handshake: it probes, it serves the MTC when that handshake will support one, and it compares thumbprints to confirm the served certificate is the one Trust stored.

## 5. Installation

Installation is per record.

**Standalone.** At Certificate Issued the standard certificate and its private key are the credential for the name. Delivery is the same path CertLocker already uses for an ACME certificate: write the key and the certificate bytes to the target, reload the service, and probe the listener. The certificate bytes are the ACME response, written through unchanged. The landmark wait is independent of this write. Section 9.1 says a client should not block deployment of the available certificate on an optional alternate.

**Landmark-relative.** At Landmark Issued the second certificate is delivered as an additional credential for the same name. The standalone credential stays. Section 8 of the MTC draft is the selection rule for a server that holds both: the landmark-relative certificate when the ClientHello indicates the landmark, and the standalone certificate when it indicates the CA.

**Automation.** Both records are installed by the same agent-free mechanism, described in [Conquering the Last Mile](https://certlocker.io/blog/conquering-the-last-mile/). The remote machine runs SSH. CertLocker does not install an agent on it. A short script names the files to write and the commands that activate them. `install` copies a certificate or key over SFTP, `exec` runs one command over SSH, and `audit` records the result. Each `exec` is a new SSH session. The automation is attached to the order and run against the selected record, so the standalone certificate and the landmark-relative certificate each keep their own bytes. A host-installed nginx service looks like this:

```
install certificate.chain to /etc/nginx/ssl/fullchain.pem mode 644
install certificate.privateKey to /etc/nginx/ssl/privkey.pem mode 600
exec sudo nginx -t
exec sudo systemctl reload nginx
audit true Deployed certificate.name to host.name
```

The paths and the reload command change with the service. Apache, HAProxy, and IIS through Windows OpenSSH use the same language. Access is a scoped host token, and the route may pass through a bastion, as long as the target is running SSH. One automation installs the same certificate on every selected host, so a name served by many machines is a multi-node rollout of that one script. Trust then probes each host and compares the served thumbprint with the thumbprint it stored.

**Trust.** [Trust](https://trust.certlocker.io) is the CertLocker application that deals with the handshake after the certificates are stored. [mtcdemo.trustasia.cn](https://mtcdemo.trustasia.cn/) is the endpoint that already handles Merkle Tree Certificates at the TLS level: one handshake, one selected certificate, as described in the [TrustAsia report](trust-asia.md). Trust probes that handshake and determines whether it will support an MTC. When the handshake will support an MTC, Trust serves the MTC. When it will not, the handshake stays on the classical certificate.

The probe also checks that installation is correct. For each stored certificate, including the standalone record and the landmark-relative record, Trust compares the thumbprint it holds with the thumbprint the endpoint is serving. A match means the certificate on the listener is the certificate in Trust. Trust warns when a certificate is approaching expiry. By default it renews that certificate automatically once 75% of its lifespan has elapsed, installs the replacement, and probes again.

The August public record shows the storage side of this sequence: both records exist, the standalone record predates the landmark record, and the workflow completes only once they are linked. A per-transition reload log for a named proxy is not part of that published record. The handshake check that corresponds to Section 5.1 is Trust's probe, using the thumbprint comparison above.

## 6. Interoperability findings

| Finding | Consequence for an ACME client or certificate manager |
| --- | --- |
| The standalone certificate and the landmark-relative certificate arrive at different times, on one order. | Model two stored artifacts and one issuance. Surface a state between them. The August names are Certificate Issued, the landmark wait, Landmark Issued, and Complete. |
| The landmark wait can outlive the moment the standalone certificate is usable. | Persist the standalone record as soon as it arrives, and keep polling the landmark URL. For a new deployment, Section 9.1 specifies installing the standalone certificate while that poll continues. On renewal, the client may wait for the landmark only while the certificates already in service remain valid. |
| Profile names on this directory changed after the run. | The August request used `mtc-47d`. On 18 November 2026 a new order with that name produces `invalidProfile`. Refresh `meta.profiles` before `newOrder`. Keep the original profile string on the stored records. Details are in the [TrustAsia report](trust-asia.md). |
| `renewalInfo` is on the directory, and the landmark has its own `Retry-After`. | Poll the landmark URL on the landmark's schedule. That wait is the second certificate of the current order. Trust's renewal is a later order: by default it runs at 75% of the certificate's lifespan. |
| Unknown directory members are present. | Ignore them. The staging directory includes one, documented in the TrustAsia report. |
| Two generations of 47-day profile use different CAs and logs. | A stored `mtc-47d` certificate and a new `mtc-47d-07` certificate are separate inventory items. Installing the new one does not retire the old one's trust-anchor configuration by profile name alone. |

## 7. Reproduction and limitations

### August run, as published

1. Point an ACME client at `https://acme.mtc-staging.trustasia.com/acme/v2/directory`.
2. Create an order for a name the tester controls. The published order used `mtc.certlocker.dev` and `"profile": "mtc-47d"`.
3. Complete domain control validation and finalization.
4. Store the body of the order `certificate` URL as the standalone record. Record the `Link` headers with it.
5. Poll the landmark URL until it returns a certificate. Store that body as the second record. Link both records to the order.
6. Install the standalone certificate without waiting for step 5 to finish. Install the landmark certificate beside it when step 5 returns.
7. From Trust, probe the listener's TLS handshake. When the handshake will support an MTC, serve the MTC. Compare the thumbprint being served with the thumbprint stored in Trust.

Repeating step 2 today requires a profile the directory advertises. On 18 November 2026 those names are `mtc-7d-07` and `mtc-47d-07`. Fetch the directory first and use that set.

### What this repository can show today

The published evidence is the workflow outcome and the staging directory:

- identifier `mtc.certlocker.dev`
- profile `mtc-47d`
- states Certificate Issued, landmark wait, Landmark Issued, Complete
- two linked certificate records
- directory and profile page fetched 18 November 2026, quoted in the [TrustAsia report](trust-asia.md)
- the `newOrder` request and the order object
- the standalone response, including `Link` and `Retry-After`
- the landmark response
- the stored PEM for each record, with account identifiers removed
- the probe observation for the listener that received the install

### Further testing

| Deployment-draft test | Status for this client |
| --- | --- |
| Test 1, enroll and preserve the standalone artifact, and also preserve the landmark-relative artifact from the same order | Tested in August on the staging CA. Byte samples are still to be added to this repository. |
| Test 1, live confirmation that the endpoint presents the stored certificate | Trust probes the handshake, serves the MTC when the handshake supports it, and compares the served thumbprint with the thumbprint stored in Trust. A captured probe result for `mtc.certlocker.dev` is still to be added. |
| Tests 2–4 and 10, Trust Anchor Identifier selection during the handshake | Exercised by [mtcdemo.trustasia.cn](https://mtcdemo.trustasia.cn/), which selects the certificate inside TLS. Trust probes that handshake and serves the MTC when the handshake will support one. A client that omits `trust_anchors` receives the classical certificate from the same host. |
| Test 5, renewal | Trust renews the certificate at 75% of its lifespan by default, then the probe checks the served thumbprint. Trust also warns when a certificate is approaching expiry. |
| Tests 6, 7, and 12, revocation, emergency withdrawal, commercial support, failure recovery | Further testing. |

## 8. References

- MTC draft, Section 9: <https://www.ietf.org/archive/id/draft-ietf-plants-merkle-tree-certs-08.html#section-9>
- Deployment draft, Test 1: <https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01#section-11>
- August workflow: <https://certlocker.io/blog/wake-up-200-day-certificates-mtcs/>
- Trust: <https://trust.certlocker.io>
- Installation automation: <https://certlocker.io/blog/conquering-the-last-mile/>
- Staging directory: <https://acme.mtc-staging.trustasia.com/acme/v2/directory>
- Contact: <mailto:mtc@certlocker.io>
