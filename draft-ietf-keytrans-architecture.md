---
title: "Key Transparency Architecture"
category: info

docname: draft-ietf-keytrans-architecture-latest
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
  github: "ietf-wg-keytrans/draft-arch"
  latest: "https://ietf-wg-keytrans.github.io/draft-arch/draft-ietf-keytrans-architecture.html"

author:
 -
    fullname: Brendan McMillion
    organization: Independent
    email: brendanmcmillion@gmail.com

normative:

informative:
  sealed-sender:
    target: https://signal.org/blog/sealed-sender/
    title: "Technology preview: Sealed sender for Signal"
    author:
      - org: Signal
    date: 2018-10-29


--- abstract

This document defines the terminology and interaction patterns involved in the
deployment of Key Transparency (KT) in end-to-end encrypted communication
services, and specifies the security properties that KT provides. It also gives
more general, non-prescriptive guidance on how to securely apply KT to a number
of common applications.

--- middle

# Introduction

Before any information can be exchanged in an end-to-end encrypted
communication service, at least two things must happen: First, participants in
the system must provide the service operator (the organization that runs the
communication service) with any public keys they wish to use to receive
messages. Second, the service operator must somehow distribute these public keys
among the participants that wish to communicate with each other.

Typically, users upload their public keys to a simple directory where other
users can download them as needed, or exchange public keys in-band with the
communication being secured. If users simply trust the service operator to
forward public keys correctly, the underlying encryption protocol is only able
to protect them against passive eavesdropping {{?RFC4949}}.

However, most messaging systems are centralized, meaning that all messages are
exchanged through the service operator's servers, which makes it extremely easy
for an operator to launch an active attack. That is, the service operator can
generate a rogue public key for which it holds the corresponding private key,
and associate that rogue public key with a user's account without the user's
knowledge, allowing it to impersonate that user or eavesdrop on conversations
with them.

Key Transparency (KT) addresses this problem by requiring the service operator
to store user public keys in a cryptographically protected append-only log. Any
malicious entries added to such a log will generally be equally visible to both
the affected user and the user's contacts. This allows users to detect whether
they are being impersonated by viewing the public keys attached to their
account. If the service operator attempts to conceal some entries of the log
from some users but not others, this creates a "forked view" that is permanent
and detectable.

The term "public key" is used broadly in this document to refer to any public
cryptographic material that a user associates with their account; depending on
the application, this may be a raw public key, a certificate containing a public
key, or some other application-defined structure. KT does not depend on the
format of this data.

The critical improvement of KT over related protocols like Certificate
Transparency {{?RFC6962}} is that KT includes an efficient protocol to search
the log for entries related to a specific participant. This means users do not
need to download the entire log, which may be substantial, to find all entries
that are relevant to them. It also means that KT can better preserve user
privacy by only showing entries of the log to participants that genuinely need
to see them.


# Definitions

**End-to-End Encrypted Communication Service:**
: A communication service that allows end-users to engage in text, voice,
  video, or other forms of communication over the Internet, and uses public key
  cryptography to ensure that communications are only accessible to their
  intended recipients.

**Service Operator:**
: The primary organization that provides the infrastructure for an end-to-end
  encrypted communication service and, typically, the software used to
  participate in it. Client software may also be provided by other parties.

**End-User Device:**
: The device at the final point in a digital communication, which may either
  send or receive encrypted data in an end-to-end encrypted communication
  service.

**User / Account:**
: A single end-user of an end-to-end encrypted communication service, who may
  interact with the service on multiple end-user devices (e.g., phone, laptop).

**End-User Identity:**
: A unique and user-visible identity associated with an account (and therefore
  one or more end-user devices) in an end-to-end encrypted communication
  service. In the case where an end-user explicitly requests to communicate with
  (or is informed they are communicating with) an end-user uniquely identified
  by the name "Alice", the end-user identity is the string "Alice". A single
  account may be represented simultaneously by multiple end-user identities
  (e.g., email, phone number, username).

**Transparency Log:**
: A specialized service capable of securely attesting to the information (such
  as public keys) associated with a given end-user identity. A transparency
  log is usually run either entirely or partially by the service operator, but
  could also be operated externally.

**Transparency Log Configuration:**
: The long-term, public parameters of a transparency log, such as the log's
  cipher suite and public key. Configuration is pre-distributed to users through
  a trustworthy channel, typically as part of the communication service's client
  software, and used for proof verification.

**Third Party:**
: An organization independent of the service operator that either operates the
  transparency log (a third-party manager) or audits its operation (a
  third-party auditor). See {{deployment-modes}}.

**Label:**
: A lookup key in the key-value database that a transparency log represents,
  such as an end-user identity.

**Label Owner:**
: A user that either initiates all changes to a label's value, or must be
  informed of all changes to it.

**Label-Version Pair:**
: A label together with one of its version numbers, which identifies a single
  value of that label.

**Prefix Tree:**
: A hash tree that commits to the set of label-version pairs stored in a
  transparency log, and their values, at a given point in time.

**Log Tree:**
: A hash tree with one leaf, called a **log entry**, for each batch of changes
  to the transparency log. Each log entry commits to a snapshot of the prefix
  tree, so the log tree as a whole commits to the history of all label
  modifications.

**Distinguished Log Entry:**
: One of the log entries, chosen at a regular interval, that label owners are
  required to check for correctness. See {{distinguished-log-entries}}.

**Reasonable Monitoring Window (RMW):**
: A duration configured by the transparency log that reflects how frequently
  label owners are expected to perform monitoring.

**Tree Head:**
: The root value and size of the log tree at a given point in time, signed by
  the transparency log (and possibly by a third party).


# Protocol Overview

From a networking perspective, KT provides a client-server architecture with a
central *transparency log*, acting as a server, which holds the authoritative
copy of all information and exposes endpoints that allow users to query or
modify stored data. Users coordinate with each other through the server by
uploading their own public keys and downloading the public keys of other
users. Users are expected to maintain relatively little state, limited to what
is required to interact with the log and ensure that it is behaving honestly.

From an application perspective, KT can be thought of as a versioned key-value
database. Users insert key-value pairs into the database where, for example, the
key is their username and the value is their public key. Users can update a key
by inserting a new version with new data. They can also look up the most recent
version of a key or any previous version that has not been pruned (see
{{pruning}}). This document uses the term **label** to refer to lookup keys in
the key-value database that a transparency log represents, to avoid confusion
with cryptographic public or private keys.

Users are considered to **own** a label if they are understood to either
initiate all changes to the label's value, or be informed of all changes to
the label's value. The latter situation may occur if, for example, KT
is deployed in a way where the service operator makes automated modifications to
stored data. The owning user would then be informed, after the fact, of
modifications so that they can verify that those modifications were authorized.

KT does not require the use of a specific transport protocol. This is intended
to allow applications to layer KT on top of whatever transport protocol they
already use. In particular, this allows applications to continue relying on
their existing access control system.

With some small exceptions, applications may enforce arbitrary access control
rules on top of KT. This may include requiring a user to be logged in to make KT
requests, only allowing a user to look up the labels of their contacts, or
applying a rate limit. Most applications will likely need to, at minimum,
prevent users from modifying labels they do not own. The exact mechanism for
rejecting requests, and possibly explaining the reason for rejection, is left to
the application.

The exceptions are the requests that users need to make to verify the
transparency log's behavior. Specifically:

- Label owners need to be able to monitor their label for as long as they own
  it.
- Users that previously owned a label need to be able to complete their
  monitoring of it, as discussed in {{?PROTO=I-D.ietf-keytrans-protocol}}.
- Users that looked up a label in a Contact Monitoring deployment (or
  with the client-side Contact Monitoring mode described in {{deployment-modes}}
  enabled) need to be able to complete their monitoring of it, even if their
  access to perform other queries is revoked.
- Users need to be able to refresh their view of recent distinguished log
  entries, including over an anonymous channel if the application relies on one
  (see {{detecting-forks}}).
- Recipients of credentials may need to perform any required monitoring that was
  not provided by the sender of the credential, possibly over an anonymous
  channel (see {{credentials}}).

Refusing these requests is indistinguishable from transparency log misbehavior
(see {{transparency-log-misbehavior}}).

## User Operations

The main operations that can be executed by a user are as follows:

1. **Search:** Looks up the value of a specific label in the current state of
   the log. Users may request either a specific version of the label or the
   most recent version available. If the label-version pair exists, the
   transparency log returns the corresponding value and a proof that the search
   operation was executed correctly.
2. **Update:** Adds a new version of a label, with its associated value, to the
   log. The transparency log returns a proof that the new version was added
   correctly. Note that this means that new values are added to the log
   immediately, rather than the transparency log returning a promise of future
   inclusion such as a Signed Certificate Timestamp (SCT) as defined in
   {{Section 3.2 of ?RFC6962}}. The application is responsible for ensuring that
   only the label's owner, or another party authorized by the application, is
   able to update a label (see the discussion of access control above).
3. **Monitor:** Checks both that the log is continuing to behave honestly (all
   previously returned label-version pairs remain in the log) and that no
   changes have been made to labels owned by the user without the user's
   knowledge. While Search and Update are run by the user as necessary,
   monitoring is done in the background on a recurring basis.

{{PROTO}} additionally defines operations for initializing a user's ownership of
a label and for retrieving recent distinguished log entries.

These operations are executed over an application-provided transport layer.
The application enforces access control at the transport layer by blocking
queries that are not allowed, as shown in {{request-response}}.

~~~aasvg
Alice                                   Transparency Log
  |                                            |
  |        (Valid / Accepted Requests)         |
  |                                            |
  | Search(Alice) ---------------------------->|
  |<---------------------- SearchResponse(...) |
  |                                            |
  | Search(Bob) ------------------------------>|
  |<---------------------- SearchResponse(...) |
  |                                            |
  | Update(Alice, ...) ----------------------->|
  |<---------------------- UpdateResponse(...) |
  |                                            |
  |                                            |
  |       (Rejected / Blocked Requests)        |
  |                                            |
  | Search(Fred) ----------------------> X     |
  | Update(Bob, ...) ------------------> X     |
  |                                            |
~~~
{: #request-response title="Example Request and Response Flow" }

## Distinguished Log Entries

Certain log entries are designated as **distinguished log entries**.
Distinguished log entries are chosen at a regular interval, roughly one per
**Reasonable Monitoring Window** (RMW), which is a duration configured by the
transparency log that reflects how frequently label owners are expected to
perform monitoring. Label owners are required to check that each distinguished
log entry correctly represents their labels, which allows distinguished log
entries to act as common reference points for all users of the transparency log,
regardless of deployment mode. The RMW is part of the transparency log's
configuration, so it is known to all users in advance, and allows clients to
schedule their monitoring accordingly. Distinguished log entries and the RMW are
both specified in more detail in {{PROTO}}.

## Credentials

While users are generally understood to interact directly with the transparency
log, many end-to-end encrypted communication services require the ability to
provide *credentials* to their users. Credentials convey a binding between an
end-user identity and public keys or other information, and can be verified with
minimal network access. Traditionally, this binding is provided by a certificate
(such as an X.509 certificate) signed by a trusted authority. KT credentials are
instead constructed from the transparency log's own responses.

Credentials are especially useful in applications that support anonymous
communication. These applications hide the sender's identity from the service
operator using a protocol like the Messaging Layer Security Protocol
{{?RFC9420}} (with the encryption of handshake messages required) or Sealed
Sender {{sealed-sender}}. These protocols have the sender provide their own
credential to recipients in encrypted form, either as part of the encrypted
handshake messages that establish the group's state (with the Messaging Layer
Security Protocol) or as part of each message (with Sealed Sender).

Encrypting the sender's credential allows the sender to submit messages over an
anonymous channel by specifying only the recipient's identity. The service
operator can deliver the message to the intended recipient, who can decrypt it
and validate the credential inside to be assured of the sender's identity. Note
that the recipient does not need access to an anonymous channel to initially
verify the credential while preserving the sender's anonymity.

At a high level, a KT credential is derived from a greatest-version Search for
the sender's label. One credential is provided for each of the lookups the
recipient would otherwise do to authenticate the relationship between the
presented end-user identity and its public keys. Recipients verify these lookups
themselves, using the transparency log's configuration, without contacting the
transparency log about the sender. Each credential is verified against the
recipient's retained view of recent distinguished log entries and, in some
cases, a signature from the transparency log, so a credential that has been
modified will fail verification.

To prevent stale credentials from being accepted, recipients only accept
credentials that are based on a recent distinguished log entry, where the
application defines what is considered recent. Senders therefore need to periodically refresh their credentials, and
recipients need to periodically refresh their view of the transparency log's
recent distinguished log entries. These refreshes do not reveal anything about
the credentials that a recipient has received.

Any future monitoring that may be required should be provided to recipients
proactively by the sender, as this is the most robust way to preserve the
sender's anonymity. If required future monitoring is not provided, either
accidentally or because the system was intentionally designed that way, the
recipient may need to perform the monitoring themselves, possibly over an
anonymous channel to preserve the sender's anonymity.

{{anonymous}} shows an example message flow in an anonymous deployment. Users
request their own label from the transparency log and provide the serialized
response, functioning as a credential, in encrypted messages to other users.
Required monitoring is provided proactively in the same way. For simplicity,
the figure shows responses being forwarded directly, although {{PROTO}} defines
dedicated structures for credentials and their subsequent monitoring.

~~~aasvg
Transparency Log               Alice            Anonymous Group
|                                |                            |
|<---------------- Search(Alice) |                            |
| SearchResponse(...) ---------->| Encrypt(Anon Group,        |
|                                |      SearchResponse) ----->|
|                                |                            |
|<--------------- Monitor(Alice) |                            |
| MonitorResponse(...) --------->| Encrypt(Anon Group,        |
|                                |      MonitorResponse) ---->|
|                                |                            |
~~~
{: #anonymous title="Example Message Flow in an Anonymous Deployment" }

## Detecting Forks

It is sometimes possible for a transparency log to present forked views of data
to different users. This means that, from an individual user's perspective, a
log may appear to be operating correctly in the sense that all of a user's
requests succeed and proofs verify correctly. However, the transparency log has
presented a view to the user that is not globally consistent with what it has
shown other users. As such, the log may be able to change a label's value
without the label's owner becoming aware.

The protocol is designed such that users always require subsequent queries to
prove consistency with previous queries. As such, each user's view of the log
always forms a single, consistent history. If a user is ever presented with a
forked view, they hold on to this forked view forever and reject the output of
any subsequent queries that are inconsistent with it.

This provides ample opportunity for users to detect when a fork has been
presented, but is not in itself sufficient for detection. To detect forks, users
require a **trusted third party**, **anonymous communication** with the
transparency log, or **peer-to-peer communication**. Anonymous and peer-to-peer
communication are collectively referred to as *out-of-band (OOB)
communication*.

With a trusted third party, an outside organization either operates the
transparency log or audits its operation. A third-party auditor, as described in
{{third-party-auditing}}, ensures, among other things, that the transparency log
is growing in an append-only manner, and produces a signature on a recent tree
head. The transparency log provides this signature to users inline with their
query responses as proof that they are not being shown a fork. A third-party
manager, as described in {{third-party-management}}, operates the transparency
log itself; in this case, the service operator is responsible for detecting
whether the manager presents forked views to different users. Either way, this
approach relies on an assumption that the third party is trusted not to collude
with the service operator to trick users into accepting malicious results.

With anonymous communication, a user accesses the transparency log over an
anonymous channel and checks that the log tree root values for recent
distinguished log entries are consistent with those that the user obtained over
an authenticated channel. The security of this approach relies on the idea that,
if the transparency log does not know which user is making the request, it will
show the user the wrong fork with significant probability. Repeating this check
over time makes it overwhelmingly likely that any fork presented to any user
will be detected. To ensure this, requests over the anonymous channel must not
contain any user-specific data or state, such as cookies, authentication tokens,
or the tree size last observed by the user. They also must not be correlated in
time with the user's authenticated requests, such as by being sent immediately
before or after them.

With peer-to-peer communication, two users gossip with each other to establish
that they both have the same view of the transparency log. This gossip can
happen over any out-of-band channel that the application supports, even if it is
heavily bandwidth-constrained, such as scanning a QR code. However, this
approach is only secure if gossip can be implemented such that gossiping users
are reasonably expected to form a connected graph of all users. If not, then the
transparency log can attempt to partition users into subsets that do not gossip
and can present each subset of users with different forks.

Applications that do not provide their users with a trusted third party,
anonymous communication, or peer-to-peer communication are not secure, as the
transparency log can present forked views to different users without detection.

In the event that a fork is successfully detected, the user is generally able to
produce non-repudiable proof of log misbehavior that can be published. This
requires that the conflicting views be authenticated by the transparency log
(or third party). For example, users gossiping peer-to-peer can exchange signed
query responses rather than bare root values, at the cost of additional
bandwidth.

{{out-of-band-checking}} illustrates fork detection with anonymous and
peer-to-peer communication. Users receive tree heads while making authenticated
requests to a transparency log. Users ensure that their views of the log are
consistent by comparing log tree root values, either among themselves or by
contacting the transparency log over an anonymous channel. Requests that require
authentication (`Search(Bob)` in this example) may be blocked over the anonymous
channel. The root values that users compare out of band (`6c063bb` in this
example) correspond to distinguished log entries.

~~~aasvg
Alice                   Bob                          Transparency Log
|                        |                                          |
|                        |  (Requests over authenticated channel)   |
|                        |                                          |
|                        | Search(Bob) ---------------------------->|
|                        |<----------- Response{Head: 8f3a91d, ...} |
|                        |                                          |
|                        | DistinguishedRequest ------------------->|
|                        |<---------- Response{Heads: 6c063bb, ...} |
|                        |                                          |
|                        |                                          |
| (OOB check with peer)  |    (OOB check over anonymous channel)    |
|                        |                                          |
|<-------------- 6c063bb | DistinguishedRequest ~~~~~~~~~~~~~~~~~~> |
| 6c063bb -------------->| <~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 6c063bb |
|                        |                                          |
|                        |     (Blocked over anonymous channel)     |
|                        |                                          |
|                        | Search(Bob) ~~~~~~~~~~~~~~~~~~~> X       |
|                        |                                          |
~~~
{: #out-of-band-checking title="Out-of-Band Fork Detection" }

# Deployment Modes

In the interest of satisfying the widest range of use cases possible, three
different modes for deploying a transparency log are supported. Each mode has
slightly different requirements and efficiency considerations for both the
service operator and end-users.

**Third-Party Management** and **Third-Party Auditing** are two deployment
modes that require the service operator to delegate part of its transparency
log operations to a third party.

With both third-party modes, all requests from end-users are initially routed to
the service operator, and the operator coordinates with the third party
itself. End-users never contact the third party directly, although they will
need a signature public key from the third party to verify its assertions.

With Third-Party Management, the third party performs the majority of the work
of actually storing and operating the transparency log, and the service operator
mainly enforces access control and signs new versions of labels as they are
added. With Third-Party Auditing, the service operator performs the majority of
the work of storing and operating the transparency log, only obtaining
signatures from a third-party auditor at regular intervals asserting that the
tree has been constructed correctly.

To reduce the probability of collusion between the service operator and the
third party, a service operator can have two or more independent third parties
coordinate as one and produce threshold signatures. In this scenario, the
threshold for a valid signature needs to be at least a majority of the third
parties, assuming each third party acts consistently, to prevent different
subsets from authenticating forked views.

**Contact Monitoring**, on the other hand, supports a single-party deployment
with no third party. The cost of this is that, when a user looks up a version of
a label that was inserted very recently, the user may need to retain some
additional state and monitor the label until it is included in a distinguished
log entry. If a user looks up many
label-version pairs that were inserted very recently, monitoring may become
relatively expensive.

Additionally, applications that rely on a transparency log deployed in Contact
Monitoring mode need to regularly attempt to detect forks through anonymous
communication with the transparency log or peer-to-peer communication, as
described in {{detecting-forks}}.

Applications that rely on a transparency log deployed in either of the
third-party modes can allow users to enable a client-side "Contact Monitoring
mode". This mode, which affects only the individual client's behavior, would
cause the client to behave as if its transparency log were deployed in Contact
Monitoring mode. As such, it would start retaining state about previously
looked-up labels and regularly engaging in out-of-band communication. Enabling
this higher-security mode allows users to double-check that the third party is
not colluding with the service operator.

## Contact Monitoring

With the Contact Monitoring deployment mode, the monitoring burden is split
between the owner of a label and those that look up the label. Stated as
simply as possible, the monitoring obligations of each user are:

1. On a regular basis, the label owner verifies that the most recent version of
   their label has not changed unexpectedly.
2. When a user that looked up a label sees that the version they received was
   inserted very recently, they check back later to see that the label-version
   pair they observed was not removed before it could be detected by the label
   owner.

This ensures that if a malicious value for a label is added to the log, then
either it is detected by the label owner or, if it is removed or obscured from
the log before the label owner can detect it, any users that observed it will
detect its removal. As such, the transparency log is not trusted to behave
honestly in this mode; rather, any deviation from the protocol or the addition
of malicious values for a label is detected by users through monitoring and
out-of-band communication.

In the example shown in {{contact-monitoring-fig}}, Alice searches for Bob's
label. One day later, Alice verifies that the label-version pair from the search
remained in the transparency log. Another day later, Bob comes online and
monitors the label. Note that Alice does not need to wait for Bob's Monitor
request before making her Monitor request.

~~~aasvg
Alice                        Transparency Log                     Bob
|                                   |                               |
| Search(Bob) --------------------->|                               |
|<------------- SearchResponse(...) |                               |
|                                   |                               |
|           (1 day later)           |                               |
|                                   |                               |
| Monitor(Bob) -------------------->|                               |
|<------------ MonitorResponse(...) |                               |
|                                   |                               |
|           (1 day later)           |                               |
|                                   |                               |
|                                   |<---------------- Monitor(Bob) |
|                                   | MonitorResponse(...) -------->|
|                                   |                               |
~~~
{: #contact-monitoring-fig title="Contact Monitoring" }

Importantly, Contact Monitoring impacts how the application is able to enforce
access control on Monitor queries. While access control can be enforced on
Search queries on an arbitrary "point in time" basis, where a user is allowed
to execute the query at one point in time but possibly not the next, users are
sometimes required by the protocol to make Monitor queries based on the Search
queries they made in the past. These Monitor queries need to be permitted,
regardless of whether or not the user is still permitted to execute such Search
queries. This is in addition to the monitoring that label owners perform in
every deployment mode, as discussed in {{protocol-overview}}.

## Third-Party Management

With the Third-Party Management deployment mode, a third party is responsible
for the majority of the work of storing and operating the log. The service
operator focuses mainly on enforcing access control and authenticating the
creation of new versions of labels. All user queries are sent to the service
operator, which proxies them to the third-party manager if they pass access
control. New versions of a label, if allowed by the service operator, are signed
by the service operator before being sent to the third-party manager to be
inserted into the tree. Users verify the service operator's signature on each
version of a label before accepting it.

As part of authenticating the creation of new versions of labels, the service
operator must implement some mechanism to prevent the same version of a label
from being submitted to the third-party manager multiple times with different
associated values. Additionally, the service operator must implement some
mechanism to detect when forks are presented by the third-party manager.

As shown in {{manager-fig}}, requests that pass access control are proxied to
the third-party manager, while requests that do not are rejected by the service
operator.

~~~aasvg
Alice                  Service Operator                  Manager
|                             |                                |
| Search(Alice) ------------->| ------------------------------>|
|<--------------------------- |<---------- SearchResponse(...) |
|                             |                                |
| Search(Bob) --------------->| ------------------------------>|
|<--------------------------- |<---------- SearchResponse(...) |
|                             |                                |
| Update(Alice, ...) -------->| ------------------------------>|
|<--------------------------- |<---------- UpdateResponse(...) |
|                             |                                |
| Search(Fred) ----------> X  |                                |
| Update(Bob, ...) ------> X  |                                |
|                             |                                |
~~~
{: #manager-fig title="Third-Party Management" }

The security of the Third-Party Management deployment mode comes from an
assumption that the service operator and the third-party manager do not collude
to behave maliciously. If the third-party manager behaves honestly, then any
improper modifications to a label's value that were requested by the
service operator will be properly published such that the label owner will
detect them when monitoring. If the service operator behaves honestly, the
third-party manager will be unable to add any new unauthorized versions of a
label such that a user will accept them, or remove any authorized version of a
label without the label owner detecting it.

## Third-Party Auditing

With the Third-Party Auditing deployment mode, the service operator obtains
signatures from a third-party auditor attesting to the fact that
the tree has been constructed correctly. These signatures are provided to users
as part of the responses for their queries.

When running synchronously, the auditor can easily become a bottleneck for the
transparency log. It is generally expected that third-party auditors run
asynchronously, downloading and authenticating a log's contents in the
background. As a result, signatures from the auditor may lag behind the view
presented by the service operator. The maximum amount of time that the auditor
may lag behind the service operator without its signature being rejected by
users, called the **maximum auditor lag**, is set in the transparency log's
configuration.

As shown in {{auditing-fig}}, the auditor is updated on changes to the tree in
the background, and a recent signature from the auditor is provided to users.

~~~aasvg
Many Users                        Service Operator            Auditor
|                                        |                          |
| Update(Alice, ...) ------------------->|                          |
| Update(Bob, ...) --------------------->|                          |
| Update(Carol, ...) ------------------->|                          |
| <===== Response{AuditorSig: 66bf, ...} |                          |
|                                        |                          |
|                                        |                          |
|                                        | AuditorUpdate ---------->|
|                                        |<-------- AuditorTreeHead |
|                                        |                          |
~~~
{: #auditing-fig title="Third-Party Auditing" }

Given the long-lived nature of transparency logs and the potentially short-lived
nature of third-party auditing arrangements, KT allows third-party auditors to
start auditing a log at an arbitrary point. This allows a new third-party
auditor to start up without ingesting the transparency log's entire
history. The point at which an auditor started auditing is provided to users in
the transparency log's configuration. When verifying query responses, users
verify that the auditor started auditing at or before the point necessary for
the query to be secure.


# Combining Logs

There are many cases where it makes sense to operate multiple cooperating
transparency log instances, for example:

- A service operator may wish to gradually migrate to a transparency log that
  uses different cryptographic keys, a different cipher suite, or a
  different deployment mode.
- A service operator may need to immediately migrate to a new transparency log
  to recover from log failure, such as the compromise of the log's private keys,
  the loss of its data, or the accidental creation of a fork.
- A service operator may operate multiple logs to handle the total load of a
  high-traffic application.
- A federated system may allow each participant in the federation to operate
  their own transparency log for their own users.

Client implementations should generally be prepared to interact with multiple
independent transparency logs to support these use cases, particularly recovery
from log failure. When multiple transparency logs are used as part of one
application, all users need to have a consistent policy for executing queries
against the logs in a way that maintains the high-level security model of KT:

- If all transparency logs behave honestly, then users observe a globally
  consistent view of the data associated with each label.
- If any transparency log behaves dishonestly, this will be detected in a timely
  manner by background monitoring or fork detection.

## Gradual Migration

In the case of gradually migrating from an old transparency log to a new one,
this policy may look like:

1. Search queries are executed against the old transparency log first, and then
   against the new transparency log only if the most recent version of a label
   in the old transparency log is a special application-defined "tombstone"
   entry. After the migration period is complete, Search queries are executed
   only against the new transparency log.
2. Update queries are executed only against the new transparency log, with the
   exception of adding a tombstone entry for the label to the old transparency
   log if one has not been added already and the migration period is still
   active. This includes labels that are created for the first time during the
   migration period, which receive a tombstone entry as their first version in
   the old transparency log.
3. Both transparency logs are monitored as they would be if they were run
   individually. Once the migration period is complete and the old transparency
   log has stopped accepting modifications, it needs to stay operational long
   enough for all users to complete their monitoring of it, keeping in mind that
   some users may be offline for a significant amount of time. If it is shut
   down before a user is able to complete their monitoring of it, that user will
   be unable to detect some forms of misbehavior by the old transparency log.
   Note that the old transparency log continues to add log entries so that its
   tree heads remain fresh.

Placing a tombstone entry for each migrated label in the old transparency log
gives users a clear indication as to which transparency log contains the most
recent version of a label. Importantly, it prevents users from incorrectly
accepting a stale version of a label.

Labels that are not updated before the old transparency log stops accepting
modifications do not receive a tombstone entry and are not carried over to the
new transparency log. These labels are effectively deleted once the migration
period is complete, as Search queries are no longer executed against the old
transparency log from that point on.

## Immediate Migration

In some situations, the service operator may instead choose to stop adding new
entries to a transparency log immediately and provide a new transparency log
that is pre-populated with the most recent versions of all labels. In this case,
the old transparency log stops accepting modifications but remains online for a
migration period that is long enough for all users to complete their monitoring
of it, keeping in mind that some users may be offline for a significant amount
of time. The policy users follow may look like:

1. Search queries are executed against the new transparency log.
2. Update queries are executed against the new transparency log.
3. The final tree head of the old transparency log is provided to
   users over a trustworthy channel. Users issue their final Monitor queries to
   complete monitoring up to this point. Label owners initialize monitoring
   state for the new transparency log by processing an Update for the migrated
   versions of their labels and verifying that the migration was done correctly.
   From then on, users will monitor only the new transparency log.

The final tree head of the old transparency log needs to be distributed to users
in a way that ensures that all users have a globally consistent view. This can
be done by storing it in a well-known label of the new transparency log. In this
case, users need to process this well-known label as if they own it, so that
they continue to monitor it for unexpected changes for the duration of the
migration period. Alternatively, the final tree head may be distributed with the
application's code distribution mechanism. Regardless, if a migration is the
result of one or more private key compromises, the authenticity of the final
tree head must not depend on a compromised private key.

If a migration is the result of data loss or an accidental fork, the final tree
head is the furthest-along tree head that the service operator can still
provide: the furthest that was not lost, or the furthest that did not fork.
Users that previously observed a tree head beyond this point, or on the
abandoned fork, will be unable to complete their monitoring of the old
transparency log. To these users, the old transparency log's behavior is
indistinguishable from misbehavior, and it should be reported to them as such
(see {{transparency-log-misbehavior}}).

## Partitioning

Some applications partition labels across multiple transparency logs, where each
transparency log is responsible for a disjoint subset of labels. In such
applications, there needs to be a consistent policy for directing KT requests
to the correct transparency log. The two primary cases for partitioning are
high-traffic applications that require horizontal scaling, and federated
applications.

In a high-traffic application, many servers that are owned and operated by a
single entity may need to cooperate to handle the total load of the application.
Where possible, this is best done within a single transparency log, since
updates can be batched into one log entry and the prefix tree can be divided
among servers, without any change that is visible to users. However, if multiple
transparency logs are indeed necessary for a single application, the application
can provide a fixed set of transparency logs and trivially route requests
between them based on the hash of the label.

Increasing the number of transparency logs is possible by following either of
the processes described in {{gradual-migration}} or {{immediate-migration}},
replacing one transparency log in the set with two or more. That is, labels that
were previously assigned to a single transparency log can be migrated and split
between two or more new transparency logs, with the previous transparency log
eventually shut down. When the application distributes the configurations for
the new transparency logs, it also distributes the updated partitioning rule,
allowing users to correctly identify the transparency log responsible for a
label.

In a federated application, many servers that are owned and operated by
different entities cooperate to provide a single end-to-end encrypted
communication service. Each entity provides its own infrastructure (in
particular, a transparency log) to serve the users that rely on it. Typically,
the end-user identity directly specifies which entity requests should be
directed to. For example, with an email end-user identity like
`alice@example.com`, the controlling entity is `example.com`, indicating that
requests related to this user should be directed to the servers of
`example.com`.

In a federated application, a controlling entity like `example.com` can act as
an anonymizing proxy for its users when they query transparency logs run by
other entities (in the manner of {{?RFC9458}}), but should typically not attempt
to "mirror" or combine other transparency logs with its own. Doing so may
prevent the other transparency logs from consistently enforcing their access
control policies as intended, managing compliance with privacy laws, or
modifying label values quickly and without interference from third-party caches.
An entity that mirrors or combines another transparency log needs to ensure that
these problems are either resolved, for example by coordinating with the other
transparency log's operator, or acceptable for the application.


# Pruning

As part of the core infrastructure of an end-to-end encrypted communication
service, transparency logs are expected to operate for many years. This presents
a problem for general append-only logs, as even moderate usage can cause the log
to grow to an unmanageable size in that time frame. This issue is
further compounded by the fact that a substantial portion of the entries added
to a log may be fake, having been added solely for the purpose of obscuring
short-term update rates (discussed in {{population-level-metrics}}). Given this,
transparency logs need to be able to manage their footprint by pruning data
that is no longer required by the communication service.

Broadly speaking, a transparency log's database will contain two types of data:

1. Serialized user data (the values corresponding to labels in the log, along
   with the openings of the cryptographic commitments that the log stores in
   place of those values), and
2. Cryptographic data, such as pre-computed portions of hash trees.

The first type, serialized user data, can be pruned by removing data that has
either expired or become permanently inaccessible due to the service
operator's access control policy.

A version of a label expires when it is no longer possible to produce a valid
search proof for the label-version pair, which happens when all of the log
entries necessary to produce the proof have expired. A log entry is considered
expired once it is older than the log's configured **maximum lifetime**, if any.
This is specified in more detail in {{PROTO}}.

Notably, the most recent version of a label is the only version that never
expires through the maximum lifetime mechanism. However, service operators can
define access control policies that permanently block access to any version of a
label, including the most recent one, subject to the constraints discussed in
{{protocol-overview}}.

The second type of data, cryptographic data, can also be pruned, but only after
considering which parts are no longer required by the protocol for producing
proofs. For example, even though a particular version of a label may have been
deleted, data corresponding to it may still need to exist in the most recent
version of the transparency log's prefix tree to produce valid search proofs for
other versions of the label. {{PROTO}} specifies which log entries and versions
of the prefix tree a transparency log is required to retain, but the exact
mechanism for determining which data is safe to delete depends on the
implementation.

The distinction between user data and cryptographic data provides a valuable
separation of concerns since KT does not provide a way for a service operator to
convey its access control policy to a transparency log. This distinction allows
the pruning of user data to be done entirely by application-defined code, while
the pruning of cryptographic data can be done entirely by KT-specific code as a
subsequent operation.


# Security Considerations

A user that correctly verifies a proof from the transparency log and does any
required monitoring afterwards receives a guarantee that the transparency log
executed the operation correctly, and in a way that is globally consistent with
what it has shown all other users. That is, when a user searches for a label,
they are guaranteed that the result they receive represents the same result that
any other user searching for the same label at roughly the same time would have
seen. When a user modifies a label, they are guaranteed that other users will
see the modification within a bounded amount of time.

Note that KT does not itself determine whether a new version of a label was
authorized by the label's owner. Instead, it guarantees that any version shown
to other users is also visible to the label's owner, who decides whether the
change is authorized when monitoring their label. As a result, detecting
unauthorized changes depends on the label owner performing regular monitoring.
If the owner is offline for an extended period, for example because their
account is dormant or abandoned, or if they lose their local state (see
{{state-loss}}), an unauthorized change may not be detected until the owner
returns, or may not be detected at all. In particular, if the owner is offline
for longer than the maximum lifetime of log entries, an unauthorized version
that was inserted and then replaced while the owner was offline may expire
before the owner is able to see it.

## Transparency Log Misbehavior

If the transparency log attempts to execute a query incorrectly, then one of the
following occurs:

1. The user detects the misbehavior immediately and rejects the proof.
2. The user permanently enters an invalid state. That is, the user accepts a
   result that is inconsistent with what the transparency log has shown other
   users, such as a forked view of the log or a version of a label that the
   transparency log later attempts to remove. Since users require all
   subsequent queries to prove consistency with previous ones, the transparency
   log is unable to reconcile the user's view with that of other users. This is
   intentional, as it ensures the misbehavior is eventually detected, either by
   background monitoring or fork detection.
   Importantly, this means that users must stay online for a bounded amount of
   time after entering an invalid state for it to be successfully detected.

Instead of executing a query incorrectly, the transparency log can attempt to
prevent the user from learning about more recent states of the log. This would
allow the log to continue executing queries correctly, but on stale versions of
data. To prevent this, the transparency log's configuration contains an upper
bound on how stale a query response can be without being rejected.

The security of a transparency log depends naturally on the security of the
cryptographic primitives that it is configured to use, as well as the deployment
mode that the transparency log relies on:

- Third-Party Management and Third-Party Auditing require an assumption that the
  service operator and the third-party manager or auditor do not collude to
  trick users into accepting malicious results.
- Contact Monitoring requires an assumption that all users that look up a label
  complete any necessary monitoring afterwards, and that users regularly engage
  in out-of-band communication to detect forks.

In short, assuming that the underlying cryptographic primitives used are secure,
that any deployment-specific assumptions hold (such as non-collusion, or the
completion of monitoring and out-of-band communication), that clients do not
lose their local state and satisfy the assumptions
described in {{client-integrity}}, and that a label owner's devices do not go
offline for longer than the maximum lifetime of log entries (if one is
configured), users are able to detect any attempt by the transparency log to
present inconsistent views of a label to different users. When fork detection
relies on anonymous communication, each check only has some probability of
exposing a fork, so detection is probabilistic rather than guaranteed within a
fixed amount of time; repeated checks make it increasingly likely that a fork
is detected. The parameters that affect how long it takes for such misbehavior
to be detected are as follows:

- The configured maximum amount of time by which a query response can be stale.
- The configured Reasonable Monitoring Window, weighed against how frequently
  users execute background monitoring in practice.
- For logs that use the Contact Monitoring deployment mode: how frequently users
  engage in out-of-band communication.
- For logs that use the Third-Party Auditing deployment mode: the configured
  maximum auditor lag.
- For logs that use the Third-Party Management deployment mode: the amount of
  lag, or potential inefficacy, of the service operator's approach to detecting
  forks.

Ultimately, a transparency log can always refuse to answer requests that would
reveal its own misbehavior. This includes refusing direct requests, requests
made over an anonymous channel, and requests for the specific tree heads that
users need to efficiently gossip peer-to-peer. Similarly, a transparency log can
refuse to apply updates to a user's own label, such as replacing a compromised
public key. A user's persistent inability to perform monitoring, engage in
out-of-band communication, or update their own label is indistinguishable from
transparency log misbehavior and should be reported to the user as such.

## State Loss

The security of KT often depends on the ability of users to maintain robust
local state. Users that lose their state in a Contact Monitoring or Third-Party
Auditing deployment will have a reduced ability to detect if they were shown a
fork, or if the transparency log later obscured data that they consumed. State
loss can occur, for example, when a user's device is lost, replaced, or reset,
when the application is reinstalled or its data is cleared, or when the user
begins using a new device without transferring state from an old one.

In a Contact Monitoring deployment, state loss reduces a user's ability to
detect misbehavior if the state loss occurs after the user consumes a version of
a label that was created either too recently to be included in a distinguished
log entry, or within a portion of the log that was insufficiently gossiped. In a
Third-Party Auditing deployment, the same is true for a version of a label that
was created within the maximum auditor lag.

Applications should consider the nature of possible state loss in their clients
when configuring a transparency log and ensure that the relevant
protocol parameters are set in a way that appropriately mitigates this risk.
For example, in a Contact Monitoring deployment, the Reasonable Monitoring
Window and the duration between out-of-band communication attempts should be
much less than the typical time between state loss events. Similarly, in a
Third-Party Auditing deployment, the maximum auditor lag should be much less
than the typical time between state loss events. This minimizes the likelihood
that a state loss event could be useful to a misbehaving transparency log.

In applications where client state is typically ephemeral (like a web page), or
where state loss could possibly be triggered adversarially (for example, by the
service operator forcing a user to log out and re-register), a Third-Party
Management deployment is recommended. Alternatively, applications could
consider implementing a policy of not consuming label-version pairs that
were inserted too recently. Once a label-version pair is included in a
distinguished log entry that has been sufficiently gossiped in a Contact
Monitoring deployment, or is older than the maximum auditor lag in a
Third-Party Auditing deployment, the risks associated with state loss are often
already sufficiently mitigated.

The considerations above apply to users consuming label-version pairs. Label
ownership is inherently stateful, regardless of deployment mode. A label owner
that loses their state can no longer act as the label's owner until they
reinitialize their state from scratch, as described in {{PROTO}}, and any
unauthorized changes made before then may go undetected.

## Client Integrity

The security of KT depends on several assumptions about the clients that users
run:

1. Clients are able to execute code securely. That is, the device and operating
   system that a client runs on do not allow an attacker to tamper with the
   client's execution or local state.
2. Clients are executing a secure implementation of the protocol.
3. Clients have an authentic copy of the transparency log's configuration.
4. Clients have sufficiently accurate local clocks.

The first assumption is common to all end-to-end encrypted systems, as a
compromised device can generally access a user's private keys and plaintext
directly. To mitigate the risk of compromised devices, applications can rely on
platform security features such as verified boot, application sandboxing, and
hardware-backed key storage. Since KT requires clients to retain local state,
applications should also consider protecting this state against modification and
against being rolled back to a previous version.

The second and third assumptions are particularly important to address, since
client software and the transparency log's configuration are usually distributed
by the service operator, which is the same party that KT is intended to protect
against. A service operator that is able to distribute modified client software
or configuration to a user can defeat KT entirely for that user. For example, it
could disable proof verification or monitoring, or replace the public key of the
transparency log or a third party with one that it controls.

To address this, applications can rely on a mechanism such as Binary
Transparency, which may be built on a general-purpose transparency service like
the one described in {{?RFC9943}}. Binary Transparency systems require each
released version of a piece of software to be published in a public, append-only
log before clients will install it. This ensures that a modified version of the
software cannot be distributed to specific users without also being visible to
everyone else. Making client software open source and reproducibly built
additionally allows outside parties to inspect each published version and verify
that it implements the protocol correctly and contains the correct
configuration.

Finally, a client with an inaccurate clock may accept a stale view of the log
that would otherwise be rejected, or may reject a view that is not stale.
Applications can ensure that clients have a sufficiently accurate clock with a
protocol like Roughtime {{?RFC10049}}, which provides authenticated time from
multiple independent servers and allows clients to detect servers that provide
incorrect time.


# Privacy Considerations

## Access Control

Service operators deploying KT expect to be able to control when sensitive
information is revealed. In particular, a service operator may
need to restrict who is able to learn that a user is a member of its service,
or information about that user's account.

KT only allows a user to learn whether or not a label exists in the transparency
log if the user obtains a valid proof for a query on that label. Similarly, KT
only allows a user to learn about the value of a label if the user obtains a
valid search proof, update proof, or credential for that exact version of the
label. These can additionally reveal information about a label's history, such
as how many versions of the label exist and roughly when it last changed.

When an application's access control policy previously permitted a user to look
up or change a label's value but no longer does, KT prevents the user from
learning whether or not the label's value has changed since the user's access
was revoked. This is true even though users are still permitted to complete
their monitoring of the label after their access to perform other queries has
been revoked.

Applications determine the privacy of data in KT by relying on these properties
when they enforce access control policies on the queries issued by users. For
example, an application's access control
policy may only allow users to search for the labels of their contacts, or of
users they are actively starting a conversation with. This prevents users that
are not contacts or actively communicating from learning about each other's
existence. If two users were previously contacts but no longer are, the
application can prevent the users from searching for each other's labels and
learning whether there have been any subsequent account updates.

## Population-Level Metrics

Service operators also expect to be able to control sensitive population-level
metrics about their users. These metrics include the size of their user base,
the frequency with which new users join, and the frequency with which existing
users update their labels.

KT allows a service operator to obscure the size of its user base by batch
inserting a large number of fake label-version pairs when a transparency log is
first initialized. Similarly, KT allows a service operator to obscure the
rate at which "real" changes are made to the transparency log by padding "real"
changes with the insertion of fake label-version pairs, creating the outside
appearance of a constant baseline rate of insertions.

## Leakage to Third Parties

In the event that a third-party manager or auditor is used, there is additional
information leaked to the third party that is not visible to outsiders.

In the case of a third-party auditor, the auditor is able to learn the total
number of distinct changes to the log. It is also able to learn the log entry in
which each change was made and, from this, the order and approximate timing of
changes across log entries. The auditor also learns when entries are removed
from the prefix tree, such as in response to an erasure request (see
{{right-to-erasure}}), and which earlier changes each removal corresponds to.
Entries that are removed together may be inferred to belong to the same label.

However, the auditor is not able to distinguish whether a change represents a
label being created for the first time or being updated, or whether a change
represents a "real" change from an end-user or a "fake" padding change.
Additionally, the auditor is not able to learn the plaintext of any labels or
values. This is because labels are masked with a Verifiable Random Function
(VRF) {{?RFC9381}}, and values are only provided to the auditor as cryptographic
commitments.

A VRF uses a private key to deterministically map each label-version pair to a
fixed-length pseudorandom output. Without the private key, a label-version
pair's VRF output cannot be computed and is indistinguishable from random.
However, the transparency log can provide users with a proof that an output was
computed correctly from a given label-version pair, which users verify with the
corresponding public key.

In the case of a third-party manager, the manager generally learns everything
that the service operator would know. This includes the total set of plaintext
labels and values and their modification history. It also includes traffic
patterns, such as how often a specific label is looked up.

## Right to Erasure

Consumer privacy laws often provide a *right to erasure*. This means that when a
consumer requests that a service operator delete their personal information, the
service operator is generally legally obligated to do so. This may seem to be
incompatible with the description of KT in {{introduction}} as an "append-only
log". Once an entry is added to a transparency log, it indeed cannot be altered
or removed from the history that the log commits to. The important caveat here
is that user data is not directly stored in the append-only log. Instead, the
log primarily contains privacy-preserving VRF outputs and cryptographic
commitments.

The value corresponding to a label is typically some serialized user account
data, like a public key or internal identifier. KT uses cryptographic
commitments to ensure that users interacting with a transparency log are unable
to learn anything about a label's value until the transparency log explicitly
provides the commitment's opening. A transparency log responding to an erasure
request can delete the commitment opening and the associated data. This can be
done immediately, and doing so permanently prevents recovery of the associated
value from the transparency log's data. Copies of the value that were already
provided to users are not affected, and a label owner that has not yet processed
the affected version will be unable to verify it.

Labels themselves are typically serialized end-user identities, like a username
or email address. Each label-version pair is processed through a VRF, as
described in {{leakage-to-third-parties}}, before being stored in the
transparency log. The set of all label-version pairs stored in a transparency
log is committed to by a prefix tree, and each version of the prefix tree is
committed to by the log tree.

The VRF outputs corresponding to all versions of the labels affected by an
erasure request can be deleted from the most recent version of the prefix tree
once they have been included in at least one distinguished log entry, which
happens within roughly one Reasonable Monitoring Window of their insertion.
However, previous versions of the prefix tree will still contain the VRF outputs
and may still be needed by the protocol. If the transparency log is configured
with a maximum lifetime for log entries, the VRF outputs can
be fully purged from the transparency log once all log entries that contain them
have passed their maximum lifetime. Otherwise, the VRF outputs can never be
fully purged.


# Implementation Guidance

Fundamentally, KT can be thought of as guaranteeing that all the users of a
service agree on the contents of a key-value database (noting that this document
refers to these keys as "labels"). Special care is needed to turn the guarantee
that all users agree on a set of labels and values into a guarantee that the
mapping between end-users and their public keys is authentic. Critically, an
end-user identity must be both *unique* and *user-visible* for KT to
authenticate it. However, what exactly constitutes a unique and user-visible
identity varies greatly from application to application.

Consider, for example, a communication service where users are uniquely
identified by a fixed username, but KT has been deployed using each user's
internal ID as their label. While the ID might be unique, it is not necessarily
user-visible if, when a user attempts to look up a contact by username, the
service operator translates the username into a user ID internally. If this
mapping (from username to ID) is unauthenticated, the service operator can
manipulate it to eavesdrop on conversations by returning the ID for an account
that it controls. From a security perspective, this would be equivalent to not
using KT at all.

However, that is not to say that the use of internal IDs in KT is never
appropriate. Many applications do not have a concept of a fixed, explicit
identifier, like a username, and instead rely on their user interface (UI),
underpinned internally by a user's ID, to indicate to users whether a
conversation is with a new person or someone they have previously contacted. The
fact that the UI behaves this way makes the user's ID a user-visible identity
even if a user may not be able to actually see it written out.

A **primary end-user identity** is one that is unique, user-visible, and unable
to change. (Or equivalently, if it changes, it appears in the application UI as
a new conversation with a new user.) An example of this type of identity would
be an email address on an email service. A primary end-user identity should
always be a label in KT, with the end-user's public keys and other account
information as the associated value.

A **secondary end-user identity** is one that is unique and user-visible, and
that can change without being interpreted as a different account because it is
associated with a primary end-user identity. Secondary end-user identities are
used solely for initial user discovery, during which they are converted into a
primary end-user identity that the application uses to identify the end-user
from then on. For example, an application may allow users to change their
username without disrupting existing conversations because each username is
associated with an internal user ID. The username is a secondary end-user
identity, and the internal user ID is the primary end-user identity. The
internal user ID is user-visible by virtue of the fact that existing
conversations with the user continue uninterrupted when their username changes.
A secondary end-user identity should be a label in KT with the primary end-user
identity as its associated value, so that the user discovery process is
authenticated.

While likely helpful to most common applications, the distinction between
primary and secondary end-user identities is not a hard-and-fast rule.
Applications must be careful to ensure they fully capture the semantics of
identity in their service with the labels and values they store in KT.

Relatedly, while KT authenticates the mapping from a label to its value, it does
not help users distinguish between labels that appear similar. For example, two
usernames that differ only in visually similar characters are distinct labels to
KT, but may not be distinct to a user. Applications should take care to prevent
confusable identifiers from being registered.


# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The author would like to thank Deb Cooley, Felix Linker, Rohan Mahy, Thibault
Meunier, Cory Francis Myers, and Orie Steele for their help reviewing and
finalizing this document.
