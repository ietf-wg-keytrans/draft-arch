---
title: "Key Transparency Architecture"
category: info

docname: draft-mcmillion-keytrans-architecture-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Key Transparency"
keyword:
 - key transparency
venue:
  group: "Key Transparency"
  type: "Working Group"
  mail: "keytrans@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/keytrans/"
  github: "Bren2010/draft-kt-arch"
  latest: "https://bren2010.github.io/draft-kt-arch/draft-mcmillion-keytrans-architecture.html"

author:
 -
    fullname: Brendan McMillion
    email: brendanmcmillion@gmail.com

normative:

informative:


--- abstract

This document defines the terminology and interaction patterns involved in the
deployment of Key Transparency (KT) in a general secure group messaging
infrastructure, and specifies the security properties that the protocol
provides. It also gives more general, non-prescriptive guidance on how to
securely apply KT to a number of common applications.


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

**End-to-end Encrypted Communication Service:**
: A communications service that allows end-users to engage in text, voice,
  video, or other forms of communication over the Internet, that uses public key
  cryptography to ensure that communications are only accessible to their
  intended recipients.

**End-user Device:**
: The device at the final point in a digital communication, which may either
  send or receive encrypted data in an end-to-end encrypted communication
  service.

**End-user Identity:**
: A unique and user-visible identity associated with an account (and therefore,
  one or more end-user devices) in an end-to-end encrypted communication
  service. In the case where an end-user explicitly requests to communicate with
  (or is informed they are communicating with) an end-user uniquely identified
  by the name "Alice", the end-user identity is the string "Alice".

**Service Provider:**
: The primary organization that provides the infrastructure and software
  resources necessary to operate an end-to-end encrypted communication service.

**Authentication Service:**
: A specialized service capable of securely attesting to the information (such
  as public keys) associated with a given end-user identity. The authentication
  service is run either entirely or partially by the Service Provider.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
