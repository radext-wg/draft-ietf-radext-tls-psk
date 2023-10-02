---
title: RADIUS and TLS-PSK
abbrev: RADIUS and TLS-PSK
docname: draft-ietf-radext-tls-psk-03

stand_alone: true
ipr: trust200902
area: Internet
wg: RADEXT Working Group
kw: Internet-Draft
cat: info
submissionType: IETF

pi:    # can use array (if all yes) or hash here
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:

- ins: A. DeKok
  name: Alan DeKok
  org: FreeRADIUS
  email: aland@freeradius.org

normative:
  BCP14: RFC8174
  RFC2865:
  RFC4279:
  RFC7585:
  RFC8446:
  RFC9258:

informative:
  RFC6613:
  RFC6614:
  RFC7360:
  RFC8937:
  RFC9257:
  RFC9325:

venue:
  group: RADEXT
  mail: radext@ietf.org
  github: freeradius/radext-tls-psk.git

--- abstract

This document gives implementation and operational considerations for using TLS-PSK with RADIUS/TLS (RFC6614) and RADIUS/DTLS (RFC7360).

--- middle

# Introduction

The previous specifications "Transport Layer Security (TLS) Encryption for RADIUS"  {{RFC6614}} and "Datagram Transport Layer Security (DTLS) as a Transport Layer for RADIUS" {{RFC7360}} defined how (D)TLS can be used as a transport protocol for RADIUS.  However, those documents do not provide guidance for using TLS-PSK with RADIUS.  This document provides that missing guidance, and gives implementation and operational considerations.

# Terminology

{::boilerplate bcp14}

# History

TLS deployments usually rely on certificates in most common uses. However, we recognize that it may be difficult to fully upgrade client implementations to allow for certificates to be used with RADIUS/TLS and RADIUS/DTLS.  These upgrades involve not only implementing TLS, but can also require significant changes to administration interfaces and application programming interfaces (APIs) in order to fully support certificates.

For example, unlike shared secrets, certificates expire.  This expiration means that a working system using TLS can suddenly stop working.  Managing this expiration can require additional notification APIs on RADIUS clients and servers which were previously not required when shared secrets were used.

Certificates also require the use of certification authorities (CAs), and chains of certificates.  RADIUS implementations using TLS therefore have to track not just a small shared secret, but also potentially many large certificates.  The use of TLS-PSK can therefore provide a simpler upgrade path for implementations to transition from RADIUS shared secrets to TLS.

# General Discussion of PSKs and PSK Identities

Before we define any RADIUS-specific use of PSKs, we must first review the current standards for PSKs, and give general advice on PSKs and PSK identities.

The requirements in this section apply to both client and server implementations which use TLS-PSK.  Client-specific and server-specific issues are discussed in more detail later in this document.

## Requirements on PSKs

Reuse of a PSK in multiple versions of TLS (e.g. TLS 1.2 and TLS 1.3) is considered unsafe ({{RFC8446}} Section E.7).  Where TLS 1.3 binds the PSK to a particular key derivation function, TLS 1.2 does not.  This binding means that it is possible to use the same PSK in different hashes, leading to the potential for attacking the PSK by comparing the hash outputs.  While there are no known insecurities, these uses are not known to be secure, and should therefore be avoided.

{{RFC9258}} adds a key derivation function to the import interface of (D)TLS 1.3, which binds the externally provided PSK to the protocol version.  In particular, that document:

> ... describes a mechanism for importing PSKs derived from external PSKs by including the target KDF, (D)TLS protocol version, and an optional context string to ensure uniqueness. This process yields a set of candidate PSKs, each of which are bound to a target KDF and protocol, that are separate from those used in (D)TLS 1.2 and prior versions. This expands what would normally have been a single PSK and identity into a set of PSKs and identities.

An implementation MUST NOT use the same PSK for TLS 1.3 and for earlier versions of TLS.  This requirement prevents reuse of a PSK with multiple TLS versions, which prevents the attacks discussed in {{RFC8446}} Section E.7.  The exact manner in which this requirement is enforced is implementation-specific.  One possibility is to have two different PSKs.  Another possibility is to forbid the use of TLS 1.3, or to forbid the use of TLS versions less than TLS 1.3.

It is RECOMMENDED that systems follow the directions of {{RFC9257}} Section 6 for the use of external PSKs in TLS.  That document provides extremely useful guidance on generating and using PSKs.

Implementations MUST support PSKs of at least 32 octets in length, and SHOULD support PSKs of 64 octets or more.  As the PSKs are generally hashed before being used in TLS, the useful entropy of a PSK is limited by the size of the hash output.  This output may be 256, 384, or 512 bits in length.  Never the less, it is good practice for implementations to allow entrty of PSKs of more than 64 octets, as the PSK may be in a form other than bare binary data.  Implementations which limit the PSK to a maximum of 64 octets are likely to use PSKs which have much less than 512 bits of entropy.  That is, a PSK with high entropy may be expanded via some construct (e.g. base32 as in the example below) in order to make it easier for people to interact with.  Where 512 bits of entropy are input to an encoding construct, the output may be larger than 64 octets.

Implementations MUST require that PSKs be at least 16 octets in length, which SHOULD be derived from a source with at least 128 bits of entropy.  That is, short PSKs MUST NOT be permitted to be used.  Further, the strength of the PSK is not determined by the length of the PSK, but instead by the number of bits of entropy which it contains.  People are not good at creating data with high entropy, so a source of cryptographically secure random numbers MUST be used.

Administrators SHOULD use PSKs of at least 24 octets, generated using a source of cryptographically secure random numbers.  Implementers needing a secure random number generator should see {{RFC8937}} for for further guidance.  PSKs are not passwords, and administrators should not try to manually create PSKs.

Passwords are generally intended to be remembered and entered by people on a regular basis.  In contrast, PSKs are intended to be entered once, and then automatically saved in a system configuration.  As such, due to the limited entropy of passwords, they are not acceptable for use with TLS-PSK, and would only be acceptable for use with a password-authenticated key exchange (PAKE) TLS method.

We also incorporate by reference the requirements of Section 10.2 of {{RFC7360}} when using PSKs.

In order to guide Implementers, we give an example script below which generates random PSKs.  While the script is not portable to all possible systems, the intent here is to document a concise and simple method for creating PSKs which are both secure, and humanly manageable.

> \#!/usr/bin/env perl
> use MIME::Base32;
> use Crypt::URandom();
> print join('-', unpack("(A4)*", lc encode_base32(Crypt::URandom::urandom(16)))), "\n";

This script reads 128 bits (16 octets) of random data from a secure source, encodes it in Base32, and then formats it to be more humanly manageable.  The generated keys are of the form "yttb-4gv2-ynfk-jbjh-2dja-cj7e-am".  This form of PSK will be accepted by any implementation which supports at least 32 octets for PSKs.  Larger PSKs can be generated by passing larger values to the "urandom()" function.  The above derivation assumes that the random source returns one bit of entropy for every bit of randomness which is returned.  Sources failing that assumption are NOT RECOMMENDED.

### Interaction between PSKs and Shared Secrets

Any shared secret used for RADIUS/UDP or RADIUS/TLS MUST NOT be used for TLS-PSK. 

It is RECOMMENDED that RADIUS clients and server track all used shared secrets and PSKs, and then verify that the following requirements all hold true:

* no shared secret is used for more than one RADIUS client
* no PSK is used for more than one RADIUS client
* no shared secret is used as a PSK

Note that the shared secret of "radsec" given in {{RFC6614}} can be used across multiple clients, as that value is mandated by the specification.  The intention here is to recommend best practices for administrators who enter site-local shared secrets.

There may be use-cases for using one shared secret across multiple RADIUS clients.  There may similarly be use-cases for sharing a PSK across multiple RADIUS clients.   Details of the possible attacks on reused PSKs are given in {{RFC9257}} Section 4.1.

There are few, if any, use-cases for using a PSK as a shared secret, or vice-versa.

Implementations SHOULD NOT provide user interfaces which allow both PSKs and shared secrets to be entered at the same time.  There is too much of a temptation for administrators to enter the same value in both fields, which would violate the limitations given above.  Implementations MUST NOT use a "shared secret" field as a way for administrators to enter PSKs.  The PSK entry fields MUST be labeled as being related to PSKs, and not to shared secrets.

## PSK Identities

It is RECOMMENDED that systems follow the directions of {{RFC9257}} Section 6.1.1 for the use of external PSK Identities in TLS.  Note that the PSK identity is sent in the clear, and is therefore visible to attackers.  Where privacy is desired, the PSK identity could be either an opaque token generated cryptographically, or perhaps in the form of a Network Access Identifier (NAI) {{?RFC7542}}, where the "user" portion is an opaque token.  For example, an NAI could be "68092112@example.com".  If the attacker already knows that the client is associated with "example.com", then using that domain name in the PSK identity offers no additional information.  In contrast, the "user" portion needs to be both unique to the client and private, so using an opaque token there is a more secure approach.

Implementations MUST support PSK Identities of 128 octets, and SHOULD support longer PSK identities.  We note that while TLS provides for PSK identities of up to 2^16-1 octets in length, there are few practical uses for extremely long PSK identities.

### Security of PSK Identities

We note that the PSK identity is a field created by the connecting client.  Since the client is untrusted until both the identity and PSK have been verified, both of those fields MUST be treated as untrusted.  That is, a well-formed PSK Identity is likely to be in UTF-8 format, due to the requirements of {{RFC4279}} Section 5.1.  However, implementations MUST support managing PSK identities as a set of undistinguished octets.

It is not safe to use a raw PSK Identity to look up a corresponding PSK.  The PSK may come from an untrusted source, and may contain invalid or malicious data.  For example, the identity may have incorrect UTF-8 format; or it may contain data which forms an injection attack for SQL, LDAP, REST or shell meta characters; or it may contain embedded NUL octets which are incompatible with APIs which expect NUL terminated strings.  The identity may also be up to 65535 octets long.

As such, implementations MUST validate the identity prior to it being used as a lookup key.  When the identity is passed to an external API (e.g. database lookup), implementations MUST either escape any characters in the identity which are invalid for that API, or else reject the identity entirely.  The exact form of any escaping depends on the API, and we cannot document all possible methods here.  However, a few basic validation rules are suggested, as outlined below.  Any identity which is rejected by these validation rules SHOULD cause the server to close the TLS connection.

The suggested validation rules are as follows:

* Identities longer than a fixed maximum SHOULD be rejected.  The limit is implementation dependent, but SHOULD NOT be less than 128, and SHOULD NOT be more than 1024.

* Identities which are not in UTF-8 format SHOULD be rejected.  This includes any identity with embedded control characters, NUL octets, etc.

* Where the NAI format is expected, identities which are not in NAI format SHOULD be rejected

It is RECOMMENDED that implementations extend these rules with any additional validation which are found to be useful.  For example, implementations and/or deployments could both generate PSK identities in a particular format for passing to client systems, and then also verify that any received identity matches that format.  For example, a site could generate PSK identities which are composed of characters in the local language.  The site could then reject identities which contain characters from other languages, even if those characters are valid UTF-8.

## PSK and PSK Identity Sharing

While administrators may desire to share PSKs and/or PSK identities across multiple systems, such usage is NOT RECOMMENDED.  Details of the possible attacks on reused PSKs are given in {{RFC9257}} Section 4.1.

Implementations MUST be able to configure a unique PSK and PSK identity for each possible client-server relationship.  This configuration allows administrators desiring security to use unique PSKs for each such relationship.  This configuration also allows administrators to re-use PSKs and PSK Identities where local policies permit.

Implementations SHOULD warn administrators if the same PSK identity and/or PSK is used for multiple client-server relationships.

# Guidance for RADIUS Clients

Client implementations MUST allow the use of a pre-shared key (TLS-PSK) for RADIUS/TLS.  The client implementation can then expose a user interface flag which is "TLS yes / no", and then also fields which ask for the PSK identity and PSK itself.

For TLS 1.3, Implementations MUST support "psk_dhe_ke" Pre-Shared Key Exchange Mode in TLS 1.3 as discussed in {{RFC8446}} Section 4.2.9 and in {{RFC9257}} Section 6.  Implementations MUST implement the recommended cipher suites in {{RFC9325}} Section 4.2 for TLS 1.2, and in {{RFC9325}} Section 4.2 for TLS 1.3.

## PSK Identities

{{RFC6614}} is silent on the subject of PSK identities, which is an issue that we correct here.  Guidance is required on the use of PSK identities, as the need to manage identities associated with PSK is a new requirement for NAS management interfaces, and is a new requirement for RADIUS servers.

RADIUS systems implementing TLS-PSK MUST support identities as per {{RFC4279}} Section 5.3, and MUST enable configuring TLS-PSK identities in management interfaces as per {{RFC4279}} Section 5.4.

Due to the security issues described above, RADIUS shared secrets cannot safely be used as TLS-PSKs. Therefore in order to prevent confusion between shared secrets and TLS-PSKs, management interfaces and APIs need to label PSK fields as "PSK" or "TLS-PSK", rather than as "shared secret".

RADIUS/TLS clients MUST still permit the configuration of a RADIUS server IP address or host name.  Where {{RFC7585}} dynamic server lookups are used, that specification only makes provisions for servers to use certificates.  Since there is no way to determine which PSK to use for a connection to a particular server, then TLS-PSK cannot be used with {{RFC7585}} dynamic lookups.

# Guidance for RADIUS Servers

In order to support clients with TLS-PSK, server implementations MUST allow the use of a pre-shared key (TLS-PSK) for RADIUS/TLS.

The following section(s) describe guidance for RADIUS server implementations and deployments.  We first give an overview of current practices, and then extend and/or replace those practices for TLS-PSK.

Implementations MUST support the recommended cipher suites in {{RFC9325}} Section 4.2 for TLS 1.2, and in {{RFC9325}} Section 4.2 for TLS 1.3.  In order to future-proof these recommendations, we give the following recommendations:

* Implementations SHOULD use the "Recommended" cipher suites listed in the IANA "TLS Cipher Suites" registry,
  * for TLS 1.3, the use "psk_dhe_ke" PSK key exchange mode,
  * for TLS 1.2 and earlier, use cipher suites which require ephemeral keying.

## Current Practices

RADIUS identifies clients by source IP address ({{RFC2865}} and {{RFC6613}}) or by client certificate ({{RFC6614}} and {{RFC7585}}).  Neither of these approaches work for TLS-PSK.  This section describes current practices and mandates behavior for servers which use TLS-PSK.

A RADIUS/UDP server is typically configured with a set of information per client, which includes at least the source IP address and shared secret.  When the server receives a RADIUS/UDP packet, it looks up the source IP address, finds a client definition, and therefore the shared secret.  The packet is then authenticated (or not) using that shared secret.

That is, the IP address is treated as the clients identity, and the shared secret is used to prove the clients authenticity and shared trust.  The set of clients forms a logical database "client table", with the IP address as the key.

A server may be configured with additional site-local policies associated with that client.  For example, a client may be marked up as being a WiFi Access Point, or a VPN concentrator, etc.  Different clients may be permitted to send different kinds of requests, where some may send Accounting-Request packets, and other clients may not send accounting packets.

## Practices for TLS-PSK

We define practices for TLS-PSK by analogy with the RADIUS/UDP use-case, and by extending the additional policies associated with the client.  The PSK identity replaces the source IP address as the client identifier.  The PSK replaces the shared secret as proof of client authenticity and shared trust.  However, systems implementing RADIUS/TLS {{RFC6614}} and RADIUS/DTLS {{RFC7360}} MUST still use the shared secret as discussed in those specifications.  Any PSK is only used by the TLS layer, and has no effect on the RADIUS data which is being transported.  That is, the RADIUS data transported in a TLS tunnel is the same no matter if client authentication is done via PSK or by client certificates.  The encoding of the RADIUS data is entirely unaffected by the use (or not) of PSKs and client certificates.

In order to securely support dynamic source IP addresses for clients, we also require that servers limit clients based on a network range.  The alternative would be to suggest that RADIUS servers allow any source IP address to connect and try TLS-PSK, which could be a security risk.

Even where {{RFC7585}} dynamic discovery is not used, servers SHOULD NOT permit TLS-PSK to be used across the wider Internet.  It is significantly easier for an attacker to crack a PSK than to forge a client certificate.  The intent for TLS-PSK is for it to be used in internal / secured networks.  The benefits of TLS-PSK are in easing management and in administative overhead, not in securing traffic from resourceful attackers.  Where TLS-PSK is used across the Internet, PSKs MUST contain at least 256 octets of entropy.

For example, a RADIUS server could be configured to be accept connections from a source network of 192.0.2.0/24.  The server could therefore discard any TLS connection request which comes from a source IP address outside of that network.  In that case, there is no need to examine the PSK identity or to find the client definition.  Instead, the IP source filtering policy would deny the connection before any TLS communication had been performed.

RADIUS servers need to be able to limit certain PSK identifiers to certain network ranges or IP addresses.  That is, if a NAS is known to have a dynamic IP address within a particular subnet, the server should limit use of the NASes PSK to that subnet.  This filtering can therefore help to catch configuration errors.

As some clients may have dynamic IP addresses, it is possible for a one PSK identity to appear at different source IP addresses over time.  In addition, as there may be many clients behind one NAT gateway, there may be multiple RADIUS clients using one public IP address.  RADIUS servers need to support multiple PSK identifiers at one source IP address.

That is, a server needs to support multiple different clients within one network range, multiple clients behind a NAT, and one client having different IP addresses over time.  All of those use-cases are common and necessary.

The following section describes these requirements in more detail.

### Requirements for TLS-PSK

A server supporting this specification MUST be configurable with a set of "allowed" network ranges from which clients are permitted to connect.  Any connection from outside of the allowed range(s) MUST be rejected before any PSK identity is checked.

Once a connection has been made from a permitted source, the server MUST use the PSK identity as the logical identifier for a RADIUS client instead of the IP address as was done with RADIUS/UDP.  The PSK identity is then looked up in the local "client table" which was described above.

Servers implementing this specification SHOULD also be able to associate an IP range or ranges for each client.  Any connection from outside the allowed range(s) MUST be rejected, even if the PSK identity is known, and before the PSK is verified.

Note that this lookup is independent from the "allowed" network ranges which are checked when the TLS connection is made.  The two range limitations can overlap completely, or only partially.  In addition, the allowed network range(s) for any two clients may overlap partially, completely, or not at all.  All of these possibilities MUST be supported by the server implementation.  That is, it is possible for multiple clients to have the same allowed source IP address or range.

Once the source IP has been verified to be allowed for this particular client, the server authenticates the TLS connection via the PSK taken from the client definition.  If the PSK is verified, the server then accepts the connection, and proceeds with RADIUS/TLS as per {{RFC6614}}.

This process satisfies all of the requirements of the previous section.

Finally, if a RADIUS server does not recognize the PSK identity or if the identity is not permitted to use PSK, then the server MAY proceed with a certificate-based handshake.  Since TLS 1.3 {{RFC8446}} uses PSK for resumption, any unrecognized PSK identity MUST result either in a full certificate-based TLS handshake being performed, or in the TLS connection being rejected.

## Interaction with Certificates

Server implementation SHOULD be able to authenticate clients using both TLS-PSK and client certificates.  Server implementations SHOULD be able to support both PSK and server authenticated TLS connections in the same instance.

# Privacy Considerations

We make no changes over {{RFC6614}} and {{RFC7360}}.

# Security Considerations

The primary focus of this document is addressing security considerations for RADIUS.

# IANA Considerations

There are no IANA considerations in this document.

RFC Editor: This section may be removed before final publication.

# Acknowledgments

Thanks to the many reviewers in the RADEXT working group for positive feedback.

# Changelog

* 00 - initial version

* 01 - update examples

--- back
