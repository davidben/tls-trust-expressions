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

  CHROME-ROOTS:
    title: Chrome Root Store
    target: https://chromium.googlesource.com/chromium/src/+/main/net/data/ssl/chrome_root_store
    date: 2023-08-30
    author:
    - org: Chromium

  MOZILLA-ROOTS:
    title: Mozilla Included CA Certificate List
    target: https://wiki.mozilla.org/CA/Included_Certificates
    date: 2023-08-30
    author:
    - org: Mozilla

  Dilithium:
    title: CRYSTALS-Dilithium Algorithm Specifications and Supporting Documentation
    target: https://pq-crystals.org/dilithium/data/dilithium-specification-round3-20210208.pdf
    date: 2021-02-08
    author:
    -
      ins: "S. Bai"
      name: "Shi Bai"
    -
      ins: "L. Ducas"
      name: "Léo Ducas"
    -
      ins: "E. Kiltz"
      name: "Eike Kiltz"
    -
      ins: "T. Lepoint"
      name: "Tancrède Lepoint"
    -
      ins: "V. Lyubashevsky"
      name: "Vadim Lyubashevsky"
    -
      ins: "P. Schwabe"
      name: "Peter Schwabe"
    -
      ins: "G. Seiler"
      name: "Gregor Seiler"
    -
      ins: "D. Stehlé"
      name: "Damien Stehlé"

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
when, and only when, they appear in all capitals, as shown here.

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

CertificatePropertyList:
: A structure associated with a certification path, containing additional information from the CA, for use by the subscriber when presenting the certification path.

# Trust Anchor Identifiers {#trust-anchor-ids}

This section defines trust anchor identifiers, which are short, unique identifiers for a trust anchor. To simplify allocation, these identifiers are defined with object identifiers (OIDs) {{X680}} and IANA-registered Private Enterprise Numbers (PENs) {{!RFC9371}}:

A trust anchor identifier is defined with a OID under the OID arc of some PEN. For compactness, they are represented as relative object identifiers (see Section 33 of {{X680}}), relative to the OID prefix `1.3.6.1.4.1`. For example, an organization with PEN 32473 might define a trust anchor identifier with the OID `1.3.6.1.4.1.32473.1`. As a relative object identifier, it would be the OID `32473.1`.

Depending on the protocol, trust anchor identifiers may be represented in one of three ways:

* For use in ASN.1-based protocols, a trust anchor identifier's ASN.1 representation is the relative object identifier described above. This may be encoded in DER {{X690}}, or some other ASN.1 encoding. The example identifier's DER encoding is the six-octet sequence `{0x0d, 0x04, 0x81, 0xfd, 0x59, 0x01}`.

* For use in binary protocols such as TLS, a trust anchor identifier's binary representation consists of the contents octets of the relative object identifier's DER encoding, as described in Section 8.20 of {{X690}}. Note this omits the tag and length portion of the encoding. The example identifier's binary representation is the four-octet sequence `{0x81, 0xfd, 0x59, 0x01}`.

* For use in ASCII-compatible text protocols, a trust anchor identifier's ASCII representation is the relative object identifier in dotted decimal notation. The example identifier's ASCII representation is `32473.1`.

Trust anchor identifiers SHOULD be allocated by the CA operator and common among relying parties that trust the CA. They MAY be allocated by another party, e.g. when bootstrapping an existing ecosystem, if all parties agree on the identifier. In particular, the protocol requires relying parties and subscribers to agree, and subscriber configuration typically comes from the CA.

The length of a trust anchor identifier's binary representation MUST NOT exceed 255 bytes. It SHOULD be significantly shorter, for bandwidth efficiency.

## Certificate Properties {#certificate-properties}

This document introduces an extensible CertificatePropertyList structure for CAs to communicate additional information to subscribers, such as associated trust anchor identifiers. A CertificatePropertyList is defined using the TLS presentation language ({{Section 3 of !RFC8446}}) below:

~~~
enum { trust_anchor_identifier(0), (2^16-1) } CertificatePropertyType;

struct {
    CertificatePropertyType type;
    opaque data<0..2^16-1>;
} CertificateProperty;

CertificateProperty CertificatePropertyList<0..2^16-1>;
~~~

The entries in a CertificatePropertyList MUST be sorted numerically by `type` and MUST NOT contain values with a duplicate `type`. Inputs that do not satisfy these invariants are syntax errors and MUST be rejected by parsers.

This document defines a single property, `trust_anchor_identifier`. The `data` field of the property contains the binary representation of the trust anchor identifier of the certification path’s trust anchor, as described in {{trust-anchor-ids}}. Future documents may define other properties for use with other mechanisms.

Subscribers MUST ignore unrecognized properties with CertificatePropertyType values. If ignoring a property would cause such subscribers to misinterpret the structure, the document defining the CertificatePropertyType SHOULD include a mechanism for CAs to negotiate the new property with the subscriber in certificate management protocols such as ACME {{?RFC8555}}.

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

Each candidate path which participates in this protocol must be provisioned with the trust anchor identifier for its corresponding trust anchor in the CertificatePropertlyList.

Subscribers MAY have candidate certification paths without associated trust anchor identifiers, but such paths will not participate in this protocol. Those paths MAY participate in other trust anchor negotiation protocols, such as the `certificate_authorities` extension.

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

If the subscriber sends a certification path that matches the relying party’s `trust_anchors` extension, as described in {{certificate-selection}}, the subscriber MUST send an empty `trust_anchors` extension in the first CertificateEntry of the Certificate message. In this case, the `certificate_list` flexibility described in {{Section 4.4.2 of !RFC8446}} no longer applies. The `certificate_list` MUST contain a complete certification path, issued by the matching trust anchor, correctly ordered and with no extraneous certificates. That is, each certificate MUST certify the one immediately preceding it, and the trust anchor MUST certify the final certificate. The subscriber MUST NOT send the `trust_anchors` extension in the Certificate message in other situations.

If a relying party receives this extension in the Certificate message, it MAY choose to disable path building {{!RFC4158}} and validate the peer's certificate list as pre-built certification path. Doing so avoids the unpredictable behavior of path-building, and helps ensure CAs and subscribers do not inadvertently provision incorrect paths.

## Retry Mechanism

When the relying party is a client, it may choose not to send its full trust anchor identifier list due to fingerprinting risks (see {{privacy-considerations}}), or because the list is too large. The client MAY send a subset of supported trust anchors, or an empty list. This subset may be determined by, possibly outdated, prior knowledge about the server, such as {{dns-service-parameter}} or past connections.

To accommodate this, when receiving a ClientHello with `trust_anchors`, the server collects all candidate certification paths which:

* Have a trust anchor identifier, and
* Satisfy the conditions in {{Section 4.4.2.2 of RFC8446}}, with the exception of `certification_authorities`, and any future extensions that play a similar role

If this collection is non-empty, the server sends a `trust_anchors` extension in EncryptedExtensions, containing the corresponding trust anchor identifiers in preference order.

When a client sends a subset or empty list in `trust_anchors`, it SHOULD implement the following retry mechanism:

If the client receives either a connection error or an untrusted certificate, the client looks in server’s EncryptedExtensions for a trust anchor identifier that it trusts. If there are multiple, it selects an option based on the server’s preference order and its local preferences. It then makes a new connection to the same endpoint, sending only the selected trust anchor identifier in the ClientHello `trust_anchors` extension. If the EncryptedExtensions had no `trust_anchor` extension, or no match was found, the client returns the error to the application.

Clients SHOULD retry at most once per connection attempt.

[[TODO: Retrying in a new connection is expensive and cannot be done from within the TLS stack in most implementations. Consider handshake modifications to instead retry within the same connection. https://github.com/davidben/tls-trust-expressions/issues/53 ]]

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

This section extends ACME {{!RFC8555}} to be able to issue certificate paths, each with an associated CertificatePropertyList by defining a new media type in {{media-type}}.

When an ACME server processes an order object, it MAY issue multiple certification paths, each with an associated Trust Anchor Identifier. The ACME server encodes each certification path using the application/pem-certificate-chain-with-properties format, defined in {{media-type}}).  Note this format is required to form a complete certification path. The CA MUST return a result that may be verified by relying parties without path building {{?RFC4158}}.

The ACME server provides additional results to the client with the link relation header fields described in {{Section 7.4.2 of !RFC8555}}. When fetching certificates, the ACME client includes application/pem-certificate-chain-with-properties in its Accept header to indicate it supports the new format. Unlike the original mechanism described in {{RFC8555}}, these certification paths do not require heuristics to be used. Instead, the server uses the associated Trust Anchor Identifiers to select a path when requested.

When the ACME client wishes to renew any of the certification paths issued in this order, it repeats this process to renew each path concurrently. Thus this extension is suitable when the CA is willing to issue and renew all paths together. It may not be appropriate if the paths have significantly different processing times or lifetimes. Future enhancements to ACME may be defined to address these cases, e.g. by allowing the ACME client to make independent orders.

# Media Type

A certification path with its associated CertificationPropertyList may be represented in a PEM {{!RFC7468}} structure in a file of type "application/pem-certificate-chain-with-properties". Files of this type MUST use the strict encoding and MUST NOT include explanatory text.  The ABNF {{!RFC5234}} for this format is
as follows, where "stricttextualmsg" and "eol" are as defined in
{{Section 3 of !RFC7468}}:

~~~
certchainwithproperties = stricttextualmsg eol stricttextualmsg
                          *(eol stricttextualmsg)
~~~

The first element MUST be the encoded CertificatePropertyList.
The second element MUST be an end-entity certificate.  Each following
certificate MUST directly certify the one preceding it. The certificate representing the trust anchor MUST be omitted from the path.

CertificatePropertyLists are encoded using the "CERTIFICATE PROPERTIES" label. The encoded data is a serialized CertificatePropertyList, defined in {{certificate-properties}}.

Certificates are encoded as in {{Section 5.1 of !RFC7468}}, except DER {{X690}} MUST be used.

The following is an example file with a certification path containing an end-entity certificate and an intermediate certificate.

~~~
-----BEGIN CERTIFICATE PROPERTIES-----
TODO fill in an example
-----END CERTIFICATE PROPERTIES-----
-----BEGIN CERTIFICATE-----
TODO fill in an example
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
TODO fill in an example
-----END CERTIFICATE-----
~~~

The IANA registration for this media type is described in {{media-type-updates}}.

# Use Cases

## Key Rotation

In most X.509 deployments, a compromise of _any_ root CA's private key compromises the entire PKI. Yet key rotation in PKIs is rare. In 2023, the oldest root in {{CHROME-ROOTS}} and {{MOZILLA-ROOTS}} was 25 years old, dating to 1998. Key rotation is challenging in a single-certificate deployment model. As long as any older relying party requires the old root, subscribers cannot switch to the new root, which in turn means relying parties cannot distrust the old root, leaving them vulnerable.

A multi-certificate deployment model avoids these transition problems. Key rotation may proceed as follows:

1. The CA operator generates a new root CA with a separate key, but continues operating the old root CA.

2. Root programs begin trusting the new root CA alongside the old one. They update their root store manifests.

3. When subscribers request certificates, the CA issues certificates from both roots and provisions the subscriber with both certificates.

4. Relying parties send trust expressions that evaluate to either the old or new root, and are served the appropriate option.

5. Once subscribers have been provisioned with new certificates, root programs can safely distrust the old root in new relying parties. The CA operator continues to operate the old root CA for as long as it wishes to serve subscribers that, in turn, wish to serve older relying parties.

This process requires no configuration changes to the subscriber, given an automated, multi-certificate-aware certificate issuance process. The subscriber does not need to know why it received two certificates, only how to select between them for each relying party.

## Adding CAs

In the single-certificate model, subscribers cannot use TLS certificates issued from a new root CA until all supported relying parties have been updated to trust the new root CA. This can take years or more. Some relying parties, such as IoT devices, may never receive trust store updates at all.

As a result, it is very difficult for subscribers that serve a wide variety of relying parties to use a newly-trusted root CA. When trust stores diverge too far, subscribers often must partition their services into multiple TLS endpoints (i.e. different DNS names) and direct different relying parties to different endpoints. Subscribers sometimes resort to TLS fingerprinting, to detect particular relying parties. But, as this repurposes other TLS fields for unintended purposes, this is unreliable and usually requires writing custom service-specific logic.

In a multi-certificate deployment model, subscribers can begin serving certificates from new root CAs without interrupting relying parties that depend on existing ones.

In some contexts, it may be possible to use other fields to select the new CA. For example, post-quantum-capable clients may be detected with the `signature_algorithms` and `signature_algorithms_cert` extensions. However, this assumes all post-quantum CAs are added at the same time. A multi-certificate model avoids this problem and allows for a more gradual deployment of post-quantum CAs.

## Removing CAs

Subscribers in a single-certificate model are limited to CAs in the intersection of their supported relying parties. As newer relying parties remove untrusted CAs over time,the intersection with older relying parties shrinks. Moreover, the subscriber may not even know which CAs are in the intersection. Often, the only option is to try the new certificate and monitor errors. For subscribers that serve many diverse relying parties, this is a disruptive and risky process.

The multi-certificate model removes this constraint. If a subscriber's CA is distrusted, it can continue to use that CA, in addition to a newer one. This removes the risk that some older relying party required that CA and was incompatible with the new one. The mechanisms in this document will select an appropriate certificate for each relying party.

## Other Root Transitions

The mechanisms in this document can aid PKI transitions beyond key rotation. For example, a CA operator may generate a postquantum root CA and use the mechanism in {{acme-extension}} to issue from the classical and postquantum roots concurrently. The subscriber will then, transparently and with no configuration change, serve both. As in {{key-rotation}}, newer relying parties can then remove the classical roots, while older relying parties continue to function.

This same procedure may also be used to transition between newer, more size-efficient signature algorithms, as they are developed.

[[TODO: There's one missing piece, which is that some servers may attempt to parse the signature algorithms out of the certificate chain. See https://github.com/davidben/tls-trust-expressions/issues/9 ]]

## Intermediate Elision

Today, root CAs typically issue shorter-lived intermediate certificates which, in turn, issue end-entity certificates. The long-lived root key is less exposed to attack, while the short-lived intermediate key can be more easily replaced without changes to relying parties.

This operational improvement comes at a bandwidth cost: the TLS handshake includes an extra certificate, which includes a public key, signature, and X.509 metadata. An average X.509 name in the Chrome Root Store {{CHROME-ROOTS}} or Mozilla CA Certificate Program {{MOZILLA-ROOTS}} is around 100 bytes alone. Post-quantum signature algorithms will dramatically shift this tradeoff. Dilithium3 {{Dilithium}}, for example, has a total public key and signature size of 5,245 bytes.

{{?I-D.ietf-tls-cert-abridge}} proposes to predistribute known intermediate certificates to relying parties, as a compression scheme. A multi-certificate deployment model provides another way to achieve this effect. To relying parties, a predistributed intermediate certificate is functionally equivalent to a root certificate. PKIs use intermediate certificates because changing root certificates requires updating relying parties, but predistributed intermediates already presume updated relying parties.

A CA operator could provide subscribers with two certification paths: a longer path ending at a long-lived trust anchor and shorter path the other ending at a short-lived, revocable root. Relying parties would be configured to trust both the long-lived root and the most recent short-lived root. A server that prioritizes the shorter path would then send the shorter path to up-to-date relying parties and the longer path to older relying parties.

This achieves the same effect with a more general-purpose multi-certificate mechanism. It is also more flexible, as the two paths need not be related. For example, root CA public keys are not distributed in each TLS connection, so a post-quantum signature algorithm that optimizes for signature size may be preferable. In this model, both the long-lived and short-lived roots may use such an algorithm. In a compression-based model, the same intermediate must optimize both its compressed and uncompressed size, so such an algorithm may not be suitable.

## Conflicting Relying Party Requirements

A subscriber may need to support relying parties with different requirements. For example, in contexts where online revocation checks are expensive, unreliable, or privacy-sensitive, user security is best served by short-lived certificates. In other contexts, long-lived certificates may be more appropriate for, e.g., systems that are offline for long periods of time or have unreliable clocks.

A single-certificate deployment model forces subscribers to find a single certificate that meets all requirements. User security then suffers in all contexts, as the PKI may not quite meet anyone's needs. In a multi-certificate deployment model, different contexts may use different trust anchors. A subscriber that supports multiple contexts would provision certificates for each, with certificate negotiation logic directing the right one to each relying party.

## Backup Certificates

A subscriber may obtain certificate paths from multiple CAs for redundancy in the face of future CA compromises. If one CA is compromised and removed from newer relying parties, the TLS server software will transparently serve the other one.

To support this, TLS serving software SHOULD permit users to configure multiple ACME endpoints and select from the union of the certificate paths returned by each ACME server.

## Public Key Pinning

To reduce the risk of attacks from misissued certificates, relying parties sometimes employ public key pinning {{?RFC7469}}. This enforces that one of some set of public keys appear in the final certificate path. This effectively reduces a relying party's trust anchor list to a subset of the original set.

As other relying parties in the PKI evolve, the pinning relying party limits the subscriber to satisfy both the pinning constraint and newer constraints in the PKI. This can lead to conflicts if, for example, the pinned CA is distrusted by a newer relying party. The subscriber is then forced to either break the pinning relying party, or break the newer ones.

This protocol reduces this conflict if the pinning relying party uses its effective, reduced trust anchor list. The subscriber can then, as needed, use a certificate from the pinned CA with the pinning relying party, and another CA with newer relying parties.

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

## Relying Party Policies

PKI-based TLS authentication depends on the relying party's certificate policies. If the relying party trusts an untrustworthy CA, that CA can intercept TLS connections made by that relying party by issuing certificates associating the target DNS name, etc., with the wrong TLS key.

This attack vector is available with or without trust anchor negotiation. The negotiation mechanism described in this document allows certificate selection to reflect a relying party's certificate policies. It does not determine the certificate policies themselves. Relying parties remain responsible for trusting only trustworthy CAs, and untrustworthy CAs remain a security risk when trusted.

## Agility

As with other TLS parameters, negotiation reduces a conflict between availability and security, which allows PKIs to better mitigate security risks to users. When relying parties in an existing TLS ecosystem improve their certificate policies, trust anchor negotiation helps subscribers navigate differences between those relying parties and existing relying parties. Each set of requirements may be satisfied without compatibility risk to the other. {{use-cases}} discusses such scenarios in more detail.

Negotiation also reduces pressures on new relying parties to sacrifice user security for compatibility. Suppose a subscriber currently uses some CA, but a relying party deems trusting that CA to pose an unacceptable security risk to its users. In a single-certificate deployment, those subscribers may be unwilling to adopt a CA trusted by the relying party, as switching CAs risks compatibility problems elsewhere. The relying party then faces compatibility pressure and adds this CA, sacrificing user security. However, in a multi-certificate deployment, the subscriber can use its existing CA _in addition to_ another CA trusted by relying party B. This allows the ecosystem to improve interoperability, while still meeting relying party B's user-security goals.

## Incorrect Selection Metadata

If the subscriber has incorrect information about trust anchor identifiers, it may send an untrusted certification path. This will not result in that path being trusted, but does present the possibility of a degradation of service. However, this information is provisioned by the CA, which is already expected to provide accurate information.

## Serving Multiple Certificates

Negotiation reduces compatibility pressures against subscribers choosing to serve certificates from a less common CA, as the subscriber can serve it in addition to other CAs that, together, satisfy all supported relying parties. The CA may even have been distrusted by some relying parties, e.g. if it is needed for older, unupdated relying parties that are still important for the subscriber to support. As discussed in {{use-cases}} and {{agility}}, this deployment model avoids compatibility risks during PKI transitions which mitigate security risks to users.

Serving such certificates does not enable the CA to decrypt or intercept the connection. If a certificate asserts the correct information about the subscriber, notably the correct public key, the subscriber can safely present it, whether or not the issuing CA is otherwise trustworthy. Issuing a certificate for the subscriber's public key does not grant the CA access to the corresponding private key. Conversely, if the attacker already has access to the subscriber's private key, they do not need to be in control of a CA to intercept a connection.

While the choice of CA is a security-critical decision in a PKI, it is the relying party's choice of trusted CAs that determines interceptibility, not the subscriber's choice of certificates to present. If the relying party trusts an attacker-controlled CA, the attacker can intercept the connection independent of whether the subscriber uses that CA or another CA. In both cases, the attacker would need to present a different certificate, with a different public key. Conversely, if the relying party does not trust the attacker's CA, the subscriber's correct but untrusted attacker-issued certificate will not enable the attacker to substitute in a different public key. The ability to intercept a connection via the PKI is determined solely by relying party trust decisions.

Relying parties thus SHOULD NOT interpret the subscriber's choice of CA list as an endorsement of the CA. The subscriber's role is to present a certificate which will convince the relying party of the correct subscriber information. Subscribers do not vet CAs for trustworthiness, only the correctness of their specific, configured certificates and the CA's ability to meet the subscriber's requirements, such as availability, compatibility, and performance. It is the responsibility of the relying party, and its corresponding root program, to determine the set of trusted CAs. Trusting a CA means trusting _all_ certificates issued by that CA, so it is not enough to observe subscribers serving correct certificates. An untrustworthy CA may sign one correct certificate, but also sign incorrect certificates, possibly in the future, that can attack the relying party. Root program management is a complex, security-critical process, the full considerations of which are outside the scope of this document.

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

## Media Type Updates

IANA is requested to create the following entry in the "Media Types" registry, defined in {{!RFC6838}}:

Type name:
: application

Subtype name:
: pem-certificate-chain-with-properties

Required parameters:
: None

Optional parameters:
: None

Encoding considerations:
: 7bit

Security considerations:
: Carries a cryptographic certificate and its associated certificate chain and additional properties. This media type carries no active content.

Interoperability considerations:
: None

Published specification:
: [this-RFC, {{media-type}}]

Applications that use this media type:
: ACME clients and servers, HTTP servers, other applications that need to be configured with a certificate chain

Additional information:
: <dl spacing="compact">
  <dt>Deprecated alias names for this type:</dt><dd>n/a</dd>
  <dt>Magic number(s):</dt><dd>n/a</dd>
  <dt>File extension(s):</dt><dd>.pem</dd>
  <dt>Macintosh file type code(s):</dt><dd>n/a</dd>
  </dl>

Person & email address to contact for further information:
: See Authors' Addresses section.

Intended usage:
: COMMON

Restrictions on usage:
: n/a

Author:
: See Authors' Addresses section.

Change controller:
: IETF

## CertificatePropertyType Registry

[[TODO: Establish a CertificatePropertyType registry.]]

--- back

# Comparison to TLS Trust Expressions

{{?I-D.davidben-tls-trust-expr}} describes Trust Expressions, another trust anchor negotiation mechanism that aims to solve similar problems. The mechanisms differ in the following ways:

* `trust_anchors` is usable by more kinds of PKIs. Trust Expressions express trust anchor lists relative to named “trust stores”, maintained by root programs. Arbitrary lists may not be easily expressible. `trust_anchors` does not have this restriction.

* When used with large trust stores, the retry mechanism in `trust_anchors` requires a new connection. In most applications, this must be implemented outside the TLS stack, so more components must be changed and redeployed. In deployments that are limited by client changes, this may be a more difficult transition. [[TODO: See {{retry-mechanism}} for an alternate retry scheme that avoids this.]]

* Trust Expressions works with static server configuration. An ideal `trust_anchors` deployment requires automation to synchronize a server’s DNS and TLS configuration. {{?I-D.ietf-tls-wkech}} could be a starting point for this automation. In deployments that are limited by server changes, this may be a more difficult transition.

* Trust Expressions require that CAs continually fetch information from manifests that are published by root programs, while `trust_anchors` relies only on static pre-assigned trust anchor identifiers.

* `trust_anchors`, when trust anchors are conditionally sent, has different fingerprinting properties. See {{privacy-considerations}}.

* `trust_anchors` can only express large client trust stores (for server certificates), not large server trust stores. Large trust stores rely on the retry mechanism described in {{retry-mechanism}}, which is not available to client certificates.

The two mechanisms can be deployed together. A subscriber can have metadata for both mechanisms available, and a relying party can advertise both.

[[TODO: remove this or move to supporting documentation after more working group consensus]]

# Acknowledgements
{:numbered="false"}

The authors thank Nick Harper, and Emily Stark for many valuable discussions and insights which led to this document.
