
DS.Iridium9603.Uxs.Control README
=================================

Overview
--------

DS.Iridium9603.Uxs.Control is a custom Comm Link integration for
QGroundControl that provides bidirectional MAVLink communication with
DS.Iridium9603.Uxs.Vehicle over an Iridium Short Burst Data (SBD) link.

DS.Iridium9603.Uxs.Control enables QGroundControl to transmit commands to
ArduPilot and receive HIGH_LATENCY2 messages without requiring an
Internet connection.

-------------------------------------------------------------------------------

Features
--------

- Custom QGroundControl Comm link.
- Bidirectional MAVLink communication.
- Iridium SBD satellite communication.

-------------------------------------------------------------------------------

Supported MAVLink Messages
--------------------------

- COMMAND_LONG
- COMMAND_INT
- MISSION_ITEM_INT
- MISSION_SET_CURRENT

-------------------------------------------------------------------------------

Supported MAVLink Commands
--------------------------

- MAV_CMD_NAV_RETURN_TO_LAUNCH
- MAV_CMD_NAV_LAND
- MAV_CMD_NAV_TAKEOFF
- MAV_CMD_NAV_VTOL_TAKEOFF
- MAV_CMD_NAV_VTOL_LAND
- MAV_CMD_DO_SET_MODE
- MAV_CMD_DO_CHANGE_SPEED
- MAV_CMD_DO_SET_SERVO
- MAV_CMD_DO_PARACHUTE
- MAV_CMD_MISSION_START
- MAV_CMD_COMPONENT_ARM_DISARM
- MAV_CMD_DO_REPOSITION
- MAV_CMD_CONTROL_HIGH_LATENCY

-------------------------------------------------------------------------------

Communication Flow
------------------

    QGroundControl
          |
          v
    DS.Iridium9603.Uxs.Control
          |
          v
    Iridium Short Burst Data (SBD)
          |
          v
    DS.Iridium9603.Uxs.Vehicle
          |
          v
    ArduPilot

Communication is bidirectional. MAVLink commands are transmitted from
QGroundControl to ArduPilot, while ArduPilot sends HIGH_LATENCY2 messages
to QGroundControl at a configured interval.

-------------------------------------------------------------------------------

Documentation
-------------

Additional documentation is available in the following files:

- Architecture.md

-------------------------------------------------------------------------------

License
-------

Proprietary software.

All rights reserved.
