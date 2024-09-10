# TLS Trust Anchor Negotiation

This document is a high-level overview and [explainer](https://tag.w3.org/explainers/) for the [TLS Trust Anchor Identifiers](https://davidben.github.io/tls-trust-expressions/draft-beck-tls-trust-anchor-ids.html) and [TLS Trust Expressions](https://davidben.github.io/tls-trust-expressions/draft-davidben-tls-trust-expr.html) drafts.

## Authors

* David Benjamin
* Devon O'Brien
* Bob Beck

## Participate

* https://github.com/davidben/tls-trust-expressions/issues
* https://www.ietf.org/mailman/listinfo/tls

## Introduction

The two drafts are different approaches to solving the same problem: **enable TLS endpoints to reliably and efficiently present certificates to peers that vary in supported trust anchors, particularly in larger PKIs like the Web PKI**.

This avoids a conflict between service availability and user security. As authentication requirements evolve to meet user security, the result is increased variance in the ecosystem. If TLS endpoints cannot reliably meet each supported peer's requirements (e.g. because no single certificate satisfies both the oldest and newest supported peers), connections will fail. Often, the result is user security is deprioritized in favor of avoiding any kind of breakage.

We approach this by following the standard TLS negotiation pattern. The existing TLS 1.3 [`certificate_authorities`](https://www.rfc-editor.org/rfc/rfc8446#section-4.2.4) extension allows endpoints to provision multiple certificates and select the appropriate one per connection. The two drafts extend this mechanism to reduce the size cost, making it usable for applications with larger PKIs. They also define a supporting ACME extension to help implementations provision multiple certificates. As with other internet drafts at this stage of the process, they are not final mechanisms. Rather, they are starting points, intended to demonstrate feasibility and some approaches, for the working group to build on for a complete solution.

The remainder of this section discusses what this problem statement means, and why it is important to solve. Subsequent sections provide an overview of the two solutions, discussion on design goals and key cases, and finally some alternatives we considered, including various existing solutions (e.g. fingerprinting and cross-signing), which do not meet the requirements.

### Background

[TLS](https://www.rfc-editor.org/rfc/rfc8446) uses [X.509 certificates]((https://www.rfc-editor.org/rfc/rfc5280) to associate a TLS endpoint's DNS names, or other application identifiers, with its TLS key. These associations are signed by certificate authorities (CAs) and are presented to the peer, known as the *relying party*. Each relying party curates a set of CAs, called *trust anchors*, whose associations the relying party accepts. If the relying party's trust anchors can be trusted to only issue correct associations, the relying party can use TLS to securely connect to the authenticating party, known as the *subscriber*. The common case in TLS is server certificate authentication, where the subscriber is the server, and the relying party is the client. The roles are reversed with client certificates. For clarity, this document will primarily discuss the server certificate case, but most of the motivations and solutions apply analogously to client certificates.

In this system, the client's trust anchors directly impact service availability and user security:

* If the authoritative server is unable to obtain or present a certificate from a CA trusted by the client, it cannot serve that client. Without trustworthy CAs that servers can use, the client cannot connect.

* If a client trusts an untrustworthy CA, that CA can forge an invalid association with an attacker's TLS key. The attacker can then intercept the client's connection to the server. Note the CA used by the authoritative server is irrelevant to this attack, only client trust. Conversely, until the client no longer trusts the untrustworthy CA, the attack has not been mitigated.

This dynamic means PKIs necessarily evolve over time. To meet user security needs, clients remove compromised or untrustworthy CAs, so that they cannot be used to intercept TLS. To meet service availability needs, clients add trustworthy CAs that add net value to the ecosystem, so that servers are able to obtain acceptable certificates.

Some transitions combine the two: a long-lived CA key is more likely to be compromised and presents a higher user security risk. To rotate a key in a PKI, the client would add a new key from the CA operator, and then remove the old one.

### Client Diversity

As with other TLS parameters, client populations do not have completely uniform trust anchors.

PKI transitions described above inherently create this diversity, if for no other reason than out-of-date clients in the ecosystem. These clients could range from unupdatable TV set-top boxes to some IoT device to an individual user's browser that has not yet, or could not, communicate with its update service.

Beyond temporal divergence, different clients have different needs and make different decisions on behalf of their users. For example, within the Web PKI, root programs make trust decisions independently. Some more specialized clients, such as mobile apps, may be constrained to trusting one or two specific CAs

To support a potentially diverse set of clients, the server must present an acceptable certificate to each one. TLS has a standard solution for this across its many parameters: negotiation. For client certificates, the CertificateRequest message listed trusted CAs since SSL 3.0 and TLS 1.0. TLS 1.3 generalized this to server certificates with the [`certificate_authorities`](https://www.rfc-editor.org/rfc/rfc8446#section-4.2.4) extension, allowing [clients](https://www.rfc-editor.org/rfc/rfc8446#section-4.4.2.3) and [servers](https://www.rfc-editor.org/rfc/rfc8446#section-4.4.2.2) to select between available certificates by the peer's trust anchors.

However, `certificate_authorities`'s size is impractical for some applications. Existing PKIs may have many CAs, and existing CAs may have long X.509 names. As of August 2023, the [Mozilla CA Certificate Program](https://wiki.mozilla.org/CA/Included_Certificates) contained 144 CAs, with an average name length of around 100 bytes. Such TLS deployments often do not use trust anchor negotiation at all.

### Security vs Availability

Without a negotiation mechanism, changes to client trust to improve user security directly conflict with service availability, with user security being deprioritized in favor of avoiding any kind of breakage.

If the server cannot dispatch between certificates, it must obtain a single certificate that simultaneously satisfies all clients. This limits it to the intersection of supported clients' trust anchors. As servers consider a broader range of clients, that intersection shrinks. At internet scales, the oldest clients effectively last forever, so finding a suitable single CA becomes challenging if not impossible. Even using a new CA, be it a new CA operator or simply a rotated key, becomes a high risk operation for server operators, as some niche client may not accept the CA.

This creates a conflict that clients must navigate. When a client must update its policies to meet new security requirements, it adds to this diversity and the resulting server and CA challenges. The client must then choose between compromising on user security or burdening the rest of the ecosystem. User security usually pays the cost. It can take just one important server with one important other client to interfere with a security transition.

The drafts aim to relieve this conflict, in order to build a robust, secure, and flexible PKI.

## Overview

The two drafts follow the standard negotiation pattern in TLS, with different approaches and tradeoffs:

### Trust Expressions

Trust Expressions compresses the certificate authorities list by referencing "trust stores", maintained by root programs. Clients send "trust expressions", which reference these trust stores by name and version, and optionally exclude portions of them. The full CA list is then the union of the trust expressions sent.

When CAs issue certificates to servers, they include "trust store inclusion" metadata, which contain information for servers to evaluate their candidate certificates against the trust expressions.

To support this compression scheme, participating CAs and root programs must coordinate to configure consistent information in clients and servers. To do this, the CA periodically fetches a "trust store manifest" from the root program. The protocol additionally must solve a "version skew" problem where the client references a newer trust store version than the server knows of. Most of the protocol's complexity comes from these two parts.

### Trust Anchor Identifiers

Trust Anchor Identifiers consists of three parts:

1. To mitigate large X.509 names, short "trust anchor identifiers" uniquely identify each participating CA, at an estimated 5 bytes per CA. These are sent in a `trust_anchors` extension, analogous to `certificate_authorities`.

2. Bandwidth and privacy constraints may still prevent a client from enumerating its full CA list. Server use an extension to [HTTPS/SVCB](https://www.rfc-editor.org/rfc/rfc9460.html) to publish available trust anchors in DNS. The client fetches this, selects the desired option, and requests it in `trust_anchors`.

3. To accommodate stale or unavailable DNS records, servers also send available trust anchors in the TLS handshake. If the client rejects the certificate, it can use this to retry with a more accurate `trust_anchors` request.

### Comparison

Although they aim to solve the same problem, the two drafts work in very different ways:

* Trust Anchor IDs are usable by more kinds of PKIs. Trust Expressions express trust anchor lists relative to named “trust stores”, maintained by root programs. Arbitrary lists may not be easily expressible. Trust Anchor IDs does not have this restriction.

* When used with large trust stores, the retry mechanism in Trust Anchor IDs requires a new connection. In most applications, this must be implemented outside the TLS stack, so more components must be changed and redeployed. In deployments that are limited by client changes, this may be a more difficult transition. (The draft also sketches out an alternate retry scheme that avoids this.)

* Trust Expressions works with static server configuration. An ideal Trust Anchor IDs deployment requires automation to synchronize a server’s DNS and TLS configuration. [draft-ietf-tls-wkech](https://datatracker.ietf.org/doc/draft-ietf-tls-wkech/) could be a starting point for this automation. In deployments that are limited by server changes, this may be a more difficult transition.

* Trust Expressions require that CAs continually fetch information from manifests that are published by root programs, while Trust Anchor IDs rely only on static pre-assigned trust anchor identifiers.

* Trust Anchor IDs, when trust anchors are conditionally sent, have different fingerprinting properties. See [Privacy Considerations](https://davidben.github.io/tls-trust-expressions/draft-beck-tls-trust-anchor-ids.html#name-privacy-considerations).

* Trust Anchor IDs can only express large client trust stores (for server certificates), not large server trust stores. Large trust stores rely on the retry mechanism described in [the draft](https://davidben.github.io/tls-trust-expressions/draft-beck-tls-trust-anchor-ids.html#name-retry-mechanism), which is not available to client certificates.

The two mechanisms can be deployed together. A subscriber can have metadata for both mechanisms available, and a relying party can advertise both. Mechanisms with aspects of both may potentially also be designed.

### Server Configuration

Both drafts anticipate a model where servers provision a collection of candidate certificate paths, with some associated selection metadata, instead of a single path. When a client connect, the server software automatically selects one to send by matching the ClientHello message against the selection metadata, along with other TLS criteria such as ECDSA vs RSA. If there are multiple matches, server software chooses based on its own criteria, such as certificate size.

If the negotiation mechanism is either not supported or did not match a candidate, servers should behave as they do today, possibly falling back to a single default certificate or heuristics. However, each client that deploys negotiation no longer constrains that fallback path, increasing the choices available for a valid fallback certificate for the remaining clients.

### ACME Extension

The two drafts also define an optional supporting ACME extension to help servers provision multiple certificates. When the server is configured to speak to some ACME server, the ACME server can return *multiple* certificate paths for the server's requested key and identity. As CAs already maintain relationships with root programs, they are well-positioned to update the kinds of certificates they provision in response to PKI changes. Automation allows subscribers to benefit from this without manual reconfiguration.

For ACME, the drafts currently propose a minimal change to existing mechanisms, where a new MIME type provides the metadata alongside each certificate path, and the [existing alternate certificate mechanism](https://www.rfc-editor.org/rfc/rfc8555.html#section-7.4.2) allows provisioning multiple alternate certificates in response to one request.

Where an individual ACME server does not cover all that some server operator needs, the server operator can also combine the outputs of multiple ACME servers, with the server software automatically selecting from the combined set.

## Design Goals

In solving this problem, there were a few notable design goals to meet:

* Enable PKIs to evolve as needed for user security, in a timely manner and without conflicting with availability. We are particularly focused on PKIs that share characteristics with the existing Web PKI.
* Minimize bandwidth cost in the TLS handshake. With the post-quantum transition, cryptographic material will become [much more expensive](https://dadrian.io/blog/posts/pqc-signatures-2024/) to send.
* Minimize burden to server operators, particularly avoiding ongoing manual work. Most deployments should leave ongoing decisions to TLS software, ACME software, and ACME server behavior.

We discuss these in more detail below:

### Timely PKI Evolution

If a CA is compromised or otherwise considered untrustworthy, mitigating the TLS interception attacks requires that the client no longer trust the CA. This means we should minimize delays in completing PKI transitions at the client, as that delay translates to time during which users remain at risk.

In other transitions, reducing delays is also valuable. If trustworthy CA is considered to provide net value to the ecosystem and added, a timely transition allows the ecosystem to realize those benefits sooner. For example, the new CA may provide post-quantum security or improved automation.

Negotiation mechanisms naturally achieve this by letting the server serve any collection of certificates it needs to maintain availability. Compared to some [alternate solutions](#considered-alternatives), the server is not constrained to wait for CAs to cross-sign each other (if they even will, as a cross-sign certifies *all* certificates from the other CA, past and future), for a cross-sign to appear in a predistributed list, etc.

The ACME multi-certificate extensions can further reduce user security risk, by allowing ACME servers to automatically manage some transitions. For example, when rotating CA keys, the ACME server can automatically provision chains for both the old and new key.

### Minimizing Bandwidth Cost

We wish to minimize bandwidth cost in the TLS handshake. This includes bandwidth from the negotiation mechanism itself, and bandwidth from the certificates that are negotiated. Increased handshake bandwidth leads to delays in serving users (e.g. slower webpage loads) and consumes resources like the user's data plan.

The two drafts reduce the bandwidth costs of the `certificate_authorities` extension. For PKIs where `certificate_authorities` was already viable, this is a size optimization over the existing deployment model. For PKIs where `certificate_authorities` was prohibitively large, this provides negotiation with minimal size cost.

Negotiation mechanisms can additionally unlock size optimizations by skipping unnecessary intermediate certificates and cross-signs in the common case. Servers no longer need to serve a larger payload to target the lowest common denominator. This will be particularly valuable for the post-quantum transition, when signatures become [much more expensive](https://dadrian.io/blog/posts/pqc-signatures-2024/). By further tailoring solutions to different clients, [even more size savings](https://github.com/davidben/merkle-tree-certs) are possible.

### Minimizing Server Operator Burden

The drafts aim to minimize server operator burden. While configuring one
vs. several certificates can be the difference between providing service to all
supported clients or not, there are more decisions to make. Where possible, it
is better for server operators to make simpler, high-level decisions (e.g. which
ACME endpoint(s) cover the clients they need), with automation handling the
rest.

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


### Abridged Certificates

Some PKI transitions can be managed, imperfectly and with significant costs, by a combination of cross-signs and intermediate compression. The [Abridged Certificates](https://datatracker.ietf.org/doc/draft-ietf-tls-cert-abridge/) draft describes a mechanism for compressing pre-shipped intermediate certificates.

In its current form, [draft-01](https://www.ietf.org/archive/id/draft-ietf-tls-cert-abridge-01.html), Abridged Certificates does not address these scenarios. It allocates a single codepoint for a fixed snapshot of intermediates in the PKI. By doing so, it cannot compress any intermediate that postdates the snapshot. This means it cannot handle future transitions. Moreover, it gives a size advantage to incumbents in the Web PKI, thus discouraging any further improvements in user security. It also cannot handle non-Web-PKI uses like private PKIs.


### Alternate Intermediate Compression

A hypothetical alternate intermediate elision scheme could be designed which avoids these drawbacks, but it would need to be much more complex to accommodate version skew, and the operational challenges of provisioning servers with the right metadata to evaluate information from newer and newer clients. The Trust Expressions draft is an example of how to address these challenges, notably CertificatePropertyList, the ACME extension, the versioned trust stores, and the cross-version invariants managed by `excluded_labels` and expiry.

Such a hypothetical scheme would be more applicable than Abridged Certificates, but imposes numerous costs, depending on the cross-signing scheme, including from delays to in security incident response, higher bandwidth usage in high-fanout scenarios, and the need to ship intermediates to clients that aren’t expected to need them. See also this [more detailed discussion](pki-transition-strategies.md).


## References and Acknowledgements

The authors thank Nick Harper, Sophie Schmieg, and Emily Stark for many valuable discussions and insights which led to this design, as well as review of early iterations of the draft specification.
