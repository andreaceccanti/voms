---
layout: post
title: VOMS Admin server v. 3.3.1 and VOMS server v. 2.0.12-2
author: andrea
summary: Support for configurable selection of enabled SSL protocols and ciphers in VOMS-Admin server and packaging fixes
---

This release fixes a few issues in the VOMS packages released on October, 24th.
In particular:

- [VOMS-Admin server 3.3.1][rn-admin-server] provides support for the
  configurable selection of enabled SSL protocols and cipher suites. VOMS-Admin
  v. 3.3.0 disabled SSL3 by default, but we have found out that mkgridmap
  clients would stop working with that setting. This version does not disable
  SSLv3 by default, but provides the ability to do it at a later time by
  changing a configuration parameter. More details in the [release
  notes][rn-admin-server].

- [VOMS server 2.0.12-2][rn-core] fixes a problem in the RPM packaging that
  prevented mock builds to behave correctly. More details in the [release
  notes][rn-core].

As usual packages can be obtained from our repositories and will soon be pushed to the
EMI-3 and UMD repositories. For instructions, refer to the [releases section][releases].

Besides the release, we have restructured the web site a bit in order to
provide more consistent access to release notes and documentation.

[rn-core]: {{site.baseurl}}/release-notes/voms-server/2.0.12-2
[rn-admin-server]: {{site.baseurl}}/release-notes/voms-admin-server/3.3.1
[releases]: {{site.baseurl}}/releases.html
