# Ollama on Raspberry Pi 5 with External SSD Storage

Running [Ollama](https://ollama.com) locally on a Raspberry Pi 5, with models stored on an external SSD instead of the SD card, running fully offline and CPU-only.

This repo documents the real process — including the mistakes made along the way and how they were diagnosed. The troubleshooting is arguably the most useful part; most quick-start guides skip it entirely.

**Why:** this is the foundation for a follow-up project — an LLM-driven SSH honeypot for defensive security research (see [Roadmap](#roadmap)).

## Hardware & Software Used

| Component | Spec |
|---|---|
| Board | Raspberry Pi 5, 16GB RAM |
| Storage | Samsung T7 Shield, 1TB (USB 3.0 external SSD) |
| OS | Raspberry Pi OS (Debian 13 "trixie", kernel 6.18.39-rpt) |
| Runtime | Ollama (official install script) |
| First model | `gemma3:1b` |

## Repository Structure

```
.
├── README.md
├── LICENSE
└── scripts/
    ├── 01-format-mount-ssd.sh       # Format & permanently mount external SSD
    ├── 02-install-ollama.sh         # Download + review + install Ollama
    ├── 03-configure-model-storage.sh # Point Ollama at the SSD via systemd
    └── verify-setup.sh              # Confirm mount, service, and env are correct
```

The scripts are the cleaned-up, parameterized versions of the exact commands used below — meant to be read before running, not blindly executed. Each has comments explaining what it does and why.

## Usage

> ⚠️ `01-format-mount-ssd.sh` **erases the target drive.** Double-check the device path with `lsblk -f` before running it.

```bash
git clone https://github.com/<your-username>/ollama-pi5-ssd-setup.git
cd ollama-pi5-ssd-setup/scripts

chmod +x *.sh

sudo ./01-format-mount-ssd.sh /dev/sda1
sudo ./02-install-ollama.sh
sudo ./03-configure-model-storage.sh /mnt/ssd/llm_data
./verify-setup.sh
```

Then pull a model:

```bash
ollama run gemma3:1b
```

## Walkthrough & Troubleshooting Log

### 1. Preparing the external SSD

The Pi needs somewhere to store models that isn't the SD card — model files run from hundreds of MB to several GB each, and SD cards are slow and wear out under repeated writes.

Checking what's connected:
```bash
lsblk -f
```

The SSD showed up as `/dev/sda1`, pre-formatted as **exFAT** (its out-of-the-box format for cross-platform use with Windows/Mac). exFAT works, but ext4 is the better choice on Linux — better performance and no permission headaches with services running as their own system user (relevant later).

Reformatting is destructive — the only thing on the drive was Samsung's bundled desktop software (Samsung Magician, an update-checker certificate), nothing needed on Linux, so it was safe to proceed:

```bash
sudo umount "/media/<user>/T7 Shield"
sudo mkfs.ext4 -L ssd_llm /dev/sda1
```

Mounted and made permanent via `/etc/fstab`, referencing the drive's UUID rather than `/dev/sda1` (which can shift if other drives are plugged in):

```bash
sudo mkdir -p /mnt/ssd/llm_data
sudo mount /dev/sda1 /mnt/ssd/llm_data
```

```
# /etc/fstab
UUID=54b9596b-ac7f-42fe-9b4f-f68b7194c2d2  /mnt/ssd/llm_data  ext4  defaults,noatime  0  2
```

**Bug #1 — mistyped UUID:**
```
mount: /mnt/ssd/llm_data: can't find UUID=54b959b-ac7f-42fe-9b4f-f68b7194c2d2.
```
A single digit was dropped while typing a 32-character UUID by hand (`54b959b` vs. the correct `54b9596b`). Lesson: copy UUIDs directly from `lsblk -f` output rather than retyping them, and always test an `fstab` edit before rebooting on it:
```bash
sudo systemctl daemon-reload
sudo umount /mnt/ssd/llm_data
sudo mount -a
```

### 2. Installing Ollama

Downloaded first rather than piped straight into a shell, so its contents could be reviewed before execution:

```bash
curl -fsSL https://ollama.com/install.sh -o install.sh
chmod +x install.sh
sudo ./install.sh
```

Confirmed: ARM64 binary installed to `/usr/local/bin/ollama`, a dedicated `ollama` system user created, a systemd service created and started on `127.0.0.1:11434`. The install script correctly reports `WARNING: No NVIDIA/AMD GPU detected. Ollama will run in CPU-only mode.` — expected, since the Pi has no discrete GPU.

### 3. Moving model storage to the SSD

**The mistake to avoid:** setting `OLLAMA_MODELS` in `~/.bashrc`. This does nothing, because Ollama runs as a **systemd service** under its own `ollama` user — not through an interactive login shell. A shell config file is invisible to it.

The correct approach is a systemd drop-in override:

```bash
sudo chown -R ollama:ollama /mnt/ssd/llm_data
sudo systemctl edit ollama.service
```

Typed as **two separate lines**:
```ini
[Service]
Environment="OLLAMA_MODELS=/mnt/ssd/llm_data"
```

**Bug #2 — merged lines:** typed directly into the `systemctl edit` nano session, the restart produced:
```
Invalid section header '[Service] Environment="OLLAMA_MODELS=/mnt/ssd/llm_data"'
```
Checking the actual file confirmed both lines had been joined into one (likely nano autoindent/paste behavior) — systemd requires them separate. The fix was writing the file directly, which guarantees the line break:

```bash
sudo tee /etc/systemd/system/ollama.service.d/override.conf > /dev/null << 'EOF'
[Service]
Environment="OLLAMA_MODELS=/mnt/ssd/llm_data"
EOF
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

**Lesson:** the service reported `active (running)` even while this config was silently broken — status alone doesn't confirm a setting took effect. Verify explicitly:
```bash
sudo systemctl show ollama --property=Environment --no-pager | tr ' ' '\n'
# → OLLAMA_MODELS=/mnt/ssd/llm_data
```

### 4. First model & benchmark

```bash
ollama run gemma3:1b
```

Verified the model landed on the SSD, not the SD card:
```bash
ls -lh /mnt/ssd/llm_data
# blobs/  manifests/   (owned by ollama:ollama)
```

**Benchmark:** prompt *"Explain what a Raspberry Pi is in two sentences"* returned in **7.4 seconds** for a ~65-word answer — roughly **10-12 tokens/second**. Published benchmarks for `gemma3:1b` on a Pi 5 report closer to 18-22 t/s; to rule out thermal throttling as the cause:

```bash
vcgencmd get_throttled
# throttled=0x0
```

`0x0` confirms zero throttling — the gap is most likely cooling setup or background load differences from whatever environment those published numbers came from, not a fault in this configuration.

## Lessons Learned

1. Mount external storage using UUIDs, not device paths — device names can shift.
2. Systemd services don't inherit your shell's environment variables — a very common mistake with `.bashrc` exports.
3. "Active (running)" doesn't mean a config change took effect — verify the actual applied setting, not just service status.
4. Small formatting mistakes (a merged line, a mistyped UUID) produce confusing errors; reading the raw file/log content directly is the fastest path to diagnosing them.
5. Measure real-world performance instead of assuming — `vcgencmd get_throttled` turns "feels slow" into either a confirmed hardware limit or a ruled-out one.

## Roadmap

- [ ] LLM-driven SSH honeypot (Cowrie + Ollama) generating dynamic fake shell responses to attacker input
- [ ] Offline log-summarization pass using a larger local model (`mistral:7b`), mapping captured sessions to MITRE ATT&CK
- [ ] Write-up + repo for the above

## License

MIT — see [LICENSE](LICENSE).
