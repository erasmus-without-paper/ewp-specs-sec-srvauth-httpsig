Authenticating Servers with HTTP Signature
==========================================

This document describes how EWP server authentication can be accomplished with
signatures included in HTTP responses.

* [What is the status of this document?][statuses]
* [See the index of all other EWP Specifications][develhub]


Status
------

This document will **eventually** describe a new server authentication protocol.
However, at the moment, this is still a DRAFT, and SHOULD NOT be implemented
yet.

Note, that this **will not** entirely replace [TLS Server Certificate
Authentication][srvauth-tlscert] method which we use now. If will **probably**
only be RECOMMENDED to be supported as an addition to that. This still requires
further discussion.


Introduction
------------

This authentication method makes use of:

 * [Standard TLS Server Certificate Authentication][srvauth-tlscert].
 * The `Digest` header, specified in [RFC 3230][digest-base].
 * The `SHA-256` Digest Algorithm, specified in [RFC 5843][digest-sha256].
 * The `Signature` header, specified in
   [draft-cavage-http-signatures-06][httpsig-signature] (presumably,
   [soon to become RFC](https://github.com/erasmus-without-paper/ewp-specs-architecture/issues/17#issuecomment-286142084)),
   along with its `rsa-sha256` signature algorithm.

Note, that - in many cases - [TLS Server Certificate
Authentication][srvauth-tlscert] is enough to provide sufficient authentication
security. However, HTTP Signatures provide much easier non-repudiation support.
See *Security Considerations* section below for more information.


Implementing a server
---------------------

### Meet all "inherited" requirements

This server authentication method **extends** the standard TLS Server
Certificate Authentication method. The servers are REQUIRED to meet all the
requirements listed [here][srvauth-tlscert].


### Generate a key-pair

You need to generate an RSA key-pair for your server. You MAY use the same
key-pair you use for your TLS communication if you want to.


### Publish your key

You MUST publish your public key in the Registry, via the Manifest file.

**Important:** This is a DRAFT. Currently the Registry does not serve any
information on server keys. But it would need to, before this authentication is
introduced. We still need to design how this would look like.


### Include all required headers

 * You MUST include the `Date` header in your response, as defined in [RFC
   2616][date-header]. You MUST make sure that your clock is synchronized.

 * You MUST include the `Digest` header in your request, as explained
   [here][digest-base]. You MUST use SHA-256 algorithm for generating the
   digest.

 * If the client's request contained the `X-Request-Id` header, then you MUST
   include the `X-Request-Id` header with the same contents in your response.

 * If [HTTP Signature method][cliauth-httpsig] has been used for client
   authentication, then you MUST include the `X-Request-Signature` header in
   your response, with the value copied from the `signature` parameter used in
   the request's `Authorization` header.


### Sign your response

You MUST include the `Signature` header in your response, as explained
[here][httpsig-signature]. You MUST use the `rsa-sha256` signature algorithm,
and you MUST include **at least** the following values in your `headers`
parameter:

 - `date`,
 - `digest`,
 - `x-request-id`, but only when you actually include this header in your
   response,
 - `x-request-signature`, but only when you actually include this header in
   your response.

If it is important for the client to recognize any other of your headers, then
you MUST sign all these headers too. The headers mentioned above are important
for handling authentication, non-repudiation and security described in this
document, but other headers can also be very important in your case.

The `keyId` parameter of the `Signature` header MUST contain a Base64-encoded
RSA public key (NOT its fingerprint, the entire key). If MUST be one of the
keys you previously published in your manifest file.


Implementing a client
---------------------

### Meet all "inherited" requirements

This server authentication method **extends** the "standard" TLS Server
Certificate Authentication method. The clients are REQUIRED to meet all the
requirements listed [here][srvauth-tlscert].


### Consider signing yourself

This method of authentication works best when both parties use it. Consider
using [HTTP Signature Client Authentication][cliauth-httpsig] (if the server
supports it). If you do, then the server will include additional headers in the
response which will enhance non-repudiation.

If you don't want to use HTTP Signature Client Authentication (or you can't use
it because the server doesn't support it), then it is still RECOMMENDED to
include `X-Request-Id` header (as described [here][x-request-id]) for
request-response correlation. See *Security Considerations* section below.


### Verify `X-Request-Id`

If you have included `X-Request-Id` in your request, then you SHOULD verify
if the value returned in the response's `X-Request-Id` header matches the value
use sent along in your request. If it doesn't, then you MUST reject the
server's response.


### Verify signature parameters

Make sure that:

 * The server's response contains the `Signature` header.

 * The `algorithm` parameter of the `Signature` header is equal to
   `rsa-sha256`. You MAY support other algorithms too, but `rsa-sha256` is
   (currently) the only one required.

 * The `headers` parameter of the `Signature` header contains **at least**
   the following values:

   - `date`,
   - `digest`,
   - `x-request-id`, but only if you actually included this header in your
     request,
   - `x-request-signature`, but only if you used [HTTP Signature Client
     Authentication][cliauth-httpsig] in your request.

If some of these conditions are not met, then you MUST reject the server's
response.


### Verify the date

You SHOULD parse and verify the value of the `Date` header included in the
response.

If the date does not match your own clock **within a threshold of 5
minutes**, then you SHOULD reject the server's response.

Your clock MUST be properly synchronized for this verification to work
properly.


### Look up the key

Extract the `keyId` from the response's `Signature` header.

It MUST contain a Base64-encoded RSA public key (we use `keyId` parameter to
transfer to *actual key*, not its ID). If it doesn't, then you MUST reject the
server's response.

You are expecting the response to be signed with a very specific key. The key's
fingerprint MUST match the fingerprint published by the server in the Registry.
If it doesn't, then you MUST reject the server's response.

**Important:** This is a DRAFT. Currently, the Registry doesn't allow server
implementers to publish this key.


### Verify the signature

You MUST verify the request's signature, as explained
[here][verifying-signature].

If the signature is invalid, then you MUST reject the server's response.


### Verify the digest

Calculate the Base64-encoded SHA-256 digest of the response's body, according
to [RFC 3230][digest-base] and [RFC 5843][digest-sha256]. Compare it to the
`Digest` header which MUST be present in the response. The values MUST match.
If they don't, then you MUST reject the server's response.



Security considerations
-----------------------

### Non-repudiation

The `Signature` header proves that the response has been generated by a
particular server, but it doesn't say anything about the requester and request
for which the response has been produced.  The `Date`, `X-Request-Id` and
`X-Request-Signature` headers are required in order to enhance non-repudiation
in request-response correlation:

 * Including a signed `Date` in the response serves as a proof to the server,
   that the response has indeed been generated by a particular server in a
   paricular time.

 * Including a signed `X-Request-Id` in the response serves as a proof to the
   client, that the response is indeed correlated to the given request.

 * Including a signed `X-Request-Signature` in the response serves as a proof
   to the server, that the request has indeed been sent by the particular
   client, and received by the particular server.

 * Note, that the client needs to archive all requests and responses (along
   with their headers) in order for this proofs to work. He also needs to [use
   HTTP Signature for client authentication][cliauth-httpsig].


### Main security questions

The [Authentication and Security][sec-intro] document
[recommends][sec-method-rules] that each server authentication method
specification explicitly answers the following questions:

> How can the client verify the server's identity?

Since this method builds on TLS, the client uses regular HTTPS server
certificate validation to make sure that it is communicating with the proper
domain and URL.

Additionally, the client also verifies the key with which HTTP headers were
signed. The key-HEI relationship is retrieved securely from the Registry.

> How can the client verify that the response has not been tampered with? Can
> it also verify that it was indeed generated for this particular request?

TLS prevents communication to be tampered with during the transport.
Additionally, HTTP Signature prevents it from being tampered behind the TLS
terminator (useful in cases when the server's internal network is not secure).

If the client includes a random `X-Request-Id` header in his requests, then it
also gets the proof that the response has been generated for this particular
request.

> Does it provide non-repudiation? Can a client provide a solid proof later
> on, that the server sent a particular response (in response to a particular
> client's request)?

Yes. See *Non-repudiation* section above.


[discovery-api]: https://github.com/erasmus-without-paper/ewp-specs-api-discovery
[develhub]: http://developers.erasmuswithoutpaper.eu/
[statuses]: https://github.com/erasmus-without-paper/ewp-specs-management/blob/stable-v1/README.md#statuses
[digest-base]: https://tools.ietf.org/html/rfc3230#section-4.3.2
[digest-sha256]: https://tools.ietf.org/html/rfc5843#section-2.2
[httpsig-signature]: https://tools.ietf.org/html/draft-cavage-http-signatures-06#section-4
[date-header]: https://tools.ietf.org/html/rfc2616#section-14.18
[verifying-signature]: https://tools.ietf.org/html/draft-cavage-http-signatures-06#section-2.5
[cliauth-none]: https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-none
[cliauth-tlscert]: https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-tlscert
[cliauth-httpsig]: https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-httpsig
[srvauth-tlscert]: https://github.com/erasmus-without-paper/ewp-specs-sec-srvauth-tlscert
[srvauth-httpsig]: https://github.com/erasmus-without-paper/ewp-specs-sec-srvauth-httpsig
[x-request-id]: https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-httpsig#headers
[sec-method-rules]: https://github.com/erasmus-without-paper/ewp-specs-sec-intro#rules
[sec-intro]: https://github.com/erasmus-without-paper/ewp-specs-sec-intro
