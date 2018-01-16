---
title: Messaging Layer Security Architecture
abbrev: MLS Architecture
docname: draft-rescorla-mls-architecture-latest
category: info

ipr: trust200902
area: Security
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: E. Rescorla
    name: Eric Rescorla
    organization: Mozilla
    email: ekr@rtfm.com
 -
    ins: B. Beurdouche
    name: Benjamin Beurdouche
    organization: INRIA
    email: benjamin.beurdouche@inria.fr
 -
    ins: E. Omara
    name: Emad Omara
    organization: Google
    email: emadomara@google.com
 -
    ins: S. Inguva
    name: Srinivas Inguva
    organization: Twitter
    email: singuva@twitter.com

normative:
  RFC2119:

--- abstract

TODO

--- middle

# Introduction

End-to-end security is a requirement for instant messaging systems
and is commonly deployed in many such systems designed over the past
few years. In this context, what end-to-end means is that the
users of the system enjoy some level of security -- with the precise
level depending on the system design -- even when the messaging
service they are using misbehaves.

Messaging Layer Security (MLS) specifies an architecture (this document)
and an abstract protocol [TODO:XREF] for providing end-to-end security
in this setting. MLS is not intended as a full instant messaging
protocol but rather is intended to be embedded in a concrete protocol
such as XMPP [TODO:REF]. In addition, it does not specify a complete
wire encoding, but rather a set of abstract data structures which
can then be mapped onto a variety of concrete encodings, such as
TLS {{?I-D.ietf-tls-tls13}}, CBOR {{?RFC7049}}, and JSON {{?RFC7159}}.
Implementations which adopt compatible encodings should be able to
have some degree of interoperability at the message level (though perhaps
not at the authentication level).


# General Setting

[TODO: Need some ASCII art]

A model system is shown in [TODO: Figure].

The Messaging Service (MS) presents as two abstract services:

- An Authentication Service (AS) which is responsible for maintaining
  user identities, issuing credentials which allow them to
  authenticate to each other, and potentially distributing
  user keying material.

- A Delivery Service (DS) which is responsible for delivering messages
  between users. In the case of group messaging, the delivery service
  may also be responsible for acting as an "exploder"
  where the sender sends a single message to a group and
  the switch then forwards it to each recipient.

In many systems, the AS and the DS are actually operated by the
same entity and may even be the same server. However, they
are logically distinct and, in other systems, may be operated
by different entities so we show them as separate here. Other
partitions are also possible, such as having a separate directory
server.

A typical scenario might look something like this:

1. Alice, Bob, and Charlie create accounts with the messaging
   service and obtain credentials from the AS.

1. Alice, Bob, and Charlie authenticate to the DS and store
   some keying material which can be used to encrypt to them
   for the first time.

1. When Alice wants to send a message to Bob and Charlie, she
   contacts the DS and looks up their keying material. She
   uses those keys to establish a set of keys which she can
   use to send to Bob and Charlie. She then sends the
   encrypted message(s) to the DS, which forwards them to
   the ultimate recipients.

1. Bob and/or Charlie respond to Alice's message. Their messages
   might include new keys which allow the joint keys to be updated,
   thus providing post-compromise security {{post-compromise-secrecy}}.

## Clients

Endpoints that are not an AS nor a DS are called Clients. These
clients will typically correspond to end-user devices such as phones,
web clients or other devices running MLS.

Each client owns a set of keys that uniquely define the identity of
its endpoints.
A single end-user may operate multiple devices simultaneously
(e.g., a desktop and a phone) or sequentially (e.g., replacing
one phone with another).

MLS has been designed to provide similar security guarantees to all
clients. Note that while MLS provide some level of security resilience
against of a compromised clients, the maximum security level requires
the endpoints to connect to the messaging service on a regular basis
and to use compliant implementations in order to realize security
operations such as deleting intermediate cryptographic keys.

## Delivery Service

The Delivery Service (DS) is expected to play multiple roles in the
MLS architecture.

Multiple levels of security and trust for the DS are considered by MLS
according to each tasks performed by the DS.

### Delivery of the initial keying material

In the MLS group communication establishment process, the first step
exercised by the DS is to store the initial cryptographic key material
provided by every Member. This key material represents the initial public
identity of the Member and will subsequently be used to establish
the set of keys that will be used by the Members to communicate with
other members of the group.

In an Untrusted setting, it is assumed by the MLS threat model that
the identity provided by the DS to an honest Member of the Group can
be incorrect. Hence, MLS offers the clients a way of multilaterally
verify the relationship between the other members of the group expected
identities and the keys provided by the MS through a public Key
Transparency (KT) log. While this is useful to circumvent trust issues
in the case of a potentially corrupted DS, this check can be
computationnaly costly for the clients.

In a Trusted setting, the DS is expected to always provide the correct
and most up-to-date information to a Member requiring another Member's
initial keying material. Still, clients can choose to examine the KT log,
if available, to make sure the keys they will be using are correct.

### Delivery of messages and attachments

Delivery in order and resilience against intermittent message loss
are the two main properties expected by MLS from the DS.
Another guarantee provided by MLS is that Clients will know after
receiving a message by a Member that all previous message sent by
this member have been properly received.

Additionally the DS is expected to be able, depending on the expectations
of the Group, to send acknowledgments (ACKs or NACKs) and to exercise
retries when a message has not been delivered properly to a client.
Meanwhile, it is possible for multiple reasons that messages can be
indefinitely hold by an dishonest or malfunctionning DS, a network loss, etc.
In this Denial Of Service scenario, the receiver has no knowledge
of this situation until it tries sending a message to the Group
and receives no valid acknowledgment.

It is typically expected that servers that are not trusted regarding
correct delivery will not be trusted regarding the group membership
information either.

### Membership knowledge

A particularly important security constraint in that an adversary
must not be able to gain access to information about the identity of
group members and the number of clients.

To prevent that from happening, the MLS threat model considers the case
of a corrupted or untrusted DS that would leak all information at its
disposal. Hence, in this Untrusted DS scenario, MLS will enforce that
the DS MUST NOT be aware these informations. While not providing the
DS with this information might be enough in certain scenarios, the
strong threat model of MLS in this scenario provides counter measures
against potential traffic analysis that could be done at the DS level.

### Membership and offline members

Clients that have been offline for a long time or not performing
mandatory security operations will affect the security of the
group in different ways depending on the amount of trust given to the DS.

In the scenario where the DS is Trusted, the MLS design ensures that
the protocol provides security against permanently offline members or
devices by signaling to the Members of the Group that one endpoint has
been kicked out of the delivery. This is an absolute requirement to
preserve security properties such as forward secrecy of messages or
post-compromise security.

## Authentication Service


# Threat Model

In order to mitigate several categories of attacks across parts of
the MLS architecture, we assume the attacker to be an active network
attacker. This means an adversary which has complete control over the
network used to communicate between the parties [RFC3552].
This assumption remains valid for communications across multiple
authentication or delivery servers if these have to collaborate
to provide a client with some kind of information.

Additionally, the MLS threat model considers possible compromissions
of both Clients and the Authentication (AS) or Delivery (DS) services. In these case
the protocol provide resilience against multiple scenarios described
in the following sections. Typically, the Delivery Service (DS) will not
be able to inject messages in the group conversation or compromise
the identity of the group members.
Depending on the level of trust given by the group to the DS, the
MLS protocol will provide the group, the AS and the DS with specific
sets of security properties. Different scenarios are considered in this
architecture document and are described in subsequent sections of this
document:

1. Client compromise: the client actively forwards secret keys, messages,
   group membership or metadata to the adversary (this dishonest client
   scenario is the only case able to defeat completely the security
   properties provided by MLS). Specific client keys, long term key or
   messages might be compromised, in this scenarios MLS will provide
   limited security.

2. Delivery Service (DS) compromise: the initial keying material delivery
   can provide wrong or adversarial keys the client (Untrusted DS).
   The DS can provide previously correct initial keys that may not be
   up to date anymore when multiple DS are involved (Trusted DS).
   Reliability of in-order delivery or message delivery all-together
   might be compromised for multiple reasons such as networking failure,
   active network attacks... Additionally, there is a scenario where a
   compromised DS could potentially leak group membership if it has this
   knowledge (Untrusted and Trusted DS).

3. Authentication service (AS) compromise: a compromised AS could
   provide incorrect or adversarial identities to clients.
   [TODO: Expand on compromised authentication service]


# System Requirements

## Functional Requirements

### Asynchronous Delivery

Messaging systems that implement MLS must provide a transport
layer for delivering messages asynchronously.

This transport layer must also support delivery ACKs and NACKs
and a mechanism for retrying message delivery.

### Asynchronous Key Update

Clients participating in conversations protected using MLS must
be able to update shared keys asynchronously.

### Recovery After State Loss

Conversation participants whose local MLS state is lost or corrupted
must be able to reinitialize their state and continue participating
in the conversation.

## Message Protection

The trust establishment step of the MLS protocol is followed by a
conversation protection step where encryption is used by clients to
transmit authenticated messages to other clients through the DS.
This ensures that the DS doesn't have access to this Group-private content.
MLS provide security properties such repudiability and unlinkability
additionnally to message secrecy, integrity and authentication
(see below).

### Message Secrecy

Message Secrecy in the context of MLS means that only intended
recipients, currently valid members of the group, should be able to
read the message. A corollary to that statement is that AS
and DS can't read the content of messages sent between Members as
they are not Members of the Group. It is expected from MLS to
optionnally provide additional protections regarding traffic analysis
techniques to reduce the ability of adversaries or a compromised
member of the messaging system to deduce the content of the messages
depending on (for example) their size. One of these protection is
typically padding messages in order to produce ciphertexts of standard
length. While this protection is highly recommended it is not
mandatory as it can be costly in terms of performance for clients
and the MS.

MLS provides additional protection regarding secrecy of past messages
and future messages. These cryptographic security properties are
Perfect Forward Secrecy (PFS) and Post-Compromise Security (PCS).
PFS ensures that access to all encrypted traffic history combined
with an access to all current keying material on clients will not
defeat the secrecy properties of messages older than the oldest key.
Note that this means that clients have the extremely important role
of deleting appropriate keys as soon as they have been used with
the expected message, otherwise the secrecy of the messages and the
security for MLS is considerably weakened.

### Message Authentication and Integrity

Message Integrity and Authentication are properties enforced by MLS.
When the protocol is under attack, it is typically expected by the threat
model that messages will be altered, dropped or substituted by the
adversary. MLS guarantees that under these circumstances an honest
client will not accept one of these scenarios and will reject messages
modified in transit or that have not by successfully authenticated as
a message from the correct Member.

In messaging systems, authentication is a very important part of the
design especially in strong adversarial environnements. This requires
MLS to provide message repudiability and unlinkability properties.
These guarantee that only Members of the group are able to verify
that a message has been sent by a specific Member but will not allow
an external entity having access to all history and keys to link a
message to a specific client, or by extension Member, (Repudiability)
and doesn't allow an external entity to link a specific Member to a
set of specific messages in the conversation (Unlinkability).
(Note that MLS is specifically careful about the case where a Member
of the group is leaking the messages and keys in that scenario.)


### Security of Attachments

While MLS does a separation between messages and attachments, the
protocol does enforce the same security properties for attachments
as it does for messages. In particular attachments benefits from
the same confidentiality and authentication properties.

## Support for Group Messaging

Messaging systems that implement MLS must provide support for
conversations involving 2 or more participants.

### Secrecy After Member Exit

Message secrecy properties must be preserved after any participant
exits the conversation.

## Support for Multiple Devices

### Adding New Devices

## System Resilience

### Forward Secrecy

###  Post-Compromise Secrecy

### Offline/old Devices

## Protection Against Server Misbehavior

### Deterministic Group Membership

### Servers and Post-Compromise Secrecy

### Unauthorized Device Additions


# Security Considerations

TODO

--- back
