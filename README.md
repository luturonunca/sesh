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
sesh walltime <sim_dir> [--to-z Z]
sesh sinkmass <sim_dir> <id>
sesh report <sim_dir> [--fields f1,f2,...|--preset name] [--out file] [--from N]
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

Per snapshot, count stellar feedback events by type from `stars_*.out*` files, plus the number of sinks born.

Event IDs correspond to RAMSES stellar particle types:

| ID | Label | Meaning |
|----|-------|---------|
| 0  | SF    | Star formation event |
| 1  | SN    | Supernova (single star) |
| 2  | B-SF  | Binary star formation |
| 3  | B-SN2 | Binary neutron star / second supernova |
| 4  | B-MRG | Binary merger |

Use `--from N` to skip outputs before `output_N`:

```bash
sesh eventstats /path/to/sim
sesh eventstats /path/to/sim --from 12

# output  ,        z  ,      SF  ,      SN  ,    B-SF  ,   B-SN2  ,   B-MRG  ,   Sinks
#     12  ,   4.2100  ,     104  ,      17  ,       3  ,       1  ,       0  ,       2
#     13  ,   3.8700  ,      98  ,      21  ,       2  ,       0  ,       0  ,       0
```

Non-zero counts are highlighted in cyan.

#### `Sinks` column: sink formation count

`stars_*.out*` files carry no information about sinks, and `sink_XXXXX.csv` is only a snapshot of currently-existing sinks (one row per live sink, no per-snapshot formation log). `Sinks` is therefore computed as the set difference between the sink IDs present in the current output and those present in the previous *processed* output — i.e. how many new sinks formed since then.

This is **not** simply `nsink_curr - nsink_prev`: RAMSES sinks can merge (`clean_merged_sinks` in `pm/sink_particle.f90`), which removes a sink from the count, so a plain difference undercounts births whenever a formation and a merger land in the same interval. Sink IDs are assigned from a monotonically increasing counter and are never reused after a merge, so comparing ID sets is exact.

`Sinks` is `NA` for the first processed snapshot (no prior state to diff against) and whenever `sink_XXXXX.csv` is missing for that output (e.g. sink physics disabled).

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

### `sesh walltime <sim_dir> [--to-z Z]`

Print cumulative wall time and CPU cost per output snapshot.

Reads the `TOTAL` wall time from each `timer_XXXXX.txt` and `ncpu` from the corresponding `info_XXXXX.txt`. Outputs are sorted by snapshot number. Use `--to-z Z` to stop at the first output with redshift below `Z`.

```bash
sesh walltime /path/to/sim
sesh walltime /path/to/sim --to-z 4.0

# output           z     step      cumul   ncpu    CPU-Kh
# -------------  -------  -------  --------  -----  ---------
# output_00001   9.0000   0.23h    0.23h      256       0.02
# output_00002   7.4320   0.28h    0.51h      256       0.03
# ...
# output_00042   3.1416   0.41h   12.34h      256       0.81
```

| Column     | Description                                             |
|------------|---------------------------------------------------------|
| `z`        | Redshift at that output                                 |
| `step`     | Wall time for this output step (hours)                  |
| `cumul`    | Cumulative wall time from start (hours)                 |
| `ncpu`     | Number of MPI ranks                                     |
| `CPU-Kh`   | Cumulative CPU cost in kilo CPU-hours (2 decimals)      |

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

---

### `sesh report <sim_dir> [options]`

Write a per-snapshot CSV table to a file, with columns chosen by `--fields` or a named `--preset`. Snapshots are sorted by number; rows where a field can't be computed (missing file, missing unit) get `NA`. Output is prefixed with a `#` header row for `numpy.genfromtxt`-style loading.

```bash
sesh report /path/to/sim --preset cosmo --out cosmo.csv
sesh report /path/to/sim --fields z,t,sinkmass:3,accrate:3 --out sink3.csv
```

#### Options

| Option           | Description                                              |
|------------------|------------------------------------------------------------|
| `--fields f1,f2,...` | Comma-separated list of fields (see table below)      |
| `--preset name`  | Named field combination (mutually exclusive with `--fields`) |
| `--out file`     | Output file (default: `sesh_report.csv`)                 |
| `--from N`       | Skip outputs before `output_N`                            |

#### Fields

| Field          | Column header            | Source                                              |
|----------------|---------------------------|------------------------------------------------------|
| `z`            | `z`                       | `aexp` in `info_XXXXX.txt`                           |
| `t`            | `t_Gyr`                   | Cosmological age (Friedmann integration)             |
| `lookback`     | `lookback_Gyr`            | Lookback time                                        |
| `nstars`       | `nstars`                  | Line count of `stars_*.out*`                         |
| `sinkmass:N`   | `msink_N_Msol`            | Column 2 (`msink`) of `sink_XXXXX.csv` for sink id `N` |
| `smbhmass:N`   | `msmbh_N_Msol`            | Column 21 (`mbh`) of `sink_XXXXX.csv` for sink id `N`  |
| `accrate:N`    | `accrate_N_Msolyr`        | Column 13 (`acc_rate`) of `sink_XXXXX.csv` for sink id `N`, converted to Msol/yr |
| `rho:N`        | `rho_N_gcm3`              | Column 15 (`rho_gas`) of `sink_XXXXX.csv` for sink id `N`, converted to g/cm³ |

Sink-derived fields (`sinkmass`, `smbhmass`, `accrate`, `rho`) use the same code-unit conversion as [`sinkmass`](#sesh-sinkmass-sim_dir-id), reading `unit_l`, `unit_d`, and (for `accrate`) `unit_t` from `info_XXXXX.txt`:

```
Msol      = code_mass  * unit_d * unit_l^3 / 1.9885e33
Msol/yr   = code_rate  * unit_d * unit_l^3 / unit_t * 3.15576e7 / 1.9885e33
g/cm^3    = code_density * unit_d
```

#### Presets

| Preset         | Equivalent `--fields`                          | Use case                                  |
|----------------|-------------------------------------------------|--------------------------------------------|
| `z_stars`      | `z,nstars`                                       | Star formation history                     |
| `cosmo`        | `z,t,lookback`                                   | Cosmological time axis                     |
| `z_bh:N`       | `z,sinkmass:N,smbhmass:N`                        | BH mass vs. SMBH sub-component for sink `N` |
| `sink_evol:N`  | `z,t,sinkmass:N,accrate:N`                       | Mass and accretion-rate growth history for sink `N` |

```bash
sesh report /path/to/sim --preset sink_evol:3 --out sink3_history.csv
# output,z,t_Gyr,msink_3_Msol,accrate_3_Msolyr
# 1,9.0000,0.5370,1.2000e+02,3.1000e-04
# 2,7.4320,0.7120,4.5000e+02,8.7000e-04
# ...
```
