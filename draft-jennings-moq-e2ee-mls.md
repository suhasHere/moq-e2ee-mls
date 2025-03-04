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
    ins: S. Nandakumar
    name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com
 -
    ins: R.L. Barnes
    name: Richard L. Barnes
    organization: Cisco
    email: rlb@ipv.sx

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
the next commit (See {{ctr-svc}}.


# MOQ Overview {#moqt-model}

MOQT {{MoQTransport}} defines a publish/subscribe based media delivery
protocol, where in endpoints, called producers, publish objects which are
delivered via participating relays to receiving endpoints, called consumers.

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
Alice (the producer), Bob (the consumer) and Relay. The MOQT
protocol exchange starts with Alice sending MOQT Announce message with
TrackNamespace under which she is going to publish media tracks.
Then Bob issues a MOQT Subscribe message to the relay for a FullTrackName
(identified  by its TrackNamespace and TrackName) expressing his interest to
receive media. Relay makes upstream subscription to Alice since the
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

## Proposed changes to MOQT Protocol

In order to realize the MLS key exchange over MOQ, this specification
proposes following changes to MOQ Transport. The changes are to
be discussed within the MOQ WG and will be deleted from this draft.

### Announce Full Track Name

Announcing to Full Track Name allows authorized original publishers to publish
their objects before the subscribers express their interest. We propose to
modify the Announce message to include the FullTrackName as shown below:

~~~

ANNOUNCE Message {
Type (i) = 0x6,
Length (i),
Track Namespace (tuple),
Track Name Length(i),
Track Name (..),
Number of Parameters (i),
Parameters (..) ...,
}
~~~

If the Track Name Length is zero, the Track Name is not included in the
Announce Message.


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
below provide further normative details on realizing those abstractions through
mapping to the MOQT data model (see {{moqt-model}}).

## Bootstrapping MLS Session {#bootstrapping}

Each participant is provisioned, out of band, the MLS Group Name for a given
MOQ application session. As part of bootstrapping a MLS Session, participating
MOQT endpoints needs to able to publish their MLS KeyPackages and express their
interest to join a MLS group. The latter of which is discussed further in
{{join-group}}.

### KeyPackage Distribution {#kp-dist}

Participants interested in joining a MLS group publish their MLS KeyPackage
by writing to the "KeyPackage" MOQT track whose details are defined below:

~~~
KeyPackage TrackNamespace := ("moq.mls.arpa/v1"),(<mls-group-name>)
KeyPackage TrackName      := ("keypackages")
KeyPackage FullTrackName  := KeyPackage TrackNamespace | KeyPackage TrackName
~~~

The MLS group name chosen MUST be unique within a MOQ relay network.

There is one MOQT Group per participant, where the Group ID represents
the Sender/Participant within the MLS Group. Each participant
is identified by a SenderID value and MUST be unique within the
MOQT Session. The MOQT Object is used to carry participant's
MLS KeyPackage. A participant can update their KeyPackage by
publishing a new object with the same group.

Online members with active subscription to the "KeyPackage" track receive
KeyPackages published by the participants. Members who are offline
continue with their subscriptions to the "KeyPackage" track when
they come online and also issue FETCH request to retrieve the missed
MLS KeyPackages published since they were last online. The Sender ID
value to be used for the MOQT Group ID for FETCH request is obtained
via "Create/Join" flow as defined in {{join-group}}.

Publishers of the KeyPackage SHOULD set the cache duration to take
into consideration the offline nature of the members. The cache
duration of 12 hours is RECOMMENDED.

#### Rationale for using Sender ID to be the MOQT Group ID

One can envision one MOQT Track per sender instead of the above
proposal for MLS KeyPackage publishing. However, the challenge
with such an approach is that it would require each subscribers
to learn about all the Sender IDs in the MOQ Session. Even
though approaches like "Subscribe_Announces" might help when
all the members are online, it doesn't help when members are
offline. The current proposal of having one MOQT Track for
KeyPackage distribution address the aforementioned drawback.

## Creating/Joining a MLS Group {#join-group}

Participants intending to join a MLS group do so by
sending "Join Request" over a MOQT Track called "Join Track",
as defined below:

~~~
Join TrackNamespace := ("moq.mls.arpa/v1"),(<mls-group-name>)
Join TrackName      := ("join")
Join FullTrackName  := Join TrackNamespace | Join TrackName
~~~

The MOQT Group ID is determined via the "Counter Service"
(see {{ctr-svc}}) as described in the {{join-group-id}}.
MOQT Object IDs starting from 0 are used to carry the
"JoinRequest" message as shown below:

~~~~
JOIN Message {
Type (i) = 0x1,
Sender ID (i)
}
~~~~

* Sender ID: Identifier of the participant intending to join the MLS group.
This MUST match the Sender ID used for publishing the MLS KeyPackage.

It is RECOMMENDED that Join messages be cached in the relays by
setting the max_cache_duration to atleast 30 minutes.


### MOQT Group ID Determination {#join-group-id}

Creating or Joining an MLS group requires a way for boostraping the
group when the first member joins and a way to decide an existing member
for processing the MLS KeyPackage to add the new member.

Participants intending to join/create a MLS group try to acquire lock
from the counter service on the join endpoint {{counter-join}}. The request
identifies the MLS Group Name as the Counter ID to obtain the lock.

The response can be one of the following:

* Ok: A response of OK on the "Join" MOQT Track implies that there
doesn't exist an MLS Group. In this scenario, the participant is the
first participant and thus creates the group unilaterally and generates the
initial secret for the group. Following which the participant releases the
acquired lock by performing the increment operation for the obtained lock,
on the counter service.

* Locked: A response of "Locked"  implies a conflicting request
and the requestor has to retry acquiring the lock, after the lock
expiry timeout provided in the response.

* CounterError: A response of CounterError implies that the service has
a different value of the current counter than the one requested (counter 0).
This happens when the requested MLS Group has already been created. In such
situations, the participant awaits for an existing member to add the
joining participant and publish the MLS Welcome message
(see {{commits_welcome}}).


## Updating Group State {#commits_welcome}

Updating MLS group state requires {{invariants}} to be satisfied.
This means that the changes have to be done linearly and changes to
the group state MUST be performed by a single member within a MLS group
for a given epoch.

Group state in MLS can be udpated by adding a new member, removing
an existing member or updating the group's entropy.

### Adding a member to the MLS Group

Members obtain a list of participants interested in joining a MLS group
either as part of updates to their subscriptions to the "Join" Track and/or
by issuing FETCH request to retrieve the missed MLS Join messages
based on the Latest Group ID in the Subscribe_OK message. This supports
processing join requests even when the members were offline for a period
of time.


The following process followed when adding a new member to a given MLS Group:

1. Acquire lock for the current epoch from the counter service ({{counter-commit}}).

2. If the lock was successfully acquired retrieve the MLS KeyPackage(s)
from the cache by issuing FETCH request to the "KeyPackage" track against the
the Sender ID in the Join message. The Sender ID maps to the start_group in the
FETCH request and end_group is set to start_group + 1. If successfully retrieved,
process the KeyPackage and generate set of MLS Welcome messages per joiner and
a single MLS Commit message for the group. Publish individual MLS Welcome messages
to the intended recipeints on per recipient welcome track (see {{process-welcome}})
and Publish MLS Commit message to all the participants (see {{process-commit}}).

### Removing a member from the MLS Group

If the lock was successfully acquired and the operation is to remove a
member, update the MLS state to remove the member, generate MLS Commit message
and publish the generated MLS Commit message to all the participants
(see {{process-commit}}).

In either of the flows, If the response was "Locked", follow the
procedures for retrying. A lock response of
"CounterError" implies the member attempting to
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
Welcome TrackNamespace := ("moq.mls.arpa/v1"),(<mls-group-name>),(<welcome>)
~~~~

MLS Welcome message is published over a track that is specific to individual
recipient. Joining participants subscribe to the "Welcome Track" as part
of MLS session bootstrapping, which has the following structure:

~~~~
Welcome Tracknamespace :=  Welcome TrackNamespace
Welcome Trackname      :=  (<Paricipant ID>)
~~~~

The Paricipant ID is same as the Sender ID obtained from the
Join message when processing the MLS KeyPackag. On receipt of the
Welcome message, local MLS state is updated with the received
MLS Welcome message to obtain the group secret for the current
epoch.

When publishing on the "Welcome Track", there is one MOQT group per MLS epoch
and objectId 0 carries the MLS Welcome message.

### Processing MLS Commit Messages {#process-commit}

All the members subscribe to receive MLS Commit message and they do so
by subscribing to the "Commit Track" as shown:

~~~~
Commit Tracknamespace :=  ("moq.mls.arpa/v1"),(<mls-group-name>)
Trackname      :=  commit
~~~~

MLS Commit message updates the existing member about group changes, such
as adds/removes and entropy updates. Publish to the "Commit Track"  happens
with one MOQT group per MLS epoch and objectId 0 carries the MLS Commit message.


# Counter Service {#ctr-svc}

A counter service tracks a collection of counters with unique identifiers.
In an MLS context, the counter value is equal to the MLS epoch when performing
MLS group commit operations (see {{commits_welcome}}) and an incrementing
counter value for processing MLS Group Join operations (see {{join-group}}), and the
counter identifier is the MLS group identifier/MLS group name.

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

### Join API {#counter-join}
Join lock API is used to acquire lock for a counter for a given Counter ID
when a participant is trying to join a MLS group. The Counter ID
MUST correspond to MLS Group Name.

~~~~
GET /lock/join/<Counter ID>?val=<counter>
~~~~

### Commit API {#counter-commit}
Commit lock API is used to acquire lock for a counter for a given Counter ID
when a participant is trying to update the MLS group state. The Counter ID
MUST correspond to MLS Group Name.

~~~~
GET /lock/commit/<Counter ID>?val=<counter>
~~~~

Above APIs can be responded with the following responses:

* "Ok" response impliesthat lock acquisition was successfull,
"Confict" response implies that lock is already held with a retry_later time for
retrying the lock acquisition.
* "CounterError" response with the current value of the counter is returned when
the requested counter doesn't match the `expected_next_value`.

## Increment API {#counter-incr}

The increment HTTPS API allows the counter value stored in `expected_next_value`
to be incremented for the provided Counter ID.

### Join API {#increment-join}
The increment Join API is used to increment the counter value for a given Counter ID
when a participant has successfully acquired the lock on performing
the join operation. The Counter ID MUST correspond to MLS Group Name.

~~~~
POST /increment/join/<Counter ID>
~~~~

### Commit API {#increment-commit}
The increment commit API is used to increment the counter value for a given Counter ID
when a participant has successfully acquired the lock on performing
the commit operation. The Counter ID MUST correspond to MLS Group Name.

~~~~
POST /increment/commit/<Counter ID>
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

For each combination of (MLS Epoch, MLS Master Key) an 'Epoch Secret'
is derived:

~~~~
Epoch Secret = HKDF.Extract("SecureObject Epoch Master Key " | MLS Epoch, MLS Master Key)
~~~~

'Epoch Secret' is used to derive `track_base_key` per `FullTrackName`
(see Section 3 of {{SecureObjects}}):

~~~~
track_base_key = HKDF.Expand("SecureObject Track Base Key " | FullTrackName, Epoch Secret)
~~~~

When encrypting/decrypting objects using SecureObject, the epoch under which the
`track_base_key` was computed is used as `KID` in the SecureObject Header. The
`track_base_key` computed is used to derive per object keys and nonce as defined in
Section 5 of {{SecureObjects}}. All the objects within a given epoch are
encrypted/decrypted with the keys derived from the `Epoch Secret` for that epoch.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
