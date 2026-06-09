# heavy-run

Run a heavy command inside a **memory-capped systemd user scope**, so that when it blows past available RAM the kernel OOM-kills _the command_ — not your whole machine, and not (under WSL2) the entire Linux VM.

```bash
heavy-run Rscript analysis/run_pipeline.R
HEAVY_RUN_MEM=10G HEAVY_RUN_SWAP=2G heavy-run python train.py
```

## The problem it solves

Fork-parallel workloads — R `mclapply` / `BiocParallel`, Python `multiprocessing.Pool` / `joblib` — default to _all_ cores and each worker inherits the parent's heap. Peak memory is closer to `parent_RSS × (1 + N_workers)` than anyone expects, and when it exceeds physical RAM the result isn't a clean error: the Linux OOM killer thrashes, and on **WSL2** the whole VM can lock up or die (`wsl.exe` exits, broken pipe, "catastrophic failure").

`heavy-run` puts the command in its own cgroup with a hard `MemoryMax`. If the job exceeds the cap, only the job dies — with a normal non-zero exit — while the rest of the system stays responsive.

## Usage

```
heavy-run <command> [args...]
```

| Env var             | Meaning                                                                                           | Default |
| ------------------- | ------------------------------------------------------------------------------------------------- | ------- |
| `HEAVY_RUN_MEM`     | `MemoryMax` for the scope                                                                         | `14G`   |
| `HEAVY_RUN_SWAP`    | `MemorySwapMax` for the scope                                                                     | `4G`    |
| `HEAVY_RUN_RESERVE` | GiB of `MemAvailable` to keep free; refuse to launch if the cap would dip below it (`0` disables) | `2`     |
| `HEAVY_RUN_FORCE=1` | Skip the headroom pre-flight entirely                                                             | —       |

Sizes accept `K`/`M`/`G` suffixes (e.g. `10G`, `512M`).

### Headroom pre-flight

Before launching, `heavy-run` reads `MemAvailable` from `/proc/meminfo` and refuses if granting the requested `MemoryMax` would leave less than `HEAVY_RUN_RESERVE` GiB free — so you find out _now_, not after a doomed job churns for an hour and gets killed anyway. Override with `HEAVY_RUN_FORCE=1`.

## Requirements

- **User-mode systemd** with the memory controller delegated to the user slice. This is the default on modern systemd Linux, and on WSL2 with `systemd=true` in `/etc/wsl.conf`.
- If `systemd-run` isn't found, `heavy-run` prints a warning and runs the command **unprotected** rather than failing.

Verify your setup:

```bash
systemd-run --user --scope --quiet -p MemoryMax=1G -- true && echo "delegation OK"
```

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/musharna/heavy-run/main/heavy-run -o ~/.local/bin/heavy-run
chmod +x ~/.local/bin/heavy-run
```

(Ensure `~/.local/bin` is on your `PATH`.)

> Review the script before installing — it sets cgroup memory limits and execs arbitrary commands you pass to it.

## How it works

It's a single ~90-line bash script. The core is one line:

```bash
exec systemd-run --user --scope --quiet --same-dir \
  -p MemoryMax="$MEM" -p MemorySwapMax="$SWAP" -- "$@"
```

`--scope` runs the command synchronously in the foreground (you keep stdout/stderr and the exit code); `--same-dir` preserves your working directory; the `-p` properties set the cgroup memory limits.

## License

MIT — see [LICENSE](LICENSE).
