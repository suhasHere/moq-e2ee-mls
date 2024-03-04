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
  SecureObects: I-D.draft-jennings-moq-secure-objects

informative:


--- abstract

TODO Abstract


--- middle

# Introduction

TODO Introduction

# Conventions and Definitions

{::boilerplate bcp14-tagged}


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
from one epoch to the next.

In order to setup end to end encryption of media delivered over MOQT
delivery network, producders and consumers participate in the MLS
exchange to setup group secret through which are used to derived the
keys needed for encrypting the media/data published by the members of
the MLS group.

Below figure captures a typical setup for clients to participate in the
MLS protocol, with Delivery Service (DS) acting as the rendezvous
point. In the example setting, participants A, B and C involve in
protocol exchange to setup end to end encrypted session keyed via MLS.



## Critical Invariants

* Only one group is created per moq session
* Linear sequence of Commits - Each Commit has exactly one successor


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
track namespace in the subscription matches the track namespace in the annoncement
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
      │         Media Flow          │                          │
      │────────────────────────────>│                          │
      │                             │                          │
      │                             │        Media Flow        │
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
of request for subscribing to the track namespace.

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
on new track under a track namespace requested in {{message-subscribe-namespace}}
message. The `NAMESPACE_INFO` message is a implicit subscription, unless
it is explicitly unsubscribed by the subscriber by sending `UNSUBSCRIBE` message.
This message provides necessary mapping between `Track Alias`, `Subscribe Id`
to the namespace requested in the `SUBSCRIBE NAMESPACE` message.

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


# MLS and MOQ


This specification defines procedures for participants engaging in MLS
key exchange to happen over MOQT protocol, thus enabling following 2 goals:

1. Use MOQT as delivery transport for MLS protocol meesages.

2. Allow MOTQ endpoints (producers/consumers) to use MLS as secure
key exchange protocol for end to end secure commnunications across
range of use-cases.


## High-level Design {#mls-hld}

MLS {{!RFC9420}} achieves group key agreement by participants/members
engaging in MLS protocol message exchange that allows:

 - New members to express their interest to join a  MLS group
 - Exsiting members to commit a new members to a MLS group
 - Existing members to commit removal of existing members from a MLS group


The central unit of functionality in MLS is a group, where at any given time,
a group represents a secret known only to its members. Membership to the group
can change over time. Each time membership changes (batch of joins or
leaves), the shared secret is changed to one known only by the current
members. Each period of time with stable membership/secret is an epoch.

At a high level, one can envision MLS protocol operation in the form
multiple queue abstractions to achieve the above functionality.

### Keypackage Distribution

All participants interested in joining a MLS group share their MLS KeyPackage(s)
with the group, thus enabling an existing member to add new members to the
MLS group. In this context, KeyPackages distribution/processing can be modeled
a "queue of KeyPackages". Such a queue provides following properties:

- Multiple parties to write to it, when participants submit theur KeyPackages.

- Mulitple parties to read/process from the queue, to process the KeyPackage for
  updating the MLS group state.

~~~~

                       +---------------------------+   +--->
  Multiple    ---+     |                           |   |   Multiple
 Simultenous     +---> |    MLS Keypackage Queue   | --+ Simulatenous
   Writers       +---> |                           |   |   Readers
              ---+     +---------------------------+   +--->
~~~~


### Welcoming New Member

Once a MLS KeyPackage is verified, an existing member can add a new member to the
MLS group following procedures in (see todo) and send MLS Welcome message
to invite the new member to join the group. This procedure can be abstracted
via a message queue for MLS Welcome messages with the following properties:

  - To be able to accessible by multiple parties to write, but constrained so
    that only one party (existing member) is allowed to write for a given epoch.

  - One party, the recipient of the welcome, is be able to read the MLS Welcome
   message in a given epoch.

~~~~

                       +--------------------------+   +--->
              ---+     |                          |   |
 1 writer per    +---> |   MLS Welcome Queue      | --+   Single
    epoch        +---> |                          | --+   Reader per
              ---+     +--------------------------+   |    epoch
                                                      +--->


~~~~


### Updating Group State

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
    queue every epoch to update their group state.

~~~~
              ---+     +--------------------------+  +--->
 1 writer per     +--->|                          |--+   Multiple
    epoch         +--->|   MLS Commit Queue       |--+ Simulatenous
              ---+     |                          |  |    Readers
                       +--------------------------+  +--->
~~~~


## MLS Group Key Exchange over MOQT

Section {{mls-hld}} provided an non-normative abstracted view (via Queue metaphor) 
to illustrate various MLS operations for setting up an MLS group. Subsections 
below provide further normative details on realizing those abstractions via 
concepts from the MOQT data model {{moqt-model}}

## Bootstrapping MLS Session

As part of bootstrapping a MLS Session, participating MOQT endpoints needs to
perform the following 2 actions:

1. Subscribe to MOQT track for processing MLS KeyPackages.

2. Publishing MLS keypackages identifying the credentials to the
   "keypackage" track


## Joining to MLS Group

Adding or joining an MLS group requires one of the following:

1. A way for boostrapping the group when the first member joins.

2. A way to choose an existing member to add a new member.

In order to realize the above functionalities and ensure the criticial
invariants, a centralized lock service is required to help resolve
contention.

Participants intending to join try to acquire lock to create/join the
group.  If the lock can be successfully acquired and the response
indicates "Create", the participant is the first participant and he
creates the group unilaterally and generate the initial secret. Then the
pariticipant release the "crate_or_join" lock.

Alternately, if the repsonse indicate "Join", then the participant
awaits for an e existing member to process the request to join the
group.

When an existing member receives KeyPackage, the process of adding the
new member is as follows:

1. Acquire lock to commit to the group for a given epoch

2. If lock acquired sucessfully, process the keyPackage for
   duplicates/error, create MLS Welcome and Commit messages, publish
   them to the "Welcome" and "Commit" tracks for the MLS Group and the
   epoch. See {{mls-tracks}} for further details on the tracks.

3. If lock cannot be acquired due to conflict for a given epoch, retry
   after a confgured timeout. One of the 2 things might happen:

   - Another member was able to successfuly perform group update for the
     current epoch, in which case it is recommended for the member to
     process MLS messages before retrying the operation again for the
     same epoch to ensure the group updates are already what the member
     wants.

   - Another member updated the group state with commits that are
     different from the member attempting to obtain the lock. In such a
     scenario, the member needs to wait to process the commits in
     transit and retry step 1 for the next right epoch in the sequence


## Processing the Welcome Message

Participants await MLS Welcome message after publishing their Keypackage
to join the group. They do so by subscribing to the "Welcome" track. On
receipt of the Welcome message, local MLS state is updated with the
recevied Welcome message to obtain the group secret for the current
epoch. If the participant is already a member of the group, the Welcome
message is dropped/ignored.


## Removing from the group

TODO: Add details


## Processing the MLS Commit Messages

All the participants subscribe to "Commit" track for a given MLS
group. This allows them to process MLS commit messages being published
under the following group update operations

 - Add a new member

 - Remove an existing member

TODO: SHOULD we even support other updates ?

# MLS MoQ Tracks {#mls-tracks}


For all the various tracks:

- The TrackNamespace is scoped to the combination of MLS Group and MLS
  operation. The MLS group name chosen should be unique within a MOQ
  relay network.

~~~~
TrackNamespace := <mls-group-name> | <mls-operation>
~~~~

- TrackName identifies the sender performing the operation. The value
  chosen for sender MUST be unique within a MOQ application.

TODO: add notes on ways to choose the sender

## KeyPackage

Subscribe happens on the track namespce

~~~~
TrackNamespace := mls-group | "keypackage" and
~~~~

and publishes happens on the

~~~~
 FullTrackName := mls-group | "keypackage" | sender-id
~~~~

There is one MOQT group and objects within that group identify different
updates of the KeyPackage with objectId of 0 being sent at the time of
joining a MLS group


## Welcome

Subscribes happens on the on the track namespce

~~~~
TrackNamespace := mls-group | "welcome"
~~~~

and Publish happens on
~~~
FullTrackName := mls-group | "welcome" | sender-id
~~~

There is one MOQT group per MLS epoch and objectId 0 carries the MLS
Welcome message.

## Commit

Subscribe happens on the namespace

~~~~
TrackNamespace := mls-group | "commit"
~~~~

and Publishes happens on
~~~~
FullTrackName := mls-group | "commit" | sender-id
~~~~

There is one MOQT group per MLS epoch and objectId 0 carries the MLS
Commit message.


# Epoch Service {#epoch-svc}

The following REST endpoints can then be used to access the different 
functionalities of the Epoch Service.


## Create/Join Group API

## Commit API


# Interactions with MOQ Secure Objects

TODO: Describe epoch secret -> track_base_key


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
