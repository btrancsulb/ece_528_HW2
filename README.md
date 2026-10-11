# ECE 528/L Homework 2

bruh

## 1. Hardware timer

A hardware timer is an on-chip counter clocked by a hardware clock. It counts clock edges on its own, so the timing does not depend on how long a software loop takes to execute. Software sets up the timer, then the peripheral counts, compares, captures, or overflows and can raise an interrupt or drive a pin while the CPU does other work.

Embedded systems commonly use hardware timers for:

- Periodic interrupts, such as a scheduler tick or a control-loop sample
- Pulse-width modulation for motors, servos, and LEDs
- Input capture, to measure pulse width, frequency, or encoder edges
- interrupts and delays

## 2. Clock signal and MSP432P401R clock sources

A clock signal is a continuous square wave that steps synchronous logic. On a microcontroller it sets how fast the CPU executes instructions and it is the time base for timers, serial baud rates, and converters.

The MSP432P401R clock system has these sources:

| Source | What it is |
| --- | --- |
| DCOCLK | Internal digitally controlled oscillator. Programmable from about 1 MHz to 48 MHz. After reset it runs at 3 MHz and is the default source for MCLK, HSMCLK, and SMCLK. |
| LFXTCLK | Low-frequency oscillator for an external 32.768 kHz crystal, or a digital clock in bypass mode. Used for accurate low-power timing. The LaunchPad fits this crystal. |
| HFXTCLK | High-frequency oscillator for an external crystal from 1 MHz to 48 MHz, or a digital clock in bypass mode. The LaunchPad fits a 48 MHz crystal. |
| REFOCLK | Internal trimmed low-frequency reference. Selectable as 32.768 kHz or 128 kHz. It is the fallback for ACLK and BCLK when LFXT is not running. |
| VLOCLK | Internal very-low-power oscillator, typically 9.4 kHz. Low accuracy, used when power matters more than precision. |
| MODOSC | Internal module oscillator, typically 25 MHz. Used by modules such as the flash controller. |
| SYSOSC | Internal system oscillator, typically 5 MHz. Available as a system clock and as a fail-safe source. |

Those sources are divided and routed onto the system clocks MCLK (CPU), HSMCLK, SMCLK, ACLK, and BCLK.

## 3. Maximum count of a 16-bit timer

A 16-bit counter has \(2^{16}\) distinct values:

\[
2^{16} = 65536
\]

Those values are 0 through 65535. The largest count is

\[
2^{16} - 1 = 65535 = \texttt{0xFFFF}
\]

The next clock rolls the count back to 0. In Timer_A up mode the roll point is `TAxCCR0`, and the largest value that register can hold is also 65535.

## 4. Overflow period of a 24-bit timer at 10 MHz

An upward 24-bit counter overflows every \(2^{24}\) clocks (the time from one rollover to the next):

\[
2^{24} = 16777216 \text{ counts}
\]

\[
T = \frac{16777216}{10 \times 10^{6} \text{ Hz}} = 1.6777216 \text{ s}
\]

## 5. SysTick on the MSP432P401R

SysTick is the 24-bit system timer built into the Cortex-M4F. It counts down from a reload value to zero, then reloads and can set `COUNTFLAG` and request a SysTick exception. It stops while the processor is halted in a debugger.

Two uses:

- An RTOS or scheduler tick, for example a 1 ms periodic interrupt
- A software time base for delays, timeouts, or a `millis()` counter

On this device the only SysTick clock is the CPU free-running clock (FCLK, the same clock as MCLK). There is no external reference clock. `STCR.NOREF` reads as 1, and `STCSR.CLKSOURCE` (bit 2) is held at 1, which selects the core clock. Writing 0 is not supported.

## 6. SysTick registers

SysTick has four registers (TRM Table 2-53):

| Register | Purpose |
| --- | --- |
| STCSR (control and status) | `ENABLE` (bit 0) starts the counter. `TICKINT` (bit 1) enables the SysTick exception. `CLKSOURCE` (bit 2) reads as 1 (core clock). `COUNTFLAG` (bit 16) is set when the count reaches 0 and clears when the register is read. |
| STRVR (reload value) | Holds the 24-bit value copied into the current-value register when the count reaches 0. The period is `RELOAD + 1` core clocks. |
| STCVR (current value) | Holds the live 24-bit count. A write of any value clears the count and `COUNTFLAG`. |
| STCR (calibration) | Read-only. `NOREF` reads as 1 (no separate reference clock). `SKEW` reads as 1 and `TENMS` reads as 0, so there is no exact 10 ms calibration value. |

Startup order from the TRM: write `STRVR`, write `STCVR` to clear it, then write `STCSR`.

## 7. Timer_A and its modes

Timer_A is a 16-bit timer/counter with capture/compare channels. It can time intervals, capture input edges, request interrupts, and generate PWM. The clock source and dividers are selected in `TAxCTL` and `TAxEX0`. The mode is the `MC` field of `TAxCTL` (bits 5-4).

| Mode | MC | Behavior |
| --- | --- | --- |
| Stop | `00` | The counter is halted. |
| Up | `01` | Counts from 0 to `TAxCCR0`, then restarts at 0. The interval is `TAxCCR0 + 1` timer clocks. `TAxCCR0` sets the period. |
| Continuous | `10` | Counts from 0 to `0xFFFF` and rolls to 0. The overflow flag is set on the rollover from `0xFFFF` to 0. |
| Up/down | `11` | Counts from 0 up to `TAxCCR0`, then back down to 0. The period is `2 * TAxCCR0` timer clocks. Used for symmetric, center-aligned PWM. |

## 8. Timer_A registers

The homework lists `TaxIV`; that register is `TAxIV`.

| Register | Purpose |
| --- | --- |
| TAxCTL | Timer control. Selects the clock (`TASSEL`), the input divider (`ID`), and the mode (`MC`). Also clears the counter (`TACLR`), enables the overflow interrupt (`TAIE`), and holds the overflow flag (`TAIFG`). |
| TAxCCTLn | Capture/compare control for channel n. Selects capture versus compare, the capture edge and input, the output mode (`OUTMOD`), and the channel interrupt (`CCIE`, `CCIFG`). |
| TAxR | The 16-bit counter. This is the current count. |
| TAxCCRn | Capture/compare value for channel n. In compare mode, channel 0 usually holds the period and the other channels hold PWM thresholds. In capture mode, the timer count is latched here when the selected edge arrives. |
| TAxIV | Interrupt vector. Reading it returns which of CCR1–CCR4 or the overflow flag is the highest pending source, and it clears that flag. CCR0 has its own interrupt vector and is not reported here. |
| TAxEX0 | Expansion control. `TAIDEX` (bits 2-0) divides the timer clock by `TAIDEX + 1` (1 through 8), in addition to the `ID` divider in `TAxCTL`. |

## 9. PWM

Pulse-width modulation is a digital waveform with a fixed period whose high time is changed. The duty cycle is the fraction of the period that the signal is high, so the average voltage (and the power delivered to a load) follows that fraction.

Examples:

- DC motor speed control
- Hobby-servo position (a 1 ms to 2 ms pulse inside a 20 ms frame)
- LED brightness
- A switching regulator or heater, where duty cycle sets average power

## 10. Timer_A modules and PWM outputs

The MSP432P401R has **four** Timer_A modules: Timer_A0, Timer_A1, Timer_A2, and Timer_A3.

Each module has **five** capture/compare registers (`CCR0` through `CCR4`) and five PWM outputs (`TAx.0` through `TAx.4`). In up or up/down mode, `CCR0` sets the shared period and `CCR1`–`CCR4` set independent duty cycles. `CCR0` can still drive its own output pin.

## 11. P2.4 PWM from Timer_A0 CCR1 (TA0.1)

P2.4’s primary module function is TA0.1, the Timer_A0 CCR1 output. Pin function select is `SEL1:SEL0 = 01` for the primary function.

Clock and waveform:

- SMCLK = 12 MHz
- `ID = 01b` divides that clock by 2, and `TAIDEX = 000b` divides by 1, so the timer clock is 6 MHz
- Up/down mode: the counter goes 0 → `CCR0` → 0, so

\[
T = \frac{2 \times \texttt{TA0CCR0}}{f_{\text{timer}}}
\]

\[
\texttt{TA0CCR0} = \frac{0.020 \times 6{,}000{,}000}{2} = 60000 = \texttt{0xEA60}
\]

- `OUTMOD = 010b` is Toggle/Reset. In up/down mode the high time is \(2 \times \texttt{CCR1}\) timer clocks and the period is \(2 \times \texttt{CCR0}\) timer clocks, so the duty cycle is `CCR1 / CCR0`. For 50%:

\[
\texttt{TA0CCR1} = \frac{60000}{2} = 30000 = \texttt{0x7530}
\]

`TAxCTL` value `0x0274`:

- bits 9-8 = `10` → SMCLK (`0x0200`)
- bits 7-6 = `01` → divide by 2 (`0x0040`)
- bits 5-4 = `11` → up/down (`0x0030`)
- bit 2 = 1 → clear the counter (`0x0004`)

`TAxCCTL1` value `0x0040`: bits 7-5 = `010`, Toggle/Reset.

```c
void PWM_P24_Init(void)
{
    P2->DIR  |= 0x0010;        /* P2.4 output */
    P2->SEL1 &= ~0x0010;       /* SEL1:SEL0 = 01: primary function (TA0.1) */
    P2->SEL0 |=  0x0010;

    TIMER_A0->CCR[0]  = 0xEA60; /* 60000: 20 ms period in up/down mode */
    TIMER_A0->CCR[1]  = 0x7530; /* 30000: 50% duty */
    TIMER_A0->CCTL[1] = 0x0040; /* OUTMOD = 010b, Toggle/Reset */
    TIMER_A0->EX0     = 0x0000; /* TAIDEX = 000b, divide by 1 */
    TIMER_A0->CTL     = 0x0274; /* SMCLK, /2, up/down, clear and start */
}
```

## 12. Timer_A0 interrupt every 2 ms, priority 2

Timer_A0 CCR0 is interrupt number 8 (`TA0_0`). That vector is the CCR0 compare interrupt, so the channel interrupt is enabled in `CCTL[0]`, not the overflow interrupt in `TAxCTL`.

Clock, as required:

- `TASSEL = 10b` selects SMCLK (12 MHz)
- `ID = 00b` divides by 1
- `TAIDEX = 000b` divides by 1
- Up mode, so one interrupt period is `TA0CCR0 + 1` timer clocks

\[
N = 0.002 \times 12{,}000{,}000 = 24000
\]

\[
\texttt{TA0CCR0} = 24000 - 1 = 23999 = \texttt{0x5DBF}
\]

The MSP432 implements 3 priority bits, in bits 7-5 of the NVIC priority byte. Priority 2 is `010b` in those bits:

\[
2 \ll 5 = \texttt{0x40}
\]

`NVIC->IP` in the CCS CMSIS header is one byte per IRQ, so byte 8 is interrupt 8. Bit 8 of `ISER[0]` enables that IRQ (`0x00000100`).

```c
void TimerA0_2ms_Init(void)
{
    TIMER_A0->CTL     = 0x0200;  /* SMCLK, ID = 00b (/1), halted */
    TIMER_A0->EX0     = 0x0000;  /* TAIDEX = 000b (/1) */
    TIMER_A0->CCR[0]  = 0x5DBF;  /* 23999: (CCR0 + 1) / 12 MHz = 2 ms */
    TIMER_A0->CCTL[0] = 0x0010;  /* compare mode, CCIE = 1, CCIFG clear */

    NVIC->IP[8]    = 0x40;       /* IRQ 8, priority 2 */
    NVIC->ISER[0]  = 0x00000100; /* enable interrupt 8 */

    TIMER_A0->CTL |= 0x0014;     /* TACLR, MC = 01b (up). Timer starts. */
}

void TA0_0_IRQHandler(void)
{
    TIMER_A0->CCTL[0] &= ~0x0001; /* clear CCIFG (bit 0) */
}
```
