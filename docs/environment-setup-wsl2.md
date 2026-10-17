# Environment setup: Windows + WSL2 + CUDA

Do this once on the laptop. Steps marked **[Windows]** happen in Windows; **[WSL]** happen in the Ubuntu terminal. Claude Code can guide each step, but a few Windows steps need you to click.

Reference: NVIDIA's CUDA on WSL User Guide — https://docs.nvidia.com/cuda/wsl-user-guide/index.html

## 1. Windows driver [Windows]

1. Install or update the latest NVIDIA Windows driver for the laptop GPU from https://www.nvidia.com/Download/index.aspx (or the GeForce/NVIDIA app).
2. This is the **only** NVIDIA driver you install. Inside WSL, the Windows driver is exposed automatically. **Never install a Linux NVIDIA driver inside WSL.**

## 2. WSL2 [Windows, PowerShell as administrator]

1. `wsl --install` (installs WSL2 with Ubuntu), then reboot if asked.
2. `wsl --update` to get the latest WSL kernel.
3. `wsl -l -v` — the Ubuntu distro must show VERSION 2.

**Give WSL enough memory.** By default WSL can use only part of the laptop's RAM. Converting SF10 data needs several GB. Create `C:\Users\<you>\.wslconfig` with, for example:

```
[wsl2]
memory=12GB
swap=8GB
```

Choose `memory` at about 60–75% of the laptop's RAM. Then run `wsl --shutdown` and reopen Ubuntu.

## 3. Check the GPU is visible [WSL]

Run `nvidia-smi`. It should list the GPU. (If not found as root, it lives at `/usr/lib/wsl/lib/nvidia-smi`.) Under WSL, `nvidia-smi` cannot show GPU utilisation; that is normal.

## 4. CUDA toolkit [WSL]

1. Open https://developer.nvidia.com/cuda-downloads and choose **Linux → x86_64 → WSL-Ubuntu → 2.0 → deb (network)**.
2. Run the commands that page shows, but for the final install line use the **`cuda-toolkit-<version>`** package only.
3. **Do not** install the `cuda`, `cuda-<version>` or `cuda-drivers` packages; they try to install a Linux driver and break GPU access in WSL.
4. Add to `~/.bashrc`:
   - `export PATH=/usr/local/cuda/bin:$PATH`
   - `export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH`
5. Open a new terminal and check `nvcc --version`.

## 5. Build tools and Python [WSL]

- `sudo apt update && sudo apt install -y build-essential cmake git python3-pip python3-venv`
- Create a virtual environment in the project and install `duckdb numpy pandas matplotlib`.
- GoogleTest is fetched by CMake later; nothing to install now.

## 6. Claude Code [WSL]

Install Claude Code inside WSL (not Windows) so it runs next to the CUDA toolkit. Follow the current instructions at https://docs.claude.com/en/docs/claude-code/overview, then sign in. Start it from the project folder with `claude`.

## 7. Project location [WSL]

Unzip this setup into your Linux home, e.g. `~/warpql`. **Do not work under `/mnt/c/...`**; file access there is many times slower and slows builds and data loading.

To open the folder in Windows Explorer: run `explorer.exe .` from inside it. VS Code: install the "WSL" extension and run `code .` from inside WSL.

## 8. SSB data generator [WSL]

- Clone https://github.com/eyalroz/ssb-dbgen and build it following its README (it uses CMake).
- Generate SF1 first: `dbgen -s 1 -T a` generates all tables (check the README; some forks need one `-T` flag per table: `c`, `p`, `s`, `d`, `l`).
- Keep generated data in `~/warpql/data/` and add `data/` to `.gitignore`.
- Generate larger scale factors only after `/hardware-check` decides the maximum.

## 9. Profilers on WSL2

From NVIDIA's WSL guide: Compute Sanitizer works on Pascal and later; Nsight Systems CLI on Volta and later; profilers need Windows 11 with driver r525+ (or Windows 10 build 19044+ with driver r545+), Volta or later.

- `compute-sanitizer` comes with the toolkit.
- `nsys` and `ncu` come with the toolkit (or as separate downloads).
- If `ncu` fails with `ERR_NVGPUCTRPERM`, enable counter access on Windows: NVIDIA Control Panel → Desktop menu → enable Developer Settings → Developer → Manage GPU Performance Counters → "Allow access to the GPU performance counters to all users". Then restart WSL (`wsl --shutdown`).
- If `ncu` still does not work, fall back to CUDA-event timing plus bytes-moved bandwidth. Note this in `docs/hardware.md`.

## 10. Known WSL2 limitations that affect us

- **Unified Memory** is not fully supported → we do not use it.
- **Pinned host memory** is limited → keep pinned buffers modest; transfer/overlap experiments are optional.
- **GPU utilisation** is not reported by `nvidia-smi` → use Nsight or our own timings.

## 11. Before every benchmarking session

- Laptop plugged in; Windows power mode set to best performance.
- Close games, browsers with video, and other GPU users.
- Let the laptop cool between long runs; record room conditions if results vary a lot.
