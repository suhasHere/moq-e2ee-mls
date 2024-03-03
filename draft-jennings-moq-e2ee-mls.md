---
title: "MoQ and MLS - Secure Group Key Agreement Over MoQ"
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
  github: "suhasHere/moq-secure-objects"
  latest: "https://suhashere.github.io/moq-secure-objects/#go.draft-jennings-moq-secure-objects.html"

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

informative:


--- abstract

TODO Abstract


--- middle

# Introduction

TODO Introduction

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# MLS Overview

MLS protocol provides continuous group authenticated key exchange.  MLS provides several important security properties

* Group Key Exchange: All members of the group at a given time know a secret key that is inaccessible to parties outside the group.
* Authentication of group members: Each member of the group can authenticate the other members of the group.
* Group Agreement: The members of the group all agree on the identities of the participants in the group.
* Forward Secrecy: There are protocol events such that if a member's state is compromised after the event, group secrets created before the event are safe.
* Post-compromise Security: There are protocol events such that if a member's state is compromised before the event, the group secrets created after the event are safe.

At a very high level, MLS protocol operates by participants sending proposals to add/remove/update the group state and an active member of the group commit the proposals  to move the group’s cryptographic state from one epoch to the next.

In order to setup end to end encryption of media delivered over MOQT delivery network, producders and consumers participate in the MLS exchange to setup group secret through which are used to derived the keys needed for encrypting the media/data published by the members of the MLS group.


Below figure captures a typical setup for clients to participate in the MLS protocol, with Delivery Service (DS) acting as the rendezvous point. In the example setting, participants A, B and C involve in protocol exchange to setup end to end encrypted  session keyed via MLS.

TODO: Add a simple call-flow here


## Critical Invariants

* Only one group is created per moq session
* Linear sequence of Commits - Each Commit has exactly one successor



# MLS and MOQ

Unit of MLS functionality is a group, where at any given time, a group represents a secret known only to its members. Membership to the group can change over time. Each time membership changes (batch of joins or leaves), the shared secret is changed to one known only by the current members. Each period of time with stable membership/secret is an epoch

## Bootstrapping MLS Session
As part of bootstrapping a MLS Session, each MOQT endpoint needs to perform
following 2 actions:

1. Subscribe to MOQT track for processing MLS KeyPackages (see {{{mls-tracks}}}) for processing addition of new members to the group.
2. Publishing their  MLS keypackages identifying the credentials to the "keypackage" track


## Joining to MLS Group

Adding or joining an MLS group requires one of the following:

1. A way for boostrapping the group when the first member joins.
2. A way to choose an existing member to add a new member.


In order to realize the above functionalities and ensure the criticial invariants, a centralized lock service is required to help resolve contention.


Participants intending to join try to acquire lock to create/join the group.
If the lock can be successfully acquired and the response indicates "Create", the participant is the first participant and he creates the group unilaterally and
generate the initial secret. Then the pariticipant release the "crate_or_join" lock.

Alternately, if the repsonse indicate "Join", then the participant awaits for an e
existing member to process the request to join the group.


When an existing member receives KeyPackage, the process of adding the new
member is as follows:

1. Acquire lock to commit to the group for a given epoch

2. If lock acquired sucessfully, process the keyPackage for duplicates/error, create MLS Welcome and Commit messages, publish them to the "Welcome" and "Commit" tracks for the MLS Group and the epoch. See {{mls-tracks}} for further details on the tracks.

3. If lock cannot be acquired due to conflict for a given epoch, retry after a confgured timeout. One of the 2 things might happen:

   - Another member was able to successfuly perform group update for the current epoch, in which case it is recommended for the member to process MLS messages before retrying the operation again for the same epoch to ensure the group updates are already what the member wants.

   - Another member updated the group state with commits that are different from the member attempting to obtain the lock. In such a scenario, the member needs to wait to process the commits in transit and retry step 1 for the next right epoch in the sequence

## Processing the Welcome Message

Participants await MLS Welcome message after publishing their Keypackage to join the group. They do so by subscribing to the "Welcome" track. On receipt of the Welcome message, local MLS state is updated with the recevied Welcome message to obtain the group secret for the current epoch. If the participant is already a member of the group, the Welcome message is dropped/ignored.


## Removing from the group

TODO: Add details


## Processing the MLS Commit Messages

All the participants subscribe to "Commit" track for a given MLS group. This allows them to process MLS commit messages being published under the following group update operations
 - Add a new member
 - Remove an existing member

 TODO: SHOULD WE even support other updates ?

# MLS MoQ Tracks {#mls-tracks}

For all the various tracks:

- The TrackNamespace is scoped to the combination of MLS Group and MLS operation. The MLS group name chosen should be unique within a MOQ relay network.

~~~~
TrackNamespace := <mls-group-name> | <mls-operation>
~~~~

- TrackName identifies the sender performing the operation. The value chosen for sender MUST be unique within a MOQ application.

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

There is one MOQT group and objects within that group identify different updates of the KeyPackage with objectId of 0 being sent at the time of joining a MLS group


## Welcome

Subscribes happens on the on the track namespce

~~~~
TrackNamespace := mls-group | "welcome"
~~~~

and Publish happens on
~~~
FullTrackName := mls-group | "welcome" | sender-id
~~~

There is one MOQT group per MLS epoch and objectId 0 carries the MLS Welcome message.

## Commit

Subscribe happens on the namespace
~~~~
TrackNamespace := mls-group | "commit"
~~~~

and Publishes happens on
~~~~
FullTrackName := mls-group | "commit" | sender-id
~~~~

There is one MOQT group per MLS epoch and objectId 0 carries the MLS Commit message.


# Centralized Lock Service {#lock}

TODO: Define API for the lock service

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
