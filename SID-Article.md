# SID-Article v0.3

**_MOS Technology SID soundchip internals and applications on the Commodore 64_**

## What is the SID Chip?

The Sound Interface Device (or SID) soundchip contributed heavily to the success
of the Commodore 64 personal computer (usually abbreviated as _C64_). It's
unique among the soundchips of the microcomputer era with its outstanding
capabilities and sound quality. Designed in 1982 by a team led by Bob Yannes, it
featured synthesizer techniques yet unseen with other computer brands: an analog
filter, mixing abilities combined with highly flexible digital control of 3
separate analog oscillators with pitch, timbre, and volume control and with
cutoff curves.

The SID chip in essence is a digitally controlled analog synthesizer. Although a
lot can be controlled just through the digital registers of the SID chip itself,
even more is possible when a computer program is updating those registers at a
rapid rate. In modern terms you can think of these programs - on the Commodore
64 usually written in 6510 assembly language - as sophisticated sequencers that
can also utilize LFOs (e.g. for vibrato effect), custom macro capabilities
(e.g. for wavetables), and several other tricks.

Soon after the Commodore 64 was released the SID became a beloved playground of
great musicians making music for the burgeoning game industry, with people like
Rob Hubbard, Ben Daglish, Martin Galway, Tim Follin, just to name a few of the
early pioneers. Thanks to its capabilities the SID chip can still achieve juicy
sounds today in good hands, be it any music genre or trend, like Drum'n'Bass or
Dubstep wave. Every now and then new techniques utilizing the SID are revealed
by creative demoscene enthusiasts who create new pieces of music for games,
demos, or just for fun.

## What is This Document About?

This document covers mostly the internals of how the SID works behind the
scenes, but that doesn't mean this document is for geeks only. This document
strives to be an easy-to-read explanation, gradually increasing its complexity.
Even in the harder parts it tries to explain the concepts in the simplest
possible way, accompanied by pictures.

SID musicians who want to get a deeper understandig of the _Hard Restart_ and
other mysterious workings of the SID will probably find parts of this document
just as beneficial as music-player routine developers or implementers of
software/hardware SID-emulation engines. Whatever you use in your life, you can
utilize it better if you know its internals rather than treating it as a mere
black-box. It's up to you whether you feel curious enough to compose SID music
by instinct or whether you try to build upon a solid knowledge of the internals
of the SID chip.

There are many resourceful materials about the SID on the Internet already, but
it can be very tiresome to dig them all up from different places. There is an
overall technical document about the graphic chip of the C64 called VIC-article[^1].
Surprisingly there hasn't been a similar document for the SID - until now.
Some of the sources used here was an interview Hermit made with Bob Yannes,
the source-code comments of Dag Lem's ReSID, the ReSID-FP engine,
the Kevtris reverse-engineering site, a document about DC levels
by Levente Hársfalvi, and Hermit's own findings that will be explained
later. So hopefully this document collects sufficient information for you in
this article in one place to get the big picture.

## Revisions and Location of the SID Chip

There are two major versions of the SID chip called by their part numbers: the
original one is called __6581__ and the newer, revised SID is called __8580__.
The 6581 can be found in the top-middle region of the old C64 mainboards and it
requires a 12V power-supply for its analog circuitry besides the 5V digital
supply. The 8580 is found slightly to the right at the bottom of new C64 boards
and it requires 9V and 5V power rails.

There are two external capacitors to support the filter circuits integrated into
SID. These capacitors are quite different in value for the old and new models,
therefore the SID versions are not readily interchangeable without any modding.
6581 models also requires a 1 kOhm resistor towards ground on their output for
the simplified output-stage driver circuit. At least their 28-pin DIP package
form-factor is the same so they fit into each others' sockets without any
hassles.

Both models are nearly equivalent in their digital portions, but the different
silicon process they are based on (6581:NMOS, 8580:HMOS-II) and the different
designs result in clear differences how their analog circuits sound like,
especially in mixing, filter curves, and with combined waveforms. More on these
later. There are big differences even between 6581 revisions themselves[^2]. (A
6582 model was also released but its internals are the same as that of the 8580,
only the labels differ.)

[^1]: https://www.cebix.net/VIC-Article.txt

[^2]: As per Lagerfeldt's findings, the differences
in 6581s are due to the filter components, not due to the chip revisions
themselves. See [Mythbusting the 6581 revisions](https://ultimatesid.dk/).

## Pinout of the SID Chip

```
            +-----------+
    CAP1A --|  1     28 |-- Vdd
    CAP1B --|  2     27 |-- AUDIO OUT
    CAP2A --|  3     26 |-- EXT IN
    CAP2B --|  4     25 |-- Vcc
     !RES --|  5     24 |-- POT X
    !PHI2 --|  6     23 |-- POT Y
     R/!W --|  7     22 |-- D7
      !CS --|  8     21 |-- D6
       A0 --|  9     20 |-- D5
       A1 --| 10     19 |-- D4
       A2 --| 11     18 |-- D3
       A3 --| 12     17 |-- D2
       A4 --| 13     16 |-- D1
      GND --| 14     15 |-- D0
            +-----------+
```

| Pin name      | Description    |
| ------------- | -------------- |
| CAP1A, CAP1B  | Filter capacitor 1 (6581: 470 pF, 8580: 22 nF |
| CAP2A, CAP2B  | Filter capacitor 2 (6581: 470 pF, 8580: 22 nF |
| !RES          | Reset input - if low for at least 10 phi2 cycles, all internal registers reset |
| PHI2          | Input for system oscillator, receives data only when high |
| R/!W          | High = read allowed, Low = write allowed |
| !CS           | Chip Select - active low input, bus data needs to be valid when active |
| A0..A4        | Address inputs to select one of the 32 internal registers |
| GND           | Ground |
| Vdd           | Second voltage (6581: +12 VDC, 8580: +9 VDC) |
| AUDIO OUT     | Audio outout, 6 VDC (6581) 4.75 VDC (8580) 3Vp-p at max volume |
| EXT IN        | External audio input, mixes with SID output and can be filtered. |
|               | (8580 needs a ca. 330 kOhm to GND on this pin to fix old digi sounds.) |
| Vcc           | Main voltage, +5 VDC |
| POT X         | Input for potentiometer (paddle) X-axis |
| POT Y         | Input for potentiometer (paddle) Y-axis |
| D0..D7        | Data bus bits 0..7 |


## How Does the C64 Control the SID? (Registers)

As you can see on the pinout diagram above the SID chip has a 5-bit address bus
through which a theoretical total of 32 internal 8-bit registers of the chip can
be accessed. Of the 32 _possible_ registers  29 are actually utilized in the SID
chip (addresses `$00..$1C` hexadecimal - addresses `$1D-$1F` are not used). On a
Commodore 64 these registers are mapped to memory addresses (a.k.a. _Memory-
mapped I/O_ ) in the range of `$D400..$D41C` (hexadecimal). This means that the
Commodore 64 sees the SID at these memory addresses and it can write to the
internal registers ('control-bytes') of the SID just like to any other portion
of the memory. Whenever you write to these addresses you essentially modify the
flip-flops inside the SID, which in turn set parameters like pitch, envelope,
filter, etc. in real-time. Simple, isn't it? (With modern hardware mods more SID
chips can be added to the C64 and in that case their base-addresses differ from
`$D400`. There's no set specification, yet, for what address they should reside
at.)

Most registers are write-only and you can't read them back, but there is also a
little feedback from SID towards the C64 in the form of read-only registers, not
to mention bit-fading which makes tricks like Hein's `ROR $D400,X` possible.

__TODO__: This ROR trick is never explained in this document. Also, needs reference link.

Internally the SID chip consists of 3 digitally controlled oscillators with
analog outputs which are mixed together to the analog AUDIO OUT pin of the chip.
In addition, it is possible to route any combination of the 3 analog outputs to
an analog filter stage before the final audio output. At any given point in time
the SID chip can produce at most 3 distinct sounds simultaneously. By the way,
this is true even when the SID chip is playing a digi sample 'over' the 3
oscillators - in that case the digital sample is technically just an _artifact_,
an _illusion_ as the SID's hardware is still only producing 3 oscillated sounds
at once.

NOTE: 'Voices', 'channels' and 'oscillators' are usually used interchangeably
when discussing the SID chip. In this document we'll refer to them as
'channels'.

With that said, let's examine the registers for the 3 channels one-by-one.

### Pitch

| Channel   | Low byte     | High byte     |
| --------- |:------------:|:-------------:|
| Channel 1 | `$D400`      | `$D401`       |
| Channel 2 | `$D407`      | `$D408`       |
| Channel 3 | `$D40E`      | `$D40F`       |

Pitch low and high-byte. These bytes together control the pitch of an
oscillator. 16 bits give us quite enough resolution to make perfect pitches in
the region of 15Hz to 3848Hz (PAL) with equal steps of cca 0.06Hz. The human ear
and brain perceives pitches in a non-linear fashion, that is we hear less
difference between the equal frequency steps in higher regions than in the lower
ones. Every upper octave has twice the frequency of its lower counterpart. So
for scales of musical notes we need to have a lookup table (or _frequency
table_) to map the notes into frequency values. In the most widely used equally
tempered chromatic (Western) scale successive notes have the frequency ratio of
12th root of 2.

### Pulse Width (Duty-cycle) of the Square Waveform

| Channel   | Low byte     | High byte (bits 0..3 only)     |
| --------- |:------------:|:------------------------------:|
| Channel 1 | `$D402`      | `$D403`                        |
| Channel 2 | `$D409`      | `$D40A`                        |
| Channel 3 | `$D410`      | `$D411`                        |

Pulse width (duty-cycle) of the square waveform. This is a really important
feature of the SID chip because variations of the pulse (square) wave have very
different spectral characteristics. This is really useful for smooth transitions
between timbres (called _sweeps_) that make it sound more lively than just a
monotonic beep. Most of the time this is used to create lead instruments. Fast-
sweeping the pulse width enriches the spectrum of the sound and adds a kind of
chorus effect.

Luckily, the shape of some combined waveforms can also be altered by the duty-
cycle setting, giving even more timbres to choose from. The upper byte has only
the lower nibble (lower 4 bits) wired in so there are 4096 possibilities of
pulsewidth to choose from, between 0 and 100% duty-cycle from the thinnest to
the fattest sound. One of the strengths of the SID is that it essentially
operates at 1MHz 'sampling' frequency and the thinnest sounds are clear. This is
not always the case with emulated SID sounds of only 44kHz or so.

### Waveform and Envelope Control

| Channel   | Byte         |
| --------- |:------------:|
| Channel 1 | `$D404`      |
| Channel 2 | `$D40B`      |
| Channel 3 | `$D412`      |

The bits in this register control different things separately. The upper nibble
controls which of the 4 available waveforms are turned on. They can also be
turned on at the same time due to the properties of the underlying silicon
technology. This results in the the so-called 'combined waveforms' which Bob
Yannes is officially against of, which is understandable because these connect
some outputs together. Nevertheless, these have been used in many masterworks
from the beginning. The new 8580 has more defined and louder combined waveforms.
How this is done deserves a separate topic in this article, so stay tuned.

#### Waveform Control Bits

| __Bit__       | __Control__    |
| ------------- |:--------------:|
| Bit 0 (`$01`) | GATE           |
| Bit 1 (`$02`) | SYNC           |
| Bit 2 (`$04`) | RING           |
| Bit 3 (`$08`) | TEST           |
| Bit 4 (`$10`) | Triangle       |
| Bit 5 (`$20`) | Sawtooth       |
| Bit 6 (`$40`) | Pulse (Square) |
| Bit 7 (`$80`) | Noise          |

If none of the waveforms are selected then the floating of the last wave-output
value can be observed for a while, then it decays. On a real C64 the duration of
the decay is model and temperature (uptime) dependent.

The oscillator in the SID can be reset at any time and stopped by turning on
bit 3 (value: 8) 'TEST'-bit. It was probably implemented in the SID for factory
testing but it comes handy as a tool in chipmusic. Whether it generates a high
or low steady output depends on the selected waveform. (Contrary to popular
belief, the test-bit doesn't have any effect on the envelope-generator - it
affects the oscillator only.)

If you want really special, jawdropping tones, there are two more weapons to
utilize, one is bit 2 (value:4) 'RING'-modulation, the other is bit 1 (value:2)
channel-'SYNC'.

Ring-modulation affects only the waveforms that contain triangle, but not
sawtooth, and in a nutshell it mirrors/folds the wave's upper half when the
neighboring channel's oscillator is in the 2nd half of its period. This creates
a richer spectrum and very interesting effects, including formant-like sounds,
all without filters.

__TODO__: Needs a diagram to explain

Channel-synchronization, on the other hand, resets the oscillator whenever a
neighboring oscillator enters the 2nd half of its period. This also creates
fascinating waves that resemble the human voice (where formants are synced to
vocal cords).

The controlling channels in both of these scenarios are always the lower
channels. For example, channel 2 is controlled by channel 1. Mostly these two
functions are mastered with experimentation, as it's hard to get a grasp how it
really works and to estimate the results. As with the waveforms, ringmod and
sync can be combined together.

Last, but not least, there is the 'GATE'-bit which does more than one would
think at first sight. It controls the volume-envelope of the generated waveform:
it starts and stops the notes, so to speak.

### Attack/Decay/Sustain/Release (ADSR) Envelope Generator Settings

| Channel   | Attack, Decay     | Sustain, Release     |
| --------- |:-----------------:|:--------------------:|
| Channel 1 | `$D405`           | `$D406`              |
| Channel 2 | `$D40C`           | `$D40D`              |
| Channel 3 | `$D413`           | `$D414`              |

* _Attack_: Bits 4..7
* _Decay_: Bits 0..3
* _Sustain_: Bits 4..7
* _Release_: Bits 0..3

When the GATE-bit is turned on, the 'ADSR' envelope-generator starts an 'attack'
phase, thus it starts a sound. If it's kept active, the volume rises at the rate
of the corresponding ADSR setting until it reaches the maximum level. Then it
falls to the 'sustain' level at the rate of 'decay' setting. Turning off the
gate-bit starts the 'release' phase, which means the note's volume falls towards
zero at the rate of the 'release' setting. Though this is the basic operation of
the GATE-bit, it can be turned on/off during any phase of the ADSR envelope. As
a rule of thumb when it turns on it always starts an attack phase, and initiates
release when it's turned off, though the envelope isn't reset to 0 if it's in
the middle region.

But, unfortunately, like with other things with SID, life is not so simple. As
you will see later in the more thorough explanations, ADSR sometimes does not do
what it's told to. You'll get weaker or even missed notes with certain ADSR
values and GATE-triggering schemes. No, it's not a 'humanize' function
intentionally built into the SID. I think the reason is the resourcefullness
that was a must for people making VLSI chip design in the early 80s. Maybe some
rushed work contributed to it, too, so ADSR rate-counters are never reset in the
SID. It would be logical to reset them when a note gets triggered by the GATE-
bit, but that's not the case. What makes it worse is the fact that the counters
can overlook their rate-settings. We'll explain it later, for now it's
sufficient to know that luckily people came up with a solution a long time ago:
the '_Hard restart_'. The optimal way to reset the ADSR before triggering notes
is still a subject of discussions at CSDb forums. Different music players
implement it in slightly different ways. There are also some lesser known
wraparound issues in the envelope-generator that will be explained in upcoming
parts of this document.

Attack happens on a linear scale. Here's a list of Attack times on PAL C64
machines:

| Attack value | Attack time |
| ------------ |:-----------:|
|  0           |   2ms       |
|  1           |   8ms       |
|  2           |  16ms       |
|  3           |  24ms       |
|  4           |  38ms       |
|  5           |  56ms       |
|  6           |  68ms       |
|  7           |  80ms       |
|  8           | 100ms       |
|  9           | 250ms       |
| 10           | 500ms       |
| 11           | 800ms       |
| 12           | 1s          |
| 13           | 3s          |
| 14           | 5s          |
| 15           | 8s          |


Decay and Release have longer (3 times that of Attack) non-linear curves.

### Filter Cutoff Frequency

|                         | __Low byte (bits 0..2 only)__ | __High byte__ |
| ----------------------- |:-----------------------------:|:-------------:|
| Filter cutoff frequency | `$D415`                       | `$D416`       |


One of the SID's strengths is its analog filter. The process of creating raw
waves with rich spectral content and then filtering out some of the components
is called _substractive sound synthesis_. The single filter is shared between
all the channels but it can be applied to them separately on demand. Once the
filter is set on a channel the cutoff-frequency can be controlled at an 11-bit
resolution. (Low-byte has only the lower 3 bits implemented, the other bits have
no effect.)

On the 6581 variant the curve of the cutoff-control is nonlinear, with cca 200Hz
below a 'threshold' and often the basses sound more muffled compared to the 8580
which has a nearly perfect linear control-curve. (But again our ears hear
differences at low frequencies better than at the high ones so we perceive it as
nonlinear too.)

On the flipside, the 6581 cutoff frequency can go up to the top of the hearable
range, while the 8580 can go down near 0Hz but tops out at ~13kHz. The 6581 has
an interesting distortion at low frequencies which will be explained later.
8580's new filter-design lacks this 'feature' and there's only distortion when
high resonances boost the signal.

### Filter Routing and Resonance

|                              | __Byte__ |
| ---------------------------- |:--------:|
| Filter routing and resonance | `$D417`  |


| __Bit__       | __Control__      |
| ---------     |:----------------:|
| Bit 0 (`$01`) | Channel 1        |
| Bit 1 (`$02`) | Channel 2        |
| Bit 2 (`$04`) | Channel 3        |
| Bit 3 (`$08`) | External input   |
| Bit 4..7      | Filter resonance |

The high-nibble here controls the resonance of the filter, the hump at the
cutoff frequency. The filter sounds more prominent with this setting than a neutral
curve with no emphasis. Just like the cutoff-control, this behaves differently
for the different SID-models: the 6581 doesn't change much up to a point while
the 8580's resonance-control is continuous, although it's non-linear.

Setting high resonance can lead to distortions as the magnified signal's level
approaches the limits presented by the 9V/12V power.

The low nibble has 3 bits dedicated to turn the filter on/off for the separate
channels: bit 2 (value 4):channel 3, bit 1 (2):channel 2, bit 0 (1):channel 1.
The number of selected filtered channels has a small effect on the cutoff and
resonance, but it's not very noticable.

Bit 3 (value: 8) is the switch for the external audio input which can be fed to
the SID and mixed into the output together with the internal channels. Some
people use this bit to decrease the noise coming into the SID from outside by
filtering it out. Originally it might have been added so that the SID could be
used as a wah-effect pedal.

### Main Volume and Filter Band Selector

|                              | __Byte__ |
| ---------------------------- |:--------:|
| Main volume and filter band  | `$D418`  |


| __Bit__       | __Control__      |
| ---------     |:----------------:|
| Bit 0..3      | Main volume      |
| Bit 4 (`$10`) | Low pass         |
| Bit 5 (`$20`) | Band pass        |
| Bit 6 (`$40`) | High pass        |
| Bit 7 (`$80`) | Mute channel 3   |

The low nibble of this register controls the main volume of the SID. There is a
little bit of leakage though, so even when you set it to 0 it passes through a
little amount of sound. However, the more important fact about this nibble is
that it causes a little shift in the output signal. The bigger the volume the
more the offset is. Since the 1980s this artifact was utilized to play digital
samples (or 'digis'). This effect is much less noticable in the refined 8580
circuitry, so this is a classical problem with new C64 machines: digitized
speech is barely audible on them. But this somewhat compensates for the harsh
clicks of the 6581 that appear when the master volume or filter- parameters are
changed.

The high nibble of this register has 3 bits that control what kind of filter to
use: bit 6 (value `$40`): high-pass, bit 5 (`$20`): band-pass, bit 4 (`$10`):
low- pass. These modes can be combined together to form e.g. a notch-filter or a
low- pass filter with brighter sound.

Bit 7 (value: `$80`) has a special function, it can prevent channel 3 from going
to the mixer, though it's still passed to the filter, so this has no effect on a
filtered 3rd channel. The idea was to use channel 3 as an LFO (low-frequency
oscillator) to control parameters without the need of the CPU to do that task.
But in practice we don't want to lose a precious channel when the CPU can create
any control-waveform easily. So let's leave this bit at zero, please.

### Paddle Values (Read-only)

| Paddle                       | __Byte__ |
| ---------------------------- |:--------:|
| Paddle X value (POTX)        | `$D419`  |
| Paddle Y value (POTY)        | `$D41A`  |

The SID chip also took on the responsibility for reading the analog resistance
values on the C64's inputs. This was used mostly for paddles to control games
but a mouse can be connected to these inputs as well, or any potentiometer with
around 500 kOhm maximal resistance to utilize the full range of 0..255 values.
Voltage can't be applied to these inputs to digitize sound, etc. It works by
charging and discharging a capacitor through the connected resistance and it
determines the resistance periodically by how much time it took to charge the
capacitor. Unfortunately this measurement has a jittering even with steady
input. Software-based filtering can help to smooth this out. (A 'moving average'
filter works just fine.)

### Oscillator and Envelope of Channel 3 (Read-only)

|                              | __Byte__ |
| ---------------------------- |:--------:|
| Oscillator channel 3 (OSC3)  | `$D41B`  |
| Envelope channel 3 (ENV3)    | `$D41C`  |


These are 8-bit readable registers that represent the waveform selector and
envelope generator outputs of the 3rd channel. In combination with the channel 3
disabling mentioned before these can be used as an LFO in rare cases. But for us
a more useful feature of these registers is to determine which model of SID is
present in the machine by checking for waveform differences. For example,
Hermit used these registers many times to display an oscilloscope for
the 3rd channel or to control graphic effects by the music. Use your imagination
what else these could be used for.

## How Does the SID Produce Sound? (SID-internals)

### Phase-accumulators (Oscillators, Pitch)

First of all, let's start with the three oscillators. Without oscillation a
sound could never be heard through the air, as you might know. In the SID this
is done by the 'phase-accumulators'. A phase-accumulator is basically a 24-bit
counter which can be incremented not only by a single step each clock, but also
by any number of steps between 0 and 65535. The 16-bit value which we add
at each clock pulse directly determines the frequency of the oscillation. How?
The phase accumulator wraps around when it reaches its maximal value, and starts
over to count up again. This represents a sawtooth-like waveform. We build upon
this basic concept in the next stages of the sound-generation chain.

The master clock-frequency and thus, also the SID's clock is running at 985248Hz
in the PAL version of the C64. If the frequency value is 0 in the frequency
registers, we don't add to the phase accumulator, and the oscillation is
stopped. Adding 1 gives the lowest hearable frequency we can produce. With the
24-bit phase-accumulator it takes 2 to the 24th power of clock pulses to fully
count up, so at the C64 clock frequency this happens 17 times per second. So,
17Hz is the lowest sound we can make. Adding 65535, the maximal value needs 256
clock steps to reach the top, so the highest the pitch can be is 3849 Hz.

__TODO__: This would prob need a diagram.

Beside setting the pitch we have some more control over the phase-accumulators,
they can be zeroed (reset) by:
- setting the TEST-bit (mentioned above) to 1 on the corresponding channel, or
- when the SYNC-bit is 1 on a channel, the phase-accumulator on that channel is
  zeroed at the moment the other (source) channel's MSB (bit 23) rises to 1.
  (Again, Sync source-to-destination channel-pairs are:  1->2 , 2->3 , 3->1 )

### Waveform-generators (Unfiltered Waveforms/Timbres)

As mentioned before we have 4 basic waveforms to choose from on each cannel.
They are created in different ways in 12-bit resolution.

Sawtooth is the simplest one, it's simply the upper 12 bits of the phase-
accumulator.

```
      /|  /|  /|  /|
     / | / | / | / |
    /  |/  |/  |/  |
```

Pulse/square-waveform is derived by comparing the pulsewidth/duty-cycle
registers (value 0..4095) to the current top 12 bits of the phase-accumulator,
and connecting all output-bits to 1 (Vcc) when it's greater, and to 0 (GND)
when it's smaller. A pulsewidth of 2047 results in a square waveform.

```
    +--+   +--+   +--+   +--+
    |  |   |  |   |  |   |  |
    |  |   |  |   |  |   |  |
   -+  +---+  +---+  +---+  +--
```

Triangle waveform is made from the phase-accumulator (sawtooth) by XOR-ing all
of its 11 upper bits with its MSB (bit 23). This causes the 2nd half of the
sawtooth-wave to fold back giving us the triangle waveform. But as this has a
halved amplitude, the 12-bit wave-output must be generated from the left-shifted
form of this to ensure that the output has the same amplitude as that of a
sawtooth wave. The lowest bit is always 0.

```
      /\    /\    /\    /\
     /  \  /  \  /  \  /  \
    /    \/    \/    \/    \
```

The ring-modulation for the triangle wave is achieved by enhancing the above-mentioned
MSB XOR-ing with an extra XOR with the inverted MSB of the source (modulation)
channel's phase-accumulator if the RING-bit is set. As a result, the triangle
is inverted/flipped when the two MSBs are equal.
(again, ring source-to-destination channel-pairs are: 1->2 , 2->3 , 3->1 ).

Noise waveform has its own 'counter' in the form of a pseudo-random sequence
generator. It's realized by a 23-bit (actually 24 on die but the MSB is unused)
LFSR (Linear Feedback Shift-Register), which is a shift-register that when clocked,
simply shifts its 0/1 contents to the 'left'. What makes it an LFSR is the feedback
mechanism that generates the signal to be fed back to its rightmost bit (LSB).
There are so-called 'taps' on carefully selected places, bit 22 and 17 of the LFSR,
that are XOR-ed and that value is fed back to the LSB.

```
                     reset  +--------------------------------------------+
                       |    |                                            |
                test--OR-->XOR<--+                                       |
                       |         |                                       |
                     3 2 2 2 1 1 1 1 1 1 1 1 1 1                         |
      Register bits: 2 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 <---+
                           |   |       |     |   |       |     |   |
      Waveform bits:       1   1       9     8   7       6     5   4
                           1   0
```

This generates a very long sequence of pseudo-random values before it repeats.
The LFSR is clocked by the rising edge of bit 20 of the phase-accumulator so the
noise spectrum - the 'sense of pitch' - can be controlled. With the TEST-bit
enabled the individual bit inputs become floating and gradually lose charge,
the inverter connected to it will become high after some time so the register
can be filled with 1s over a variable number of cycles' period, depending on the
chip revison and temperature, reaching the value of `$7FFFFF` which is the initial
value of it at startup (will actually become `$7FFFFE` when the TEST-bit or the RESET
signal is released).

The 23-bit LFSR value still has some linearity/predictability between the
adjacent bits so we take the noise-output from a so-called 'scrambler' instead.
In case of the SID the scrambling is simply done by using 8 different bits of
the LFSR to constitute to the wave-output (bit 20,18,14,11,9,5,2,0) which only
has 8 bit resolution, but it's sufficient for noise. The 4 low-bits are not used
for noise and fixed at 0.

### Waveform Routing

Now that we have generated the 4 basic waveforms (still digital, 12-bit wide)
we are ready for routing. Inside the SID there are pass-transistors (FETs) on
all 12 bits of the waveform outputs acting as series-switches to select which of
the 4 waveform-ouptputs we want to route to the bit-drivers of a channel's
output.

```
              Vx
               ^
               |
           +---+---+
           |   |   |
       Saw \   |   |
           |   |   |          Tri
 Tx+1 <----+   |   +------+----\---< Tx
           |   |          |
           |   |          |
           |   \ Pul      \ Noi
           |   |          |
           ^   ^          ^
          Sx   Px         Nx

    Tri/Saw/Pul/Noi: waveform selectors
    Tx/Sx/Px/Nx: waveform bit x
    Vx: voice output bit x
```

Ideally only one of them would be allowed to be turned on at a time, but there's
no multiplexing logic to ensure that. This brings us further possibilities. Bob
Yannes himself discouraged the usage of combining the waveforms by turning on
multiple outputs, probably because connecting active output-drivers together is
never a good idea in electronics - they will fight against each other when one
wants to drive a low signal while the other wants to drive a high signal.
However, in chips made with NMOS and HMOS chip-fabrication technologies the
driving strengths of the outputs are less for high logic signals than for low
ones due to the 'upper' MOSFETs used as 'active'/'dynamic' resistors. (CMOS can
drive both logic values equally strong, so that would be more prone to high
currents and failures if the SID was ever recreated with this more up-to-date
manufacturing technology.)

In practice there have been no reports that SIDs broke due to the usage of
combined waveforms but who can tell? These chips can become hot enough with
just standard usage...

### Combined Waveforms (A More Thorough Explanation)

During the development of jsSID Hermit did research on how the complex combined
waveforms are generated. (He could generate them with functions, see the jsSID
source code for more details and for ASCII schematics.)

If you look at them closely you probably notice that they look like fractals,
small portions of them resembling their overall shape. It's logical to deduce
from this that there is some recursive bit-wise manipulation responsible for
that.

Checking on the great reverse-engineering results of decapped SIDs at the
Kevtris webpage (__TODO__: Link needed) revealed the circuit of the above-
mentioned waveform-selector logic. There are simple amplifiers for all of the
selected (routed) waveform-bits before the DAC (Digital-to-Analog) stage. The
waveform-combining happens *before* these bit-amplifiers and the DAC, so the
combining is not made on the analog outputs, but bit-by-bit in the previous
digital stage.

To understand how the combined waveforms are generated, the analog behaviours
of this digital circuit-region needs to be discussed. The 3 crucial analog
contributors are the above mentioned weak driving of high-bits, the resistance
of the chip-fabric and the treshold-level of the amplifiers before the DACs.

For example, take the simplest case: when you connect sawtooth and pulse
waveforms together, you connect all their bits together through a weaker
connection, because that is what a square waveform does, as explained above: it
connects all 12 bits either to GND or to power-rail.

If the square/pulse output is 0, driving it to GND is so strong that no matter
if the sawtooth-bit wants to drive 1, it will be below the treshold of the bit-
amplifier so the DAC gets 0 to output. But when the pulse-output is 1 the bits
are driven by it to high only 'weakly'. In that case a 0 sawtooth-bit can bring
the combined bit-value low enough to ensure 0 at the output of the bit-
amplifier and the DAC. This sounds like an AND operation between the pulse and
sawtooth waveform-bits, and we would get a sawtooth if pulsewidth is 100%. But
it's not that simple: as the pulse-waveform connects all bits together through a
given resistance (depending on the chip-technology) it's possible for
neighboring bits of the sawtooth to affect each other. The closer a bit to the
other bit is, the more it pulls it down towards 0 or up towards 1, eventually
agreeing on a level that is above or below the bit-output's treshold. This is
the recursive process responsible for the fractal-like look. (In code Hermit
made this by creating two nested 'for' loops where each bit has a value affected
by all the others. With proper parameters the waveforms were very close to the
original.)

It's worth noting that if you look at a pulse+sawtooth with 100% duty-cycle, the
combined waveform samples don't go above the corresponding sawtooth values, only
below. This means that the FETs driving high are really much weaker than the
ones driving low. Especially on the old 6581 SID where most combined waveforms
are weak (contain many 0s) and have the MSB suppressed when sawtooth is selected
(as a result their amplitudes are halved but their frequencies are doubled).

If you add triangle to the 'mix' it gets even more complex: it connects adjacent
bits (for the left-shifting mentioned at waveform-generators) and provides even
more connections to 0 level, so combined waveforms containing a triangle have
more low or zero values when you look at them.

Noise can be combined with other waveforms but the discussed zeroing effect is
able to clear the bits in the LFSR gradually and when the LFSR is filled with 0s
it 'locks up' and can only be restarted by test-bits. (Interestingly, in a VICE
emulator it's possible to set a pulsewidth very close to 100% and combine that
pulse with a noise (`$C1` waveform) without locking it up. Never tried this on
real SID, by the way.)

### Envelope Generator (ADSR a.k.a. Channel Volume)

The ADSR envelope generator affects the analog part of the SID around the DAC.
Each channel has one, and its output is an 8-bit value (0..255) controlling the
VCA (Voltage-Controlled Amplifier) that determines the volume of the
corresponding channel.

Based on the ADSR parameters given in the register and the GATE bit in the
waveform control register, three internal counters (per channel) are operated:

- The 8-bit 'Envelope-counter' which is fed to the DAC controlling the
  channel-VCA.

- The 15-bit 'Rate-counter' is a prescaler to set the steepness (speed) of the
  envelope-counter in Attack/Decay/Release phases. Tthese are the prescale
  values (periods) for the Attack/Decay/Release values of 0..F:

  | Nibble value | Periods |
  | ------------:| ------- |
  |            0 | 9       |
  |            1 | 32      |
  |            2 | 63      |
  |            3 | 95      |
  |            4 | 149     |
  |            5 | 220     |
  |            6 | 267     |
  |            7 | 313     |
  |            8 | 392     |
  |            9 | 977     |
  |            A | 1954    |
  |            B | 3126    |
  |            C | 3907    |
  |            D | 11720   |
  |            E | 19532   |
  |            F | 31251   |

  (Note that a value of 0 still has a period, and therefore there's nothing like
  'zero-time' Attack/Decay. That's why sounds with AD=00 have clicky starts.)

- The 'Exponent-counter' is a further prescaler for the envelope-counter in
  Decay and Release phases to ensure more ear-friendly nonlinear sound decays.
  There's a space-efficient exponential table in the SID with prescale-values
  paired to envelope-counter value ranges/stages:

  | Envelope counter value range | Prescale value |
  | ----------------------------:| -------------- |
  |                      255..95 | 1x             |
  |                       93..55 | 2x             |
  |                       54..27 | 4x             |
  |                       26..15 | 8x             |
  |                        14..7 | 16x            |
  |                         6..1 | 30x            |
  |                            0 | 1x             |

  (At the start of a fast 1x prescaling/division, it gradually grows to a 30x
  slow decay when the envelope-counter falls below 6.)

Several state-bits determine the current ADSR state/phase:
- Attack-phase
- Decay+Sustain phase
- Hold-at-zero state

A transition of GATE-bit to 1 turns on Attack-phase and prepares for the next
Decay+Sustain-phase, and disables any Hold-at-zero state.

Attack-phase lasts until the envelope-counter counts up to `$FF` then it's
turned off and the Decay+Sustain-phase dominates in which the envelope-counter
through the exponent-counter prescaling counts down until it reaches the
sustain-value. (Which is expanded to 0..$FF by doubling the 0..F sustain-value
to the high-nibble.)

A transition of GATE-bit to 0 turns off any Attack or Decay+Sustain-phase and
counts down through the exp-prescaler until it reaches 0 (Release-phase) and
enters 'Hold-at-zero' state and only leaves this state when GATE goes to 1.

#### ADSR delay-bug

For hardware-efficiency in the SID most counters are made from LFSRs that need
less parts and it doesn't matter if they don't count linearly, the comparison
values are simply selected according to their predetermined (pseudo-random)
sequence.

But that turned out to be a problem with rate-counters that determine the speed
of the envelope-counter. The rate-counters are only reset when they count up to
their current comparison value which is based on a prescale-table looked up by
the 0..F Attack/Decay/Release value, depending on the current ADSR phase.

As there's a single rate-counter for all the 3 timed ADSR-phases and starting a
new phase doesn't reset it, it's possible for the rate-counter to miss a
prescale-value (rate-period) that it already went through: when for example a
faster (lower-period) Attack follows a slower (greater-period) Release. In that
case a match is not found until the rate-counter goes up through its full
sequence and starts over by wrapping around. That can take as long as 32.8ms
(counting at 1MHz, cycletime is 1 microsecond, 32768 * 1 microsecond = 32.8ms).
The next note can delay that long, and it's quite audible. This is the so-called
ADSR delay-bug.

The ADSR delay-bug appears statistically rarely when the difference between the
rate-periods of adjacent phases is small (doesn't decrease much), but is more
frequent when the next rate-period is much smaller than the previous. (It's
less-known/less-noticed but it is logical: this delay-bug can happen after a
transition from an Sustain to a Release phase, too.)

We'll see later how to overcome (or how to enforce) this delay-bug situation.
But for now these are all the important details about how the ADSR basically
works.

### Filter (Shaped Timbre, Substractive Synthesis)

After the channel-DACs there are 2 routes for the now analog waveforms to take:
either the direct lines to the main output mixer or through the filter circuitry.
In the register-section I described the exact addresses and bits of the filter-
controls which determine this route. The channels going into the filter are
summed through resistors.

```
                +---------------------------------------------------+
                |                                                   |
                |             +---Rf--+                             |
                |             |       |                             |
                |   +---------o--<A]--o------Rr------+              |
                |   |                                |              |
                |   |                                |              |
  $17           |   |                    (CAP2B)     |  (CAP1B)     |
  0=to mixer    |   +--R---+  +---Rf--+      +---C---o      +---C---o
  1=to filter   |          |  |       |      |       |      |       |
                +------R---o--o--[A>--o--Rw--o--[A>--o--Rw--o--[A>--o
      Ve (EXT IN)          |          |              |              |
  D3  \ ---------------R---o          |              | (CAP2A)      | (CAP1A)
      |   V3               |          | Vhp          | Vbp          | Vlp
  D2  |   \ -----------R---o    +-----+              |              |
      |   |   V2           |    |                    |              |
  D1  |   |   \ -------R---o    |   +----------------+              |
      |   |   |   V1       |    |   |                               |
  D0  |   |   |   \ ---R---+    |   |   +---------------------------+
      |   |   |   |             |   |   |
      R   R   R   R             R   R   R
      |   |   |   | $18         |   |   |  $18
      |    \  |   | D7: 1=open   \   \   \ D6 - D4: 0=open
      |   |   |   |             |   |   |
      +---o---o---o-------------o---o---+
                  |
                  V
             Mixer output

  V1  - voice 1
  V2  - voice 2
  V3  - voice 3
  Ve  - ext in
  Vhp - highpass output
  Vbp - bandpass output
  Vlp - lowpass output
  [A> - inverting op-amp
  R   - "resistors", implemented with custom FETs
  C   - capacitor
```

This circuit in the SID is a '2-integrator loop bi-quadratic' filter and as its
name suggests, it contains two integrators in a loop through a 3rd member, a
simple amplifier for resonance/emphasis. The integrators are basically inverting
operational amplifiers with capacitive feedback. These capacitors are both
outside the SID-chip on the motherboard and are of fixed values. What makes it
possible to change the filter cutoff-frequency is that the VCR (Voltage-
Controlled Resistance) is in series to the inputs of these integrators, serving
as the variable resistors (both controlled in tandem) in the RC
filter/integrator.

The 8580 and the old 6581 SIDs differ very much in their implementation of the
VCR: the 6581 uses 2 single FETs as VCRs with some attempts for linearity by
having negative feedback resistances to their gates and operating in their near-
zero signal-region. There are resistor-ladder DACs that convert the filter-
frequency set in the registers to an analog signal, which also control the gates
of the VCR MOSFETs. But this simple control of a series (common-source) FET has
the disadvantage of generating feedback from the signal-path: the FET is a
transconductance device that transforms voltage between its Gate and Source
terminals. But the source is not at GND so the Gate-Source voltage depends not
only on the cutoff control signal but also slightly on the integrator's small
input-signal. That causes a kind of distortion unique to the 6581 old SID,
because thanks to the 'resistance-modulation' of the VCR the cutoff-control
signal gets a bit of the audio-signal, so the audio signal essentially alters
its own filter-cutoff frequency, and even during a single wave, the waveform
becomes less rounded. The unique fat sound produced by this is actually
preferred by many people.

The control-curve of the 6581 SID is also nonlinear, because there are about
1.5GOhm 'shunt' resistors between the Drain and Source terminals of the VCR-
FETs. So when their resistances go above this value, less and less change is
seen in the cutoff-frequency. On average this 1.5GOhm resistance ensures a
minimum cutoff-frequency of about 200Hz. There is big spread among 6581 SIDs, so
these resistance-values and the cutoff-frequencies vary wildly from chip-to-
chip. (These are the so-called 'dark' and 'light' SID cutoff-curves...)

The 8580 SID's redesign affected its filter a lot: as seen in the die photos at
kevtris.org ( __TODO__: Link needed ), the single-FET VCRs were replaced by a
different method. Essentially, the filter-cutoff control-voltage DACs seem to be
integrated with the VCRs into a digitally controlled resistor-ladder VCR.
Anyhow, this results in a very precise (I'd say laboratory quality) and linear
filter-cutoff control, and filter-distortion seems to be gone, too.

(Back at the time Robert Moog could only make sophisticated analog VCFs from
bipolar transistors connected as differential amplifiers in a ladder layout,
having capacitors in the rungs, which was the so-called 'Moog-filter'. MOSFETs
in SID made this easier.)

### Mixing and Output (Main Volume)

The output stage is simply mixing the non-filtered audio route and the filter's
output and applies the main volume on them through a VCA, then the SID's sound
is ready for amplification which goes to the outside world in ways described
earlier. (Some people say the simple transistor-based output-amplifier in the
C64 might add some characteristic nonlinearity to its sound... maybe.)

---

## Usage of SID in Practice, Tips, Tricks and Secrets

Knowing the internals of SID is not enough for squeezing good music out of it.
We still need an interface from the SID to the Human, namely: the composer.

The composer needs a tool to compose and execute his/her ideas. There are many
so-called trackers around and most of them add a lot of extras by software to
the SID to bring it closer to ready-made synthesizers: frequency-tables,
vibratos, LFOs, etc.

### Hard-Restart (Preventing the Delay-bug)

One of the most important features of such tools is to eliminate the ADSR delay-
bug so the composer can rely on sound-starts. The workaround to the bug is
called '_Hard-Restart_'. (Hermit has another article about it in the FlexSID's user
manual, see Appendix.)

The method 'resets' the ADSR rate-counters, so when a new sound (Attack) happens
we know the rate-counter is not just anywhere, but it's counting inside the
first 9 steps rapidly. To achieve this we simply need to set both AD and SR
registers to 00 and the GATE to 0 for at least 2 PAL-frames (40ms) before a new
sound is about to start. This also ensures that the Release-phase arrives at a 0
level.

At least this is the method that always works, no matter what ADSR settings the
previous and the next instruments have. Different programs use different
approaches, many times only the SR is zeroed, which is OK too. In programs where
the Hard-Restart-ADSR can be set to any value, people tend to use in-between
values in the pursuit of softer sound-starts, but most methods seem casually
suitable for a given piece.

There's a 'new kind of hard-restart' mentioned at CodeBase64 (by Shrydar, and
Lft is involved here too), they call it 'Bottle', and it's a totally different
cycle-exact code approach. It is able to reset the rate-counter in the timeframe
of about 10 rasterlines (less than 1ms) instead of a 20ms frame, by utilizing
the delaybug-free safe transition from a slow attack to a fast decay. There's
another ADSR-bug in the SID: the envelope-counter can also wrap around when it
is at value `$FF` and an Attack is triggered. This is used to bring the envelope
back to `$00` fast with this 'Bottle' approach.

Only time will tell how soon this restart-method gets implemented in players...

### To Delay-bug Or Not to Delay-bug? (Sexy-start)

What differs in most editors and their players/drivers is how the new sound
starts after the Hard-Restart. Different recipes work differently. To avoid a
new delay-bug after Hard-Restart the best write-order - as per Hermit - is to
set AD first before turning on GATE so it won't affect the rate-counter (thanks
to it being in release-phase after Hard-Restart), then set the GATE-bit to 1 to
change to Attack phase, and then immediately set the SR-register which will only
have an effect when the next GATE-bit turnoff happens (note ends).

But most player-routines do quite the opposite instead (causing the delay-bug)
to achieve the so-called 'sexy-start' of sounds: they write AD and SR before
turning the GATE-bit on and if the Release-value is big enough (above 3..4), a
new sound with a small Attack value of 0..1 is started, during which time the
rate-counter had a chance to advance and cause a miss of the Attack compare-
value. The more one waits between AD+SR and GATE writing the more probable this
situation becomes. The result is having a delay in the start of sound of about
32.8ms, so the 1st frame of the sound is silent (usually waveform `$09` is set
here to set TEST-bit), but the 2nd frame's last 7..8 milliseconds are audible.
And that is what musicians need, a very short waveform (the first row of a
waveform- sequencer table in the instrument-editor) to make the start of a sound
percussive without turning to multi-speed tunes. Usually high-pitched white-
noise is placed here. Our ears are much more sensitive to changes and a special
start of the sound fulfills that scenario.

It's not important to use Hard-Restart for fairly stable 'sexy-start' notes, but
the release of a previous note should be much bigger than the attack of the new
note, and it's advised to decay to 0 level before the new note starts. It's
easier with drums that are decayed fast from within the waveform-tables. Many
composers in the past didn't use Hard-Restart at all, but knew these rules and
selected ADSR values of adjacent sounds carefully.

To allow more diverse instruments but still retain a much bigger release before
an attack we can set the SR register to `$0F` artificially for 1 frame before
the sound-start. This is a simpler kind of 'hard-restart' and this doesn't zero
the rate-counter, but it does the opposite: it allows enough time (20ms) for it
to count into a region which will almost surely be above the compare-value of
the next Attack and cause a delay-bug, and thanks to that, a shortened 1st frame
of the next sound, aka 'sexy-start'.

### Digital samples on the SID

SID was not designed to play back digital samples, so digis were not the
strongest side of the SID in the past. The volume-register setting trick causing
an offset is described in the section about SID registers above, but that could
only produce 4-bit resolution sound and it affected the main volume of the SID.
So if normal SID-music was played alongside it, it sounded distorted/modulated.
(Not to mention the big difference between the volume of the digital samples on
the two SID variants.) Some emulations (e.g PlaySID on the Amiga) could separate
the AC-component of the main volume and send it to a separate digi-channel, and
the DC-component could still be used as main volume, and was even freed from
audible pops when the volume was changed slowly for fade-in.)

The next 'easiest' method to play digital samples on a C64 is Mahoney's method:
He measured the offsets caused by all the bits (lowpass/highpass/volume/etc.) in
the SID with different settings of 100% duty-cycle pulses and ordered them in a
256-byte table for both the 8580 and 6581 SIDs. There are about 30..50 different
quantization-levels that can be reached with the proper settings and tables
which is better than the 16 levels of simple digis. The downside is that no
normal SID-channels can be used beside the digi, but it's still as simple as
writing `$D418` at a given sampling frequency.

The other methods that can achieve 8-bit digi-resolution on a SID need precisely
timed (cycle-exact) code. Some methods (e.g. those used in the _Wonderland_
demo-series) used the pulse width control of the SID to create an up to ~15kHz
PWM signal by resetting the phase-accumulator at this rate. Because of the
audible carrier frequency this was not so appealing for music (maybe a lowpass-
filter is a solution for the carrier noise), but a step forward, nevertheless.

The ultimate solution at the moment is _SounDemon_'s digi-routine which utilizes
the floating signal on a channel when no waveform is selected (waveform `$01`).
The task of this method is similar to the PWM-method: to periodically reset the
phase-accumulator at the given sample rate with the TEST-bit, and set the
oscillator frequency proportionally to the desired sample level. The ADSR must
be kept at sustain-level, so the waveform is kept at value `$01` most of the
time. Then after a given amount of time (that should fit in the sample-period)
the waveform is set to `$11` (triangle) for a short moment to update the
floating value at the waveform-selector output. The next round comes so fast
that the floating value doesn't decay significantly. This is like a sample-and-
hold circuitry. The upward-slope of the triangle waveform instead of a sawtooth
ensures higher range (resolution) in less time.

The only disadvantage of the SounDemon digi is the strict timing and the high
CPU resource it needs, but it sounds good, and the other SID-channels can still
be used along with the digi just fine.


## Sound Design & Composition Tips

Here I collect some of my findings about good SID sounds. In the past I coded a
tool called 'SIDhack' to separate the SID-channels and debug them in realtime to
determine what different SID tunes do to sound so good. That tool came handy and
I made some tunes containing ripoffs from other SIDs as case studies, like the
tune 'sidhack' in the SID-Wizard package. The CSDb forum can also be a good
source for sound-design tips & tricks where people share their experiences with
each other.

In general, our brain, our neurons are more sensitive to changes rathe than
steady signals. This might be an evolutionary solution to long-term stimuli and
to focus better on new events. We have 'differentators' built-in. For example,
the sense of smells and colours and even touch degrades over a short time: we
get used to smells, we see the opposite of a color when it's removed, placing
our hand on a raw surface feels it but the feeling fades. Our eyes make micro-
movements just to keep the nerve-signals frequently updated, stopping them would
lead to slow loss of vision.

The ears work similarly and our whole appreciation of music depends on it: we
like changing sounds better than the steady ones, therefore a pulsewidth-
modulation instead of a fixed duty cycle can make a big difference in a lead
instrument. This is the same for the filter: a filter sweep on a bass sound or
performing some keyboard- tracking (opening the filter as the pitch increases)
is more desirable than a muffled, steady cutoff frequency.

So one ingredient of good SID sound is pulse/filter-sweep. Even better if the
program supports turning off the filter/pulsewidth-program reset when the new
sound starts, which provides even more variations.

Speaking about variations, rhythmic variations, syncopation and, of course,
melodic variations, and even variations at a higher level (in the structure,
arrangement) are desired in music, but that's a whole other topic in its own
right, so I'll stay with the sound design for now.

Sawtooth and triangle waveforms are invariable compared to pulse, but sometimes
they providea a better character for an instrument. I especially like when
arpeggios are made with triangle, they provide a feeling of ambience.

But I guess pulse/square is the most used because its spectral content can vary
much. The thin pulses are similar to sawtooth waves, the 50% pulses are glassy
and Nintendo-like, but for a bit more harmonic content I usually like to set the
pulse width somewhere near 50% instead. Slow pulse-sweep is the key for
beautiful lead sounds, but a very fast pulse-sweep is the key for some 'chorus'
effect. (Probably the reason for the 'chorus'/'room' effect is two-fold: the
change of harmonic content might cause a bit of percieved detuning and my other
assumption is that when a pulse width is changed it doesn't happen in sync with
the phase-accumulator and as a result there are many partial pulses creating
different frequencies temporarily.)

And last but not least, the pitch: our ears are very sensitive to even small
changes in sound, but to me it seems we're most sensitive to pitch (or
frequency) in music. A very minimal detuning can cause unpleasant sounds in
melodies. But detuning can be our friend, too, to create good instruments. Sure,
detuning a channel compared to an other can achieve a choir effect like with the
accordion, doing it with vibrato can make the music even more lively. Vibratos
are ment to be used with a little bit of delay even on live instruments because
our ear needs a stable note-start before letting the rest of it to vibrate.
Vibratos are good tools to emphasize notes, just like dynamics. Some chorus
effect comes very handy for basses, too, to make them appear stronger under the
lowpass-filter.

For high-pass filters I usually don't prefer to use big resonances, but they're
nearly essential on the SID where the resonance is not too strong compared to
analog syths and VSTs. To make a bass sound somewhat richer I usually set
both low-pass and bandbass filters. For special sounds like claps or speech,
the bandbass filter alone seems very good and should be used more frequently
in SID-tunes.

Sometimes it's good to enhance a lead instrument by dedicating the sole filter
to it. Basses in a mix can stay unfiltered and still sound good because they
have the deepness, but their harmonic contents still keep the richness of the
music. Sometimes with jazzy and soft tunes the triangle waveform is good to
act as a doublebass-like sound or even in techno tunes, and the filter can be
used to make the leads sound modern, sound more like an expensive synth.

### Sync/ringmod?

These have always been mysterious despite knowing how they work. Most of the
time some good sounds could be made by tweaking. In general, sync-effect seems
to be more deterministic, but ringmod gives frequencies that are hard to follow.
Using both is even more interesting and uncontrollable, but playing around can
lead to good results...

Echo-simulations on a single channel in a melody are quite possible by inserting
softer notes between the normal notes. I don't know what others think, but I
usually feel the notes that best fit there are repeated notes and not
necessarily some notes appearing earlier on that channel or notes in the melody.

Good bass drums can be made without the filter in the waveform-table, sometimes
even with triangles. But the strongest bass drums contain ~50% square waves in
a sudden frequency-drop at the first 1..2 frames, then only several notes
of drops in the last frames.

Good snare sounds can be made by only 1 but maximum 2 frames of ~50% square,
then the decaying white-noise sould continue asap. I like snares in funky
music which end abruptly but it's not obvious how to make them on the SID.
Usually release values of 5..7 are fine for snare. I like the snares of Shogoon
which are made on 2 channels and you can really hear the oomph and the
snare noise at the same time. But that's not always possible.

Arpeggios are tricky beasts. They can sound ugly when the pitch changes every
frame, I usually let pitches last at least 2..3 frames. To reduce the abrupt
pitch-changes even further arpeggios can return to the base note. Chord-
inversions can also make arpeggios even more listenable. If done well, complex
harmonies with dissonant intervals sometimes are more listenable as arpeggios
than when they're played together. After all an arpeggio is a fast melody, so
dissonances disappear fast, maybe that's why.

Dynamics in music are usually desired, there's more variety in a hihat or kick
or snare when there are strong and weak hits in good places. But interestingly
for some oldschool C64 tunes the fast repetitions without any dynamics sound
better, maybe they emphasize that this is a different style of music and not
something played by a human who automatically adds dynamics by the laws of
physics.

All in all, good ears and being open-minded for new possibilities are probably
the best leads for creating interesting SID music. And the importance of
composing should never be underestimated, a good composition with simple
instruments is often more joy to listen than music with good sounds but
without an idea or a story.

# Authors

## Original Document

v0.1 by Hermit (Mihály Horváth), 2022

## Other Contributors

- LaLa (Imre Olajos) - proofreading, reformatting
- Leandro Nini - minor additions

---

# Appendix

## Some in-depth info about Hard-Restart and ADSR-delaybug

_Extracted from FlexSID[^3] docs, by Hermit_

 It's not essential to have hard-restart in your arsenal, great SID-musicians in
the past were aware about the SID-delaybug and selected ADSR values carefully
to avoid it, or cause it if that was what they needed...
 Typical players perform Hard-Restart automatically. This is not the case with
FlexSID, but at least the specialized C6 command makes it possible in less space
than it could be done with ordinary InsFX-table commands. Its usage has been
told above, but how Hard-Restart works in general is a mystery to many people.
 So here I take the chance to share what I know about it, with the experience I
had by coding players and SID-emulation engines several times (FlexSID contains
my cSID engine, you can find the source-code in file 'SIDemulation.c'.)

 To understand this, one needs to know some internal workings of the SID's ADSR:
The ADSR delay-bug which makes SID soundstarts unreliable sometimes is caused by
a lacking/simplified implementation of the so-called 'rate-counters'. These are
affected by a lookup-table and the values written into SID ADSR registers, and
they determine the Attack/Decay/Release speeds/rates of the ADSR-envelope curve.
How? Rate-counter counts at 1MHz and when it reaches the looked-up value it is
reset to 0. This  is done periodically, and the envelope-counter (essentially
the ADSR curve) 8bit register can increase/decrease at each period to eventually
reach the target value (which is 255 for attack, 0 for release, and the Sustain
value for Decay). Decay and Release sometimes skip these steps to ensure non-
linear fadeout which is more natural to the ears, but it's not important here.

 There are 2 main problems: First there is only 1 rate-counter (per channel) in
the SID, so it's shared between these 3 ADSR phases. This wouldn't be much of a
problem, but the rate-counter compare-value is only tested for equality. That
can cause the bug, because the compare-value depends on the phase/state of the
ADSR and the rate-counter is not reset when the ADSR advances from one phase to
another, only when it is equal to the compare-value. The reason behind this must
be the fact that rate-counters are not actual binary counters but simpler LFSRs,
in other words, pseudo-random generators. They go through all possible values
just like counters, but not in a linear fashion. And that makes only equality
comparison possible in a simple circuitry (pobably by XOR-ing). Linear counting
and magnitude comparison is out of the question, probably due to chip-area
constraints at the time of SID's development. Let's see through an example in
slow-motion why/how this can be a problem and cause delay-bug:

 Let's say we have an Attack set to 4 in SID by the C64, and we turn on the
Gate-bit in Waveform-control register. The ADSR then goes to Attack phase and
the rate-counter, at whatever value it is currently, is now compared to a new
value periodically, which corresponds to its 150th step, whatever it is for the
LFSR. Let's pretend from now on the rate-counter is linear. So it counts, and
when it reaches 149 (assigned to Attack value 4 in the table), it resets back
to 0 which allows one step up on the Attack curve. Normally, at 1MHz clock, the
rate-counter period is 150 microseconds, and if everything goes fine, the Attack
gradually steps up to the envelope top-value 255 in 150us*255 = 38ms to give
place for the next 'Decay' phase. But what if rate-counter was not between the
0..149 values before the very first Attack step? If it was set bigger by a
bigger Release previously, it doesn't get reset until it reaches the maximum
value 32767 (being a 15bit counter) where it wraps around back to 0. But
32768*1us=32.8ms has elapsed meanwhile, without any increase in the envelope
value. Our Attack phase was delayed by this amount of time, we're facing an
audible delay bug in this case.
 This problem won't happen in the Attack-to-Decay transition if Attack-rate is
bigger than Decay-rate, because the transition between these 2 phases is
strictly determined by the rate-counter, and is synchronized by it.
However, there is a second place too in the ADSR curve where this delay-bug can
happen, the transition from Sustain-phase to Release-phase, caused by a gate-bit
turn-off at any time, no matter where the rate-counter is in counting. It's not
as audible usually as the Attack-bug described first. This happens when the
Decay rate was set bigger than the Release, so the rate-counter could possibly
have passed through the new Release-compare-value to wrap around again.

 Now we know the problem, and we have a solution for it called 'Hard-Restart'.
To ensure the rate-counter being below the rate-compare-value of the new note's
Attack, we reset the previous note's rate-counter to 0. As there's no direct way
to do it, first we need to know whether we're in Decay or Release phase, and set
its rate to 0. Usually it's done by turning off the gate-bit, so the phase is
known to become 'Release', then setting only Release to 0. But if it's not sure
that the previous note was turned off, setting Decay register-value (rateperiod)
to 0 at the same time can be beneficial. Because we never know where the rate-
counter is in the counting at any moment, we can only be sure that it reaches
zero after resetting ADSR to 0, if we wait at least the above mentioned 32.8ms.
 So if the wrap-around delay-bug happened, we give enough time for the
rate-counter to 'settle down'. As most music routines work at 50Hz PAL rate,
this takes 2 screen-refresh/vsync frames. After this we have a fresh start.

 But prefroming HardRestart is only half of the story, it's important too how
we start a new note. Turning on Gate-bit of course starts the note. But we have
to set new ADSR values for the new note. Before turning on the Gate-bit, we're
in Release phase if gate was turned off and the rate-counter rapidly counts
between 0 and 9 (the internal compare-value for a Release value 0). If we now
change SR register (to the instrument SR-data) before turning on gate-bit,
we lose control over the rate-counter again, because it leaves the 0..9 region
if the new Release is greater than 0. So it's clearly seen it does matter in
what order and timing we set the new AD/SR and gate-bit to start the new note.
 If we were really in Release-phase during the hard-restart, Attack/Decay
register can be set without a problem before turning gate-bit on. If we set it
afterwards, and Attack/Decay-register was not reset during the Hard-Restart,
we can cause a delay-bug if the previous note's Attack was bigger than the new,
because the old Attack is being performed with larger rate-period, only then
comes the new Attack with smaller period, possibly missing a big compare-value.
 Sustain/Release register can safely be set right after turning gate-bit on,
because its value is not used in Attack-phase as rate-counter data-source. In
short, the safest ADSR vs Gate setting order would be: AD -> Gate -> SR.

 As with other quirks of C64, we can turn this delay-bug too to our advantage,
and cause it intentionally. They sometimes call this method the 'sexy-start'.
 The waveform-sequencer table's 1st waveform, which takes a 20ms PAL-frame, will
be inaudible during the 32.8ms delay-period, but the 2nd waveform's end can
be heard in the last 2*20ms-32.8ms = 7.2ms part of the 2nd frame. it's shortened
significantly compared to 20ms, and sound-start is nicer, more 'percussive',
nearly all 1x-framespeed SID music today exploits this effect.
 This is not necessarily preceded by a hard-restart, we can cause fairly stable
delay bug by setting the new note's Attack much smaller than the previous
note's Release. Statistically the delay-bug will happen in nearly 100% of the
cases. Though sometimes glitches in the new notes can happen if the rate-counter
was in the region of the new small Attack's period. If we want even more stable
sexy-soundstart, a hard-restart before it can ensure a more predictable output,
albeit the 4 frames of hardrestart+delaybug activity aggregates the sounds more.
 Life is still not easy, because to cause a delay-bug for sure after a hard-
restart, Release should be set bigger than Attack, enough CPU-cycles must be
waited before turning on gate, so the rate-counter counted up to a region above
the new Attack's counting region. In SID-Wizard I mention it in the source code,
and Lft's BlackBird player has this kind of cyclecounting in the source as well,
but these in-player timings only ensure delay-bug with Release-values above 2,
if Attack-value is smaller.

 A 1-frame shorter/smaller Hard-restart variant exists as well, which is based
on this sexy-restart idea. This method turns off gate-bit and sets Release to
the maximum F value and waits one whole frame to give the highest probability for
the rate-counter to count into the region around 20000 during the 20ms
time-period of this frame. This is also seen in BlackBird and in Cadaver's new
mini-player at github (look for lda #$0F sta $d406,x).
 It doesn't work for all ADSR values, but most Attack/Decay/Release rate-periods
are much smaller than the one corresponding to $F, which is 31251. Value $E has
19532, which is cca half of it, and the other values are getting exponentially
smaller. Even with A/D/R value '$D' counting between 0..11720, setting $F for
1 whole frame, rate-counter counts up to maximum 11720+20000=31720 which is
already safe from wrapping around at 32767. Now that we know our rate-counter
is bigger than 20000 and smaller than 32767 at the end of the 20ms frame,
any Attack-value below $F will result in a wraparound aka delay-bug in the next
frame. The only problem with this approach is that for Attack/Release values
above $D the delay can be small and jittering. But for $0..$C the total delay is
around 29000, as rate-period of $C is 3907, much smaller than with $D..$F.
 Other advantage of this method is that even the typical $09 inaudible 1st-frame
waveform can be omitted, because 20ms was already spent with the 'Hard-Restart',
and the first audible waveform is soon audible in the 2nd half of he next frame.
 This 1frame HardRestart is best for sounds that have short-enough decay/release
or they end/decay before the next note, because the release-value set to $F for
1 frame, while sets the rate-period, it won't ensure a total envelope-decay till
the next gate-on, and while the next note is predictable and always sounds the
same, the Attack phase starts from a nonzero envelope value, is not percussive.

 There's a 'new kind of hard-restart' mentioned at CodeBase64[^4] (by Shrydar, and
Lft is involved here too), they call it 'Bottle', and it's a totally different
cycle-exact code approach. It is able to reset the rate-counter in the timeframe
of about 10 rasterlines (less than 1ms) instead of a 20ms frame, by utilizing
the delaybug-free safe transition from a slow attack to a fast decay. There's
another ADSR-bug in SID too: the envelope-counter can wrap around when it is at
value $FF and an Attack is triggered. This is used to bring the envelope back
to $00 fast in this 'Bottle' approach.
Only time will tell how soon this restart-method gets implemented in players...

[^3]: https://csdb.dk/release/?id=260718

[^4]: https://codebase64.net/doku.php?id=base:a_new_kind_of_hard-restart

## Sound Design hints & tips

_Hermit's comments extracted from CSDb discussion[^5]_

Now, I'll start by some accumulated experiences I had since I started to make
C64 music...
I don't want to take away the mystery from newcomers of exploring the SID sounds
by themselves, therefore I won't give complete solutions to do this or that exactly,
but some tips might come handy to get goin'...
And on the other hand I'd be curious about the approach of others to get kickin' sounds.

Let me start with my basic ADSR knowledge:
My general experience with ADSR is that for most of the sounds the:

'Decay' is usually set to '0' because that has three main advantages:
- the 1st frame's waveform will sound shorter, sexier if 'Release' is bigger than '3'
(due to some little delay-time is still spent during 1st frame when goes through
zero Attack & Decay phases)
- the volume/velocity of the individual notes can be set by the 'Sustain' value
(while using 'Decay' would always reach the max. volume peak)
- most of the sounds start more precisely, at the same time.
(mixing too different ADSR settings would align notes backward/forward a bit)

'Release' should be more than '3' to be on the SID's safe side with repetitive
note-triggerings. (This may depend on player and hard-restart type. Hard restart
is not always necessary, there's a whole topic about that.)
In drums I usually use 6..9 values for 'Release' phase.

These are true for most of my instrument to sound precise, but they're rules
to break whenever it's needed to avoid being too commercial. Sometimes a full-length
1st frame comes handy.

These settings written above work for me with nearly all kind of percussive
drum sounds, but with solo instruments too.

In the "Waveform" program of an instrument we can refine/complicate the existing
ADSR behaviour by utilizing the 'Gate' bit of the waveform. For example,
percussive sounds with adjustable velocity can be produced by setting 'Gate' bit
to '0' somewhere around the 2nd/3rd/4th frame (i.e. program-row)...

__Snare__:
For example, my snare waveforms look like this (pulsewidth 50%): 81 41 41 80
('Gate' bit is cleared in 4th frame)

The very 1st waveform of a sound many times is $09 in trackers, considered
an old type of hard-restart, which triggers 'Test' bit of the wave-control
register and supposedly stabilizes the sound. Nowadays it starts to be unnecessary
for me, I don't tend to set it in 1raster-tracker but still have good sound starts
by using the rules above)
The $81 (hexa) value above gives the sound a crisp, strong percussive noise-start
if used with high pitch-values (usually $d0..$ef in SID-Wizard/GoatTracker).
As I wrote, with good ADSR settings this can be short enough to sound 'sexy'
and 'modern'.
The $41 pulse waveform is used like a sine, imitates when the membrane and body
of a drum resonates in sinus waveform. (Pitch is tipically $98..$a8 for me in SW/GT).
We could filter it to be a real sinus but it's not really necessary.
We could use $11 triangle waveform here which resemples sinus better, but that
would be around half the strength, and usually we need strong snare in the tunes,
so $41 would be a good choice. ($80 coming after $11 would sound a bit weird anyway,
compared to $41-$80 sequence)
The second $41 can be $40 too, because 2-3 frames were enough time for the ADSR
envelope to go to max. volume safely, but if we can leave the 'Gate' as long
as we can, we can get stronger sound... recently I even left out this second $41
waveform and still had a 'body' feel to the snare. But if used it should be a bit
below the pitch of the previous $41 waveform, as the membrane of a drum gets
loosened after the hit. (And in the opposite way, more $41 rows can be added too,
but that's a bit old-fashioned and resembles tom instead of snare IMO...)
So the $80 at the end is the simulation of the snare-wires, its pitch should be
well selected to the previous pitches used with $41 waveforms.(usually $c0..$d0
in SW/GT)

__Kick__:
(waveform-program example: 81 41 41 41 41 11 10 )
The Kick-drum/bass-drum is based on similar principles as the snare, but we don't
have a $80 snare-wire simulation at the end of the waveform-program, but a fat
(50% duty cycle) 'pulse' or a sinus-like 'triangle' waveform. Therefore the kick
can go through many pitches downwards rapidly to give the feel of a hit membrane.
After the initial $81 high pitched 'step'/'kick' sound, pitches can start going down
rapidly but from as much as $90..$a0 (SW/GT) values to very low frequencies
like $84..$88. The pitch changes should be faster 'boosty' in the first frames,
while in the last frames they shouldn't change a lot. That gives more percussive feel,
but I'm sure there are other alternatives...
I usually use the last $11..$10 waveforms to make the kick-sound shorter, but if left
on $41..$40 it will decay stronger, sharper...

The 1st waveforms of drums are not always necessarily $81 waveform (which btw
sounds good most of the times), clever usage of other rich sounds, mixed waveforms
can give even sharper/faster 'clicky' sound-starts if used with fitting pitch-value.
Probably a recording or a storing oscilloscope comes handy to observe this event,
but a good ear and some trials can make them happen too. I took this idea from
Nata's tunes, he made a lot of investigation in this area.

The pitches of drums are sometimes fatter and fit better to the rest of the sounds
in a tune if they're kinda compatible with the key of the tune. I saw kick drum
which had similar sound-sequence like a major chord (e.g. Thiefklang by Gangsta)
and sounded very bassy... Possibly our ear can hear faster than our brain and
it realises even from that fast pitch-sequence that it's better to listen.
On the other hand, for conga/bell like sounds, big fast pitch changes in every
frame chan cheat our ears to hear two distinct sounds. That's the most valuable
trick of waveform-table, used in snare too.
And of course you can use ring-modulation for belly sounds, btw. I'm not expert
in it, but Drax has a lot of good examples for percussive/solo sounds with innovative
ring/sync waveforms... (like snare in Sinful)

For kick and snare you can use filters if possible, which can make it stronger
and cleaner, more on that later... I've heard really good snare and kick sounds
without any filter usage, while using filters are not guarantee for strong
percussive sounds if not used properly...

__Solo/lead sounds__:
Expressions: In the past I always wondered how solo lead sounds sounded so good
in JT's, and other people's tunes. From the C64 Manual we know the plain waveforms
but in BASIC there were no 'vibrato' and 'slide' 'dynamics' and 'sweep' mentioned IIRC.
But IMO these are the things that make it sound good, 'pulse-sweep' in 1st place.
These are the things Jeff is talking about, to use a sound stylistically, using
expressions. This is true for VST instruments too. A 'violin' is really bad if it's
just put into the tune by notes from staff without slides/vibratos/legato/dynamics
where needed...
These are the fields where analog synthesis and C64 is strong compared to
MIDI-controlled VSTs on modern PCs. MIDI is able to make slides/legato, but in my
opinion in a more restricted way than on the C64, I could simulate violin expressions
more easily (e.g. Rakoczi Indulo) than on PC.... (It depends on the VST too of course,
how it handles CC.)
Often in VST based C64 remixes lack the flexible/variable execution of the leads
which we got used to on SID years ago.

_solo-ADSRs_: Most probably the instruments have short attack and no 'decay' by default
 (But the 'decay' can be useful for some kind of solos.) I like to give solo instruments
a long 'release' sometimes, so they has a 'reverb'-like feel to them. But that's not good
for fast, staccato-oriented solos... Later in patterns the ADSR (especially 'attack'
and 'release' can be modified to give variety)
The 'sustain' (i.e. volume of the notes) shouldn't be too harsh, it should be set to be
in balance with the rest of the channels...that could make the tune sound more
professional if some gives attention to volume ratios.
When using created sounds, you should play much attention to 'sustain' to enhance
dynamics of the tune and un-machinarize it a bit, to sound more lively. (A good musician
usually feels like by instinct, where to put stronger/weaker/ghost notes.)
In music theory there's a rule, often in a hierarchical mode the first sounds of a beat
are strong, between them in the middle the sounds are medium-strong, and if there are
notes inbetween, they're usually weak. This can change of course (for example with
syncopation, as in 'Conga Beat' or 'Garden Party' where bass is a bit before
the downbeat..)...
Another useful field of the 'sustain' is to create 'delay echo'-like effect for the lead
sounds on ONE channel. What you do there is you don't stop a note with the gate-off
('---') signal in the pattern, but you leave the note running and you decrease the sustain
value much lower with pattern-FX... (Then later you can put gate-off also...)..
Backwards it doesn't work on the SID by default, 'sustain' can only be decreased without
note-retriggering, cannot be simply increased (will kill the sound)...
To simulate even longer 'delay echo' impressions, you can make the notes change together
with decreasing the 'sustain' value, usually to notes 1-2 rows before, but the base note
of the musical key can even work in that case...
With delay-tricks you can create phase difference between 2 SID channels, that's how
you can create real echoes (Like Robocop3 title beginning solo)... It's up to you
and the music-editor how you solve the delaying of the 'echo' channel, where the solo
content is mainly the same as on the 'dry' original channel... And of course the 'echo'
channel should be more silent, that can be achieved by using another instrument or
placing 'sustain' value pattern-effect for the notes. A good 'economical' solution
can be seen in Drax's 'Winterbird' tune, where on the 'echo' channel he sets a sound
with long 'attack' and the upcoming notes are all played legato....
The delay and volume difference can be really small between the two channels, and that
gives a strong 'room' reverb feel, I like that very much...

_solo-waveforms_: All kinds of waveforms (except $81 in average players) can be used
as lead, it depends on the taste for the tune, or maybe an instrument that should be simulated.

_$11 (triangle)_ - Triangle has half the amplitude, so it can be used as a light
flute-like sound. Not too flexible, but some vibratos/slides/legato can make interesting feelings.
A good trick I see in many tunes that the first frame has a stronger waveform, maybe
the 2nd and 3rd too, so the sound can be heard, and then it goes light into the $11 waveform.

_$21 (sawtooth)_ - This seems to be the best to simulate violin/trumpet-like sounds.
But it stands well for solos in many other cases (e.g. Toggle's SW tunes)..
A plain waveform so it needs some vibrato/slide/ADSR-manipulation/detuning not to
sound too plain. In contrast, trance-like tunes like the plain vibrato-less waveform
but they usually use a lot of other effects in place like detuning...

_$31_ - I don't use it many times but has interesting sound. The problem is on 6581
old SID, where it's very silent IIRC.

$41 (pulse) - The most useful and most flexible waveform is pulse. That's the secret
for good lead sounds, because it can have spectrum from sine-like to sawtooth-like
depending on the pulse-width, and is controllable on fine grade (12 bit, from $000 to $fff)..
The pulse is symmetrical with $800 value, that's the fattest sound (maybe a bit more),
and as we go towards $000 or $fff it gets sharper, richer, with more harmonics, but looses
fatness gradually. I don't hear big difference between very low (e.g. $100) or very high
(e.g. $F00) pulse-width values, but their polarity is the opposite. So selecting them
might depend on the rest of the channels, not to kill some other waveforms by going into
the opposite direction with pulse...
The pulse waveform can also be too plain and machine-like without vibrato/detune/etc.
The most common way, and the real strength of C64 solo voices is the pulsewidth-sweep effect,
which is handled by the players in software. This modifies (increases/decreases)
the pulsewidth in slow/rapid pace, and gives a 'moving' or 'lively' feel to even simple
long notes. In my opinion music is about changes, as our brain interprets differences
instead of absolute values, that's coming from the neurons' workings. (E.g. you feel
smell for a while but you get more used to it after a while. That's coming from nature
and the result of a million years' evolution...on music content side, maybe using dissonances,
and then resolving them lies around the same principle.)
So the pulse-sweep is a solution to give 'difference'&'movement' to a sound. The starting
value of the pulsewidth and the direction/speed should be selected according what you want
to hear (needs some experimenting and practice)...
If you want a thin solo, you can start with $100..$400 values, but if you want a guitar-like
'distorted sine' waveform, you can start with $500..$b00 values. The $800 is the fattest
as a start, but I usually don't like to start a solo with this bumping 50% duty cycle,
instead I start around $700 or $900 which has more harmonics and is less aggressive.
The speed of the sweep depends, I'd classify two kinds:
- Slow sweep ($08..$20 in SW): it's a good technique for 'beautiful' solo sounds...
the majority of C64 tunes uses this technique for solo
- Fast sweep ($20..$70 in SW): it's advantage is that the spectral distribution of the sound
varies rapidly, and a lot of harmonics appear in a short timeframe, often with the feel of detuning.
This generates a 'choir'-like effect on ONE channel, and the solo has a space (this is true
for bass sounds as well, like Golden Axe's starting). I use this kind of solo mainly
in techno/trance like tunes, where that's closer to the style...
The keyboard-tracking (supported with a 4bit resolution in SID-Wizard) can be also a good
thing to enhance the variable feel to a solo. If values are selected right (after experimenting)
you can reach that for example, what a solo-guitarist does by hand: thinner pulsewidth
for deeper notes and fatter pulsewidth (around $800, 50%) for the high-pitched notes...

_$51 (pulse+triangle)_, _$61 (pulse+saw)_, _$71 (pulse+triangle+saw)_ - can produce
interesting sounds with mixed waveform. Probably on 6581 old SID not all of them
are working well like on 8580. I use $51 waveform with $400..$700 pulsewidth setting
for hammond-like sounds (idea taken from Fun Factory solo of Shogoon, I use it in tunes
like Arok 2013 invitro)...
In Lenore I also imitate the 'vocal' at the outro of the tune with mixed waveform.
These waveforms are to be experimented, they might cause some surprises, haven't been
fully utilized yet...
(These waveforms can be used for 'slap bass' imitation too, being similar sometimes...)
Pulse-sweep is not very hearable with the mixed waveforms, but a little bit hearable.
After a certain pulsewidth the sound gets silent...
I had some luck with mixed waveforms to produce brass-like sounds (heard in Garden Party cover).
Especially thudding them with some low-pass or mixed filter... check out Shogoon's 'Sling'
for good brassy sounds.

_$81_ - normally not used as solo (anyway, pitch setting makes sense), but Soundemon's
technique with a special player could generate new waveforms from noise. I haven't seen
a lot of examples of that yet...

Ringmod/sync effects can come handy in solo-sounds occasionally to give the sound
some 'roaring' feel, Jeroen Tel uses this in the 'Eliminator' tune I guess but
didn't check so far if this is what exactly happens there...
There are some examples where ringmod is used extensively throughout the whole tune,
Necropolo is a master of it. His 'engine' sound in 'Cadmium' is unforgettable...

_solo-pitches_: An automatic delayed vibrato can be good, vibratos can be good
to 'un-machinarize' the plain simple pitches, but they shouldn't be used everywhere.
That's why a delay of 8..20 frames can be handy, so the fast note-changes won't suffer
from intonation issues due to vibrato... Usually the vibrato sounds good for long notes,
but sometimes I can use them to cause some extra dynamics (!) as well, especially
when 'sustain' cannot be bigger.. The vibratos can bend the pitch into both directions
on analog synths and SID, but in SID-Wizard they can be set to bend only upwards
(like on guitar without tremolo-arm) or downwards....Increasing the vibrato-amplitude
for a long note over time even gives a special feel (like in old SID tunes, or on real violin)...
Other commonly used trick for solo sounds is to give an octave up/down shift
for the 1st frame of the sound, which cheats the ears to hear a doubled sound.
Non-octave intervals in 1st frame are also good (like a third/fourth) and can give
a totally modified mood to a solo voice (e.g. beginning solo of Pimp My Commodore tune)...
A $81 noise waveform with a certain pitch is also applicable for 1st frames of solo-sounds,
for example pressing a key on the real Hammond organs has this little white-noise
for some milliseconds before the real musical note starts (I use this in Arok 2013
invitation tune).
The detuning can come handy to make 'choir'-like effect on solo sounds, it needs 2 channels,
one in normal pitch, the other with the same note but pitched up/down with some cents.
Most current trackers support this function. A $21 with good detuning sounds like a real
string-ensemble in many SID tunes.

Finally, if you have already mastered the above mentioned techniques to tweak a good sound
you want to hear (or you create a good one by accident), you can make your solo sound
even more interesting by placing a filter/filtersweep on it...

As SID has only one filter
to share between the 3 channels you have several options:
- Make only the solo channel filtered.
- Use the solo channel as primary filter-controller, and the others will follow it.
- Or an other (e.g. bass) channel controls the filter and the solo channel uses the same filter...

Filter sweep is a bit similar to the pulsewidth-sweep, can be controlled in 11bit resolution.
On 6581 the fast filter-changes (cutoff/type/resonance) are heard as popping sound depending
on the degree of change, on 8580 it's much less audible. So on 6581 the wild filter-programs
should be avoided...These clicking sounds can be heard in my 'Deep Though' tune's intro
for example. Increasing frame-speed can make the transitions smoother and these clicks
less audible...
Anyway, when I decide to use filter for solos I usually use gentle, smooth filters...
Jeroen Tel's solos using filter-sweep (and echo) are good examples, like Myth...
Generally filters (pulse/filter) should be two-directional to avoid overflow
when reaching $00 or $ff values, but sometimes it's used as advantage...
Sometimes the sweeps are into only one direction but the notes are short enough
not to wait for this range-overriding...

In more restricted trackers musicians who care about 'expressions' by timbre,
used separate instruments in parts of the solos. For example, the same sound,
but with less 'release' in parts of the solo, where 'staccato' sounding was desired...
For an example to this solution: I used several instruments of different, timbre (pulsewidth)
in 'The Loner' cover, mainly to simulate the solo-guitar sound (described above
with 'keyboard-tracking)...

Many trackers (like DMC,SW for example) support a setting to disable the 'sweep-reset'
on sound start, so a new note won't start the sweep from the starting point but let it continue.
I use this too in Pimp My Commodore solo, this gives even more variety to a solo...
This is also usable to simulate filter-cutoff automation in techno-like tunes,
if there are no pattern-effects for that...


[^5]: https://csdb.dk/forums/index.php?roomid=14&topicid=97576

