# Lightweight network protocol
Description and implementation of the lightweight network protocol for home automation

## Goal
The main goal is to provide simple but secure protocol for transferring information to and from home 
automation gadgets. Primary usage expects NRF24L01 as wireless interface, so the packet length is limited to 32 bytes.
Protocol shall provide channel for secure transfer of informations and validation of the sender itself.

## Encryption schema
### Assumptions
* communication is between one server and many devices
* each device has the unique AES key for encryption and AES key for authentication
* the server knows AES keys for every device
* each device guarantees the uniqueness of the nonce value
  - static part is unique for each boot of device
  - nonce overall is unique for each packet sent

TODO - authenticate the IV?

### Initialization
TODO detailed description
1) device after boot takes static part of the nonce and sends packet announcing this nonce
  - nonce is open text
2) server acknowledges usage of this nonce
Server and device now both knows AES key and static part of the nonce.

### Sending a message
1) server or device creates a message payload
2) encrypts payload and updates variables for per-packet part of the nonce
3) adds header
4) signs it (24 b reserved for routing are treated as zeroes)
+5) adds per-packet part of the nonce
6) sends it

### Receiving a message
1) server or device receives a packet
2) validates destination address
3) verifies signature (24 b reserver for routing are treated as zeroes)
4) decrypts payload
5) checks the mark byte
6) processes payload

TODO - retransmit indication in header?

## Packet structure
The protocol uses packets of a length 32 B (256 bits). This is primary due to planned usage NRF24L01 wireless modules.
The packet has following structure:
* *12 b* - destination address
* *12 b* - source address
* *4 b* - flags
  - *1 b* - stream start
  - *1 b* - stream end
  - *2 b* - reserved
* *4 b* - type: beacon, nonce-solicitation, data, ???
* *8 b* - version of protocol
* *24 b* - reserved for future use (routing information)
* *128 b* - AES payload
  - *1 B* - mark
  - *1 B* - length
  - *1 B* - command
  - *1 B* - flags
  - *12 B* - data
* *32 b* - MIC
* *32 b* - nonce (per packet part)
  - *16 b* - time in seconds
  - *16 b* - packet counter
  
### Streams
The protocol allows to gather multiple packets to one to transfer larger amount of information. The stream starts with packet with `stream start` set and contains all packets until the one with `stream end` set. There must not be sent packet with `stream start` set during the stream transmission and this situation is treated as error and the stream should not be processed. The unit stream consist of a packet with `stream start` and `stream end` bits both set and every device and server must be able to transmit and receive it. The support of the multi-packet stream is optional.

## Acknowledge
TODO - WIP

### Message types
* `beacon (0x01)` - keep alive message "I am here and still alive". Payload can contain information about device.
* `IV solicitation (0x02)` - message for solicitation of IV. The only type of packet with unencrypted payload. Payload is 96bit static part of the IV padded with zeroes.
* `IV solicitation response (0x03)` - message for solicitation of IV. Payload is 96bit static par of the IV padded with zeroes, encrypted.
* `data (0x04)` - the most common type of packed. Contains data as payload (read further for description).

### Protocol version
* `0x00` for the first version of the protocol. Numbering plan is not yet specified.

### Payload
Payload is encrypted using AES-CTR (TODO) except the `IV solicitation` packet. The structure of the packet is not yet stable but it is expected to have:
* `mark` - to ensure that the message has been decrypted succesfully
* `length` - determines the length of the payload. If there is a stream, it determines length of entire stream. Counts in packets (TODO present only in multi-packet streams?)
* `command` - what is transferred
* `data` - interpretation depends on the `command`

### MIC
  
## Generating the IV
The initialization vector (IV) consists of the static part and the per-packet part. Static part is generated on the boot of the device or after dynamic part is getting close to limit. Dynamic part changes predictably for every message. The static part is 96 b long and the dynamic part is 32 b long.

### Static part
The static part of the IV is generated from the number of boots, time and the most random data which we can get. It also stores a bit determining the direction of the message (uplink or downlink). This simplifies the maintenance of the uniquness of the IV.

The static part of the IV has following structure:
* *16 b* - number of boots
* *32 b* - time (specified below)
* *47 b* - random number
* *1 b* - direction (uplink == 1, downlink == 0)
TODO further explanation

The application should not derive anything from the static part of the IV, this structure is only valid for reference implementation.

### Per-packet part
The per-packet part of the IV is deterministic and has following structure:
* *16 b* - packet counter
* *16 b* - time of send

The packet counter increments after every packet send, if packed needs to be retransmitted, it should be retransmitted as original. The time of send is counted in the unit of a 1/10 of a second. If the device has ability to handle current time, the number of units should be aligned to the seconds since the start of an hour.

The packet counter must not overflow because there is not guaranteed that the time of send and packet counter are not corelated. If the packet counter is greater than 64000 the new IV solicitation should take a place. There must not be sent any encrypted packer if there is not solicitated new IV and the packet-counter overflows.
