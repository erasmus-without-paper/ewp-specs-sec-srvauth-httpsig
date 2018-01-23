Release notes
=============

This document describes all the changes made to the *Authenticating Servers
with HTTP Signature* document, starting from its first released version.


1.0.1
-----

* Added a notice for the client implementers to *ignore* unsigned response
  headers. (This hasn't been previously stressed enough, and it could lead to
  security vulnerabilities.)

* Added a notice for server implementers to take care not to allow their
  frameworks and proxies modify the response after it has been signed (as this
  could break the signature).


1.0.0
-----

Upgraded to a stable version.


0.4.0
-----

* Clarified, that it is RECOMMENDED for the servers to also declare their
  support for *TLS Server Certificate Authentication* (whenever they support
  *HTTP Signature Server Authentication*).

* Changed the expected behavior when no supported algorithms are found in the
  client's `Accept-Signature` header. Previously, servers were recommended to
  respond with HTTP 400. Now, servers are recommended to ignore such signature
  request (and proceed as if no `Accept-Signature` header was sent).

* Explicitly stated that if the client includes the `Want-Digest` header with
  an algorithm other than `SHA-256`, then the server MAY take the client's
  wishes into account when choosing its digest algorithm. This step is
  optional - the server MAY also simply always use `SHA-256`.

* Explicitly stated that if the client includes the `Accept-Signature` header
  with an algorithm other than `rsa-sha256`, then the server MAY take the
  client's wishes into account when choosing its signature algorithm. This step
  is optional - the server MAY also simply always use `rsa-sha256`.

* Added a reminder that the `Digest` header may contain multiple digests.


0.3.0
-----

* Use public key digests instead of actual keys in `keyId` parameter of the
  `Signature` header (see
  [this issue](https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-httpsig/issues/1)).

* Allow `Original-Date` to be used in place of the `Date` header (see
  [this issue](https://github.com/erasmus-without-paper/ewp-specs-sec-srvauth-httpsig/issues/1)).


0.2.0
-----

* Make it usable with the v2 security. Described how the key needs to be
  published, how the server detects if the client is requesting this type of
  server authentication, etc. In particular, the usage of `Accept-Signature`
  header is now REQUIRED.

* Add `security-entries.xsd` file (which describes the XML element used to
  identify this method of authentication in the manifest files, and the
  Registry).

* Upgraded `draft-cavage-http-signatures-06` to
  `draft-cavage-http-signatures-07`. Still no RFC though.


0.1.0
-----

Initial revision.
