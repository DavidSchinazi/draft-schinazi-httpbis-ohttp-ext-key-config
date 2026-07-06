---
title: "An Extensible Key Configuration Format for Oblivious HTTP"
abbrev: "OHTTP Extended Key Configuration"
category: std
docname: draft-schinazi-httpbis-ohttp-ext-key-config-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
keyword:
 - oblivious
 - HTTP
 - configuration
 - extension
area: "Web and Internet Transport"
workgroup: "HTTP"
venue:
  group: "HTTP"
  type: "Working Group"
  home: https://httpwg.org/
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "DavidSchinazi/draft-schinazi-httpbis-ohttp-ext-key-config"
  latest: "https://DavidSchinazi.github.io/draft-schinazi-httpbis-ohttp-ext-key-config/draft-schinazi-httpbis-ohttp-ext-key-config.html"

author:
 -
    ins: D. Schinazi
    name: David Schinazi
    org: Google LLC
    email: dschinazi.ietf@gmail.com

normative:

informative:

...

--- abstract

Oblivious HTTP is a protocol for forwarding encrypted HTTP messages. This
requires communicating the gateway's key configuration to clients. While a
key configuration media type was defined for this purpose, it has some
limitations such as the inability to convey key lifetimes and
interoperability issues. This document defines a similar extensible key
configuration format that addresses those issues.


--- middle

# Introduction

Oblivious HTTP ({{!OHTTP=RFC9458}}) is a protocol for forwarding encrypted
HTTP messages. This requires communicating the gateway's key configuration
to clients. A key configuration media type was defined for this purpose in
{{Section 3 of OHTTP}}. That format has the following limitations.

## Key Lifetimes

The security properties of OHTTP rely on periodic rotation of gateway keys.
Many gateways rotate their keys on a weekly cadence. However, the original
key configuration format has no way of informing the client of the lifetime
of the keys it contains. Clients then need to assume an expiration time; if
they didn't, they might keep using keys past when the gateway forgot them,
or worse: they might encrypt sensitive data using a key that leaked after
its expiration. Since those bounds aim to be conservative, clients can end
up marking a key as expired even though it is still fresh on the gateway.
This reduces reliability when periodic key fetch operations fail due to
intermittent connectivity issues.

## Interoperability Issues

During the development of {{OHTTP}}, the authors discovered a forward
compatibility
[issue](https://github.com/ietf-wg-ohai/oblivious-http/issues/285) with
the key configuration format: parsing multiple keys required explicit
support for each included KEM because the length of the HPKE public key
was not encoded in the format. This would prevent deploying new KEMs in
a backwards-compatible way. The authors
[resolved](https://github.com/ietf-wg-ohai/oblivious-http/pull/284/changes)
this by adding a length prefix to each configuation entry. Unfortunately,
they did not change the media type when doing so. Since this change to the
format was made after the document's working group last call, there were
already implementations in production. At the time of writing, there are
multiple billions of devices in production that understand the
"application/ohttp-keys" media type as not including the length prefix.
Other client implementations support both formats by attempting to parse
with the length prefix and falling back to the older format if that fails.
This runs the risk of segregating clients, as described in
{{Section 3.2 of OHTTP}}.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses terminology from {{!QUIC=RFC9000}}. Where this document
defines protocol types, the definition format uses the notation from
{{Section 1.3 of QUIC}}. This specification uses the variable-length
integer encoding from {{Section 16 of QUIC}}. Variable-length integer
values do not need to be encoded in the minimum number of bytes necessary.

# Extensible Format

This document defines a new media type (see {{iana-media}} for its value).
It represents a list of key configurations. It has the following format:

~~~
HPKE Symmetric Algorithms {
  HPKE KDF ID (16),
  HPKE AEAD ID (16),
}

Extension {
  Extension ID (i),
  Extension Length (i),
  Extension Value (..),
}

Key Config {
  Key Config Length (16),
  Key Identifier (8),
  HPKE KEM ID (16),
  HPKE Public Key (Npk * 8),
  HPKE Symmetric Algorithms Length (16) = 4..65532,
  HPKE Symmetric Algorithms (32) ...,
  Extensions (..) ...
}

Key Configs {
  Key Config (..) ...
}
~~~
{: #format-key-config title="A List of Key Configurations"}

The "HPKE Symmetric Algorithms" struct consists of the following fields:

HPKE KDF ID:

: A 16 bit HPKE KDF identifier as defined in {{Section 7.2 of !HPKE=RFC9180}}
  or [the HPKE KDF IANA
  registry](https://www.iana.org/assignments/hpke/hpke.xhtml#hpke-kdf-ids).

HPKE AEAD ID:

: A 16 bit HPKE AEAD identifier as defined in {{Section 7.3 of HPKE}} or
  [the HPKE AEAD IANA
  registry](https://www.iana.org/assignments/hpke/hpke.xhtml#hpke-aead-ids).

The "Extension" struct consists of the following fields:

Extension ID:

: A 62 bit OHTTP Key Configuration Extension identifier as defined in
  {{ext}}.

Extension Length:

: The length of the "Extension Value" field that follows it.

Extension Value:

: Extension-specific information, with encoding rules dependent on the value
  of the "Extension ID" field.

The "Key Config" struct consists of the following fields:

Key Config Length:

: A 16 bit integer in network byte order that encodes the length, in bytes, of
  this "Key Config" struct, not including this field.

Key Identifier:

: An 8 bit value that identifies the key used by the Oblivious Gateway Resource.

HPKE KEM ID:

: A 16 bit value that identifies the Key Encapsulation Method (KEM) used for the
  identified key as defined in {{Section 7.1 of HPKE}} or [the HPKE KDF IANA
  registry](https://www.iana.org/assignments/hpke/hpke.xhtml#hpke-kem-ids).

HPKE Public Key:

: The public key used by the gateway. The length of the public key is `Npk`, which is
  determined by the choice of HPKE KEM as defined in {{Section 4 of HPKE}}.

HPKE Symmetric Algorithms Length:

: A 16 bit integer in network byte order that encodes the length, in bytes, of
  the HPKE Symmetric Algorithms field that follows.

HPKE Symmetric Algorithms:

: One or more "HPKE Symmetric Algorithms" structs.

Extensions:

: Zero or more "Extension" structs. This field continues until the end of
  this "Key Config" struct.

The new media type defined in this document represents a list of concatenated
"Key Config" structs.

# Extensions {#ext}

In addition to providing an extensible format for OHTTP key configurations,
this document defines two extensions. The EXPIRATION extension indicates the
date/time after which the key can no longer be used. The NOT_BEFORE
extension indicates the date/time before which the key cannot yet be used.
Both are encoded as QUIC variable-length integers representing UNIX
timestamps (number of seconds since January 1, 1970, UTC -- see
{{Section 4.2.1 of !TIMESTAMP=RFC8877}}).

The NOT_BEFORE extension allows a gateway to publish public keys in advance
of them being valid. Given that and the EXPIRATION extension, the bounds
can then be tightenned on the validity of keys, increasing both security
and reliability.

Gateways SHOULD provide some leeway between their stated times and the
actual times to account for a few minutes of clock skew.

# Parsing

When parsing the list of key configurations, clients MUST silently skip any
key configuration that carries a KEM ID unknown to the client. Clients MUST
also silently skip over any extension for which the extension ID is unknown.

As mentioned in {{Section 3.2 of OHTTP}}, a client that receives an list of
key configuration object with encoding errors might be able to recover one
or more key configurations. Differences in how key configurations are
recovered might be exploited to segregate clients, so clients MUST discard
the entire list if they encounter any encoding error in one of the key
configurations or its extensions.

If an Extension Value field contains more data than expected for that
Extension ID, the client MUST treat it as an encoding error. If the
Extension Length field is such that the extension would extend past the
end of the Key Config struct, clients MUST treat it as an encoding error.

# Security Considerations {#sec}

The security considerations described in {{Section 6 of OHTTP}} apply to
this document as well.

# IANA Considerations

## Media Type {#iana-media}

The "application/ohttp-keys-ext-00" media type identifies a key configuration used
by Oblivious HTTP. Note that the sybtype is expected to be changed if
non-backwards-compatible changes are made to the format. If this document is
approved, the "-NN" suffix will be removed before publication.

Type name:

: application

Subtype name:

: ohttp-keys-ext-00

Required parameters:

: N/A

Optional parameters:

: N/A

Encoding considerations:

: "binary"

Security considerations:

: see {{sec}}

Interoperability considerations:

: N/A

Published specification:

: this specification

Applications that use this media type:

: This type identifies a key configuration as used by Oblivious HTTP and
  applications that use Oblivious HTTP.

Fragment identifier considerations:

: N/A

Additional information:

: <dl spacing="compact">
  <dt>Magic number(s):</dt><dd>N/A</dd>
  <dt>Deprecated alias names for this type:</dt><dd>N/A</dd>
  <dt>File extension(s):</dt><dd>N/A</dd>
  <dt>Macintosh file type code(s):</dt><dd>N/A</dd>
  </dl>

Person and email address to contact for further information:

: see Authors' Addresses section

Intended usage:

: COMMON

Restrictions on usage:

: N/A

Author:

: see Authors' Addresses section

Change controller:

: IETF
{: spacing="compact"}

## OHTTP Key Configuration Extension Registry {#iana-ext}

This document establishes a registry for extension IDs. The "OHTTP Key
Configuration Extensions" registry governs a 62-bit space and operates under
the QUIC registration policy documented in {{Section 22.1 of QUIC}}. This new
registry includes the common set of fields listed in {{Section 22.1.1 of QUIC}}.
In addition to those common fields, all registrations in this registry MUST
include a "Name" field that contains a short name or label for the extension.

Permanent registrations in this registry are assigned using the Specification
Required policy ({{Section 4.6 of !IANA-POLICY=RFC8126}}), except for values
between 0x00 and 0x3f (in hexadecimal; inclusive), which are assigned using
Standards Action or IESG Approval as defined in {{Sections 4.9 and 4.10 of
IANA-POLICY}}.

Extensions with a value of the form 0x29 * N + 0x17 for integer values of N
are reserved to exercise the requirement that unknown Extension IDs be ignored.
These extensions have no semantics and can carry arbitrary values. These values
MUST NOT be assigned by IANA and MUST NOT appear in the listing of assigned
values.

This registry initially contains the following entries:

| Value |    Name    |
|:------|:-----------|
| 0x00  | EXPIRATION |
| 0x01  | NOT_BEFORE |
{: #iana-extensions-table title="Extensions"}

All of these new entries use the following values for these fields:

Status:
: provisional (permanent if this document is approved)

Reference:
: This document

Change Controller:
: IETF

Contact:
: HTTPBIS Working Group <ietf-http-wg@w3.org>

Notes:
: None
{: spacing="compact"}

--- back

# Acknowledgments
{:numbered="false"}

The "Not Before" and "Expiration" extensions were inspired by the "nbf"
and "exp" claims in JSON Web Keys ({{?JWT=RFC7519}}).
