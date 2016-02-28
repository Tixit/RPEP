Author: **Billy Tetrud**
Date:   **2016-02-26**
Category: **Standards Track**

# The Remote Procedure and Event Protocol

### Abstract

This document defines the Remote Procedure and Event Protocol (RPEP), which is a protocol that provides three messaging modes:

* Fire and Forget
* Request and Response
* Duplex Event Stream

It is intended to connect application components in distributed applications. RPEP is both transport-agnostic and serialization-agnostic, where the transport protocol and serialization format only have to meet some minimum requirements (defined in "Serializations" section).

### 1.  Introduction

##### 1.1.  Background

*This section is non-normative.*

While the WebSocket protocol brings bi-directional real-time connections to the browser, it requires users who want to use WebSocket connections in their applications to define their own semantics on top of it. RPEP provides three of the most common semantics needed in application development so not only can browsers talk to servers, but servers can talk to other servers, devices can communicate with other devices, and (with WebRTC) browsers can talk directly to other browsers, using familiar RPC or event semantics. The WAMP protocol, by comparison, does not support peer to peer communication.

##### 1.2. Protocol Overview

There is always two parties involved in a connection. We will call these parties `Peers`.

For `Fire and Forget`, there are two roles:

* `Sender` - The Peer that fires the message.
* `Receiver` - The Peer that receives the message.

For `Request and Response`, there are two roles:

* `Requester` - The Peer that makes the request.
* `Responder` - The Peer who responds to the Requester's request.

For the `Duplex Event Stream` mode, there are three roles:

* `Initiator` - The Peer is that requests the establishment of the event stream.
* `Confirmer` - The Peer that confirms the establishment of the event stream.
* `Emitter` - The Peer emitting an event to the other Peer over that stream. The Initiator and Confirmer can both act as Emitters once the event stream is established.

##### 1.3 Philosophy

*This section is non-normative.*

RPEP is designed to be performant, reliable, and easy to implement.

This spec embraces the [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single_responsibility_principle), where it enables the most general behaivor, and only makes requirements that are central to the needs of a single purpose - in this case, application level messaging needs. In keeping with this principle, RPEP is agnostic to transport protocol, serialization format, and programming langauge.

Unlike the WAMP protocol, RPEP doesn't define any routing or brokering semantics, and applications are free to use any of the three modes of communication to implement routing of calls and brokering of subscriptions. No requirements or limitations are enforced in that regard.

### 2.  Conformance Requirements

All examples in this specification are non-normative, as are all sections explicitly marked non-normative. Everything else in this specification is normative (including diagrams).

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 [RFC2119].

Requirements phrased in the imperative as part of algorithms (such as "strip any leading space characters" or "return false and abort these steps") are to be interpreted with the meaning of the key word ("MUST", "SHOULD", "MAY", etc.) used in introducing the algorithm.

Conformance requirements phrased as algorithms or specific steps MAY be implemented in any manner, so long as the end result is equivalent.

##### 2.1.  Terminology and Other Conventions

Key terms such as named algorithms or definitions are indicated like `This` when they first occur, and are capitalized throughout the text.

### 3.  Peers and Roles

An RPEP connection connects two Peers, a `Client` and `Service`. A Service is the Peer that listens for a connection, and the Client is the Peer that connects to the service. Each RPEP Peer MUST implement one role, and MAY implement more roles. A Peer may implement any combination of the Roles:

* Sender
* Receiver
* Requester
* Responder
* Initiator
* Emitter

### 4. Building Blocks

##### 4.1.  Serializations

The minimum requirements for a serialization format are that it must provide the following types:

* "integer" (non-negative)
* "string"
* "list"

RPEP itself only uses the above types ( e.g. it does't use the JSON data types "number" (non-integer) nor "null").  The application payloads transmitted by RPEP (e.g. in call arguments or event payloads) may use other types the chosen concrete serialization format supports.

Note: JSON, MessagePack, and BSON all fit these critera.

##### 4.2.  Transports

RPEP requires a transport with the following characteristics:

* message-oriented
* reliable
* ordered
* bi-directional (full-duplex)

Note: TCP and WebSockets both fit these criteria.

###### 4.2.1.  Transport and Session Lifetime

RPEP implementations are NOT required to tie the lifetime of the underlying transport connection for an RPEP connection to that of an RPEP session, i.e. establishing a new transport-layer connection as part of each new session establishment. An implementation may therefore choose to allow re-use of a transport connection, allowing subsequent RPEP sessions to be established using the same transport connection.

### 5.0.  Messages

Each RPEP message is a "list" with either an id, a commandName, or both, and then some payload data. The types of messages are:

* Fire and Forget: `[commandName, data]`
* Request: `[commandName, id, data]`
* Success Response: `[id, data]`
* Error Response: `[id, "e", data]`
* Initiation: `[commandName, id, data]`
* Emission: `[id, eventName, data]`

where

* `id` is an integer ID as defined in section 5.1
* `commandName` is any "string" value
* `data` is any valid value the serialization format can represent. This value may be omitted in any message type.

##### 5.1.  Structure

The application payload `data` (that is call arguments, call results, event payload etc) is always at the end of the message element list.  This allows the message receiver to choose not to parse the application payload.  This can improve efficiency and performance. For example, a server may route the call to another service that handles the payload.

###### 5.1.1  IDs

RPEP Peers need to identify the following ephemeral entities:

* Requests
* Event Streams

These are identified in RPEP using IDs that are integers between (inclusive) *0* and *2^53* (9007199254740992). Services must only create even number IDs when sending Requests and Event Stream Initiation messages, and Clients must only use odd number IDs (so that the namespaces don't collide). IDs SHOULD be incremented by 2 beginning with 0 (for a Service) or 1 (for a Client). Requests and Event streams use the same ID space, and so a unique ID should be used across requests and event streams.

The reason to choose the specific upper bound is that 2^53 is the largest integer such that this integer and _all_ (positive) smaller integers can be represented exactly in IEEE-754 doubles. Some languages (e.g.  JavaScript) use doubles as their sole number type.  Most languages do have signed and unsigned 64-bit integer types that both can hold any value from the specified range.

If incrimenting IDs by 2 is the method used, some handling of the case where the ID reaches 2^53 should be done. For example, the ID could wrap back around to 0 (or 1) but skip over IDs that are still currently in use by some open request or event stream.

###### 5.1.2  Command Names

Like IDs, the namespace of command names (the `commandName` argument) is shared among all three messaging modes. Registering the same command name for more than one mode is not allowed.

##### 5.2. Special Messages

There are a couple reserved message sub-formats that have special meanings:

* `["e", [errorMessage, errorData]` - This indicates that some error happened related to either a received Fire and Forget message or at a global level (*Request-Response errors and Event Stream errors should not use this form to report errors*).
* `[id, "e", [errorMessage, errorData]]` - This indicates that some error happened related to a received Request or in an Event Stream (which one depends on what the `id` identifies).
* `[id, "end", endData]` - This indicates that an Event Stream is completed. No more responses will be received and no more events should be emitted.
* `["close", closeData]` - This indicates that the Sender is going to close the connection. This event is optional for certain transports (see the section "Connection Closure" for more details).

In the above:

* `errorMessage` should be a "string" type.
* `errorData`, `endData`, and `closeData` may be any valid value that the serialization format can represent or may be omitted.

###### 5.2.1 Special Errors

There are a couple standard errors an implementation must send (the `errorMessage` argument):

* "idNotFound" - This indicates that the `id` in a sent Response message or Event Stream Emission doesn't exist.
* "invalidId" - This indicates that the `id` sent is not valid in some way. The errorData should indicate further what was wrong with the Id (whether it was odd when it should be even, or if it was out of bounds, etc).
* "invalidMessage" - This indicates that the message could not be parsed as one of the five valid types of messages defined in section 5.
* "noSuchFireAndForgetEndpoint" - This indicates that the `commandName` is not available to receive a Fire and Forget message. *This error must still be sent if the `commandName` is available for a different messaging mode*.
* "noSuchRequestOrEventEndpoint" - This indicates that the `commandName` is not available to respond to a Request message nor as an Event Stream. *This error must still be sent if the `commandName` is available for the Fire and Forget mode*.

##### 5.3 Messaging Modes

As described above, there are three messaging modes. Here I'll describe the protocol for each mode.

###### 5.3.1 Fire and Forget

* Message: `[commandName, data]`

Fire and Forget is as simple as sending a single message in the above format.

              ,------.           ,--------.
              |Sender|           |Receiver|
              `---+--'           `---+----'
                  |      Message     |
                  | ----------------->

###### 5.3.2 Request and Response

* Request: `[commandName, id, data]`
* Response: `[id, data]` or `[id, "e", data]`

Request and response involves two messages before being considered complete. The `id` in the response must match the `id` in the request. One and only one response must be sent.

             ,---------.         ,---------.
             |Requester|         |Responder|
             `----+----'         `---+-----'
                  |      Request     |
                  | ----------------->
                  |                  |
                  |      Response    |
                  <------------------|

###### 5.3.3 Event Stream

* Initiation: `[commandName, id, data]`
* Emission: `[id, eventName, data]`

An Event Stream requires at least two messages, but most likely more than two. It begins with a single Initiation message, may have any number of Emission messages, and must end with two "end" Emission messages (one from each Peer). The `id` in each Emission must match the `id` in the original Initiation message. Either Peer may send Emissions messages and either Peer may send the final "end" Emission message.

             ,---------.         ,---------.
             |Initiator|         |Confirmer|
             `----+----'         `---+-----'
                  |      Initiation     |
                  | -------------------->
                  |                     |
                  |       Emission      |
                  ?- - - - - - - - - - -?
                  |                     |
                  .                     .
                  .         ...         .
                  .                     .
                  |                     |
                  |   "end" Emission    |
                  ?A--------------------?B
                  |                     |
                  | "end" Confirmation  |
                  ?B--------------------?A

         In the above diagram, the qusetion marks (?)
         indicate that the message may originate from
         either end. The reversal of A and B indicate
         that whoever receives an "end" message first
         should then also send one.

Note that the reason both sides must emit an "end" message is because valid Emission messages may be already in transit when one Peer sends the first "end" message.

##### 5.4 Identifying Messaging Mode

While there is no explicit named mode in any given message, the type of mode can always be determined either by the `commandName` or by the `id`. For messages with a `commandName`, the registered endpoints to handle that command must keep track of what mode the command was registered as. For messages using an `id` from a previous message that had a `commandName`, the impelmentor must somehow keep track of the mode the original message was using.

### 6. Examples

The following examples use JSON as the serialization format.

##### 6.1 Fire and Forget Examples

Normal Fire and Forget message:

`["log", "User used the drag and drop feature"]`

Fire and Forget error:

`["error", ["Browser Error: Cannot access property 'zalgo' of undefined", {location:"depths.js", stack: "at Error (native)\n    at handle (https://localho ..."}]]`

##### 6.2 Request and Response Examples

Successful request:

1. Request: `["fetch", 0, {"resource":'zombie', "databaseId": '3290f2j8'}]`
2. Response: `[0, [{"type":"exploder"}, {"type":"slowWalker"}, {"type":"runner"}]]`

Errored Request:

1. Request: `["send", 2, {"type":"skype", "message":"OMG I'm on the skypz!"}]`
2. Response: `[2, "e", ["unknownError", {"details": "No you're not"}]]`

##### 6.3 Event Stream Examples

Search Example (mostly 1 way communication):

1. Initiation: `["search", 4, {"query": {"make":"Acura"} }]`
2. Confirmer Emission: `[4, "result", {"model":"Legend", "year": 1986}]`
3. Confirmer Emission: `[4, "result", {"model":"Legend", "year": 1987}]`
4. Confirmer Emission: `[4, "result", {"model":"Legend", "year": 1990}]`
5. Confirmer Emission: `[4, "e", ["Error retrieving data", {details: "Unknown error retriving year of vehicle 3717"}"}]]`
6. Confirmer Emission: `[4, "result", {"model":"Integra", "year": 1987}]`
7. Confirmer Emission: `[4, "result", {"model":"Integra", "year": 1988}]`
8. Confirmer Emission: `[4, "result", {"model":"NSX", "year": 1991}]`
9. Confirmer Emission: `[4, "end"]`
10. Initiator Emission: `[4, "end"]`

Canceled Search Example:

1. Initiation: `["search", 6, {"query": {"make":"Mazda"} }]`
2. Confirmer Emission: `[6, "result", {"model":"Mazda3", "year": 2004}]`
3. Confirmer Emission: `[6, "result", {"model":"Mazda3", "year": 2006}]`
4. Confirmer Emission: `[6, "result", {"model":"Verisa", "year": 2004}]`
5. Initiator Emission: `[6, "end"]`
6. Confirmer Emission: `[6, "end"]`

Distributed Computing Example (more bi-directionality):

1. Initiation: `["workerAvailable", 8]`
2. Confirmer Emission: `[8, "hash", {"algorithm":"v5"}]`
3. Confirmer Emission: `[8, "fragment", "This document defines the Remote Procedure and E"]`
4. Confirmer Emission: `[8, "fragment", "vent Protocol (RPEP), which is a protocol that p"]`
5. Confirmer Emission: `[8, "fragment", "rotocol that provides three messaging modes:\n\n* "]`
6. Confirmer Emission: `[8, "fragment", "Fire and Forget\n\n* Request and Response\n\n* Duple"]`
7. Confirmer Emission: `[8, "fragment", "x Event Stream"]`
8. Confirmer Emission: `[8, "finish"]`
9. Initiator Emission: `[8, "result", "32rf2893f7hf"]`
10. Confirmer Emission: `[8, "hash", {"algorithm":"v5"}]
11. Confirmer Emission: `[8, "fragment", "It is intended to connect application component"]`
12. Confirmer Emission: `[8, "fragment", "s in distributed applications. RPEP is both tra"]`
13. Confirmer Emission: `[8, "fragment", "nsport-agnostic and serialization-agnostic, whe"]`
9. Initiator Emission: `[8, "unavailable", {"completionOK":true}]`
14. Confirmer Emission: `[8, "fragment", "re the transport protocol and serialization form"]`
15. Confirmer Emission: `[8, "fragment", "at only have to meet some minimum requirements ("]`
16. Confirmer Emission: `[8, "fragment", "defined in "Serializations" section)."]`
17. Confirmer Emission: `[8, "finish"]`
18. Initiator Emission: `[8, "result", "fg99438hga7d"]`
19. Initiator Emission: `[8, "end"]`
20. Confirmer Emission: `[8, "end"]`

### 7. Ordering Guarantees

Message order must be guaranteed for a given pair of Peers. If *Peer A* sends *Message A* to *Peer B* before sending *Message B*, *Message A* must have its handling *initiated* before *Message B* can have its handling initiated.

For example, If *Peer A* has an open Event Stream endpoint for *Command 1* and a Request-Response endpoint for *Command 2*, and *Peer B* first emits an *event message (1)* on the *Command 1* Event Stream and then sends *request message (2)* for *Command 2*, then Peer A will first receive and begin the handling of the *event message (1)* before begginning the handling of *request message (2)*. This also holds for any combination of messaging modes and when *Command 1* and *Command 2* are identical.

There are no guarantees on the order of Responses or Events in relation to when their Requests or related Events were sent, since the execution of different messages may run at different speeds. A first message might trigger an expensive, long-running computation, whereas a second, subsequent message might finish immediately.

### 8. Connection Closure

Implementations must provide some way for a peer to indicate that the connection will be closed. One of two ways of doing this must be available:

* Some transport-protocol-level message, or
* An RPEP "close" Fire and Forget message of the form `["close", closeData]`

Implementations are not required to use the implemented way to inform the other Peer of connection closure, ie it is allowed to drop a connection without informing the other Peer. But to reitterate, a method of closure that does involve informing the other Peer must be implemented.

### 9.  Security Model

RPEP deliberately does not require or specify any kind of encryption, integrity validation, or authentication. RPEP transports may optionally provide guarantees of these kinds.

### 10. Copyright Notice - The MIT License (MIT)

Copyright (c) 2016 Billy Tetrud

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

### 11.  Contributors

* Billy Tetrud

### 12.  Acknowledgements

   RPEP was developed partly in response to Web Application Messaging Protocol. Thanks to them for the

### 13.  References

##### 13.1.  Normative References

##### 13.2.  Informative References

   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119, DOI 10.17487/
              RFC2119, March 1997,
              <http://www.rfc-editor.org/info/rfc2119>.


### 14. Authors' Addresses

   Billy Tetrud
   Email: billy.tetrud at gmail.com
