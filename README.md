# Network Time Security in Go

Network Time Security (NTS) is a development of the venerable Network
Time Protocol (NTP). NTS defines a separate Network Time Security Key
Establishment (NTS-KE) protocol and [uses extension
fields](https://tools.ietf.org/html/rfc7822) in NTPv4.

This is an attempt to implement NTS-KE and SNTP with NTS extension
fields according to the [Dansarie
specification](https://datatracker.ietf.org/doc/draft-dansarie-nts/?include_text=1)
in Go.

The NTS-KE is implemented as its own package for use in both NTS-KE
clients and servers. An example client and server is provided. The
client is also an SNTP client.

We also provide the beginning of an NTP server but it doesn't yet
support any NTS extensions and is mostly cobbled together from
internal structures in [Brett Vickers NTP
library](https://github.com/beevik/ntp/), but it's a start.

The NTS-KE client has been tested against Martin Langer's server
implementation on `nts2-e.ostfalia.de` which fortunately for uss also
speaks TLS 1.2 even though the spec explicitly says 1.3.

## Background

The current project was started in two days during the [102 IETF
Hackathon](https://trac.ietf.org/trac/ietf/meeting/wiki/102hackathon).
Michael "MC" Cardell Widerkrantz, Daniel "quite" Lublin, omni and
racoon gathered as remote participants in the hackathon in central
Malmö with lots of Club-Mate. Lots of specification reading, some
false starts and kludgy code were the results.

MC wrote [a blog post](https://hack.org/mc/blog/nts.html) about the
hackathon that you might want to read.

## NTS Architecture

1. The NTS-KE client initiates traffic. It initiates an application
   layer TLS session to the NTS-KE server and sends a selection of
   AEAD algorithms.

2. The NTS-KE server lists its selection of AEAD algorithms. If the
   client and server have overlapping algorithms they continue to
   export keying materials from the TLS session. Now they both have a
   pair of keys for use in each direction in NTP.

   The server also creates, encrypts, and sends initial cookies for
   later use. These cookies are opaque to the client.

3. The NTS-KE client might also be an NTP client. It can now ask for
   the time with the cookie it got from NTS-KE and using the C2S key
   for encrypted fields.
   
4. The NTP server gets the NTP request, unpacks the cookie andd can
   now reply to the NTP client with the current time, encrypted with
   the server->client key that was in the cookie (see below).
   
   Since all information it needs is contained in the cookie it
   doesn't need to keep any state about the client.

There are several keys involved here:

1. A private key for the server's X.509 certificate.

2. A session key used in the TLS session.

3. A negotiated pair of keys for later NTP C2S and S2S communication.

2. A server-side key used to encrypt cookies.

## About cookies

Cookies are generated by the server, then encrypted by a server-only
master secret. The NTS-KE and NTP servers should both share the secret
used for cookie encryption.
   
The cookies contain:
   
in encrypted form:
 - the negotiated AEAD algorithm, 
 - the S2C key
 - the C2S key.

in plaintext:

 - identifier of the server-side secret used to encrypt this cookie.

Note for this to work the cookies *should not* be encrypted by the C2S
key when sent over NTP!

The NTP client sends boths a cookie and asks for a new cookie with
every request. It will use the new cookie on the next request.