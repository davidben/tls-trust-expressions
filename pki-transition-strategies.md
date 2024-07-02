# PKI Transition Strategies

As PKIs evolve to meet user security needs, clients change the requirements they impose on web servers. Such changes result in challenges for server operators, who must meet these new requirements, and CAs, who must maintain infrastructure to meet server operator needs.

This document discusses some of these changes, and different strategies for the ecosystem to respond to them. We evaluate the strategies against following goals:

* Keep websites working in all their supported clients, including older clients and clients using different root programs
* Minimize bandwidth in the TLS handshake, particularly with larger post-quantum signatures
* Minimize the time needed for clients to make the desired user security improvement
* Minimize operational burden to TLS servers
* Minimize operational burden to CAs

Difficulties in meeting these goals, e.g. if some PKI change prevents a server from simultaneously supporting old and new clients, will result in delays and pressure against the PKI change. When this happens, user security usually pays the price, so it is important for PKI transitions to proceed smoothly.

When this document refers to "trust anchor negotiation" it means either of [TLS Trust Anchor Identifiers](https://davidben.github.io/tls-trust-expressions/draft-beck-tls-trust-anchor-ids.html) or [TLS Trust Expressions](https://davidben.github.io/tls-trust-expressions/draft-davidben-tls-trust-expr.html). While they differ in approach, both mechanisms aim to allow the server to select certificates based on the client's trusted CAs.


## Key Rotation

As a key grows in age, the risk of it being compromised or misused increases. A client may wish to rotate the keys that it trusts, trusting only newer keys and removing support for older ones, so that it is no longer vulnerable to the old key.


### Old Key Cross-Signs New Key

The CA operator can issue an intermediate from the old key to the new key, and then the server sends the intermediate on every connection. Older clients build a longer path, while newer clients stop the path earlier.

This wastes bandwidth on intermediates that only the oldest clients use. The bandwidth cost grows over time; as keys rotate, the server needs to send more and more old intermediates.


### Retiring Obsolete Cross-Signs

The server could stop sending intermediates past some point. This will break clients, so the server can only retire an intermediate when the oldest supported client no longer needs it. In the steady state, the server will still send many intermediates that are unnecessary for the vast majority of clients.

This is not a good solution for a post-quantum PKI, where even a single unnecessary signature has a high bandwidth cost. There is also an operational challenge here, as one must somehow translate the server’s support goals (the oldest supported clients) to the necessary chains, which is determined by CA behavior and client policy. No solution exists today to coordinate this.


### Authority Information Access

Older clients can survive a missing intermediate if they fetched them from the [Authority Information Access](https://www.rfc-editor.org/rfc/rfc5280#section-4.2.2.1) (AIA) extension. A server might then be able to cut off older intermediates sooner. However, this is not universally implemented, does not work reliably (e.g. in a captive portal), and has performance and privacy costs. This makes it difficult for a server to rely on it in the older clients that would need it.

As a result, while it might reduce the above bandwidth problem, servers would still be forced to send unnecessary intermediates, at a significant bandwidth cost for a post-quantum PKI. The above operational challenges also apply, with yet another criteria to incorporate.

Moreover, where this solution does work, it significantly degrades performance on older clients, adding bandwidth and latency costs. While these older clients are likely rarer, they are also likely weaker, so some deployments may prefer another tradeoff.


### Abridged Certificates

[draft-ietf-tls-cert-abridge](https://datatracker.ietf.org/doc/draft-ietf-tls-cert-abridge/) aims to allow servers to skip intermediates the client already has. In its current form in [draft-01](https://www.ietf.org/archive/id/draft-ietf-tls-cert-abridge-01.html), it is unusable for this as it blesses a single, global point-in-time snapshot of the PKI as one codepoint. This cannot handle future rotations, advantages incumbents, and thus discourages this and other PKI transitions.


### Alternate Intermediate Compression Scheme

An alternate intermediate compression scheme with versioned intermediate sets would avoid the above issues. This would require a much more complex design and repeat much of the work done in Trust Expressions to handle version skew and how server software can learn about their corresponding sets. This hypothetical scheme would have some consequences:

First, the entire intermediate set must be pushed and stored to each client. The client cannot safely trim an unreachable intermediate because it may become reachable in the future with a cross-sign. With every PKI transition, the intermediate set will grow over time.

Second, without named sets analogous to Trust Expressions’ named trust stores, this scheme adds some coordination delays to PKI transitions. Until the global intermediate set mints a new version, there is no size optimization and the transition cannot practically begin.

Finally, older clients would not have an updated intermediate set and would need to be served the cross-sign. These older clients incur extra bandwidth cost. While these older clients are likely rarer, they are also likely weaker, so some deployments may prefer another tradeoff.


### Trust Anchor Negotation with Cross-Sign

The two trust anchor negotiation drafts, with the ACME extension, can directly support rotation. The ACME server provides two chains, one with the cross-sign and one without the cross-sign, for TLS clients that trust the old root and the new root, respectively. If the client supports the shorter chain, the server will send it. Otherwise, it will send the longer chain.

The multiple certificate provisioning mechanism on the ACME side ensures this can proceed automatically, without manual configuration by the server operator.

As above, older clients would need to be served the cross-sign and incur extra bandwidth costs. While these older clients are likely rarer, they are also likely weaker, so some deployments may prefer another tradeoff. However, trust anchor negotiation enables deployments to navigate this tradeoff, as discussed below.


### Trust Anchor Negotation with Parallel Issuance

Trust anchor negotiation enable a second deployment strategy. Rather than cross-signing the new key with the old one, the CA could operate both roots in parallel. Whenever a subscriber requests a certificate, it issues two parallel chains. As above, the ACME extensions automate this: the CA simply provides both chains, and server software automatically selects between the two.

This is a trade-off between CA operational burden and bandwidth. Parallel issuance means the CA must maintain two instances of issuing infrastructure. But, by doing so, they keep the chains for both the old and new roots small.

Both strategies meet the security and availability goals, but this strategy may be preferable with large post-quantum signatures. In the various cross-signing designs, once a CA switches to issuing from the new root, older clients pay a bandwidth cost. While older clients are generally more rare, this may not always be the preferred tradeoff. At the start of a transition, older clients will be more common. Server operators and clients may then prefer to avoid the bandwidth costs until the percentage of old clients is sufficiently small. Parallel issuance keeps the costs to both populations low, so the PKI can transition faster, at the cost of higher CA operational overhead.

One might also combine these strategies, temporarily maintaining parallel infrastructure during the initial stages of the transition, then switching to a lower-burden cross-sign when optimizing for the older clients is no longer justified.


## New CA Operators

Root programs add new CAs when they consider doing so to be beneficial to their end users. Consider a server which wishes to serve certificates from this new CA:


### Switching to the New CA

If the server only uses the new CA, it cannot interoperate with older clients which do not support it yet. This option means that the new CAs cannot be adopted until all older clients are out of support by the server. Determining when this has occurred is inherently risky to the server operator.


### Cross-Sign from Existing CA

If an existing CA cross-signs the newer CA, the server can serve the cross-sign along with the new CA. This improves compatibility, but has some bandwidth and organizational challenges.

First, this wastes bandwidth on intermediates that only older clients use. This cost is particularly pronounced for post-quantum PKIs, where even a single unnecessary signature and public key pair is significant.

Second, by cross-signing the new CA, the existing CA delegates all its authority to the new CA. The existing CA is now responsible for all certificates issued by the new CA, and the new CA can now issue in all contexts where the existing CA was trusted. This dramatically reduces the scenarios where the existing CA would be willing to perform this cross-sign. An individual server operator has little ability to influence this outcome. If no sufficiently ubiquitous existing CA is willing to cross-sign the new CA, server operators cannot use any solutions of this nature.


### Abridged Certificates

One might attempt to solve the bandwidth, but not organizational, challenges with [draft-ietf-tls-cert-abridge](https://datatracker.ietf.org/doc/draft-ietf-tls-cert-abridge/). As discussed above, in its current form in [draft-01](https://www.ietf.org/archive/id/draft-ietf-tls-cert-abridge-01.html), it is unusable for this as it blesses a single, global point-in-time snapshot of the PKI as one codepoint. This cannot handle new cross-signs that post-date this single snapshot, advantages incumbents, and thus discourages this and other PKI transitions.


### Alternate Intermediate Compression Scheme

As discussed above, one could design an alternate intermediate compression scheme by adapting the versioning mechanisms in trust expressions. This hypothetical scheme would have some consequences:

First, the entire intermediate set must be pushed and stored to each client. The client cannot safely trim an unreachable intermediate because it may become reachable in the future with a cross-sign. With every such cross-sign, the intermediate set will grow over time.

Second, without named sets analogous to Trust Expressions’ named trust stores, this scheme adds coordination delay before the new CA is useful. Until the global intermediate set mints a new version, there is no size optimization and the new CA cannot practically be used.


### Trust Anchor Negotiation with Cross-Sign

The two trust anchor negotiation drafts, with the ACME extension, can directly support intermediate elision. The ACME server provides two chains, one with the cross-sign and one without the cross-sign, for TLS clients that trust the old root and the new root, respectively. If the client supports the shorter chain, the server will send it. Otherwise, it will send the longer chain.

The multiple certificate provisioning mechanism on the ACME side ensures this can proceed automatically, without manual configuration by the server operator.


### Root Program Cross-Sign

None of the above address the organizational challenges with cross-CA signatures. One could imagine [root programs operating roots for the purpose of compatibility cross-signs](https://mailarchive.ietf.org/arch/msg/tls/XXPVFcK6hq3YsdZ5D-PW9g-l8fY/). This solution shares the above discussion on cross-signs, with an added fan-out cost: there are many root programs, and thus many cross-signs. This means:

* Bandwidth costs of cross-signs multiply. In any case where the server sends all cross-signs, the cost will be prohibitively expensive in a post-quantum PKI.
* Robust X.509 [path-building](https://www.rfc-editor.org/rfc/rfc4158.html) is needed with non-linear PKI topologies, but this is not reliably implemented. This makes it difficult for a server to rely on it in the older clients that would need the intermediates.
* [Abridged certificates](https://datatracker.ietf.org/doc/draft-ietf-tls-cert-abridge/) do not address the problem. As of writing, [draft-01](https://www.ietf.org/archive/id/draft-ietf-tls-cert-abridge-01.html) is a point-in-time snapshot and advantages incumbents in the PKI, and thus discourages this and other PKI transitions.
* A hypothetical alternate intermediate compression scheme would have several costs. The set now contains cross-signs scaling with the number of root programs. Moreover, the older clients, who will not have the updated intermediate set, must be sent all cross-signs, not just the one applicable to them. This cost is prohibitively high in a post-quantum PKI.

As above, trust anchor negotiation enables a solution whose costs do not scale with the number of cross-signs and has no significant fan-out costs. By negotiating the actual trust anchors, the server can send only the particular cross-sign needed, or no cross-sign at all if none is needed. Clients that directly trust the new CA do not need to retain a copy of any cross-sign locally.


### Trust Anchor Negotiation with Parallel Issuance

As with the root rotation scenario above, trust anchor negotiation also enables a parallel issuance strategy. Instead of cross-signs, with their organizational risks, the server can simply obtain certificates from both the new and an existing CA, with the TLS software automatically selecting which to use.

This could either be done automatically, if the server’s ACME server provides both types of certificates, or with manual configuration by the server operator, by running ACME against both URLs. The first option has lower organizational risk than cross-signing, as providing an additional certificate via ACME does not make one responsible for issuance by that CA. The second option, while more work, is the _only_ solution is within reach for a server operator acting unilaterally.


## CA Removal

To meet user security needs, root programs sometimes remove CAs, e.g. because they are no longer considered trustworthy. When this happens, the security goal is for new clients to no longer be vulnerable to the removed CA.


### Switching CAs

The most natural server remediation is to reconfigure servers to switch to a different CA, still considered trustworthy. This, however, has two challenges:

1. The client cannot remove the untrustworthy CA until practically all server operators have taken this manual action. At the point of removal, any servers still using the untrustworthy CA will stop working.
2. Selecting this replacement CA can be challenging. The replacement CA must be supported by _all_ clients that the server wishes to support and, by removing one, this intersection shrinks.


### Distrust Only New Certificates

The client may choose to only distrust new certificates, issued after some cutoff date, while continuing to trust existing certificates. This helps with challenge (1) above, with some caveats:

* If using the notBefore date, this is not an effective security measure; the user is still vulnerable to the distrusted CA, who may backdate certificates
* Distrusting only new certificates still leaves the user vulnerable to existing misissued certificates. Those must be revoked separately, which requires identifying the set of certificates to re-evaluate.

Since Certificate Transparency pairs certificates with SCTs issued from a third party, these timestamps can be used to measure the cutoff date, so the distrusted CA cannot backdate certificates in CT-enforcing clients. Since the existence of SCTs implies the corresponding certificate is present in multiple CT logs, the set of remaining trusted certificates is known and will decrease over time as certificates expire or are replaced. While this strategy is a useful tool, it is not a complete solution:

* Not all distrusts present fact patterns that lend themselves to SCT-based distrust. For example, if too many certificates were misissued before the cutoff date, revocation lists may become too large to feasibly deliver to clients. Moving cutoff dates earlier risks additional breakage for sites unaffected by misissuance. During PKI transitions, root programs evaluate such tradeoffs, picking the approach that best suits the security needs of their users.

* There are certain CA failure modes for which trusting certificates before a certain date is inappropriate. For example, a CA that has been performing domain validation incorrectly may have never issued a certificate correctly. Auditing the entire corpus of certificates issued from this CA to determine which were misissued may be infeasible due to both scale and changes in subscribers and domain ownership over time.

* Clients that do not enforce Certificate Transparency are also still vulnerable to backdating, since such certificates would successfully validate without the SCTs present to indicate the incorrect issuance time asserted in the certificate.

* Additionally, this strategy does not help server operators address challenge (2). The server must still switch to a replacement CA before the cutoff date, under the same constraints as in a more straightforward distrust. The following section discusses this further.


### Identifying a Replacement CA

If limited to one certificate, switching CAs is risky for server operators. The new certificate must stay in the intersection of all supported relying parties. Every untrustworthy CA that is removed reduces this intersection. As the PKI evolves over time, this intersection will shrink.

This conflict impacts scenarios such as:

* A website needs to work with both modern clients and unupdatable clients like TV set-top boxes, payment terminals, etc. The latter’s CA set is effectively frozen until the consumer, merchant, etc., replaces the device.
* Although some client, like a browser, may broadly have an update service, coverage is not complete and some very old clients remain in the ecosystem. The server operator needs to support users using both modern clients and those old ones.
* Some client [pinned](https://www.rfc-editor.org/rfc/rfc7469.html) the site to a particular CA. Years later, the CA is compromised and must be removed from modern clients.
* Root programs in the Web PKI, which do not coordinate on trust actions, disagree on what actions to take with a particular CA.

Moreover, the subscriber may not even know the precise intersection. Often, the only option is to try the new certificate and monitor errors. For subscribers that serve many diverse relying parties, this is a disruptive and risky process. Adding such risk to what is ultimately a security incident response adds complications and delays. This ultimately makes it more difficult to remove the untrustworthy CA and address the user security risk.


### Cross-Sign from Distrusted CA

If the distrusted CA cross-signs another existing CA, the server operator can serve a chain from that CA, along with a cross-sign to the distrusted CA. This has analogous organizational challenges to the other cross-signing scenarios discussed above: the distrusted CA must be willing to cross-sign some new CA. Even when it is willing, the server operator likely has no choice in the replacement CA and must use the one chosen _by the distrusted CA_. This replacement CA may not meet the server operator’s ubiquity requirements, or other requirements such as availability or automation.

Also as in the above discussions, this cross-sign has a significant bandwidth cost for a post-quantum PKI, unless the extra intermediate can be omitted. Analogous considerations apply:

* [Abridged certificates](https://datatracker.ietf.org/doc/draft-ietf-tls-cert-abridge/) do not address the problem. As of writing, [draft-01](https://www.ietf.org/archive/id/draft-ietf-tls-cert-abridge-01.html) is a point-in-time snapshot and advantages incumbents in the PKI, and thus discourages this and other PKI transitions.
* A hypothetical alternate intermediate compression scheme would address it, but it must ship the unused cross-sign to modern clients that do not trust the distrusted CA. Every distrust which uses this transition strategy grows the set.
* If using a global versioning for the hypothetical compression scheme, the cross-sign is not compressible until the global intermediate set makes a new revision. **This adds coordination delay to a security incident response, and leaves users at risk for longer.**

Trust anchor negotiation allows the server software to decide whether to include the cross-sign. However, it also enables a more direct solution, which avoids the organizational challenges with the cross-sign.


### Trust Anchor Negotiation with Parallel Issuance

Instead of limiting to the distrusted CA’s chosen successor, the server operator can simply obtain certificates from both the distrusted CA and another CA. Trust anchor negotiation allows the server software to direct the right certificate to each client, and thus the server operator can react to the client policy change with minimal added deployment risk.

From there, the server operator may asynchronously monitor usage of the old certificate and retire it when needed. Or it may continue to serve it indefinitely if needed. Trust anchor negotiation decouples this decision from the immediate security incident response, reducing user security risk faster.


### Backup CAs

The server operator can obtain certificates from multiple CAs ahead of time, for redundancy, further reducing their availability risk during a client’s security incident response. With trust anchor negotiation, the server software will then automatically react to changes in client requirements and serve the correct one, with no manual configuration change by the server operator.


## Long-Lived Offline Keys

Today, CAs in the Web PKI typically keep their long-lived (e.g. > 2 year lifetime) keys offline, instead periodically issuing shorter-lived intermediates. If this security requirement continues to a post-quantum PKI, the significant size costs of post-quantum intermediates again becomes a challenge. The solution space is analogous to the above discussion on intermediates:


### Lowest Common Denominator

Today, the server always sends these intermediates. This has a significant bandwidth cost with post-quantum.


### Abridged Certificates

As above, the [abridged certificates](https://datatracker.ietf.org/doc/draft-ietf-tls-cert-abridge/) design is currently, in [draft-01](https://www.ietf.org/archive/id/draft-ietf-tls-cert-abridge-01.html), a fixed snapshot of intermediates. It can compress intermediates that predate its introduction, but not future ones. This means it advantages incumbents. New CAs, introduced for transitions such as those described in this document, cannot take advantage of this optimization.


### Alternate Intermediate Compression Scheme

As above, an alternate intermediate compression scheme, incorporating something like the versioning scheme of trust expressions, could compress the intermediate without the above drawbacks of abridged certificates.

However, in an intermediate compression scheme, clients must carry the entire intermediate set, even CAs they do not trust. Even if an intermediate appears unreachable from the client’s trusted CAs, it may become reachable with a cross-sign.


### Trust Anchor Negotiation with Intermediate Certificate

Pre-shipping a trusted intermediate with a client is equivalent to simply treating the intermediate as a trust anchor, with some time bound. Trust anchor negotiation can then be used to implement intermediate compression with no additional mechanism. (See the [explainer](explainer.md#minimizing-server-operator-burden) for a discussion on why this is valuable.)

CAs use the ACME extension to ship two chains to the TLS server, one with the intermediate, rooted at the long-lived key, and one without the intermediate, rooted at the short-lived key. Clients trust both the long-lived key and the most recent corresponding short-lived key. If the client’s trust anchors are up-to-date, the server will use the shorter chain. If the client’s trust anchors become out-of-date, the server will automatically fallback to the longer chain, until the client is updated.


### Trust Anchor Negotiation with Parallel Issuance

As in the other scenarios, trust anchor negotiation enables a parallel issuance alternative. The CA maintains two parallel signing infrastructures:

1. An offline, long-lived root A1, which cross-signs an online, short-lived intermediate A2
2. An online, short-lived root B1, separate from A1 and A2

The CA then issues from the two chains independently. The short-lived keys, A2 and B1, are periodically rotated. The clients then trust A1 and the current instance of B1, with negotiation happening as above.

While this solution has higher operational overhead for the CA, it allows a significant size optimization with some post-quantum algorithms. A signature scheme like [UOV](https://www.uovsig.org/) trades off small signatures for extremely large public keys. It is impractical for intermediates, but may be viable for one of a small number of roots. This is because intermediates require shipping both public key and signature, while root public keys are pre-distributed, so most (but not all) of the cost is in the signature.

The CA could then use UOV for A1 and B1, but ML-DSA for A2. This allows newer clients to take advantage of short UOV signatures from B1, without shipping a prohibitively large UOV public key in A2 for older clients. This optimization requires parallel issuance. Cross-signing would require A2 and B1 to be the same key.

If using, e.g., uov-Is and ML-DSA-44, this scheme uses 96 + 1,312 + 2,420 = 3,828 bytes for old clients, and 96 bytes for new clients. A cross-signing design, which practically forces B1 to use ML-DSA, uses 2,420 bytes for new clients, a significant increase.
