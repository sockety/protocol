# Sockety - protocol specification

The protocol has performance in mind, so it limits overhead to minimum.

## Dictionary

* **Multiplexing** - handling concurrent channels within a single connection
* **Channel** - single communication context within a connection
* **Stream** - dynamic content that may be attached to Message/Response

## Encoding

* All numbers in connection are unsigned low endian integers
* Text in connection is UTF-8

## Overview

The Sockety protocol is designed to be used over TCP (directly, or i.e. over TLS).

### Limitations

There are some limitations, because of the protocol structure:

| Name                           | Limit        | Description                                                                     |
|--------------------------------|--------------|---------------------------------------------------------------------------------|
| **Maximum number of channels** | 4,096        | *0-4095, max. uint12 used for switching*                                        |
| **Maximum action name size**   | 65,535 bytes | *max. uint16 to describe size*                                                  |
| **Maximum payload size**       | ~256TiB      | *max. uint48 to describe size*                                                  |
| **Maximum total files size**   | ~256TiB      | *max. uint48 to describe size*                                                  |
| **Maximum number of files**    | 16,777,215   | *max. uint24 to describe count*                                                 |
| **Maximum file size**          | ~256TiB      | *max. uint48 to describe size*                                                  |
| **Number of FastReply codes**  | 4,096        | *0-4095, max. uint12 used to respond*                                           |
| **Maximum single packet size** | ~4GiB        | *max. uint32 to describe size; all packets can be split, so it's not a problem* |

## Connection header

After connection, both sides should send the connection header to each other. It consists of 1-3 bytes - control byte and number of own channels:

| Binary       | Bytes     | Required                 | Name               | Legend                                                                                                      |
|--------------|-----------|--------------------------|--------------------|-------------------------------------------------------------------------------------------------------------|
| `111000AA`   | 1 byte    | Yes                      | Control byte       | `A` - number of channels header (`00` - single channel, `01` - uint8, `10` - uint16, `11` - maximum number) |
| dynamic uint | 1-2 bytes | When `A` is `01` or `10` | Number of channels | As unsigned integer, based on `A`, maximum of 4096                                                          |

## Packets

There are 13 types of packets in protocol - 7 with static size and the rest with dynamic content size:

| Type               | Description                                                                                     | Size                                                                   |
|--------------------|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| **Switch Channel** | Switch multiplexing to another channel                                                          | 1-2 bytes                                                              |
| **Heartbeat**      | Informs that the connection is persisting, when it's idle                                       | 1 byte                                                                 |
| **Go Away**        | The server or client will soon gracefully disconnect                                            | 1 byte                                                                 |
| **Message**        | Regular message sent over the connection                                                        | min. 21 bytes *= 16 bytes of UUID + 4 header bytes + 1 byte of action* |
| **Response**       | Full-size response to the Message                                                               | min. 35 bytes *= 32 bytes of 2 UUIDs + 3 header bytes*                 |
| **Continue**       | Continue the Message or Response packet, if the previous one was not full                       | min. 2 header bytes + content                                          |
| **Fast Reply**     | Short response to the Message - code only - number between 0 and 4095                           | 17-18 bytes                                                            |
| **Stream**         | Send some data for the Stream of Message or Response                                            | min. 2 header bytes + content                                          |
| **Stream End**     | Inform that the Stream has been ended                                                           | 1 byte                                                                 |
| **File**           | Send some file content of Message or Response                                                   | min. 2 header bytes + content                                          |
| **File End**       | Inform that the file content of specified index has been ended                                  | min. 1 byte                                                            |
| **Data**           | Send payload for Message or Response                                                            | min. 2 header bytes + content                                          |
| **Abort**          | Inform that the Message or Response that is in progress should be marked as aborted and ignored | 1 byte                                                                 |

## Packet format

Every packet, in the first byte, will contain information about the packet type, and may contain some bits describing details.

First 4 bits in the first byte are enough to indicate instruction type. Often, to avoid unnecessary header and too big integer size, these bits may i.e. indicate how many bytes should be read for some value (i.e. uint8 = 1 byte, instead of uint32 = 4 bytes)

| Type                          | Binary                                                | Legend                                                                                                                                                       |
|-------------------------------|-------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Switch Channel: low value** | `0000AAAA`                                            | `A` - uint4 (0-15) - channel ID                                                                                                                              |
| **Switch Channel**            | `0001AAAA` `AAAAAAAA`                                 | `A` - uint12 (0-4095) - channel ID                                                                                                                           |
| **Heartbeat**                 | `10100000`                                            | -                                                                                                                                                            |
| **Go Away**                   | `10110000`                                            | -                                                                                                                                                            |
| **Message**                   | `0010AABC` *[rest described in further section]*      | `A` - packet size header (`00` - uint8, `01` - uint16, `10` - uint24, `11` - uint32)<br>`B` - has stream attached<br>`C` - expects any response              |
| **Response**                  | `0101AABC` *[rest described in further section]*      | `A` - packet size header (`00` - uint8, `01` - uint16, `10` - uint24, `11` - uint32)<br>`B` - has stream attached<br>`C` - expects any response              |
| **Continue**                  | `0110AA00` *[follows rest of message/response]*       | `A` - packet size header (`00` - uint8, `01` - uint16, `10` - uint24, `11` - uint32)                                                                         |
| **Fast Reply: low value**     | `0011AAAA` `[uuid]`                                   | `A` - uint4 (0-15) - response code                                                                                                                           |
| **Fast Reply**                | `0100AAAA` `AAAAAAAA` `<uuid bytes>`                  | `A` - uint12 (0-4095) - response code                                                                                                                        |
| **Stream**                    | `0111AA00` `<packet size>` `<content>`                | `A` - packet size header (`00` - uint8, `01` - uint16, `10` - uint24, `11` - uint32)                                                                         |
| **Stream End**                | `10000000`                                            | -                                                                                                                                                            |
| **File: index 0**             | `1100AA00` `<packet size>` `<content>`                | `A` - packet size header (`00` - uint8, `01` - uint16, `10` - uint24, `11` - uint32)                                                                         |
| **File**                      | `1100AABB` `<packet size>` `<file index>` `<content>` | `A` - packet size header (`00` - uint8, `01` - uint16, `10` - uint24, `11` - uint32)<br>`B` - file index header (`01` - uint8, `10` - uint16, `11` - uint24) |
| **File End: index 0**         | `11010000`                                            | -                                                                                                                                                            |
| **File End**                  | `110100AA` `<file index>`                             | `A` - file index header (`01` - uint8, `10` - uint16, `11` - uint24)                                                                                         |
| **Data**                      | `1110AA00` `<packet size>` `<content>`                | `A` - packet size header (`00` - uint8, `01` - uint16, `10` - uint24, `11` - uint32)                                                                         |
| **Abort**                     | `10010000`                                            | -                                                                                                                                                            |

### Message

Message is the most complex packet, so it would be hard to describe it in the table above.

The structure of the message is described in the table below, part by part:

| Binary        | Bytes       | Required             | Name                     | Legend                                                                                                                                                                                                                                                                                                                                                                                             |
|---------------|-------------|----------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `0010AABC`    | 1 byte      | Yes                  | Message packet header    | `A` - packet size header (`00` - uint8, `01` - uint16, `10` - uint24, `11` - uint32)<br>`B` - has stream attached<br>`C` - expects any response                                                                                                                                                                                                                                                    |
| dynamic uint  | 1-4 bytes   | Yes                  | Packet size              | Packet size as unsigned integer, based on `A`                                                                                                                                                                                                                                                                                                                                                      |
| `DDEEFFG0`    | 1 byte      | Yes                  | Message flags            | `D` - payload size header (`00` - no payload, `01` - uint8, `10` - uint16, `11` - uint48)<br>`E` - files count header (`00` - no files, `01` - uint8, `10` - uint16, `11` - uint24)<br>`F` - total files size header, ignored when `E` is `00` (`00` - uint16, `01` - uint24, `10` - uint32, `11` - uint48)<br>`G` - action name header (`0` - uint8, `1` - uint16)<br>*Last bit is not used yet.* |
| dynamic bytes | 16 bytes    | Yes                  | Message UUID             | UUID v4 as 16 bytes, not of string representation                                                                                                                                                                                                                                                                                                                                                  |
| dynamic uint  | 1-2 bytes   | Yes                  | Action name size         | Action name size as unsigned integer, based on `G`.<br>*Remember, that it's binary size, not string size, so Unicode characters may use multiple bytes each.*                                                                                                                                                                                                                                      |
| dynamic utf-8 | above value | Yes                  | Action name              | The action name                                                                                                                                                                                                                                                                                                                                                                                    |  
| dynamic uint  | 0-6 bytes   | When `D` is not `00` | Payload size             | Payload size as unsigned integer, based on `D`                                                                                                                                                                                                                                                                                                                                                     |
| dynamic uint  | 0-3 bytes   | When `E` is not `00` | Files count              | Number of files in the message, based on `E`                                                                                                                                                                                                                                                                                                                                                       |
| dynamic uint  | 0-6 bytes   | When `E` is not `00` | Total files size         | Total files size in the message, based on `F`                                                                                                                                                                                                                                                                                                                                                      |
| dynamic bytes | dynamic     | When `E` is not `00` | List of headers of files | see section below                                                                                                                                                                                                                                                                                                                                                                                  | 

#### Files header

The last part of Message/Response consists of list of files headers. Until the header for specific file is sent, there should be no `File` packet sent for this file.

There is no separators between consecutive files. File header consists of name and size information:

| Binary        | Bytes     | Required | Name           | Legend                                                                                                                                      |
|---------------|-----------|----------|----------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| `0000AAB0`    | 1 byte    | Yes      | Header         | `A` - file size header (`00` - uint8, `01` - uint16, `10` - uint24, `11` - uint48)<br>`B` - name size header (`0` - uint8, `1` - uint16)    |
| dynamic uint  | 1-6 bytes | Yes      | File size      | As unsigned integer, based on `A`                                                                                                           |
| dynamic uint  | 1-2 bytes | Yes      | File name size | As unsigned integer, based on `B`<br>*Remember, that it's binary size, not string size, so Unicode characters may use multiple bytes each.* |
| dynamic bytes | dynamic   | Yes      | File name      | The file name                                                                                                                               |

### Response

The response is similar to Message, except 2 changes:

* Response has na action name, so there is no action name, its size, and the size flag is ignored
* Before Response's UUID, there is additional UUID of the Message, that it's response for

## Multiplexing

Initially, there is channel `0` used for writing. To favor synchronous messages, you don't need to send channel for every instruction - the previous one persists.

You need to use another channel, if on the previous one there is a Message or Response ongoing. The Message or Response is finished when:

* The Message/Response packet has been finished
* If there was payload expected, `Data` packet is already finished
* If there were files expected, all are ended
* If there was a stream attached to, it's ended
