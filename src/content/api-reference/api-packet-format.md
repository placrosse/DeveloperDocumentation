---
title: API Packet Format
order: 1
section: API Reference
subsections:
  - Creating Command Packets
  - Parsing Async Packets
---

## Packet Structures

### Client Command Packets

Packets are sent from Client → Sphero in the following byte format:

SOP1 | SOP2 | DID | CID | SEQ | DLEN | \<data\> | CHK
-----|------|-----|-----|-----|------|--------|----

### Description of the fields

|      |                      |
-----  | -------------------- | -----------
SOP1   | Start of Packet #1   | Always FFh
SOP2   | Start of Packet #2   | F8 to FFh encoding 4 bits of per-message options (see below)
DID    | Device ID            | The virtual device this packet is intended for
CID    | Command ID           | The command code
SEQ    | Sequence Number      | This client field is echoed in the response for all synchronous commands (and ignored by Sphero when SOP2 has bit 0 clear)
DLEN   | Data Length          | The number of bytes following through the end of the packet
\<data\> | Data                 | Optional data to accompany the Command
CHK    | Checksum             | The modulo 256 sum of all the bytes from the DID through the end of the data payload, bit inverted (1's complement)

### SOP2 bitfield encoding

bits 7-4  | bit 3   | bit 2   | bit 1           | bit 0
--------- | ------- | ------- | --------------- | -----
1111      | 1       | 1       | Reset timeout   | Answer

* Answer – When set to 1, act upon this command and send a reply.
  When 0, act but do not reply.
* Reset timeout – When set to 1, reset the client inactivity timeout after executing this command.
  When 0, do not reset the timer.

The Answer bit has essentially existed since FW 0.99; the remainder of the bit definitions came into existence with FW 1.26 in late 2012.
￼
## Sphero Response Packets

Commands are acknowledged from Sphero → Client in a similar format:

SOP1  | SOP2   | MRSP   | SEQ   | DLEN   | \<data\>   | CHK
----- | ------ | ------ | ----- | ------ | -------- | ----

### Description of the fields

|      |                      |
-----  | -------------------- | -----------
SOP1   | Start of Packet #1   | Always FFh
SOP2   | Start of Packet #2   | Set to FFh when this is an acknowledgement, FEh when this is an asynchronous message
MRSP   | Message Response    ￼| This is generated by the message decoder of the virtual device (refer to the appropriate appendix for a list of values)
SEQ    | Sequence Number      | Echoed to the client when this is a direct message response (set to 00h when SOP2 = FEh)
DLEN   | Data Length          | The number of bytes following through the end of the packet
\<data\> | Data                 | Optional data in response to the Command or based on "streaming" data settings
CHK    | Checksum             | Packet checksum (as computed above)

## Few things to note

* Asynchronous (aka "streaming") packets are implemented by changing the value of SOP2 and clearing the Answer bit.
  This can improve responsiveness (and decrease command latency) but through non-guaranteed delivery.
  The packet format is slightly different in the Sphero → Client direction.
* DLEN is always at least 01h since the CHK byte follows.
  In some special cases it is set to FFh to signify a fixed \<data\> length greater than 254 bytes.
  This is specific to certain DID/CID combinations.
* The SOP1/SOP2 and CHK fields are used to identify correctly formed packets before they're submitted to a DID for processing.
* Here is an example of computing a checksum to transmit a Ping packet.
  The bytes for the packet (with a sequence number of 52h) are: `FFh FFh 00h 01h 52h 01h \<chk\>`.
  The checksum equals the sum of the underlined bytes (54h) modulo 256 (still 54h) and then bit inverted (ABh).

Commands are grouped into two categories: set and get.
Set commands assign a defined variable in Sphero and include a non-zero data payload that contains the assignment.
Responses are in the most simple form, without a data payload.
Rather than duplicate them all through the document.

Here is the Simple Response to a successful set command:

P1   | SOP2   | MRSP   | SEQ        | DLEN   | CHK
---- | ------ | ------ | ---------- | ------ | -----
FFh  | FFh    | 00h    | \<echoed\>   | 01h    | \<computed\>

Get commands request settings, status or the current value of dynamic values.
The formats of these responses are detailed in each CID.

## Sphero Asynchronous Packets

As mentioned previously, the format of asynchronous packets originating from Sphero is slightly different:

There are no MRSP or SEQ bytes, since they don't make sense in this context.
The ID CODE field identifies what type of data is arriving in this packet and as you can see, the DLEN field has been expanded to (clearly) permit payloads exceeding 254 bytes.
The following is a list of the currently defined ID codes and the DID/CID commands that control generation of those packets where applicable.

ID CODE  | Description                                      | Generating DID   | CID
-------- | -------------                                    | ---------------- | ----
01h      | Power notifications                              | 00h              | 21h
02h      | Level 1 Diagnostic response                      | 00h              | 40h
03h      | Sensor data streaming                            | 02h              | 11h
04h      | Config block contents                            | 02h              | 40h
05h      | Pre-sleep warning (10 sec)                       | n/a              | n/a
06h      | Macro markers                                    | n/a              | n/a
07h      | Collision detected                               | 02h              | 12h
08h      | orbBasic PRINT message                           | n/a              | n/a
09h      | orbBasic error message, ASCII                    | n/a              | n/a
0Ah      | orbBasic error message, binary                   | n/a              | n/a
0Bh      | Self Level Result                                | 02h              | 09h
0Ch      | Gyro axis limit exceeded (FW ver 3.10 and later) | n/a              | n/a
0Dh      | Sphero's soul data                               | 02h              | 43h
0Eh      | Level up notification                            | n/a              | n/a
0Fh      | Shield damage notification                       | n/a              | n/a
10h      | XP update notification                           | n/a              | n/a
11h      | Boost update notification                        | n/a              | n/a

Power notification (01h) details are included with “Set Power Notification”.

Level 1 diagnostic response details are included with “Perform Level 1 Diagnostics”.

Sensor data streaming details are included with “Set Data Streaming”.

Config block contents details are included with “Get Configuration Block”.

The Pre-Sleep warning is sent once, 10 seconds prior to Sphero entering sleep due to client inactivity.
Macro markers come from special macro commands and optionally at the end of a macro.

Collision detection messages are based on the accelerometer, measured speed, etc.

The orbBasic PRINT ID 08h is akin to STDOUT, 09h to STDERR and 0Ah a machine readable version of STDERR.

Self Level Result is sent after the self level routine completes, but only if the routine was initiated by an API call.

The Gyro Axis Limit Exceeded message contains one byte of data where the bits signify the axes that exceeded the limit: bit 0 = X positive, bit 1 = X negative, bit 2 = Y+, bit 3 = Y-, bit 4 = Z+ and bit 5 = Z-.
The message is emitted when one threshold is exceeded and all of the max measurements are cleared upon receipt of a Set Heading API command (DID 02h, CID 01h).

The level up notification contains two 16-bit unsigned integers.
The first is the new robot level.
The second is the total number of attribute points the user has to spend.

The Shield damage notification contains one unsigned byte representing the portion of shield left (out of 255).
The shields are damaged when Sphero collides with other objects.
The shields are regenerate automatically over time.
Both collisions and regeneration generate asynchronous updates.

The XP update notification contains one byte representing how much experience Sphero has gained toward the next robot level.
The scale is from 0=0% to 255=100%.

The boost update notification contains one byte representing how much boost capability Sphero has.
The value goes down when boost is used and automatically regenerates over time.
Regeneration and use both generate asynchronous updates.
The scale is from 0=0% to 255=100%.

## Data Packing

Multi-byte numbers are sent MSB first in both directions.
Here are two examples of how the data looks "on the wire."

22h     | 78h      | 00h   | 41h
------- | -------- | ----- | -------
byte 0  |          |       | byte 3

= 22780041h (unsigned 32-bit integer)

40h     | 49h      | 0Fh   | DBh
------- | -------- | ----- | -------
byte 0  |          |       | byte 3

= 3.1415927 (single precision IEEE-754)


