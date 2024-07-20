## Overview

[TLS Trust Expressions](https://datatracker.ietf.org/doc/draft-davidben-tls-trust-expr/) is a proposed IETF draft that defines a way for TLS clients to signal their trust store contents to servers as well as supporting mechanisms that allow TLS servers to use this information to select a trusted certification path for authentication. In response to this proposal, several different scenarios pertaining to government surveillance have been discussed with various proposed arguments regarding the impact of TLS Trust Expressions.

The purpose of this document is to concretely specify these scenarios, anchored by the goals, capabilities, and constraints of a surveillance-motivated government attempting to abuse this technology beyond its intended use so that we can analyze these proposed arguments in greater detail.

**Surveillance Goals:**



1. Passively surveil the TLS-protected content of TLS clients visiting domains of interest
2. Actively intercept (MITM) traffic for surveillance
3. Prevent TLS clients or third parties from detecting surveillance and interception

**Surveillance Capabilities:**



1. Operate a trust anchor and issue TLS certificates from it
2. Mandate that websites that operate within their jurisdiction obtain and provision TLS certificates from their trust anchor
3. Mandate that TLS clients that operate within their jurisdiction add trust in their trust anchor
4. Forbid TLS clients that operate within their jurisdiction from removing trust in their trust anchor, regardless of their root program policies
5. Mandate key escrow of TLS server private keys and TLS secrets for websites that operate within their jurisdiction

In order to narrow the analysis, we make the following assumptions in each of these scenarios pertaining to TLS Trust Expressions deployments and limitations on the capabilities of such a government.

**Assumptions:**



1. TLS Trust Expressions is standardized and deployed in a variety of web server software implementations
2. TLS Trust Expressions is supported by the TLS client in question
3. The government is unable to invent new technology and relies on mechanisms that are already existing and deployed
4. The government is unable to create, maintain, and mandate its own jurisdiction-specific TLS client / browser


## Scenarios


### Scenario 1: TLS client does not add a trust anchor when deployment is mandated

**Scenario Description:**



1. Capability 1: Surveilling government operates and controls a trust anchor
2. Capability 2: This government mandates that websites operating within its jurisdiction obtain and provision TLS certificates from this trust anchor 
3. TLS client <span style="text-decoration:underline;">does not add</span> this trust anchor to its trust store

**Effects of Trust Expressions deployment:**

Support for trust expressions by the TLS client, web server, and a participating trusted CA provides the ability for web servers to negotiate separate trusted TLS certificates if web servers provision them in addition to the government-mandated untrusted certificates.

Trust Expressions does not modify which certificates are trusted by a TLS client. Since both passive surveillance and interception require successful certificate validation, Trust Expressions deployment does not affect the ability to achieve any of the listed surveillance goals.

**Analysis:**

Operating a trust anchor and mandating that websites provision certificates issued from this trust anchor does not grant the ability to surveil users of TLS clients intending to connect to websites under this government’s jurisdiction. Without TLS client trust, none of the surveillance-related goals are enabled by this scenario.



*   Proposed argument 1 discusses the possibility that the TLS client would add trust in a government-operated trust anchor due to the deployment of untrusted certificates, absent a mandate to do so.
*   Proposed arguments 2a and 2b discuss the possibility that a government would impose a trust mandate based on the deployment of untrusted certificates and the ability to negotiate trusted ones.


### Scenario 2: TLS client does not add trust anchor when trust is mandated (distrust allowed)

**Scenario Description:**



*   Capability 1: Surveilling government operates and controls a trust anchor
*   Capability 2: This government mandates that websites operating within its jurisdiction obtain and provision TLS certificates from this trust anchor 
*   Capability 3: This government mandates that TLS clients operating within its jurisdiction add this trust anchor to their trust stores
*   The TLS client <span style="text-decoration:underline;">does not add</span> the trust anchor to its trust store

**Effects of Trust Expressions Deployment:**

Support for trust expressions by the TLS client, web server, and a participating trusted CA provides the ability for web servers to negotiate separate trusted TLS certificates if web servers provision them in addition to the government-mandated untrusted certificates.

Trust Expressions does not modify which certificates are trusted by a TLS client. Since both passive surveillance and interception require successful certificate validation, Trust Expressions deployment does not affect the ability to achieve any of the listed surveillance goals.

**Analysis:**

Without TLS client trust, none of the surveillance-related goals are enabled by this scenario.


### Scenario 3: TLS client adds trust anchor when trust is mandated (distrust allowed)

**Scenario Description:**



*   Capability 1: Surveilling government operates and controls a trust anchor
*   Capability 2: This government mandates that websites operating within its jurisdiction obtain and provision TLS certificates from this trust anchor 
*   Capability 3: This government mandates that TLS clients operating within its jurisdiction add this trust anchor to their trust stores
*   This government applies for inclusion to the trust store
*   The TLS client <span style="text-decoration:underline;">adds</span> the trust anchor to its trust store

**Effects of Trust Expressions Deployment:**

Goal 1: A TLS client trusting a government-operated trust anchor does not grant that government the ability to _passively_ surveil connections between the client and the intended server unless it is also in possession of private keys or other TLS secrets. Trust Expressions does not provide a mechanism for web servers to unwittingly divulge these secrets.

Goal 2: A TLS client trusting a government-operated trust anchor allows for interception using a mis-issued certificate; however, this is a generic property of malicious mis-issuance from any trust anchor and is not unique to the government surveillance scenario.

Goal 3: Trust Expressions deployment and usage has some amount of impact on the ability to actively surveil connections without detection: if some clients added the trust anchor, while others did not, the surveillant may wish to only intercept the clients that added the trust anchor. This reduces the odds of detection from certificate validation failures in other clients. Trust Expressions provides a mechanism to differentiate the two client populations.

However, TLS fingerprinting—using other properties like cipher suite preferences and TLS extensions—is already used to differentiate clients. While fingerprinting is [not viable](https://github.com/davidben/tls-trust-expressions/blob/main/explainer.md#fingerprinting) for a broad, diverse set of TLS servers, it is viable for a surveillant’s single interception service. Trust Expressions only impacts detection where this differentiation was not already possible. \
 \
If the surveillant targets any clients that enforce [Certificate Transparency](https://certificate.transparency.dev/), the mis-issued certificates will need to be publicly logged. In this case, detection is more robust, and client differentiation, with or without Trust Expressions, has no significant impact.

**Analysis:**

In this scenario, the government-operated trust anchor is able to issue certificates that are trusted by the TLS client. Certificate issuance and usage are detectable via a variety of means, and the TLS client’s root program is able to respond to developments in accordance with their policies and procedures. 

Should the entity operating this trust anchor be caught mis-issuing certificates or otherwise violating root program requirements to further surveillance, this will likely result in the removal of trust anchors operated by this entity.



*   Proposed argument 3 further discusses the impacts of trust anchor negotiation on the detectability of active interception in TLS. This is particularly relevant to this scenario, since distrust is allowed upon detection of misbehavior.


### Scenario 4: TLS client does not add trust anchor when trust is mandated (distrust forbidden)

**Scenario Description:**



*   Capability 1: Surveilling government operates and controls a trust anchor
*   Capability 2: This government mandates that websites operating within its jurisdiction obtain and provision TLS certificates from this trust anchor
*   Capability 3: This government mandates that TLS clients that operate within their jurisdiction add trust in their trust anchor
*   Capability 4: This government forbids TLS clients that operate within their jurisdiction from removing trust in their trust anchor
*   The TLS client <span style="text-decoration:underline;">does not add</span> this trust anchor to its trust store

**Effects of Trust Expressions Deployment:**

Support for trust expressions by the TLS client, web server, and a participating trusted CA provides the ability for web servers to negotiate separate trusted TLS certificates if web servers provision them in addition to the government-mandated untrusted certificates.

Trust Expressions does not modify which certificates are trusted by a TLS client. Since both passive surveillance and interception require successful certificate validation, Trust Expressions deployment does not affect the ability to achieve any of the listed surveillance goals.

**Analysis:**

Without TLS client trust, none of the surveillance-related goals are enabled by this scenario.


### Scenario 5: TLS client adds trust anchor when trust is mandated and distrust is forbidden

**Scenario Description:**



*   Capability 1: Surveilling government operates and controls a trust anchor
*   Capability 2: This government mandates that websites operating within its jurisdiction obtain and provision TLS certificates from this trust anchor
*   Capability 3: This government mandates that TLS clients that operate within their jurisdiction add trust in their trust anchor
*   Capability 4: This government forbids TLS clients that operate within their jurisdiction from removing trust in their trust anchor, regardless of their root program policies
*   TLS client <span style="text-decoration:underline;">adds</span> the trust anchor to its trust store

**Effects of Trust Expressions Deployment:**

The effects of Trust Expressions deployment in Scenario 5 are the same as those in Scenario 3, discussed above. 

**Analysis:**

If a TLS client were to add this trust anchor when they would not otherwise have done so absent this government mandate, this would undermine trustworthy and secure authentication on the web and render the core purpose of TLS client root programs moot. 

Such a course of action would have significant security consequences, and would likely cause a significant uproar both among its user community and from the TLS ecosystem at large. Various existing TLS clients have fought such proposals for many years, working with policy-makers to help understand the consequences of such a requirement. 



*   Proposed argument 3 further discusses the impacts of trust anchor negotiation on the detectability of active interception in TLS. The detectability is less significant for this scenario since distrust is forbidden.


### Scenario 6: Government mandates escrow of TLS private keys and secrets



*   TLS Trust Expressions is standardized and deployed in a variety of web server software implementations
*   TLS Trust Expressions is supported by a given TLS client
*   Capability 5: This government mandates key escrow for TLS server private keys and other TLS secrets for websites that operate within their jurisdiction

**Effects of Trust Expressions Deployment:**

Once a government is in possession of TLS private keys and secrets, they are fully capable of performing surveillance of TLS connections to the affected domains. This capability does not depend on control of a trust anchor, or mis-issued certificates, because the authoritative TLS keys themselves have been compromised. This means Trust Expressions has no impact on these goals, positive or negative.

**Analysis:**

In this scenario, the surveilling government can achieve Goals 1 and 2 for servers that operate within its jurisdiction without relying on any of the other listed capabilities. Where it is able to achieve Goals 1 and 2, it is also able to do relatively stealthily, as this approach does not require any changes to certificate issuance to achieve.


## Proposed Arguments

The following are a list of arguments made on the IETF TLSwg mailing list and related discussions with some initial analysis from the authors of TLS Trust Expressions.


### Proposed Argument 1: Pre-provisioning untrusted certificates increases the chances of voluntary client trust 

_Trust Expressions can enable web servers to pre-provision untrusted certificates before they become trusted, such that they will be available if clients later trust them. This wide deployment of government-issued certificates will convince a TLS client’s root program to add this government trust anchor when they would not otherwise do so._

The primary purpose of a TLS client’s root program is to assess the trustworthiness of an organization operating one or more trust anchors and determining the net benefit to users if included ([Chrome](https://g.co/chrome/root-policy), [Mozilla](https://wiki.mozilla.org/CA/Root_Inclusion_Considerations), [Microsoft](https://aka.ms/RootCert), [Apple](https://www.apple.com/certificateauthority/ca_program.html), etc.). There are a multitude of signals that go into this decision-making process, including: 



*   **Competency:** maintaining clear policy and procedures, clean audits, and high-quality past performance
*   **Dedication:** undertaking thorough incident response and remediation, adopting advances in PKI technology, timely and transparent communication
*   **Value Proposition:** demonstrating that the benefits of inclusion outweigh the significant risk of adding new trust anchors

Broad deployment of certificates issued by a given trust anchor is, at best, only a slightly meaningful signal in determining the value proposition of including that trust anchor, and has no bearing on the trustworthiness, competency, or dedication of its operator. The value of existing pre-deployment vanishes when such deployment was forced upon server operators via mandate, and could even be reasonably interpreted as a sign of untrustworthiness. 

This outcome relies on root programs subverting their own trust anchor inclusion process based on the government-mandated popularity of unused TLS certificates on the web. This process subversion would be highly visible to external parties, risking significant blowback from users, regulators from other jurisdictions, and other members of the TLS ecosystem.

Contrary to this proposed argument, TLS Trust Expressions removes some pressure on TLS clients adding new trust anchors due to their broad deployment. Today, a root program might heavily consider the interoperability concerns of not adding a widely-trusted trust anchor due to web servers’ inability to distinguish TLS client trust. If instead, the root program knew that web servers could use trust expressions to deliver TLS client -specific certificates, there is no longer an interoperability need to trust the same set of trust anchors as all other TLS clients.

This is further explored in the [Security Considerations](https://github.com/davidben/tls-trust-expressions/blob/main/draft-davidben-tls-trust-expr.md#security-considerations) section of the draft.


### Proposed Argument 2a: Pre-provisioning untrusted certificates increases the chances of mandated client trust 

_Trust Expressions can enable web servers to pre-provision untrusted certificates before they become trusted, such that they will be available if clients later trust them. This will convince policy-makers to mandate TLS clients trust this anchor (Capability 3) and forbid its distrust (Capability 4)._

The existence of untrusted certificates deployed alongside trusted certificates on a web server does not form the basis of a coherent argument to mandate their trust. As explored in the previous argument, government-mandated popularity of untrusted certificates does not create a meaningful signal for the trustworthiness of the organization operating the trust anchor that issued those certificates. 

There is indeed the possibility that government policy-makers currently unwilling to enact a trust mandate today may change their course of action based on a variety of signals, whether these signals are related to such a mandate or not. Discussing this argument in meaningful detail necessarily involves relying on various opinion-based positions since neither the draft authors nor those proposing this argument are able to speak authoritatively as to the likelihoods of such an outcome.

Acknowledging the differing opinions expressed thus far, based on our experience working in TLS and PKI, the authors of TLS Trust Expressions hold the following opinions:



*   A government that does not mandate trust in a surveillance trust anchor chooses not to based on its evaluation of the significant tradeoffs that accompany such a decision, including dissatisfaction from its constituents, other governments, and technology providers, as well as the serious risks of isolation or reduced service availability as a result of these responses. 
*   As in the past, TLS client vendors will likely vociferously oppose the imposition of any such trust mandates, as this poses a direct risk to the security of their users.
*   There are existing mechanisms that are better suited than TLS Trust Expressions for enabling a government trust mandate. For example, the certificate_authorities TLS extension fails to provide a useful tool for communicating modern client trust stores, but can efficiently signal trust in a mandated trust anchor. 
*   While support for certificate_authorities is far from ubiquitous, supporting a government-mandated trust anchor with this extension is significantly simpler than with Trust Expressions, which has zero implementation. In the years since certificate_authorities uncontroversial standardization, this has not happened.
*   Governments that do not wish to mandate trust in, and forbid distrust of, their own trust anchor, but would reverse on that position in the presence of Trust Expressions are not making decisions relative to the actual capabilities provided by this mechanism. While such decision-making is entirely possible, predicting a specific outcome from this is speculative.

In order for Trust Expressions to meaningfully change the decision calculus on mandating client trust in a government-operated trust anchor, it would have to be the case that:



*   The government would not be willing to pass a global trust mandate, but would be convinced to change course based on the deployment of untrusted, and unused TLS certificates.
*   The cost and complexity of enforcing a trust mandate by relying on trust expressions is less than available alternatives.
*   The government would not outright mandate a change to directly include surveillance measures (e.g. a custom browser extension, disclosing [SSLKEYLOGFILE](https://www.ietf.org/archive/id/draft-thomson-tls-keylogfile-00.html) contents).

Scenario 2 discusses the consequences of a TLS client not adding a mandated trust anchor, and Scenarios 3 and 4 discuss the consequences of the TLS client adding a mandated trust anchor, differentiated based on whether distrust is allowed or forbidden, respectively.


### Proposed Argument 2b: Enabling geo-specific client trust decisions increases the chances of mandated client trust

_Trust Expressions allows a web server to deploy multiple certificates and serve them to different TLS clients, based on what they trust. As a result of this, a government could impose a mandate requiring trust in a government-operated trust anchor only for TLS clients in their jurisdiction. The ability to localize the impact of this decision will convince policy-makers to mandate TLS clients trust this anchor (Capability 3) and forbid its distrust (Capability 4)._

Today, some policy-makers may wish to mandate the use of a government-operated trust anchor within its jurisdiction. In order to achieve this, requirements must be placed on both web servers and TLS clients. Today, this would look like the following:



*   Web servers operating within the jurisdiction are required to obtain TLS certificates from the government-operated trust anchor, <span style="text-decoration:underline;">AND</span>
*   TLS clients operating within the jurisdiction are required to add this trust anchor to their trust store and successfully validate certificates issued by it.

TLS clients _outside_ the jurisdiction could not reliably be forced to recognize this trust anchor, so this leaves policy-makers with the following decision:



1. Forbid web servers from using TLS certificates issued by any other trust anchor. This would result in significant global breakage caused by incompatible TLS client trust stores.
2. Permit web servers to obtain TLS certificates issued by a widely-trusted trust anchor and require web servers to determine a connection’s origin in order to serve the widely-trusted certificate to connections originating outside the jurisdiction to allow external compatibility. This requires web servers to infer client location using GeoIP or similar mechanisms.
3. Permit web servers to obtain TLS certificates form a widely-trusted trust anchor and require web servers and clients to implement existing trust anchor negotiation solutions such as the certificate_authorities TLS extension. This requires clients to be able to self-determine their location in order to know when to negotiate TLS certificates issued from the government trust anchor.

Of these decisions, only (3) would be impacted by future Trust Expressions deployment. This would look like the following:



*   Mandate TLS clients trust the government-operated trust anchor,
*   Mandate the TLS client’s root program to publish and maintain a trust store manifest signaling this inclusion,
*   Have the government-operated trust anchor consume these manifests and append the corresponding trustStoreInclusionList metadata to certificates it issues,
*   Mandate web servers change their software to preferentially serve certificates from the government-operated trust anchor upon receiving a trust expression that matches this metadata,
*   And finally, mandate that TLS clients self-determine their location and send a trust expression that signals trust in this trust anchor.

Alternatively, the government could use an existing mechanism such as the certificate_authorities TLS extension, which would look like the following:



*   Mandate TLS clients within the jurisdiction trust the government-operated trust anchor,
*   Mandate TLS clients within the jurisdiction send a single trust anchor in the certificate_authorities extension,
*   Mandate web servers preferentially serve certificates from the government-operated trust anchor upon receiving a certificate_authorities extension containing it,
*   And finally, mandate that TLS clients self-determine their location and send a certificate_authorities TLS extension that signals trust in this trust anchor.

In order for Trust Expressions to meaningfully change the decision calculus on mandating client trust in a government-operated trust anchor, it would have to be the case that:



*   The government would not be willing to simply pass a trust mandate and accept the breakage or force technology providers to design mitigations (decision 1).
*   Existing geolocation techniques are too unreliable for servers to determine a connection’s origin to infer client trust and trust expressions improves this by a significant margin
*   Client-side self-determination must be measurably more reliable than server-side GeoIP, as that would make decision 2 more preferable otherwise 
*   The cost and complexity of enforcing a trust mandate by relying on trust expressions is less than available alternatives
*   The government would mandate software providers to change clients to self identify as being in jurisdiction, but not outright mandate a change to directly include surveillance measures (e.g. a custom browser extension, disclosing [SSLKEYLOGFILE](https://www.ietf.org/archive/id/draft-thomson-tls-keylogfile-00.html) contents)


### Proposed Argument 3: Mandated deployment of certificates makes misbehavior more difficult to detect

_The mandated trust and deployment of certificates from the government-issued trust anchor makes it harder to detect whether a connection is being intercepted since Trust Expressions support allows for broader negotiation and use of certificates from this trust anchor._

Malicious activity or intention to surveil a connection cannot be deduced from the certification authority that issued a certificate used in TLS. If certificates from a specific trust anchor were de facto malicious, that trust anchor should have not been added or removed if already present. Scenario 5 explores the consequences of being unable to distrust a misbehaving trust anchor. 

This argument then reduces to the efficacy of various detection and prevention methods for maliciously mis-issued certificates:



*   The surveilling government mis-issuing certificates for victim domains would be detectable via CT, if logged there, so malicious certificate issuance would presumably be restricted to targeting connections from non-CT-enforcing clients. The deployment of TLS Trust Expressions does not affect the ability to detect a CT enforcing client as the request for SCTs is already visible today in the Client Hello. 
*   Certificate pinning, while no longer recommended for wide use due to inherent operational risks, would prevent TLS clients from establishing TLS connections with a MITM using a certificate issued from a different CA. TLS trust expressions could make the detection of pinning clients easier, allowing them to be excluded from receiving the mis-issued certificates. 
*   Clients using telemetry for failures or other properties that could increase the likelihood of detection could potentially also be avoided if their use of TLS trust expressions made fingerprinting them easier

The core risk to users presented by this argument’s scenario stems from the client’s trust in a misbehaving trust anchor, not the deployment of Trust Expressions.

While there exist a variety of fingerprinting signals for determining basic facts about a TLS client, if sending trust expressions presents a more reliable mechanism for determining a client’s detection capabilities, it could facilitate the ability to tailor surveillance towards connections that are less likely to result in detection. This would then increase the likelihood of being able to intercept a connection surreptitiously for some subset of TLS clients. The authors do not believe that TLS Trust Expression implementations, following the recommendations in the privacy section of the draft, would provide significant information in the handshake that is not today discoverable by other observable properties of the TLS handshake. 
