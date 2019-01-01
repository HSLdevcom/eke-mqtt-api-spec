# EKE MQTT API specification

The EKE MQTT API is a streaming status reporting API for trains.
The EKE unit in the train cabin captures messages from the local network and publishes them as MQTT messages.

This document specifies the data transport, encoding and content, in that order.


## Transport protocol

Use MQTT over TCP without TLS.

Use exactly one working MQTT connection at a time from the point of view of EKE.
That will retain the message order in the data stream if we ignore the effect of QoS.

### MQTT version

[MQTT Version 3.1.1 Plus Errata 01](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/errata01/os/mqtt-v3.1.1-errata01-os-complete.html)

**FIXME:** Consider MQTT 5.0 if relevant closer to implementation. Trustworthy client libraries, our bridge and possibly our broker needs to be able to handle 5.0 before using it. "MQTT v5.0 is not backward compatible (like v3.1.1)" says https://blog.codecentric.de/en/2017/11/hello-mqtt-version-5-0/ .

### Broker

The MQTT Broker can be given either as an IP address or as a domain name. This must be configurable.

Port is `8883` but must be configurable.

This is usually defined in the arguments for `connect()` or equivalent of the MQTT client library.

### Username and password

No username or password is required.

### Topics

```
eke/<apiVersion>/<trainType>/<trainUnit>/<cabin>/<dataSourceInTrain>/<messageType>
```

`apiVersion` for the first production version is `v1`.
For development, e.g. `dev` may be used.

`trainType` is `sm5`.

`trainUnit` is e.g. `1`, `17` or `81`.

`cabin` is either `A` or `B`.

`dataSourceInTrain`:
- `stadlerUDP`
- `ekeJKV`

`messageType` if `dataSourceInTrain` has value `stadlerUDP`:
- `status`

`messageType` if `dataSourceInTrain` has value `ekeJKV`:
- `status`
- `journeyID`
- `trainInfo`
- `ekeJKVError`
- `ekeJKVTransmissionError`
- `balise`

For example `eke/v1/sm5/13/A/stadlerUDP/status` or `eke/v1/sm5/47/B/ekeJKV/ekeJKVTransmissionError`.

This is usually defined in the arguments for `publish()` or equivalent of the MQTT client library.

### Client Identifier

Use `eke-<trainType>-<trainUnit>-<cabin>`, e.g. `eke-sm5-25-A`.

This is usually defined in the arguments for `connect()` or equivalent of the MQTT client library.

### QoS

Use QoS 1.
Handling the duplicate messages due to QoS 1 is the responsibility of the MQTT subscriber, as usual.

This is usually defined in the arguments for `publish()` or equivalent of the MQTT client library.

### Retained messages

All messages should be retained messages.

This is usually defined in the arguments for `publish()` or equivalent of the MQTT client library.

### Clean session

Set clean session to false.

This is usually defined in the arguments for `connect()` or equivalent of the MQTT client library.

### Last Will scheme

**FIXME:** Test that last wills are forwarded by an MQTT bridge like any other message.

Use a connection status topic:
```
eke/<apiVersion>/<trainType>/<trainUnit>/<cabin>/connectionStatus
```

When connecting to the broker with the MQTT `CONNECT` message, specify a Last Will UTF-8 message payload `disconnected` into the connection status topic.

Ask broker to send the Last Will as retained message with QoS 2.

These are usually defined in the arguments for `connect()` or equivalent of the MQTT client library.

When connecting or reconnecting, first send a UTF-8 message `connected at <connectedTime>` to the connection status topic.
`<connectedTime>` should be created as early as possible when the MQTT (re)connection succeeds.
It must contain an ISO 8601 timestamp with a fixed precision, for example nanosecond precision, and with UTC time zone denoted with `Z`, e.g. `20180518T082928.123456789Z`.

**FIXME:** Select and document the precision here instead of specifying with "for example". Nanosecond precision matches the precision of the Protocol Buffers Timestamp.

This message should be sent as a retained message with QoS 2, too.

This is usually handled in the `connectionSucceededCallback()` or equivalent of the MQTT client library.

### Buffering

In case of lost connections to the MQTT broker, all data is retained in a deque or ringbuffer for `k` days.
Another data structure may be used instead as long as the oldest messages are thrown away first if the size limit is reached.

Persist on disk.
MQTT client libraries will not usually do this but will offer the necessary callbacks.

The deque size/length needs to be parameterized.
The hardware is expected to have at least 256 MiB of storage space, though the deque might need some extra storage space for sending the oldest messages in spotty network conditions where `PUBACK`s from the broker are not received.

The locally persisted messages should be removed only after receiving the corresponding `PUBACK` from the broker.

Note: The maximum number of MQTT messages in-flight is 65534 as the MQTT packet identifier for QoS > 0 lies on the closed interval [1, 65534].
Even though they should, not all MQTT client libraries can internally handle their users trying to publish over 65k messages before the `PUBACK`s start clearing the sending queue.
If the MQTT client library cannot handle this, then a semaphore of size 65534 solves the issue.

The idea fleshed out a bit:
```
Capturing and encoding   Storage in durable deque on disk   Publishing   Semaphore of size 2**16 - 1   MQTT client library

...

|                        |                                  |            |                             |
|                        |                                  |            |   PUBACK callback           |
|                        |                                  |<-----------------------------------------|
|                        |   remove the oldest              |            |                             |
|                        |<---------------------------------|            |                             |
|                        |                                  |   release  |                             |
|                        |                                  |----------->|                             |
|                        |                                  |            |                             |
|----------------------->|                                  |            |                             |
|                        |   peek at the oldest             |            |                             |
|                        |--------------------------------->|            |                             |
|                        |                                  |   acquire  |                             |
|                        |                                  |----------->|                             |
|                        |                                  |            |   publish                   |
|                        |                                  |----------------------------------------->|

...
```

### Reconnections

Reconnect automatically.
This is usually handled by the MQTT client library by configuration.

If useful, wait `n` seconds after noticing the disconnection.
`n` must be configurable and may be 10 by default.

This is usually handled in the `connectionLostCallback()` or equivalent of the MQTT client library.


## Encoding

Before forwarding over MQTT, each message received by the EKE unit is wrapped into a [Protocol Buffers](https://developers.google.com/protocol-buffers/) message as specified in the schema file `./fi_hsl_eke.proto` in this repository.

Use the schema to generate stubs for the messages.


## Data content

There are two data sources, Stadler UDP and EKE JKV.
EKE JKV produces two types of messages, status and event messages.

The messages the EKE unit receives are not modified except for wrapping them with metadata.
The metadata is specified in the Protocol Buffers schema file `./fi_hsl_eke.proto`.

### Logic for Stadler UDP

Copy 1:1 the binary content of the UDP packets without the UDP header.

Send the Stadler UDP messages only from the active cabin.
If neither cabin is active, send from cabin A.

**FIXME:** Is it still valid that only the active cabin sends?

### Logic for EKE JKV status

Copy 1:1 the binary content of the JKV status packet.

### Logic for EKE JKV events

Copy 1:1 the binary content of the JKV event packet.
