# Retro SFX Forge

A retro sound effect synth in the browser — pickups, lasers, explosions, jumps and blips from a seeded generator or a full parameter rack, with WAV export. Runs entirely in your browser.

**Live:** <https://sfx-forge.slippylabs.com/>

## What it does

- Seven shaped generators (pickup, laser, explosion, power-up, hurt, jump, blip) plus a wild card, each seeded so the button gives you a different pickup every press but always a pickup.
- A 20-dial parameter rack: waveform, ADSR with punch, frequency slide and slide acceleration, duty cycle and sweep, vibrato, arpeggio, resonant low-pass with sweep, high-pass, repeat and volume.
- Mutate, to nudge every dial while keeping the character.
- WAV export at 44100/22050/11025 Hz in 16-bit signed or 8-bit unsigned.
- A share string that encodes the whole patch, and a waveform display with the envelope drawn over it.

## How it works

One oscillator, one envelope, and a frequency that moves — the architecture of the sound chips this imitates.

Frequency over time is a **closed form**, not an accumulated per-sample nudge:

    f(t) = f₀ · 2^(slide·t + deltaSlide·t²/2)

so slide is in octaves per second and reads the same at any sample rate. An accumulated nudge makes the same patch sound different at 22 kHz, which is the bug that makes an exported effect not match the preview. Only the phase is accumulated, because it has to be.

Two details that are easy to get wrong: 8-bit WAV is **unsigned** with 128 as silence while 16-bit is **signed** with 0, and scaling a 16-bit sample by 32768 wraps +1.0 to the most negative value there is. And every sound gets a 2 ms ramp at each end on top of its envelope — an instant attack is the point of a blip, but a waveform that starts and stops at full scale is a step, and a step is a click you hear on every single playback.

## Verification

`verify_sfx.py` decodes the exported files with `scipy.io.wavfile`, which parses the RIFF header itself, so a wrong byte rate or block align shows up as a read failure rather than as a passing test.

- **35 WAVs**: every header field checked byte by byte, every sample identical to the rendered float buffer. (JS `Math.round` is half-up while numpy's is half-to-even — the oracle uses `floor(x + 0.5)`, or a sample landing exactly on .5 reads as a one-LSB encoder bug that is not there.)
- A 440 Hz sine measures **439.835 Hz** by FFT; the same test at 22050 Hz measures 999.675 Hz.
- A 2 oct/s sweep tracks the closed form to **0.24%**, measured from the audio in 4096-sample windows.
- The measured envelope follows the attack/sustain/decay parameters to **0.0155** of full scale.
- No waveform carries a DC offset — and a 95% duty square averages +0.90, so that test discriminates.
- The same patch at 11025, 22050 and 44100 Hz has the same duration and the same pitch at the same instant.

**718 checks.**
