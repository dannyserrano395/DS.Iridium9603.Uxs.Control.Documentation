Uxs.Control Architecture
========================

Overview
--------

This document describes the architecture of the Uxs.Control subsystem.

The Uxs.Control subsystem provides bidirectional communication between
QGroundControl and the Uxs.Vehicle subsystem over an Iridium Short Burst
Data (SBD) datalink.

The subsystem integrates with QGroundControl through a custom
communication link using the UDP transport protocol.

The subsystem manages MAVLink message processing, Iridium Short Burst
Data (SBD) datalink establishment, message filtering, and message
transfer status reporting.

-------------------------------------------------------------------------------

High-Level Architecture
-----------------------

                  +---------------------+
                  |   QGroundControl    |
                  |                     |
                  | +-----------------+ |
                  | |   Custom Comm   | |
                  | |      Link       | |
                  | +--------+--------+ |
                  +----------+----------+
                             |
                             v
                       +-----------+
                       | UdpClient |
                       +-----+-----+
                             |
                             v
                       +------------+
                       |  Control   |
                       +-----+------+
                             |
               +-------------+-------------+
               |                           |
               v                           v
    +------------------+             +-----------+
    |  Ground Control  |             | Satellite |
    | Station Workers  |             |  Workers  |
    +------------------+             +-----+-----+
                                           |
                                           v
                                    +-------------+
                                    | Iridium9603 |
                                    +------+------+
                                           |
                                           v
                                    +-------------+
                                    |   Iridium   |
                                    |   Network   |
                                    +------+------+
                                           |
                                           v
                                    +-------------+
                                    | Uxs.Vehicle |
                                    +-------------+

-------------------------------------------------------------------------------

Responsibilities
----------------

Control

The Control class implements the subsystem.

Responsibilities include:

- Processing inbound and outbound MAVLink messages.
- Filtering inbound MAVLink messages.
- Establishing Iridium Short Burst Data (SBD) datalinks.
- Processing Iridium Short Burst Data (SBD) Ring Alerts.
- Reporting message transfer failures to QGroundControl.

UdpClient

The UdpClient class provides access to the UDP interface connected
to the custom communication link within QGroundControl.

Responsibilities include:

- Sending MAVLink messages to QGroundControl.
- Receiving MAVLink messages from QGroundControl.

Iridium9603

The Iridium9603 subsystem provides an interface for communicating with
an Iridium 9603 satellite transceiver.

Responsibilities include:

- Managing lifecycle state.
- Processing commands.
- Processing unsolicited result codes (URCs).
- Coordinating access to the transceiver.

-------------------------------------------------------------------------------

Worker Architecture
-------------------

The subsystem uses dedicated asynchronous workers to perform internal
operations.

The following workers are defined:

`GroundControlStationInboundMessageWorkerAsync`

Responsible for receiving MAVLink messages from QGroundControl and
queuing supported messages for transmission.

`GroundControlStationOutboundMessageWorkerAsync`

Responsible for sending MAVLink messages to QGroundControl.

`SatelliteUnsolicitedResultCodeWorkerAsync`

Responsible for processing unsolicited result codes (URCs) received
from the Iridium9603 subsystem.

`SatelliteDatalinkWorkerAsync`

Responsible for establishing Iridium Short Burst Data (SBD) datalinks
and transferring inbound and outbound MAVLink messages.

-------------------------------------------------------------------------------

Ground Control Station Inbound Message Worker
---------------------------------------------

A dedicated worker continuously monitors the UDP interface connected
to the custom communication link within QGroundControl for MAVLink
messages.

The inbound message processing sequence is:

    Wait For UDP Datagram
        ↓
    Receive UDP Datagram
        ↓
    Extract MAVLink Messages
        ↓
    Process MAVLink Messages

Supported MAVLink messages are placed into an internal channel for
processing by the satellite datalink worker.

Unsupported MAVLink messages are not queued for transmission.
A MAVLink STATUSTEXT message is generated to notify QGroundControl
that the MAVLink message is not supported by the subsystem.
The STATUSTEXT message is placed into the outbound message channel
for processing by the ground control station outbound message worker.

-------------------------------------------------------------------------------

Ground Control Station Outbound Message Worker
----------------------------------------------

A dedicated worker continuously monitors the outbound message channel
and sends queued MAVLink messages to QGroundControl through the UDP
interface.

The outbound message processing sequence is:

    Wait For Message
        ↓
    Read Queued Message
        ↓
    Process Message
        ↓
    Send Message To QGroundControl

MAVLink messages received from the Uxs.Vehicle subsystem are placed
into an internal channel by the satellite datalink worker for
processing by the ground control station outbound message worker.

MAVLink STATUSTEXT messages generated by the Uxs.Control subsystem
are also placed into the outbound message channel.

-------------------------------------------------------------------------------

Satellite Datalink Worker
-------------------------

A dedicated worker continuously manages datalink establishment
and the transfer of inbound and outbound MAVLink messages.

An Iridium Short Burst Data (SBD) datalink is established when:

- A supported MAVLink message is received from QGroundControl.
- A Short Burst Data (SBD) Ring Alert is received.
- The configured datalink establishment interval expires.

The datalink establishment sequence is:

    Wait To Establish Datalink
        ↓
    Prepare Mobile Originated Message
        ↓
    Establish Datalink
        ↓
    Process Mobile Originated Transfer Status
        ↓
    Process Mobile Terminated Transfer Status
        ↓
    Clear Mobile Originated And Mobile Terminated Buffers

MAVLink messages queued for transmission to the Uxs.Vehicle subsystem
are prepended with the destination RockBLOCK serial number using the
RockBLOCK direct messaging format.

If a Short Burst Data (SBD) Ring Alert is received, the AT+SBDIXA command
is used to establish the datalink; otherwise, the AT+SBDIX command is used.

If a Mobile Originated transfer is unsuccessful, a MAVLink STATUSTEXT
message is generated to notify QGroundControl that the MAVLink message
failed to transfer to the Iridium Gateway. The STATUSTEXT message is
placed into the outbound message channel for processing by the ground
control station outbound message worker.

Mobile Terminated messages are retrieved from the transceiver
following a successful Mobile Terminated transfer and placed
into the outbound message channel for processing by the ground
control station outbound message worker.

-------------------------------------------------------------------------------

Satellite Unsolicited Result Code Worker
----------------------------------------

A dedicated worker continuously monitors the Iridium9603 subsystem
for unsolicited result codes (URCs).

The unsolicited result code processing sequence is:

    Wait For URC
        ↓
    Identify SBD Ring Alert
        ↓
    Queue SBD Ring Alert

Short Burst Data (SBD) Ring Alerts are placed into an internal
channel for processing by the satellite datalink worker.

An interval of ten seconds is enforced between queued Short
Burst Data (SBD) Ring Alerts.

-------------------------------------------------------------------------------

MAVLink Message Filtering
-------------------------

The subsystem filters MAVLink messages received from QGroundControl
to restrict the messages transmitted over the Iridium Short Burst
Data (SBD) datalink.

The following MAVLink message types are supported:

- COMMAND_LONG
- COMMAND_INT
- MISSION_ITEM_INT
- MISSION_SET_CURRENT

COMMAND_LONG and COMMAND_INT messages are filtered according to
the command contained within the message.

The following MAVLink commands are supported within COMMAND_LONG
and COMMAND_INT messages:

    Command Identifier    Command
    ------------------    --------------------------------
    20                    MAV_CMD_NAV_RETURN_TO_LAUNCH
    21                    MAV_CMD_NAV_LAND
    22                    MAV_CMD_NAV_TAKEOFF
    84                    MAV_CMD_NAV_VTOL_TAKEOFF
    85                    MAV_CMD_NAV_VTOL_LAND
    176                   MAV_CMD_DO_SET_MODE
    178                   MAV_CMD_DO_CHANGE_SPEED
    183                   MAV_CMD_DO_SET_SERVO
    192                   MAV_CMD_DO_REPOSITION
    208                   MAV_CMD_DO_PARACHUTE
    300                   MAV_CMD_MISSION_START
    400                   MAV_CMD_COMPONENT_ARM_DISARM
    2600                  MAV_CMD_CONTROL_HIGH_LATENCY

MISSION_ITEM_INT and MISSION_SET_CURRENT messages are not subject
to command filtering.

-------------------------------------------------------------------------------

Message Transfer Status Reporting
---------------------------------

The subsystem uses MAVLink STATUSTEXT messages to report message
transfer failures to QGroundControl.

MAVLink STATUSTEXT messages are generated under the following
conditions:

- A MAVLink message received from QGroundControl is not supported
  by the subsystem.
- A Mobile Originated message fails to transfer from the
  transceiver to the Iridium Gateway.

MAVLink STATUSTEXT messages generated by the subsystem use the
following identifiers and severity:

    System Identifier       254
    Component Identifier    1
    Message Identifier      253
    Severity                4 (Warning)

Generated MAVLink STATUSTEXT messages are placed into the outbound
message channel for processing by the ground control station
outbound message worker.

-------------------------------------------------------------------------------

Lifecycle Management
--------------------

The subsystem exposes two lifecycle operations:

    RunAsync(...)
    StopAsync(...)

RunAsync initializes the UDP interface associated with the custom
communication link within QGroundControl, establishes a connection
to the Iridium9603 subsystem, starts the workers, and waits for
the workers to complete.

StopAsync initiates cancellation of the workers and waits for
the workers to complete.

A shared CancellationTokenSource is used to coordinate cancellation
of workers.

When a worker terminates, cancellation of the remaining workers
is initiated through the shared CancellationTokenSource.

`Exception Propagation`

Unhandled exceptions that occur within workers are propagated to
the consumer after all workers have completed.

If a single unhandled exception occurs, the original exception
is propagated. If multiple unhandled exceptions occur, an
AggregateException containing the exceptions is propagated.

-------------------------------------------------------------------------------

Disposal
--------

The subsystem supports deterministic disposal of internal resources.

Consumers should call StopAsync and wait for it to complete before
disposing the subsystem.

The subsystem disposes the UdpClient associated with the custom
communication link within QGroundControl and the Iridium9603
subsystem.

The CancellationTokenSource is disposed only if execution has been
initiated and StopAsync has completed the shutdown of the subsystem.

Otherwise, the subsystem does not dispose the CancellationTokenSource.
This prevents the CancellationTokenSource from being disposed while
it may still be in use. In this case, the CancellationTokenSource is
reclaimed by the garbage collector when it is no longer referenced.