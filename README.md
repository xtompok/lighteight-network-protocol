# Lightweight network protocol
Description and implementation of the lightweight network protocol for home automation

## Goal

## Encryption schema
TODO - reserved bits

## Packet structure
The protocol uses packets of a length 32 B (256 bits). This is primary due to planned usage NRF24L01 wireless modules.
The packet has following structure:
* *12 b* - destination address
* *12 b* - source address
* *4 b* - flags
  - *1 b* - stream start
  - *1 b* - stream end
  - *2 b* - reserved
  - *4 b* - type: beacon, nonce-solicitation, data, ???
* *8 b* - version of protocol
* *24 b* - reserved for future use (routing information)
* *128 b* - AES payload
  - *1 B* - length
  - *1 B* - command
  - *14 B* - data
* *32 b* - MIC
* *32 b* - nonce (per packet part)
  - *16 b* - time in seconds
  - *16 b* - packet counter
