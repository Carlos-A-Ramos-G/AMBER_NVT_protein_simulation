# MM MD NVT protein simulation (stages 00 → 04)

A self-contained protocol for running classical MD NVT production simulations of
proteins with AMBER. Edit `config.yaml`, run `python run_upto_NVT.py setup`, and
submit — all input files and job scripts are generated automatically.

Generated scripts are **resume-safe**: each stage checks whether its output files
already exist and skips itself if so, making it safe to re-run after a partial or
interrupted execution without redoing completed work.

Two execution modes are supported:

- **cluster** — generates `run_gpu` (SLURM master script) and one NVT chunk job
  script per SLURM job (`04_NVT/run_NVT_N.cmd`). Chunk jobs chain automatically
  via `sbatch` so only the first needs to be submitted.
- **local** — generates `run_local` (master script) and one NVT chunk script per
  chunk (`04_NVT/run_NVT_N_local.sh`). Scripts chain automatically via `bash`.

## Requirements

- AMBER (>= 18, tested with 24):
  - **cluster**: must be available as a `module` (`amber.module` in `config.yaml`).
  - **local**: either on PATH, or set `amber_home` to your AMBER install directory.
- Python >= 3.8 with PyYAML (`pip install pyyaml`).
- A PDB file plus any non-standard residue parameter files (`*.lib`, `*.frcmod`)
  placed in `00_prep/`. These are only required for the first run; if
  `structure.parm7` (or `structure_HMR.parm7`) and `structure.rst7` already exist
  in `00_prep/`, `setup` skips the tleap input checks.

## Quick start — cluster (SLURM)

1. Drop your PDB and parameter files (`.lib`, `.frcmod`) into `00_prep/`.
2. Edit `config.yaml`: set `pdb`, `job_name`, `execution_mode: cluster`,
   `amber.module`, `slurm.master.account`, `slurm.nvt.account`, walltimes,
   and list your parameter files under `leap.lib_files` and `leap.frcmod_files`.
3. Generate all input and job scripts:

   ```bash
   python run_upto_NVT.py setup
   ```

4. Submit:

   ```bash
   sbatch run_gpu
   ```

   Or generate and submit in one step:

   ```bash
   python run_upto_NVT.py setup --submit
   ```

## Quick start — local GPU workstation

1. Drop your PDB and parameter files (`.lib`, `.frcmod`) into `00_prep/`.
2. Edit `config.yaml`: set `execution_mode: local`. If AMBER is not on your PATH,
   also set `amber_home: /path/to/amber24`.
3. Generate and run:

   ```bash
   python run_upto_NVT.py setup
   bash run_local
   ```

   Or generate and launch immediately in the background:

   ```bash
   python run_upto_NVT.py setup --submit
   ```

## Commands

```
python run_upto_NVT.py setup  [--config config.yaml] [--mode cluster|local] [--submit]
python run_upto_NVT.py submit [--config config.yaml] [--mode cluster|local]
```

- `setup` generates all input files and run scripts. The mode defaults to
  `execution_mode` in `config.yaml`; pass `--mode` to override.
- `submit` launches existing scripts without regenerating files — useful after
  manually editing a generated script.
- `--submit` on the `setup` command generates files and immediately launches.

## What `setup` writes

| File | Description |
|------|-------------|
| `00_prep/leap_structure` | tleap input built from the `leap:` section of `config.yaml` |
| `00_prep/HMR.ccptraj` | cpptraj input for hydrogen mass repartitioning (only if `use_hmr: true`) |
| `01_min/min.in` | Minimization input |
| `02_heat/heat.in` | Heating input (200 ps ramp, 1 K → target temperature) |
| `03_equil/equil_1.in` … `equil_6.in` | Equilibration inputs (see Simulation stages below) |
| `04_NVT/prod.in` | Production NVT input |
| `run_gpu` *(cluster)* | Master SLURM script: runs stages 00–03, then submits `run_NVT_1.cmd` |
| `04_NVT/run_NVT_N.cmd` *(cluster)* | One SLURM script per chunk-job; each submits the next when done |
| `run_local` *(local)* | Master bash script: runs stages 00–03, then launches `run_NVT_1_local.sh` |
| `04_NVT/run_NVT_N_local.sh` *(local)* | One bash script per chunk-job; each launches the next when done |

Stage directories (`01_min/`, `02_heat/`, `03_equil/`, `04_NVT/`) are created
automatically by `setup` if they do not exist.

All templates are embedded in `run_upto_NVT.py` — no external template files
are needed.

## Simulation stages

### 00_prep — topology and coordinates

`tleap` reads `leap_structure` (generated from `config.yaml`) to build
`structure.parm7` and `structure.rst7`. If `use_hmr: true`, `cpptraj` then reads
`HMR.ccptraj` to produce `structure_HMR.parm7` with hydrogen masses repartitioned.

**Auto-skip:** if both the topology and `structure.rst7` already exist, this entire
stage is skipped at runtime.

### 01_min — energy minimization

Minimization runs in a convergence-checked outer loop (up to `min.max_cycles_cap`
repetitions of `min.maxcyc` steps). After each cycle the RMS gradient is read from
the output file; the loop stops as soon as it falls below `min.convergence_threshold`
(default 3 × 10⁻³ kcal/mol/Å). If the threshold is never reached, the loop exits
after the cap and a warning is printed.

**Auto-skip:** if any `structure_min_N.rst7` (N ≥ 1) already exists, the loop is
skipped and the highest-numbered restart file is used for the next stage.

### 02_heat — heating

A single 200 ps `pmemd.cuda` run ramps the temperature from 1 K to the target
(NMR-style `TEMP0` ramp). The ramp ends at 80% of the total heating steps; the
remaining 20% hold at the target temperature. Backbone heavy atoms are restrained
at 20 kcal/mol/Å² throughout.

**Auto-skip:** if `structure_heat.rst7` already exists, this stage is skipped.

### 03_equil — equilibration

Six sequential cycles:

| Cycle | Ensemble | Duration | Backbone restraint |
|-------|----------|----------|--------------------|
| 1 | NPT | `equil.npt_ns` | 15 kcal/mol/Å² |
| 2 | NPT | `equil.npt_ns` | 12 kcal/mol/Å² |
| 3 | NPT | `equil.npt_ns` | 9 kcal/mol/Å² |
| 4 | NPT | `equil.npt_ns` | 6 kcal/mol/Å² |
| 5 | NPT | `equil.npt_ns` | 3 kcal/mol/Å² |
| 6 | NVT | `equil.nvt_ns` | none |

**Auto-skip:** each cycle is skipped individually if its `structure_equil_N.rst7`
already exists, so a run interrupted mid-equilibration resumes at the failed cycle.

### 04_NVT — production

NVT production runs in chunks of `ns_per_chunk` ns. Each chunk is a separate job
script. `chunks_per_job` controls how many chunks are batched into a single SLURM
job (or a single local script invocation):

| Queue type | `chunks_per_job` | `slurm.nvt.time` |
|------------|-----------------|------------------|
| 1-day queue | `1` | `1-00:00:00` |
| 5-day queue | `total_chunks` | `5-00:00:00` |

With `chunks_per_job: 1` (the default), each SLURM job runs one chunk and then
submits the next via `sbatch`, so only the first job needs to be submitted manually
(done automatically by `run_gpu`).

**Auto-skip:** each chunk is skipped if its `structure_NVT_N.rst7` already exists.

## NMR restraints

Protocol-wide NMR restraints (DISANG format) can be applied to all stages —
minimization, heating, equilibration, and production — by setting two keys in
`config.yaml`:

```yaml
restraints:
  enabled: true
  file: 00_prep/my_restraints.rst   # path relative to the project root
```

When enabled, `setup` injects `nmropt = 1` into every `&cntrl` block and appends
a `DISANG = ../path/to/file` line to each input file. The path is automatically
prefixed with `../` since all stage directories sit one level below the project
root.

The restraint file must exist before running `setup`; validation will exit with an
error if it is missing.

## HMR

If `use_hmr: true`:

- `setup` writes `00_prep/HMR.ccptraj`; stage 00 runs `cpptraj -i HMR.ccptraj`
  after `tleap`, producing `structure_HMR.parm7`.
- All `pmemd.cuda -p` calls reference `structure_HMR.parm7`.
- All timesteps switch to `dt = 0.004` ps; `nstlim` values scale accordingly to
  preserve wall time per stage.

If `use_hmr: false`, a 2 fs timestep with `structure.parm7` is used.

## Configuration reference (`config.yaml`)

```yaml
pdb: MY_PROTEIN.pdb       # PDB file inside 00_prep/
use_hmr: true             # true -> 4 fs timestep, structure_HMR.parm7
temperature: 300.0        # K
job_name: MY_SIM

ns_per_chunk: 100         # ns per production chunk
total_chunks: 5           # number of chunks (100 ns × 5 = 500 ns total)
chunks_per_job: 1         # chunks per SLURM job (1 for 1-day queues)

execution_mode: cluster   # cluster or local (sets the default for --mode)

leap:
  forcefields:            # source commands in order
    - leaprc.protein.ff14SB
    - leaprc.water.tip3p
    - leaprc.gaff
  lib_files:              # .lib files in 00_prep/ (loadoff), one per entry
    - MY_LIG.lib
  frcmod_files:           # .frcmod files in 00_prep/ (loadamberparams), one per entry
    - MY_LIG.frcmod
  ions:                   # addions commands, one per entry
    - "Na+ 0"
    - "Cl- 0"
  box_type: TIP3PBOX
  box_size: 12            # minimum distance from solute to box edge (Å)

amber:
  module: apps/amber/24   # cluster only: module name loaded by 'module load'

amber_home: ""            # local only: AMBER install dir (sources amber.sh)
                          # leave blank if pmemd.cuda is already on PATH

slurm:
  master:                 # SLURM settings for the master job (stages 00-03)
    time: "1-00:00:00"
    ntasks: 1
    gres: gpu:1
    partition: gpu
    account: MY_ACCOUNT
  nvt:                    # SLURM settings for each NVT chunk job
    time: "1-00:00:00"
    ntasks: 1
    gres: gpu:1
    partition: gpu
    account: MY_ACCOUNT

min:
  maxcyc: 30000           # max steps per minimization cycle
  ncyc: 500               # steepest-descent steps before switching to CG
  cut: 10.0               # non-bonded cutoff (Å)
  max_cycles_cap: 100     # max outer loop iterations
  convergence_threshold: 3.0e-3   # RMS gradient target (kcal/mol/Å)

heat:
  ps: 200                 # total heating duration (ps)

equil:
  npt_ns: 1.25            # duration of each restrained NPT cycle (cycles 1-5)
  nvt_ns: 5.0             # duration of unrestrained NVT cycle (cycle 6)

prod:
  ntpr: 10000             # energy log frequency (steps)
  ntwx: 50000             # trajectory write frequency (steps)
  ntwr: 10000             # restart write frequency (steps)

restraints:
  enabled: false          # true to apply NMR restraints throughout all stages
  file: ""                # path to DISANG file, relative to project root
                          # e.g. 00_prep/my_restraints.rst
```

## Error handling

All generated bash scripts run with `set -euo pipefail`. Any failed `pmemd.cuda`
call (bad GPU, bad input, out of memory) immediately halts the job rather than
continuing and producing empty output files.
