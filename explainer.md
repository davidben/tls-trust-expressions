# TLS Trust Expressions

This document is a high-level overview and [explainer](https://tag.w3.org/explainers/) for [TLS Trust Expressions](https://davidben.github.io/tls-trust-expressions/draft-davidben-tls-trust-expr.html).

## Authors

* David Benjamin
* Devon O'Brien
* Bob Beck

## Participate

* https://github.com/davidben/tls-trust-expressions/issues
* https://www.ietf.org/mailman/listinfo/tls

## Introduction

[TLS](https://www.rfc-editor.org/rfc/rfc8446) endpoints typically authenticate using [X.509 certificates](https://www.rfc-editor.org/rfc/rfc5280). These are used as assertions by a certification authority (CA) that associate some TLS key with some DNS name or other identifier. If the peer trusts the CA, it will accept this association. The authenticating party (usually the server) is known as the *subscriber* and the peer (usually the client) is the *relying party*.

Today, subscribers typically provision a single certificate for all supported relying parties, because relying parties do not communicate which CAs are trusted. We call this a *single-certificate deployment model*. In this model, the single certificate must simultaneously satisfy all relying parties.

This constraint imposes costs on the ecosystem as PKIs evolve over time. The older the relying party, the more its requirements may have diverged from newer ones, making it increasingly difficult for subscribers to support both. This translates to analogous costs for CAs and relying parties:

* For a new CA to be usable by subscribers, it must be trusted by all relying parties. This is particularly challenging for older, unupdatable relying parties. Existing CAs face similar challenges when rotating or deploying new keys.

* When a relying party must update its policies to meet new security requirements, it must choose between compromising on user security or imposing a significant burden on subscribers that still support older relying parties.

TLS trust expressions aims to remove this constraint, by enabling a *multi-certificate deployment model*. Subscribers are instead provisioned with multiple certificates and automatically select the correct one to use with each relying party. This allows a single subscriber to use different certificates for different relying parties, including older and newer ones.

There are three parts to understanding this proposal:

1. The multi-certificate model itself
2. A TLS extension for relying parties to succinctly communicate trusted CAs
3. An [ACME](https://www.rfc-editor.org/rfc/rfc8555.html) extension for provisioning multiple certificates

Together, these aim to support a more flexible and robust Public Key Infrastructure (PKI).

## Goals

At a high level, the goal for TLS Trust Expressions is to enable a multi-certificate deployment model for TLS, particularly as used in HTTPS on the web.

Our goals in doing so are:

* PKIs can evolve over time, to meet user security needs, without conflicting with availability.
* Subscribers can configure multiple TLS certificates, with TLS software automatically sending the right one on each connection.
* CAs, via automated protocols like ACME, can transparently provision subscribers with multiple TLS certificates.
* As much as possible, minimize manual changes by server operators. Most ongoing decisions should instead come from TLS software, ACME client software, and ACME servers.
* Minimal bandwidth cost to the TLS handshake.

We discuss the motivations for a multi-certificate deployment model in more
depth below.

### Why Multiple Certificates?

PKIs need to evolve over time to meet user security needs. For example:

* CAs that add net value to the ecosystem may be added to relying parties
* Long-lived CA keys should be rotated to reduce risk
* Compromised or untrustworthy CAs are removed from relying parties

These kinds of changes inherently create divergence, if for no other reason than
out-of-date clients in the ecosystem. These clients could range from unupdatable
TV set-top boxes to some IoT device to an individual user's browser that could
not communicate with its update service.

Today, these old clients limit security improvements for other, unrelated
clients. Consider a TLS client making some trust change for user security. For
availability, TLS servers must have some way to satisfy it. Some server,
however, may also support an older client. If the server uses a single
certificate, that certificate is limited to the intersection of both clients.

As servers consider older and older clients, that intersection shrinks, causing
availability and security to conflict. At internet scales, the oldest clients
effectively last forever. It takes just one important server with one important
old client to jam everything, with user security paying the cost.

A multi-certificate deployment model removes this conflict. Servers can deliver
different certificate chains to different clients as needed. It is no longer
necessary that the certificate for a 10-year-old TV set-top box is the same as
the certificate for an evergreen web browser. This has some analogies elsewhere
in TLS and the web platform:

* TLS uses version negotiation and cipher suite negotiation, so that clients and
  servers can talk to a range of peers.

* Browsers independently decide when and whether to ship individual web APIs.
  Different browsers also ship and update at different cadences. Web developers
  use techniques like [progressive enhancement](https://developer.mozilla.org/en-US/docs/Glossary/Progressive_Enhancement)
  and [feature detection](https://developer.mozilla.org/en-US/docs/Learn/Tools_and_testing/Cross_browser_testing/Feature_detection)
  to support a range of browser capabilities.

Multiple certificates can additionally unlock size optimizations by skipping
unnecessary intermediate certificates and cross-signs in the common case. This
will be particularly valuable for the post-quantum transition, when signatures
become [much more expensive](https://pq-crystals.org/dilithium/). The
flexibility will also simplify the post-quantum transition itself, as different
post-quantum CAs may be added at different times.

## Overview

See the [draft specification](https://davidben.github.io/tls-trust-expressions/draft-davidben-tls-trust-expr.html#name-overview) for an overview of the protocol.

## Key Scenarios

The following section is an overview of scenarios where TLS Trust Expressions
are helpful. For more details, see the
[draft specification](https://davidben.github.io/tls-trust-expressions/draft-davidben-tls-trust-expr.html#name-use-cases).

### Key Rotation

In most X.509 deployments, a compromise of _any_ root CA's private key compromises the entire PKI. Yet key rotation in PKIs is rare. As of 2023, the oldest certificate in Chrome's and Mozilla's root stores dates to 1998. Key rotation is challenging in a single-certificate deployment model. As long as any older relying party requires the old root, subscribers cannot switch to the new root, which in turn means relying parties cannot distrust the old root, leaving them vulnerable.

In a multi-certificate deployment model, the CA simply starts issuing from both the old and new root. Certificate negotiation in the TLS software then transparently allows the subscriber to send the correct one to each relying party. This requires no configuration changes to the subscriber. The subscriber does not need to know why it received two certificates, only how to select between them.

### Adding CAs

Today, subscribers cannot use TLS certificates issued from a new root CA until all supported relying parties have been updated to trust the new root CA. This can take years or more. Some relying parties, such as IoT devices, may never receive trust store updates at all.

In a multi-certificate deployment model, subscribers can begin serving certificates from new root CAs without interrupting relying parties that depend on existing ones.

### Removing CAs

Subscribers in a single-certificate model are limited to CAs in the intersection of their supported relying parties. As newer relying parties remove untrusted CAs over time, the intersection with older relying parties shrinks. Moreover, the subscriber may not even know which CAs are in the intersection. Often, the only option is to try the new certificate and monitor errors. For subscribers that serve many diverse relying parties, this is a disruptive and risky process.

The multi-certificate model removes this constraint. If a subscriber's CA is distrusted, it can continue to use that CA, in addition to a newer one. This removes the risk that some older relying party required that CA and was incompatible with the new one.

### Other Root Transitions

The mechanisms in this document can aid PKI transitions beyond key rotation. For example, a CA operator may generate a postquantum root CA and use the mechanism in {{acme-extension}} to issue from the classical and postquantum roots concurrently. The subscriber will then, transparently and with no configuration change, serve both. As in {{key-rotation}}, newer relying parties can then remove the classical roots, while older relying parties continue to function.

This same procedure may also be used to transition between newer, more size-efficient signature algorithms, as they are developed.

[[TODO: There's one missing piece, which is that some servers may attempt to parse the signature algorithms out of the certificate chain. See https://github.com/davidben/tls-trust-expressions/issues/9 ]]

### Intermediate Elision

Today, root CAs typically issue shorter-lived intermediate certificates which, in turn, issue end-entity certificates. The long-lived root key is less exposed to attack, while the short-lived intermediate key can be more easily replaced without changes to relying parties. This operational improvement comes at a bandwidth cost: the TLS handshake includes an extra certificate. Post-quantum algorithms will further inflate this cost. A single [Dilithium3](https://pq-crystals.org/dilithium/) intermediate certificate uses 5,245 bytes in cryptographic material (public key and signature) alone.

The multi-certificate model reduces this cost. A CA operator could provide subscribers with two certificate paths: a longer path ending at a long-lived root and shorter path the other ending at a short-lived root. Relying parties would trust both the long-lived root and the most recent short-lived root. Up-to-date relying parties will match on the short-lived root and use less bandwidth. On mismatch, older relying parties will continue to work with the long-lived root. Subscribers are no longer limited to the lowest-common denominator.

### Conflicting Relying Party Requirements

A subscriber may need to support relying parties with different requirements. For example, in contexts where online revocation checks are expensive, unreliable, or privacy-sensitive, user security is best served by short-lived certificates. In other contexts, long-lived certificates may be more appropriate for, e.g., systems that are offline for long periods of time or have unreliable clocks.

A single-certificate deployment model forces subscribers to find a single certificate that meets all requirements. User security then suffers in all contexts, as the PKI may not quite meet anyone's needs. In a multi-certificate deployment model, different sets of requirements can simply use different root CAs.

### Backup Certificates

A subscriber may obtain certificate paths from multiple CAs for redundancy in the face of future CA compromises. If one CA is compromised and removed from newer relying parties, the TLS server software will transparently serve the other one.

## Considered Alternatives

### TLS `certificate_authorities` Extension

TLS already has a `certificate_authorities` extension to send the list of CAs, but it is impractical. Trust stores can be large, and the X.509 Name structure is inefficient. For example, as of August 2023, the [Mozilla CA Certificate Program](https://wiki.mozilla.org/CA/Included_Certificates) would encode 144 names totaling 14,457 bytes.

### Path Building

X.509 certificate chains have some limited degree of adaptability via
[path building](https://www.rfc-editor.org/rfc/rfc4158.html). For example, a
server may support clients that require an older and a newer CA if the older
CA cross-signs the newer CA. The server then sends this cross-sign along with
any other certificates needed to chain up to the newer CA. Clients that
trust the newer CA will ignore the cross-sign. Clients that trust the older CA
will use the cross-sign to chain up to the older CA. Further complicated PKI
topologies are possible, see
[figure 1](https://www.rfc-editor.org/rfc/rfc4158.html#page-9),
[figure 2](https://www.rfc-editor.org/rfc/rfc4158.html#page-10),
[figure 3](https://www.rfc-editor.org/rfc/rfc4158.html#page-11), and
[figure 4](https://www.rfc-editor.org/rfc/rfc4158.html#page-12) in
[RFC 4158](https://www.rfc-editor.org/rfc/rfc4158.html).

This solution has some drawbacks:

Without negotiation, the server must assume the lowest common denominator and
send all certificates. This wastes bandwidth, and will be even more expensive
with post-quantum cryptography, where every signature costs kilobytes. The
server may try to optimize for the common case and omit the cross-signs, relying
on clients to fetch the certificates with the
[authority information access](https://www.rfc-editor.org/rfc/rfc5280#section-4.2.2.1)
extension. However, this is unreliable, with older clients being less likely to
implement it. Requiring an extra fetch also costs round-trips, has privacy
implications, and may not work under certain network conditions, such as captive
portals.

Additionally, an individual server operator cannot unilaterally configure this.
The leaf certificate must be the same in all paths, so the server operator
cannot simply send two unrelated paths. The cross-signs must happen at the CA
level. These cross-signs apply to entire intermediate CAs, not just the
individual server’s certificate, which limits the cases when this may
be viable.

Finally, path-building is unreliably implemented. It is ultimately an
exponential search algorithm. Not all certificate verifiers implement it
robustly. Those that do require ad-hoc limits to avoid denial-of-service
vulnerabilities. As a result, relying on it, particularly in more complex
topologies, adds compatibility risk to servers.

Certificate negotiation avoids these problems and complements path-based
strategies. The server can send predictable, pre-built paths to the relying
party. It is also free to use unrelated paths, which reduces the need for
intermediate certificates and cross-signs. This is particularly valuable for
post-quantum cryptography's larger sizes.

### Fingerprinting

Servers sometimes resort to TLS fingerprinting to decide which certificate to
send. However, this is ultimately just a heuristic. It boils down to assuming
that one TLS change (e.g. adding some cipher suite) was, in practice, correlated
with an unrelated TLS change (trusting some new CA).

While such correlations do exist in practice, this solution is inherently
unreliable and usually requires writing custom service-specific logic. It is
only viable for large server operators, who can continually update their
heuristics based on what they observe in the ecosystem.

TLS Trust Expressions avoids this ambiguity, and thus can be evaluated
automatically in TLS software.

### Client-specific DNS Names

In some circumstances, it is possible to separate traffic by deploying services
for different clients at different DNS names. For example, a payment processor
might use a different subdomain for payment terminals from its browser-facing
web services. Using separate endpoints allows the operator to separately
configure the certificate and other TLS settings based on the needs of the two
client populations.

While this is often good practice, it only addresses very limited scenarios:

* The site operator needs to set this up ahead of time. Once, say, payment
  terminals have already been deployed using the same endpoint as browser-facing
  services, it is difficult to separate them after the fact.

* This strategy is only viable for clients where the DNS name is not
  user-visible. While a payment terminal's API endpoint is largely internal,
  users directly see and interact with the URLs and DNS names used for
  browser-facing web services. It would not be viable to use different DNS names
  for, say, older and newer browsers.

## References and Acknowledgements

The authors thank Nick Harper, Sophie Schmieg, and Emily Stark for many valuable discussions and insights which led to this design, as well as review of early iterations of the draft specification.
