# aiodiameter

A pure-Python implementation of the Diameter protocol (RFC 6733) on top of asyncio. There are no dependencies and no C bindings: the whole protocol, from the byte layout of AVPs up to the peer state machine, is implemented in plain Python.

I wrote this in 2019 while integrating a charging system with an Ericsson online charging system over SCAP. I keep it here as a reference implementation, since all the protocol internals are visible and easy to read.

## What's inside

* Binary encoder and decoder for Diameter headers and AVPs, including flag handling (`R/P/E/T`, `V/M/P`) and 4-octet padding
* The RFC 6733 peer state machine (`WAIT_CONN_ACK`, `WAIT_I_CEA`, `I_OPEN`, ...) with CER/CEA capability exchange and DPR/DPA disconnect
* Device watchdog keep-alives (DWR/DWA, RFC 3539)
* Session handling for credit control in the style of RFC 4006
* Over 20 AVP data types: `Unsigned32/64`, `Integer32/64`, `Float32/64`, `OctetString`, `UTF8String`, `DiameterIdentity`, `DiameterURI`, `Address`, `Time`, `Enumerated`, `Grouped`, `IPFilterRule` and so on
* An XML dictionary with about 1,700 AVP definitions, plus Ericsson's SCAP vendor dictionary
* asyncio transport, where each peer is an `asyncio.Protocol`

## Usage

Connecting to a peer. The state machine does the capability exchange and the watchdog keeps the connection alive:

```python
import asyncio
from async_handler import addPeer

async def main():
    await addPeer(host="192.0.2.12", port=3868)
    await asyncio.Event().wait()

asyncio.run(main())
```

Building a message by hand:

```python
from message import Message
from baseavp import AVP
from datatypes.diamidentity import DiameterIdentity
from datatypes.unsigned32 import Unsigned32

# Capabilities-Exchange-Request
cer = Message(cmdflags=0x80, cmdcode=257, appId=0)
cer.addNewAVP(AVP(code=264, flags=0x40, data=DiameterIdentity("client.example.com")))
cer.addNewAVP(AVP(code=296, flags=0x40, data=DiameterIdentity("example.com")))
cer.addNewAVP(AVP(code=266, flags=0x40, data=Unsigned32(193)))

wire_bytes = cer.encode()
```

There are also ready-made builders for the base protocol commands in `boilerplatemessages.py` (`makeCER`, `makeDWR`, `makeDPR` and friends).

## Layout

```
message.py, diameterheader.py    message and header model, binary encoding
baseavp.py, avpheader.py         AVP model, flags, padding
datatypes/                       one module per RFC 6733 data type
xml/                             AVP dictionaries (base + SCAP) and parsers
peer.py, peertable.py            peer model and RFC 6733 state machine
watchdogtask.py                  DWR/DWA keep-alive task
sessionFactory.py                credit-control session state machine
async_handler.py, handler.py     asyncio protocol glue and dispatch
boilerplatemessages.py           builders for base protocol commands
```

## What's covered

| Spec | Scope | State |
|------|-------|-------|
| RFC 6733 | Base protocol: header, AVPs, CER/CEA, DPR/DPA, peer FSM | done |
| RFC 3539 | Transport watchdog (DWR/DWA) | done |
| RFC 4006 | Credit-control session states | client side |
| Ericsson SCAP | Vendor charging application | dictionary and messages |
| SCTP transport | | not done, TCP only |

## Status

Tested in 2019 against a live Ericsson OCS. The code predates type hints and modern packaging, so if you want to build on it, adding a `pyproject.toml` and a test suite is the obvious first step.

## License

[MIT](LICENSE)
