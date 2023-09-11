# Everynet RAN Routing API Specification

> Note: This document is a draft (beta) specification of the RAN Routing API

## Table of contents
- [Foreword](#foreword)
- [Introduction](#introduction)
- [Subscription Management](#subscription-management)
  - [Select devices](#devicesselect)
  - [Insert devices](#devicesinsert)
  - [Update devices](#devicesupdate)
  - [Delete devices](#devicesdrop)
  - [Delete all devices](#devicesdropall)
- [Multicast Management](#multicast-management)
  - [Create multicast group](#create-multicast-group)
  - [Delete multicast groups](#delete-multicast-groups)
  - [Get multicast groups](#get-multicast-groups)
  - [Add device to multicast group](#add-device-to-multicast-group)
  - [Remove device from multicast group](#remove-device-from-multicast-group)
- [Message Streaming](#message-streaming)
- [Upstream Traffic = Uplinks + Join Requests](#upstream-traffic-uplinks-join-requests)
  - [Messages from RAN to LNS](#messages-from-ran-to-lns)
  - [Messages from LNS to RAN](#messages-from-lns-to-ran)
  - [MIC Challenge Procedure](#mic-challenge-procedure)
- [Downstream Traffic = Downlinks + Join Accepts](#downstream-traffic-downlinks-join-accepts)
  - [Messages from LNS to RAN](#messages-from-lns-to-ran-1)
  - [Messages from RAN to LNS](#messages-from-ran-to-lns-1)
- [Message Types](#message-types)
  - [Message Types Overview](#message-types-overview)
    - [`FromRAN` Message Breakdown](#fromran-message-breakdown)
    - [`FromLNS` Message Breakdown](#fromlns-message-breakdown)
  - [UpstreamMessage](#upstreammessage)
  - [UpstreamAckMessage](#upstreamackmessage)
  - [UpstreamRejectMessage](#upstreamrejectmessage)
  - [DownstreamMessage](#downstreammessage)
  - [MulticastDownstreamMessage](#multicastdownstreammessage)
  - [DownstreamAckMessage](#downstreamackmessage)
  - [DownstreamResultMessage](#downstreamresultmessage)
  - [MulticastDownstreamResultMessage](#multicastdownstreamresultmessage)
- [Models](#models)
  - [Position](#position)
  - [LoRaUpstreamRadio](#loraupstreamradio)
  - [FSKUpstreamRadio](#fskupstreamradio)
  - [DownstreamResultCode](#downstreamresultcode)
  - [LoRaDownstreamRadio](#loradownstreamradio)
  - [FSKDownstreamRadio](#fskdownstreamradio)
  - [TransmissionWindow](#transmissionwindow)
  - [MulticastTransmissionWindow](#multicasttransmissionwindow)
  - [MulticastDeviceResult](#multicastdeviceresult)
  - [TMMSList](#tmmslist)

## Foreword

Everynet operates a Neutral-Host Cloud RAN, which can support customers with two different integration options:

- LNS may subscribe to specific devices (by DevEUI) on the Everynet RAN using proprietary RAN API (recommended)
- Devices may roam to Everynet RAN using the LoRa Alliance Backend Interfaces specification

There are several benefits to this first implementation. The most obvious is that it allows a host to subscribe and unsubscribe to individual devices, as the use case and business model support. It also eliminates message traffic from devices that are no longer subscribed to the LNS (or never were).

## Introduction

This API is intended to connect Everynet Cloud RAN (RAN) with customer LoRaWAN Network Server (LNS). The API is LNS-agnostic.

Everynet RAN core funtionality is LoRaWAN traffic routing. It receives messages from gateways and then matches each message with the _customer_ using either _DevAddr_ or pair _(DevEUI, JoinEUI)_. **The relations between device details and customer details are stored in a routing table.**

Everynet RAN API is designed to let customer control the routing table. It also provides both upstream and downstream messaging capabilities.

**Cloud RAN does not store any device-related cryptographic keys and is not capable of decrypting customer traffic.** Maintaining data ownership gurantees without an access to the device keys the RAN enabled with a purpose-built [MIC challenge procedure](#mic-challenge-procedure).

All streaming interactions between RAN and LNS are organized via messaging API. It is based on asynchronous secure websockets (wss://).

## Subscription Management

To start receiving uplinks or join requests we require LNS to **explicitly** subscribe (and unsubscribe) for every device using the API methods below.

Note that subscriptions are simply rows in the RAN traffic routing table, hence the naming convention.

The methods are HTTP-based and are not available via WebSocket interface.

| Method                                                                                                                     | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| -------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [devices.select(CoverageID, ClientID, Optional[DevEUIs])](#Devices.Select)                                                 | Select device subscriptions from the routing table. List of devices is specified via `DevEUIs` parameter. If `DevEUIs` parameter is not set, then all device subscriptions are returned.                                                                                                                                                                                                                                                                                                |
| [devices.insert(CoverageID, ClientID, DevEUI, OneOf[JoinEUI, DevAddr], Optional[Details])](#Devices.Insert)                | Insert device subscription into the routing table to start receiving messages from the specified device. Both `DevEUI` and `DevAddr` are mandatory parameters for ABP devices. while `DevEUI` and `JoinEUI` are mandatory parameters for OTAA devices. Provided `DevEUI` must be unique, while single `DevAddr` may be assigned to several `DevEUIs` simultaneously. Optional `Details` field provides additional device information, such as geographical coordinates or device model. |
| [devices.update(CoverageID, ClientID, DevEUI, JoinEUI, Optional[ActiveDevAddr], Optional[TargetDevAddr])](#Devices.Update) | Update device subscription in a routing table by `DevEUI`. This API function is intended to be used by the LNS to set new device address after the join request has been processed. Optional parameters that are ommited won't be updated, `null` values are not allowed.                                                                                                                                                                                                               |
| [devices.drop(CoverageID, ClientID, DevEUIs)](#Devices.Drop)                                                               | Deletes device subscription from the routing table by `DevEUI`.                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [devices.drop_all(CoverageID, ClientID)](#Devices.DropAll)                                                                 | Deletes all device subscriptions from the routing table.                                                                                                                                                                                                                                                                                                                                                                                                                                |

### Devices.Select

Select device subscriptions from the routing table.

List of devices is specified via `DevEUIs` parameter. If `DevEUIs` parameter is not set, then all device subscriptions are returned.

**Params info:**
| Param        | Required | Type  | Description                                                                                                        |
| ------------ | -------- | ----- | ------------------------------------------------------------------------------------------------------------------ |
| `CoverageID` | yes      | int   | This ID refers to one of the available Everynet network deployments: Brazil, Indonesia, USA, Italy, Spain, UK, ... |
| `ClientID`   | yes      | int   | This ID refers to the client.                                                                                      |
| `DevEUIs`    | no       | str[] | List of `DevEUI` hex strings.                                                                                      |

#### Example

Select multiple devices data:

```bash



$ curl -s --request GET 'https://eu.cloud.everynet.io/api/v1/devices/select' \
--header 'Authorization: Bearer secrettoken' \
--header 'Content-Type: application/json' | jq
[
  {
    "DevEUI": "dddddddddddddddd",
    "JoinEUI": "dddddddddddddddd",
    "ActiveDevAddr": "dddddddd",
    "TargetDevAddr": null,
    "Details": null,
    "CreatedAt": "2022-05-30T10:06:44.726471"
  },
  {
    "DevEUI": "ffffffffffffffff",
    "JoinEUI": null,
    "ActiveDevAddr": "ffffffff",
    "TargetDevAddr": null,
    "Details": null,
    "CreatedAt": "2022-05-31T07:20:00.360463"
  },
  {
    "DevEUI": "eeeeeeeeeeeeeeee",
    "JoinEUI": "eeeeeeeeeeeeeeee",
    "ActiveDevAddr": null,
    "TargetDevAddr": null,
    "Details": null,
    "CreatedAt": "2022-05-31T07:32:16.545797"
  }
]
```

Select single device data:

```bash
$ curl -s --request GET 'https://eu.cloud.everynet.io/api/v1/devices/select?DevEUIs=ffffffffffffffff&DevEUIs=dddddddddddddddd' \
--header 'Authorization: Bearer secrettoken' \
--header 'Content-Type: application/json' | jq
[
  {
    "DevEUI": "dddddddddddddddd",
    "JoinEUI": "dddddddddddddddd",
    "ActiveDevAddr": "dddddddd",
    "TargetDevAddr": null,
    "Details": null,
    "CreatedAt": "2022-05-30T10:06:44.726471"
  },
  {
    "DevEUI": "ffffffffffffffff",
    "JoinEUI": null,
    "ActiveDevAddr": "ffffffff",
    "TargetDevAddr": null,
    "Details": null,
    "CreatedAt": "2022-05-31T07:20:00.360463"
  }
]
```

### Devices.Insert

Subscribe to the device messages. Insert device into the routing table to start receiving messages from the specified device.

Both `DevEUI` and `DevAddr` are mandatory parameters for ABP devices. while `DevEUI` and `JoinEUI` are mandatory parameters for OTAA devices.

Provided `DevEUI` must be unique, while single `DevAddr` may be assigned to several `DevEUIs` simultaneously.

For some NetID types, the DevAddr space is much smaller than the number of possible DevEUIs and it is possible that multiple devices (DevEUIs) share the same DevAddr. RAN supports mupliple DevEUIs pointing to one DevAddr and provides an array of DevEUIs in the `Upstream` message. It is expected that LNS provides a correct `DevEUI` in the `UpstreamAck` message.

Optional `Details` field provides additional device information, such as geographical coordinates or device model.

**Params info:**
| Param        | Required     | Type | Description                                                                                                                                                                                  |
| ------------ | ------------ | ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CoverageID` | yes          | int  | This ID refers to one of the available Everynet network deployments: Brazil, Indonesia, USA, Italy, Spain, UK, ...                                                                           |
| `ClientID`   | yes          | int  | This ID refers to client. This identifier may be obtained automatically from credentials, provided by user.                                                                                  |
| `DevEUI`     | yes          | str  | Hex string, represents 64 bit integer of end-device identifier. This field should be unique for each device.                                                                                 |
| `DevAddr`    | conditional* | str  | Hex string, represent 32 bit int device address. Single `DevAddr` may be assigned to several `DevEUIs` simultaneously. This param must be passed only for ABP devices.                       |
| `JoinEUI`    | conditional* | str  | Hex string, represents 64 bit int unique `JoinEUI`. This param must be passed only for OTAA devices.                                                                                         |
| `Details`    | no           | str  | Some JSON in string form. This field provides additional device information, such as geographical coordinates or device model. Size of data, passed as this param, may be limited by server. |

`*` Only one of `DevAddr` or `JoinEUI` shall be provided, providing both values will result in an error.

#### Example

Create device with a known DevEUI and JoinEUI:

```json
{
    "DevEUI": "ffffffffffffffff",
    "JoinEUI": "ffffffffffffffff"
}
```

```bash
$ curl -s --request POST 'https://eu.cloud.everynet.io/api/v1/devices/insert' \
--header 'Authorization: Bearer secrettoken' \
--header 'Content-Type: application/json' \
--data-raw '{
"DevEUI":  "ffffffffffffffff",
"JoinEUI": "ffffffffffffffff"
}' | jq

{
  "DevEUI": "ffffffffffffffff",
  "JoinEUI": "ffffffffffffffff",
  "ActiveDevAddr": null,
  "TargetDevAddr": null,
  "Details": null,
  "CreatedAt": "2022-05-31T07:12:02.875460"
}
```

Create device with a known DevAddr:

```json
{
    "DevEUI": "ffffffffffffffff",
    "DevAddr": "ffffffff"
}
```

```bash
$ curl -s --request POST 'https://ranapi.everynet.com/api/v1.0/devices/insert' \
--header 'Authorization: Bearer secrettoken' \
--header 'Content-Type: application/json' \
--data-raw '{
"DevEUI": "ffffffffffffffff",
"DevAddr": "ffffffff"
}' | jq
{
  "DevEUI": "ffffffffffffffff",
  "JoinEUI": null,
  "ActiveDevAddr": "ffffffff",
  "TargetDevAddr": null,
  "Details": null,
  "CreatedAt": "2022-05-31T07:14:04.473749"
}
```

### Devices.Update

Update device subscription in a routing table by `DevEUI`. This API function is intended to be used by the LNS to set new device address after the join request has been processed. Optional parameters that are ommited won't be updated, `null` values are not allowed.

Parameters `ActiveDevAddr` and `TargetDevAddr` are used to handle two security contexts during the join procedure. The security-context is only switched after the device sends its first uplink with `TargetDevAddr`. It is valid for both LoRaWAN 1.0.x and 1.1.

At least one of `ActiveDevAddr` or `TargetDevAddr` values must be provided.

**Params info:**
| Param           | Required    | Type | Description                                                                                                                                                                                     |
| --------------- | ----------- | ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CoverageID`    | yes         | int  | This ID refers to one of the available Everynet network deployments: Brazil, Indonesia, USA, Italy, Spain, UK, ...                                                                              |
| `ClientID`      | yes         | int  | This ID refers to client.                                                                                                                                                                       |
| `DevEUI`        | yes         | str  | Hex string representing `DevEUI` of the updated device. Returns an error if device is missing from the routing table.                                                                           |
| `JoinEUI`       | yes         | str  | Hex string representing `JoinEUI` of the updated device.                                                                                                                                        |
| `ActiveDevAddr` | conditional | str  | Hex string, represent 32 bit int device address. Used to update current device address.                                                                                                         |
| `TargetDevAddr` | conditional | str  | Hex string, represent 32 bit int device address. This DevAddr was generated and sent to device by Join Server (via Join Accept), but NS is not informed about this change right now (by Uplink) |

#### Example

```json
{
    "DevEUI": "ffffffffffffffff",
    "JoinEUI": "ffffffffffffffff",
    "TargetDevAddr": "ffffffff"
}
```

```bash
$ # This call is not required, here is just for example to see the difference after update
$ curl -s --request GET 'https://eu.cloud.everynet.io/api/v1/devices/select?DevEUIs=ffffffffffffffff' \
--header 'Authorization: Bearer secrettoken' \
--header 'Content-Type: application/json' | jq
[
  {
    "DevEUI": "ffffffffffffffff",
    "JoinEUI": "ffffffffffffffff",
    "ActiveDevAddr": null,
    "TargetDevAddr": null,
    "Details": null,
    "CreatedAt": "2022-05-31T07:20:00.360463"
  }
]

$ curl -s --request POST 'https://eu.cloud.everynet.io/api/v1/devices/update' \
--header 'Authorization: Bearer secrettoken' \
--header 'Content-Type: application/json' \
--data-raw '{
"DevEUI": "ffffffffffffffff",
"JoinEUI": "ffffffffffffffff",
"TargetDevAddr": "ffffffff"
}' | jq

{
  "DevEUI": "ffffffffffffffff",
  "JoinEUI": "ffffffffffffffff",
  "ActiveDevAddr": null,
  "TargetDevAddr": "ffffffff",
  "Details": null,
  "CreatedAt": "2022-05-31T07:20:00.360463"
}
```

### Devices.Drop

Deletes device subscription from the routing table by `DevEUI`.

**Params info:**
| Param        | Required | Type  | Description                                                                                                                    |
| ------------ | -------- | ----- | ------------------------------------------------------------------------------------------------------------------------------ |
| `CoverageID` | yes      | int   | This ID refers to one of the available Everynet network deployments: Brazil, Indonesia, USA, Italy, Spain, UK, ...             |
| `ClientID`   | yes      | int   | This ID refers to the client. This identifier may be obtained automatically from credentials, provided by user.                |
| `DevEUIs`    | no       | str[] | List of `DevEUI` represented as hex strings. Subscription info about devices from the list will be deleted from routing table. |

#### Example

```json
{
    "DevEUIs": [
        "ffffffffffffffff"
    ]
}
```

```bash
$ curl -s --request POST 'https://eu.cloud.everynet.io/api/v1/devices/drop' \
--header 'Authorization: Bearer secrettoken' \
--header 'Content-Type: application/json' \
--data-raw '{"DevEUIs": ["ffffffffffffffff"]}' | jq

{
  "deleted": 1
}
```

### Devices.DropAll

Deletes all device subscriptions from the routing table.

**Params info:**
| Param        | Required | Type | Description                                                                                                        |
| ------------ | -------- | ---- | ------------------------------------------------------------------------------------------------------------------ |
| `CoverageID` | yes      | int  | This ID refers to one of the available Everynet network deployments: Brazil, Indonesia, USA, Italy, Spain, UK, ... |
| `ClientID`   | yes      | int  | This ID refers to client. This identifier may be obtained automatically from credentials, provided by user.        |

#### Example

```bash
$ curl -s --request POST 'https://eu.cloud.everynet.io/api/v1/devices/drop-all' \
--header 'Authorization: Bearer secrettoken' \
--header 'Content-Type: application/json' \

{
  "deleted": 10
}
```

---

## Multicast Management

This API provides management multicast group (create, delete) and management of devices in those groups (add, remove).

To start sending multicast downlinks AS must create multicast group and add devices to this group and then send multicast-downlink.

The methods are HTTP-based and are not available via WebSocket interface.

| Method                                                                                   | Description                                                                                                                                               |
| ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [create_multicast_group(name, addr)](#create-multicast-group)                            | This method provides to create multicast group.                                                                                                           |
| [delete_multicast_groups(List[addr])](#Delete-multicast-group)                           | This method provides to remove multicast group by group addr.                                                                                             |
| [get_multicast_groups(List[addr])](#get-multicast-groups)                                | This method provides to show multicast groups with all devices by specified list of addrs, if list is empty then this method return all multicast groups. |
| [add_device_to_multicast_group(addr, dev_eui)](#add-device-to-multicast-group)           | This method provides to add the device to a specified multicast group.                                                                                    |
| [remove_device_from_multicast_group(addr, dev_eui)](#remove-device-from-multicast-group) | This method provides to remove the device from a specified multicast group.                                                                               |

### Create multicast group

Creates new multicast group.

If multicast group with this `addr` already exists, will return error.

**Params info:**
| Param  | Required | Type | Description                                                                                    |
| ------ | -------- | ---- | ---------------------------------------------------------------------------------------------- |
| `name` | yes      | str  | Multicast group name                                                                           |
| `addr` | yes      | str  | Hex string representing address of multicast group. This address may be used to send downlinks |

#### Example

```bash
curl -X 'POST' \
  'https://eu.cloud.everynet.io/api/v1/multicast/multicast-groups/create' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "My first multicast group",
  "addr": "dafa0c11"
}' | jq

{
  "addr": "dafa0c11",
  "created_at": "2022-08-26T13:35:30.969638",
  "name": "My first multicast group",
  "devices": []
}
```

### Delete multicast groups

Deletes multicast groups.

If no multicast group with this `addr` exists, it won't be deleted.

**Params info:**
| Param   | Required | Type  | Description                                                                            |
| ------- | -------- | ----- | -------------------------------------------------------------------------------------- |
| `addrs` | yes      | str[] | Array of hex string, each represents address of multicast group, which must be deleted |

#### Example

```bash
curl -s -X 'POST' \
  'https://eu.cloud.everynet.io/api/v1/multicast/multicast-groups/delete' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
  "addrs": ["dafa0c11"]
}' | jq

{
  "deleted": 1
}
```

### Get multicast groups

Get info about multicast groups.

If empty `addrs` provided, will return all multicast groups you have.

**Params info:**
| Param   | Required | Type  | Description                                                    |
| ------- | -------- | ----- | -------------------------------------------------------------- |
| `addrs` | yes      | str[] | Aray of hex strings, each represents multicast group addresses |

#### Example

```bash
curl -s -X 'POST' \
  'https://eu.cloud.everynet.io/api/v1/multicast/multicast-groups/get' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
  "addrs": []
}' | jq

[
  {
    "created_at": "2022-08-26T13:35:30.969638",
    "name": "My first multicast group",
    "addr": "dafa0c11",
    "devices": [
      "fafafafafafafafa",
      "fafafafafafafafb",
      "fafafafafafafafc"
    ]
  }
```

### Add device to multicast group

Adds device to multicast group.

If no multicast group with this `addr` exists, will return error.

If not device with this `dev_eui` exists, will return error.

**Params info:**
| Param     | Required | Type | Description                                                                             |
| --------- | -------- | ---- | --------------------------------------------------------------------------------------- |
| `addr`    | yes      | str  | Hex string representing address of multicast group you want to use to add device in it. |
| `dev_eui` | yes      | str  | Hex string representing dev_eui of device you want to add to multicast group.           |

#### Example

```bash
curl -s -X 'POST' \
  'https://eu.cloud.everynet.io/api/v1/multicast/multicast-groups/add-device' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
  "addr": "dafa0c11",
  "dev_eui": "fafafafafafafafa"
}' | jq

{
  "is_added": true
}
```

### Remove device from multicast group

Removes device from multicast group.

If no multicast group with this `addr` exists, will return `{"is_removed": false}`.

If not device with this `dev_eui` exists, will return `{"is_removed": false}`.

**Params info:**
| Param     | Required | Type | Description                                                                               |
| --------- | -------- | ---- | ----------------------------------------------------------------------------------------- |
| `addr`    | yes      | str  | Hex string representing address of multicast group you want to use to remove device from. |
| `dev_eui` | yes      | str  | Hex string representing dev_eui of device you want to remove from multicast group.        |

#### Example

```bash
curl -s -X 'POST' \
  'https://eu.cloud.everynet.io/api/v1/multicast/multicast-groups/remove-device' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
  "addr": "dafa0c11",
  "dev_eui": "fafafafafafafafa"
}' | jq

{
  "is_removed": true
}
```

## Message Streaming

Message streams are available through a RAN Streaming API, implemented as single websocket at URL:

- `wss://.../api/v1/gateway/`

For example, for EU region:

- `wss://eu.cloud.everynet.io/api/v1/gateway/`

To interact with both upstream and downstream messages, you should connect to the RAN Streaming API.

LNS must handle Websocket ping-pong functionality properly.

## Upstream Traffic = Uplinks + Join Requests

Everynet RAN considers both join requests and uplinks as part of the upstream traffic without any distinction.

For billing purposes, the LNS must either acknowledge or reject every upstream message based on the [MIC challenge procedure](#mic-challenge-procedure).

### Messages from RAN to LNS

| Message           | Description                                   |
| ----------------- | --------------------------------------------- |
| `UpstreamMessage` | Data message received upstream from a device. |

### Messages from LNS to RAN

| Message                 | Description                                                                                               |
| ----------------------- | --------------------------------------------------------------------------------------------------------- |
| `UpstreamAckMessage`    | Confirmation that the LNS has received the `UpstreamMessage` and successfully resolved the MIC-challenge. |
| `UpstreamRejectMessage` | Notification that the LNS received the `UpstreamMessage` but failed to solve the MIC-challenge.           |

### MIC Challenge Procedure

To ensure the integrity of upstream traffic and for billing purposes, the LNS is required to either acknowledge or reject each upstream message.

**Acknowledgment procedure is designed this way to let RAN check whether LNS has correct device keys in possession without revealing these keys.**

Procedure for the LNS:

1. Upon receiving an `UpstreamMessage`, compute the MIC using the `phy_payload_no_mic` data and the `NwkKey`/`NwkSKey` stored at the LNS.
2. Verify that the received `mic_challenge` contains the computed MIC.
3. If correct, send the `UpstreamAckMessage` to RAN (with calculated MIC in the `mic` field); if not, send the `UpstreamRejectMessage`.

The size of the `mic_challenge` field varies from 2 to 4096 and is reduced with every successful mic challenge solved and acknowledged with `UpstreamAckMessage`.

Failure to follow this procedure may lead to the LNS being unsubscribed from the device's traffic. This MIC challenge procedure is patent-pending.

## Downstream Traffic = Downlinks + Join Accepts

Everynet RAN classifies both join accepts and downlinks as downstream traffic without distinction.

### Messages from LNS to RAN

| Message                      | Description                                                       |
| ---------------------------- | ----------------------------------------------------------------- |
| `DownstreamMessage`          | LNS request to schedule a downlink to a device.                   |
| `MulticastDownstreamMessage` | LNS request to schedule a multicast downlink to multiple devices. |

Note: Only devices with an active subscription can receive downstream messages. All other messages will be discarded.

### Messages from RAN to LNS

| Message                            | Description                                                                                                                                      |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `DownstreamAckMessage`             | Confirmation that the Streaming API server received and scheduled a downstream message for processing. Same for multicast and regular downlinks. |
| `DownstreamResultMessage`          | Notification that the Streaming API server completed processing the downstream message.                                                          |
| `MulticastDownstreamResultMessage` | Notification that the Streaming API server processed a multicast downstream message.                                                             |

## Message Types

### Message Types Overview

The communication between the RAN and LNS is facilitated using two primary message types: `FromRAN` and `FromLNS`. These messages encapsulate various sub-messages specific to direction of interaction.

#### `FromRAN` Message Breakdown

This message type represents communication from the RAN to the LNS.

- **UpstreamMessage**: Represents data received upstream from a device.
- **DownstreamAckMessage**: A confirmation indicating that the Streaming API server received a downstream message and scheduled it for processing.
- **DownstreamResultMessage**: A notification indicating the completion status of a processed downstream message.
- **MulticastDownstreamResultMessage**: A notification detailing the outcome of a multicast downstream message processed by the Streaming API server.

#### `FromLNS` Message Breakdown

This message type represents communication from the LNS to the RAN.

- **UpstreamAckMessage**: Serves as an acknowledgment that the LNS received an `UpstreamMessage` and successfully resolved the MIC-challenge.
- **UpstreamRejectMessage**: Indicates that the LNS received the `UpstreamMessage` but failed to solve the MIC-challenge.
- **DownstreamMessage**: A message request from the LNS, instructing the scheduling of a downlink to a device.
- **MulticastDownstreamMessage**: A request from the LNS aiming to schedule a multicast downlink for a group of devices.

### UpstreamMessage

**Direction**: `FromRAN`

| Field                | Type                                                                                   | Mandatory | Description                                                                                                                                             |
| -------------------- | -------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `protocol_version`   | uint32                                                                                 | True      | RAN protocol version                                                                                                                                    |
| `transaction_id`     | bytes                                                                                  | True      | Unique ID (UUID). This field is used by the `UpstreamAckMessage` or `UpstreamRejectMessage` to identify corresponding `UpstreamMessage` to acknowledge. |
| `dev_euis`           | uint64[]                                                                               | True      | List of LoRaWAN DevEUIs potentially associated with the upstream message                                                                                |
| `radio`              | oneof [LoRaUpstreamRadio](#loraupstreamradio) or [FSKUpstreamRadio](#fskupstreamradio) | True      | Radio data                                                                                                                                              |
| `phy_payload_no_mic` | bytes                                                                                  | True      | PHYPayload with detached MIC                                                                                                                            |
| `mic_challenge`      | [fixed32[]](https://protobuf.dev/programming-guides/proto3/#scalar)                    | True      | List of MICs for upstream traffic                                                                                                                       |
| `position`           | [Position](#position)                                                                  | False     | Geographical position of the device                                                                                                                     |

#### Example

```
protocol_version: 1
transaction_id: "42b7ef6a-7bc7-4672-9489-eb39e37fbd21"
dev_euis: [8844537008791951183, 8844537008791951184]
radio {
  lora {
    frequency: 868100000
    spreading: 12
    bandwidth: 125000
    rssi: -52
    snr: -3.0
  }
}
phy_payload_no_mic: "EFGH5678"
mic_challenge: [1308830714, 114830713, 170883]
position {
  lat: 51.509865
  lng: -0.118092
  alt: 35.0
}
```

---

### UpstreamAckMessage

**Direction**: `FromLNS`

| Field              | Type   | Mandatory | Description                                                                             |
| ------------------ | ------ | --------- | --------------------------------------------------------------------------------------- |
| `protocol_version` | uint32 | True      | RAN protocol version                                                                    |
| `transaction_id`   | bytes  | True      | ID (UUID) of the corresponding `UpstreamMessage`                                        |
| `dev_eui`          | uint64 | True      | DevEUI of the device associated with the upstream message                               |
| `mic`              | uint32 | True      | LNS-calculated MIC according to the [MIC challenge procedure](#mic-challenge-procedure) |

#### Example

```
protocol_version: 1
transaction_id: "42b7ef6a-7bc7-4672-9489-eb39e37fbd21"
dev_eui: 8844537008791951183
mic: 1308830714
```

---

### UpstreamRejectMessage

**Direction**: `FromLNS`

| Field              | Type                           | Mandatory | Description                                                                  |
| ------------------ | ------------------------------ | --------- | ---------------------------------------------------------------------------- |
| `protocol_version` | uint32                         | True      | RAN protocol version                                                         |
| `transaction_id`   | bytes                          | True      | ID (UUID) of the corresponding `UpstreamMessage`                             |
| `result_code`      | RejectResultCode (_see below_) | True      | Code indicating the reason why the upstream was rejected                     |
| `result_message`   | string                         | Optional  | An optional message providing a human-readable explanation for the rejection |

**RejectResultCode**:

| Result Code  | Value |
| ------------ | ----- |
| `SUCCESS`    | 0     |
| `MIC_FAILED` | 1     |


#### Example

```
protocol_version: 1
transaction_id: "42b7ef6a-7bc7-4672-9489-eb39e37fbd21"
result_code: MIC_FAILED
result_message: "Upstream MIC challenge failed"
```

---

### DownstreamMessage

**Direction**: `FromLNS`

| Field              | Type                                      | Mandatory | Description                                                                                                                                                                                                                                                                           |
| ------------------ | ----------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `protocol_version` | uint32                                    | True      | RAN protocol version                                                                                                                                                                                                                                                                  |
| `transaction_id`   | bytes                                     | True      | Unique ID (UUID) for the downstream message. Will be to associated with `DownstreamAck` and `DownstreamResult` feedback messages later.                                                                                                                                               |
| `dev_eui`          | uint64                                    | True      | DevEUI of the device                                                                                                                                                                                                                                                                  |
| `target_dev_addr`  | uint32                                    | Optional  | Target device address, optional. Mandatory for join accept messages. In the due course of the Join procedure device may end up having two device addresses: device address that was active before join procedure has started and target device address provided in JoinAccept message |
| `tx_window`        | [TransmissionWindow](#transmissionwindow) | True      | The transmission window for the downstream message                                                                                                                                                                                                                                    |
| `phy_payload`      | bytes                                     | True      | The physical payload of the downstream message                                                                                                                                                                                                                                        |

#### Example

```
protocol_version: 1
transaction_id: "e2321bba-52af-4cd7-8d45-081a16923e15"
dev_eui: 8844537008791951183
target_dev_addr {
  value: 4294967295
}
tx_window {
  radio {
    lora {
      frequency: 868100000
      spreading: 12
      bandwidth: 125000
      power: -3
    }
  }
  timing {
    deadline: 1234567890123
  }
}
phy_payload: "IJKL9012"
```

### MulticastDownstreamMessage

**Direction**: `FromLNS`

| Field              | Type                                                        | Mandatory | Description                                                                                                                                                |
| ------------------ | ----------------------------------------------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `protocol_version` | uint32                                                      | True      | RAN protocol version                                                                                                                                       |
| `transaction_id`   | bytes                                                       | True      | Unique ID (UUID) for the multicast downstream message. Will be to associated with `DownstreamAck` and `MulticastDownstreamResult` feedback messages later. |
| `addr`             | uint32                                                      | True      | Address of the multicast group                                                                                                                             |
| `tx_window`        | [MulticastTransmissionWindow](#multicasttransmissionwindow) | True      | The transmission window for the multicast downstream message                                                                                               |
| `phy_payload`      | bytes                                                       | True      | The physical payload of the multicast downstream message                                                                                                   |

#### Example

```
protocol_version: 1
transaction_id: "87f6aa33-3144-452f-84e4-19ca26552f73"
addr: 12345678
tx_window {
  radio {
    lora {
      frequency: 868100000
      spreading: 12
      bandwidth: 125000
      power: -3
    }
  }
  timing {
    tmms_list: {
      tmms: [1234567890123, 1234567890456]
    }
  }
}
phy_payload: "QRST7890"
```

---

### DownstreamAckMessage

**Direction**: `FromRAN`

| Field              | Type   | Mandatory | Description                                               |
| ------------------ | ------ | --------- | --------------------------------------------------------- |
| `protocol_version` | uint32 | True      | RAN protocol version                                      |
| `transaction_id`   | bytes  | True      | Transaction ID for the downstream acknowledgement message |
| `mailbox_id`       | uint64 | True      | Mailbox ID associated with the downstream acknowledgement |

#### Example

```
protocol_version: 1
transaction_id: "e2321bba-52af-4cd7-8d45-081a16923e15"
mailbox_id: 1234567890
```

### DownstreamResultMessage

**Direction**: `FromRAN`

| Field              | Type                                          | Mandatory | Description                                     |
| ------------------ | --------------------------------------------- | --------- | ----------------------------------------------- |
| `protocol_version` | uint32                                        | True      | RAN protocol version                            |
| `transaction_id`   | bytes                                         | True      | Transaction ID for the downstream result        |
| `result_code`      | [DownstreamResultCode](#downstreamresultcode) | True      | The result code for the downstream transmission |
| `result_message`   | string                                        | True      | Detailed result message                         |
| `mailbox_id`       | uint64                                        | True      | Mailbox ID associated with the result           |

#### Example

```
protocol_version: 1
transaction_id: "e2321bba-52af-4cd7-8d45-081a16923e15"
result_code: WindowNotFound
result_message: "Transmission window not found."
mailbox_id: 1234567890
```

---

### MulticastDownstreamResultMessage

**Direction**: `FromRAN`

| Field              | Type                                              | Mandatory | Description                                            |
| ------------------ | ------------------------------------------------- | --------- | ------------------------------------------------------ |
| `protocol_version` | uint32                                            | True      | RAN protocol version                                   |
| `transaction_id`   | bytes                                             | True      | Transaction ID for the multicast downstream result     |
| `results`          | [MulticastDeviceResult[]](#multicastdeviceresult) | True      | List of results for each device in the multicast group |
| `mailbox_id`       | uint64                                            | True      | Mailbox ID associated with the multicast result        |

#### Example

```
protocol_version: 1
transaction_id: "87f6aa33-3144-452f-84e4-19ca26552f73"
results: [
  {
    dev_eui: 8844537008791951183
    result_code: WindowNotFound
    result_message: "Transmission window not found"
  }
]
mailbox_id: 1234567890
```

---

## Models

### Position

| Field | Type  | Mandatory | Description               |
| ----- | ----- | --------- | ------------------------- |
| `lat` | float | True      | Latitude of the position  |
| `lng` | float | True      | Longitude of the position |
| `alt` | float | Optional  | Altitude of the position  |

#### Example

```
lat: 51.509865
lng: -0.118092
alt: 35.0
```

---

### LoRaUpstreamRadio

| Field       | Type   | Mandatory | Description                        |
| ----------- | ------ | --------- | ---------------------------------- |
| `frequency` | uint32 | True      | Radio frequency                    |
| `spreading` | uint32 | True      | Spreading factor                   |
| `bandwidth` | uint32 | True      | Bandwidth of the radio channel     |
| `rssi`      | int32  | True      | Received Signal Strength Indicator |
| `snr`       | float  | True      | Signal to Noise Ratio              |

#### Example

```
frequency: 868100000
spreading: 12
bandwidth: 125000
rssi: -52
snr: -3.0
```

---

### FSKUpstreamRadio

| Field       | Type   | Mandatory | Description                        |
| ----------- | ------ | --------- | ---------------------------------- |
| `frequency` | uint32 | True      | Radio frequency                    |
| `bit_rate`  | uint32 | True      | Data transmission rate             |
| `rssi`      | int32  | True      | Received Signal Strength Indicator |

#### Example

```
frequency: 868100000
bit_rate: 50000
rssi: -52
```

---

### DownstreamResultCode

| Result Code       | Value |
| ----------------- | ----- |
| `Success`         | 0     |
| `WindowNotFound`  | 1     |
| `GatewayNotFound` | 2     |
| `TooLate`         | 3     |
| `NoAck`           | 4     |
| `GatewayError`    | 5     |

---

### LoRaDownstreamRadio

| Field       | Type   | Mandatory | Description                        |
| ----------- | ------ | --------- | ---------------------------------- |
| `frequency` | uint32 | True      | Radio frequency                    |
| `spreading` | uint32 | True      | Spreading factor                   |
| `bandwidth` | uint32 | True      | Bandwidth of the radio channel     |
| `power`     | int32  | True      | Power setting for the transmission |

#### Example

```
frequency: 868100000
spreading: 12
bandwidth: 125000
power: -3
```

---

### FSKDownstreamRadio

| Field                 | Type   | Mandatory | Description                        |
| --------------------- | ------ | --------- | ---------------------------------- |
| `frequency`           | uint32 | True      | Radio frequency                    |
| `frequency_deviation` | uint32 | True      | Frequency deviation                |
| `bit_rate`            | uint32 | True      | Data transmission rate             |
| `power`               | int32  | True      | Power setting for the transmission |

#### Example

```
frequency: 868100000
frequency_deviation: 5000
bit_rate: 50000
power: -3
```

---

### TransmissionWindow

| Field    | Type                                                                                           | Mandatory | Description                             |
| -------- | ---------------------------------------------------------------------------------------------- | --------- | --------------------------------------- |
| `radio`  | oneof [LoRaDownstreamRadio](#loradownstreamradio) or [FSKDownstreamRadio](#fskdownstreamradio) | True      | Radio data for the transmission window  |
| `timing` | oneof `delay` (uint32), `tmmslist` ([TMMSList](#tmmslist)) or `deadline` (uint64)              | True      | Timing data for the transmission window |

#### Example

```
radio {
  lora {
    frequency: 868100000
    spreading: 12
    bandwidth: 125000
    power: -3
  }
}
timing {
  delay: 1
}
```

---

### MulticastTransmissionWindow

| Field    | Type                                                                                           | Mandatory | Description                                       |
| -------- | ---------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------- |
| `radio`  | oneof [LoRaDownstreamRadio](#loradownstreamradio) or [FSKDownstreamRadio](#fskdownstreamradio) | True      | Radio data for the multicast transmission window  |
| `timing` | oneof `tmmslist` ([TMMSList](#tmmslist)) or `deadline` (uint64)                                | True      | Timing data for the multicast transmission window |

#### Example

```
radio {
  fsk {
    frequency: 868100000
    frequency_deviation: 5000
    bit_rate: 50000
    power: -3
  }
}
timing {
  tmms_list: {
    tmms: [1234567890123, 1234567890456]
  }
}
```

---

### MulticastDeviceResult

| Field            | Type                                          | Mandatory | Description                                      |
| ---------------- | --------------------------------------------- | --------- | ------------------------------------------------ |
| `dev_eui`        | uint64                                        | True      | DevEUI of the device                             |
| `result_code`    | [DownstreamResultCode](#downstreamresultcode) | True      | The result code for the multicast transmission   |
| `result_message` | string                                        | True      | Detailed result message for the multicast device |

#### Example

```
dev_eui: 8844537008791951183
result_code: WindowNotFound
result_message: "Transmission window not found."
```

---

### TMMSList

| Field  | Type     | Mandatory | Description         |
| ------ | -------- | --------- | ------------------- |
| `tmms` | uint64[] | True      | List of TMMS values |

#### Example

```
tmms_list: {
  tmms: [1234567890123, 1234567890456]
}
```

---
