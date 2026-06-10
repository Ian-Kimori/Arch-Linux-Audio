# Arch-Linux-Audio-Debugging

---

### Step 1 — Check if the audio server is running
```bash
systemctl --user status pipewire wireplumber
```
Confirms PipeWire (the audio server) and WirePlumber (the session manager that routes hardware to PipeWire) are both active.

---

### Step 2 — Check what sink PipeWire is using
```bash
pactl list sinks short
```
If the output shows `auto_null` or `Dummy Output`, PipeWire is running but has no real audio device. If it shows something like `alsa_output.pci-...`, audio is wired up correctly.

---

### Step 3 — Check if ALSA sees any soundcards
```bash
cat /proc/asound/cards
```
ALSA is the kernel's audio layer. If this says `no soundcards`, the problem is below PipeWire — at the driver or firmware level.

---

### Step 4 — Check what driver is bound to the audio hardware
```bash
lspci | grep -i audio
```
```bash
lsmod | grep snd
```
`lspci` shows what audio hardware you have.

`lsmod` shows which kernel modules (drivers) are loaded.

Look for `snd_sof_*` modules — these are SOF (Sound Open Firmware) drivers that require separate firmware blobs to function.

---

### Step 5 — Check if the required firmware is present
```bash
ls /lib/firmware/intel/sof/
```
```bash
ls /lib/firmware/intel/sof-tplg/
```
If these directories don't exist, the SOF driver has claimed your device but has no firmware to run — the card never initialises.

---

### Step 6 — Install the missing firmware
```bash
sudo pacman -S sof-firmware
```
```bash
sudo reboot
```
Installs the Intel SOF firmware blobs and reboots so the driver can load them properly.

---

### Step 7 — Verify after reboot
```bash
cat /proc/asound/cards
```
```bash
pactl list sinks short
```
```bash
wpctl status
```
`/proc/asound/cards` should now list your card.

`pactl` should show a real `alsa_output.*` sink.

`wpctl status` should show your device under Audio → Devices and Sinks.

---

### The stack to keep in mind
```
Application
    ↓
PipeWire / PulseAudio   →  pactl, wpctl
    ↓
WirePlumber             →  systemctl --user status wireplumber
    ↓
ALSA                    →  /proc/asound/cards
    ↓
Kernel driver           →  lsmod, lspci
    ↓
Firmware                →  /lib/firmware/intel/
    ↓
Hardware
```
