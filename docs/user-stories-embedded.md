# User Stories: Embedded Target Requirements

## US-01: Reserved Table Light

As restaurant staff, I want the embedded table light to turn red when a table is reserved, so that I can quickly see which tables are not available.

Acceptance criteria:
- The table light receives the table reservation status.
- The light turns red when the table is reserved.
- The light turns off or changes color when the table becomes available.

## US-02: People Counting Sensor

As a restaurant manager, I want embedded sensors to count people entering and leaving, so that the system knows how many people are inside the restaurant.

Acceptance criteria:
- The sensor detects people entering the restaurant.
- The sensor detects people leaving the restaurant.
- The embedded target sends the current occupancy count to the system.

## US-03: Embedded Device Status

As restaurant staff, I want the embedded devices to report their status, so that I know when a light or sensor is not working correctly.

Acceptance criteria:
- Each embedded device sends an online or offline status.
- The device reports errors when it cannot read data or control the light.
- The system can show which embedded device needs attention.
