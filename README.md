# RADIUS and TLS-PSK

This document gives implementation and operational considerations for using TLS-PSK with RADIUS.

[RFC6614](https://www.rfc-editor.org/rfc/rfc6614) and [RFC7360](https://www.rfc-editor.org/rfc/rfc7360) define TLS and DTLS transports for [RADIUS](https://www.rfc-editor.org/rfc/rfc2865).  However, neither of those documents discuss how to use TLS-PSK.

While [FreeRADIUS](http://freeradius.org) has implemented TLS-PSK for nearly a [decade](https://github.com/FreeRADIUS/freeradius-server/commit/20b0712d537), its use is not wide-spread.

This document is for the IETF RADEXT WG
http://datatracker.ietf.org/wg/radext
