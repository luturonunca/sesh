# sesh

Lightweight Bash CLI toolkit for inspecting RAMSES simulation outputs.

## Installation

Clone the repo and either run directly or add to your PATH:

```bash
# Option 1: run from repo root
chmod +x bin/sesh
export PATH="$PWD/bin:$PATH"

# Option 2: symlink to a directory already in PATH
ln -s /path/to/sesh/bin/sesh ~/.local/bin/sesh
```

Add the `export` line to your `~/.bashrc` or `~/.bash_profile` to make it permanent.

## Quick reference

```bash
sesh z <output_dir>
sesh z all
sesh time <output_dir>
sesh time all
sesh eventstats <outputs_root> [--from N]
sesh simstatus [--path DIR]
sesh sinkmass <sim_dir> <id>
```

---

## Subcommands

### `sesh z <output_dir>`

Print the redshift of a single output snapshot.

Reads `aexp` from `info_XXXXX.txt` and computes `z = 1/aexp - 1`.

```bash
sesh z /path/to/sim/output_00042
# output_00042 z=3.14159265
```

Use `all` to iterate over all `output_XXXXX` directories in the current directory:

```bash
cd /path/to/sim
sesh z all
# output_00001 z=9.000
# output_00002 z=7.432
# ...
```

---

### `sesh time <output_dir>`

Print the cosmological time and lookback time of a single output snapshot.

Reads `aexp`, `H0`, `omega_m`, and `omega_l` from `info_XXXXX.txt` and integrates the Friedmann equation.

```bash
sesh time /path/to/sim/output_00042
# z=3.142  t=2.131 Gyr  lookback=11.532 Gyr
```

Use `all` to iterate over all snapshots in the current directory:

```bash
cd /path/to/sim
sesh time all
# output_00001 z=9.000  t=0.537 Gyr  lookback=13.126 Gyr
# output_00002 z=7.432  t=0.712 Gyr  lookback=12.951 Gyr
# ...
```

---

### `sesh eventstats <outputs_root> [--from N]`

Per snapshot, count stellar feedback events by type from `stars_*.out*` files.

Event IDs correspond to RAMSES stellar particle types:

| ID | Label | Meaning |
|----|-------|---------|
| 0  | SF    | Star formation event |
| 1  | SN    | Supernova (single star) |
| 2  | B-SF  | Binary star formation |
| 3  | B-SN2 | Binary neutron star / second supernova |

Use `--from N` to skip outputs before `output_N`:

```bash
sesh eventstats /path/to/sim
sesh eventstats /path/to/sim --from 12

# output_00012 SF:104 SN:17 B-SF:3 B-SN2:1        z=4.21
# output_00013 SF:98  SN:21 B-SF:2 B-SN2:0        z=3.87
```

Non-zero counts are highlighted in cyan.

---

### `sesh simstatus [--path DIR]`

Print a one-line summary per simulation showing the latest snapshot, redshift, and how recently the output was written.

Iterates over all subdirectories of the target path, finds the latest `output_XXXXX` in each, and reads `aexp` and the file modification time from `info_XXXXX.txt`.

```bash
sesh simstatus
sesh simstatus --path /path/to/simulations

# snapshot      simulation                                                       z_latest     last_write
# -----------   ---------------------------------------------------------------  ----------   ----------------
# output_00042  cosmo_hr_bns                                                     3.142        2026-03-12 14:05 (>1d)
# output_00017  cosmo_lr_test                                                    6.800        2025-12-01 09:22 (>1mo)
```

#### Color coding by age of last write

| Color  | Tag   | Meaning                        |
|--------|-------|--------------------------------|
| Green  | ~1h   | Written less than 1 hour ago   |
| Cyan   | >1h   | Written 1 hour – 1 day ago     |
| Blue   | >1d   | Written 1 – 7 days ago         |
| Yellow | >1w   | Written 1 week – 1 month ago   |
| Red    | >1mo  | Written more than 1 month ago  |

Colors are suppressed when output is not a TTY or when `NO_COLOR` is set.

---

### `sesh sinkmass <sim_dir> <id>`

Print the total sink mass and black hole mass of a given sink particle in solar masses, from the latest snapshot.

Reads `sink_XXXXX.csv` in the latest `output_XXXXX` and converts from code units to solar masses using `unit_l` and `unit_d` from the corresponding `info_XXXXX.txt`.

```bash
sesh sinkmass /path/to/sim 3
# BH 3  msink=2.1543e+09 Msol  msmbh=1.8732e+09 Msol  (output_00042)
```

#### Columns reported

| Field   | Source column in `sink_XXXXX.csv` | Description                          |
|---------|-----------------------------------|--------------------------------------|
| `msink` | 2                                 | Total sink particle mass             |
| `msmbh` | 21                                | Black hole mass (SMBH sub-component) |

#### Unit conversion

Both masses are stored in code units in the CSV. The conversion to solar masses is:

```
M_sol = msink * unit_d * unit_l^3 / 1.9885e33
```

where `unit_d` (g/cm³) and `unit_l` (cm) are read from the `info_XXXXX.txt` of the same snapshot.
