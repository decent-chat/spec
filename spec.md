# Decent Messaging Protocol Specification

| Version       | Released        |
|:-------------:| --------------- |
|               | Not yet         |

* [1.](#1-introduction) Introduction
  * [1.1](#11-about) About
  * [1.2](#12-terminology) Terminology
  * [1.3](#13-data-types) Data types
* [2.](#2-http-request-protocol) HTTP request protocol
  * [2.1](#21-content-type) Content-type
  * [2.2](#22-routing) Routing
  * [2.3](#23-error-responses) Error responses
  * [2.4](#24-session-ids-and-authentication) Session IDs and authentication
  * [2.5](#25-permissions-and-roles) Permissions and roles
* [3.](#3-bidirectional-websocket-communication) Bidirectional WebSocket communication
  * [3.1](#31-websocket-protocol) WebSocket protocol
  * [3.2](#32-pingdata-and-pongdata-events) "pingdata" and "pongdata" events
  * [3.3](#33-useronline-and-useroffline-events) "user/online" and "user/offline" events
* [4.](#4-api-endpoints-and-events) API endpoints and events
  * [4.1](#41-miscellaneous) Miscellaneous
  * [4.2](#42-user-related) User-related
  * [4.3](#43-role-related) Role-related
  * [4.4](#44-message-related) Message-related
  * [4.5](#45-channel-related) Channel-related
  * [4.6](#46-emote-related) Emote-related
  * [4.7](#47-session-related) Session-related

## 1. Introduction

### 1.1 About

Decent is an open-source messaging platform for the modern internet. It is
inspired by closed-sourced platforms such as Discord and Slack, and intends to
supercede both in terms of features and morality.

A common use-case for Decent is private servers between friends, however it is
flexible enough to allow larger, more varied servers to exist. It can even be
utilised as a platform for multiplayer text-based video games!

Note that this document acts as a preface/specification around
[doc.md](doc.md#api), which pre-documents all WebSocket events and endpoints
that servers MUST implement in full.

### 1.2 Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

Note that "server" refers to a running instance of an implementation of this
specification, and that "client" refers to any software attempting to
communicate with the instance.

A "parameter" refers to either data passed via an HTTP request body or through
an HTTP request's query parameters (that is, data given in the URL itself).

### 1.3 Data types

The data types "undefined", "null", "string", "boolean", "object", and "array"
in this document are to be understood as described in
[ECMA 404](https://www.ecma-international.org/publications/files/ECMA-ST/ECMA-404.pdf).
Also note "int" and "float", which are to be interpreted as integer and
floating-point number respectively.

A type postfixed by a question mark (`?`) should be taken as 'maybe', that is,
it must be either the type defined, not present, or null. For example, valid
`int?`s include `5`, `15`, and `null`.

Types delimeted by a pipe character (`|`) are unions ie. logical OR. For
example, valid `boolean | string`s include `true`, `false`, and `"dog"`.

The `id` type may be any scalar type (`string | int | float | boolean`) however
it is up to the server to conclude which are valid. IDs MUST be unique for the
resource type they are respresenting. Servers SHOULD use a consistent datatype
for IDs; whether that is a random string, an ever-increasing integer, or
something else is an implementation detail.

## 2. HTTP request protocol

### 2.1 Content type

Decent servers MUST support HTTP/1.1 as a form of data transfer. HTTP/1.0 and
HTTP/2.0 support is OPTIONAL. Please refer to
[RFC 2616](https://tools.ietf.org/html/rfc2616) and related specifications.

All HTTP endpoints other than `/` MUST respond with valid JSON as described in
[ECMA 404](https://www.ecma-international.org/publications/files/ECMA-ST/ECMA-404.pdf).

All HTTP endpoints (other than `/api/upload-image`) that expect request bodies
SHOULD NOT accept any content type other than valid JSON. If an invalid body
is provided, the server SHOULD respond with a FAILED error as defined in
section 2.3: Error responses.

If a given request's query parameters or request body contains duplicate keys
the request SHOULD be terminated with a REPEATED_PARAMETERS error. Likewise, if
a request is missing expected parameters the request SHOULD be terminated with
an INCOMPLETE_PARAMETERS error, and if a request's parameters are given but
are of the wrong type, the server should respond with an INVALID_PARAMETER_TYPE
error.

### 2.2 Routing

Valid HTTP requests for routes under `/api/` MUST be responded to as detailed
in section 4.

GET requests for the route (`/`) SHOULD serve sufficient information to connect
to the relevant server, for example a web-based client or instance details.

Other requests MUST be responded to with HTTP status code 404 Not Found. If the
request was for anything under `/api/`, the server SHOULD respond with the
NOT_FOUND error code as described in section 2.3: Error responses.

If an error occurs during the processing of a request, the server SHOULD respond
with the HTTP status code 500 Internal Server Error. See the following section
(2.3: Error responses) for more information.

### 2.3 Error responses

When the processing of an HTTP request for anything under `/api/` fails, the
server SHOULD respond with a valid JSON body with the following form:

    {
      "error": {
        "code": string,
        ...
      }
    }

The server SHOULD NOT respond with anything other than the `error` object as
described here, however extra data under the `error` object MAY be returned.

`error.code` MUST be a string equal to one of the following:

* FAILED
* NO
* NOT_FOUND
* NOT_YOURS
* NOT_ALLOWED
* ALREADY_PERFORMED
* INCOMPLETE_PARAMETERS
* REPEATED_PARAMETERS
* INVALID_PARAMETER_TYPE
* INVALID_SESSION_ID
* INVALID_NAME
* NAME_ALREADY_TAKEN
* SHORT_PASSWORD
* INCORRECT_PASSWORD

The server SHOULD respond with a relevant HTTP status code with the error body,
however it MAY simply use '200 OK' for all responses.

If the server responds with any error to a request, the request MUST have no
action. For example, if any parameters are of the wrong type, the request MUST
be a no-op.

### 2.4 Session IDs and authentication

Session IDs are unique identifier strings that are sent in HTTP requests from
clients to identify the requester as a particular user. They SHOULD NOT be
predictable in any way.

When processing an HTTP request at any endpoint under `/api/`, the server MUST
search for a valid, non-expired session ID using the following three methods:

1. `sessionID` query-string parameter
2. `sessionID` in request body
3. `X-Session-ID` in request headers (MUST be case-insensitive)

If a single request provides a session ID multiple times, the server SHOULD
respond with a REPEATED_PARAMETERS error as defined in section 2.3.

If a session ID is provided but is unknown, expired, or otherwise invalid, the
server SHOULD immediately respond with an INVALID_SESSION_ID error. Otherwise,
the session ID MUST be ignored.

Servers MUST ensure that valid session IDs actually identifies existing,
non-deleted users.

### 2.5 Permissions and roles

Permissions are a way of limiting and granting certain abilities to groups of
users, called 'roles'.

Permissions are stored within an object mapping permission key to
`boolean | undefined`. A single permission in this object can have three states:

| Value       | Meaning                |
| ----------- | ---------------------- |
| `true`      | Granted                |
| `false`     | Denied                 |
| `undefined` | Unset, follow cascade  |

For example, a role that grants the `sendMessages` permission but changes
nothing else would have the permission object:

  {
    "sendMessages": true
  }

There are 2 constant role IDs that servers MUST apply based on whether the
requester provided a session ID or not (see section 2.4):

* `_user`, applied to all logged-in users as a fallback role
* `_everyone`, applied to **all** requesters

Servers MUST NOT allow any of these constant roles to be deleted. However,
they SHOULD be editable (that is, their permissions object and name).

Note: Servers MAY give a particular group of users some kind of automatically
generated 'administrator' role to allow changes to be made by the owner of the
server without manually editing the database or similar.

Individual permissions MUST be computed according to the following algorithm.

1. Let `k` be the permission key we are computing
2. Let `r` be an array of permission objects (from roles) applied to the user
3. Let `v` be the value of the key `k` of the value at index 0 of `r`; ie. `r[0][k]`
4. If `v` is `true`, return `true` (granted)
5. If `v` is `false`, return `false` (denied explicitly)
6. If `v` is `undefined` or the key `k` is not present in the object, continue to step 7
7. Repeat from step 3 for the next value in `r` (`index++`). If we reach the end of `r` without returning, return `false` (denied by default)

If a request requires a permission and the requesting user/guest does not have
the required permission as per the above algorithm, the server MUST respond with
a NOT_ALLOWED error (and, therefore, be a no-op).

The order of `r` (role priority) MUST be sorted in the following way:

* Channel-specific permissions for roles of the user (First.)
* Channel-specific permissions for the `_user` role, if the user is a logged-in member of the server
* Channel-specific permissions for the `_everyone` role
* Server-wide permissions for roles of the user
* Server-wide permissions for the `_user` role (if applicable, as above)
* Server-wide permissions for the `_everyone` role (Last.)

Permissions for roles applied to a user (both server-wide and channel-specific)
MUST be prioritized according to the role prioritization order (see
`PATCH /api/roles/order` in section 4.3). Note that the order of any given
user's `roles` property MUST NOT have any effect on the order roles are
applied when calculating their permissions.

## 3. Bidirectional WebSocket communication

Decent servers MUST support the WebSocket protocol as specified in
[RFC 6455](https://tools.ietf.org/html/rfc6455) for client-server
communication.

The server MUST support WebSocket handshakes at the root (`/`) HTTP endpoint,
but MAY also support connection upgrades at any route.

The server MUST support as many open WebSocket connections at once as possible.

Note that, in this document, "emit" refers to the act of sending data via a
WebSocket to a client. A client response is the act of sending data via a
WebSocket to the server.

### 3.1 WebSocket protocol

All server-sent data down the WebSocket MUST be formatted in JSON with the
following form:

    {
      "evt": string,
      ...
    }

Any extra data MUST be provided within an object `data`, for example:

    {
      "evt": "pingdata",
      "data": {
        "sessionID": "secret"
      }
    }

Valid event (`evt`) strings to be sent from the server are defined later in this
document alongside related HTTP endpoints.

The server MUST NOT send invalid JSON (as described in ECMA 404) down the wire.

Invalid JSON or unknown events received from client sockets SHOULD be ignored.

### 3.2 "pingdata" and "pongdata" events

The server MUST send `{"evt": "pingdata"}` to all connected client sockets
periodically (preferably no longer than every 30 seconds). The server MAY send
this event immediately upon the client socket's connection. Clients SHOULD
respond with a `"pongdata"` event, as described below.

The server MUST acknowledge the `"pongdata"` event when sent from client
sockets. It MUST NOT be sent from the server at any time. This event follows the
following form:

    {
      "evt": "pingdata",
      "data": { "sessionID": string? }
    }

If the server receives a valid `"pingdata"` event from a connected client
socket, it MUST associate the sender socket with the provided `sessionID`. This
data will later be used to determine whether particular events should be sent
to the socket or not. Likewise, if the sessionID is not provided, null, or
otherwise invalid, the server MUST unassociate the sender socket from any user
or session.

Upon receiving a valid `"pongdata"` event where `data.sessionID` is not null or
undefined and is a known session ID (see section 2.4: Session IDs), the server
SHOULD mark the related user as 'online' (see section 3.3).

After a reasonable amount of time after a `"pingdata"` event is emitted, the
server SHOULD check for any users that have not sent `"pongdata"` events tied
to them. These users SHOULD all be marked as 'offline'.

Servers MAY use other criteria to determine user online/offline status.

### 3.3 "user/online" and "user/offline" events

A user that is marked as online should be logged in, however note that this
state is an approximation.

See the previous section, 3.2: "pingdata" and "pongdata" events.

When a user is marked as online and was offline previously, the server SHOULD
emit the `"user/online"` event to all connected client sockets, where
`data.userID` is the ID of the user that has just been marked online. Servers
MUST emit this event using the following form:

    {
      "evt": "user/online",
      "data": {
        "userID": id
      }
    }

When a user is decidedly marked offline and was online previously, the server
SHOULD emit the `"user/online"` event to all connected client sockets. It is
similar to the `"user/online"` event:

    {
      "evt": "user/offline",
      "data": {
        "userID": id
      }
    }

# 4. API endpoints and events

**For more details on the user-facing API of any of the following endpoints or
events, see [doc.md](doc.md#api).** If a server cannot handle or does not
implement a particular endpoint for whatever reason, it MUST respond with an
error (code NO) with the form specified in section 2.3.

Servers SHOULD implement all endpoints and events fully.

## 4.1 Miscellaneous

| Type   | Endpoint / event name                                               |
| ------:| ------------------------------------------------------------------- |
| event  | pingdata                                                            |
| event  | pongdata (from client)                                              |
| GET    | /api/                                                               |
| GET    | /api/settings                                                       |
| PATCH  | /api/settings                                                       |
| event  | server-settings/update                                              |
| POST   | /api/upload-image                                                   |

## 4.2 User-related

| Type   | Endpoint / event name                                               |
| ------:| ------------------------------------------------------------------- |
| event  | user/new                                                            |
| event  | user/delete                                                         |
| event  | user/online                                                         |
| event  | user/offline                                                        |
| event  | user/update                                                         |
| event  | user/mentions/add                                                   |
| event  | user/mentions/remove                                                |
| GET    | /api/users                                                          |
| POST   | /api/users                                                          |
| GET    | /api/users/:id                                                      |
| PATCH  | /api/users/:id                                                      |
| DELETE | /api/users/:id                                                      |
| GET    | /api/users/:id/permissions                                          |
| GET    | /api/users/:id/mentions                                             |
| GET    | /api/users/:userID/channel-permissions/:channelID**                 |
| GET    | /api/username-available/:username                                   |

## 4.3 Role-related

| Type   | Endpoint / event name                                               |
| ------:| ------------------------------------------------------------------- |
| event  | role/new                                                            |
| event  | role/update                                                         |
| event  | role/delete                                                         |
| GET    | /api/roles                                                          |
| POST   | /api/roles                                                          |
| GET    | /api/roles/order                                                    |
| PATCH  | /api/roles/order                                                    |
| GET    | /api/roles/:id                                                      |
| PATCH  | /api/roles/:id                                                      |
| DELETE | /api/roles/:id                                                      |

## 4.4 Message-related

| Type   | Endpoint / event name                                               |
| ------:| ------------------------------------------------------------------- |
| event  | message/new                                                         |
| event  | message/edit                                                        |
| event  | message/delete                                                      |
| POST   | /api/messages                                                       |
| GET    | /api/messages/:id                                                   |
| PATCH  | /api/messages/:id                                                   |
| DELETE | /api/messages/:id                                                   |

## 4.5 Channel-related

| Type   | Endpoint / event name                                               |
| ------:| ------------------------------------------------------------------- |
| event  | channel/new                                                         |
| event  | channel/update                                                      |
| event  | channel/delete                                                      |
| event  | channel/pins/add                                                    |
| event  | channel/pins/remove                                                 |
| GET    | /api/channels                                                       |
| POST   | /api/channels                                                       |
| GET    | /api/channels/:id                                                   |
| PATCH  | /api/channels/:id                                                   |
| DELETE | /api/channels/:id                                                   |
| POST   | /api/channels/:id/mark-read                                         |
| GET    | /api/channels/:id/messages                                          |
| PATCH  | /api/channels/:id/role-permissions                                  |
| GET    | /api/channels/:id/pins                                              |
| POST   | /api/channels/:id/pins                                              |
| DELETE | /api/channels/:channelID/pins/:messageID                            |

## 4.6 Emote-related

| Type   | Endpoint / event name                                               |
| ------:| ------------------------------------------------------------------- |
| event  | emote/new                                                           |
| event  | emote/delete                                                        |
| GET    | /api/emotes                                                         |
| POST   | /api/emotes                                                         |
| GET    | /api/emotes/:shortcode                                              |
| DELETE | /api/emotes/:shortcode                                              |

## 4.7 Session-related

| Type   | Endpoint / event name                                               |
| ------:| ------------------------------------------------------------------- |
| GET    | /api/sessions                                                       |
| POST   | /api/sessions                                                       |
| GET    | /api/sessions/:id                                                   |
| DELETE | /api/sessions/:id                                                   |
