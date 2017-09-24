Category: **Standards Track**
Version: **1.2.0**

# The Remote Procedure and Event Protocol

### Abstract

This document defines the Remote Procedure and Event Protocol (RPEP), which is a protocol that provides three messaging modes:

* Fire and Forget
* Request and Response
* Duplex Event Stream

It is intended to connect application components in distributed applications. RPEP is both transport-agnostic and serialization-agnostic, where the transport protocol and serialization format only have to meet some minimum requirements (defined in the "Serializations" and "Transports" sections).

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE *generated with [DocToc](https://github.com/thlorenz/doctoc)* -->

- [1.  Introduction](#1--introduction)
    - [1.1.  Background](#11--background)
    - [1.2 Philosophy](#12-philosophy)
- [2.  Conformance Requirements](#2--conformance-requirements)
    - [2.1.  Terminology and Other Conventions](#21--terminology-and-other-conventions)
- [3.  Peers and Roles](#3--peers-and-roles)
- [4. Building Blocks](#4-building-blocks)
    - [4.1.  Serializations](#41--serializations)
    - [4.2.  Transports](#42--transports)
      - [4.2.1.  Transport and Session Lifetime](#421--transport-and-session-lifetime)
- [5.0.  Messages](#50--messages)
    - [5.1.  Structure](#51--structure)
      - [5.1.1  IDs](#511--ids)
      - [5.1.2  Command Names](#512--command-names)
    - [5.2. Special Messages](#52-special-messages)
      - [5.2.1 Special Errors](#521-special-errors)
    - [5.3 Messaging Modes](#53-messaging-modes)
      - [5.3.1 Fire and Forget](#531-fire-and-forget)
      - [5.3.2 Request and Response](#532-request-and-response)
      - [5.3.3 Event Stream](#533-event-stream)
    - [5.4 Identifying Messaging Mode](#54-identifying-messaging-mode)
- [6. Ordering Guarantees](#6-ordering-guarantees)
- [7. Connection Establishment and Closure](#7-connection-establishment-and-closure)
- [8.  Security Model](#8--security-model)
- [9. Examples](#9-examples)
    - [9.1 Fire and Forget Examples](#91-fire-and-forget-examples)
    - [9.2 Request and Response Examples](#92-request-and-response-examples)
    - [9.3 Event Stream Examples](#93-event-stream-examples)
- [10. Copyright Notice - The MIT License (MIT)](#10-copyright-notice---the-mit-license-mit)
- [11. Change Log](#11-change-log)
- [12.  References](#12--references)
    - [12.1.  Normative References](#121--normative-references)
    - [12.2.  Informative References](#122--informative-references)
- [13.  Contributors](#13--contributors)
- [14.  Acknowledgements](#14--acknowledgements)
- [15. Authors' Addresses](#15-authors-addresses)
- [16. How to Contribute!](#16-how-to-contribute)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

### 1.  Introduction

##### 1.1.  Background

*This section is non-normative.*

While the WebSocket protocol brings bi-directional real-time connections to the browser, it requires users who want to use WebSocket connections in their applications to define their own semantics on top of it. RPEP provides three of the most common semantics needed in application development so not only can browsers talk to servers, but servers can talk to other servers, devices can communicate with other devices, and (with WebRTC) browsers can talk directly to other browsers, using familiar RPC or event semantics. RPEP can also handle unordered messages in UDP-style. The WAMP protocol, by comparison, does not support peer to peer communication or unordered messages.

##### 1.2 Philosophy

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

##### 3.1. Peers

There is always two parties involved in a connection. We will call these parties `Peers`. An RPEP connection connects two Peers, a `Client` and `Service`. A Service is the Peer that listens for a connection, and the Client is the Peer that connects to the service. Each RPEP Peer MUST implement one role, and MAY implement any role.

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
* bi-directional (full-duplex)

Note: WebSockets fit these criteria.

Protocols that don't fit all these criteria must be augmented to fulfill the missing pieces if they are to be used as an RPEP transport. For example, since TCP operates on string streams rather than being message-oriented, it would be necessary to implement some way of chunking the stream into distinct messages at a lower level than RPEP if TCP were to be used for RPEP. Similarly, while UDP is message-oriented and bi-directional, it isn't reliable, and so those facilities would need to be added on top of it before the resulting protocol could be used as an RPEP transport.

###### 4.2.1.  Transport and Session Lifetime

RPEP implementations are NOT required to tie the lifetime of the underlying transport connection for an RPEP connection to that of an RPEP session, i.e. establishing a new transport-layer connection as part of each new session establishment. An implementation may therefore choose to allow re-use of a transport connection, allowing subsequent RPEP sessions to be established using the same transport connection.

### 5.0.  Messages

Each RPEP message is a "list" with either an id, a commandName, or both, and then some payload data. The types of messages are:

* Fire and Forget: `[commandName, data]`
* Request or Event Stream Initiation: `[commandName, id, data]`
* Success Response: `[id, data]`
* Event Stream Emission: `[id, eventName, data]`

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

These are identified in RPEP using IDs that are integers between (inclusive) *0* and *2^53* (9007199254740992). Services must only create even number IDs when sending Requests and Event Stream Initiation messages, and Clients must only use odd number IDs (so that the namespaces don't collide). IDs MUST be incremented by 2 beginning with 0 (for a Service) or 1 (for a Client). Requests and Event streams use the same ID space, and so a unique ID should be used across requests and event streams.

The reason to choose the specific upper bound is that 2^53 is the largest integer such that this integer and _all_ (positive) smaller integers can be represented exactly in IEEE-754 doubles. Some languages (e.g.  JavaScript) use doubles as their sole number type.  Most languages do have signed and unsigned 64-bit integer types that both can hold any value from the specified range.

Some handling of the case where the ID reaches 2^53 should be done. For example, the ID could wrap back around to 0 (or 1) but skip over IDs that are still currently in use by some open request or event stream. When this happens, an ID discontinuity message must be sent. 

###### 5.1.2  Command Names

Like IDs, the namespace of command names (the `commandName` argument) is shared among all three messaging modes. Registering the same command name for more than one mode is not allowed.

##### 5.2. Special Messages

There are a couple reserved message sub-formats that have special meanings:

* Error Response/Event: `[id, "e", [errorMessage, errorData]]`
* Global Error: `["e", [errorMessage, errorData]]`
* Event Stream Ended: `[id, "end", endData]` - This indicates that an Event Stream is completed. No more responses will be received and no more events should be emitted. After an "end" event is received, the Peer that sent that event Emission must not emit any more messages on that stream.
* Event Stream - Enable order data: `[id, "order", yesNo]`
* Connection Established: `["open", openData]` - This indicates that the Service has established the connection requested by the Client. This command is optional for certain transports (see the section ["Connection Establishment and Closure"](#7-connection-establishment-and-closure))
* Impending Connection Closure: `["close", closeData]` - This indicates that the Sender is going to close the connection. This command is optional for certain transports (see the section ["Connection Establishment and Closure"](#7-connection-establishment-and-closure) for more details).
* ID discontinuity message: ["idDiscontinuity", prevId, nextId] - This indicates that some IDs can't be used, and describes which IDs those are and where the IDs will continue from. `prevId` is the last usable ID, and `nextId` is the next usable ID. One use-case for this is if the maximum ID value is reached. The primary purpose of this is to ensure the ability to order request-response messages if desired. 

In the above:

* `errorMessage` must be a "string" type.
* `errorData`, `endData`, and `closeData` may be any valid value that the serialization format can represent or may be omitted.
* `yesNo` should either be 1 or 0 (for on or off respectively)
* `prevId` and `nextId` must be an integer type. 

###### 5.2.1 Special Errors

There are a couple standard errors an implementation must send (the `errorMessage` argument). Errors that indicate a failure of the RPEP implementation of the other Peer are prefixed with 'rpep' to distinguish them from application errors.

* "rpepIdNotFound" - This indicates that the `id` in a sent Response message or Event Stream Emission doesn't exist.
* "rpepInvalidId" - This indicates that the `id` sent is not valid in some way. The errorData should indicate further what was wrong with the Id (whether it was odd when it should be even, or if it was out of bounds, etc).
* "invalidMessage" - This indicates that the message could not be parsed as one of the five valid types of messages defined in section 5.
* "noSuchCommand" - This indicates that the `commandName` is not available in any mode.

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

An Event Stream requires at least two messages, but most likely involves more than that. It begins with a single Initiation message, may have any number of Emission messages, and must end with at least one "end" Emission message. The `id` in each Emission must match the `id` in the original Initiation message. Either Peer may send Emissions messages and both Peers must send the final "end" Emission message. After sending the "end" message, that sending Peer must NOT send any more messages. After sending the "end" message *and* receiving an "end" message, subsequent events that come in on that event stream MUST return a global-level "rpepIdNotFound" error. If the sender of an "end" event might receive valid events after sending that "end" event, the user of the implementation is strongly advised to require both ends to send "end" events. 

             ,---------.         ,---------.
             |Initiator|         |Confirmer|
             `----+----'         `------+--'
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
                  ?---------------------?
                  | "end" Confirmation  |
                  ?---------------------?

         In the above diagram, the qusetion marks (?)
         indicate that the message may originate from
         either end.

##### 5.4 Identifying Messaging Mode

While there is no explicit named mode in any given message, the type of mode can always be determined either by the `commandName` or by the `id`. For messages with a `commandName`, the registered endpoints to handle that command must keep track of what mode the command was registered as. For messages using an `id` from a previous message that had a `commandName`, the impelmentor must somehow keep track of the mode the original message was using.

The following procedure can be used to identify what kind of message has been received:

- 1. If the first value is a "string" type, the message is either a Fire and Forget, a Request, or an Event Stream Initiation.
  - 1.1. Check the commandName for the mode it was registered as.
  - 1.2. If the commandName isn't registered, send a Fire and Forget error with the error message "noSuchCommand"
- 2. Otherwise, if the first value is an "integer" type, the message is either a Response or an event Emission message.
  - 2.1. Check the id for its mode and other information as to how to handle it. If the id can't be found, send an "rpepIdNotFound" error response.
  - 2.2.1. Then, if it is a Response and there are 3 top-level values in the message, it is an error Response. Otherwise it is a success Response.
  - 2.3.2. Or, if it is an Event, if ordering data is turned on, an extra ordering ID should be expected after the message ID.
- 3. If the first value is neither a "string" type nor an "integer" type, send a Fire and Forget error with the error message "invalidMessage"

### 6. Ordering Guarantees

Message order is not guaranteed by RPEP for a given pair of Peers. Implementations may provide a way for users to order Request-Response messages and Event messages, but not Fire-Forget messages, not even to selectively ensure order based on the content of a message.  

There are no guarantees on the order of Responses or Events in relation to when their Requests or related Events were sent, since the execution of different messages may run at different speeds. A first message might trigger an expensive, long-running computation, whereas a second, subsequent message might finish immediately.

### 7. Connection Establishment and Closure

Implementations must provide some way for a peer to indicate that a connection has been established and that the connection will be closed. One of two ways of doing this must be available:

* Some transport-protocol-level message, or
* An RPEP "open" or "close" Fire and Forget message as described above.

Implementations are required to use one of these ways of informing the other Peer of connection establishment. Implementations are, on the other hand, NOT required to use one of those ways to inform the other Peer of connection closure, ie a Peer is allowed to drop a connection without informing the other Peer. But to reiterate, an implementation must implement a method of closure that *does* involve informing the other Peer must be implemented, even if the user of the implementation chooses not to use that method.

### 8.  Security Model

RPEP deliberately does not require or specify any kind of encryption, integrity validation, or authentication. RPEP transports may optionally provide guarantees of these kinds.

### 9. Examples

The following examples use JSON as the serialization format.

##### 9.1 Fire and Forget Examples

Normal Fire and Forget message:

`["log", "User used the drag and drop feature"]`

Fire and Forget error:

`["error", ["Browser Error: Cannot access property 'zalgo' of undefined", {location:"depths.js", stack: "at Error (native)\n    at handle (https://localho ..."}]]`

##### 9.2 Request and Response Examples

Successful request:

1. Request: `["fetch", 0, {"resource":'zombie', "databaseId": '3290f2j8'}]`
2. Response: `[0, [{"type":"exploder"}, {"type":"slowWalker"}, {"type":"runner"}]]`

Errored Request:

1. Request: `["send", 2, {"type":"skype", "message":"OMG I'm on the skypz!"}]`
2. Response: `[2, "e", ["unknownError", {"details": "No you're not"}]]`

##### 9.3 Event Stream Examples

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

Canceled Progressive Command Example:

1. Initiation: `["initialize", 6, "spaceship"]`
2. Confirmer Emission: `[6, "progress", {"percent":0.05, "message": "Checked out backup flight systems"}]`
3. Confirmer Emission: `[6, "result", {"percent":0.07, "message": "Activated and tested navigational systems"}]`
4. Confirmer Emission: `[6, "result", {"percent":0.12, "message": "Completed preparation to load power reactant storage and distribution system"}]`
5. Confirmer Emission: `[6, "result", {"percent":0.15, "message": "Finished loading cryogenic propellants into orbiter's PRSD system"}]`
6. Confirmer Emission: `[6, "result", {"percent":0.2, "message": "Purged external tank nose-cone"}]`
7. Initiator Emission: `[6, "abort"]`
8. Confirmer Emission: `[6, "aborting", {"percent":30, "message": "Control alerted to abort"}]`
9. Confirmer Emission: `[6, "aborting", {"percent":60, "message": "Unloaded propellants"}]`
10. Confirmer Emission: `[6, "aborting", {"percent":100, "message": "Deactivated navigational systems"}]`
12. Confirmer Emission: `[6, "end"]`
13. Initiator Emission: `[4, "end"]`

Distributed Computing Example (more bi-directionality):

1. Initiation: `["workerAvailable", 8]`
2. Confirmer Emission: `[8, "hash", {"algorithm":"v5"}]`
3. Confirmer Emission: `[8, "fragment", "This document defines the Remote Procedure and E"]`
4. Confirmer Emission: `[8, "fragment", "vent Protocol (RPEP), which is a protocol that p"]`
5. Confirmer Emission: `[8, "fragment", "rotocol that provides three messaging modes:\n\n* "]`
6. Confirmer Emission: `[8, "fragment", "Fire and Forget\n\n* Request and Response\n\n* Duple"]`
7. Initiator Emission: `[8, "backPressure"]`
8. Initiator Emission: `[8, "pressureRelieved"]`
9. Confirmer Emission: `[8, "fragment", "x Event Stream\n\nIt is intended to connect applicatio"]`
10. Confirmer Emission: `[8, "fragment", "n components in distributed applications. "]`
11. Confirmer Emission: `[8, "finish"]`
12. Initiator Emission: `[8, "result", "32rf2893f7hf"]`
13. Confirmer Emission: `[8, "hash", {"algorithm":"v5"}]`
14. Confirmer Emission: `[8, "fragment", "RPEP is both transport-agnostic and serializatio"]`
15. Confirmer Emission: `[8, "fragment", "n-agnostic, where the transport protocol and ser"]`
16. Initiator Emission: `[8, "unavailable", {"completionOK":true}]`
17. Confirmer Emission: `[8, "fragment", "ialization format only have to meet some minimum"]`
18. Confirmer Emission: `[8, "fragment", " requirements (defined in "Serializations" secti"]`
19. Confirmer Emission: `[8, "fragment", "on)."]`
20. Confirmer Emission: `[8, "finish"]`
21. Initiator Emission: `[8, "result", "fg99438hga7d"]`
22. Initiator Emission: `[8, "end"]`
23. Confirmer Emission: `[8, "end"]`

A similar example to the previous Distributed Computing Example except with ordering-data enabled:

1. Initiation: `["workerAvailable", 8]`          
2. Initiator Enable ordering: `[8, "order", 1]`
2. Confirmer Emission: `[8, 0, "hash", {"algorithm":"v5"}]`
3. Confirmer Emission: `[8, 1, "fragment", "This document defines the Remote Procedure and E"]`
4. Confirmer Emission: `[8, 2, "fragment", "vent Protocol (RPEP), which is a protocol that p"]`
5. Confirmer Emission: `[8, 3, "fragment", "rotocol that provides three messaging modes:\n\n* "]`
6. Confirmer Emission: `[8, 4, "fragment", "Fire and Forget\n\n* Request and Response\n\n* Duple"]`
7. Initiator Emission: `[8, "backPressure"]`
8. Initiator Emission: `[8, "pressureRelieved"]`
9. Confirmer Emission: `[8, 5, "fragment", "x Event Stream\n\nIt is intended to connect applicatio"]`
10. Confirmer Emission: `[8, 6, "fragment", "n components in distributed applications. "]`
11. Confirmer Emission: `[8, 7, "finish"]`
12. Initiator Emission: `[8, "result", "32rf2893f7hf"]`
13. Confirmer Emission: `[8, 8, "hash", {"algorithm":"v5"}]`
14. Confirmer Emission: `[8, 9, "fragment", "RPEP is both transport-agnostic and serializatio"]`
15. Confirmer Emission: `[8, 10, "fragment", "n-agnostic, where the transport protocol and ser"]`
16. Initiator Emission: `[8, "unavailable", {"completionOK":true}]`
17. Confirmer Emission: `[8, 11, "fragment", "ialization format only have to meet some minimum"]`
18. Confirmer Emission: `[8, 12, "fragment", " requirements (defined in "Serializations" secti"]`
19. Confirmer Emission: `[8, 13, "fragment", "on)."]`
20. Confirmer Emission: `[8, 14, "finish"]`
21. Initiator Emission: `[8, "result", "fg99438hga7d"]`
22. Initiator Emission: `[8, "end"]`
23. Confirmer Emission: `[8, 15, "end"]`

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

### 11. Change Log

* 1.0.2 - 2017-03-04 - Removed ordering requirement, and adding optional order number for event messages. 
* 1.0.1 - 2016-02-28 - Removed second "end" message requirement for event streams
* 1.0.0 - 2016-02-26 - Created

### 12.  References

##### 12.1.  Normative References

##### 12.2.  Informative References

   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119, DOI 10.17487/
              RFC2119, March 1997,
              <http://www.rfc-editor.org/info/rfc2119>.

### 13.  Contributors

* Billy Tetrud

### 14.  Acknowledgements

RPEP was developed partly in response to Web Application Messaging Protocol. Thanks to them for the inspiration and motivation.

### 15. Authors' Addresses

   Billy Tetrud
   Email: billy.tetrud at gmail.com

### 16. How to Contribute!

Feel free to ask questions or discuss protocol changes in github issues, or at https://gitter.im/Tixit/RPEP-Discussion . If you want to write an implementation, please let us know so we can add your implmentation here.
