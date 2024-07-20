---
title: "TLS Trust Anchor Identifiers"
category: std


docname: draft-beck-tls-trust-anchor-ids-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Transport Layer Security"
venue:
  group: "Transport Layer Security"
  type: "Working Group"
  mail: "tls@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/tls/"
  github: "davidben/tls-trust-expressions"
  latest: "https://davidben.github.io/tls-trust-expressions/draft-beck-tls-trust-anchor-ids.html"

author:
 -
    ins: "B. Beck"
    name: "Bob Beck"
    organization: "Google LLC"
    email: bbe@google.com

 -
    ins: "D. Benjamin"
    name: "David Benjamin"
    organization: "Google LLC"
    email: davidben@google.com

 -
    ins: "D. O'Brien"
    name: "Devon O'Brien"
    organization: "Google LLC"
    email: asymmetric@google.com

normative:
  X680:
       title: "Information technology - Abstract Syntax Notation One (ASN.1): Specification of basic notation"
       date: 2021
       author:
         org: ITU-T
       seriesinfo:
         ISO/IEC: 8824-1:2021

  X690:
       title: "Information technology - ASN.1 encoding Rules: Specification of Basic Encoding Rules (BER), Canonical Encoding Rules (CER) and Distinguished Encoding Rules (DER)"
       date: 2021
       author:
         org: ITU-T
       seriesinfo:
         ISO/IEC: 8825-1:2021

informative:

  MOZILLA-ROOTS:
    title: Mozilla Included CA Certificate List
    target: https://wiki.mozilla.org/CA/Included_Certificates
    date: 2023-08-30
    author:
    - org: Mozilla

--- abstract

This document defines the TLS Trust Anchors extension, a mechanism for relying parties and subscribers to convey trusted certification authorities. It describes individual certification authorities more succinctly than the TLS Certificate Authorities extension.

Additionally, to support TLS clients with many trusted certification authorities, it supports a mode where servers describe their available certification paths and the client selects from them. Servers may describe this during connection setup, or in DNS for lower latency.

--- middle

# Introduction

TLS {{!RFC8446}} endpoints typically authenticate using X.509 certificates {{!RFC5280}}. These are assertions by a certification authority (CA) that associate a TLS key with DNS names or other identifiers. If the peer trusts the CA, it will accept this association. The authenticating party (usually the server) is known as the *subscriber* and the peer (usually the client) is the *relying party*.

{{Section 4.2.4 of RFC8446}} defines the `certificate_authorities` extension, which allows the subscriber, who may have multiple certificates available, to select the correct certificate for the relying party. However, this extension’s size is impractical for some applications.

Without such a negotiation mechanism, the subscriber must somehow obtain certificates which simultaneously satisfy all relying parties. This is a challenge for subscribers when relying parties are diverse. This translates to analogous challenges for CAs and relying parties:

* For a new CA to be usable by subscribers, all relying parties must trust it. This is particularly challenging for older, un-updatable relying parties. Existing CAs face similar challenges when rotating or deploying new keys.

* When a relying party must update its policies to meet new security requirements, it must choose between compromising on user security or imposing a significant burden on subscribers that still support older relying parties.

The `certificate_authorities` extension’s size becomes impractical due to two factors. First, X.509 names are large. Second, many clients, notably web browsers, trust a large number of CAs. As of August 2023, the Mozilla CA Certificate Program {{MOZILLA-ROOTS}} contained 144 CAs, with an average name length of around 100 bytes. This document introduces Trust Anchor Identifiers, which aims to address both of these challenges.

There are four parts to this mechanism:

1. {{trust-anchor-ids}} defines *trust anchor identifiers*, which are short, unique identifiers for X.509 trust anchors.


2. {{tls-extension}} defines a TLS extension that communicates the relying party’s requested trust anchors, and the subscriber’s available ones. When the relying party is a TLS client, it can mitigate large lists by sending a, possibly empty, subset of its trust anchors to the TLS server. The server provides its list of available trust anchors in response so that the client can retry on mismatch.

3. {{dns-service-parameter}} allows TLS servers to advertise their available trust anchors in HTTPS or SVCB {{!RFC9460}} DNS records. TLS clients can then request an accurate initial subset and avoid a retry penalty.

4. {{acme-extension}} defines an ACME {{!RFC8555}} extension for provisioning multiple certification paths, each with an associated trust anchor identifier.

Together, they extend the `certificate_authorities` mechanism to a broader set of applications, enabling a more flexible and robust PKI.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitaIs, as shown here.

This document additionally uses the TLS presentation language, defined in {{Section 3 of !RFC8446}}, and ASN.1, defined in {{X680}}.

## Terminology and Roles

This document discusses three roles:

Subscriber:
: The party whose identity is being validated in the protocol. In TLS, this is the side sending the Certificate and CertificateVerify message.

Relying party:
: The party authenticating the subscriber. In TLS, this is the side that validates a Certificate and CertificateVerify message.

Certification authority (CA):
: The service issuing certificates to the subscriber.

Additionally, there are several terms used throughout this document to describe this proposal:

Trust anchor:
: A pre-distributed public key or certificate that relying parties use to determine whether a certification path is trusted.

Certification path:
: An ordered list of X.509 certificates starting with the target certificate. Each certificate is issued by the next certificate, except the last, which is issued by a trust anchor.


# Trust Anchor Identifiers {#trust-anchor-ids}

This section defines trust anchor identifiers, which are short, unique identifiers for a trust anchor. To simplify allocation, these identifiers are defined with object identifiers (OIDs) {{X680}} and IANA-registered Private Enterprise Numbers (PENs) {{!RFC9371}}:

A trust anchor identifier is defined with a OID under the OID arc of some PEN. For compactness, they are represented as relative object identifiers (see Section 33 of {{X680}}), relative to the OID prefix `1.3.6.1.4.1`. For example, an organization with PEN 32473 might define a trust anchor identifier with the OID `1.3.6.1.4.1.32473.1`. As a relative object identifier, it would be the OID `32473.1`.

Depending on the protocol, trust anchor identifiers may be represented in one of three ways:

* For use in ASN.1-based protocols, a trust anchor identifier's ASN.1 representation is the relative object identifier described above. This may be encoded in DER {{X690}}, or some other ASN.1 encoding. The example identifier's DER encoding is the six-octet sequence `{0x0d, 0x04, 0x81, 0xfd, 0x59, 0x01}`.

* For use in binary protocols such as TLS, a trust anchor identifier's binary representation consists of the contents octets of the relative object identifier's DER encoding, as described in Section 8.20 of {{X690}}. Note this omits the tag and length portion of the encoding. The example identifier's binary representation is the four-octet sequence `{0x81, 0xfd, 0x59, 0x01}`.

* For use in ASCII-compatible text protocols, a trust anchor identifier's ASCII representation is the relative object identifier in dotted decimal notation. The example identifier's ASCII representation is `32473.1`.

Trust anchor identifiers SHOULD be allocated by the CA operator and common among relying parties that trust the CA. They MAY be allocated by another party, e.g. when bootstrapping an existing ecosystem, if all parties agree on the identifier. In particular, the protocol requires relying parties and subscribers to agree, and subscriber configuration typically comes from the CA.

## Relying Party Configuration

Relying parties are configured with one or more supported trust anchors. Each trust anchor which participates in this protocol must have an associated trust anchor identifier.

When trust anchors are self-signed X.509 certificates, the X.509 trust anchor identifier extension MAY be used to carry this identifier. The trust anchor identifier extension has an `extnID` of `id-trustAnchorIdentifier` and an `extnValue` containing a DER-encoded TrustAnchorIdentifier structure, defined below. The TrustAnchorIdentifier is the trust anchor identifier's ASN.1 representation, described in {{trust-anchor-ids}}. This extension MUST be non-critical.

~~~
id-trustAnchorIdentifier OBJECT IDENTIFIER ::= { TBD }

TrustAnchorIdentifier ::= RELATIVE-OID
~~~

Relying parties MAY instead or additionally configure trust anchor identifiers via some application-specific out-of-band information.

Relying parties MAY support trust anchors without associated trust anchor identifiers, but such trust anchors will not participate in this protocol. Those trust anchors MAY participate in other trust anchor negotiation protocols, such as the `certificate_authorities` extension.

## Subscriber Configuration

Subscribers are configured with one or more candidate certification paths to present in TLS, in some preference order. This preference order is used when multiple candidate paths are usable for a connection. For example, the subscriber may prefer candidates that minimize message size or have more performant private keys.

Each candidate path which participates in this protocol must be provisioned with the trust anchor identifier for its corresponding trust anchor.

To carry this information, this document defines the `trust_anchor_identifier` property for a CertificatePropertyList {{Section 5 of !I-D.davidben-tls-trust-expr}}. This `data` field of the property contains the binary representation of the trust anchor identifier of the certification path’s trust anchor, as described in {{trust-anchor-ids}}.

~~~
enum {
    trust_anchor_identifier(TBD), (2^16-1)
} CertificatePropertyType;
~~~

Subscribers MAY have candidate certification paths without associated trust anchor identifiers, but such paths will not participate in this protocol. Those paths MAY participate in other trust anchor negotiation protocols, such as the `certificate_authorities` extension.

[[TODO: Move the definition of CertificatePropertyList from the trust expressions draft to here, if this ends up being the canonical one. When doing so, also import media type definition.]]

# TLS Extension

This section defines the `trust_anchors` extension, which is sent in the ClientHello, EncryptedExtensions, CertificateRequest, and Certificate messages in TLS 1.3 or later.

## Overview

The `trust_anchors` extension is defined using the structures below:

~~~
enum { trust_anchors(TBD), (2^16-1) } ExtensionType;

opaque TrustAnchorIdentifier<1..2^8-1>;

TrustAnchorIdentifier TrustAnchorIdentifierList<0..2^16-1>;
~~~

When the extension is sent in the ClientHello or CertificateRequest messages, the `extension_data` is a TrustAnchorIdentifierList and indicates that the sender supports the specified trust anchors. The list is unordered, and MAY be empty. Each TrustAnchorIdentifier uses the binary representation, as described in {{trust-anchor-ids}}.

When the extension is sent in EncryptedExtensions, the `extension_data` is a TrustAnchorIdentifierList containing the list of trust anchors that server has available, in the server’s preference order, and MUST NOT be empty.

When the extension is sent in Certificate, the `extension_data` MUST be empty and indicates that the sender sent the certificate because the certificate matched a trust anchor identifier sent by the peer. When used in this form, the extension may only be sent in the first CertificateEntry. It MUST NOT be sent in subsequent ones.

## Certificate Selection

A `trust_anchors` extension in the ClientHello or CertificateRequest is processed similarly to the `certificate_authorities` extension. The relying party indicates some set of supported trust anchors in the ClientHello or CertificateRequest `trust_anchors` extension. The subscriber then selects a certificate from its candidate certification paths (see {{subscriber-configuration}}), as described in {{Section 4.4.2.2 of RFC8446}} and {{Section 4.4.2.3 of RFC8446}}. This process is extended as follows:

If the ClientHello or CertificateRequest contains a `trust_anchors` extension, the subscriber SHOULD send a certification path whose trust anchor identifier appears in the relying party’s `trust_anchors` extension.

If the ClientHello or CertificateRequest contains both `trust_anchors` and `certificate_authorities`, certification paths that satisfy either extension’s criteria may be used. This additionally applies to future extensions which play a similar role.

If no certification paths satisfy either extension, the subscriber MAY return a `handshake_failure` alert, or choose among fallback certification paths without considering `trust_anchors` or `certification_authorities`. See {{retry-mechanism}} for additional guidance on selecting a fallback when the ClientHello contains `trust_anchors`.

Sending a fallback allows the subscriber to retain support for relying parties that do not implement any form of trust anchor negotiation. In this case, the subscriber must find a sufficiently ubiquitous trust anchor, if one exists. However, only those relying parties need to be considered in this ubiquity determination. Updated relying parties may continue to evolve without restricting fallback certificate selection.

If the subscriber sends a certification path that matches the relying party’s`trust_anchors` extension, as described in {{certificate-selection}}, the subscriber MUST send an empty `trust_anchors` extension in the first CertificateEntry of the Certificate message. In this case, the `certificate_list` flexibility described in {{Section 4.4.2 of !RFC8446}} no longer applies. The `certificate_list` MUST contain a complete certification path, issued by the matching trust anchor, correctly ordered and with no extraneous certificates. That is, each certificate MUST certify the one immediately preceding it, and the trust anchor MUST certify the final certificate. The subscriber MUST NOT send the `trust_anchors` extension in the Certificate message in other situations.

If a relying party receives this extension in the Certificate message, it MAY choose to disable path building {{!RFC4158}} and validate the peer's certificate list as pre-built certification path. Doing so avoids the unpredictable behavior of path-building, and helps ensure CAs and subscribers do not inadvertently provision incorrect paths.

## Retry Mechanism

When the relying party is a client, it may choose not to send its full trust anchor identifier list due to fingerprinting risks (see {{privacy-considerations}}), or because the list is too large. The client MAY send a subset of supported trust anchors, or an empty list. This subset may be determined by, possibly outdated, prior knowledge about the server, such as {{dns-service-parameter}} or past connections.

To accommodate this, when receiving a ClientHello with `trust_anchors`, the server collects all candidate certification paths which:

* Have a trust anchor identifier, and
* Satisfy the conditions in {{Section 4.4.2.2 of RFC8446}}, with the exception of `certification_authorities`, and any future extensions that play a similar role

If this collection is non-empty, the server sends a `trust_anchors` extension in EncryptedExtensions, containing the corresponding trust anchor identifiers in preference order.

When a client sends a subset or empty list in `trust anchors`, it SHOULD implement the following retry mechanism:

If the client receives either a connection error or an untrusted certificate, the client looks in server’s EncryptedExtensions for a trust anchor identifier that it trusts. If there are multiple, it selects an option based on the server’s preference order and its local preferences. It then makes a new connection to the same endpoint, sending only the selected trust anchor identifier in the ClientHello `trust_anchors` extension. If the EncryptedExtensions had no `trust_anchor` extension, or no match was found, the client returns the error to the application.

Clients SHOULD retry at most once per connection attempt.

[[TODO: Retrying in a new connection is expensive and cannot be done from within the TLS stack in most implementations. Consider handshake modifications to instead retry within the same connection.]]

This mechanism allows the connection to recover from a certificate selection failure, e.g. due to the client not revealing its full preference list, at additional latency cost. {{dns-service-parameter}} describes an optimization which can avoid this cost.

This mechanism also allows servers to safely send fallback certificates that may not be as ubiquitously acceptable. Without some form of trust anchor negotiation, servers are limited to selecting certification paths that are ubiquitously trusted in all supported clients. This often means sending extra cross-certificates to target the lowest common denominator at a bandwidth cost. If the ClientHello contains `trust_anchors`, the server MAY opportunistically send a less ubiquitous, more bandwidth-efficient path based on local heuristics, with the expectation that the client will retry when the heuristics fail.

# DNS Service Parameter

This section defines the `tls-trust-anchors` SvcParamKey {{!RFC9460}}. TLS servers can use this to advertise their available trust anchors in DNS, and aid the client in formulating its `trust_anchors` extension (see {{retry-mechanism}}). This allows TLS deployments to support clients with many trust anchors without incurring the overhead of a reconnect.

## Syntax

The `tls-trust-anchors` parameter contains an ordered list of one or more trust anchor identifiers, in server preference order.

The presentation `value` of the SvcParamValue is a non-empty comma-separated list ({{Appendix A.1 of RFC9460}}). Each element of the list is a trust anchor identifier in the ASCII representation defined in {{trust-anchor-ids}}. Any other `value` is a syntax error. To enable simpler parsing, this SvcParam MUST NOT contain escape sequences.

The wire format of the SvcParamValue is determined by prefixing each trust anchor identifier with its length as a single octet, then concatenating each of these length-value pairs to form the SvcParamValue. These pairs MUST exactly fill the SvcParamValue; otherwise, the SvcParamValue is malformed.

For example, if a TLS server has three available certification paths issued by `32473.1`, `32473.2.1`, and `32473.2.2`, respectively, the DNS record in presentation syntax may be:

~~~ dns
example.net.  7200  IN SVCB 3 server.example.net. (
    tls-trust-anchors=32473.1,32473.2.1,32473.2.2 )
~~~

The wire format of the SvcParamValue would be the 17 octets below. In the example, the octets comprising each trust anchor identifier are placed on separate lines for clarity

~~~
0x04, 0x81, 0xfd, 0x59, 0x01,
0x05, 0x81, 0xfd, 0x59, 0x02, 0x01,
0x05, 0x81, 0xfd, 0x59, 0x02, 0x02,
~~~

## Configuring Services

Services SHOULD include the trust anchor identifier for each of their available certification paths, in preference order, in the `tls-trust-anchors` of their HTTPS or SVCB endpoints. As TLS configuration is updated, services SHOULD update the DNS record to match. The mechanism for this is out of scope for this document, but services are RECOMMENDED to automate this process.

Services MAY have certification paths without trust anchor identifiers, but those paths will not participate in this mechanism.

## Client Behavior

When connecting to a service endpoint whose HTTPS or SVCB record contains the `tls-trust-anchors` parameter, the client first computes the intersection between its configured trust anchors and the server’s provided list. If this intersection is non-empty, the client MAY use it to determine the `trust_anchors` extension in the ClientHello (see {{retry-mechanism}}).

If doing so, the client MAY send a subset of this intersection to meet size constraints, but SHOULD offer multiple options. This reduces the chance of a reconnection if, for example, the first option in the intersection uses a signature algorithm that the client doesn’t support, or if the TLS server and DNS configuration are out of sync.

Although this service parameter is intended to reduce trust anchor mismatches, mismatches may still occur in some scenarios. Clients and servers MUST continue to implement the provisions described in {{retry-mechanism}}, even when using this service parameter.

# ACME Extension

We reuse the ACME extension defined in {{Section 7 of !I-D.davidben-tls-trust-expr}}, except that each CertificatePropertyList will contain a `trust_anchor_identifier` property.

[[TODO: Move this over from the trust expressions draft, if this ends up being the canonical one.]]

# Use Cases

See {{Section 9 of I-D.davidben-tls-trust-expr}}.

[[TODO: Move the text here if this document ends up being the canonical one.]]


# Privacy Considerations

## Relying Parties

The negotiation mechanism described in this document is analogous to the `certificate_authorities` extension ({{Section 4.2.4 of RFC8446}}), but more size-efficient. Like that extension, it presumes the requested trust anchor list is not sensitive.

The privacy implications of this are determined by how a relying party uses this extension. Trust anchors supported by a relying party may be divided into three categories:

1. Trust anchors whose identifiers the relying party sends *unconditionally*, i.e. not in response to the server’s HTTPS/SVCB record, trust anchor list in EncryptedExtensions, etc.

2. Trust anchors whose identifiers the relying party sends *conditionally*, i.e. only if the server offers them. For example, the relying party may indicate support for a trust anchor if its identifier is listed in the server’s HTTPS/SVCB record or trust anchor list in EncryptedExtensions.

3. Trust anchors whose identifiers the relying party never sends, but still trusts. These are trust anchors which do not participate in this mechanism.

Each of these categories carries a different fingerprinting exposure:

Trust anchor identifiers sent unconditionally can be observed passively. Thus, relying parties SHOULD NOT unconditionally advertise trust anchor lists which are unique to an individual user. Rather, such lists SHOULD be either empty or computed only from the trust anchors common to the relying party's anonymity set ({{Section 3.3 of !RFC6973}}).

Trust anchor identifiers sent in response to the subscriber can only be observed actively. That is, the subscriber could vary its list and observe how the client responds, in order to probe for the client’s trust anchor list.

This is similar to the baseline exposure for trust anchors that do not participate in negotiation. A subscriber in possession of a certificate can send it and determine if the relying party accepts or rejects it. Unlike this baseline exposure, trust anchors that participate in this protocol can be probed by only knowing the trust anchor identifier.

Relying parties SHOULD determine which trust anchors participate in this mechanism, and whether to advertise them unconditionally or conditionally, based on their privacy goals. PKIs that reliably use the DNS service parameter ({{dns-service-parameter}}) can rely on conditional advertisement for stronger privacy properties without a round-trip penalty.

Additionally, a relying party that computes the `trust_anchors` extension based on prior state may allow observers to correlate across connections. Relying parties SHOULD NOT maintain such state across connections that are intended to be uncorrelated. As above, implementing the DNS service parameter can avoid a round-trip penalty without such state.

## Subscribers

If the subscriber is a server, the mechanisms in {{dns-service-parameter}} and {{retry-mechanism}} enumerate the trust anchors for the server’s available certification paths. This mechanism assumes they are not sensitive. Servers SHOULD NOT use this mechanism to negotiate certification paths with sensitive trust anchors.

In servers that host multiple services, this protocol only enumerates certification paths for the requested service. If, for example, a server uses the `server_name` extension to select services, the addition to EncryptedExtensions in {{retry-mechanism}} is expected to be filtered by `server_name`. Likewise, the DNS parameter in {{dns-service-parameter}} only contains information for the corresponding service. In both cases, co-located services are not revealed.

The above does not apply if the subscriber is a client. This protocol does not enumerate the available certification paths for a client.

# Security Considerations

## Relying Party Policies and Agility

The negotiation mechanism described in this document facilitates which certification path is served to relying parties, but has no impact on the relying party's trust preferences themselves.

As a result, this enables a more flexible and agile PKI, which can better mitigate security risks to users. {{use-cases}} discusses some scenarios which benefit from the multi-certificate deployment this document enables. In general, robust trust anchor negotiation helps subscribers navigate differences in relying party requirements. This means security improvements for one set of relying parties can be deployed without needing to risk incompatibility or breakage for others.

For example, agility reduces pressures on relying parties to sacrifice user security for compatibility. Suppose a subscriber currently uses some CA, but a relying party deems trusting that CA to pose an unacceptable security risk to its users. In a single-certificate deployment, those subscribers may be unwilling to adopt a CA trusted by the relying party, as switching CAs risks compatibility problems elsewhere. The relying party then faces compatibility pressure and adds this CA, sacrificing user security. However, in a multi-certificate deployment, the subscriber can use its existing CA _in addition to_ another CA trusted by relying party B. This allows the ecosystem to improve interoperability, while still meeting relying party B's user-security goals.

## Incorrect Selection Metadata

If the subscriber has incorrect information about trust anchor identifiers, it may send an untrusted certification path. This will not result in that path being trusted, but does present the possibility of a degradation of service. However, this information is provisioned by the CA, which is already expected to provide accurate information.

## Serving Multiple Certificates

In a multi-certificate deployment, subscribers have a collection of certificates available to satisfy relying parties with potentially different security policies. It is possible that some of these policies will only be satisfied by certificates from CAs that have been distrusted by other relying parties, such as if the relying party is out of date but still important for the subscriber to support. If a certificate asserts the correct information about the subscriber (TLS key, DNS names, etc.), the subscriber can safely present it, even if the CA is otherwise untrustworthy. Issuing a certificate for the subscriber's public key does not grant the CA access to the corresponding private key. Presenting such a certificate does not enable the CA decrypt or intercept the connection.

However, the subscriber presenting a certificate is not an endorsement of the CA. The subscriber's role is to present a certificate which will convince the relying party of the correct subscriber information. Subscribers do not vet CAs for trustworthiness, only the correctness of their specific, configured certificates and the CA's ability to meet the subscriber's requirements, such as availability, compatibility, and performance. It is the responsibility of the relying party, and its corresponding root program, to determine the set of trusted CAs. Trusting a CA means trusting _all_ certificates issued by that CA, so it is not enough to observe subscribers serving correct certificates. An untrustworthy CA may sign one correct certificate, but also sign incorrect certificates, possibly in the future, that can attack the relying party. Root program management is a complex, security-critical process, the full considerations of which are outside the scope of this document.

## Targeting TLS Interception

Trust Anchor Identifiers, like `certificate_authorities`, enables a TLS server to differentiate between clients that do and do not trust some CA. If this CA mis-issued a certificate, this could be used by a network attacker to only enable TLS interception with clients that trust the CA. The network attacker may wish to do this reduce the odds of detection. Clients which do not trust the CA will not accept the mis-issued certificate, which may be user-visible.

However, the attacker could also use any existing mechanism for differentiating clients. In TLS parameter negotiation, the client offers all its available TLS features, including cipher suites and other extensions, in the TLS ClientHello. This means any variation in any client TLS policies, related or unrelated to trust anchors, may be used as an implementation fingerprint to differentiate clients. While fingerprinting's heuristic nature makes it not viable for a broad, diverse set of TLS servers, it is viable for a network attacker's single interception service. Trust Anchor Identifiers only impacts detection where this differentiation was not already possible.

If the attacker targets any clients that enforce Certificate Transparency {{?RFC6962}}, the mis-issued certificates will need to be publicly logged. In this case, detection is more robust, and client differentiation, with or without Trust Anchor Identifiers, has no significant impact.

# IANA Considerations

## TLS ExtensionType Updates

IANA is requested to create the following entry in the TLS ExtensionType Values registry, defined by {{RFC8446}}:

| Value | Extension Name | TLS 1.3        | DTLS-Only | Recommended | Reference  |
|-------|----------------|----------------|-----------|-------------|------------|
| TBD   | trust_anchors  | CH, EE, CR, CT | N         | Y           | [this-RFC] |

## CertificatePropertyType Registry

[[TODO: Establish / add to the CertificatePropertyType registry]]

--- back

# Comparison to TLS Trust Expressions

{{I-D.davidben-tls-trust-expr}} describes Trust Expressions, another trust anchor negotiation mechanism that aims to solve similar problems. The mechanisms differ in the following ways:

* `trust_anchors` is usable by more kinds of PKIs. Trust Expressions express trust anchor lists relative to named “trust stores”, maintained by root programs. Arbitrary lists may not be easily expressible. `trust_anchors` does not have this restriction.

* When used with large trust stores, the retry mechanism in `trust_anchors` requires a new connection. In most applications, this must be implemented outside the TLS stack, so more components must be changed and redeployed. In deployments that are limited by client changes, this may be a more difficult transition. [[TODO: See {{retry-mechanism}} for an alternate retry scheme that avoids this.]]

* Trust Expressions works with static server configuration. An ideal `trust_anchors` deployment requires automation to synchronize a server’s DNS and TLS configuration. {{?I-D.ietf-tls-wkech}} could be a starting point for this automation. In deployments that are limited by server changes, this may be a more difficult transition.

* Trust Expressions require that CAs continually fetch information from manifests that are published by root programs, while `trust_anchors` relies only on static pre-assigned trust anchor identifiers.

* `trust_anchors`, when trust anchors are conditionally sent, has different fingerprinting properties. See {{privacy-considerations}}.

* `trust_anchors` can only express large client trust stores (for server certificates), not large server trust stores. Large trust stores rely on the retry mechanism described in {{retry-mechanism}}, which is not available to client certificates.

The two mechanisms can be deployed together. A subscriber can have metadata for both mechanisms available, and a relying party can advertise both.

[[TODO: remove this or move to supporting documentation if this becomes the canonical thing]]

# Acknowledgements
{:numbered="false"}

The authors thank Nick Harper, and Emily Stark for many valuable discussions and insights which led to this document.
