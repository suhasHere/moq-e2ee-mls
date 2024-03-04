---
title: "Secure Group Key Agreement with MLS over MoQ"
abbrev: "moq-mls"
docname: draft-jennings-moq-e2ee-mls-latest
date: {DATE}
category: info

ipr: trust200902
area: Applications and Real-Time
submissionType: IETF
workgroup: MOQ
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs, docmapping]

keyword: Internet-Draft
v: 3
venue:
  group: "Media over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "suhasHere/moq-e2ee-mls"
  latest: "https://suhashere.github.io/moq-e2ee-mls"

author:
 -
    ins: C. Jennings
    name: Cullen Jennings
    organization: Cisco
    email: fluffy@cisco.com
 -
    ins: R.L. Barnes
    name: Richard L. Barnes
    organization: Cisco
    email: rlb@ipv.sx
 -
    ins: S. Nandakumar
    name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com

normative:
  MoQTransport: I-D.ietf-moq-transport
  SecureObjects:
    title: "Secure Objects for Media over QUIC"
    target: https://suhashere.github.io/moq-secure-objects/#go.draft-jennings-moq-secure-objects.html

informative:


--- abstract

This specification defines a mechanism to use Message Layer Security (MLS)
to provide end-to-end group key agreement for Media over QUIC (MOQ) applications.
Almost all communications are done via the MOQ transport.  MLS requires a
small degree of synchronization, which is provided by a simple counter service.

--- middle

# Introduction

Media Over QUIC Transport (MOQT) is a protocol that is optimized for the
QUIC protocol, either directly or via WebTransport, for the
dissemination of delivery of low latency media. MOQT defines a
publish/subscribe media delivery layer across set of participating
relays for supporting wide range of use-cases with different resiliency
and latency (live, interactive) needs without compromising the
scalability and cost effectiveness associated with content delivery
networks. It supports sending media objects through sets of relays
nodes.

MLS is a key establishment protocol that provides efficient asynchronous
group key establishment with forward secrecy (FS) and post-compromise
security (PCS) for groups in size ranging from two to thousands.

This document defines procedures for MOQ endpoints to engage in
secure E2EE key establishment protocol using MLS over MOQT.

More specifically, this document provides

- Design for using MOQT data model to carrying out MLS protocol exchange
- Simple counter service interface enabling synchronization of MLS protocol
  messages.
- Procedures to derive keys for MOQT object protection when using {{SecureObjects}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The "|" operator is is used to indicate concatenation of two strings or
bytes arrays.

# MLS Overview

MLS protocol provides continuous group authenticated key exchange.  MLS
provides several important security properties

* Group Key Exchange: All members of the group at a given time know a
  secret key that is inaccessible to parties outside the group.

* Authentication of group members: Each member of the group can
  authenticate the other members of the group.

* Group Agreement: The members of the group all agree on the identities
  of the participants in the group.

* Forward Secrecy: There are protocol events such that if a member's
  state is compromised after the event, group secrets created before the
  event are safe.

* Post-compromise Security: There are protocol events such that if a
  member's state is compromised before the event, the group secrets
  created after the event are safe.

At a very high level, MLS protocol operates by participants sending
proposals to add/remove/update the group state and an active member of
the group commit the proposals to move the group’s cryptographic state
from one epoch to the next (see section 3.2 of {{!RFC9420}}).

In order to setup end to end encryption of media delivered over MOQT
delivery network, producers and consumers participate in the MLS
exchange to setup group secret through which are used to derive the
keys needed for encrypting the media/data published by the members of
the MLS group.


## Critical Invariants {#invariants}

MLS requires a linear sequence of MLS Commits in that each MLS Commit
has exactly one successor. This is achieved by using a centralized
server that hands out a token to the client that is allowed to make
the next commit.


# MOQ Overview {#moqt-model}

MOQT {{MoQTransport}} defines a publish/subscribe based media delivery protocol, where in
endpoints, called producers, publish objects which are delivered via
participating relays to receiving endpoints, called consumers.

Section 2 of MoQ Transport defines hierarchical object model for
application data, comprised of objects, groups and tracks.

Objects defines the basic data element, an addressable unit whose
payload is sequence of bytes. All objects belong to a group, indicating
ordering and potential dependencies. A track contains a sequence of
groups and serves as the entity against which a consumer issues a
subscription request.

~~~~~
  Media Over QUIC Application


          |                                                       time
          |
 TrackA   +-+---------+-----+---------+--------------+---------+---->
          | | Group1  |     | Group2  |  . . . . . . | GroupN  |
          | +----+----+     +----+----+              +---------+
          |      |               |
          |      |               |
          | +----+----+     +----+----+
          | | Object0 |     | Object0 |
          | +---------+     +---------+
          | | Object1 |     | Object1 |
          | +---------+     +---------+
          | | Object2 |     | Object2 |
          | +---------+     +---------+
          |      .
          |      .
          |      .
          | +---------+
          | | ObjectN |
          | +---------+
          |
          |                                                       time
          |
 TrackB   +-+---------+-----+---------+--------------+---------+---->
          | | Group1  |     | Group2  | . . .. .. .. | GroupN  |
          | +---+-----+     +----+----+              +----+----+
          |     |                |                        |
          |     |                |                        |
          |+----+----+      +----+----+              +----+----+
          || Object0 |      | Object0 |              | Object0 |
          |+---------+      +---------+              +---------+
          |
          v


~~~~~

Objects are comprised of two parts: envelope and a payload. The envelope
is never end to end encrypted and is always visible to relays. The
payload portion may be end to end encrypted, in which case it is only
visible to the producer and consumer. The application is solely
responsible for the content of the object payload.

Tracks are identified by a combination of its `TrackNamespace ` and
`TrackName`. TrackNamespace and TrackName are treated as a sequence of
binary bytes. Group and Objects are represented as variable length
integers called GroupId and ObjectId respectively.

## Simple Callflow

Below is a simple callflow that shows the message exchange between,
Alice (the producer) , Bob (the consumer) and Relay. The MOQT
protocol exchange starts with Alice sending MOQT Announce message with
TrackNamespace under which she is going to publish media tracks.
Then Bob issues a MOQT Subscribe message to the relay for a FullTrackName
(identified  by its TrackNamespace and TrackName) expressing his interest to
receive media. Relay makes downstream subscription to Alice since the
track namespace in the subscription matches the track namespace in the announcement
from Alice. This is followed by Alice publishing media over the requested track,
which is eventually forwarded to Bob via the Relay.

~~~~
 ┌──────────┐                    ┌─────┐                   ┌────────┐
 │Alice(Pub)│                    │Relay│                   │Bob(Sub)│
 └────┬─────┘                    └──┬──┘                   └───┬────┘
      │                             │                          │
      │Announce(id=1,TrackNamespace)│                          │
      │────────────────────────────>│                          │
      │                             │                          │
      │      AnnounceOk(id=1)       │                          │
      │<────────────────────────────│                          │
      │                             │                          │
      │                             │Subscribe(id=1, TrackName)│
      │                             │<─────────────────────────│
      │                             │                          │
      │                             │    SubscribeOk(id=1)     │
      │                             │─────────────────────────>│
      │                             │                          │
      │ Subscribe(id=2, TrackName)  │                          │
      │<────────────────────────────│                          │
      │                             │                          │
      │      SubscribeOk(id=2)      │                          │
      │────────────────────────────>│                          │
      │                             │                          │
      │        Object Flow          │                          │
      │────────────────────────────>│                          │
      │                             │                          │
      │                             │       Object Flow        │
      │                             │─────────────────────────>│
      │                             │                          │
      │                             │    Unsubscribe(id=1)     │
      │                             │<─────────────────────────│
      │                             │                          │
      │      Unsubscribe(id=1)      │                          │
      │<────────────────────────────│                          │
 ┌────┴─────┐                    ┌──┴──┐                   ┌───┴────┐
 │Alice(Pub)│                    │Relay│                   │Bob(Sub)│
 └──────────┘                    └─────┘                   └────────┘

~~~~

## TrackNamespace Subscription

In order to realize the MLS key exchange over MOQ, this specification
proposes MOQT endpoints and Relays to be able to subscribe to TrackNamespace.
A sketch of the proposal is here but this would be moved out of this draft.

Following additions is proposed to the core MOQT protocol.

### SUBSCRIBE_NAMESPACE {#message-subscribe-namespace}

A subscriber sends `SUBSCRIBE_NAMESPACE` to express its interest in all
the tracks that will eventually be produced under the requested namespace.

~~~
SUBSCRIBE_NAMESPACE
{
  Track Namespace (b),
  Subscribe Namespace ID (i)
}
~~~


### SUBSCRIBE_NAMESPACE_RESPONSE {#message-subscribe-namespace-response}

Publishers sends `SUBSCRIBE_NAMESPACE_RESPONSE` indicating the status
of the request for subscribing to the track namespace.

~~~
SUBSCRIBE_NAMESPACE_RESPONSE
{
  Subscribe Namespace ID (i),
  Status Code (i),
  [Reason Phrase (b)]
}
~~~


### NAMESPACE_INFO {#message-namespace-info}

Publisher sends NAMESPACE_INFO message whenever it is ready to publish
on new track under a track namespace {{message-subscribe-namespace}}
message. The `NAMESPACE_INFO` message is a implicit subscription to the
track, unless it is explicitly unsubscribed by the subscriber by
sending `UNSUBSCRIBE` message. This message provides necessary mapping
between `Track Alias`, `Subscribe Id` to the namespace requested in
the `SUBSCRIBE NAMESPACE` message.

~~~
NAMESPACE_INFO
{
  Track Alias (i),
  Subscribe ID (i),
  Mapped Track Namespace (b),
  Mapped Track Name (b),
  Mapped Request Id (i)
}
~~~


{{bootstrapping}} provides one of the applications of namespace subscription
for MLS KeyPackage distribution.

# MLS and MOQ

This specification defines procedures for participants engaging in MLS
key exchange to happen over MOQT protocol, thus enabling following 2 goals:

1. Use MOQT as delivery transport for MLS protocol messages.

2. Allow MOQT endpoints (producers/consumers) to use MLS as secure
key exchange protocol for end to end secure communications across
range of use-cases.


## High-level Design {#mls-hld}

MLS {{!RFC9420}} achieves group key agreement by participants/members
engaging in MLS protocol message exchange that allows:

- New members to express their interest to join a  MLS group

- Existing members to commit a new members to a MLS group

- Existing members to commit removal of existing members from a MLS group

The central unit of functionality in MLS is a group, where at any given time,
a group represents a secret known only to its members. Membership to the group
can change over time. Each time membership changes (batch of joins or
leaves), the shared secret is changed to one known only by the current
members. Each period of time with stable membership/secret is an epoch.

At a high level, one can envision MLS protocol operation in the form
multiple queue abstractions to achieve the above functionality.

### KeyPackage Distribution

All participants interested in joining a MLS group share their MLS KeyPackage(s)
with the group, thus enabling an existing member to add new members to the
MLS group. In this context, KeyPackages distribution/processing can be modeled
a "queue of KeyPackages". Such a queue provides following properties:

- Multiple parties to write to it, when participants submit their KeyPackages.

- Multiple parties to read/process from the queue, to process the KeyPackage for
  updating the MLS group state.

~~~~
                       +---------------------------+   +--->
  Multiple    ---+     |                           |   |   Multiple
 Simultaneous    +---> |    MLS KeyPackage Queue   | --+ Simultaneous
   Writers       +---> |                           |   |   Readers
              ---+     +---------------------------+   +--->
~~~~


### Welcoming New Member

Once a MLS KeyPackage is verified, an existing member can add a new member to the
MLS group and send MLS Welcome message to invite the new member to join the
group. This procedure can be abstracted via message queues for each joiner to
receive MLS Welcome messages with the following properties:

- Accessible by multiple parties to write, but constrained so
  that only one party is allowed to write for a given epoch.

- One party, the recipient of the welcome message, is be able to read the
  MLS Welcome message.

~~~~
                       +--------------------------+   +--->
              ---+     |                          |   |
 1 writer per    +---> |   MLS Welcome Queues     | --+   Single
    epoch        +---> |   (1 queue per joiner)   | --+   Reader
              ---+     +--------------------------+   |
                                                      +--->
~~~~


### Updating MLS Group State

Members can update group's state when adding a new member, removing an
existing member or updating group's entropy at any time during a MLS
session. Group updates are performed via MLS Commit messages and successful
commits result in moving the MLS epoch further. MLS Commit message needs to
be processed by all the members to compute the shared group secret for that
epoch.

The distribution of commit messages can be modeled with a message queue
for MLS Commit messages with the following properties:

- Any member can access the commit queue for writing MLS Commit messages,
  but only one member is allowed to write per epoch.

- All the members can read and process MLS Commit message from the commit
  queue to update their group state.

~~~~
              ---+     +--------------------------+  +--->
 1 writer per     +--->|                          |--+   Multiple
    epoch         +--->|   MLS Commit Queue       |--+ Simultaneous
              ---+     |                          |  |    Readers
                       +--------------------------+  +--->
~~~~


# MLS Group Key Exchange over MOQT

Section {{mls-hld}} provided an non-normative abstracted view (via Queue metaphor)
to illustrate various MLS operations. Subsections
below provide further normative details on realizing those abstractions via
concepts from the MOQT data model (see {{ moqt-model}}).

## Bootstrapping MLS Session {#bootstrapping}

Each participant is provisioned, out of band, the MLS Group Name for a given
MOQ application instance. As part of bootstrapping a MLS Session, participating
MOQT endpoints needs to perform the following 2 actions:

1. Subscribe to receive published MLS KeyPackages over MOQT Track for an MLS group.

2. Publishing MLS KeyPackages over a MOQT Track.

To enable the above, following MOQT track definition is specified:

The TrackNamespace, termed "KeyPackage Namespace" is made up of
2 parts as shown below:

~~~~
KeyPackage Namespace := <mls-group-name> |  "keypackages"
~~~~
Note: The MLS group name chosen should be unique within a MOQ relay network.

All the members subscribe to the "KeyPackage Namespace" to receive
KeyPackages published over sender specific "KeyPackage Track"s as shown below,
where the Trackname identifes the sender of the the KeyPackage.

~~~~
KeyPackage Track
Tracknamespace :=  KeyPackage Namespace
Trackname      :=  SenderId
~~~~

The SenderId value choosen MUST be unique within the MOQ application. The
RECOMMENDED way to ensure uniques would be to use certifcate fingerprint
of the sender's public key.

There is one MOQT Group within the KeyPackage Track and objects within that
group identify different updates to the KeyPackage from a given publisher.
KeyPackage published at `Object ID` of '0' is used to initiate joining
a MLS group.

Below figure depicts a sample call flow on how the MOQT Namespace subscribe
is used to enable 2 participants (joiner1 and joiner2) to publish their
keypackages and have the member is able to process both of them


~~~~

┌───────┐                                       ┌─────┐                                       ┌───────┐┌──────┐
│Joiner1│                                       │Relay│                                       │Joiner2││Member│
└───┬───┘                                       └──┬──┘                                       └───┬───┘└──┬───┘
    │                                              │                                              │       │
    │        Announce(mls-grp1|keypackages)        │                                              │       │
    │─────────────────────────────────────────────>│                                              │       │
    │                                              │                                              │       │
    │                                              │        Announce(mls-grp1|keypackages)        │       │
    │                                              │<─────────────────────────────────────────────│       │
    │                                              │                                              │       │
    │                                              │      Subscribe_Namespace(mls-grp1|keypackages)       │
    │                                              │<─────────────────────────────────────────────────────│
    │                                              │                                              │       │
    │  Subscribe_Namespace(mls-grp1|keypackages)   │                                              │       │
    │<─────────────────────────────────────────────│                                              │       │
    │                                              │                                              │       │
    │                                              │  Subscribe_Namespace(mls-grp1|keypackages)   │       │
    │                                              │─────────────────────────────────────────────>│       │
    │                                              │                                              │       │
    │Namespace_Info(namespace=mls-grp1|keypackages,│                                              │       │
    │name=joiner1,alias=j1)                       │                                              │       │
    │─────────────────────────────────────────────>│                                              │       │
    │                                              │                                              │       │
    │                                              │    Namespace_Info(namespace=mls-grp1|keypackages,    │
    │                                              │    name=joiner1,alias=j1)                   │       │
    │                                              │─────────────────────────────────────────────────────>│
    │                                              │                                              │       │
    │                                              │Namespace_Info(namespace=mls-grp1|keypackages,│       │
    │                                              │name=joiner2,alias=j2)                       │       │
    │                                              │<─────────────────────────────────────────────│       │
    │                                              │                                              │       │
    │                                              │    Namespace_Info(namespace=mls-grp1|keypackages,    │
    │                                              │    name=joiner2,alias=j2)                   │       │
    │                                              │─────────────────────────────────────────────────────>│
    │                                              │                                              │       │
    │   Object(alias=j1,.., payload=KeyPackage)    │                                              │       │
    │─────────────────────────────────────────────>│                                              │       │
    │                                              │                                              │       │
    │                                              │       Object(alias=j1,.., payload=KeyPackage)│       │
    │                                              │─────────────────────────────────────────────────────>│
    │                                              │                                              │       │
    │                                              │   Object(alias=j2,.., payload=KeyPackage)    │       │
    │                                              │<─────────────────────────────────────────────│       │
    │                                              │                                              │       │
    │                                              │       Object(alias=j2,.., payload=KeyPackage)│       │
    │                                              │─────────────────────────────────────────────────────>│
┌───┴───┐                                       ┌──┴──┐                                       ┌───┴───┐┌──┴───┐
│Joiner1│                                       │Relay│                                       │Joiner2││Member│
└───────┘                                       └─────┘                                       └───────┘└──────┘

~~~~

## Creating/Joining a MLS Group

Creating or Joining an MLS group requires a way for boostrapping the
group when the first member joins and a way to decide an existing member
for processing the MLS KeyPackage to add the new member.

In order to realize the above functionalities and ensure the criticial
invariants {{invariants}}, a centralized "Epoch Counter Service"
(see epoch-svc) is required to address/resolve contention issues
when multiple participants carryout the create/join procedures.

Participants intending to join/create a MLS grooup, try to acquire lock from
the counter service. The request identifies the MLS Group identified by its
GroupId and epoch '0' as the counter to obtain the lock. The response can
be one of the following:

* Ok: A response of OK implies that there doesn't exist an MLS Group.
In this scenario, the participant is the first participant and thus
creates the group unilaterally and generates the initial secret for the group.
Following which the participant releases the acquired lock by performing the
increment operation for the obtained lock, on the counter
service.

* Locked: A response of "Locked"  implies a conflicting request
and the requestor has to retry acquiring the lock, after the lock expiry timeout
provided in the response.

* CounterError: A response of CounterError implies that the service has
a different value of the current counter than the one requested (epoch 0).
This happens when the requested MLS Group has already been created. In such
situations, the participant awaits for an existing member to add the
participant and publish the MLS Welcome message (see {{commits_welcome}})


## Updating Group State {#commits_welcome}

Updating MLS group state requires {{invariants}} to be satifisfied. This means
that the changes have to be done linearly and changes to the group state
MUST be performed by a single member within a MLS group for a given epoch.

The process of updating the group state is described below:

1. Acquire lock for the current epoch from the counter service.

2.a If the lock was successfully acquired and member is attempting to add
a new member, process the MLS KeyPackage(s) available over per
joiner's KeyPackage track, generate set of MLS Welcome
messages per joiner and a single MLS Commit message for the group. Publish
individual MLS Welcome messages to the intended recipeints
on per recipient welcome track (see {{process-welcome}}) and Publish MLS Commit
message to all the participants (see {{process-commit}}).

2.b If the lock was successfully acquired and the operation is to remove a
member, update the MLS state to remove the member, generate MLS Commit message
and publish the generated MLS Commit message to all the participants (see {{process-commit}}).

3. If the response was "Locked", following the procedures for retrying as
defined in (locked).

4. A lock response of "CounterError" implies the member attempting to
update the MLS group state is behind and MUST await until it catches
up with all the MLS Commit messages in transit. It is important to note,
this situation MAY also imply that another member won the contention
to update the group state before this member can make the change.


### Processing MLS Welcome Message {#process-welcome}

In order to be able to publish MLS Welcome message and process the
same over MOQT, following track naming scheme is specified.

The TrackNamespace, termed "Welcome Namespace" is divided into
2 parts as shown below:

~~~~
Welcome Namespace := <mls-group-name> |  "welcome"
~~~~
Note: The MLS group name chosen should be unique within a MOQ relay network.

MLS Welcome message is published over a track that is specific to individual
recipient. Joining participants subscribe to the "Welcome Track" as part
of MLS session bootstrapping, which has the following structure:

~~~~
Welcome Track
Tracknamespace :=  Welcome Namespace
Trackname      :=  RecipientId
~~~~

The RecipientId value choosen MUST be unique within the MOQ application. The
RECOMMENDED way to ensure uniques would be to use certifcate fingerprint
of the recipients's public key. The group member publishing the MLS Welcome
message can obtain the RecipientId while processing the KeyPackage of
the member being addded.

On receipt of the Welcome message, local MLS state is updated with the
received MLS Welcome message to obtain the group secret for the current
epoch.

When publishing on the "Welcome Track", there is  one MOQT group per MLS epoch
and objectId 0 carries the MLS Welcome message.

### Processing the MLS Commit Messages {#process-commit}

All the members subscribe to receive MLS Commit message and they do so
by subscribing to the "Commit Track" as shown:

~~~~
Commit Track
Tracknamespace :=  <mls-group-name>
Trackname      :=  commits
~~~~

MLS Commit message updates the existing member about group changes, such
as adds/removes and entropy updates. Publish to the "Commit Track"  happens
with one MOQT group per MLS epoch and objectId 0 carries the MLS Commit message.


# Epoch Counter Service {#epoch-svc}

A counter service tracks a collection of counters with unique identifiers.
In an MLS context, the counter value is equal to the MLS epoch, and the
counter identifier is the MLS group identifier.

Before a counter can be incremented, it must be locked.  As part of the lock
operation, the caller states what their expected next counter value, which
much match the service's expectation in order for the caller to acquire the
lock.  Since the actual updates to the counter are out of band, this ensures
that the caller has the correct current value before incrementing.

There is no explicit initialization of counters.  The first call to `lock`
for a counter must have `expected_next_value` set to 0.

There is no method provided to clean up counters.  A service may clean up a
counter if it has some out-of-band mechanism to find out that the counter is
no longer needed.  For example, in an MLS context, once the MLS group is no
longer in use, its counter can be discarded.

## Lock API {#counter-lock}

This is a simple REST style API over HTTPS used to request lock for
a counter for a provided Counter ID.

~~~~
GET /lock/<Couner ID>?val=<counter>
~~~~

Returns "Ok" if lock acquistion succeded, a "Confict" response when lock is
already held with a retry_later time for retrying the lock acquistion or a
"CounterError" with the current value of the counter when the requested counter
doesn't match the `expected_next_value`.

## Increment API {#counter-incr}

The increment HTTPS API allows the counter value stored in `expected_next_value`
to be incremented for the provided Counter ID.

~~~~
POST /increment/<Counter ID>
~~~~

Returns "Ok" if the counter value was successfully incremented, a "Error"
responses if the provider "Counter ID" hasn't been locked yet.

TODO: Define Error responses and codes for authorization failures.


# Interactions with MOQ Secure Objects

MLS Key agreement generates a group shared secret, called "MLS Mater Key",
per MLS Epoch. Epochs in MLS are incremented whenever there is changed
in the group state due to an existing member commit the changes to the group.

MLS generated shared group secret per epoch can be used to derive
`track_base_key` when using SecureObjects (see Section 5 ) for
protecting the objects within a MOQT track.

The procedure for the same is as defined below:

1. For each combination of (MLS Epoch, MLS Master Key) an 'Epoch Secret' is dervied

~~~~
Epoch Secret = HKDF.Extract("SecureObject Epoch Master Key " | MLS Epoch, MLS Master Key)
~~~~

2. 'Epoch Secret' is used to derive `track_base_key` per `FullTrackName`
(see Section 3 of {{SecureObjects}})

~~~~
track_base_key = HKDF.Expand("SecureObject Track Base Key " | FullTrackName, Epoch Secret)
~~~~

When encrypting/decrypting objects using SecureObject, the epoch under which the
`track_base_key` was computed is used as `KID` in the SecureObject Header. The
`track_base_key` computed is used to derive per object keys and nonce as defined in
Section 5 of {{SecureObjects}}. All the objects within a given epoch are encrypted/decrypted with the keys derived from the `Epoch Secret` for that epoch.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
