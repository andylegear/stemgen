# Stemgen Guide for Michael

This gets you making your own `.stem.m4a` files (playable in Traktor) from your own WAV/FLAC tracks, using Docker so you don't have to install Python or any dependencies yourself.

## 1. Install Docker Desktop

1. Download Docker Desktop for Windows: https://www.docker.com/products/docker-desktop
2. Run the installer, accept the defaults, restart your PC if it asks you to.
3. Open Docker Desktop and wait until it says it's running (whale icon in the system tray goes still/green).

## 2. Get the Stemgen files

You need the full project from GitHub, then swap in the fixed `Dockerfile` Andy sent you.

1. Install Git for Windows: https://git-scm.com/download/win
2. Open a PowerShell window (Start menu → type "PowerShell" → Enter).
3. Clone the repo:
   ```
   cd D:\
   git clone https://github.com/axeldelafosse/stemgen.git
   cd stemgen
   ```
4. Copy the `Dockerfile` (and `.dockerignore`) Andy sent you into this `D:\stemgen` folder, **overwriting** the `Dockerfile` that came from GitHub.

   Check it worked — run `Get-ChildItem D:\stemgen` and make sure you see `stemgen.py`, `stem.py`, `Dockerfile`, `.dockerignore`, and a `stemgen` subfolder all at this top level. If `stemgen.py` isn't there, you copied the wrong folder — go back and re-clone.

The GitHub version's `Dockerfile` is broken (won't build, and even if it did, the final step would crash) — Andy's fixes it:

1. **Base image**: uses `python:3.9-slim-bookworm` instead of `python:3.9-slim-buster` — the `buster` version of Debian is no longer supported, so its package downloads (ffmpeg, sox, etc.) 404. `bookworm` also has the newer system libraries (`glibc`, `libssl`) that the bundled `MP4Box` tool (which stitches the final stem file together) needs to actually run — without this, the last step fails with a `libssl.so.3: cannot open shared object file` error.
2. **CPU-only PyTorch**: installs `torch`/`torchaudio` from the CPU-only wheel index before installing `demucs`. Without this, pip pulls several GB of NVIDIA CUDA packages that are irrelevant on a normal PC and make the build painfully slow.
3. **`.dockerignore`**: added to stop Docker uploading the `.git` folder and screenshots into the build, which was slowing builds down.

## 3. Build the Docker image (one-time setup)

In PowerShell, from inside the `stemgen` folder:

```
docker build -t stemgen .
```

This downloads everything needed (ffmpeg, sox, the AI stem-separation model dependencies) into a self-contained image. It only needs to be done once — takes a few minutes.

## 4. Set up your input/output folders

Still inside the `stemgen` folder:

```
mkdir input
mkdir output
```

Copy the track(s) you want to turn into stems into the `input` folder. Supported formats: `.wav`, `.wave`, `.aiff`, `.aif`, `.flac`.

## 5. Generate a stem file

Run this (replace `mytrack.wav` with your actual filename):

```
docker run --rm -v ${PWD}/input:/stemgen/input -v ${PWD}/output:/stemgen/output stemgen -i /stemgen/input/mytrack.wav -o /stemgen/output -d cpu -n htdemucs
```

What the options mean:
- `-i` — input file (inside the container, always `/stemgen/input/...`)
- `-o` — output folder (always `/stemgen/output`)
- `-d cpu` — use your CPU (fine for this, GPU isn't set up in Docker on Windows)
- `-n htdemucs` — use the Demucs AI model to split into drums/bass/vocals/other (this is the model actually installed in the image)
- Add `-f aac` if you want smaller lossy files instead of the default lossless ALAC

This will take a minute or two per track (longer for longer tracks) — it's running an AI model to separate the stems.

## 6. Get your file

Your finished `.stem.m4a` file will appear in the `output` folder, named after your track (e.g. `output\mytrack\mytrack.stem.m4a`). Copy that into Traktor's music folder and it should show up as a stem-enabled track.

## Doing more tracks

Once it's built (step 3), you only need to repeat steps 4–6 for each new track — no need to rebuild the image.

## If something goes wrong

- **"docker: command not found" or connection errors** — Docker Desktop isn't running. Open it from the Start menu and wait for it to say "Running".
- **Build fails with a network/package error** — let me know, sometimes package mirrors change and the Dockerfile needs a small tweak.
- **Output file sounds odd / stems missing** — double-check your input filename matches exactly what you typed after `-i`.
