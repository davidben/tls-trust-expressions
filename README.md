# TLS Trust Anchor Identifiers

This repository contains a draft mechanism for negotiating certificate
trust anchors in TLS. For an overview, the informal [explainer](explainer.md)
document is a good starting point.

* [Editor's Copy](https://davidben.github.io/tls-trust-expressions/#go.draft-beck-tls-trust-anchor-ids.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-beck-tls-trust-anchor-ids)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-beck-tls-trust-anchor-ids)
* [Compare Editor's Copy to Individual Draft](https://davidben.github.io/tls-trust-expressions/#go.draft-beck-tls-trust-anchor-ids.diff)

**Historical note:** This repository previously also contained [TLS Trust Expressions](https://datatracker.ietf.org/doc/draft-davidben-tls-trust-expr/), which was an earlier design for the same problem. That design was replaced by Trust Anchor Identifiers, but some discussion remains as a point of comparison in the design space.

## Additional Materials

* [PKI Transition Strategies](pki-transition-strategies.md)
* [Surveillance and Trust Anchor Negotiation](surveillance-and-trust-anchor-negotiation.md)

## Contributing

See the
[guidelines for contributions](https://github.com/davidben/tls-trust-expressions/blob/main/CONTRIBUTING.md).

Contributions can be made by creating pull requests.
The GitHub interface supports creating pull requests using the Edit (✏) button.


## Command Line Usage

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed.  See
[the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

