# sesh

Lightweight Bash CLI toolkit for inspecting RAMSES simulation outputs.

## Usage

Make the executable available:

```bash
chmod +x bin/sesh
```

Run from the repo root:

```bash
bin/sesh z /path/to/output_00042
bin/sesh time /path/to/output_00042
bin/sesh eventstats /path/to/outputs_root
bin/sesh eventstats /path/to/outputs_root --from 12
bin/sesh simstatus
bin/sesh simstatus --path /path/to/simulations
bin/sesh sinkmass /path/to/sim 3
bin/sesh z all
bin/sesh time all
```

Optional PATH setup:

```bash
export PATH="$PWD/bin:$PATH"
```

## Subcommands

- `sesh z <output_dir>`: print redshift from `info_XXXXX.txt`
- `sesh time <output_dir>`: print redshift, time, and lookback time from `info_XXXXX.txt`
- `sesh eventstats <outputs_root> [--from N]`: per `output_XXXXX`, count event ids in `stars_*.out*`, starting at `output_N`
- `sesh simstatus [--path DIR]`: summarize latest output per simulation with z and last write time
- `sesh sinkmass <sim_dir> <id>`: print `msink` and `msmbh` in solar masses for a given sink ID, reading from the latest `sink_XXXXX.csv` and converting units via `unit_l`/`unit_d` from the info file

Example output:

```
output_00001 SF:12 SN:3 B-SF:1 B-SN2:0

BH 3  msink=2.1543e+09 Msol  msmbh=1.8732e+09 Msol  (output_00042)
```
