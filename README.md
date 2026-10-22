# OpenID Federation Key Attestation 1.0

This repository contains the OpenID Federation Key Attestation specification.
This specification acts as an extension to OpenID Federation 1.0 / 1.1 and
defines an additional federation endpoint that lets a Subordinate ask its
Immediate Superior to update the attested Federation Entity Keys in the
Subordinate Statement.

## Builds

The latest editors' draft can be viewed at
https://harrykodden.github.io/federation-key-attestation/main.html

Branches can be viewed at
https://harrykodden.github.io/federation-key-attestation/{BRANCH-NAME}.html

## Build the HTML

```docker run -v `pwd`:/data danielfett/markdown2rfc openid-federation-key-attestation-1_0.md```

## Contact

For further information and to get involved, please read
[CONTRIBUTING.md](CONTRIBUTING.md) and/or contact the authors.
