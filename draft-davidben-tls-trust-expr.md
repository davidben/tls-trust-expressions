---
title: "TLS Trust Expressions"
category: std

docname: draft-davidben-tls-trust-expr-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
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
  latest: "https://davidben.github.io/tls-trust-expressions/draft-davidben-tls-trust-expr.html"

author:
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

 -
    ins: "B. Beck"
    name: "Bob Beck"
    organization: "Google LLC"
    email: bbe@google.com

normative:
  POSIX: DOI.10.1109/IEEESTD.2018.8277153

  X690:
       title: "Information technology - ASN.1 encoding Rules: Specification of Basic Encoding Rules (BER), Canonical Encoding Rules (CER) and Distinguished Encoding Rules (DER)"
       date: 2002
       author:
         org: ITU-T
       seriesinfo:
         ISO/IEC: 8825-1:2002

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

  CCADB:
    title: Common CA Database
    target: https://www.ccadb.org/
    date: 2023-10-09
    author:
    -
      org: "Mozilla"
    -
      org: "Microsoft"
    -
      org: "Google"
    -
      org: "Apple"
    -
      org: "Cisco"

--- abstract

This document defines TLS trust expressions, a mechanism for relying parties to succinctly convey trusted certification authorities to subscribers by referencing named and versioned trust stores. It also defines supporting mechanisms for subscribers to evaluate these trust expressions, and select one of several available certification paths to present. This enables a multi-certificate deployment model, for a more agile and flexible PKI that can better meet security requirements.

--- middle

# Introduction

TLS {{!RFC8446}} endpoints typically authenticate using X.509 certificates {{!RFC5280}}. These are used as assertions by a certification authority (CA) that associate some TLS key with some DNS name or other identifier. If the peer trusts the CA, it will accept this association. The authenticating party is known as the subscriber and the peer is the relying party.

Subscribers typically provision a single certificate for all supported relying parties, because relying parties do not communicate which CAs are trusted. This certificate must then simultaneously meet requirements for all relying parties.

This constraint imposes costs on the ecosystem as PKIs evolve over time. The older the relying party, the more its requirements may have diverged from newer ones, making it increasingly difficult for subscribers to support both. This translates to analogous costs for CAs and relying parties:

* For a new CA to be usable by subscribers, it must be trusted by all relying parties. This is particularly challenging for older, unupdatable relying parties. Existing CAs face similar challenges when rotating or deploying new keys.

* When a relying party must update its policies to meet new security requirements, it must choose between compromising on user security or imposing a significant burden on subscribers that still support older relying parties.

This document aims to remove this constraint with a multi-certificate deployment model. Subscribers are instead provisioned with multiple certificates and automatically select the correct one to use with each relying party. This allows a single subscriber to use different certificates for different relying parties, including older and newer ones.

This model requires the endpoints to somehow negotiate which certificate to use. {{Section 4.2.4 of RFC8446}} defines the `certificate_authorities` extension, but it is often impractical. Some trust stores are large, and the X.509 Name structure is inefficient. For example, as of August 2023, the Mozilla CA Certificate Program {{MOZILLA-ROOTS}} would encode 144 names totaling 14,457 bytes. Instead, this document defines a TLS extension and supporting mechanisms that allow relying parties and subscribers to succinctly communicate supported trust anchors to subscribers, using pre-shared information provisioned by root programs and CAs, respectively.

Together, these mechanisms enable subscribers to deploy multiple certificates, which supports a more flexible, robust Public Key Infrastructure (PKI). {{use-cases}} discusses several deployment use cases that benefit from this model.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

This document additionally uses the TLS presentation language defined in {{Section 3 of !RFC8446}}.

## Time

Times in this document are represented by POSIX timestamps, which are integers containing a number of seconds since the Epoch, defined in Section 4.16 of {{POSIX}}. That is, the number of seconds after 1970-01-01 00:00:00 UTC, excluding leap seconds. A UTC time is converted to a POSIX timestamp as described in {{POSIX}}.

Durations of time are integers, representing a number of seconds not including leap seconds. They can be added to POSIX timestamps to produce other POSIX timestamps.

The current time is a POSIX timestamp determined by converting the current UTC time to seconds since the Epoch as per Section 4.16 of {{POSIX}}. One POSIX timestamp is said to be before (respectively, after) another POSIX timestamp if it is less than (respectively, greater than) the other value.

## Terminology and Roles

This document discusses four roles:

Subscriber:
: The party whose identity is being validated in the protocol. In TLS, this is the side sending the Certificate and CertificateVerify message.

Relying party:
: The party that authenticates the subscriber. In TLS, this is the side that validates a Certificate and CertificateVerify message.

Certification authority (CA):
: The service that issues certificates to the subscriber.

Root program:
: An entity that manages a set of common trusted CAs for a set of relying parties.

While the first three roles are common to most X.509 deployments, this document discusses the role of a root program as distinct from the relying party. Trust decisions are often common between related relying parties, such as multiple clients running the same application. The root program is the entity making those common decisions, such as the software vendor for the application or operating system.

Additionally, there are several terms used throughout this document to describe this proposal:

Trust anchor:
: A pre-distributed public key or certificate that relying parties use to determine whether a certification path is trusted.

Trust store:
: A collection of trust anchors managed by the root program.

Certification path:
: An ordered list of X.509 certificates starting the target certificate. Each certificate is issued by the next certificate, except the last, which is issued by a trust anchor.

Trust store manifest:
: A structure describing a series of versioned trust stores.

Trust expression:
: A compact description of a relying party's trust store contents, referencing named, versioned trust stores.

CertificatePropertyList:
: A structure associated with a certification path, containing additional information from the CA, for use by the subscriber when presenting the certification path.

TrustStoreInclusionList:
: A property in the CertificatePropertyList which describes the corresponding trust anchor's inclusion in trust stores. This is used by the subscriber when evaluating trust expressions.

# Overview

In the TLS handshake, a relying party sends trust expressions, which reference named, versioned trust stores to describe a list of trust anchors. The subscriber uses this information to select a certification path to return.

These structures are intended to be provisioned by root programs and CAs as follows:

* Root programs maintain versions of a trust store over time in a trust store manifest, which is published to a well-known location. (See {{trust-store-manifests}}.)

* CAs regularly fetch trust store manifests. CAs use these manifests to associate TrustStoreInclusionList structures with issued certification paths. This communicates the trust anchor's inclusion status to subscribers. (See {{trust-store-inclusion}}.)

* The CA issues multiple certification paths to subscribers, which together cover all supported relying parties. The provisioning process is expected to be automated. (See {{issuing-certificates}}).

* When updating a relying party's list of supported trust anchors, root programs also send a compact description as a TrustExpressionList. (See {{constructing-trust-expressions}}.)

* During a TLS handshake, relying parties send their TrustExpressionList to subscribers. (See {{trust-expressions}}.)

* Subscribers select certification paths by evaluating the TrustExpressionList with each path's TrustStoreInclusionList. Subscribers are then assured that all supported relying parties will be satisfied by some available path. (See {{subscriber-behavior}}.)

By providing accurate trust anchor negotiation, this process avoids the need for relying parties to perform path-building {{!RFC4158}}. Certification paths can be pre-constructed to end at one of the relying party's supported anchors.

# Trust Store Manifests

This section defines a trust store manifest, which is a structure published by the root program to some well-known location. These manifests are used to compute certificate properties for the subscriber ({{certificate-properties}}) and trust expressions for the relying party ({{tls-certificate-negotiation}}).

Trust store manifests are identified by a short, unique trust store name and define a list of trust store versions. A trust store version contains a set of trust anchors, and is identified by a name (from the manifest) and a version number.

[[TODO: Trust store names need to be unique, or at least unique among relying parties and subscribers that talk to each other, but also short. How do we allocate them? Registry? Just pick values? OIDs?]]

A trust store manifest is a JSON object {{!RFC8259}} containing:

`name`:
: A string containing a short, unique name that identifies the collection of trust stores

`max_age`:
: A non-negative integer containing the number of seconds that this document may be cached.

`trust_anchors`:
: A non-empty object describing a set of trust anchors. Each member's name is a short identifier for the trust anchor, used in `entries` below. Each member's value is an object describing the trust anchor, containing:

  `type`:
  : A string describing the type of trust anchor.

  In this document, the value of `type` is always "x509". In this case, the object additionally contains a member `data` whose value is a string containing a base64-encoded {{!RFC4648}}, DER-encoded {{X690}} X.509 certificate.

  Future documents may extend this format with other trust anchor types. Recipients MUST ignore trust anchors with unrecognized `type`.

`versions`:
: A non-empty array describing versions of the trust store. Each element of the array is a JSON object containing:

  `timestamp`:
  : An integer, which is the time at which this trust store version was defined, as a POSIX timestamp (see {{time}}).

  `entries`:
  : A non-empty array describing the contents of this version of the trust store. These are known as trust store entries. Each element is an object containing:

    `id`:
    : A string containing the name of some member of the `trust_anchors` object.

    `labels`:
    : A non-empty array of non-negative, 24-bit integer labels associated with the trust anchor. See {{labels}} for how this field is interpreted.

    `max_lifetime`:
    : A non-negative integer containing the maximum lifetime, in seconds, of certification paths that this trust anchor is expected to issue.

Versions in `versions` are identified by their zero-indexed position in the array. That is, the first entry is version 0, the second is version 1, and so on.

Recipients MUST ignore JSON object members with unrecognized names in each of the objects defined above. Future documents MAY extend these formats with new names. {{cddl-schema}} contains a CDDL {{?RFC8610}} describing the above structure.

When updating a trust store manifest, root programs append a new object to the `versions` array to represent the new trust store version. When the new object references a trust anchor, the root program uses the existing members of `trust_anchors`, or adds new members as required. Updates MUST NOT modify or remove existing entries in the `versions` array.

## Trust Store Entry Expiration {#expiration}

The `max_age` and `max_lifetime` fields define an expiration time for trust store entries in versions other than the latest version. For a trust store entry in version V, the expiration time is the sum of:

* The `timestamp` of the subsequent version, i.e. version V+1
* The `max_age` of the manifest
* The `max_lifetime` of the trust store entry

Expiration times for entries in the latest version are not defined. They are determined once the root store appends a new version.

Trust store entries are not removed from their containing version after they expire. Rather, the expiration time is the point at which all unexpired certificates have incorporated information about subsequent trust store versions, per {{computing-trust-store-inclusions}}. This ensures the negotiation procedures in {{evaluating-trust-expressions}} and {{constructing-trust-expressions}} interoperate.

## Labels

Trust store entries reference labels, which are 24-bit integers that identify subsets of a trust store version. These integers are defined relative to the trust store manifest and are not globally unique. Root programs allocate labels when defining versions of a trust store.

Labels are used in trust expressions ({{trust-expressions}}) to exclude trust anchor entries from the expression. A label may identify an individual trust anchor if it is the only one with that label, or multiple trust anchors if they share the label.

The root program's label allocation scheme determines which sets may be efficiently represented. In particular, when trust anchors are removed in a later version of a trust store, trust expressions must express the removed set with labels as described in {{constructing-trust-expressions}}. Root programs SHOULD allocate individual labels for each trust anchor, and shared labels for trust anchors that will likely be removed together. For example, the root program may allocate shared labels for trust anchors with the same operator, or trust anchors that use some signature algorithm if it expects some relying parties to exclude by algorithm.

When allocating labels, root programs MAY repurpose a previously allocated label value after all previous entries referencing it have expired ({{expiration}}).

# Certificate Properties

In order to evaluate references to trust stores, e.g. in {{trust-expressions}}, subscribers require information about which trust stores contain each candidate certification path's trust anchor. This document introduces an extensible CertificatePropertyList structure to carry this information.

CertificatePropertyLists are constructed by CAs and sent to subscribers, alongside the certification path itself. They contain a list of properties the subscriber may use when presenting the certificate, e.g. as an input to certificate negotiation ({{subscriber-behavior}}).

A CertificatePropertyList is defined using the TLS presentation language ({{Section 3 of !RFC8446}}) below:

~~~
enum { trust_stores(0), (2^16-1) } CertificatePropertyType;

struct {
    CertificatePropertyType type;
    opaque data<0..2^16-1>;
} CertificateProperty;

CertificateProperty CertificatePropertyList<0..2^16-1>;
~~~

The entries in a CertificatePropertyList MUST be sorted numerically by `type` and MUST NOT contain values with a duplicate `type`. Inputs that do not satisfy these invariants are syntax errors and MUST be rejected by parsers.

This document defines a single property, `trust_stores`, which describes trust store inclusion. Future documents may define other properties for use with other mechanisms.

Subscribers MUST ignore unrecognized properties with CertificatePropertyType values. If ignoring a property would cause such subscribers to misinterpret the structure, the document defining the CertificatePropertyType SHOULD include a mechanism for CAs to negotiate the new property with the subscriber in certificate management protocols such as ACME {{?RFC8555}}.

## Trust Store Inclusion

The `trust_stores` property specifies which trust stores contain the certification path's trust anchor. Its `data` field contains a TrustStoreInclusionList, defined below:

~~~
enum {
    previous_version(0),
    latest_version_at_issuance(1)
} TrustStoreStatus;

struct {
    opaque name<1..2^8-1>;
    uint24 version;
} TrustStore;

struct {
    TrustStore trust_store;
    TrustStoreStatus status;
    uint24 labels<1..2^16-1>;
} TrustStoreInclusion;

TrustStoreInclusion TrustStoreInclusionList<1..2^16-1>;
~~~

Each TrustStoreInclusion describes a trust store version which contains this certification path's trust anchor. `trust_store` specifies a version of the trust store, and `labels` specifies the trust anchor's labels within that trust store.

If `status` is `latest_version_at_issuance`, `trust_store` was the latest trust store of its name at the time the certificate was issued. In this case, the trust expression evaluation algorithm ({{evaluating-trust-expressions}}) predicts this information additionally applies to all versions greater than `trust_store`'s `version`, up to the expiration of the certification path.

A TrustStoreInclusionList MUST satisfy the following invariants:

* Each TrustStoreInclusion has a unique `trust_store` value.

* The TrustStoreInclusion structures are sorted, first by length of `trust_store`'s `name`, then by `trust_store`'s `name` lexicographically, then by `trust_store`'s `version`.

* If `status` is `latest_version_at_issuance` in some TrustStoreInclusion, no `trust_store` with the same `name` but higher `version` appears in the list.

## Computing Trust Store Inclusions

Each CA regularly fetches trust store manifests from root programs in which it participates, caching the most recently fetched manifest from each root program. When issuing a certification path to a subscriber, it runs the following procedure on each root program's trust store manifest, to compute a list of TrustStoreInclusions:

1. If the cached manifest was fetched more than `max_age` seconds ago, fetch and cache a new copy.

2. For each trust store version in the cached manifest's `versions`:

   {: type="a"}
   1. Look for a trust store entry whose `id`, when indexed into `trust_anchors`, matches the certification path's trust anchor.

   2. If found, output a TrustStoreInclusion structure:

      * Set `trust_store` to the manifest's `name`, encoded in UTF-8 {{!RFC3629}}, and the version's version number.
      * Set `status` to `latest_version_at_issuance` if the trust store version is currently the latest version. Otherwise, set it to `previous_version`.

      * Set `labels` to the trust store entry's `labels`.

If the above procedure outputs a TrustStoreInclusion with `status` of `latest_version_at_issuance`, the certification path's lifetime MUST NOT exceed the corresponding trust store entry's `max_lifetime` field. This ensures the procedures in {{evaluating-trust-expressions}} and {{constructing-trust-expressions}} interoperate correctly.

The CA combines the outputs of this procedure for each known manifest to form the final TrustStoreInclusionList. To aid the CA in collecting manifests, repositories such as {{CCADB}} can publish a directory of trust store manifests from participating root programs.

## Media Type

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

# TLS Certificate Negotiation

This section defines the `trust_expressions` extension, which is sent in the ClientHello, CertificateRequest, and Certificate messages.

## Trust Expressions

When the `trust_expressions` extension is sent in ClientHello and CertificateRequest, the `extension_data` field is a TrustExpressionList, defined below:

~~~
enum { trust_expressions(TBD), (2^16-1) } ExtensionType;

struct {
    TrustStore trust_store;
    uint24 excluded_labels<0..2^16-1>;
} TrustExpression;

TrustExpression TrustExpressionList<1..2^16-1>;
~~~

A TrustExpressionList is an unordered set of TrustExpression objects. When a relying party sends a TrustExpressionList, it indicates support for all trust anchors specified by any TrustExpression contained in the list. A TrustExpression specifies a list of trust anchors in two parts:

First, `trust_store` specifies a trust store by name and version (see {{trust-store-manifests}}). Any trust anchors contained in the specified trust store are included.

Second, `excluded_labels` specifies a set of labels, each of which identify one or more trust anchors in a trust store manifest. Any trust anchors identified by any label in `excluded_labels` are excluded. {{constructing-trust-expressions}} discusses this set in more detail.

`excluded_labels` MUST be sorted in ascending order and contain no duplicate values to be valid. If invalid, receivers MUST terminate the connection with an `illegal_parameter` alert.

Together, a TrustExpression indicates that the relying party accepts all trust anchors in the referenced trust store version, except for any that were excluded via `excluded_labels`.

## Subscriber Acknowledgement

When the `trust_expressions` extension is sent in a Certificate message, the extension MUST be in the first CertificateEntry and the `extension_data` field MUST be empty. This extension indicates the subscriber selected a certification path using trust expressions.

In this case, the `certificate_list` flexibility described in {{Section 4.4.2 of !RFC8446}} no longer applies. The `certificate_list` MUST contain a complete certification path, correctly ordered and with no extraneous certificates. That is, each certificate MUST certify the one immediately preceding it, and the trust anchor MUST certify the final certificate.

If a relying party receives this extension in the Certificate message, it SHOULD NOT use path building {{!RFC4158}} to validate the result.

## Evaluating Trust Expressions

Given a certification path with a `trust_store` certificate property ({{trust-store-inclusion}}), a subscriber can evaluate a TrustExpressionList to determine whether the certification path matches.

A TrustExpression is said to match a TrustStoreInclusionList if there is at least one TrustStoreInclusion in the TrustStoreInclusionList such that all of the following are true:

* The `name` fields of two structures' `trust_store` fields are equal.

* Either the `version` fields of the two structures' `trust_store` fields are equal, or the TrustStoreInclusion's `status` is `latest_version_at_issuance` and the `version` field of TrustExpression's `trust_store` is greater than that of the TrustStoreInclusion's `trust_store`.

* There is no value in common between the TrustStoreInclusion's `labels` field and the TrustExpression's `excluded_labels` field.

The invariants in {{trust-store-inclusion}} imply that at most one TrustStoreInclusion will satisfy the first two properties. Implementations may evaluate this efficiently by performing a binary search over the TrustStoreInclusionList, and then checking the third property.

A TrustExpressionList is said to match a TrustStoreInclusionList if any TrustExpression in the TrustExpressionList matches the TrustStoreInclusionList.

When a TrustStoreInclusion's `status` is `latest_version_at_issuance`, the above criteria predicts that trust anchors in the latest version will continue to be present in later versions. This allows relying parties using later trust store versions to continue to interoperate with certification paths that predate the later version. If this prediction is incorrect and the trust anchor has been removed from the later version, {{constructing-trust-expressions}} requires that the TrustExpression include appropriate `excluded_labels` values to mitigate this.

## Subscriber Behavior

Subscribers using this negotiation scheme are configured with a list of certification paths, with corresponding CertificatePropertyList ({{certificate-properties}}) structures, in some preference order. When responding to a ClientHello (as a server) or CertificateRequest (as a client) containing the `trust_expressions` extension, the subscriber collects all candidate certification paths such that all of the following are true:

* The certification path has not expired.

* The CertificatePropertyList has a `trust_store` property with a TrustStoreInclusionList.

* The matching algorithm described in {{evaluating-trust-expressions}} returns true.

* The certification path is suitable for use based on other TLS criteria. For example, the TLS `signature_algorithms` ({{Section 4.2.3 of RFC8446}}) extension constrains the types of keys which may be used.

* The certification path satisfies any other application-specific constraints which may apply. For example, TLS servers often select certificates based on the `server_name` ({{Section 3 of ?RFC6066}}) extension.

Once all candidate paths are determined, the subscriber picks one to present to the relying party. The subscriber MUST include an empty `trust_expressions` extension in the first CertificateEntry. If multiple candidates match, the subscriber picks its most preferred option. For example, it may try to minimize message size, or prefer options with more performant private keys.

If no candidates match, or if the peer did not send a `trust_expressions` extension, the subscriber falls back to preexisting behavior outside the scope of this document, without including a `trust_expressions` extension. For example, the subscriber may have a default certificate configured, or select a certificate using the `certificate_authorities` extension.

## Constructing Trust Expressions

Relying parties send the `trust_expressions` extension to advertise a set of accepted trust anchors. {{privacy-considerations}} discusses which trust anchors to advertise. Each trust expression to be sent advertises a subset of some trust store version. To communicate a set of trust anchors that span multiple trust stores, relying parties MAY send multiple trust expressions, each referencing a different trust store.

For each referenced trust store version, the following procedure constructs a trust expression. This procedure is expected to be run by the root program, as part of defining new trust store versions and provisioning supported trust anchors in relying parties.

[[TODO: This is written as a procedure, but we expect root programs to fill in the expected exclusions when they define a new trust store version, and then trim the compatibility exclusions as they expire. Also the root programs know their label allocation scheme and are the ones deciding on removals, so they're best situated to pick a set of `excluded_labels`. Perhaps this should get moved to the manifest?]]

1. Let `include_entries` and `exclude_entries` be two empty sets of trust anchor entries.

2. For each trust store entry in the trust store version:

   {: type="a"}
   1. If the trust store entry's `id` references a trust anchor that is in the desired subset, add it to `include_entries`.

   2. Otherwise, add it to `exclude_entries`.

3. For all trust store entries in trust store versions before the specified version:

   {: type="a"}
   1. If the current time is before the entry's expiration time ({{expiration}}) and if the entry's `id` references a trust anchor that is not in the desired subset, add the entry to `exclude_entries`.

4. Compute a set of labels, `excluded_labels` such that:

   {: type="a"}
   1. No label appears in any entry in `include_entries`.
   2. For each entry in `exclude_entries`, there is some label in common between `excluded_labels` and the labels in the entry.

   How to compute this set depends on the root program's label allocation scheme ({{labels}}). If the root program allocates a label for each trust anchor, this set is always computable, though more efficient sets may be possible depending on the allocation scheme.

5. Output a TrustExpression such that `trust_store` is the referenced trust store version, and `excluded_labels` is the computed `excluded_labels` value.

This procedure uses `excluded_labels` for two kinds of exclusions:

First, if the trust store version includes undesired trust anchors, the trust expression should exclude them. This may occur if, for example, the trust store is used by all versions of the relying party's software, but some trust anchors are gated by software version.

Second, trust expressions exclude unexpired entries from previous versions. This is because the matching criteria described in {{evaluating-trust-expressions}} predictively applies TrustStoreInclusion values with `status` of `latest_version_at_issuance` to all future versions of a trust store. This allows relying parties to interoperate with subscribers with stale information. Unexpired entries are those for which such an unexpired certification path may still exist. Where this prediction is incorrect, trust expressions MUST mitigate this by excluding the past entries.

# Issuing Certificates

Subscribers SHOULD use an automated issuance process where the CA transparently provisions multiple certification paths, without changes to subscriber configuration. As relying party requirements evolve, the CA adjusts its output to ensure its subscribers continue to interoperate with all supported relying parties. This results in a more flexible and agile PKI that can better respond to security incidents and changes in requirements. (See {{use-cases}} for details.)

Subscribers MAY additionally combine the outputs from multiple CAs. This may be used, for example, to maintain backup certificates as described in {{backup-certificates}}.

## ACME Extension

This section extends ACME {{!RFC8555}} to be able to issue certificate paths, each with an associated CertificatePropertyList by defining a new media type in {{media-type}}.

When an ACME server processes an order object, it MAY issue multiple certification paths, each with an associated TrustStoreInclusionList, computed as in {{computing-trust-store-inclusions}}. These paths MAY share an end-entity certificate but end at different trust anchors, or they MAY be completely distinct. The ACME server encodes each certification path using the application/pem-certificate-chain-with-properties format, defined in {{media-type}}).  Note this format is required to form a complete certification path. The CA MUST return a result that may be verified by relying parties without path building {{?RFC4158}}.

The ACME server provides additional results to the client with the link relation header fields described in {{Section 7.4.2 of !RFC8555}}. When fetching certificates, the ACME client includes application/pem-certificate-chain-with-properties in its Accept header to indicate it supports the new format. Unlike the original mechanism described in {{RFC8555}}, these certification paths do not require heuristics to be used. Instead, the ACME client uses the associated TrustStoreInclusionLists to select a path as described in {{subscriber-behavior}}.

When the ACME client wishes to renew any of the certification paths issued in this order, it repeats this process to renew each path concurrently. Thus this extension is suitable when the CA is willing to issue and renew all paths together. It may not be appropriate if the paths have significantly different processing times or lifetimes. Future enhancements to ACME may be defined to address these cases, e.g. by allowing the ACME client to make independent orders.

# Example

Suppose A1, A2, B1, B2, C1, and C2 are X.509 root certificates, such that A1 and A2 share an operator, B1 and B2 share an operator, and C1 and C2 share an operator.

On January 1st, 2023, the root program includes A1, A2, B1, and B2. It allocates labels 0 through 3 for each individual CA and then 100 and 101 for the two CA operators. The CAs issue 90 day certificates and are permitted to use manifests stale by 10 days. The root store manifest may then be:

~~~
{
  "name": "example",
  "max_age": 864000,
  "trust_anchors": {
    "A1": {"type": "x509", "data": "..."},
    "A2": {"type": "x509", "data": "..."},
    "B1": {"type": "x509", "data": "..."},
    "B2": {"type": "x509", "data": "..."}
  },
  "versions": [
    {
      "timestamp": 1672531200,
      "entries": [
        {"id": "A1", "labels": [0, 100],
         "max_lifetime": 7776000},
        {"id": "A2", "labels": [1, 100],
         "max_lifetime": 7776000},
        {"id": "B1", "labels": [2, 101],
         "max_lifetime": 7776000},
        {"id": "B2", "labels": [3, 101],
         "max_lifetime": 7776000}
      ]
    }
  ]
}
~~~

A certification path, A1_old, issued by A1, would have a TrustStoreInclusion:

* `trust_store.name` is "example"
* `trust_store.version` is 0
* `status` is `latest_version_at_issuance`
* `labels` is 0, 100

A certification path, B1_old, issued by B1, would have a TrustStoreInclusion:

* `trust_store.name` is "example"
* `trust_store.version` is 0
* `status` is `latest_version_at_issuance`
* `labels` is 2, 101

A certification path, C1_old, issued by C1, would have no TrustStoreInclusions that reference "example".

On February 1st, 2023, the root program added CAs C1 and C2 but removed CAs B1 and B2. It continues the previous label allocation scheme, but now wishes to allocate label 200 for CAs A1 and C1. The manifest may then be:

~~~
{
  "name": "example",
  "max_age": 864000,
  "trust_anchors": {
    "A1": {"type": "x509", "data": "..."},
    "A2": {"type": "x509", "data": "..."},
    "B1": {"type": "x509", "data": "..."},
    "B2": {"type": "x509", "data": "..."},
    "C1": {"type": "x509", "data": "..."},
    "C2": {"type": "x509", "data": "..."}
  },
  "versions": [
    {
      "timestamp": 1672531200,
      "entries": [
        {"id": "A1", "labels": [0, 100],
         "max_lifetime": 7776000},
        {"id": "A2", "labels": [1, 100],
         "max_lifetime": 7776000},
        {"id": "B1", "labels": [2, 101],
         "max_lifetime": 7776000},
        {"id": "B2", "labels": [3, 101],
         "max_lifetime": 7776000}
      ]
    },
    {
      "timestamp": 1675209600,
      "entries": [
        {"id": "A1", "labels": [0, 100, 200],
         "max_lifetime": 7776000},
        {"id": "A2", "labels": [1, 100],
         "max_lifetime": 7776000},
        {"id": "C1", "labels": [4, 102, 200],
         "max_lifetime": 7776000},
        {"id": "C2", "labels": [5, 102],
         "max_lifetime": 7776000}
      ]
    }
  ]
}
~~~

A certification path, A1_new, newly issued by A1, would have two TrustStoreInclusions. The first:

* `trust_store.name` is "example"
* `trust_store.version` is 0
* `status` is `previous_version`
* `labels` is 0, 100

And the second:

* `trust_store.name` is "example"
* `trust_store.version` is 1
* `status` is `latest_version_at_issuance`
* `labels` is 0, 100, 200

A certification path, B1_new, newly issued by B1, would have a TrustStoreInclusion:

* `trust_store.name` is "example"
* `trust_store.version` is 0
* `status` is `previous_version`
* `labels` contains 2, 101

A certification path, C1_new, newly issued by C1, would have a TrustStoreInclusion:

* `trust_store.name` is "example"
* `trust_store.version` is 1
* `status` is `previous_version`
* `labels` contains 2, 101, 200

A relying party which trusts trust anchors A1, A2, B1, and B2 might send a TrustExpression referencing trust store "example", version 0, with empty `excluded_labels`. This would match A1_old, A1_new, B1_old, and B1_new by the corresponding TrustStoreInclusion. It would not match C1_old or C1_new.

A relying party which trusts trust anchors A2, B1, and B2, but not A1, might send a TrustExpression referencing trust store "example", version 0, with `excluded_labels` of 0. This would match B1_old and B1_new, but not A1_old or A1_new because the TrustExpression excludes A1.

A relying party which trusts trust anchors A1, A2, C1, and C2 might send a TrustExpression referencing trust store "example", version 1, with `excluded_labels` of 101. Although B1 and B2 are not contained in version 1 of the trust store, the relying party must exclude them, per {{constructing-trust-expressions}}. This TrustExpression matches the above certification paths as follows:

* A1_old matches. Although it has no version 1 TrustStoreInclusion, the version 0 TrustStoreInclusion has a `status` of `latest_version_at_issuance`.

* A1_new matches via its version 1 TrustStoreInclusion.

* B1_old does not match. Although its version 0 TrustStoreInclusion has a `status` of `latest_version_at_issuance` and thus applies, `excluded_labels` excludes it.

* B1_new does not match. Its version 0 TrustStoreInclusion has a `status` of `previous_version` and does not apply. It has no version 1 TrustStoreInclusion.

* C1_old does not match. Although the relying party trusts C1, C1_old was issued before C1 was able to communicate this to the subscriber.

* C1_new matches via its version 1 TrustStoreInclusion.

The relying party could equivalently have sent a TrustExpression referencing trust store "example", version 1, with `excluded_labels` of 2 and 3. This pair of labels excludes the same CAs as 101.

Of the above examples, B1_old depends on the exclusion to avoid a false positive match. 100 days after February 1st, B1 and B2's trust store entries from version 0 will expire (see {{expiration}}), and the relying party no longer needs to exclude them. By this point, B1_old will have expired. B1_new may still be valid, but it does not require the exclusion.

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

A CA operator could provide subscribers with two certification paths: a longer path ending at a long-lived trust anchor and shorter path the other ending at a short-lived, revocable root. Relying parties would be configured to trust both the long-lived root and the most recent short-lived root. The negotiation mechanism in {{tls-certificate-negotiation}} would then send the shorter path to up-to-date relying parties, and the longer path to older relying parties.

This achieves the same effect with a more general-purpose multi-certificate mechanism. It is also more flexible, as the two paths need not be related. For example, root CA public keys are not distributed in each TLS connection, so a post-quantum signature algorithm that optimizes for signature size may be preferable. In this model, both the long-lived and short-lived roots may use such an algorithm. In a compression-based model, the same intermediate must optimize both its compressed and uncompressed size, so such an algorithm may not be suitable.

## Conflicting Relying Party Requirements

A subscriber may need to support relying parties with different requirements. For example, in contexts where online revocation checks are expensive, unreliable, or privacy-sensitive, user security is best served by short-lived certificates. In other contexts, long-lived certificates may be more appropriate for, e.g., systems that are offline for long periods of time or have unreliable clocks.

A single-certificate deployment model forces subscribers to find a single certificate that meets all requirements. User security then suffers in all contexts, as the PKI may not quite meet anyone's needs. In a multi-certificate deployment model, different contexts may use different trust anchors. A subscriber that supports multiple contexts would provision certificates for each, with certificate negotiation logic directing the right one to each relying party.

## Backup Certificates

A subscriber may obtain certificate paths from multiple CAs for redundancy in the face of future CA compromises. If one CA is compromised and removed from newer relying parties, the TLS server software will transparently serve the other one.

To support this, TLS serving software SHOULD permit users to configure multiple ACME endpoints and select from the union of the certificate paths returned by each ACME server.

# Privacy Considerations

The negotiation mechanism described in this document is analogous to the `certificate_authorities` extension ({{Section 4.2.4 of RFC8446}}), but more size-efficient. Like that extension, it presumes the advertised trust anchor list is not sensitive.

Thus, this mechanism SHOULD NOT be used to advertise trust anchors or distrusts which are unique to an individual user. Rather, trust expressions SHOULD be computed based only on the trust anchors common to the relying party's anonymity set ({{Section 3.3 of !RFC6973}}). Additionally, multiple trust expressions may evaluate to the same trust anchor list, so relying parties in the same anonymity set SHOULD send the same trust expression. To achieve this, trust expressions SHOULD be assembled by the root program and configured in relying parties alongside trust store updates.

For example, a web browser may support both a common set of trust anchors configured by the browser vendor, along with user-specified additions and removals. The common trust anchors would reveal, at most, which browser is used, while the user-specified modifications may reveal identifying information about the user. The trust expression SHOULD reflect only the common trust anchors. This limits the benefits of trust anchor agility in two ways:

* If a subscriber relies on a user-specified addition, the procedure in {{subscriber-behavior}} will fallback to preexisting behavior, such as selecting a default certificate. The subscriber then relies on the default certificate matching the relying party.

* If a subscriber has a certificate issued by a CA distrusted by the user, the procedure in {{subscriber-behavior}} may select a distrusted certificate. In that case, the connection will fail, even if the subscriber has another certificate available.

Both of these cases match the preexisting behavior in PKIs that do not use trust expressions.

# Security Considerations

The certificate negotiation mechanism described in this document facilitates which certification path is served to relying parties, but has no impact on the relying party's trust preferences themselves.

As a result, this allows for a more flexible and agile PKI, which can better mitigate security risks to users. {{use-cases}} discusses some scenarios which benefit from the multi-certificate deployment this document enables. In general, robust certificate negotiation helps subscribers navigate differences in relying party requirements. This means security improvements for one set of relying parties can be deployed without needing to risk incompatibility or breakage for others.

If either the subscriber's TrustStoreInclusionList or the relying party's TrustExpressionList are incorrect, the matching algorithm described in {{subscriber-behavior}} may incorrectly identify an untrusted certification path. This mechanism will not result in that path being trusted, but does present the possibility of a denial of service. These structures are provisioned by the CA and root program, respectively, who are already expected to provide accurate information.

# IANA Considerations

## TLS ExtensionType Updates

IANA is requested to create the following entry in the TLS ExtensionType Values registry, defined by {{RFC8446}}:

| Value | Extension Name    | TLS 1.3    | DTLS-Only | Recommended | Reference  |
|-------|-------------------|------------|-----------|-------------|------------|
| TBD   | trust_expressions | CH, CR, CT | N         | Y           | [this-RFC] |

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

# CDDL Schema

The following is a non-normative CDDL {{?RFC8610}} schema which describes a trust store manifest structure ({{trust-store-manifests}}):

~~~
trust-anchor = {
    type: text,
    * text => any,
}

trust-store-entry = {
    id: text,
    labels: [+ uint],
    max_lifetime: uint,
    * text => any,
}

trust-store-version = {
    timestamp: int,
    entries: [+ trust-store-entry],
    * text => any,
}

trust-store-manifest = {
    name: text,
    max_age: uint,
    trust_anchors: {+ text => trust-anchor},
    versions: [+ trust-store-version],
    * text => any,
}
~~~

# Acknowledgements
{:numbered="false"}

The authors thank Nick Harper, Sophie Schmieg, and Emily Stark for many valuable discussions and insights which led to this document, as well as review of early iterations.


