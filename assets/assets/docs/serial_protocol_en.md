# User Guide: Serial Communication

## 1. Purpose

This guide is for host application developers and developers integrating PLCs, microcontrollers, or industrial computers. It explains how to communicate with the 8-channel color contrast sensor over a serial port, read device data, and interpret the returned fields.

## 2. What You Can Do over the Serial Port

The host can use the serial port to:

- Check whether the device is online.
- Query data from the eight sensor channels on demand.
- Query basic device information and saved calibration data.
- Capture the background and line colors, then save the calibration results.
- Restart the device.
- Start or stop a real-time data stream.
- Identify error responses from the device.

## 3. Serial Port Settings

| Item | Setting |
| --- | --- |
| Communication | UART serial port |
| Baud rate | `115200` |
| Data bits | `8 bit` |
| Parity | None |
| Stop bits | `1 bit` |
| Hardware flow control | None |
| Common notation | `115200 8N1` |

## 4. Communication Overview

Serial communication takes three forms:

1. **PING:** The host sends `CC` and the device returns `CC` to confirm the connection quickly.
2. **Standard request and response frames:** The host sends a request frame in the `AA ... BB` format, and the device returns a response frame with the same structure.
3. **Real-time data stream:** After the host sends the stream control bytes, the device outputs fixed-length data frames periodically.

Use a sensor data query to read data occasionally. Use the real-time data stream when data needs to be refreshed continuously.

## 5. PING

PING confirms that the device is online and the serial link is working.

### 5.1 Request

```text
CC
```

### 5.2 Response

```text
CC
```

### 5.3 Notes

- PING is a single-byte exchange and does not use the standard frame format.
- When the host sends `0xCC`, an online device returns `0xCC`.

## 6. Standard Frame Format

Standard frames carry request and response commands. Requests and responses have the same structure.

```text
AA Command Length Payload CRC_L CRC_H BB
```

| Order | Field | Length | Description |
| --- | --- | --- | --- |
| 1 | Frame header | 1B | Fixed at `0xAA`; marks the start of a standard frame |
| 2 | Command | 1B | Command code indicating the frame's operation |
| 3 | Length | 1B | Number of bytes in Payload |
| 4 | Payload | N bytes | Payload data; its size is specified by Length |
| 5 | CRC_L | 1B | Low byte of CRC-16 |
| 6 | CRC_H | 1B | High byte of CRC-16 |
| 7 | Frame tail | 1B | Fixed at `0xBB`; marks the end of a standard frame |

### 6.1 Length Field

`Length` specifies only the length of `Payload`. It does not include the frame header, command code, Length byte, CRC, or frame tail.

For example:

```text
AA 01 00 00 20 BB
```

| Byte(s) | Meaning |
| --- | --- |
| `AA` | Frame header |
| `01` | Command code: sensor data query request |
| `00` | Payload length is 0 |
| `00 20` | CRC-16, low byte first |
| `BB` | Frame tail |

## 7. CRC-16 Check

Standard frames use CRC-16.

| Item | Description |
| --- | --- |
| Algorithm | CRC-16/MODBUS |
| Initial value | `0xFFFF` |
| Polynomial | `0xA001` |
| Output byte order | Low byte first, high byte second |
| Calculation range | `Command + Length + Payload` |

The CRC does not include:

- Frame header `0xAA`
- The CRC field itself
- Frame tail `0xBB`

## 8. Quick Start

### 8.1 Check Whether the Device Is Online

Host sends:

```text
CC
```

Device returns:

```text
CC
```

### 8.2 Query Sensor Data Once

Host sends this standard frame:

```text
AA 01 00 00 20 BB
```

The device returns a standard frame with command code `0x81` and a 20-byte Payload.

### 8.3 Start the Real-Time Data Stream

Host sends:

```text
DD 0A
```

This starts the real-time data stream. `Interval` is a one-byte output interval in milliseconds; `0x0A` means one output every 10 ms.

### 8.4 Stop the Real-Time Data Stream

Host sends:

```text
DD 00
```

This stops the real-time data stream.

## 9. Sensor Data Query

The sensor data query reads the eight digital outputs, analog values, normalized values, and offset.

### 9.1 Request

| Item | Value |
| --- | --- |
| Command | `0x01` |
| Length | `0x00` |
| Payload | None |

Complete request example:

```text
AA 01 00 00 20 BB
```

### 9.2 Response

| Item | Value |
| --- | --- |
| Command | `0x81` |
| Length | `0x14`, or 20 bytes |
| Payload | Sensor data |

Payload fields:

| Offset | Field | Length | Description |
| --- | --- | --- | --- |
| 0 | Status | 1B | Status bitmap; see below |
| 1 | Digital | 1B | Bitmap of the eight digital outputs |
| 2-9 | Analog | 8B | Eight analog values, one byte per channel |
| 10-17 | Normal | 8B | Eight normalized values, one byte per channel |
| 18-19 | Offset | 2B | Signed 16-bit offset, low byte first |

`Status` bits:

| Bit or value | Meaning |
| --- | --- |
| bit0 = `1` | The sensor head is currently overexposed |
| bit1 = `1` | Calibration is locked: a previous calibration failed, and the device cannot currently enter calibration mode. Restart the device to clear this state. |
| `0x00` | Neither of the above states is present |

### 9.3 Digital Field

`Digital` is a bitmap of the eight digital outputs.

| Bit | Channel |
| --- | --- |
| bit0 | Channel 0 |
| bit1 | Channel 1 |
| bit2 | Channel 2 |
| bit3 | Channel 3 |
| bit4 | Channel 4 |
| bit5 | Channel 5 |
| bit6 | Channel 6 |
| bit7 | Channel 7 |

Example:

```text
Digital = 05
```

`0x05` is `0000 0101` in binary, meaning channels 0 and 2 output 1.

### 9.4 Analog Field

`Analog` contains eight bytes, representing the analog values of channels 0 through 7 in order.

| Byte position | Channel |
| --- | --- |
| Analog[0] | Channel 0 |
| Analog[1] | Channel 1 |
| Analog[2] | Channel 2 |
| Analog[3] | Channel 3 |
| Analog[4] | Channel 4 |
| Analog[5] | Channel 5 |
| Analog[6] | Channel 6 |
| Analog[7] | Channel 7 |

### 9.5 Normal Field

`Normal` contains eight bytes, representing the normalized values of channels 0 through 7 in order.

| Byte position | Channel |
| --- | --- |
| Normal[0] | Channel 0 |
| Normal[1] | Channel 1 |
| Normal[2] | Channel 2 |
| Normal[3] | Channel 3 |
| Normal[4] | Channel 4 |
| Normal[5] | Channel 5 |
| Normal[6] | Channel 6 |
| Normal[7] | Channel 7 |

### 9.6 Offset Field

`Offset` is two bytes. Combine the bytes in little-endian order to obtain a 16-bit value, then interpret that value as a signed int16.

| Raw bytes | Parsed value |
| --- | --- |
| `01 00` | `1` |
| `FF FF` | `-1` |
| `00 80` | `-32768` |

## 10. Device Information Query

The device information query reads the hardware version, device runtime, software version, and board type.

### 10.1 Request

| Item | Value |
| --- | --- |
| Command | `0x04` |
| Length | `0x00` |
| Payload | None |

Complete request example:

```text
AA 04 00 03 70 BB
```

The CRC-16/MODBUS result for `04 00` is `0x7003`. The low byte is sent before the high byte, so the CRC bytes are `03 70`.

### 10.2 Response

| Item | Value |
| --- | --- |
| Command | `0x84` |
| Length | `0x08`, or 8 bytes |
| Payload | Device information |

Payload fields:

| Offset | Field | Length | Description |
| --- | --- | --- | --- |
| 0 | HardwareVersion | 1B | Hardware version; `0xFF` means unwritten or invalid |
| 1-4 | RuntimeSeconds | 4B | Runtime since the current power-up, in seconds, low byte first |
| 5 | RunMode | 1B | Current operating mode |
| 6 | SoftwareVersion | 1B | Software version |
| 7 | BoardType | 1B | Board type |

`RunMode` values:

| Value | Meaning |
| --- | --- |
| `0x00` | Normal operation |

For hardware and software versions, the upper four bits are the major version and the lower four bits are the minor version. For example, `0x21` means `2.1`. The current firmware's software version byte is `0x37`, or `3.7`. `BoardType` `0x00` means a curved board; `0x01` means a straight board.

## 11. Calibration Data and Operations

To calibrate, point the sensor at the background color and capture it, then point it at the line color and capture it, and finally save. A capture command reads the current measured values when the command is sent.

### 11.1 Query Saved Calibration Data

Send command `0x05` with an empty request Payload:

```text
AA 05 00 02 E0 BB
```

The device returns command `0x85` with a 33-byte Payload:

| Offset | Field | Length | Description |
| --- | --- | --- | --- |
| 0-15 | High[0..7] | 16B | High-value calibration data for eight channels, two bytes per channel, little-endian |
| 16-31 | Low[0..7] | 16B | Low-value calibration data for eight channels, two bytes per channel, little-endian |
| 32 | LineType | 1B | `0x00`: background is high and line is low; `0x01`: background is low and line is high |

### 11.2 Capture the Background Color

Point the sensor at the background color, then send command `0x06`:

```text
AA 06 00 02 10 BB
```

On success, the device returns command `0x86` with Payload length `0x01` and Payload `00`. If the sensor is currently overexposed or calibration is locked, it returns error code `0x05`.

### 11.3 Capture the Line Color

Point the sensor at the line color, then send command `0x07`. Capture all eight channels at once or in two groups:

| Capture range | Complete request frame |
| --- | --- |
| All eight channels | `AA 07 00 03 80 BB` |
| First four channels (channels 0-3) | `AA 07 01 01 00 51 BB` |
| Last four channels (channels 4-7) | `AA 07 01 02 40 50 BB` |

A straight board can capture all eight channels at once. For a curved board, capture the first four channels and then the last four; calibration cannot be saved after only one of these captures. A one-byte Payload of `00` also selects all eight channels.

On success, the device returns command `0x87` with Payload length `0x01` and Payload `00`. If the sensor is overexposed during capture or calibration is locked, it returns error code `0x05`; an invalid range value returns `0x03`.

### 11.4 Save Calibration Data

After capturing the background and all line-color channels, send command `0x08`:

```text
AA 08 00 06 70 BB
```

On success, the device returns command `0x88` with Payload length `0x01` and Payload `00`. An incomplete capture returns error code `0x03`; locked calibration returns `0x05`. After saving, send `0x05` again to query the calibration data.

## 12. Restart the Device

Send command `0x10` with Payload `01`:

```text
AA 10 01 01 B0 55 BB
```

The device first returns command `0x90` with Payload `00 01`, then restarts. The serial port is briefly unavailable during the restart. Send PING after the device is ready again.

## 13. Real-Time Data Stream

The real-time data stream makes the device output sensor data periodically.

It uses fixed-length data frames instead of the standard frame format.

### 13.1 Start or Update the Real-Time Data Stream

Host sends:

```text
DD Interval
```

| Byte | Description |
| --- | --- |
| `DD` | Real-time stream control header |
| `Interval` | One-byte output interval in milliseconds; a nonzero value starts or updates periodic output, and `00` stops it |

Example:

```text
DD 0A
```

This starts the real-time data stream with a 10 ms output interval.

### 13.2 Stop the Real-Time Data Stream

Host sends:

```text
DD 00
```

When `Interval` is `0x00`, the device stops the real-time data stream. While the stream is active, the device does not process ordinary query or calibration requests. Send `DD 00` to stop the stream before sending those requests.

### 13.3 Real-Time Data Frame Format

The device outputs fixed-length, 21-byte data frames:

```text
DD Digital Analog[0] ... Analog[7] Normal[0] ... Normal[7] Offset_L Offset_H EE
```

| Offset | Field | Length | Description |
| --- | --- | --- | --- |
| 0 | Frame header | 1B | Fixed at `0xDD` |
| 1 | Digital | 1B | Bitmap of the eight digital outputs |
| 2-9 | Analog | 8B | Eight analog values, one byte per channel |
| 10-17 | Normal | 8B | Eight normalized values, one byte per channel |
| 18-19 | Offset | 2B | Signed 16-bit offset, low byte first |
| 20 | Frame tail | 1B | Fixed at `0xEE` |

### 13.4 Parsing Notes for the Real-Time Data Stream

- Each real-time data frame is exactly 21 bytes long.
- Real-time data frames have no Command, Length, or CRC fields.
- To parse the stream, find the `0xDD` frame header and `0xEE` frame tail and confirm the frame length is 21 bytes.
- `Digital`, `Analog`, `Normal`, and `Offset` have the same meanings as the corresponding fields in a sensor data query response.

## 14. Error Responses

The device may return an error response for an unsupported command, incorrect length, invalid parameter, or overexposure during calibration.

Error responses use the standard frame format.

| Item | Value |
| --- | --- |
| Command | `0x7F` |
| Length | `0x02` |
| Payload | Original command code + error code |

Payload fields:

| Offset | Field | Length | Description |
| --- | --- | --- | --- |
| 0 | OriginCommand | 1B | Original command code that caused the error |
| 1 | ErrorCode | 1B | Error code |

Error codes:

| Error code | Meaning |
| --- | --- |
| `0x01` | Unsupported command |
| `0x02` | Incorrect data length |
| `0x03` | Invalid parameter value |
| `0x04` | Execution failed |
| `0x05` | Overexposure during calibration capture, or calibration locked |

Example:

```text
Command = 7F
Payload = 99 01
```

This means command `0x99` is unsupported.

## 15. Fields and Byte Order

### 15.1 Hexadecimal Notation

All bytes in this guide are shown in hexadecimal. For example:

```text
AA 01 00 00 20 BB
```

This represents six bytes sent in sequence: `0xAA`, `0x01`, `0x00`, `0x00`, `0x20`, `0xBB`.

### 15.2 Multibyte Fields

Multibyte fields use little-endian order: the low byte comes before the high byte.

For example, these raw `Offset` bytes:

```text
34 12
```

are parsed as:

```text
0x1234
```

### 15.3 Suggested Standard Frame Parsing Sequence

1. Find the `0xAA` frame header.
2. Read `Command`.
3. Read `Length`.
4. Read the number of `Payload` bytes specified by `Length`.
5. Read the two CRC bytes.
6. Confirm that the frame tail is `0xBB`.
7. Calculate the CRC over `Command + Length + Payload` and compare it with the received CRC.
8. After the CRC passes, parse Payload according to the command code.

### 15.4 Suggested Real-Time Stream Parsing Sequence

1. Find the `0xDD` frame header.
2. Read 21 consecutive bytes.
3. Confirm that the 21st byte is the `0xEE` frame tail.
4. Parse `Digital`, `Analog`, `Normal`, and `Offset` at their fixed offsets.

## 16. Frequently Asked Questions

### 16.1 Why Is There No Response after Sending `CC`?

Check:

- Is the correct serial port selected?
- Is the baud rate `115200`?
- Is the data format `8N1`?
- Are TX and RX swapped?
- Is the device powered correctly?

### 16.2 Why Is There No Response to a Standard Frame?

Check:

- Is the frame header `AA`?
- Is the frame tail `BB`?
- Does Length equal the actual Payload length?
- Is the CRC calculated over `Command + Length + Payload`?
- Is the low CRC byte sent first?

### 16.3 When Should I Use a Query or the Real-Time Stream?

- To read data occasionally, use the sensor data query.
- To display or plot data continuously, use the real-time data stream.
- To confirm the device is connected, use PING.
