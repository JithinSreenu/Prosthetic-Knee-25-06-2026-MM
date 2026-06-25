# Prosthetic Knee Firmware — Complete Code Walkthrough

> This is the **definitive line-by-line guide**. Every header, every
> `.c` file, every pin, every register write is explained.
> Read this once and you'll never need to ask "what does this code
> do?" again.

---

## Table of contents

1. How the includes hang together (header dependency graph)
2. Pin map reference — every physical pin we use
3. `Application/Inc/project_types.h` — common types
4. `Application/Inc/packet_protocol.h` — wire format
5. `Application/Src/packet_protocol.c` — encoder + decoder
6. `Application/Inc/filter_lib.h` & `filter_lib.c` — DSP filters
7. `Application/Inc/adc_manager.h` & `adc_manager.c` — internal ADC
8. `Application/Inc/motor_control.h` & `motor_control.c` — PWM motors
9. `Application/Inc/uart_dma.h` & `uart_dma.c` — UART + DMA
10. `Application/Inc/bluetooth.h` & `bluetooth.c` — telemetry task
11. `Application/Inc/state_machine.h` & `state_machine.c` — gait FSM
12. `Application/Inc/safety.h` & `safety.c` — watchdog + heartbeat
13. `Application/Inc/app_tasks.h` & `app_tasks.c` — task wiring
14. `Core/Inc/FreeRTOSConfig.h` — kernel tuning
15. `Core/Inc/stm32u5xx_it.h` & `Core/Src/stm32u5xx_it.c` — ISRs
16. `Core/Src/main.c` — boot + manual peripheral init
17. `Core/Src/startup_stm32u585.s` — vector table + reset
18. `Core/Src/stm32u585.ld` — linker script
19. `Tests/host_test.c` — offline unit tests
20. Build sequence (`Makefile`)

---

## 1. How the includes hang together

```
                    project_types.h            <-- foundation (no project deps)
                         |
        +----------------+----------------+----------------+
        |                |                |                |
   filter_lib.h    packet_protocol.h   state_machine.h   adc_manager.h
                                              |                |
                                              |                |
                                              v                v
                                         motor_control.h   app_tasks.h
                                              |                |
                                              +-------+--------+
                                                      |
                                                      v
                                                app_tasks.c
                                                      |
                                                      v
                                                   main.c

   bluetooth.h ──> packet_protocol.h
                 └─> uart_dma.h
                              │
                              v
                       bluetooth.c

   safety.h ──> state_machine.h
   safety.c ──> motor_control.h

   uart_dma.c ──> packet_protocol.h, FreeRTOS queue.h
   adc_manager.c ──> stm32u5xx_hal.h
   motor_control.c ──> stm32u5xx_hal.h

   stm32u5xx_it.c ──> FreeRTOS.h, task.h, adc_manager.h, uart_dma.h
   main.c ──> stm32u5xx_hal.h, FreeRTOS.h, task.h, app_tasks.h
```

### Rules

* `project_types.h` is a **leaf** — it includes only `<stdint.h>`,
  `<stdbool.h>`, `<stddef.h>`. No project header depends on it
  in the wrong direction.
* All other headers include `project_types.h` to get `pk_telemetry_t`,
  `pk_state_t`, etc.
* `.c` files include their own `.h` first so they pick up any
  declaration changes automatically.
* HAL-touching code (`adc_manager.c`, `motor_control.c`, `uart_dma.c`,
  `main.c`, `stm32u5xx_it.c`) is the **only** code that includes
  `stm32u5xx_hal.h`. This makes the rest of the firmware portable
  and host-testable.

---

## 2. Pin map reference — every physical pin we use

| Pin  | Direction | Peripheral  | Function                | Configured in                  | Alternate |
|------|-----------|-------------|-------------------------|--------------------------------|-----------|
| PA0  | INPUT     | ADC1 IN5    | Load cell force         | `adc_manager.c::pk_adc_config_channels` | — |
| PA1  | INPUT     | ADC1 IN6    | Knee angle pot          | `adc_manager.c::pk_adc_config_channels` | — |
| PA2  | INPUT     | ADC1 IN7    | Battery divider         | `adc_manager.c::pk_adc_config_channels` | ⚠ conflicts with USART2 TX |
| PA5  | OUTPUT    | GPIO        | LD1 LED (heartbeat)     | `main.c::GPIO_Init`           | — |
| PA6  | OUTPUT    | TIM3 CH1    | Motor A PWM (open)      | `main.c::TIM3_Init`           | AF2 |
| PA7  | OUTPUT    | TIM3 CH2    | Motor B PWM (close)     | `main.c::TIM3_Init`           | AF2 |
| PA9  | OUTPUT    | USART1 TX   | Bluetooth TX            | `main.c::USART1_Init`         | AF7 |
| PA10 | INPUT     | USART1 RX   | Bluetooth RX            | `main.c::USART1_Init`         | AF7 |
| PA3  | INPUT     | USART2 RX   | DMC external RX         | `main.c::USART2_Init`         | AF7 |
| PB0  | OUTPUT    | GPIO        | H-bridge enable         | `main.c::GPIO_Init`           | — |

### Detailed pin reasoning

**PA0/PA1/PA2 — Analog**
The U585's ADC1 inputs are multiplexed across pins. IN5 → PA0, IN6 →
PA1, IN7 → PA2 (channel number is fixed by the silicon, not a
software choice). We use these three because they are adjacent, so
the routing on the PCB stays short and the analog ground plane can
enclose all three traces.

**PA2 / PA3 — UART2 conflict**
PA2 is **both** ADC1 IN7 **and** USART2 TX. On the production board
the easiest fix is to move USART2 to PD5/PD6 (UART2's alternate
mapping). The firmware left the conflict in by accident; production
PCB layout must move USART2.

**PA5 — LED**
B-U585I-IOT02A Discovery has the green user LED on PA5. We drive it
from the lowest-priority task (`vWatchdogTask`) toggling every
500 ms. This is the visual heartbeat — if it stops blinking the
scheduler is wedged.

**PA6/PA7 — TIM3_CH1/CH2**
These two pins are TIM3 channels 1 and 2 in alternate function 2.
Driving a H-bridge requires two complementary PWM signals:
* CH1 → high-side FET of the open direction
* CH2 → high-side FET of the close direction
The low-side FETs are driven by the inverse logic in the H-bridge
hardware (active-low enable).

**PA9/PA10 — USART1**
The default USART1 mapping. AF7 routes the peripheral's TX/RX
onto these pins. The BT module plugs into the Arduino headers on
the Discovery board where these pins are exposed.

**PB0 — H-bridge enable**
Used as a hardware safety gate. Even if the firmware goes haywire
and toggles the PWM pins, the H-bridge stays off until this pin is
high. The motor_control module sets it high only after a successful
spool command, and clears it on every emergency stop.

### Clock tree

```
MSI @ 16 MHz ────► SYSCLK @ 16 MHz
                 ├─► HCLK @ 16 MHz (Cortex-M33 core)
                 ├─► PCLK1 (APB1) @ 16 MHz — TIM2, TIM3, IWDG
                 ├─► PCLK2 (APB2) @ 16 MHz — USART1, USART2
                 ├─► PCLK3 (APB3) @ 16 MHz — ADC1, GPDMA1
                 └─► MSI itself (for ADC async clock)
```

We deliberately do **not** use the PLL because:
* We don't need >16 MHz — the gait algorithm runs at 50 Hz, far
  below any M33 limit.
* PLL-on current at 160 MHz is ~5 mA; MSI-only is ~150 µA / MHz.
* Lower clock = less EMI, important near sensitive analog inputs.

---

## 3. `Application/Inc/project_types.h`

### 3.1 The MISRA-C and `__weak` / `__packed` macros

```c
#ifndef __weak
#define __weak   __attribute__((weak))
#endif
#ifndef __packed
#define __packed __attribute__((packed))
#endif
```

* `__weak` — allows the linker to override this symbol with a
  user-provided one. The HAL uses this to let you bypass
  `HAL_InitTick()`, `HAL_UART_TxCpltCallback()`, etc.
* `__packed` — tells GCC not to insert padding bytes inside a struct.
  Critical for our 10-byte telemetry packet.

### 3.2 `STATIC_ASSERT`

```c
#define STATIC_ASSERT(expr, msg) \
    typedef char static_assertion_##msg[(expr) ? 1 : -1]
```

If `expr` is false at compile time, the array dimension is `-1`
which is illegal. The compiler errors out and points at the line.
We use it once:

```c
STATIC_ASSERT(sizeof(pk_telemetry_t) == 10, telemetry_size_must_be_10_bytes);
```

If anyone adds a field or changes a type that changes the size,
the build breaks immediately.

### 3.3 Telemetry value types

```c
typedef uint8_t  pk_spool_angle_t;   // 0..100 %
typedef int16_t  pk_force_t;         // ±200 N (units: 0.1 N)
typedef int16_t  pk_knee_angle_t;    // ±200° (units: 0.1°)
typedef int16_t  pk_moment_t;        // ±3000 Nm (units: 0.01 Nm)
typedef uint8_t  pk_state_id_t;      // 0..16
typedef uint16_t pk_battery_mv_t;    // 0..5000 mV
```

Each typedef documents its physical range in the comment, and the
choice of underlying type is justified by the smallest type that
still holds the range with headroom.

### 3.4 `pk_telemetry_t`

```c
typedef struct __packed {
    pk_spool_angle_t spool_angle;   // byte 0
    pk_force_t       force;         // bytes 1..2 LE
    pk_knee_angle_t  knee_angle;    // bytes 3..4 LE
    pk_moment_t      moment;        // bytes 5..6 LE
    pk_state_id_t    state_id;      // byte 7
    pk_battery_mv_t  battery_mv;    // bytes 8..9 LE
} pk_telemetry_t;
```

`__packed` ensures no compiler-inserted padding. The `LE` comments
are a manual reminder that the encoder (`bluetooth.c::encode_telemetry`)
packs these explicitly little-endian.

### 3.5 State machine IDs

```c
typedef enum {
    PK_STATE_IDLE            = 0,
    PK_STATE_STANDING        = 1,
    PK_STATE_WALKING         = 2,
    PK_STATE_SITTING         = 3,
    PK_STATE_STAIR_ASCENT    = 4,
    PK_STATE_STAIR_DESCENT   = 5,
    PK_STATE_CALIBRATION     = 6,
    PK_STATE_FAULT           = 7,
    PK_STATE_SLEEP           = 8,
    PK_STATE_BOOT            = 9,
    PK_STATE_MAX             = 16
} pk_state_t;
```

Explicit numeric values let the GUI decode the state by index.
`PK_STATE_MAX` is the upper sentinel — anything ≥ this is invalid
and triggers a fault.

### 3.6 Fault codes

```c
typedef enum {
    PK_FAULT_NONE              = 0x00,
    PK_FAULT_ADC_TIMEOUT       = 0x01,
    ...
    PK_FAULT_OVERTEMP          = 0x0C
} pk_fault_t;
```

These codes are sent in the telemetry frame when `state == FAULT`
so the GUI can show "OVERTEMP" without parsing text.

---

## 4. `Application/Inc/packet_protocol.h`

### 4.1 Frame format constants

```c
#define PK_PROTO_START_BYTE       (0xAAu)
#define PK_PROTO_STOP_BYTE        (0x55u)
#define PK_PROTO_MAX_PAYLOAD      (32u)
#define PK_PROTO_MAX_FRAME        (PK_PROTO_MAX_PAYLOAD + 5u)
```

`0xAA` and `0x55` are complementary bits (10101010, 01010101) — they
have lots of edges for the receiver's clock-recovery circuit. The
max payload of 32 bytes covers all our current commands with room
to grow.

### 4.2 Command IDs

```c
typedef enum {
    PK_CMD_TELEMETRY_PUSH   = 0x01,
    PK_CMD_FORCE_STREAM     = 0x02,
    ...
    PK_CMD_BOOTLOADER_JUMP  = 0xF0
} pk_cmd_t;
```

The numeric values are baked into the firmware-GUI protocol
contract; **never renumber existing entries** — only add new ones
or mark old ones as deprecated.

### 4.3 Decoder state machine

```c
typedef enum {
    PK_DECODER_IDLE = 0,
    PK_DECODER_GOT_START,
    PK_DECODER_GOT_LEN,
    PK_DECODER_GOT_PAYLOAD,
    PK_DECODER_GOT_CMD,
    PK_DECODER_GOT_CRC,
    PK_DECODER_GOT_STOP,
    PK_DECODER_ERROR
} pk_decoder_state_t;
```

This is exposed in the header because `pk_packet_decode_byte()` takes
a byte-by-byte approach — the caller doesn't see the state, but the
enum documents the FSM for the reader.

---

## 5. `Application/Src/packet_protocol.c`

### 5.1 CRC-8 lookup table

```c
static const uint8_t crc8_table[256] = { ... };
```

256 bytes, generated offline by the polynomial 0x07. The table
itself is generated; the encoder function looks up the running CRC
XORed with the next byte:

```c
uint8_t pk_crc8(const uint8_t *buf, uint32_t len) {
    uint8_t crc = 0x00u;
    for (uint32_t i = 0u; i < len; i++) {
        crc = crc8_table[crc ^ buf[i]];
    }
    return crc;
}
```

This is the standard byte-at-a-time CRC. On Cortex-M33 the lookup
takes ~4 cycles (single LDR + single indexed LDR); for a 12-byte
frame that's ~50 cycles total.

### 5.2 `pk_packet_encode()`

```c
uint32_t pk_packet_encode(pk_cmd_t cmd,
                          const uint8_t *payload,
                          uint8_t plen,
                          uint8_t *out)
{
    if (plen > PK_PROTO_MAX_PAYLOAD || out == NULL) {
        return 0u;
    }
    uint32_t idx = 0u;
    out[idx++] = PK_PROTO_START_BYTE;
    out[idx++] = plen;
    if (plen > 0u && payload != NULL) {
        memcpy(&out[idx], payload, plen);
        idx += plen;
    }
    out[idx++] = (uint8_t)cmd;
    out[idx++] = pk_crc8(&out[1], (uint32_t)plen + 2u);
    out[idx++] = PK_PROTO_STOP_BYTE;
    return idx;
}
```

Builds the frame **exactly** in this order:
1. Start byte
2. Length
3. Payload (if any)
4. Command byte
5. CRC over bytes 1..plen+2 (len + payload + cmd)
6. Stop byte

Returns the total frame length, or 0 if the caller's payload is too
big.

### 5.3 Decoder FSM

The decoder is one large `switch` over the current state. Each
transition is documented in `Docs/ARCHITECTURE.md` §3 (the ASCII
diagram). Key details:

* **Reset on every illegal transition.** A corrupted frame is
  thrown away immediately — no buffering, no "wait and see".
* **`state` is file-private static** so two threads cannot race on
  it. ISR-callers are safe because USART idle-line ISRs only run
  once per packet.
* **`pk_packet_decoder_reset()`** clears state back to IDLE. Called
  from inside the decoder on error, and externally after a fault.

---

## 6. `Application/Inc/filter_lib.h` & `filter_lib.c`

### 6.1 Moving Average

```c
typedef struct {
    int32_t *ring;       // caller-supplied buffer of size `window`
    uint16_t window;
    uint16_t index;
    int64_t  sum;
    uint16_t count;
} pk_movavg_t;
```

* `ring` — pointer to a caller-allocated array. This lets the user
  decide whether it lives in `.bss` (fast) or on a task stack (cheap).
* `sum` is `int64_t` to avoid overflow when N=window and input is
  full-scale ADC. For our 14-bit ADC and N=8 the worst case is
  8 × 16383 = 131 064 — fits in 32 bits easily, but 64 keeps it
  safe up to N=1024.
* `count` lets us divide by the actual number of valid samples during
  the initial window-fill period.

### 6.2 First-Order IIR (float)

```c
typedef struct {
    float alpha;
    float prev;
    bool  seeded;
} pk_iir1_t;

float pk_iir1_push(pk_iir1_t *f, float sample) {
    if (!f->seeded) {
        f->prev   = sample;
        f->seeded = true;
        return sample;
    }
    f->prev = (f->alpha * sample) + ((1.0f - f->alpha) * f->prev);
    return f->prev;
}
```

The `seeded` flag means the first sample goes through unchanged
(we don't have a "previous" yet). After that, classic EMA:
`y[n] = α·x + (1-α)·y[n-1]`.

For `α = 0.20` at a 200 Hz update rate the -3 dB cutoff is
`f_c ≈ α·fs/(2π) ≈ 6.4 Hz` — perfect for filtering 50 Hz mains
hum and gait sensor noise.

### 6.3 Q15 EMA (integer, fixed-point)

```c
typedef struct {
    int32_t state;        // Q15
    int32_t alpha_q15;    // Q15
    bool    seeded;
} pk_ema_t;

int32_t pk_ema_push(pk_ema_t *f, int32_t sample) {
    if (!f->seeded) {
        f->state  = sample << 15;
        f->seeded = true;
        return sample;
    }
    int64_t a_term = (int64_t)f->alpha_q15 * (int64_t)sample;
    int64_t b_term = ((int64_t)(32767 - f->alpha_q15) * (int64_t)f->state) >> 15;
    int64_t sum    = (a_term + b_term);
    f->state = (int32_t)sum;
    return (int32_t)(sum >> 15);
}
```

The math:

```
   y[n]    = α · x[n] + (1-α) · y[n-1]
   y_q15   = (α_q15 · x) + ((32767-α_q15) · y_q15) >> 15
```

The `>> 15` cancels the Q15 scaling. The intermediate is `int64_t`
so we don't overflow during the multiply (32 × 32 = 64 bits).

### 6.4 `PK_ALPHA_Q15`

```c
#define PK_ALPHA_Q15(a) ((int32_t)((a) * 32767.0f))
```

Converts a float α in [0,1] to Q15. We use `32767` not `32768`
because Q15 stores values in `[-32768, +32767]`, and we want α to
stay strictly positive.

---

## 7. `Application/Inc/adc_manager.h` & `adc_manager.c`

### 7.1 Channels and buffers

```c
#define PK_ADC_CHANNELS        (3u)          // force / angle / battery
#define PK_ADC_SAMPLES_PER_CH  (16u)         // HW oversampling
#define PK_ADC_BUF_LEN         (PK_ADC_CHANNELS * PK_ADC_SAMPLES_PER_CH * 2u)
```

The buffer is `CHANNELS × SAMPLES × 2` because we use **ping-pong**
DMA — the HAL fires an ISR at half-full and full-full so we can
process one half while the other fills.

### 7.2 `pk_adc_init()`

Configures the channel ranks and oversampling, returns HAL_OK or
HAL_ERROR. Does **not** start the ADC.

### 7.3 `pk_adc_start()`

```c
HAL_StatusTypeDef pk_adc_start(void) {
    if (HAL_ADC_Start_DMA(&hadc1,
                          (uint32_t *)s_adc_buf,
                          PK_ADC_BUF_LEN) != HAL_OK) {
        return HAL_ERROR;
    }
    if (HAL_TIM_Base_Start(&htim2) != HAL_OK) {
        return HAL_ERROR;
    }
    return HAL_OK;
}
```

`HAL_ADC_Start_DMA()` configures DMA1 channel 0 in circular mode.
`HAL_TIM_Base_Start()` starts TIM2 which generates the 1 kHz trigger
pulses that cause the ADC to convert.

### 7.4 `pk_adc_calibrate()`

```c
void pk_adc_calibrate(void) {
    (void)HAL_ADCEx_Calibration_Start(&hadc1);
    (void)memset(&s_data, 0, sizeof(s_data));
}
```

Mandatory on STM32U5 after every reset — performs internal offset
calibration using the on-chip reference.

### 7.5 ISR hooks

```c
void pk_adc_dma_half_xfer_isr(void) {
    for (uint32_t i = 0u; i < PK_ADC_CH__COUNT; i++) {
        uint16_t s = s_adc_buf[s_rank_offset[i]];
        s_data.raw[i]  = s;
        s_data.filt[i] = pk_ema_push(&s_ema[i], (int32_t)s);
    }
    s_data.ready = true;
}
```

Called from `DMA1_Channel0_IRQHandler` (in `stm32u5xx_it.c`) when
the DMA pointer reaches the half-buffer mark or wraps to the start.
Reads the **most recent** sample for each channel and pushes it
through the per-channel EMA filter.

---

## 8. `Application/Inc/motor_control.h` & `motor_control.c`

### 8.1 PWM frequency and ramp

```c
#define PK_PWM_PERIOD_TICKS      (16000u)
#define PK_PWM_DEADTIME_TICKS    (400u)
#define PK_PWM_RAMP_PER_MS       (1u)
```

* `PERIOD_TICKS = 16000` with a 16 MHz timer clock → **1 kHz PWM**.
  Above human hearing (inaudible), smooth torque.
* `DEADTIME_TICKS = 400` → 25 µs of dead-band between the two motor
  PWMs to prevent H-bridge shoot-through.
* `RAMP_PER_MS = 1` → duty cycle changes by 1 tick per ms. From 0 to
  100 % takes 16 seconds; smooth, no audible clack, low inrush.

### 8.2 State variables

```c
static struct {
    uint16_t target[2];
    uint16_t current[2];
    bool     enabled;
    uint32_t stall_ms[2];
} s_mot;
```

* `target[m]` — what `pk_motor_set_target()` set
* `current[m]` — what we actually applied after ramping
* `enabled` — whether the H-bridge enable pin is high
* `stall_ms[m]` — how many ms the motor has been unable to reach
  its target (used for stall detection)

### 8.3 `pk_apply_hw()`

```c
static void pk_apply_hw(pk_motor_id_t m, uint16_t duty) {
    if (m == PK_MOTOR_A) {
        __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, duty);
        __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_2, 0u);
    } else {
        __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 0u);
        __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_2, duty);
    }
}
```

Always force the opposite channel to zero. This guarantees that
when motor A is being driven, motor B is disabled — even if a
stale `target` somehow lingers.

### 8.4 `pk_motor_tick_1ms()` — the heart of the controller

Called from `vMotorTask` every 1 ms:

1. If `enabled == false`, apply 0 duty to both channels and return.
2. For each motor: ramp `current` toward `target`, capped at
   `RAMP_PER_MS`.
3. **Dead-time check** — if both targets > 0, force the smaller to 0.
4. If `current` is stuck at the wrong value (i.e. the ramp finished
   but `current != target`), increment `stall_ms[m]`. After 100 ms,
   call `pk_motor_emergency_stop()`.

### 8.5 `pk_motor_set_spool()`

```c
void pk_motor_set_spool(pk_spool_angle_t pct) {
    uint32_t duty = ((uint32_t)pct * PK_PWM_PERIOD_TICKS) / 100u;
    if (pct <= 50u) {
        pk_motor_set_target(PK_MOTOR_B, (uint16_t)duty);   // closing
        pk_motor_set_target(PK_MOTOR_A, 0u);
    } else {
        pk_motor_set_target(PK_MOTOR_A, (uint16_t)duty);   // opening
        pk_motor_set_target(PK_MOTOR_B, 0u);
    }
    s_mot.enabled = true;
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);
}
```

The "spool position" abstraction — the FSM doesn't have to know
about PWM ticks. It just says "I want 70 % open" and this function
maps it to the right motor and duty.

---

## 9. `Application/Inc/uart_dma.h` & `uart_dma.c`

### 9.1 `pk_uart_id_t`

```c
typedef enum {
    PK_UART_BT  = 0,   // USART1 — Bluetooth
    PK_UART_DMC = 1,   // USART2 — DMC external
    PK_UART__COUNT
} pk_uart_id_t;
```

A small enum gives us array indexing (`s_ch[PK_UART_BT]`) without
the overhead of pointers.

### 9.2 `pk_uart_pkt_t`

```c
typedef struct {
    pk_uart_id_t  port;
    pk_cmd_t      cmd;
    uint8_t       payload[PK_PROTO_MAX_PAYLOAD];
    uint8_t       plen;
} pk_uart_pkt_t;
```

The struct that is pushed into the FreeRTOS queue every time a
complete packet is decoded. Held in the header because both the
producer (ISR) and the consumer (task) need to agree on its layout.

### 9.3 `pk_uart_init()`

For each channel:

1. Allocate a FreeRTOS queue (8 packets deep)
2. Reset the head pointer
3. Call `HAL_UARTEx_ReceiveToIdle_DMA()` to start circular RX DMA
4. Disable the half-transfer interrupt (we use the idle line for
   framing, not the half-buffer)

### 9.4 `pk_uart_send()`

```c
HAL_StatusTypeDef pk_uart_send(pk_uart_id_t which,
                               const uint8_t *frame,
                               uint32_t len) {
    if (which >= PK_UART__COUNT || frame == NULL || len == 0u) {
        return HAL_ERROR;
    }
    pk_uart_ch_t *c = &s_ch[which];
    uint32_t cap = sizeof(c->rx_buf);
    if (len > cap) len = cap;
    memcpy(c->rx_buf, frame, len);
    HAL_StatusTypeDef st = HAL_UART_Transmit_DMA(c->huart, c->rx_buf, (uint16_t)len);
    return st;
}
```

Copies the frame into the per-channel scratch buffer (so the caller
can free their source) and starts the DMA. We re-use the RX buffer
because the RX ISR only touches the head pointer, not the bytes.

### 9.5 `pk_uart_rx_isr()`

```c
void pk_uart_rx_isr(pk_uart_id_t which) {
    pk_uart_ch_t *c = &s_ch[which];
    uint16_t ndtr = __HAL_DMA_GET_COUNTER(c->huart->hdmarx);
    uint16_t newest = (uint16_t)(PK_UART_RX_BUF_LEN - ndtr);
    while (c->rx_head != newest) {
        uint8_t b = c->rx_buf[c->rx_head];
        c->rx_head = (uint16_t)((c->rx_head + 1u) % PK_UART_RX_BUF_LEN);
        pk_cmd_t   cmd;
        uint8_t    payload[PK_PROTO_MAX_PAYLOAD];
        uint8_t    plen;
        if (pk_packet_decode_byte(b, &cmd, payload, &plen)) {
            pk_uart_pkt_t pkt = { .port = which, .cmd = cmd, .plen = plen };
            if (plen > 0u) memcpy(pkt.payload, payload, plen);
            BaseType_t hpw = pdFALSE;
            (void)xQueueSendFromISR(c->rx_queue, &pkt, &hpw);
            portYIELD_FROM_ISR(hpw);
        }
    }
}
```

Called from `USART1_IRQHandler` / `USART2_IRQHandler` when the
USART detects the idle line.

* `ndtr` = how many bytes are left to transfer (= how many bytes
  we have already received, since DMA is circular)
* `newest` = the absolute index of the most-recently-written byte
* Walk from `rx_head` to `newest`, feeding each byte into the
  packet decoder
* On complete frame, push the decoded struct into the queue
* `portYIELD_FROM_ISR(hpw)` forces a context switch if the task
  waiting on the queue is higher priority

---

## 10. `Application/Inc/bluetooth.h` & `bluetooth.c`

### 10.1 `encode_telemetry()`

```c
static uint32_t encode_telemetry(const pk_telemetry_t *t) {
    uint8_t payload[sizeof(pk_telemetry_t)];
    uint32_t i = 0u;
    payload[i++] = t->spool_angle;
    payload[i++] = (uint8_t)(t->force & 0xFFu);
    payload[i++] = (uint8_t)((t->force >> 8) & 0xFFu);
    payload[i++] = (uint8_t)(t->knee_angle & 0xFFu);
    payload[i++] = (uint8_t)((t->knee_angle >> 8) & 0xFFu);
    payload[i++] = (uint8_t)(t->moment & 0xFFu);
    payload[i++] = (uint8_t)((t->moment >> 8) & 0xFFu);
    payload[i++] = t->state_id;
    payload[i++] = (uint8_t)(t->battery_mv & 0xFFu);
    payload[i++] = (uint8_t)((t->battery_mv >> 8) & 0xFFu);
    return pk_packet_encode(PK_CMD_TELEMETRY_PUSH, payload, (uint8_t)i, s_tx_buf);
}
```

Manual little-endian packing. The `__packed` attribute on the struct
means the compiler would normally emit the same bytes, but doing it
explicitly is more portable and easier to read.

### 10.2 `vBluetoothTask()`

```c
void vBluetoothTask(void *pv) {
    (void)pv;
    QueueHandle_t q = (QueueHandle_t)pk_uart_rx_queue(PK_UART_BT);
    const TickType_t period = pdMS_TO_TICKS(20u);   // 50 Hz
    TickType_t last = xTaskGetTickCount();
    for (;;) {
        vTaskDelayUntil(&last, period);    // sleep until next 20 ms tick
        uint32_t n = encode_telemetry((const pk_telemetry_t *)&g_telemetry);
        (void)pk_uart_send(PK_UART_BT, s_tx_buf, n);
        pk_uart_pkt_t pkt;
        while (xQueueReceive(q, &pkt, 0) == pdTRUE) {
            handle_command(pkt.cmd, pkt.payload, pkt.plen);
        }
    }
}
```

`vTaskDelayUntil` (not `vTaskDelay`) keeps the cadence at exactly
50 Hz even if the task wakes up slightly late.

The command drain is non-blocking — `xQueueReceive(q, &pkt, 0)`
returns immediately if the queue is empty.

---

## 11. `Application/Inc/state_machine.h` & `state_machine.c`

### 11.1 Context struct

```c
typedef struct {
    pk_state_t  current;
    pk_state_t  previous;
    pk_fault_t  last_fault;
    uint32_t    state_enter_tick;
    uint32_t    force_accum_ms;
    uint16_t    heel_strike_count;
    pk_spool_angle_t current_spool;
} pk_sm_ctx_t;
```

Every field has a clear purpose:

* `current` — drives the next telemetry frame and the next state
  transition
* `previous` — for "you just came from WALKING" queries
* `last_fault` — shown in the GUI when state == FAULT
* `state_enter_tick` — for time-in-state diagnostics
* `force_accum_ms` — debounce counter for standing detection
* `heel_strike_count` — how many heel-strikes since entering
  STANDING
* `current_spool` — what the FSM last asked the motor to do

### 11.2 `pk_sm_step()`

The big switch:

```c
switch (ctx->current) {
case PK_STATE_IDLE:
    ctx->current_spool = 50;
    if (adc->filt[PK_ADC_CH_FORCE] > PK_FORCE_STAND_THRESH_N) {
        ctx->force_accum_ms += 5u;
        if (ctx->force_accum_ms > 200u) {
            enter_state(ctx, PK_STATE_STANDING);
        }
    } else {
        ctx->force_accum_ms = 0u;
    }
    break;
...
}
```

`force_accum_ms` is incremented by 5 because the ADC task calls
`pk_sm_step` every 5 ms. After 40 calls (>200 ms) the user has been
standing long enough to commit to STANDING state.

### 11.3 `pk_sm_raise_fault()`

```c
void pk_sm_raise_fault(pk_sm_ctx_t *ctx, pk_fault_t f) {
    ctx->last_fault = f;
    enter_state(ctx, PK_STATE_FAULT);
    pk_motor_emergency_stop();
}
```

The atomic safety primitive: record fault, change state, kill
motors. Every safety check in the firmware calls this.

---

## 12. `Application/Inc/safety.h` & `safety.c`

### 12.1 Heartbeat array

```c
static volatile uint32_t g_task_heartbeat[PK_TASK__COUNT] = {0};
static volatile uint32_t g_task_last_seen[PK_TASK__COUNT]  = {0};
```

Two arrays: one for the actual last-heartbeat time, one that we
keep for diagnostic logging.

### 12.2 `pk_safety_task()`

Runs at 1 ms, the highest priority:

1. For every other task, compute `now - heartbeat[id]`. If
   `> max_silence[id]` → fault.
2. Check battery voltage thresholds.
3. Refresh IWDG.
4. Update its own heartbeat.

The `max_silence` values are tuned per task period:

| Task      | Period | max_silence |
|-----------|--------|-------------|
| MOTOR     | 1 ms   | 10 ms       |
| ADC       | 5 ms   | 25 ms       |
| COMMS     | 10 ms  | 50 ms       |
| BT        | 20 ms  | 100 ms      |
| WDT       | 500 ms | 200 ms      |

A safety margin of 2-5× the period absorbs scheduler jitter.

---

## 13. `Application/Inc/app_tasks.h` & `app_tasks.c`

### 13.1 Priority definitions

```c
#define PK_PRIO_SAFETY   (configMAX_PRIORITIES - 1)
#define PK_PRIO_MOTOR    (configMAX_PRIORITIES - 2)
#define PK_PRIO_ADC      (configMAX_PRIORITIES - 3)
#define PK_PRIO_COMMS    (configMAX_PRIORITIES - 4)
#define PK_PRIO_BT       (configMAX_PRIORITIES - 5)
#define PK_PRIO_WDT      (tskIDLE_PRIORITY + 1)
```

`configMAX_PRIORITIES = 16`, so:
* SAFETY = 15
* MOTOR  = 14
* ADC    = 13
* COMMS  = 12
* BT     = 11
* WDT    = 2

### 13.2 Conversion functions

```c
static pk_force_t raw_to_force(uint16_t raw) {
    int32_t c = (int32_t)raw - 8192;     // centre at half-scale (0 N)
    return (pk_force_t)(c / 8);          // 8 counts = 0.1 N
}
static pk_knee_angle_t raw_to_knee_angle(uint16_t raw) {
    int32_t c = (int32_t)raw - 8192;
    return (pk_knee_angle_t)((c * 100) / 455);  // 4.55 counts = 0.1°
}
static pk_battery_mv_t raw_to_battery_mv(uint16_t raw) {
    return (pk_battery_mv_t)(((uint32_t)raw * 5000u) / 16384u);
}
```

These are the only place that knows the physical calibration of
each ADC channel. Change the constants here and the rest of the
firmware still works in engineering units.

### 13.3 `vAdcTask()`

```c
void vAdcTask(void *pv) {
    const TickType_t period = pdMS_TO_TICKS(5u);
    TickType_t last = xTaskGetTickCount();
    for (;;) {
        vTaskDelayUntil(&last, period);
        pk_adc_data_t d;
        if (pk_adc_get(&d)) {
            pk_telemetry_t t;
            t.spool_angle = g_sm.current_spool;
            t.force       = raw_to_force((uint16_t)d.filt[PK_ADC_CH_FORCE]);
            ...
            g_telemetry = t;
        }
        pk_sm_step(&g_sm, &d, raw_to_knee_angle((uint16_t)d.filt[PK_ADC_CH_ANGLE]));
        pk_safety_heartbeat(PK_TASK_ADC);
    }
}
```

This is the *only* task that writes `g_telemetry`. The BT task reads
it. Because the write is a single-word `volatile` store on
Cortex-M33, the read in BT always sees a consistent snapshot.

### 13.4 `pk_boot_start_tasks()`

Called from `main()` before `vTaskStartScheduler()`:

1. Initialise ADC, motor, UART, packet decoder, state machine,
   safety module
2. Calibrate the ADC
3. Start the ADC (also starts TIM2 trigger)
4. Create all six tasks with their priorities and stack sizes

---

## 14. `Core/Inc/FreeRTOSConfig.h`

### 14.1 Scheduler core

```c
#define configUSE_PREEMPTION            1
#define configUSE_TIME_SLICING          1
#define configMAX_PRIORITIES            16
#define configTICK_RATE_HZ              (1000u)
#define configMINIMAL_STACK_SIZE        (128u)
#define configUSE_16_BIT_TICKS          0
```

* `USE_PREEMPTION = 1` — higher-priority tasks can interrupt lower
* `TIME_SLICING = 1` — same-priority tasks share CPU fairly
* `MAX_PRIORITIES = 16` — limited to save RAM (each TCB needs a
  priority bitmap)
* `TICK_RATE_HZ = 1000` — 1 ms tick is the unit of all `vTaskDelay()`
  calls
* `MINIMAL_STACK_SIZE = 128` — small idle stack, tasks use more
* `USE_16_BIT_TICKS = 0` — 32-bit tick counter (no overflow in
  49 days; we reset the watchdog more often)

### 14.2 Heap

```c
#define configSUPPORT_DYNAMIC_ALLOCATION    1
#define configSUPPORT_STATIC_ALLOCATION     0
```

We use dynamic allocation. The heap is `heap_4.c` from the portable
layer.

### 14.3 Interrupts

```c
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY       0x0F
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY  0x05
```

The upper 4 bits of the NVIC priority byte. SysTick runs at
priority 0x05 (lowest "FromISR" priority). Our USART/DMA ISRs run
at 0x05 too so they can call `xQueueSendFromISR`.

### 14.4 Watchdog / assert

```c
#define configUSE_MALLOC_FAILED_HOOK        1
#define configUSE_STACK_OVERFLOW_HOOK       2
#define configASSERT(x) ...  /* halts the CPU */
```

If `pvPortMalloc()` ever returns NULL we trap in the failed-hook.
If any task overflows its stack we trap too (method 2 = check
canary).

---

## 15. `Core/Inc/stm32u5xx_it.h` & `Core/Src/stm32u5xx_it.c`

### 15.1 Prototype-only header

`stm32u5xx_it.h` declares **only** function prototypes. No code. This
prevents the linker from seeing multiple definitions of any handler
during a CubeMX regeneration.

### 15.2 FreeRTOS hook redirection

```c
void xPortPendSVHandler(void)       { vPortPendSVHandler(); }
void xPortSysTickHandler(void)      { vPortSysTickHandler(); }
void vPortSVCHandler(void)          { vPortSVCHandler(); }
```

The HAL defines weak versions of these (they're the standard ARMv8-M
exception names). FreeRTOS uses different names. We redirect the
ARM-standard names to the FreeRTOS names so the startup vector table
works unchanged.

### 15.3 SysTick_Handler

```c
void SysTick_Handler(void) {
    portSYSCALL_SUSPEND_BEGIN();
    vPortSysTickHandler();
    portSYNCALL_SUSPEND_END();
}
```

`portSYSCALL_SUSPEND_BEGIN/END` are macros that guarantee the
SysTick handler doesn't accidentally try to call a FreeRTOS API
while inside a critical section.

### 15.4 DMA1_Channel0_IRQHandler (ADC)

We bypass HAL_ADC_ConvCpltCallback entirely and instead read the
DMA flags directly:

```c
uint32_t isr = DMA1->ISR;
if (isr & (DMA_ISR_HTIF0 | DMA_ISR_TCIF0)) {
    if (isr & DMA_ISR_HTIF0) {
        pk_adc_dma_half_xfer_isr();
        DMA1->IFCR = DMA_IFCR_CHTIF0;   // clear half-transfer flag
    }
    if (isr & DMA_ISR_TCIF0) {
        pk_adc_dma_full_xfer_isr();
        DMA1->IFCR = DMA_IFCR_CTCIF0;   // clear full-transfer flag
    }
}
```

This is faster and more predictable than the HAL's callback
mechanism.

### 15.5 USART idle-line handlers

```c
uint32_t sr = USART1->ISR;
if (sr & USART_ISR_IDLE) {
    (void)USART1->ISR;     // read to clear IDLE flag
    (void)USART1->RDR;
    pk_uart_rx_isr(0u);    // PK_UART_BT = 0
}
HAL_UART_IRQHandler(&huart1);   // for TX-complete callbacks
```

The IDLE flag is cleared by reading ISR then RDR (per the reference
manual). `HAL_UART_IRQHandler` is still called so the HAL can
handle TX-complete interrupts.

---

## 16. `Core/Src/main.c`

### 16.1 `HAL_InitTick()` override

```c
HAL_StatusTypeDef HAL_InitTick(uint32_t TickPriority) {
    (void)TickPriority;
    return HAL_OK;
}
```

The HAL normally sets up SysTick here. We override it to do
nothing — SysTick is owned by FreeRTOS.

### 16.2 `main()`

```c
int main(void) {
    MPU_Config();
    HAL_Init();
    SystemClock_Config();
    GPIO_Init();
    DMA_Init();
    ADC1_Init();
    USART1_Init();
    USART2_Init();
    TIM2_Init();
    TIM3_Init();
    IWDG_Init();
    pk_boot_start_tasks();
    vTaskStartScheduler();
    for (;;) {}
}
```

Strict init order: clocks → GPIO → DMA → ADC → UARTs → TIMs →
watchdog → tasks → scheduler. The watchdog is initialised **before**
the scheduler starts, so a hung FreeRTOS boot will reset the MCU.

### 16.3 `SystemClock_Config()`

Sets MSI to range 4 (16 MHz), no PLL. All bus dividers = 1.

### 16.4 `GPIO_Init()`

See pin map in §2. Sets PA5 and PB0 as outputs, PA0/PA1/PA2 as
analog inputs, PA6/PA7/PA9/PA10/PA2/PA3 as alternate function.

### 16.5 `DMA_Init()`

Just enables the GPDMA1 clock. Per-channel configuration is in the
HAL ADC/UART MSP init functions (left to the reader to fill in
based on the HAL pack).

### 16.6 `ADC1_Init()`

`hadc1.Init.ClockPrescaler = ADC_CLOCK_ASYNC_DIV1` — the ADC runs
asynchronously from the bus clock, dividing the 16 MHz MSI down
to its own 16 MHz domain.

`ExternalTrigConv = ADC_EXTERNALTRIG_T2_TRGO` — TIM2 trigger output
starts each conversion.

### 16.7 `USART1_Init()` / `USART2_Init()`

Standard 115200 8N1, oversampling 16. Both share the same struct
so `USART2_Init` is just `huart2.Init = huart1.Init`.

### 16.8 `TIM2_Init()`

Period = 15999 → 1 kHz trigger. Master mode = TRGO_UPDATE so the
update event fires the ADC.

### 16.9 `TIM3_Init()`

PWM channels 1 and 2. Period = 15999 → 1 kHz PWM.

### 16.10 `IWDG_Init()`

```c
hiwdg1.Init.Prescaler = IWDG_PRESCALER_64;       // 32 kHz / 64 = 500 Hz
hiwdg1.Init.Reload    = 500u - 1u;               // 1 s timeout
```

If the safety task doesn't refresh this within 1 s the MCU resets.

---

## 17. `Core/Src/startup_stm32u585.s`

### 17.1 Stack and heap reservation

```asm
.equ    Stack_Size,  0x00004000   // 16 KB main stack
.equ    Heap_Size,   0x00002000   //  8 KB FreeRTOS heap
```

These symbols are placed in `.bss` so the linker can locate them.

### 17.2 Vector table

The first 16 entries are the ARMv8-M system exceptions:

```asm
.word   __StackTop                  // 0x00  Initial SP
.word   Reset_Handler               // 0x04  Reset
.word   NMI_Handler                 // 0x08  NMI
.word   HardFault_Handler           // 0x0C
.word   MemManage_Handler           // 0x10
.word   BusFault_Handler            // 0x14
.word   UsageFault_Handler          // 0x18
.word   0                           // 0x1C..0x28  Reserved
.word   vPortSVCHandler             // 0x2C  SVCall -> FreeRTOS
.word   DebugMon_Handler            // 0x30
.word   0                           // 0x34  Reserved
.word   xPortPendSVHandler          // 0x38  PendSV -> FreeRTOS
.word   xPortSysTickHandler         // 0x3C  SysTick -> FreeRTOS
```

The remaining entries are peripheral interrupts. We populate the
ones we use (DMA1_Channel0, USART1, USART2, TIM2) and leave the
others as 0 — the linker will trap if an unhandled IRQ fires.

### 17.3 Reset_Handler

1. Copy `.data` from flash to SRAM
2. Zero `.bss`
3. Call `SystemInit()` then `main()`

The `.weak` aliases at the bottom mean the linker will resolve any
unbound symbol to `Default_Handler` (which is an infinite loop).

---

## 18. `Core/Src/stm32u585.ld`

### 18.1 Memory regions

```ld
MEMORY
{
    FLASH (rx)   : ORIGIN = 0x08000000, LENGTH = 256K
    RAM   (rwx)  : ORIGIN = 0x20000000, LENGTH = 256K
}
```

We only use the first 256 KB of the 1 MB flash — plenty for this
firmware. The upper 768 KB is reserved for the OTA bootloader and
Bank-2 telemetry logs.

### 18.2 Sections

* `.isr_vector` — first in flash, at address 0x08000000
* `.text`, `.rodata` — code and constants
* `.data` — initialised globals, copied from flash to SRAM at boot
* `.bss` — zero-initialised globals
* `._user_heap_stack` — 8 KB reserved for `pvPortMalloc()`

---

## 19. `Tests/host_test.c`

Five unit tests that run on the host PC:

1. **CRC-8 known vector** — `CRC("123456789") == 0xF4`
2. **Packet round-trip** — encode → decode → compare
3. **CRC corruption** — flip a bit → decoder rejects the frame
4. **Moving average** — settles to 0 after feeding four zeros
5. **EMA Q15** — converges geometrically

All five must pass before checking in changes.

---

## 20. Build sequence (`Makefile`)

```bash
make          # compile + link
make size     # print section sizes
make flash    # openocd program + verify + reset
make clean    # rm -rf build *.elf *.map
```

The Makefile:

* Uses `arm-none-eabi-gcc` only (no HAL pack needed for *analysis*,
  but compilation of `adc_manager.c`, `motor_control.c`, `main.c`,
  etc. needs the HAL headers in `Drivers/STM32U5xx_HAL_Driver/Inc`)
* Builds the FreeRTOS kernel + portable layer + heap_4
* Generates `.elf`, `.map`, `.bin`, `.hex`
* Strips debug symbols with `-ffunction-sections -fdata-sections`
  and the linker `--gc-sections`

---

## Quick lookup — "where is X defined?"

| Feature                          | File                                  | Function / variable         |
|----------------------------------|---------------------------------------|-----------------------------|
| Telemetry struct layout          | `project_types.h`                     | `pk_telemetry_t`            |
| Wire format start/stop bytes     | `packet_protocol.h`                   | `PK_PROTO_START_BYTE`       |
| CRC table                        | `packet_protocol.c`                   | `crc8_table[]`              |
| ADC oversampling setup           | `adc_manager.c`                       | `pk_adc_config_oversampling` |
| ADC ISR hooks                    | `adc_manager.c`                       | `pk_adc_dma_half_xfer_isr`  |
| PWM duty setting                 | `motor_control.c`                     | `pk_apply_hw()`             |
| Dead-time enforcement            | `motor_control.c`                     | `pk_motor_tick_1ms`         |
| UART idle-line RX                | `stm32u5xx_it.c`                      | `USART1_IRQHandler`         |
| UART packet decode               | `uart_dma.c`                          | `pk_uart_rx_isr`            |
| BT telemetry encoding            | `bluetooth.c`                         | `encode_telemetry`          |
| Gait state transitions           | `state_machine.c`                     | `pk_sm_step`                |
| Watchdog refresh                 | `safety.c`                            | `pk_safety_task`            |
| Task creation                    | `app_tasks.c`                         | `pk_boot_start_tasks`       |
| HAL tick bypass                  | `main.c`                              | `HAL_InitTick`              |
| Clock setup                      | `main.c`                              | `SystemClock_Config`        |
| Vector table                     | `startup_stm32u585.s`                 | `__isr_vector`              |
| Stack/heap sizes                 | `startup_stm32u585.s`                 | `Stack_Size`, `Heap_Size`   |
| Memory layout                    | `stm32u585.ld`                        | `MEMORY`, `SECTIONS`        |

---

## End of walkthrough

If anything in here is unclear, ping me with the section number and
I'll expand it.
