# Borui / "power" device protocol

This document describes the low-level serial protocol implemented by the Borui-style programmable regulated power supply. It is written for developers who need to talk directly to the instrument (implement a client, driver or test harness).

## Summary (quick reference)
- Transport: serial UART, typically 9600 8N1 (no flow control).
- Supported baud rates: 1200, 2400, 4800, 9600, 19200. Default: 9600.
- Framing: ASCII bracketed frames: `<...>` with no embedded newlines; device messages are entirely contained between `<` and `>`.
- Message format: fixed-width digit fields inside the brackets; fields are zero-padded decimal ASCII.
- Numeric encoding: many fields use integer encodings with an implied decimal position (centi-units are common).

## Physical/serial parameters
- Baudrate: 9600 (common), Data bits: 8, Parity: N, Stop bits: 1.
- No special flow-control bytes. Messages are short; implementations should allow a modest inter-character timeout when reading.

## Framing and I/O semantics
- Each frame begins with a literal `<` and ends with a literal `>`; the content between is an ASCII digit string (no CR/LF inside the frame).
- The device does not use newline terminators inside the frame; the closing `>` is the definitive end-of-message marker.
- On the wire a request looks like: `<xxxxxxxxxxx>` and the reply uses the same bracketed format.
- Implementations should read until `>` (or until a reasonable timeout) and then process the characters between `<` and `>` as a complete message.

### Inter-frame timing
- The instrument expects a silent idle period of at least 3.5 character times before the start of a new frame. If a pause of more than 3.5 character times occurs while a frame is being received the device will discard the incomplete frame and treat the next byte as the start of a new frame. Conversely, if bytes of a new frame arrive faster than 3.5 character times after a previous frame, the device may consider them part of the previous frame. Drivers must therefore respect the 3.5-character gap when composing multi-frame exchanges.

## Message structure
Typical device messages use the following fixed layout (decimal digits):

  <op(2)><payload(5)><param(4)>

- `op` (2 digits): operation or response code. Interpreted as a small integer opcode or status identifier.
- `payload` (5 digits): primary numeric value or measurement field. Zero-padded ASCII digits (e.g. `00123`).
- `param` (4 digits): auxiliary parameter, flags or additional numeric data.

The exact semantics of `op`, `payload` and `param` depend on the command; the device uses different opcodes for measurement vs. control frames.

## Numeric encoding and scaling
- Numeric fields are plain unsigned decimal ASCII with zero-padding. Many numeric values encode fixed-point numbers by using an implied decimal point:
  - A common convention is centi-units: integer `1234` represents `12.34` (i.e., divide by 100 to get human units).
  - When composing requests, values must be scaled and zero-padded to the field width expected by the device.

### Encoding details (common conventions)
- Voltage payloads are commonly encoded in centi-volts (payload / 100 → volts). Example: payload `00458` → 4.58 V.
- Current payloads may use milli- or centi-units depending on the instrument model and magnitude. Examples observed in firmware traces:
  - `<14009300000>` → payload `00930` → interpreted as 9.30 A (divide by 100)
  - `<14000183000>` → payload `00183` → interpreted as 0.183 A (divide by 1000)

Because the device does not explicitly indicate the payload scale in every response, drivers should consult the instrument's manual for exact scaling per-opcode, or infer scale from expected ranges and precision.

Examples:
- If the payload is `01234` and the instrument uses centi-units, the physical value is 12.34 (payload / 100).
- To send 1.23 (human units) in a 5-digit payload with centi-unit encoding: scale to 123 and send `00123`.

## Command/response behavior
- Requests are sent as complete `<...>` frames. The device processes the frame and usually replies with a bracketed response frame.
- Some control frames do not produce device responses (they act as one-way commands); the device silently acknowledges by changing state rather than producing an error string.
- When the device cannot parse a command or receives invalid parameters it commonly ignores the request (no explicit error string). Client implementations should validate by reading back values after a write if confirmation is required.

## Opcode summary (common opcodes)
The two-digit `op` field identifies the request or response type. Observed conventions:

- Requests (examples):
  - `01` — Set voltage (request). Example: `<01012100001>` sets device address `0001` to 12.10 V.
  - `02` — Read voltage (request). Example: `<02012200000>` asks for the measured voltage.
  - `03` — Set current (request). Example: `<03006920000>` sets current limit.
  - `04` — Read current (request). Example: `<04003300000>` asks for the measured current.
  - `07` — Power on (request): `<07000000000>`
  - `08` — Power off (request): `<08000000000>`
  - `09` — Remote lock (keypad lock). Param leading digit: `1` = remote lock on, `2` = remote lock off. Examples: `<09100000000>` (remote lock on), `<09200000000>` (remote lock off).

- Responses (observe request response + 10):
  - `11` — Set-voltage acknowledgement; example reply: `<11OK0000000>` indicates OK.
  - `12` — Read-voltage response; example reply: `<12004580000>` encodes 4.58 V.
  - `13` — Set-current acknowledgement; example reply: `<13OK0000000>`.
  - `14` — Read-current response; example reply: `<14000183000>` encodes 0.183 A.

Note: response opcodes are typically the request opcode plus 10 (decimal). Implementations should accept these observed mappings but consult the device manual for complete opcode tables.

## Testing-derived specifics
- Transport/IO hints:
  - Example serial settings used: `baud=9600`, `8N1`, timeouts around `2.0s`, and `recv_chunk_size=13` bytes.
  - Many mappings use the `>` character as the message terminator; drivers should read until `>` when parsing responses for these commands.

- Exact request frames observed in the wild (useful examples):
  - Read voltage: `<02012200000>` (device replies with `<12xxxxx....>` style frame)
  - Read current: `<04003300000>`
  - Set voltage template: `<01ppppp0000>` where `ppppp` is a zero-padded 5-digit scaled payload (scale: ×100). Example intent: to set 12.10 V the scaled payload is `00121` → `<01001210000>` (config used `<01${1}0000>`).
  - Set current template: `<03ppppp000>` where the payload width is 6 and observed scaling is ×1000 in the configuration (i.e. scale 1000).
  - Output on: `<07000000000>`; Output off: `<08000000000>`
  - Remote lock / remote unlock (keypad lock): `<09100000000>` (remote lock on), `<09200000000>` (remote lock off)
  - Static identity response (no transport I/O): `*IDN?` → `KUAIQU`

## Timing and robustness
- Because the device uses a single-character end marker (`>`), clients should accumulate bytes until `>` is received or a read timeout occurs.
- Use a small inter-character timeout to avoid blocking indefinitely; allow a slightly longer overall receive timeout for full frames.

## Example frames
- Example request (control):

```
<02012200000>   # Example: read voltage request
```

- Example reply (measurement):

```
<02012345000>
```

In the reply above, `op` = `02`, `payload` = `01234`, `param` = `5000` (field meanings depend on opcode); if payload uses centi-units, `01234` → 12.34 units.

## CV/CC state and parameter field
- The `param` (4-digit) field often encodes status bits and the device address. In vendor examples the first digit(s) of the trailing region indicate CV/CC status (for current read replies the device may set a flag indicating whether it is in CV or CC mode) and the low-order digits include the device address. Example: `<12000000001>` indicates a voltage reading for device address 1.

## Hex representation (wire bytes)
- ASCII frame `<01012100001>` corresponds to hex byte sequence: `3C 30 31 30 31 32 31 30 30 30 30 31 3E`.

## Implementation recommendations
- Respect the 3.5-character silent gap between frames when sending multiple commands.
- Compose frames exactly as ASCII digits; zero-pad numeric fields to the required widths (5 digits for payload, 4 digits for param).
- After writes, perform a read-back (read-voltage/read-current) to confirm the change because the device may not send explicit error messages on invalid input.
- Provide configurable per-opcode scaling (voltage: /100; current: /100 or /1000 depending on model) and allow overrides for specific instrument models.

## Summary
This instrument uses short bracketed ASCII frames with fixed digit fields: 2-digit opcode, 5-digit numeric payload and a 4-digit parameter/status block. Numeric payloads are fixed-point integers (implied decimal) and scaling varies by measurement type; use the vendor examples above and the device manual to choose the correct scaling per-opcode.

## Error handling and best practices
- The device tends to fail silently on invalid commands. Always read back important settings or measurements after issuing a write.
- Zero-pad numeric fields to the expected widths. Do not include extra whitespace inside `<...>` frames.
- Prefer conservative timeouts and read-until-`>` logic rather than fixed-length reads.

## Implementation checklist for a driver
- Open serial port at 9600 8N1 (or the device-specific settings).
- To send a command: compose the ASCII payload, ensure numeric fields are scaled and zero-padded, wrap with `<` and `>`, and write the bytes.
- To receive a response: read bytes until `>` is seen, validate the leading `<`, then parse the inner text according to the opcode-specific layout.
- Scale numeric payloads (divide by 100 or other device-specific scale) before exposing values to higher-level code.

