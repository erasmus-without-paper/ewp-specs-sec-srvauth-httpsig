Authenticating Servers with HTTP Signature
==========================================

This document describes how EWP server authentication can be accomplished with
signatures included in HTTP responses.

* [What is the status of this document?][statuses]
* [See the index of all other EWP Specifications][develhub]


Introduction
------------

This authentication method makes use of:

 * [Standard TLS Server Certificate Authentication][srvauth-tlscert].
 * The `Digest` header, specified in [RFC 3230][digest-base].
 * The `SHA-256` Digest Algorithm, specified in [RFC 5843][digest-sha256].
 * The `Signature` header, specified in
   [draft-cavage-http-signatures-07][httpsig-signature] (presumably,
   [soon to become RFC](https://github.com/erasmus-without-paper/ewp-specs-architecture/issues/17#issuecomment-286142084)),
   along with its `rsa-sha256` signature algorithm.

Note, that - in many cases - [TLS Server Certificate
Authentication][srvauth-tlscert] is enough to provide sufficient authentication
security. However, HTTP Signatures provide much easier non-repudiation support.
See *Security Considerations* section below for more information.


Implementing a server
---------------------

### Meet all "inherited" requirements

This server authentication method **extends** the standard *TLS Server
Certificate Authentication* method. The servers are REQUIRED to meet all the
requirements listed [here][srvauth-tlscert].

In particular, it is RECOMMENDED for servers to declare their support for *TLS
Server Certificate Authentication*. For example, if they use
`sec:HttpSecurityOptions` datatype to describe their security requirements,
then they should have *TLS Server Certificate Authentication* listed among
their `<sec:client-auth-methods>` elements. This will allow clients which do
not support *HTTP Signature Server Authentication* to connect to your endpoints
in a backward-compatible way.


### Generate a key-pair

You need to generate an RSA key-pair for your server. You MAY use the same
key-pair you use for your TLS communication if you want to.


### Publish your key

Each partner declares (in his [Manifest file][discovery-api]) a list of public
keys which its servers can use for signing their responses. This list is later
fetched by registry, and the keys (and/or their fingerprints) are served to all
other partners see (see [Registry API][registry-api] for details).

Usually (but not necessarily always) you will bind a single public key to all
endpoints you serve. Once the client confirms that the server is in possession
of the matching private key, it is then able to identify (with the help of the
Registry again) if this key-pair has been listed as one with which server
authentication at this endpoint can be performed.

Note, that the Registry will verify if your keys meet certain security
standards (i.e. their length). These standards MAY change in time. Remember to
include `<admin-email>` elements in your manifest file if you want to be
notified about such changes.


### Check if you *need* to sign (optional)

This step is RECOMMENDED, but not required. If you decide *not* to implement
it, then you need to sign all your responses (even if the client doesn't want
you to).

A single server endpoint MAY support multiple server authentication methods.
For this reason, the *client* notifies the server which of these methods it
wants to use.

For example, if your endpoint supports both `<tlscert/>` and `<httpsig/>`, then
the client might be okay with using `<tlscert/>` and you are NOT REQUIRED to
sign your response in this case.

How does the client notify you if it wants your signature? Since we couldn't
find any standard header for expressing that, we have invented our own:

 * If the request contains the `Accept-Signature` header (case insensitive),
   with one of the comma-separated values equal to `rsa-sha256` (case
   insensitive again), then you MUST sign your response.

   Note, that this header MAY contain multiple algorithms. You are required to
   support only `rsa-sha256`.

 * If the request contains the `Accept-Signature` header, but you don't support
   any of the algorithms requested there, then you SHOULD ignore such signature
   request. It is RECOMMENDED to proceed as if the client didn't send any
   `Accept-Signature` header.

 * If the request doesn't contain the `Accept-Signature` header (or you don't
   support any of the requested algorithms), then you are NOT REQUIRED to sign
   your response (and you don't need to follow the rest of the steps below).
   This simply means that the client won't verify your signature, so you don't
   need to bother and sign it (but you MAY, if you wish).

   Note, that you SHOULD NOT force your client to send the `Accept-Signature`
   header. You SHOULD allow your client to *not* include the `Accept-Signature`
   header, and therefore "fall back" to the regular *TLS Server Certificate
   Authentication*.


### Include all required headers

 * You MUST include either the `Date` or [`Original-Date`][original-date-header]
   header in your  response. You MAY include both of them. The format of the
   `Original-Date` header, if included, MUST match the "regular" format of the
   `Date` header, as defined in [RFC 2616][date-header]. You MUST make sure
   that your clock is synchronized (otherwise the client may reject your
   response).

 * You MUST include the `Digest` header in your response, as explained
   [here][digest-base]. If the client sent you a `Want-Digest` header, then you
   MAY take it into account when choosing your digest algorithm, but you are
   REQUIRED to support only one digest algorithm - `SHA-256`.

   If the client didn't send a `Want-Digest` header, then you still MUST
   include the digest (in the `SHA-256` algorithm).

   Also note, that your `Digest` header MAY contain many digests, in different
   algorithms.

 * If the client's request contained the `X-Request-Id` header, then you MUST
   include the `X-Request-Id` header with the same contents in your response.

 * If [HTTP Signature method][cliauth-httpsig] has been used for client
   authentication, then you MUST include the `X-Request-Signature` header in
   your response, with the value copied from the `signature` parameter used in
   the request's `Authorization` header.


### Sign your response

You MUST include the `Signature` header in your response, as explained
[here][httpsig-signature]. You MUST support the `rsa-sha256` signature
algorithm. You MAY use a different algorithm, if the client listed this
algorithm in its `Accept-Signature` header. You MUST include **at least** the
following values in your `headers` parameter:

 - `date` or `original-date` (it MUST contain at least one of those, it also
   MAY contain both),
 - `digest`,
 - `x-request-id`, but only when you actually include this header in your
   response,
 - `x-request-signature`, but only when you actually include this header in
   your response.

If it is important for the client to recognize any other of your headers, then
you MUST sign all these headers too. The headers mentioned above are important
for handling authentication, non-repudiation and security described in this
document, but other headers can also be very important in your case.

The `keyId` parameter of the `Signature` header MUST contain a lower-case
HEX-encoded SHA-256 fingerprint of the *binary public key* part of the kay-pair
which you have used to sign your response. It MUST match one of the keys you
previously published in your manifest file.


### Take care to *not* modify it again

Many frameworks or proxies might try to automatically modify your response
*after* you sign it. For example, they may try to add additional `gzip` coding
to your response's `Content-Encodings` if they detect that the client supports
it. In many cases, this would be a good thing, but in this case, such changes
could break your HTTP Signature (because we sign the content *after* it has
been encoded). Make sure that you disable such automatic modifications when you
use HTTP Signatures for signing.

On this topic, also read the chapter about the `Original-Date` header below.


Implementing a client
---------------------

### Meet all "inherited" requirements

This server authentication method **extends** the "standard" TLS Server
Certificate Authentication method. The clients are REQUIRED to meet all the
requirements listed [here][srvauth-tlscert].


### Include `Want-Digest` header (optional)

You MAY include the `Want-Digest` header in your request, as explained
[here][want-digest]. If included, then one of the digest algorithms included
SHOULD be `SHA-256` (because it's currently the only algorithm required by the
servers to support).

If you don't include this header, servers will assume it's equal to:

```http
Want-Digest: SHA-256
```


### Include `Accept-Signature` header

You MUST include the `Accept-Signature` header in your request. This header
informs the server that you want it to sign its responses with HTTP Signatures.
The value contains a comma-separated list of signature algorithms, ordered by
client's preference. One of these algorithms SHOULD be `rsa-sha256`.

Currently, `rsa-sha256` is the only algorithm required to be implemented by
servers. If you don't include it in your `Accept-Signature` list, then the
server MAY not sign the response at all.


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

   - `date` or [`original-date`][original-date-header] (it MUST contain at
     least one of those, it also MAY contain both),
   - `digest`,
   - `x-request-id`, but only if you actually included this header in your
     request,
   - `x-request-signature`, but only if you used [HTTP Signature Client
     Authentication][cliauth-httpsig] in your request.

If some of these conditions are not met, then you MUST reject the server's
response.


### Verify the date(s)

You need to parse and verify the values of the `Date` and
[`Original-Date`][original-date-header] headers, if they are included in the
response (at least one of them MUST be). In particular, you MUST verify at
least the ones which have been listed in the response's `Signature` header, but
it is RECOMMENDED to verify both (if both are included in the response).

Verification process consists of parsing the date, and matching it against the
date reported by your own clock. The verification fails if:

 * The date cannot be parsed.

 * The date does not match your clock **within a certain threshold of time**.

   - It is RECOMMENDED to use the **5 minutes** threshold.
   - You MAY choose a greater threshold than 5 minutes, but you MUST NOT choose
     a lower threshold than this.

If the verification fails, then you MUST reject the server's response.

Also note, that you MUST make sure that your clock is synchronized (otherwise
your verification process will fail constantly).


### Look up the key

Extract the `keyId` from the response's `Signature` header.

It is supposed to contain a lower-case HEX-encoded SHA-256 fingerprint of the
RSA binary public key published by the server in the Registry. If it doesn't,
then you MUST reject the server's response.


### Verify the signature

You MUST verify the request's signature, as explained
[here][verifying-signature].

If the signature is invalid, then you MUST reject the server's response.


### Verify the digest

Calculate the Base64-encoded `SHA-256` digest of the response's body, according
to [RFC 3230][digest-base] and [RFC 5843][digest-sha256]. Compare it to the
`Digest` header which MUST be present in the response. The values MUST match.
If they don't, then you MUST reject the server's response.

Remember, that the `Digest` header MAY contain many digests, in different
algorithms. You MUST parse the header and extract the `SHA-256` algorithm from
this list.


### Ignore unsigned headers

Clients MUST ignore all response headers which hadn't been signed by the
server. They might have been added by the attacker in transport. This is
important especially in cases when TLS is not used in some parts of the
transport, or when you don't fully trust the partner's TLS implementation.

The safest way to properly ignore such headers is to modify your `Response`
object *now* (during the authentication and authorization process), by either
removing the suspicious headers, or at least changing their name (e.g.
prepending it with `Unsigned-`). Then, pass the modified `Response` along as the
result of your verification, so that the actual client you are implementing
believes that the server didn't supply these headers at all. This approach
is much safer than trusting yourself to remember to verify this every time
before you access every header in every single ones of your underlying clients
(and some clients might be dependent on response's headers).


<a name="original-date"></a>

About the `Original-Date` header
--------------------------------

While testing EWP HTTP Signature specifications (both of them), some developers
[reported](https://github.com/erasmus-without-paper/ewp-specs-sec-srvauth-httpsig/issues/1)
that the values of their `Date` headers got replaced by proxies, thus
invalidating their signatures. The `Original-Date` has been introduced in this
specification as a measure to circumvent this problem.

 * If your proxy is replacing the `Date` header, and you cannot reliably
   reconfigure it to not do so, then you MAY use the `Original-Date` header as
   a replacement for the `Date` header. The format of the `Original-Date`
   header, if included, MUST match the "regular" format of the `Date` header,
   as defined in [RFC 2616][date-header].

 * If your proxy is not replacing the `Date` header, or you can reliably
   reconfigure your proxy to not do so, then it is RECOMMENDED to use the
   `Date` header only, and to NOT include the `Original-Date` header in signed
   payloads.


Security considerations
-----------------------

### Non-repudiation

The `Signature` header proves that the response has been generated by a
particular server, but it doesn't say anything about the requester and request
for which the response has been produced.  The `Date` (`Original-Date`),
`X-Request-Id` and `X-Request-Signature` headers are required in order to
enhance non-repudiation in request-response correlation:

 * Including a signed `Date` or [`Original-Date`][original-date-header] header
   in the response serves as a proof, that the response has indeed been
   generated by a particular server in a particular time.

 * Including a signed `X-Request-Id` in the response serves as a proof, that
   the response is indeed correlated to the given request.

 * Including a signed `X-Request-Signature` in the response serves as a proof,
   that the request has indeed been sent by the particular client, and received
   by the particular server.

 * Note, that the client needs to archive all requests and responses (along
   with their headers) in order for this proofs to work. He also needs to [use
   HTTP Signature for client authentication][cliauth-httpsig].


### Main security questions

The [Authentication and Security][sec-intro] document
[recommends][sec-method-rules] that each server authentication method
specification explicitly answers the following questions:

> How the client's request must look like? How can the server know, that the
> client *wants the server* to use this particular method of authentication?

The server knows that this method is used, when the request contains the
`Accept-Signature` header.

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
[want-digest]: https://tools.ietf.org/html/rfc3230#section-4.3.1
[digest-sha256]: https://tools.ietf.org/html/rfc5843#section-2.2
[httpsig-signature]: https://tools.ietf.org/html/draft-cavage-http-signatures-07#section-4
[date-header]: https://tools.ietf.org/html/rfc2616#section-14.18
[verifying-signature]: https://tools.ietf.org/html/draft-cavage-http-signatures-07#section-2.5
[cliauth-none]: https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-none
[cliauth-tlscert]: https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-tlscert
[cliauth-httpsig]: https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-httpsig
[srvauth-tlscert]: https://github.com/erasmus-without-paper/ewp-specs-sec-srvauth-tlscert
[srvauth-httpsig]: https://github.com/erasmus-without-paper/ewp-specs-sec-srvauth-httpsig
[x-request-id]: https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-httpsig#headers
[sec-method-rules]: https://github.com/erasmus-without-paper/ewp-specs-sec-intro#rules
[sec-intro]: https://github.com/erasmus-without-paper/ewp-specs-sec-intro
[original-date-header]: https://github.com/erasmus-without-paper/ewp-specs-sec-srvauth-httpsig#original-date
[registry-api]: https://github.com/erasmus-without-paper/ewp-specs-api-registry
