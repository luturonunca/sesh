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
sesh report <sim_dir> [--fields f1,f2,...|--preset name] [--out file] [--from N] [--source output|movie|both] [--ir-cloud N]
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

Write a per-snapshot CSV table to a file, with columns chosen by `--fields` or a named `--preset`. Snapshots are sorted by number (by `aexp` when `--source both` merges two independently numbered sequences); rows where a field can't be computed (missing file, missing unit, e.g. no sink yet) get `nan`, so the file loads directly with `numpy.loadtxt(file, delimiter=',')` — no `NA`/missing-value handling needed. Output is prefixed with a `#` header row.

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
| `--from N`       | Skip outputs before `output_N` (for `--source movie`, skip frames before frame `N`) |
| `--source output\|movie\|both` | Where to read snapshots from (default: `output`) — see below |
| `--ir-cloud N`   | Accretion radius in cells, for `mgas:N` (default: from `namelist.txt`, else `4`) |

#### Sink data sources

RAMSES writes sink state in two places, through the same `output_sink_csv` routine and in the same format:

| Source     | Files                                              | Cadence           |
|------------|-----------------------------------------------------|--------------------|
| `output`   | `output_XXXXX/info_XXXXX.txt` + `sink_XXXXX.csv`   | Output cadence     |
| `movie`    | `movie1/info_XXXXX.txt` + `sink_XXXXX.txt`         | Movie-frame cadence |

Movie frames are usually written far more often than outputs, so `--source movie` gives a much finer time sampling of sink evolution. Only `movie1/` is read: RAMSES guards the sink dump with `proj_ind==1`, so the other `movieN/` directories contain maps and info files but no sink files.

The first column changes with the source:

| `--source` | First column | Meaning                                                              |
|------------|--------------|------------------------------------------------------------------------|
| `output`   | `output`     | Output number                                                          |
| `movie`    | `frame`      | Movie frame number                                                     |
| `both`     | `index`      | Positive = output number, negative = movie frame number                |

The two counters are independent, so `both` negates frame numbers to keep the sequences distinguishable, and sorts all rows by `aexp` (falling back to `time` for non-cosmological runs) rather than by index — giving one merged, time-ordered table.

`nstars` is only available for `output` rows; movie frames have no `stars_*.out*` and report `nan`.

```bash
sesh report /path/to/sim --preset sink_evol:1 --source both --out sink1_fine.csv
# index,z,t_Gyr,msink_1_Msol,accrate_1_Msolyr
# -1,5.666667,1.0239,1.0003e+08,1.3421e-06
# 1,4.000000,1.5732,1.0003e+08,6.7103e-07
# -2,3.000000,2.1915,2.0006e+08,1.3421e-06
# ...
```

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
| `mgas:N`       | `mgas_N_Msol`             | Gas mass inside the accretion radius of sink `N` (see below)  |
| `ffcold:N`     | `ffcold_N_Msolyr`         | Column 31 (`dMdc_cold`) of `sink_XXXXX.csv`, freefall cold-gas channel, converted to Msol/yr |
| `ffhot:N`      | `ffhot_N_Msolyr`          | Column 32 (`dMdc_hot`) of `sink_XXXXX.csv`, freefall hot-Bondi channel, converted to Msol/yr |
| `eddrat:N`     | `eddrat_N`                | `acc_rate / msink`, in units of the Eddington ratio (dimensionless) |
| `torque:N`     | `torque_N_Msolyr`         | Column 23 (`dMtorque`) of `sink_XXXXX.csv`, `bondi_torque_blend` rate, converted to Msol/yr |
| `torque2:N`    | `torque2_N_Msolyr`        | Column 24 (`dMtorque2`) of `sink_XXXXX.csv`, `bondi_torque_twoch` cold-gas torque channel, converted to Msol/yr |
| `bondi2:N`     | `bondi2_N_Msolyr`         | Column 25 (`dMbondi2`) of `sink_XXXXX.csv`, `bondi_torque_twoch` hot-gas Bondi channel, converted to Msol/yr |
| `torquerot:N`  | `torquerot_N_Msolyr`      | Column 26 (`dMtorque_rot`) of `sink_XXXXX.csv`, rotation-only torque diagnostic (not applied to the accretion rate), converted to Msol/yr |
| `bondinorot:N` | `bondinorot_N_Msolyr`     | Column 27 (`dMbondi_norot`) of `sink_XXXXX.csv`, non-rotating Bondi diagnostic (not applied to the accretion rate), converted to Msol/yr |
| `starcloud:N`  | `starcloud_N_Msol`        | Column 28 (`M_star_cloud`) of `sink_XXXXX.csv`, star particle mass in the accretion-zone aperture, converted to Msol |
| `stardisc:N`   | `stardisc_N_Msol`         | Column 29 (`M_star_disc_rot`) of `sink_XXXXX.csv`, rotationally-supported star particle mass, converted to Msol |
| `torquestar:N` | `torquestar_N_Msolyr`     | Column 30 (`dMtorque_star`) of `sink_XXXXX.csv`, AA17 torque rate using physical star-particle masses, converted to Msol/yr |

Sink-derived fields (`sinkmass`, `smbhmass`, `accrate`, `rho`, `ffcold`, `ffhot`, `torque`, `torque2`, `bondi2`, `torquerot`, `bondinorot`, `starcloud`, `stardisc`, `torquestar`) use the same code-unit conversion as [`sinkmass`](#sesh-sinkmass-sim_dir-id), reading `unit_l`, `unit_d`, and (for the `_Msolyr` rate fields) `unit_t` from `info_XXXXX.txt`:

```
Msol      = code_mass  * unit_d * unit_l^3 / 1.9885e33
Msol/yr   = code_rate  * unit_d * unit_l^3 / unit_t * 3.15576e7 / 1.9885e33
g/cm^3    = code_density * unit_d
```

#### `mgas:N` — gas mass inside the accretion radius

Useful because the torque and free-fall accretion channels are both proportional to the enclosed gas mass. RAMSES computes that mass exactly (`rho_gas * volume_gas`, printed as `Mgas(Msol)` under `verbose_AGN`) but writes only `rho_gas` to the sink file, so `mgas:N` reconstructs it from the cloud volume:

```
dx_min  = boxlen / 2**levelmax / aexp          # code units, from info_XXXXX.txt
V_cloud = (4/3) * pi * (ir_cloud * dx_min)**3
Msol    = rho_gas * V_cloud * unit_d * unit_l**3 / 1.9885e33
```

`ir_cloud` is the accretion radius in units of the finest cell. It's a run constant, read from the first `output_*/namelist.txt` found (movie frame dirs have no namelist copy of their own) and falling back to the RAMSES default of `4`. Override it with `--ir-cloud N`.

This is an approximation: `volume_gas` is the *kernel-weighted* sink-sphere volume, not the naive sphere, so the result carries a fixed O(1) offset. That factor is constant in physical units — the cloud radius `ir_cloud*dx_min` is constant by construction — so it cancels in ratios and leaves the *shape* of `M_gas(t)` correct. Calibrate it once against a `verbose_AGN` stdout line if you need absolute masses.

```bash
sesh report /path/to/sim --fields z,t,mgas:1,accrate:1 --source movie --out mgas1.csv
```

#### Presets

| Preset         | Equivalent `--fields`                          | Use case                                  |
|----------------|-------------------------------------------------|--------------------------------------------|
| `z_stars`      | `z,nstars`                                       | Star formation history                     |
| `cosmo`        | `z,t,lookback`                                   | Cosmological time axis                     |
| `z_bh:N`       | `z,sinkmass:N,smbhmass:N`                        | BH mass vs. SMBH sub-component for sink `N` |
| `sink_evol:N`  | `z,t,sinkmass:N,accrate:N`                       | Mass and accretion-rate growth history for sink `N` |
| `torque_evol:N`| `z,t,torque2:N,bondi2:N,starcloud:N,stardisc:N`  | Torque/Bondi channel split and disc star mass for sink `N` (`bondi_torque_twoch`) |

```bash
sesh report /path/to/sim --preset sink_evol:3 --out sink3_history.csv
# output,z,t_Gyr,msink_3_Msol,accrate_3_Msolyr
# 1,9.0000,0.5370,1.2000e+02,3.1000e-04
# 2,7.4320,0.7120,4.5000e+02,8.7000e-04
# ...
```
