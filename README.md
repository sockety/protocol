# Sockety - blazing fast communication protocol

## Overview

The Sockety protocol has performance in mind, and is used for P2P communication between workers - sending commands, data and files between them.

The overhead is cut to minimum, to improve performance and reduce transfer.

## Features

* Multiplexing
  * Allows up to 4096 concurrent read/write channels
* UUIDs built in the protocol
  * Every message will have UUID by design
* Real-time streams
  * Every message may attach a stream of any kind of data
* Files
  * Message has a concept of files - file with known name, size and contents
* Payload
  * Message may contain additional payload bytes
* Flexible responses
  * Message doesn't need to get any response, yet may get multiple
  * The response may be either just a lightweight code (`FastReply` - 0-4095 code), or a full-scale message (`Response` with data, files and/or stream)

## Links

* [**Specification**](./SPECIFICATION.md) - protocol specification
* [**Node.js implementation**](https://github.com/sockety/sockety-js) - TypeScript/JavaScript implementation of client/server
