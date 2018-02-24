# Lightweight network protocol
Description and implementation of the lightweight network protocol for home automation

## Goal

## Encryption schema
### Assumptions
* each device has the unique AES key for encryption and AES key for authentication
* the server knows AES keys for every device
* each device guarantees the uniqueness of the nonce value
  - static part is unique for each boot of device
  - nonce overall is unique for each packet sent

### Initialization
1) device after boot takes static part of the nonce and sends packet announcing this nonce
  - nonce is open text
2) server acknowledges usage of this nonce
Server and device now both knows AES key and static part of the nonce.

### Sending a message
1) server or device creates a message payload
2) encrypts payload and updates variables for per-packet part of the nonce
3) adds header
4) signs it (24 b reserved for routing are treated as zeroes)
5) adds per-packet part of the nonce
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
