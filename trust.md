# Trust — creating an MTC order

Trust is the CertLocker application. This file is the operator path through that application to create a Merkle Tree Certificate order. Issuance, the staging directory, and the profile catalogue are in the [TrustAsia report](trust-asia.md). Storage, the two certificate records, and installation are in the [CertLocker report](certlocker.md). The repository index is the [README](README.md).

An MTC order is an ordinary ACME workflow whose directory profile names an MTC product. Trust does not have a separate MTC screen. The order is created from **ACME → Outbound orders**.

## 1. Before the order

The order is placed by an ACME client that already exists in Trust.

1. Open **ACME → ACME Clients**.
2. Open the client that points at the MTC directory. For the staging interoperability environment that directory is `https://acme.mtc-staging.trustasia.com/acme/v2/directory`.
3. On the client detail, use **Update Profiles**. Trust refreshes `meta.profiles` from that directory and stores the names on the client.
4. Confirm the client is not suspended. A suspended client cannot be selected for a new order.

On 18 November 2026 that directory advertised `mtc-7d-07` and `mtc-47d-07`. Those are the names the profile list should show for a new order. The August name `mtc-47d` is no longer advertised; selecting it on a new order produces `invalidProfile`. Retained profiles still serve orders that were already accepted. Details are in the [TrustAsia report](trust-asia.md).

`mtc-7d-07` is the 7-day, revocation-exempt product. `mtc-47d-07` is the 47-day product that carries revocation information. Each uses its own CA and log.

## 2. Open the order form

1. Open **ACME → Outbound orders**.
2. Choose **Create**.

Trust opens **Create ACME order**. The form has three steps: **Client**, **Certificates**, and **Automation**. **Next** and **Back** move between them. **Create order** is available on the last step and submits `POST /acme/workflow-create`. When the response includes a workflow id, Trust opens that workflow.

## 3. Step 1 — Client

| Field | What to set for an MTC order |
| --- | --- |
| ACME client | The client from Section 1. Required. Selecting it loads that client's directory profiles. |
| Profile (optional) | An advertised MTC profile, `mtc-7d-07` or `mtc-47d-07`. The control appears only after a client with profiles is selected. **No profile (default)** does not ask the CA for an MTC product. |
| Challenge type | HTTP-01, DNS-01, or DNS-PERSIST-01. Required. This is how each domain authorisation is validated. |
| DNS client (optional) | Shown for DNS-01 and DNS-PERSIST-01. The DNS client creates the TXT records. It cannot be combined with a challenge automation on step 3. |

**Next** requires an ACME client.

## 4. Step 2 — Certificates

Leave both certificate selectors on their new-certificate choices. Trust then creates the standalone record when issuance completes, and the landmark-relative record when the CA issues it. Linking an existing record is for an order that should attach to a certificate already in inventory.

| Field | What to set for a new MTC order |
| --- | --- |
| Standalone certificate (optional) | **New standalone certificate**. |
| Landmark certificate (optional) | **New landmark certificate**. |
| Certificate algorithm | RSA, P-256, or P-384. Default is P-256. Trust uses this when it generates the key pair. |
| Auto renew | On by default. Trust renews at 75% of the certificate lifespan. |
| Domains (one per line) | The DNS names for this order. Each line becomes an authorisation. Required. |
| Expiry | Optional. **Not before** and **Not after** must be today or later. |
| Private Key | Optional. Expand the panel to supply a PKCS#8 key, or leave it blank so Trust generates one during issuance. |

**Next** requires at least one valid domain.

## 5. Step 3 — Automation

Every control on this step is optional. The order can be created with none of them set.

| Field | When it runs |
| --- | --- |
| Challenge automation (optional) | After the challenge is ready, for any challenge type. Cannot be selected together with a DNS client from step 1. |
| Standalone automation (optional) | After the standalone certificate is issued. |
| Landmark automation (optional) | After the landmark-relative certificate is issued. |
| Create access token | When checked, Trust creates a certificate access token for each certificate this order generates. |

**Create order** starts the workflow.

## 6. What the workflow shows

The new row appears on **Outbound orders**. Open it to follow the timeline. An open workflow updates as it changes state. Tiles are the workflow history: the state and when Trust recorded it.

The states, in order, are:

1. Order Scheduled
2. Order Pending
3. Order Submitted
4. Challenges Waiting
5. Challenges Triggered
6. Challenges Verified
7. Issue Final Order
8. Wait for Order
9. Certificate Issued
10. Wait for Landmark
11. Landmark Issued
12. Complete

For an MTC order the last four states are the part that differs from a classical certificate:

- **Certificate Issued.** Trust has stored the standalone certificate. The tile links to that record. The standalone certificate can be installed while the landmark is still pending.
- **Wait for Landmark.** The CA has not yet allocated the landmark. When the ACME server returned `Retry-After`, the tile shows **Retry after** until that time. Trust keeps the standalone record and keeps polling.
- **Landmark Issued.** Trust has stored the landmark-relative certificate as a second record, linked to the same order and the same name. The tile links to that record. The standalone record stays.
- **Complete.** Both records exist and are linked.

The detail also shows the ACME client, the DNS client when one was used, and the profile name and description when a profile was selected.

**Wait for Order** and **Wait for Landmark** are different waits. Wait for Order is the delay before the standalone certificate. Wait for Landmark is the delay before the second certificate of the same order. Renewal is a later order, driven by Auto renew, and is not this landmark wait.

On failure, the workflow footer shows the message, the error, and any ACME problem subproblems under the last tile. **Cancel order** stops a workflow that is still in flight. **Delete order** removes the workflow record.

## 7. Where the certificates go

Certificate Issued and Landmark Issued are two inventory records, not two orders. Replacing the standalone bytes with the landmark bytes would drop the cosignatures a relying party still needs when it does not have the landmark. Trust leaves both records in place and links them.

The profile string is stored with the order. A certificate issued under `mtc-47d` and a certificate issued under `mtc-47d-07` are separate inventory items, from different CAs and logs.

Installation of each record, and the handshake probe Trust runs after install, are described in the [CertLocker report](certlocker.md).
