# TLS Trust Anchor Negotiation

This document is a high-level overview and [explainer](https://tag.w3.org/explainers/) for [TLS Trust Anchor Identifiers](https://davidben.github.io/tls-trust-expressions/draft-beck-tls-trust-anchor-ids.html) and [TLS Trust Expressions](https://davidben.github.io/tls-trust-expressions/draft-davidben-tls-trust-expr.html). While they differ in approach, both mechanisms aim to allow the server to select certificates based on the client's trusted CAs. We refer to overall problem being solved as "trust anchor negotiation".

The two drafts apply to both TLS server certificates, where the server authenticates to the client, and TLS client certificates, where the client authenticates to the server. The two cases use similar protocol mechanisms, but their PKIs are often structured differently. As most of the motivating scenarios are drawn from server certificate deployments, this explainer primarily focuses on those roles, but most of it applies analogously to client certificates.

## Authors

* David Benjamin
* Devon O'Brien
* Bob Beck

## Participate

* https://github.com/davidben/tls-trust-expressions/issues
* https://www.ietf.org/mailman/listinfo/tls

## Introduction

[TLS](https://www.rfc-editor.org/rfc/rfc8446) uses [X.509 certificates](https://www.rfc-editor.org/rfc/rfc5280) to associate the authenticating party's, or *subscriber's*, TLS key with its application identifiers, such as DNS names. These associations are signed by some certificate authority (CA). The peer, or *relying party*, curates a set of CAs that are trusted to only sign correct associations, which allows it to rely on the TLS to authenticate application identifiers. Typically the subscriber is the server and the relying party is the client.

A single subscriber may need to interoperate with relying parties that trust different sets of CAs. TLS 1.3 defines the [`certificate_authorities` extension](https://www.rfc-editor.org/rfc/rfc8446#section-4.2.4) to accommodate this. It allows the subscriber to provision multiple certificates and select the one that will allow the relying party to accept its TLS key. This is analogous to parameter negotiation elsewhere in TLS.

However, `certificate_authorities`'s size is impractical for some applications. Existing PKIs may have many CAs, and existing CAs may have long X.509 names. As of August 2023, the Mozilla CA Certificate Program {{MOZILLA-ROOTS}} contained 144 CAs, with an average name length of around 100 bytes. Such TLS deployments often do not use trust anchor negotiation at all.

Without a negotiation mechanism, the subscriber must obtain a single certificate that simultaneously satisfies all relying parties. This is challenging when relying parties are diverse. PKI transitions, including those necessary for user security, naturally lead to relying party diversity, so the result is that service availability conflicts with security and overall PKI evolution:

* For a subscriber to use a CA in its single certificate, all supported relying parties must trust the CA. PKI transitions then become difficult when subscribers support older, unupdated relying parties. This impacts both new keys from existing CA operators and new CA operators.

* When a relying party must update its policies to meet new security requirements, it adds to relying party diversity and the challenges that subscribers and CAs face. The relying party must then choose between compromising on user security or burdening the rest of the ecosystem, potentially impacting availability in the process.

To address this, the two proposals extend the `certificate_authorities` extension to reduce the size cost, supporting flexible and robust PKIs for more applications. The proposals also define a supporting [ACME](https://www.rfc-editor.org/rfc/rfc8555.html) extension to help subscribers provision multiple certificates.

## Goals

At a high level, the goal for these proposals is to *enable TLS servers to handle diversity in client trust, so that server availability does not conflict with PKI security and evolution*.

In doing so, we aim to:

* Enable PKIs to evolve as needed for user security, in a timely manner and without conflicting with availability.
* Minimize burden to server operators, particularly avoiding ongoing manual work. Most ongoing decisions should instead come from TLS software, ACME client software, and ACME servers.
* Minimize bandwidth cost to the TLS handshake.

Matching the standard pattern for other TLS parameters, we target a deployment model where subscribers can configure multiple TLS certificates, with TLS software automatically sending the right one on each connection. To aid in this, CAs can transparently provision subscribers with multiple TLS certificates with ACME or another automated protocol.

We discuss these motivations in more depth below.

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

### Minimizing Server Operator Burden

In reasoning through PKI deployment strategies, one of our primary goals is to
minimize server operator burden. Where possible, permit server operators to make
simpler, high-level decisions (e.g. which ACME endpoint(s) cover the clients
they need), with automation handling the rest.

This is important because there are many HTTPS servers on the web, operated by
different parties. While some strategies, such as fingerprinting or manually
collecting an optimal set of intermediates, can be done by a small number of
large hosting providers, they are impractical for a more diverse set of HTTPS
servers. Avoiding this overhead helps both support the current server diversity
and to maintain it going forward.

This means:

* **Prefer fewer mechanisms with broad coverage, over more mechanisms that cover
  individual scenarios.** While one cannot solve every problem at once, broad
  coverage minimizes software updates needed. If one mechanism addresses
  multiple similar PKI scenarios, it can be implemented in server software once,
  then used whenever each of these scenarios arise. In contrast, inventing a
  bespoke mechanism for each kind of PKI transition resets the protocol
  deployment clock each time.

* **Prefer unambiguous, systematic server decisions.** While an individual
  deployment might observe that, e.g., all of their newer clients support some
  CA, or use more complex fingerprinting mechanisms, this requires server
  operators to make risk assessments about imprecise properties of their client
  base. It cannot be generalized and automated. In contrast, a mechanism that
  directly negotiates the selection critera can be implemented in the TLS
  library, with HTTPS software and server operators only providing the inputs.
  See "Considered Alternatives" for further discussion of the implications of
  imprecise strategies.

* **Shift work from server operators to CAs and root programs.** CAs are
  well-positioned to react to changes in root program requirements. They mint
  the certificates and maintain relationships with both server operators and the
  root programs whose requirements they follow. User security improvements are
  dramatically easier to deploy when only CAs and root programs need to act.

* **Minimize additional risk during incident response.** As root program
  requirements change over time, servers may need to change in response, e.g.
  migrating to a new CA when an old one is distrusted. Such changes, however,
  are inherently risky to the server operator, because an old client may be
  depending on the old configuration. This makes incident response much riskier
  and more difficult for the ecosystem. Robust certificate negotiation relieves
  this tension.

## Overview

[RFC 8446](https://www.rfc-editor.org/rfc/rfc8446.html) defines the [Certificate Authorities](https://www.rfc-editor.org/rfc/rfc8446.html#section-4.2.4) extension which allows clients (and servers) to send the list of supported CAs as a list of X.509 names. This mechanism is sufficient to negotiate trust anchors, but is an inefficient use of bandwidth. Web PKIs often have over 100 CAs, whose names average 100 bytes each.

The two protocols use different approaches to address these bandwidth concerns, with different tradeoffs.

### Trust Expressions

Trust Expressions compresses the certificate authorities list be referencing "trust stores", maintained by root programs. Clients send "trust expressions", which reference these trust stores by name and version, and optionally subset them down by excluding portions of them. The full CA list is then the union of the trust expressions sent by the client.

When CAs issue certificates to servers, they include "trust store inclusion" metadata, which contain sufficient information for servers to evaluate trust expressions and determine whether their candidate certificates match the trust expression.

To support this compression scheme, participating CAs and root programs must coordinate to ensure all the information configured at clients and servers is consistent. This coordination is in the form of a "trust store manifest" structure, published by the root program and periodically fetched by the CA. The protocol additionally must accommodate a "version skew" problem where the client references a newer trust store version than the server has available. Most of the protocol's complexity comes from these two parts.

### Trust Anchor Identifiers

The TLS extension in Trust Anchor Identifiers consists of three parts:

1. To mitigate large X.509 names, we introduce short "trust anchor identifiers" to uniquely identify each participating CA, an estimated 5 bytes per CA. These are sent in a `trust_anchors` extension, which is analogous to `certificate_authorities`, but uses these much shorter identifiers.

2. At 5 bytes per CA, bandwidth and privacy constraints may still prevent a client from enumerating its CA list. We thus define an extension to [HTTPS/SVCB](https://www.rfc-editor.org/rfc/rfc9460.html) DNS records for servers to list their supported trust anchors by ID. The client fetches this, selects the desired option, and requests it in `trust_anchors`.

3. To accommodate stale DNS records, or cases where the DNS mechanism does not work, servers send a list of available trust anchors in the TLS handshake. If the client rejects the certificate, it can retry with more definitive information from the server and send a more accurate `trust_anchors` request.

### Comparison

Although they aim to solve the same problem, the two proposals work in very different ways:

* Trust Anchor IDs are usable by more kinds of PKIs. Trust Expressions express trust anchor lists relative to named “trust stores”, maintained by root programs. Arbitrary lists may not be easily expressible. Trust Anchor IDs does not have this restriction.

* When used with large trust stores, the retry mechanism in Trust Anchor IDs requires a new connection. In most applications, this must be implemented outside the TLS stack, so more components must be changed and redeployed. In deployments that are limited by client changes, this may be a more difficult transition. (The draft also sketches out an alternate retry scheme that avoids this.)

* Trust Expressions works with static server configuration. An ideal Trust Anchor IDs deployment requires automation to synchronize a server’s DNS and TLS configuration. [draft-ietf-tls-wkech](https://datatracker.ietf.org/doc/draft-ietf-tls-wkech/) could be a starting point for this automation. In deployments that are limited by server changes, this may be a more difficult transition.

* Trust Expressions require that CAs continually fetch information from manifests that are published by root programs, while Trust Anchor IDs rely only on static pre-assigned trust anchor identifiers.

* Trust Anchor IDs, when trust anchors are conditionally sent, have different fingerprinting properties. See [Privacy Considerations](https://davidben.github.io/tls-trust-expressions/draft-beck-tls-trust-anchor-ids.html#name-privacy-considerations).

* Trust Anchor IDs can only express large client trust stores (for server certificates), not large server trust stores. Large trust stores rely on the retry mechanism described in [the draft](https://davidben.github.io/tls-trust-expressions/draft-beck-tls-trust-anchor-ids.html#name-retry-mechanism), which is not available to client certificates.

The two mechanisms can be deployed together. A subscriber can have metadata for both mechanisms available, and a relying party can advertise both.

## Key Scenarios

The following section is an overview of scenarios where trust anchor negotiation is helpful. For more details, see the
[draft specification](https://davidben.github.io/tls-trust-expressions/draft-davidben-tls-trust-expr.html#name-use-cases) and [this more detailed discussion](pki-transition-strategies.md).

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

### Public Key Pinning

To reduce the risk of attacks from misissued certificates, TLS clients sometimes employ [public key pinning](https://www.rfc-editor.org/rfc/rfc7469.html). This enforces that one of some set of public keys appear in the final certificate path. This effectively reduces a client's trust anchor list to a subset.

As above, such a client decision constrains how the server evolves over time. As other clients in the PKI evolve, the pinning clients limit the server to satisfy both the pinning constraint and newer constraints in the PKI. This can lead to conflicts if, for example, the pinned CA is distrusted by a newer client. The server is then forced to either break the pinning clients, or break the newer ones.

Trust anchor negotiation relieves this conflict. The server can select a certificate from the pinned CA with the pinning client, and another CA with newer clients. The server must decide whether to continue to obtaining certificates from pinned CA, or drop support for those pinning clients, but negotiation decouples this decision from the newer clients.

For this to work, the pinning client must accurately negotiate its reduced trust anchor list. TLS Trust Anchor Identifiers can efficiently handle this. TLS Trust Expressions is not as well-suited, as it cannot as efficiently represent arbitrary trust store subsets. However, the same supporting server infrastructure can be used with the existing `certificate_authorities` extension. While size often makes `certificate_authorities` impractical, a pinning client's reduced trust anchor list is small.

## Server Software Changes

Server software will need to be modified to support Trust Expressions, or Trust Anchor Identifiers. We expect this to look something like:

Servers are configured to obtain multiple certificate paths with associated metadata for clients that do present a trust expression, possibly from more than one CA. The CA maintains relationships with root programs, so it can populate this metadata. Ideally, these come from some form of automated certificate distribution mechanism, such as ACME, so that the server operator only needs to specify, e.g., an ACME URL and the rest occurs automatically.

Trust Anchor Identifiers adds an additional server component, which is to [publish some information in the DNS](https://davidben.github.io/tls-trust-expressions/draft-beck-tls-trust-anchor-ids.html#name-dns-service-parameter). While the protocol will function without it, the DNS mechanism enables optimal performance. This would ideally also be automated, e.g. as in [draft-ietf-tls-wkech](https://datatracker.ietf.org/doc/draft-ietf-tls-wkech/).

With a collection of candidate certificate paths and associated metadata, the server software automatically selects one to send. For each candidate, the software matches the ClientHello message against the metadata to determine if the client trusts it, along with other TLS criteria such as ECDSA vs RSA. If there are multiple matches, server software chooses based on its own criteria (such as the size of matching path, performance characteristics, etc.).

If the negotiation mechanism is either not supported or did not match a candidate, servers should behave as they do today, either by serving a single certificate to all clients, or relying on fingerprinting signals to choose among a set of credentials (e.g. ECDSA vs. RSA based on inferred client support). The existing behaviour remains unchanged.

## Certificate Provisioning

The server software changes above take, as input, a collection of certificate paths with associated metadata. In principle, these may come from any source, even manual configuration. However, Trust Expressions and Trust Anchor Identifiers are designed with automation in mind, with the aim of reducing server operator burden.

When using an automated issuance protocol, such as ACME, we intend that the issuance protocol be extended so that a single ACME endpoint can return *multiple* certificate paths for the server's requested key and identity. As CAs already maintain relationships with root programs, they are well-positioned to update the kinds of certificates they provision in response to PKI changes. Automation allows subscribers to benefit from this without manual reconfiguration.

For ACME, the draft currently proposes a minimal change to existing mechanisms, where a new MIME type provides the metadata alongside each certificate path, and the [existing alternate certificate mechanism](https://www.rfc-editor.org/rfc/rfc8555.html#section-7.4.2) allows provisioning multiple alternate certificates in response to one request.

Where an individual ACME server does not cover all that some server operator needs, the server operator can also combine the outputs of multiple ACME servers, with the server software automatically selecting from the combined set.

## Considered Alternatives

See also this [more detailed discussion](pki-transition-strategies.md).

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

For a more detailed discussion on path-based approaches, see [more detailed discussion](pki-transition-strategies.md).

### Fingerprinting

Servers sometimes resort to TLS fingerprinting to decide which certificate to
send. However, this is ultimately just a heuristic. It boils down to assuming
that one TLS change (e.g. adding some cipher suite) was, in practice, correlated
with an unrelated TLS change (trusting some new CA).

While such correlations do exist in practice, this solution is inherently
unreliable and usually requires writing custom service-specific logic. It is
only viable for large server operators, who can continually update their
heuristics based on what they observe in the ecosystem.

TLS Trust Expressions and TLS Trust Anchor Identifiers each avoid this
ambiguity, and thus can be evaluated automatically in TLS software.

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

### Global Timestamp or Version

Rather than trust-store-specific version numbers, one could imagine a
Web-PKI-only solution, where the client sends a trust store timestamp or,
equivalently, a global trust store version number, without reference to
different trust stores.

In addition to being Web-PKI-specific, this approach already does not match the
reality of the Web PKI today. A global timestamp or version number presumes
that all root programs make the same changes in, if not the same time, the
same order. However, the root programs which make up the Web PKI do not
coordinate on decisions. Different root programs may make different
decisions at different times, for a variety of reasons, ranging from:

* Different prioritization and resourcing decisions
* New CA applications being processed at different times
* Different decisions on how to remediate untrustworthy CAs
* Different judgements on what policies best meet their respective users' needs

For example, some root programs adopted a
[Certificate Transparency](https://www.rfc-editor.org/rfc/rfc6962.html)
requirement in as early as 2018, while others have not, as of writing in 2024.

This mismatch has direct consequences for the post-quantum transition. As many
post-quantum CAs will likely come online in a short time, it is implausible
that every root program will qualify the same CAs in the order. Until the
whole process converges, it will not be possibly for early adopters to deploy
post-quantum authentication. This is especially problematic given the
challenges posed by large post-quantum signatures. It will take experimentation
and iteration to design a good solution.

Additionally, such a solution would also still require most of the machinery in
this draft, so that server software can be correctly configured with
certificates across different generations of trust store. Populations which
*are* sufficiently coordinated for a global version number can simply use a
shared trust store with TLS trust expressions, or configure same trust anchors
with TLS trust anchor identifiers.

### Signature Algorithm Negotiation

TLS already defines the `signature_algorithms` and `signature_algorithms_cert`
extensions, which allow servers to select certificates by supported algorithms.
While this does allow differentiating classical and post-quantum clients, it
only works once.

Ultimately, knowing that a client can verify, say, ML-DSA signatures, does not
imply that the client trusts a *particular* ML-DSA key. Relying solely on these
extensions for a post-quantum transition presumes that all post-quantum CAs are
added at the same time and fixed. That is, it repeats the root ubiquity
problem, but worse:

The RSA roots in the Web PKI developed slowly over time, during a period where
the Web and HTTPS were not as large as they are now. There were fewer root
programs, fewer websites, so it was less crucial for the ecosystem to be robust.
Once things settled, they largely ossified, with security improvements to the
PKI becoming much more challenging. Indeed, the previous algorithm transition,
ECDSA, in the PKI has already faced challenges. ECDSA support in clients is now
common, yet servers often must deploy mixed RSA/ECDSA chains for compatibility.

While a partial ECDSA transition merely wastes bandwidth, post-quantum will
require a full transition to be secure. But the Web and HTTPS is now even larger
and more important than when we began the ECDSA transition.
`signature_algorithms` and `signature_algorithms_cert` are not robust enough on
their own for a smooth post-quantum transition.


## Abridged Certificates

Some PKI transitions can be managed, imperfectly and with significant costs, by a combination of cross-signs and intermediate compression. The [Abridged Certificates](https://datatracker.ietf.org/doc/draft-ietf-tls-cert-abridge/) draft describes a mechanism for compressing pre-shipped intermediate certificates.

In its current form, [draft-01](https://www.ietf.org/archive/id/draft-ietf-tls-cert-abridge-01.html), Abridged Certificates does not address these scenarios. It allocates a single codepoint for a fixed snapshot of intermediates in the PKI. By doing so, it cannot compress any intermediate that postdates the snapshot. This means it cannot handle future transitions. Moreover, it gives a size advantage to incumbents in the Web PKI, thus discouraging any further improvements in user security. It also cannot handle non-Web-PKI uses like private PKIs.


## Alternate Intermediate Compression

A hypothetical alternate intermediate elision scheme could be designed which avoids these drawbacks, but it would need to be much more complex to accommodate version skew, and the operational challenges of provisioning servers with the right metadata to evaluate information from newer and newer clients. The Trust Expressions draft is an example of how to address these challenges, notably CertificatePropertyList, the ACME extension, the versioned trust stores, and the cross-version invariants managed by `excluded_labels` and expiry.

Such a hypothetical scheme would be more applicable than Abridged Certificates, but imposes numerous costs, depending on the cross-signing scheme, including from delays to in security incident response, higher bandwidth usage in high-fanout scenarios, and the need to ship intermediates to clients that aren’t expected to need them. See also this [more detailed discussion](pki-transition-strategies.md).


## References and Acknowledgements

The authors thank Nick Harper, Sophie Schmieg, and Emily Stark for many valuable discussions and insights which led to this design, as well as review of early iterations of the draft specification.
