# Badge module

The `badge` module exposes the badge's hardware to MicroPython programs. PIO0 is used for data transfer to the screen. Be careful if you decide to drive random pins :)

```python
import badge
```

## SAO connector

The board defines these aliases for `machine.Pin`:

| Name        | GPIO |
| ----------- | ---- |
| `SAO_GPIO1` | 4    |
| `SAO_GPIO2` | 5    |
| `SAO_SDA`   | 6    |
| `SAO_SCL`   | 7    |

Use string names or attributes such as `Pin.board.SAO_GPIO1`.
The SAO I2C pins belong to hardware bus **I2C1**.

```python
from machine import I2C, Pin

gpio1 = Pin("SAO_GPIO1", Pin.IN)
i2c = I2C(1)  # SAO SDA=6, SCL=7; default frequency 400 kHz.
# Explicit named-pin configuration is also supported:
i2c = I2C(1, sda=Pin.board.SAO_SDA, scl=Pin.board.SAO_SCL)
```

## Target device

### `en_target_device(en)`

MicroPython firmware drives the reset pin of the STM32 target LO automatically on boot, explicitly enable it if needed.

```python
badge.en_target_device(True)   # Enable the STM32 target.
badge.en_target_device(False)  # Hold the STM32 target in reset.
```

## Buttons and joystick

The badge has two buttons, A and B, plus a five-way directional switch
(up, down, left, right and centre push). All of the functions below return
`True` while the button is held down and `False` otherwise.

| Function           | Description                 |
| ------------------ | --------------------------- |
| `read_up_btn()`    | Directional switch, up.     |
| `read_down_btn()`  | Directional switch, down.   |
| `read_left_btn()`  | Directional switch, left.   |
| `read_right_btn()` | Directional switch, right.  |
| `read_push_btn()`  | Directional switch, centre. |
| `read_a_btn()`     | Button A.                   |
| `read_b_btn()`     | Button B.                   |

```python
import badge

if badge.read_a_btn():
    print("A is pressed")
```

## Battery

### `read_batt_voltage()`

Returns the battery voltage in volts as a float. Only returns valid value when the slider switch is ON, otherwise 0V.

```python
voltage = badge.read_batt_voltage()
print("Battery: %.3f V" % voltage)
```

## Display

The screen is 320x240 pixels and expects a MicroPython `framebuf.FrameBuffer`
in `RGB565` format (that is, 2 bytes per pixel). Build a framebuffer, draw to
it, then send the whole thing to the screen.

Note that unfortunately we don't have enough RAM for two screen buffers, and the buffer passed to write_screen\* shoudn't be modified while actually writing to the screen.

### `write_screen(fb)`

Blocking full-frame write. Returns when the frame has been sent.

```python
import framebuf
import badge

buf = bytearray(320 * 240 * 2)
fb = framebuf.FrameBuffer(buf, 320, 240, framebuf.RGB565)
fb.fill(0xFFFF)  # white

badge.write_screen(fb)
```

### `write_screen_async(fb)`

As `write_screen`, but returns without waiting for the write to finish. See `write_screen_wait` below.

```python
badge.write_screen_async(fb)
```

### `write_screen_wait()`

Blocks until any in-flight `write_screen_async` has finished. Safe to call
when nothing is in flight.

```python
badge.write_screen_async(fb)
badge.write_screen_wait()
```

### `write_screen_backlight(intensity)`

Sets the backlight brightness. `intensity` is a float from `0.0` (off) to
`1.0` (full). Values outside this range are clamped.

```python
badge.write_screen_backlight(0.5)  # half brightness
```

## CAN bus

The badge can talk to the outside world over CAN. The game bus runs at 50 kbit/s.

### `can_init(baudrate, sample_point=0.75, sjw=1)`

Configures and starts the CAN controller. Call once before using `can_send`
or `can_recv`.

- `baudrate` – bit rate in bits/second (1 to 1,000,000).
- `sample_point` – optional float from `0.0` to `1.0` (default `0.75`).
- `sjw` – optional sync jump width, `1` to `4` (default `1`).

Raises `ValueError` for an unsupported configuration and `OSError` if the
transceiver cannot be reached.

```python
badge.can_init(50000)
```

### `can_send(id, data, extended=False, rtr=False)`

Transmits one frame.

- `id` – CAN identifier.
- `data` – payload, a bytes-like object of up to 8 bytes.
- `extended` – optional bool; use a 29-bit extended identifier.
- `rtr` – optional bool; send a remote-transmission-request frame.

Raises `ValueError` if the payload is too long and `OSError` if CAN is not
initialised or no transmit buffer is free.

```python
badge.can_send(0x123, b"\x01\x02\x03")
```

### `can_recv()`

Returns the oldest received frame as a tuple
`(id, data, extended, rtr)`, where `data` is a `bytes` object. Returns
`None` if no frames are waiting. For remote frames `data` is `None`.

```python
msg = badge.can_recv()
if msg is not None:
    can_id, data, extended, rtr = msg
    print(can_id, data)
```

### `can_irq(callback)`

Registers a callable to be run (via the MicroPython scheduler) whenever a
frame arrives. Pass `None` to clear the handler. The callback takes no
arguments; read the frame with `can_recv()` inside it.

```python
def on_can():
    print("frame received", badge.can_recv())

badge.can_irq(on_can)
```

### `can_status()`

Returns a tuple of raw controller status registers
`(canstat, canintf, eflg)`, useful for debugging. Returns the same values
whether CAN is initialised or not.

## Raw CAN controller registers

For debugging and experimentation, the XL2515 register-level access
commands are exposed directly.

### `can_reg_read(addr)`

Reads one register and returns its value as an int. `addr` must be `0` to
`0xFF`.

```python
print(hex(badge.can_reg_read(0x0E)))  # CANSTAT
```

### `can_reg_write(addr, value)`

Writes `value` to register `addr`. Both must be `0` to `0xFF`. Note that
some registers (such as CANCTRL/CANSTAT mode bits) are best changed with
`can_reg_modify`.

```python
badge.can_reg_write(0x2C, 0x00)  # clear CANINTF
```

### `can_reg_modify(addr, mask, value)`

Bit-modify command: for every bit set in `mask`, sets the corresponding
register bit to the corresponding bit of `value`. Bits not set in `mask`
are left unchanged. All arguments must be `0` to `0xFF`.

```python
badge.can_reg_modify(0x0F, 0xE0, 0x00)  # request normal mode via CANCTRL REQOP bits
```
