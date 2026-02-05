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
- `sesh eventstats <outputs_root>`: per `output_XXXXX`, count event ids in `stars_*.out*`

Example output:

```
output_00001 SF:12 SN:3 B-SF:1 B-SN2:0
```
