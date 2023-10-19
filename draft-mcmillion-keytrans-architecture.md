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
  RFC6962: DOI.10.17487/RFC6962

informative:


--- abstract

This document defines the terminology and interaction patterns involved in the
deployment of Key Transparency (KT) in a general secure group messaging
infrastructure, and specifies the security properties that the protocol
provides. It also gives more general, non-prescriptive guidance on how to
securely apply KT to a number of common applications.

--- middle

# Introduction

Before any information can be exchanged in an end-to-end encrypted system, two
things must happen. First, participants in the system must provide the service
operator with any public keys they wish to use to receive messages. Second, the
service operator must somehow distribute these public keys amongst the
participants that wish to communicate with each other.

Typically this is done by having users upload their public keys to a simple
directory where other users can download them as necessary, or by providing
public keys in-band with the communication being secured. With this approach,
the service operator needs to be trusted to provide the correct public keys,
which means that the underlying encryption protocol can only protect users
against passive eavesdropping on their messages.

However most messaging systems are designed such that all messages exchanged
between users flow through the service operator's servers, so it's extremely
easy for an operator to launch an active attack. That is, the service operator
can provide fake public keys which it knows the private keys for, associate
those public keys with a user's account without the user's knowledge, and then
use them to impersonate or eavesdrop on conversations with that user.

Key Transparency (KT) solves this problem by requiring the service operator to
store user public keys in a cryptographically-protected append-only log. Any
malicious entries added to such a log will generally be visible to all
users, in which case a user can detect that they're being impersonated
by viewing the public keys attached to their account. However, if the service
operator attempts to conceal some entries of the log from some users but not
others, this creates a "forked view" which is permanent and easily detectable
with out-of-band communication.

The critical improvement of KT over related protocols like Certificate
Transparency  {{RFC6962}} is that KT includes an efficient
protocol to search the log for entries related to a specific participant. This
means users don't need to download the entire log, which may be substantial, to
find all entries that are relevant to them. It also means that KT can better
preserve user privacy by only showing entries of the log to participants that
genuinely need to see them.


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
: A unique and user-visible identity associated with an account (and therefore
  one or more end-user devices) in an end-to-end encrypted communication
  service. In the case where an end-user explicitly requests to communicate with
  (or is informed they are communicating with) an end-user uniquely identified
  by the name "Alice", the end-user identity is the string "Alice".

**User / Account:**
: A single end-user of an end-to-end encrypted communication service, which may
  be represented by several end-user identities and end-user devices. For
  example, a user may be represented simultaneously by multiple identities
  (email, phone number, username) and interact with the service on multiple
  devices (phone, laptop).

**Service Operator:**
: The primary organization that provides the infrastructure and software
  resources necessary to operate an end-to-end encrypted communication service.

**Transparency Log:**
: A specialized service capable of securely attesting to the information (such
  as public keys) associated with a given end-user identity. The transparency
  log is run either entirely or partially by the service operator.


# Protocol Overview

From a networking perspective, KT follows a client-server architecture with a
central *Transparency Log*, acting as a server, which holds the authoritative
copy of all information and exposes endpoints that allow users to query or
modify stored data. Users coordinate with each other through the server by
uploading their own public keys and/or downloading the public keys of other
users. Users are expected to maintain relatively little state, limited only
to what is required to interact with the log and ensure that it is behaving
honestly.

From an application perspective, KT works as a versioned key-value database.
Users insert key-value pairs into the database where, for example, the key is
their username and the value is their public key. Users can update a key by
inserting a new version with new data. They can also look up the most recent
version of a key or any past version. Users are considered to **own** a key if,
in the normal operation of the application, they should be the only one making
changes to it. From this point forward, "key" will refer to a lookup key in a
key-value database and "public key" or "private key" will be specified if
otherwise.

KT does not require the use of a specific transport protocol. This is intended
to allow applications to layer KT on top of whatever transport protocol their
application already uses. In particular, this allows applications to continue
relying on their existing access control system.

Applications may enforce arbitrary access control rules on top of KT such as
requiring a user to be logged in to make KT requests, only allowing a user to
lookup the keys of another user if they're "friends", or simply applying a rate
limit. Applications SHOULD prevent users from modifying keys that they don't
own. The exact mechanism for rejecting requests, and possibly explaining the
reason for rejection, is left to the application.

# User Interactions

As discussed in {{protocol-overview}}, KT follows a client-server architecture.
This means that all user interaction is directly with the service operator. The
operations that can be executed by a user are as follows:

1. **Search:** Performs a lookup on a specific key in the most recent version of
   the log. Users may request either a specific version of the key, or the
   most recent version available. If the key-version pair exists, the server
   returns the corresponding value and a proof of inclusion.
2. **Update:** Adds a new key-value pair to the log, for which the server
   returns a proof of inclusion. Note that this means that new values are added
   to the log immediately and are not queued for later insertion with a batch of
   other values.
3. **Monitor:** While Search and Update are run by the user as necessary,
   monitoring is done in the background on a recurring basis. It both checks
   that the log is continuing to behave honestly (all previously returned keys
   remain in the tree) and that no changes have been made to keys owned by the
   user without the user's knowledge.

These operations are executed over an application-provided transport layer,
where the transport layer enforces access control by blocking queries which are
not allowed:

TODO diagram showing a search request over an application-provided transport
layer getting accepted / rejected

## Out-of-Band Communication

It is sometimes possible for a Transparency Log to present forked views of data
to different users. This means that, from an individual user's perspective, a
log may appear to be operating correctly in the sense that all of a user's
requests succeed and proofs verify correctly. However, the Transparency Log has
presented a view to the user that's not globally consistent with what it has
shown other users. As such, the log may be able to associate data with keys
without the key owner's awareness.

The protocol is designed such that users always require subsequent queries to
prove consistency with previous queries. As such, users always stay on a
linearizable view of the log. If a user is ever presented with a forked view,
they hold on to this forked view forever and reject the output of any subsequent
queries that are inconsistent with it.

This provides ample opportunity for users to detect when a fork has been
presented, but isn't in itself sufficient for detection. To detect forks, users
must either use **out-of-band communication** with other users or **anonymous
communication** with the Transparency Log.

With out-of-band communication, two users gossip with each other to establish
that they both have the same view of the log's data. This gossip is able to
happen over any supported out-of-band channel, even if it is heavily
bandwidth-limited, such as scanning a QR code or talking over the phone.

With anonymous communication, a single user accesses the Transparency Log over
an anonymous channel and tries to establish that the log is presenting the same
view of data over the anonymous channel as it does over authenticated channels.

In the event that a fork is successfully detected, the user is able to produce
non-repudiable proof of log misbehavior which can be published.

TODO diagram showing a user talking to a log over an authenticated & anonymous
channel, gossipping with other users

# Deployment Modes

In the interest of satisfying the widest range of use-cases possible, three
different modes for deploying a Transparency Log are supported. Each mode has
slightly different requirements and efficiency considerations for both the
service operator and the end-user.

**Third-Party Management** and **Third-Party Auditing** are two deployment modes
that require the service operator to delegate part of the operation of the
Transparency Log to a third party. Users are able to run more efficiently as
long as they can assume that the service operator and the third party won't
collude to trick them into accepting malicious results.

With both third-party modes, all requests from end-users are initially routed to
the service operator and the service operator coordinates with the third party
themself. End-users never contact the third party directly, however they will
need a signature public key from the third party to verify its assertions.

With Third-Party Management, the third party performs the majority of the work
of actually storing and operating the log, and the service operator only needs
to sign new entries as they're added. With Third-Party Auditing, the service
operator performs the majority of the work of storing and operating the log, and
obtains signatures from a lightweight third-party auditor at regular intervals
asserting that the service operator has been constructing the tree correctly.

**Contact Monitoring**, on the other hand, supports a single-party deployment
with no third party. The tradeoff is that executing the background monitoring
protocol requires an amount of work that's proportional to the number of keys a
user has looked up in the past. As such, it's less suited to use-cases where
users look up a large number of ephemeral keys, but would work ideally in a
use-case where users look up a small number of keys repeatedly (for example, the
keys of regular contacts).

| Deployment Mode        | Supports ephemeral keys? | Single party? |
|------------------------| -------------------------|---------------|
| Contact Monitoring     | No                       | Yes           |
| Third-Party Auditing   | Yes                      | No            |
| Third-Party Management | Yes                      | No            |
{: title="Comparison of deployment modes" }

## Contact Monitoring

TODO diagram showing user request going to service operator, followed by
monitoring queries later.

## Third-Party Auditing

With the Third-Party Auditing deployment mode, the service operator obtains
signatures from a lightweight third-party auditor attesting to the fact that the
service operator is constructing the tree correctly. These signatures are
provided to users along with the responses for their queries.

The third-party auditor is expected to run asynchronously, downloading and
authenticating a log's contents in the background, so as not to become a
bottleneck for the service operator.

TODO diagram showing a user request going to a service operator and a response
with an auditor signature coming back. Batched changes going to auditor in
background.

## Third-Party Management

With the Third-Party Management deployment mode, a third party is responsible
for the majority of the work of storing and operating the log, while the service
operator serves mainly to enforce access control and authenticate the addition
of new entries to the log. All user queries are initially sent by users directly
to the service operator, and the service operator proxies them to the
third-party manager if they pass access control.

TODO diagram showing user request going to service operator, immediately being
proxied to manager with operator signature.


# Security Guarantees

A user that correctly verifies a proof from the Transparency Log (and does any
required monitoring afterwards) receives a guarantee that the Transparency Log
operator executed the key-value lookup correctly, and in a way that's globally
consistent with what it has shown all other users. That is, when a user searches
for a key, they're guaranteed that the result they receive represents the same
result that any other user searching for the same key would've seen. When a user
modifies a key, they're guaranteed that other users will see the modification
the next time they search for the key.

If the Transparency Log operator does not execute a key-value lookup correctly,
then either:

1. The user will detect the error immediately and reject the proof, or
2. The user will permanently enter an invalid state.

Depending on the exact reason that the user enters an invalid state, it will
either be detected by background monitoring or the next time that out-of-band
communication is available. Importantly, this means that users must stay
online for some fixed amount of time after entering an invalid state for it to
be successfully detected.

The exact caveats of the above guarantee depend naturally on the security of
underlying cryptographic primitives, but also the deployment mode that the
Transparency Log relies on:

- Third-Party Management and Third-Party Auditing require an assumption that the
  service operator and the third-party manager/auditor do not collude
  to trick users into accepting malicious results.
- Contact Monitoring requires an assumption that the user that owns a key and
  all users that look up the key do the necessary monitoring afterwards.

## Privacy Guarantees

For applications deploying KT, service operators expect to be able to control
when sensitive information is revealed. In particular, an operator can often
only reveal that a user is a member of their service, and information about that
user's account, to that user's friends or contacts.

KT only allows users to learn whether or not a lookup key exists in the
Transparency Log if the user obtains a valid search proof for that key.
Similarly, KT only allows users to learn about the contents of a log entry if
the user obtains a valid search proof for the exact key and version stored at
that log entry.

Applications are primarily able to manage the privacy of their data in KT by
relying on these properties when they enforce access control policies on the
queries issued by users, as discussed in {{protocol-overview}}. For example if
two users aren't friends, an application can block these users from searching
for each other's lookup keys. This prevents the two users from learning about
each other's existence. If the users were previously friends but no longer are,
the application can prevent the users from searching for each other's keys and
learning the contents of any subsequent account updates.

TODO specify the rest of the privacy guarantees of the finished protocol

<!-- Operators may also wish to conceal when individual users perform a given task
like rotate their public key or add a new device to their account, or even
conceal the exact number of users their application has overall. -->

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
