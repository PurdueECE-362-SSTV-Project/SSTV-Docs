# SSTV: Slow Scan Television

Captures images transmitted over FM radio.

The format is still in active use. The International Space Station transmits SSTV images on 145.8 MHz during special events, usually commemorative pictures sent over several days. The signal is strong enough that anyone under the pass with a handheld radio and an antenna can receive them, which makes it one of the few ways to pull something directly off a spacecraft with hobbyist equipment. This receiver is built for that band.

***

<p align="center">
  <img src="diagrams/Core%20&%20OS/images/SSTV%20Design%20Workflow.png" width="650">
  <br>
  <em>Two-phase development workflow</em>
</p>

***

## How It Works

SSTV sends pictures as sound. Each pixel's brightness becomes an audio tone: dark pixels are low tones, bright pixels are high ones. The transmitter plays its way across the image one line at a time, and that audio goes out over FM radio like any other broadcast. A full picture takes tens of seconds to arrive, which is where the "slow scan" name comes from.

Receiving one means reversing those steps:

1. **Radio to audio.** Tune the 145 MHz signal and demodulate it to get the sound back.
2. **Audio to tones.** Measure what frequency the audio is playing, moment by moment. This is the difficult part.
3. **Tones to picture.** Turn each frequency back into a brightness, find the markers that separate one line from the next, and stack the lines into an image.

This project does all three on a **Proton** board (an RP2350 with two CPU cores) paired with a custom radio front-end. One core handles the radio, display, and controls. The other does the signal processing. Everything from capture to finished picture happens on the device.

***

## How the System Fits Together

The receiver is a one-way pipeline. It crosses from the analog world into the digital one exactly once, at the analog-to-digital converter. Everything before that point is radio and audio hardware. Everything after it is software.

```
   Antenna                                    ┌──────────── RP2350 (Proton) ────────────┐
      │                                       │                                         │
      ▼                                       │   Core 1: I/O          Core 2: DSP      │
  ┌────────┐   85 MHz   ┌──────────┐  audio   │  ┌───────────┐       ┌──────────────┐   │
  │ LT5512 │ ────IF───▶ │  Si4703  │ ───────▶ │  │ ADC + DMA │ ────▶ │ PLL demod    │   │
  │ mixer  │            │ FM tuner │          │  └───────────┘       │ level detect │   │
  └────────┘            └──────────┘          │        │             │ line decode  │   │
      ▲                      ▲                │        │ messages    └──────────────┘   │
   145 MHz RF            I2C control          │        │                     │          │
      │                      │                │        ▼                     │          │
   60 MHz LO                 └────────────────┼─ keypad / rotary enc.        │          │
                                              │  OLED display  ◀─────────────┘          │
                                              └─────────────────────────────────────────┘
```

**Core 1** reads the ADC, drives the display, and watches the keypad and rotary encoder. **Core 2** runs demodulation and image reconstruction. Large blocks of audio samples travel between the cores through shared memory using DMA. Short notifications travel through a hardware mailbox. Keeping bulk data out of the mailbox, and keeping small signals out of DMA, is the rule the whole messaging design is built around.

***

## Development Roadmap

### Phase 1
Phase 1 focuses on the barebone design of the SSTV
The main goal is to get a working model before we can add more complexity

List of Goals: 
1) Define Inputs
2) Configure Outputs
3) Core working
   - Demodulate
   - De-encode
   - Image Processing (encoding)
4) Display Menu
5) Image Display

### Phase 2
Phase 2 focues on more QOL (Quality of Life) changes in the design. It makes it easier to use and utilize in the real world.

List of Goals: 
1) Memory (SD) with (Image, Metadata); (Frequency, Metadata)
2) On Board Power Supply and Battery Monitor
3) RTC for Transmission Source times of operation
4) Cleaner UI

***

## Radio Front-End

SSTV on the 2 metre amateur band sits near 145 MHz. That is far above what any inexpensive FM receiver chip can tune, so the front-end uses a **superheterodyne** design: shift the signal down to a frequency a common tuner already handles, and let that tuner do the FM demodulation.

| Stage | Part | What it does |
| :--- | :--- | :--- |
| Mixer | LT5512 | Multiplies the incoming 145 MHz signal against a 60 MHz local oscillator |
| Intermediate frequency | | The useful product lands at 85 MHz |
| Tuner | Si4703 | Tunes that signal, demodulates the FM, and outputs audio |

Multiplying two signals together produces both a sum and a difference. The 85 MHz difference is the one kept, and it sits comfortably inside the Si4703's tuning range. From the tuner's point of view, an SSTV transmission now looks like an ordinary FM radio station.

### Controlling the Tuner

The Si4703 is configured over **I2C** at address `0x10`, running at 400 kHz. Start-up has to happen in a specific order:

1. Pulse the reset line low then high. The chip decides whether it is an I2C or SPI device at the moment of reset, so this cannot be corrected afterwards.
2. Enable the crystal oscillator and wait roughly 500 ms for it to settle.
3. Enable the receiver, unmute, and set the volume.

Tuning works by writing a channel number and setting a "tune" bit, then waiting for the chip to report that it is done. Channel number and frequency are related by:

$$\text{channel} = \frac{f_{\text{MHz}} \times 10 - 875}{2}$$

which reflects the chip's 87.5 MHz starting point and 200 kHz spacing between channels. After tuning, the chip reports a signal strength value, which the interface uses to show how good the reception is.

***

## Capturing the Audio

Audio from the tuner is sampled by the RP2350's built-in ADC and moved into memory entirely by DMA, so the processor never has to touch the sample stream itself.

- The **ADC** runs continuously and signals the DMA engine each time a new sample is ready.
- The **DMA channel** reads from a fixed address (the ADC output) and writes into an incrementing position in a 512-sample buffer.
- The channel is set to **restart itself** when it finishes. The result is a continuous recording loop that needs no attention from either core.

A ping-pong buffer arrangement is planned next, so Core 2 can process one half of the buffer while the DMA engine fills the other.

***

## Recovering the Image: The Software PLL

This is the heart of the project, and the reason the algorithm lives in its own repository. It was prototyped and tuned in Python against real recorded SSTV audio before any of it was written in C++.

### Why a Phase-Locked Loop

After the tuner, the picture information lives in the audio's *frequency*. Measuring frequency from raw samples is awkward. A Fourier transform gives precise frequency but poor timing, and SSTV needs both, because the tone changes every few hundred microseconds. A **phase-locked loop** solves this by tracking continuously rather than analysing in blocks. It runs its own internal oscillator and constantly steers it to match the incoming signal. Whatever correction the loop is applying *is* the answer, and it is available on every single sample.

```
input ──▶ [ bandpass ] ──▶ ( × ) ──▶ [ lowpass ] ──▶ [ phase detect ] ──┐
                             ▲                                          │
                             │                                          │ phase error
                        ┌────┴─────┐                                    │
                        │ internal │ ◀─── [ loop gain ] ◀───────────────┘
                        │   osc.   │
                        └──────────┘
                             │
                             └──▶ frequency estimate ──▶ [ lowpass ] ──▶ output
```

### What Each Stage Does

- **Internal oscillator.** Generates a reference tone whose frequency is adjustable. It idles at 1750 Hz, chosen to sit midway between the 1200 Hz sync tone and the 2300 Hz white level, so it is never far from whatever it needs to lock onto.
- **Mixer.** Multiplies the incoming audio against the reference tone. The result carries the phase difference between the two, which is the error signal the loop needs.
- **Filters.** Clean up the mixer output and remove unwanted products. These are written to process one sample at a time rather than as block operations, specifically so the same logic transfers directly to firmware running on a live audio stream.
- **Phase detector.** Measures how far the reference has drifted from the input since the last sample. This difference, scaled by a gain factor, becomes the correction applied to the oscillator.
- **Lock detector.** Watches how much the output is jumping around. A steady output means the loop is tracking a real tone and the result can be trusted. A noisy one means it is still searching, and the decoder should wait.

### Filter Bands

| Filter | Type | Range | Purpose |
| :--- | :--- | :--- | :--- |
| Input | Bandpass | 1200–2300 Hz | Restrict to the range SSTV actually uses |
| Loop | Lowpass | 2300 Hz | Keep the useful mixer output, discard the rest |
| Output | Lowpass | 2400 Hz | Smooth the final frequency estimate |

### Open Problems

Two questions remain from the prototyping work:

- **How fast the loop settles.** The loop converges in a curve very similar to a capacitor charging. Relating the loop's settings to that settling time would let the design guarantee it locks fast enough to keep up with the incoming lines.
- **Telling a real level from noise.** Each pixel is a brief period where the frequency holds steady. Reliably distinguishing a genuine change in brightness from a momentary noise spike, quickly enough not to smear neighbouring pixels together, is the remaining obstacle to decoding a complete picture.

***

## Software Architecture

### Choosing Bare Metal

An early design study compared running a real-time operating system against writing directly to the hardware:

| Criteria | Weight | Bare Metal | RTOS |
| :--- | :--- | :--- | :--- |
| Performance | 0.3 | High | High |
| Power efficiency | 0.2 | High | Medium |
| Development time | 0.2 | High | Medium |
| Ease of implementation | 0.3 | Low | High |
| **Total** | | **0.5** | **0.8** |

FreeRTOS scored higher and was chosen first, mainly for easier multicore coordination and a large existing ecosystem. The study also recorded the risks: it might not be compatible, it might use too much memory, and the team should stay willing to reverse the decision.

<p align="center">
  <img src="diagrams/Core%20&%20OS/images/RTOS%20Workflow.png" width="550">
  <br>
  <em>The originally planned FreeRTOS structure</em>
</p>

Porting FreeRTOS to the Proton required building new support for the RP2350 from scratch. That work reached the point of compiling but stalled on a fault at runtime. The fallback plan was used, and the project moved to bare metal with a messaging layer built for this specific application rather than a general-purpose scheduler.

The result does exactly what this pipeline needs and nothing more.

### The Messaging Layer

The two cores are treated as separate programs that exchange explicit messages, rather than quietly sharing variables. The design follows one rule, stated directly in the source:

> The FIFO queue should never be used for bulk data transfers. It is solely for triggering interrupts between cores for IO-bound tasks, i.e., as a notifier that something must be done. If you need more than 26 bits, then the FIFO is not for you. Use shared memory and DMA in that case.

That rule produces three separate transport mechanisms, each suited to a different job.

#### Short notifications

The RP2350 provides a hardware mailbox between the two cores that carries a single 32-bit word. The project gives that word a structure:

| Field | Bits | Purpose |
| :--- | :--- | :--- |
| Destination | 3 | Which handler should receive this, up to 8 per core |
| Source | 3 | Which core or subsystem sent it |
| Data | 26 | A small payload: a command, an index, or a flag |

Twenty-six bits is deliberately *just enough* to point at something. It is plenty to say "image 7 is ready, put it on the display", and nowhere near enough to tempt anyone into sending the image itself.

Each core keeps a table of handlers indexed by destination. When a message arrives, an interrupt empties the mailbox and calls the matching handler. A handler can only be registered once, so two parts of the program cannot quietly claim the same address.

Handlers must never wait for anything. Blocking inside one would freeze the whole system, so a handler that needs to do real work hands it off: it sets a flag, queues the job, or triggers a lower-priority interrupt. The receiving interrupt runs at high priority so messages are never missed, while the sending side runs at low priority so it cannot starve the main program. A watchdog timer catches the case where a send stalls.

#### Bulk data

Audio and image buffers move through dedicated DMA channels acting as private pipes between the cores. Each pipe tracks its own state, moving from idle, to waiting for the receiver to be ready, to waiting for the transfer to finish.

The important detail is that the **receiver** decides when a transfer may start, not the sender. This makes it impossible for a producer to overwrite a buffer the consumer is still reading.

#### Shared buffers

For memory both cores need to reach, a buffer is paired with a lock. The preferred way to use it hands in a function to run while the lock is held, which makes it structurally impossible to forget to release it.

The interesting case is a handoff assisted by DMA. A producer takes the lock, starts a transfer, and then deliberately does *not* release it. Instead, the interrupt that fires when the transfer completes releases it. The lock therefore protects the data for the real duration of the transfer rather than just the function call that began it, leaving the processor free to do other work in the meantime.

#### Pipeline memory

For multi-stage processing, a single allocator owns every buffer and feeds them into the first stage. Buffers then travel from stage to stage, each one taking a buffer from its input queue, working on it, and passing it to the next. Because exactly one component ever allocates or frees, there is no fragmentation and no confusion about ownership. A buffer is always in exactly one place.

Each stage picks how it runs based on what it does. Stages waiting on hardware, like audio capture, are driven by interrupts. Stages doing heavy computation, like the PLL, run in the main program loop where taking a long time is acceptable.

***

## Display and Controls

<p align="center">
  <img src="diagrams/Display%20&%20Menu/SSTV%20Menu%20UI%20Workflow.png" width="620">
  <br>
  <em>Menu navigation</em>
</p>

Navigation starts at a boot screen and opens into a main menu, shown above.

**Image display** draws each line as it is decoded rather than waiting for the full picture. This gives immediate feedback while tuning, because a badly tuned signal shows up as visible skew or noise within a few seconds instead of a minute later.

**Frequency entry** uses significant-digit control. The rotary encoder picks which digit is active and turning it changes that digit, so coarse and fine tuning both happen quickly without switching modes.

**Load and store**, planned for Phase 2, keeps two libraries on an SD card: a gallery of received pictures, and a list of stations pairing a name with its frequency and the times it usually transmits.

<p align="center">
  <img src="diagrams/Display%20&%20Menu/SSTV%20Menu%20UI%20Implementation.png" width="620">
  <br>
  <em>Screen layouts</em>
</p>

### Controls

| Control | Role |
| :--- | :--- |
| Rotary encoder | Tuning, scrolling menus, selecting |
| 4×4 keypad | Typing a frequency directly |
| Buttons | Back, select, and cycling between fields |

The keypad is read by driving one column at a time and letting an interrupt fire when a row line goes high. The interrupt works out which key was pressed from the combination of row and column.

***

## Enclosure

The receiver is housed in a 3D-printed case styled after a vintage television set. The display sits recessed in the bezel where the picture tube would be, with the keypad and rotary encoder as front-panel controls. It is a deliberate nod to what SSTV actually is: television, just very slowly.

<table>
<tr>
<td width="50%" align="center">
  <img src="diagrams/CADs%20&%20PCBs/images/cad-old-tv-frame.gif" width="100%">
  <br>
  <strong>Television-inspired frame</strong><br>
  <em>Main enclosure</em>
</td>
<td width="50%" align="center">
  <img src="diagrams/CADs%20&%20PCBs/images/cad-makeshift-cover.gif" width="100%">
  <br>
  <strong>Makeshift cover</strong><br>
  <em>Simplified alternative</em>
</td>
</tr>
</table>

Alongside the main case there is a simplified faceplate, which is faster to print and easier to lift off for access to the boards during testing.

**Full models and earlier iterations: [CADs & PCBs](diagrams/CADs%20&%20PCBs/readme.md)**

***

## Building the Firmware

The firmware builds with **PlatformIO** and defines two targets.

**Hardware target.** Builds against the Pico SDK in C++17 for the Proton board, and flashes and debugs over a hardware debug probe. Optimisation is deliberately turned off so that stepping through signal-processing code in a debugger behaves predictably.

**Host target.** Compiles only the hardware-independent half of the codebase and runs its tests on a normal computer. This is possible because the code is split along a deliberate line:

| Layer | Depends on |
| :--- | :--- |
| Portable | Nothing hardware-specific |
| Hardware | The Pico SDK and RP2350 registers |

Queues, optional values, and the message packing and unpacking logic live on the portable side and can be fully tested without a board plugged in. Interrupt handling, DMA setup, and anything touching a hardware register live on the other side.

### Configuration

Tunable values are collected in a single configuration header rather than scattered through the code:

| Setting | Default | Meaning |
| :--- | :--- | :--- |
| Queue depth | 256 | Messages buffered before sending |
| Interrupt priority | 5 | Priority of the message interrupt, 0 is highest |
| DMA buffer size | 1024 | Words per bulk transfer |
| Pipeline buffers | 256 | Buffers circulating through processing stages |
| Message channels | 3 | DMA channels reserved, with a hardware maximum of 6 |

Several of these are checked when the code is compiled rather than when it runs. Buffer sizes that are not powers of two, message fields that do not add up to 32 bits, and requests for more channels than the chip has all stop the build with a clear error instead of failing quietly on real hardware.

***

## Testing

Testing is split the same way the code is.

**Host tests** run on a normal computer with no board attached. They cover the data structures and logic that have no hardware dependency: queue behaviour at its boundaries, optional value handling, and verifying that messages survive being packed into 32 bits and unpacked again.

**On-board tests** run on the Proton itself, for the things that only exist with real hardware: interrupt timing, DMA completion, audio capture integrity, and decoding a picture end to end.

The Python prototype acts as the reference. Filter behaviour and loop performance are confirmed there first against recorded SSTV audio, and the firmware is then checked to make sure it produces the same results.

***

## Repository Layout

The project spans several repositories:

| Repository | Contents |
| :--- | :--- |
| **[SSTV-Docs](https://github.com/PurdueECE-362-SSTV-Project/SSTV-Docs)** | Design documents, diagrams, and this overview |
| **[SSTV-Core](https://github.com/PurdueECE-362-SSTV-Project/SSTV-Core)** | Firmware: messaging layer, drivers, and signal processing |
| **[SSTV-Hardware](https://github.com/PurdueECE-362-SSTV-Project/SSTV-Hardware)** | Schematics and PCB layouts for the radio front-end and carrier boards |
| **[SSTV-Algo-Prototype](https://github.com/PurdueECE-362-SSTV-Project/SSTV-Algo-Prototype)** | Python reference implementation of the demodulator |

Within this repository:

| Folder | Contents |
| :--- | :--- |
| [`diagrams/Core & OS`](diagrams/Core%20&%20OS) | System architecture and messaging design |
| [`diagrams/Display & Menu`](diagrams/Display%20&%20Menu) | Menu structure and screen layouts |
| [`diagrams/CADs & PCBs`](diagrams/CADs%20&%20PCBs) | Enclosure models |
| [`Meeting Notes`](Meeting%20Notes) | Design proposals and planning documents |
