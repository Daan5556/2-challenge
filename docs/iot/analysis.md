# IoT analysis

**Author:** Daan Eggen  
**Date:** 13/06/2026  
**Version:** 1.0

---

In this document, I will perform an analysis for the user story `US-EMB-01`:

| US-EMB-01 - Reserved Table Light | As restaurant staff, I want the embedded table light to turn red when a table is reserved, so that I can quickly see which tables are not available. |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |

This document describes the architecture choice, the MQTT communication setup,
implementation risks and the validation approach for the embedded table light.

## Context

The reservation system is the source of truth for table availability. When a
table has an active reservation, the restaurant staff and customers should be
able to see this directly from the physical table light. The embedded device
should therefore not decide by itself whether a table is reserved. It should
only receive the current state and display it in a clear way.

The main behavior for this user story is:

- when a table becomes reserved, the table light turns red
- when the table is no longer reserved, the table light returns to the available
  state
- when the device reconnects after a restart or network interruption, it
  receives the latest known state again

## Server-side timing or on-premise timing

There are multiple places where the decision can be made to turn a table light
red. The first option is to send reservation data to the embedded device and let
the device calculate the state locally. The second option is to place a local
controller in the restaurant which reads the reservation data and controls all
table lights. The third option is to keep the timing and reservation logic on
the server and let the devices only listen for state changes.

For this project I chose the third option: the server decides when a table light
has to become red.

This keeps the embedded device simple. The device does not need to understand
reservation rules, time zones, booking changes or cancellation logic. It only
needs to connect to MQTT, listen to its own topic, and display the received
state. This reduces the amount of firmware logic and makes it easier to replace
or update devices later.

The server already has access to the reservation data, so it is the best place
to calculate whether a table is currently reserved. When a reservation is
created, changed or cancelled, the server can immediately publish the new table
state. The server can also periodically recalculate the table states, which
protects the system against missed events or incorrect states.

I decided to place a cloud MQTT broker between the server and the edge devices.
The devices keep an outgoing connection to the broker, and the server publishes
messages to that same broker. This means the server does not need to connect
directly to every device in the restaurant network. It also avoids requiring
incoming network access to the restaurant.

Using a broker also keeps the architecture open for future extensions. For
example, the same MQTT connection can later be used to store telemetry such as
device uptime, signal strength, battery level, firmware version or connection
status.

## Architecture

The intended architecture is:

```text
Reservation system
        |
        | publishes table state
        v
Cloud MQTT broker
        |
        | subscribed table topic
        v
Embedded table light
```

The reservation system remains responsible for business logic. The MQTT broker
is responsible for message routing. The embedded table light is responsible only
for displaying the latest received table state.

The device can also publish information back to the broker:

```text
Embedded table light
        |
        | publishes state, status and telemetry
        v
Cloud MQTT broker
        |
        | consumed by server
        v
Reservation system
```

This return flow is not required for the minimum implementation of the user
story, but it is useful for monitoring and future development.

## MQTT

MQTT is a good fit for this feature because it is lightweight and works well for
small devices that need to receive short messages. The publish and subscribe
model also matches the problem well. The server does not need to know whether a
device is directly reachable. It publishes a message to a topic, and the broker
delivers that message to the subscribed device.

Each table light should have its own topic. A possible topic structure is:

```text
restaurants/{restaurantId}/tables/{tableId}/light/set
restaurants/{restaurantId}/tables/{tableId}/light/state
restaurants/{restaurantId}/tables/{tableId}/status
restaurants/{restaurantId}/tables/{tableId}/telemetry
```

The `light/set` topic is used by the server to tell the device what it should
display. The `light/state` topic can be used by the device to confirm the state
that it applied. The `status` topic can show whether a device is online or
offline. The `telemetry` topic can be used later for monitoring data.

An example command payload for a reserved table is:

```json
{
  "table_id": "12",
  "reserved": true,
  "color": "red",
  "reason": "reservation",
  "reservation_id": "res_1024",
  "issued_at": "2026-06-13T18:00:00+02:00",
  "valid_until": "2026-06-13T20:00:00+02:00",
  "sequence": 42
}
```

An example command payload for an available table is:

```json
{
  "table_id": "12",
  "reserved": false,
  "color": "off",
  "reason": "available",
  "reservation_id": null,
  "issued_at": "2026-06-13T20:00:00+02:00",
  "valid_until": null,
  "sequence": 43
}
```

The `sequence` value can be used by the device to ignore old messages. This is
important when a device reconnects or when messages arrive later than expected.
The device should only apply a message when it is newer than the last applied
message.

For the command topic, the broker should retain the latest message. This means
that when a table light restarts, it receives the most recent table state as
soon as it subscribes again. Without retained messages, a restarted device would
need to wait until the next state change before it knows what to display.

QoS 1 is a practical choice for the command messages. It means the message is
delivered at least once. Because the payload contains a full state and a
sequence number, receiving the same command more than once is not a problem.

## Device behavior

The embedded device should follow a simple flow:

1. Connect to Wi-Fi.
2. Connect to the MQTT broker with its own credentials.
3. Subscribe to its `light/set` topic.
4. Receive the retained table state.
5. Apply the requested color.
6. Publish the applied state back to `light/state`.
7. Continue listening for new commands.

The device should not store customer data. It only needs a table identifier, a
color and some technical metadata. This keeps the privacy impact low and avoids
placing reservation details on a small device.

When the connection to MQTT is lost, the device should not silently pretend that
everything is correct forever. A practical approach is to keep the last known
state for a short period, and after that show an offline/error indication. This
makes it visible to staff that the device needs attention.

## Implementation risks

| Risk                                         | Impact                                                   | Mitigation                                                                                       |
| -------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| MQTT broker is unavailable                   | Devices cannot receive new table states                  | Use a reliable broker, monitor broker health and keep retained messages for reconnects           |
| Device loses network connection              | A table light can become stale                           | Publish device status, use Last Will and Testament, and show an offline indication on the device |
| Reservation event is missed                  | The physical light does not match the reservation system | Recalculate and republish table states periodically from the server                              |
| Old message is applied after a newer message | A table can show the wrong color                         | Include `issued_at` and `sequence` fields and ignore older commands                              |
| Wrong device is linked to a table            | Staff receive incorrect availability information         | Store the table/device mapping centrally and test every device during installation               |
| Broker credentials are reused or leaked      | One device could affect another device                   | Use unique credentials per device and restrict topics with broker ACL rules                      |
| Personal reservation data is sent to devices | Privacy risk                                             | Send only technical state data and avoid customer names or contact details                       |
| Device power failure                         | The light does not show availability                     | Monitor device status and make failures visible in the server dashboard later                    |

## Security

The MQTT connection should use TLS so the commands cannot be read or changed by
someone on the same network. Every device should have its own credentials. This
makes it possible to revoke one device without replacing all other devices.

The broker should also use access control rules. A table light should only be
allowed to subscribe to its own `light/set` topic and publish to its own
`light/state`, `status` and `telemetry` topics. It should not be able to publish
commands to another table.

The payload should not include unnecessary reservation information. For this
user story, the device only needs to know whether it should display red or not.
Reservation IDs can be useful for debugging, but names, phone numbers, e-mail
addresses or other personal information should not be sent to the device.

## Validation

The implementation can be validated with the following checks:

- create a reservation and verify that the matching table light turns red
- cancel or finish a reservation and verify that the light returns to the
  available state
- restart the device and verify that it receives the retained state immediately
- disconnect the device from the network and verify that the server can see the
  device as offline
- publish an older command after a newer command and verify that the device
  ignores it
- test that one device cannot subscribe to or publish on another device's topic

For the server side, the reservation-to-table-state logic should be covered by
automated tests. This is the most important logic in the feature, because the
embedded device only follows the state that the server publishes.

## Conclusion

For `US-EMB-01`, the most suitable design is to keep the reservation timing and
state calculation on the server. The embedded table light should stay simple and
only display the state it receives through MQTT.

Using a cloud MQTT broker separates the reservation system from the restaurant
network and makes the design easier to extend later. The same architecture can
support telemetry and device status reporting without changing the basic
communication model.

The main risks are stale device state, broker availability, incorrect
table-device mapping and security of MQTT topics. These risks can be reduced
with retained messages, QoS 1, sequence numbers, device status messages, unique
credentials and topic access control.
