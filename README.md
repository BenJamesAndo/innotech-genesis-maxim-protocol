# Innotech Genesis / Maxim Protocol

Partially reverse-engineered TCP protocol for controlling Innotech air conditioning systems via the iComm interface. Data captured via packet capture of the [Supervisor software](https://innotech.com/Products/Digital/GenesisSeries.aspx) (V5.40) communicating with a Genesis II board.

## Overview

This repository contains a Node-RED flow for controlling Innotech AC units by replaying and constructing TCP packets that mimic the Supervisor software. The temperature setpoint encoding was reverse-engineered; everything else (on/off schedules, keepalive) was manually captured from Supervisor traffic.

**Note:** This protocol documentation is incomplete. Only the portions that have been captured and/or reverse-engineered are documented here.

---

## Protocol Basics

### Communication

- **Transport:** TCP
- **Port:** 20000
- **IP:** The IP address of the iComm controller (e.g., `192.168.1.10`)
- **Web Interface:** Port 8080 (e.g., `http://192.168.1.10:8080/iComm/Connection`)

### Hardware

- **Controller:** Innotech Genesis II
- **Interface:** iComm (SW v2.38)
- **Management Software:** Supervisor V5.40

### Magic Bytes

All observed packets begin with the bytes `78 56` (`xV` in ASCII). This appears to be a protocol identifier/magic header.

### Observed Command Types

The bytes following the `78 56` header indicate the command type:

| Bytes | Purpose |
|-------|---------|
| `78 56 48 00` | Hello / keepalive |
| `78 56 3f 00` | Temperature setpoint write |
| `78 56 0d 00` | Schedule write trailer / delimiter |

---

## Keepalive / Hello Packet

A hello packet is sent every 30 seconds to maintain the TCP connection. It identifies the client as Supervisor:

```
78 56 48 00 11 00 01 00 00 00 0a 00 53 75 70 65
72 76 69 73 6f 72 20 56 35 2e 34 31 48 00 ...
```

The packet contains the ASCII string `Supervisor V5.41` followed by null padding for the Name and `HOMEASSISTANT` as the Host Name.

---

## Temperature Setpoint (Reverse-Engineered)

This is the only part of the protocol that was genuinely reverse-engineered rather than simply captured.

### Encoding

Temperature values are encoded as **IEEE 754 double-precision floating-point numbers in little-endian byte order** (8 bytes), appended to a zone-specific header.

### Packet Structure

```
[zone-specific header] [8-byte temperature as IEEE 754 double LE]
```

The zone-specific headers begin with `78 56 3f 00 06 00` followed by bytes that identify the AHU and zone. These headers also contain internal Supervisor data path references (service names, IO references, etc.) as embedded ASCII strings.

### Temperature Encoding Examples

| Temperature | IEEE 754 Double LE (hex) |
|-------------|--------------------------|
| 15.0°C | `00 00 00 00 00 00 2e 40` |
| 15.5°C | `00 00 00 00 00 00 2f 40` |
| 16.0°C | `00 00 00 00 00 00 30 40` |
| 20.0°C | `00 00 00 00 00 00 34 40` |
| 23.5°C | `00 00 00 00 00 80 37 40` |
| 25.0°C | `00 00 00 00 00 00 39 40` |
| 30.0°C | `00 00 00 00 00 00 3e 40` |

### Supported Range

- **Minimum:** 15.0°C
- **Maximum:** 30.0°C
- **Increment:** 0.5°C steps (0.1°C steps are also possible)

### AHU / Zone Mapping

Each AHU and zone combination has a unique packet header. The following have been captured:

| AHU | Zone |
|-----|------|
| 3 | 1 |
| 5 | 1 |
| 5 | 2 |
| 5 | 3 |
| 6 | 1 |
| 7 | 1 |
| 8 | 1 |
| 9 | 1 |

---

## On/Off Control (Captured)

On/off control is achieved by **writing or clearing the weekly schedule data** on the controller. This was not reverse-engineered — the on and off packets for each area were manually captured from Supervisor.

### How It Works

- **Turn ON:** Sends a schedule write packet that populates the weekly schedule with time slot entries (effectively enabling the AC to run on its normal schedule)
- **Turn OFF:** Sends a schedule write packet with all time slots zeroed out (effectively blanking the schedule so the AC never runs)

### Packet Structure

Schedule packets follow this general structure:

```
[schedule header] [schedule name (ASCII)] [time config] [time slot data] 78 56 0d 00
```

- The schedule name is the area's weekly schedule name (e.g., `West Rear Weekly`, `Kitchen Weekly`, `East Audit Weekly`)
- Time slot data contains start/end time pairs — populated for ON, zeroed for OFF
- Each command packet is sent **twice** (the payload is duplicated within the same TCP send)
- Packets are terminated with the `78 56 0d 00` trailer

---

## Files

### Node-RED Flows

#### AC Control.json

Controls AC units via Home Assistant MQTT integration. Contains:

- **Keepalive:** Sends the hello packet every 30 seconds to maintain the TCP session
- **On/Off switches:** Pre-captured schedule write commands for each area, exposed as MQTT switches in Home Assistant
- **Temperature setpoints:** The reverse-engineered temperature encoding function, exposed as MQTT number entities in Home Assistant (15–30°C range)
- **`convert to hex value Multi-zone`:** The key function node that dynamically generates temperature setpoint packets for any supported AHU/zone. This is the reverse-engineered component
- **Rate limiting:** Commands are sent through a delay/rate limiter to avoid overwhelming the controller

---

## Home Assistant Integration

The Node-RED flow integrates with Home Assistant via MQTT using `ha-mqtt` nodes. AC areas appear as:

- **Switch entities** — for on/off control per area
- **Number entities** — for temperature setpoint per AHU/zone (15–30°C)

The MQTT device is configured as:

| Property | Value |
|----------|-------|
| Manufacturer | Innotech |
| Model | iComm |
| HW Version | Genesis II |
| SW Version | 2.38 |

---

## Limitations and Unknowns

- Only temperature setpoint encoding has been reverse-engineered; on/off commands are static captured packets
- The full packet header structure (bytes between `7856 3f00 0600` and the temperature value) is not fully decoded — it contains Supervisor internal path references but the addressing scheme is not understood
- No read-back / status polling has been implemented — commands are fire-and-forget (I do have a web scraper for the iComm Control that obtains the sensor information, not uploaded  yet)
- Schedule time slot encoding has not been reverse-engineered (on/off uses pre-captured payloads)
- Only the AHU/zone combinations listed above have been captured

---

## Dependencies

### Node-RED Modules

- `node-red-contrib-string-to-hex` — Converts hex string payloads to binary buffer for TCP transmission
- `node-red-contrib-ha-mqtt` — Home Assistant MQTT device integration

---

## Disclaimer

This is an unofficial, reverse-engineered protocol implementation. Use at your own risk. The author is not affiliated with Innotech.
