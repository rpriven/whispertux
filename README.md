# WhisperTux

> **⚡ Performance-Optimized Fork:** This fork includes significant performance improvements and comprehensive documentation. See [Performance Optimizations](#performance-optimizations) below for details. Original project: [cjams/whispertux](https://github.com/cjams/whispertux)

Simple voice dictation application for Linux. Press the shortcut key, speak, press the shortcut key again, and text will appear in whatever app owns the cursor at the time.

Uses [whisper.cpp](https://github.com/ggml-org/whisper.cpp) for offline speech-to-text transcription.
No fancy GPUs are required although whisper.cpp is capable of using them if you have one available. Once your speech is transcribed, it is sent to a
[ydotool daemon](https://github.com/ReimuNotMoe/ydotool) that will write the text into the focused application.

Super useful for voice prompting AI models and speaking terminal commands.

Here's a quick [demo](https://www.youtube.com/watch?v=6uY2WySVNQE)

## Screenshots

<table align="center">
<tr>
<td><img src="https://raw.githubusercontent.com/cjams/whispertux/main/assets/whispertux-main.png" alt="whispertux-main" width="400"></td>
<td><img src="https://raw.githubusercontent.com/cjams/whispertux/main/assets/whispertux-settings.png" alt="whispertux-settings" width="400"></td>
</tr>
</table>

## Features

- Local speech-to-text processing via whisper.cpp (no cloud dependencies)
- No expensive hardware required (works well on a plain x86 laptop with AVX instructions)
- Global keyboard shortcuts for system-wide operation
- Automatic text injection into focused applications
- Configurable [whisper](https://github.com/openai/whisper) models and shortcuts

## Performance Optimizations

This fork includes significant performance improvements discovered through extensive testing on AMD Ryzen CPUs:

### Thread Count Tuning (5.7x Faster!)

**Key Discovery:** Using optimal thread count instead of all CPU cores dramatically improves performance.

- **Before:** 24+ seconds for 11-second audio sample (16 threads)
- **After:** ~3 seconds for same sample (12 threads, benchmarked optimal for AMD Ryzen 7 8845HS)
- **Improvement:** ~8x faster transcription
- **Note:** Optimal count varies by CPU - test with benchmark.sh

**Why fewer threads are faster:**
- Prevents cache thrashing between cores
- Reduces memory bandwidth saturation
- Eliminates thread synchronization overhead

**Optimal thread counts by CPU:**
- 8-core CPUs (16 threads): Use 4 threads
- 6-core CPUs (12 threads): Use 3-4 threads
- 4-core CPUs (8 threads): Use 2-3 threads

See [WORKING_CONFIG.md](WORKING_CONFIG.md) for detailed build configuration and benchmarks.

### How to Apply These Optimizations

**The optimization is already built into this fork!** The thread count is configured in `src/whisper_manager.py`.

**If using the original WhisperTux and want this optimization:**

1. Edit `src/whisper_manager.py` (around line 147)
2. Change the threads parameter:
```python
# Find this line:
'--threads', '16',  # or whatever it currently is

# Change to optimal for your CPU:
'--threads', '12',   # For AMD Ryzen 7 8845HS (8-core/16-thread)
                     # Benchmark different values: 4, 8, 12 to find your optimal
```

**To find your optimal thread count:**
```bash
# Get your CPU core/thread count
nproc  # Shows total threads

# General rule: Use 1/4 to 1/3 of total threads
# Example: 16 threads → use 4
#          12 threads → use 3-4
#           8 threads → use 2-3
```

**Test different thread counts:**
```bash
# Use the included benchmark script
./benchmark.sh
```

### Comprehensive Documentation

- **[WORKING_CONFIG.md](WORKING_CONFIG.md)** - Optimal whisper.cpp build configuration, thread tuning, model selection
- **[RECOVERY_NOTES.md](RECOVERY_NOTES.md)** - Troubleshooting guide, performance debugging, recovery procedures

### Contributing Back

These optimizations are being submitted as PRs to the upstream project. If you benefit from these improvements, please consider contributing back to [cjams/whispertux](https://github.com/cjams/whispertux).

## Installation

Run the setup script:

```bash
git clone https://github.com/cjams/whispertux
cd whispertux
python3 setup.py
```

The setup script handles everything: system dependencies, creating Python virtual environment, building whisper.cpp, downloading models, configuring services, and testing the installation. See [setup.md](docs/setup.md) for details.

## Usage

Start the application:

```bash
./whispertux
# or
python3 main.py
```

### Desktop Integration (Optional)

After building the project, you can add WhisperTux to your desktop environment's applications menu:

```bash
# Create desktop entry for GNOME/KDE/other desktop environments
bash scripts/create-desktop-entry.sh
```

This will:

- Add WhisperTux to your applications menu
- Optionally configure it to start automatically on login
- Create proper desktop integration for launching from GUI

### Basic Operation

1. Press $GLOBAL_SHORTCUT (configurable within the app) to start recording
2. Speak clearly into your microphone
3. Press $GLOBAL_SHORTCUT again to stop recording
4. Transcribed text appears in the currently focused application

You can say 'tux enter' to simulate Enter keypress after you're done speaking for
automated carriage return.

You can also add overrides that will replace words before writing the
final output text. For example, if you want every instance of 'duck' to
be replaced by 'squirrel', you would add an override in the Word Overrides
section with Original being 'duck'.

## Configuration

Settings are stored in `~/.config/whispertux/config.json`:

```json
{
  "primary_shortcut": "F12",
  "model": "base",
  "key_delay": 15,
  "use_clipboard": false,
  "always_on_top": true,
  "theme": "darkly",
  "audio_device": null
}
```

### Available Models

Any [whisper](https://github.com/openai/whisper) model is usable. By default the
base model is downloaded and used. You can download additional models from within the app.

## System Requirements

- Linux with a GUI. Has only been tested on GNOME/Ubuntu but should work on others. Depends on evdev for handling low-level input events
- Python 3
- Microphone access

## Troubleshooting

### Global Shortcuts Not Working

Test shortcut functionality:

```bash
python3 -c "from src.global_shortcuts import test_key_accessibility; test_key_accessibility()"
```

### Audio Issues

Check microphone access:

```bash
python3 -c "from src.audio_capture import AudioCapture; print(AudioCapture().is_available())"
```

List available audio devices:

```bash
python3 -c "from src.audio_capture import AudioCapture; AudioCapture().list_devices()"
```

### Text Injection Problems

If you see `failed to open uinput device` errors, run the fix script:

```bash
./scripts/fix-uinput-permissions.sh
```

This script will:

- Add your user to the `input` and `tty` groups
- Create the necessary udev rule for `/dev/uinput` access
- Reload udev rules

You may need to log out and back in or reboot for group changes to take effect.

Verify ydotoold service status:

```bash
systemctl status ydotoold
sudo systemctl restart ydotoold  # if needed
```

Test text injection directly:

```bash
ydotool type "test message"
```

### Whisper Model Issues

Check available models:

```bash
python3 -c "from src.whisper_manager import WhisperManager; print(WhisperManager().get_available_models())"
```

Download models manually:

```bash
cd whisper.cpp/models
bash download-ggml-model.sh base.en
```

## Documentation

- [Architecture](docs/architecture.md) - Technical architecture and component design
- [Setup Details](docs/setup.md) - Manual installation and system configuration

## License

MIT License
