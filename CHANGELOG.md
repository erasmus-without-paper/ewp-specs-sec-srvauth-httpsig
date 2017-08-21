Release notes
=============

This document describes all the changes made to the *Authenticating Servers
with HTTP Signature* document, starting from its first released version.


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
