---
slug: 33
title: 33/ZHTTP
name: HTTP requests and responses over ZeroMQ
status: stable
editor: Justin Karneges <justin@fanout.io>
---

This document describes a protocol for performing HTTP requests and responses over ZeroMQ. It is partly modeled after the protocols of Mongrel2 and ZeroGW, but with the addition of streaming capability and credits-based flow control.

## License

Copyright (c) 2013 Fanout, Inc.

This Specification is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation; either version 3 of the License, or (at your option) any later version.

This Specification is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with this program; if not, see <http://www.gnu.org/licenses>.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 (see "[Key words for use in RFCs to Indicate Requirement Levels](http://tools.ietf.org/html/rfc2119)").

## Goals

The goals of ZHTTP are:

* Provide an abstract interface for HTTP interactions over ZeroMQ. Transport-level concerns such as connections, HTTP versioning, or transfer-encodings should not be a concern of a user of the protocol.
* Enable streaming of request and response data, with flow control.
* Support long-lived sessions.
* Have an ability to "hand off" a ZHTTP session to another ZeroMQ entity.

## Protocol

### Description

Two arrangements are possible, basic and advanced.

Basic:

* DEALER <-> request/response <-> ROUTER

In basic arrangement, the initiator includes an entire HTTP request in one message and the responder replies with an entire HTTP response in one message. The initiator envelopes the message with its return address, which the responder uses for routing back. This also means the arrangement is compatible with REQ and REP.

Advanced:

* PUSH   -> request  -> PULL
* ROUTER -> request  -> DEALER (recv only)
* SUB    <- response <- PUB

In advanced arrangement, the initiator may send one or more HTTP request fragments and the responder may reply with one or more HTTP response fragments. Credits-based flow control is used to ensure either side does not get overloaded. The first message from the initiator is sent using the PUSH socket. All messages from the responder are sent using the PUB socket. Once the initiator has received at least one message from the responder, it can send further messages using the ROUTER socket. This design enables to the possibility of having multiple indepedent responder workers. The first message goes to an arbitrary worker via PUSH, the worker that receives the message replies with its own address, and then all subsequent messages are sent directly to that same worker via ROUTER. The type of socket used by the responder to receive subsequent packets can be anything so as long as it is compatible with ROUTER, since the responder never actually sends from this socket.

Message format is primarily tnetstrings, with some minor framings depending on socket type.

Basic HTTP request contains the following fields:

* id        - unique id among requests sent by initiator

* method    - http method
* uri       - full uri. e.g. scheme://domain.com/path?query
* headers   - list of items, where each item is a list of two strings: header name, header value
* body      - body content

* user-data - data to be echoed back

The message is encoded as a tnetstring dictionary. The initiator is identified via its socket, which is usually a random value or explicitly set on the socket. An enveloped request is then sent.

Basic HTTP response contains the following fields:

* id        - id of request

* type      - data (omitted) or "error"
* condition - error condition

* code      - http status code
* reason    - http status reason
* headers   - same format as request headers
* body      - body content

* user-data - if user-data was present in request, copy the data here

The message is encoded as a tnetstring dictionary. The responder must read the envelope to determine the initiator's address, and then put a proper envelope on its response when replying back.

Advanced HTTP request contains the following fields:

* from      - address of initiator
* id        - unique id among requests sent by initiator

* type      - data (omitted), "credit", "cancel", "keep-alive", "handoff-start", or "handoff-proceed"
* condition - error condition
* seq       - sequence number of request stream starting at 0

* credits   - offer credits to responder
* more      - means there are more data packets after this one
* stream    - prefer stream mode. if false, responder should reply with a single message
* max-size  - don't accept a response larger than this value (non-stream mode only)

* method    - http method
* uri       - full uri. e.g. scheme://domain.com/path?query
* headers   - list of items, where each item is a list of two strings: header name, header value
* body      - body content

* user-data - data to be echoed back

First message is encoded as a tnetstring dictionary for the PUSH socket. Any subsequent messages are encoded as a tnetstring dictionary and additionally enveloped with the responder's address.

Advanced HTTP response contains the following fields:

* from      - address of responder
* id        - id of request

* type      - data (omitted), "credits", "cancel", "keep-alive", "handoff-start", or "handoff-proceed"
* condition - error condition
* seq       - sequence number of response stream starting at 0

* credits   - offer credits to initiator
* more      - means there are more data packets after this one

* code      - http status code
* reason    - http status reason
* headers   - same format as request headers
* body      - body content

* user-data - if user-data was present in request, copy the data here

The message is encoded as a tnetstring dictionary and prefixed with the initiator's address and a space, e.g. "{initiator-address} {tnetstring}".

Notes:
* the first message from the initiator must be a "data" message
* method, uri, and headers must be present in the first "data" message of the initiator
* code, reason, headers, and body must be present in the first "data" message of the responder
* all messages must be sequenced except for "cancel" type which may be sent and processed out of sequence
* "credits" may be provided in "data" or "credit" types
* "more" may be provided in "data" type only. if not present, then the stream is finished
* "stream" may only be provided in the first message
* if the responder does not support streaming input (detected by the presence of "more" in the first message from the initiator), then the responder may cancel
* if user-data is supplied in a subsequent request message, it replaces the previous value known by the responder

Each side is assumed to start with 0 credits and should not send data unless they believe they have credits. Credits only apply to body data. As a special exception, the initiator may provide body data without having been granted any credits. It should assume it has 0 after this. Additionally, if stream mode was not requested, then it is not necessary to provide credits to the responder. The responder may respond with the entire HTTP response no matter what in that case.

There are some additional fields depending on the use-case:

Inbound-only fields, sent by a webserver:

* peer-address      - remote address of requesting client
* peer-port         - remote port of requesting client

Outbound-only flags, to be sent to an http client worker:

* connect-host      - override host to connect to
* connect-port      - override port to connect to
* ignore-policies   - ignore any rules about what requests are allowed
* ignore-tls-errors - ignore cert of http server

These additional fields only have meaning in the first request message.

Handoff:

It is possible to pass the state of a session from one worker to another via handoff. The way this works is a "handoff-start" message is sent. No further messages may be sent until the other side replies with an expected "handoff-proceed", meaning the other side will also not send any further messages either. At this point, the entity requesting the handoff may move state to another worker. That worker then may resume the session by sending a message to the other side. If the entity is the initiator, then the message should contain an extra field "old-from" set to the address of the previous worker. This is needed because responders identify outstanding sessions by address+id pairs.

If both sides attempt to start a handoff at the same time, then this is a tie and the responder wins. This means if a responder sends "handoff-start" and enters a WaitForHandoffProceed state, it should ignore any "handoff-start" message received while in this state. If an initiator sends "handoff-start" and enters a WaitForHandoffProceed state, it should cease (and possibly queue for the future) its desire to handoff, send a "handoff-proceed", and immediately consider the responder to be the one performing a handoff.

### Reference Implementations

Zurl (see "[Zurl](http://github.com/fanout/zurl)") is the prime preference implementation for ZHTTP. It translates incoming ZHTTP requests into outgoing HTTP requests.
