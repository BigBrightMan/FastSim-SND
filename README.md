# Fast Neutrino Flux Simulation for SND@LHC

This guide shows how to produce the forward neutrino flux at SND@LHC with the
**Fast Neutrino Flux Simulation** by F. Kling and M. Fieg.

<p align="center">
  <img src="workflow.png" alt="Workflow of the fast neutrino flux simulation" width="800"><br>
  <em><b>Figure 1.</b> Workflow of the simulation.</em>
</p>

| Program | What it does | Version in example | Source |
|-----|-----|-----|-----|
| CRMC | Generates pp collisions and saves them in a HepMC3 file | v2.2.1 | https://gitlab.iap.kit.edu/AirShowerPhysics/crmc |
| Rivet | Reads the HepMC3 file event by event and runs analysis modules on it | 4.1.0 | https://heprivet.org/releases/ |
| Fast Neutrino Flux Simulation | Rivet module that computes the neutrino flux | | https://github.com/KlingFelix/FastNeutrinoFluxSimulation |


---

# Part 1: Installation

Install in this order: **Rivet -> CRMC -> Fast Neutrino Flux Simulation**. CRMC
needs the HepMC3 library that comes with Rivet, and the Fast Neutrino Flux
Simulation is compiled with a Rivet tool.

> **Prerequisites:** if you work on lxplus, you can ignore the "prerequisites" sections in
> the instructions of every program. Compilers, CMake, Boost and Python are already installed. 

## 1.1 Rivet

Rivet is the standard toolkit for analysing Monte Carlo events in particle physics.
Install version 4.1.0 with the `rivet-bootstrap` script for that release, following
https://heprivet.org/releases/. The examples in this guide use `~/work/public/RIVET`
as the install folder.

Things to watch out for:
- The installation creates a file called `rivetenv.sh` in the install folder. It
  sets `PATH`, `LD_LIBRARY_PATH` and `PYTHONPATH` so that the shell can find
  `rivet`, `rivet-build`, HepMC3 and their libraries.
- Run `source <install folder>/rivetenv.sh` in every new terminal and in every
  HTCondor job before using Rivet or CRMC. Use `source`, not `./rivetenv.sh`,
  otherwise the settings are lost when the script ends. For csh/tcsh, use
  `rivetenv.csh`.

To check the installation:

```bash
source ~/work/public/RIVET/rivetenv.sh
rivet --version
```

The version should be 4.0.0 or newer.

## 1.2 CRMC

CRMC (Cosmic Ray Monte Carlo) is a program developed at KIT that gives access to
several hadronic interaction models.

Follow the `README.md` in https://gitlab.iap.kit.edu/AirShowerPhysics/crmc. The
examples in Part 2 expect CRMC to be installed in `~/work/public/CRMC/build/installed`:

```bash
source ~/work/public/RIVET/rivetenv.sh
mkdir -p ~/work/public/CRMC && cd ~/work/public/CRMC
git clone https://gitlab.iap.kit.edu/AirShowerPhysics/crmc.git
mkdir -p build/installed && cd build/installed
cmake ../../crmc -DCMAKE_INSTALL_PREFIX=$PWD -DCRMC_ENABLE_HepMC3=ON
make install -j8
```

Things to watch out for:
- Run `source ~/work/public/RIVET/rivetenv.sh` before `cmake`, so that CRMC finds
  HepMC3.
- Add `-DCRMC_ENABLE_HepMC3=ON` to `cmake`. Without it, CRMC cannot write the
  HepMC3 file that Rivet needs.
- Only EPOS LHC is compiled by default. To use another model, turn it on at the
  `cmake` step, for example `-DCRMC_DPMJET19=ON` or `-DCRMC_SIBYLL=ON`.

## 1.3 Fast Neutrino Flux Simulation

This is the main tool ([arXiv:2105.08270](https://arxiv.org/abs/2105.08270),
[arXiv:2402.13318](https://arxiv.org/abs/2402.13318)). It reads the pp collisions,
follows the long lived hadrons (π, K, hyperons) through the LHC beam pipe and
magnets, lets them and the charm hadrons decay into neutrinos, and keeps the
neutrinos that reach the detector plane at z = 480 m. Read the README of the
repository first: https://github.com/KlingFelix/FastNeutrinoFluxSimulation

The modules are in `RIVET_Modules/`:

| Folder | Setup |
|---|---|
| `Rivet_Run2_Histogram` | 13 TeV, histogram output (original version) |
| `Rivet_Run2_MCFiles` | 13 TeV, neutrino event output |
| `Rivet_Run3_MCFiles` | 13.6 TeV, Run 3 crossing angle, neutrino event output |
| `Rivet_Run4_MCFiles` | 14 TeV, Run 4, neutrino event output |

Each `MCFiles` folder has one module per neutrino source: `*_Neutrinos_Light`
(π, K and hyperon decays), `*_Neutrinos_Charm` (charm hadron decays) and
`*_Neutrinos_Charm_weighted` (charm, for weighted events such as POWHEG).

**Download**

```bash
cd ~/work/public/RIVET
git clone https://github.com/KlingFelix/FastNeutrinoFluxSimulation.git
cd FastNeutrinoFluxSimulation
```

**Update for Rivet 4.** The modules were written for Rivet 3. Change these two
lines in the `.cc` file, otherwise it does not compile:

```cpp
DEFAULT_RIVET_ANALYSIS_CTOR(X);  ->  RIVET_DEFAULT_ANALYSIS_CTOR(X);
DECLARE_RIVET_PLUGIN(X);         ->  RIVET_DECLARE_PLUGIN(X);
```

**Settings for SND@LHC.** The module compiles and runs with any values, so wrong
settings give a wrong flux without any error message. Check these values in the
`.cc` file before building:

| Setting | Meaning | Light: original -> used here | Charm: original -> used here |
|---|---|---|---|
| `_dim` | detector plane `{xmin, xmax, ymin, ymax}` in m | `{-1.5, 1.5, -0.5, 0.5}` -> `{-1.2, 1.2, -1.2, 1.2}` | `{-0.5, 0.5, -0.5, 0.5}` -> `{-1.2, 1.2, -1.2, 1.2}` |
| `_xing_urad_h` | horizontal beam crossing angle in µrad (positive = towards +x) | `0` -> `+250` | `0` -> `0` |
| `_xing_urad_v` | vertical beam crossing angle in µrad (positive = upwards) | `-160` -> `0` | `-160` -> `+160` |
| `nevent` | neutrinos written to the output file | 500000 -> 2000000 | 450000 -> 90000 |

The original values are the Run 3 setup for FASER from the upstream repository
(crossing angle 160 µrad downwards).

Coordinate convention in the module:
- z is along the beam line, from the ATLAS interaction point (IP1) towards the
  detector. The plane is at z = 480 m.
- x is horizontal and y is vertical, in m. (0, 0) is the beam axis without a
  crossing angle.
- The crossing angle rotates every hadron (position and momentum) before it is
  propagated, so the centre of the neutrino beam moves on the plane by about
  480 m × angle (for example 160 µrad moves it by about 7.7 cm).
- A neutrino is kept if its (x, y) at z = 480 m lies inside `_dim`. The plane of
  ±1.2 m used here is larger than the SND@LHC target, so the cut to the real
  detector acceptance has to be applied afterwards on `events_*.txt`.

> **To be confirmed:** the Light and Charm modules currently use different crossing
> angles (horizontal +250 µrad vs vertical +160 µrad). Check which one matches the
> LHC run you simulate, and use the same in both modules.

**Build** (example: the Run 3 light hadron module)

```bash
source ~/work/public/RIVET/rivetenv.sh
cd ~/work/public/RIVET/FastNeutrinoFluxSimulation/RIVET_Modules/Rivet_Run3_MCFiles/Run3_Neutrinos_Light
rivet-build RivetRun3_Neutrinos_Light.so Run3_Neutrinos_Light.cc
```

This creates `RivetRun3_Neutrinos_Light.so` in the module folder. Build other
modules the same way inside their own folders (the Charm module is needed for the
named pipe example in Part 2). Rebuild after every change to the `.cc` file.

---

# Part 2: Running

Run in this order: **CRMC -> Rivet**. CRMC writes the pp collisions to a `.hepmc`
file, and Rivet runs the Fast Neutrino Flux Simulation module on that file.

## 2.1 Generate pp collisions with CRMC

Example: 200k pp collisions at √s = 13.6 TeV with EPOS LHC

```bash
source ~/work/public/RIVET/rivetenv.sh
cd ~/work/public/CRMC/build/installed
./bin/crmc -I 2212 -P -6800 -i 2212 -p 6800 -m 0 -o hepmc3 -n 200000
```

| Option | Meaning |
|---|---|
| `-i 2212 -p 6800` | first beam: proton with momentum +6800 GeV |
| `-I 2212 -P -6800` | second beam: proton with momentum −6800 GeV (head on, √s = 13.6 TeV) |
| `-m` | model: `0` EPOS LHC (default), `6` SIBYLL, `8` PHOJET, `12` DPMJET. Only models turned on at the `cmake` step are available; `crmc --help` lists them |
| `-o hepmc3` | output format (Rivet needs **HepMC3**) |
| `-n` | number of events |
| `-s` | random seed (random if not given) |
| `-f` | output file name (automatic if not given) |

For large samples, split the run into HTCondor jobs. The example below sends 5 jobs
of 200k events (1M in total). Each job gets its own seed and output file from the
job number `$(Process)` = 0, 1, 2, 3, 4.

`job.sh`
```bash
#!/bin/bash
# $1 = job number, given by HTCondor
source $HOME/work/public/RIVET/rivetenv.sh
cd $HOME/work/public/CRMC/build/installed
./bin/crmc -I 2212 -P -6800 -i 2212 -p 6800 -m 0 -o hepmc3 -n 200000 \
    -s $((1000 + $1)) -f crmc_eposlhcr_job$1_p_p_6800.hepmc
```

`job.submit`
```
executable  = job.sh
arguments   = $(Process)
log         = xjob_$(Cluster)_$(Process).log
output      = xjob_$(Cluster)_$(Process).out
error       = xjob_$(Cluster)_$(Process).err
+JobFlavour = "testmatch"
queue 5
```

```bash
condor_submit job.submit
```

CRMC uses one CPU per job, so there is no need to ask for more.

**Expected output.** The log starts with a summary of the run settings:

```
          >> crmc <<

  seed:                       243531089 (automatic)
  projectile id:              2212 (p)
  projectile momentum:        6800
  target id:                  2212 (p)
  target momentum:            -6800

  number of collisions:       200000
  output file format:         HepMC3
  output file name:           .../crmc_eposlhcr_243531089_p_p_6800.hepmc
  HE model:                   0 (EPOS.LHC-R full (slow))

 ==[crmc]==> Collision number 10
 ==[crmc]==> Collision number 20
 ...
```

The output file is `crmc_eposlhcr_243531089_p_p_6800.hepmc`. CRMC names it
`crmc_<model>_<random seed>_p_p_<momentum>.hepmc`, so your file name will have a
different seed. This file is the input for step 2.2.

Inside, the file has one block per event: the event header (`E`), the cross section
(`A ... GenCrossSection`), the two incoming protons (`P 1`, `P 2`), and then every
produced particle (`P`: id, parent vertex, PDG code, px, py, pz, E, mass, status):

```
HepMC::Version 3.03.00
HepMC::Asciiv3-START_EVENT_LISTING
E 0 9 43
U GEV MM
A 0 GenCrossSection 7.74692917e+10 0.00000000e+00 -1 -1
P 1 0 2212 0.0000000000000000e+00 0.0000000000000000e+00 6.8000000000000000e+03 ...
P 2 0 2212 0.0000000000000000e+00 0.0000000000000000e+00 -6.8000000000000000e+03 ...
V -1 0 [1,2] @ ...
P 3 -1 2212 -6.3083600997924805e-01 -1.5025800466537476e-01 -6.7993442382812500e+03 ...
P 4 -1 2212 1.3884599506855011e-01 -8.8443398475646973e-02 3.9045229492187500e+03 ...
P 5 -1 111 1.8577499687671661e-01 -9.2968098819255829e-02 5.3535101318359375e+02 ...
...
```

The files are large (about 40 GB per 1M events), so keep them on EOS or in your
work area.

## 2.2 Run the Fast Neutrino Flux Simulation in Rivet

Example: run `Run3_Neutrinos_Light` on the file from step 2.1

`RIVET_ANALYSIS_PATH` tells Rivet which folder has the module `.so`. The module
writes `events_light.txt` and `Rivet.yoda` in the folder where `rivet` is started,
so start it in a separate output folder for every run. Otherwise the next run
overwrites the files.

```bash
source ~/work/public/RIVET/rivetenv.sh
export RIVET_ANALYSIS_PATH=~/work/public/RIVET/FastNeutrinoFluxSimulation/RIVET_Modules/Rivet_Run3_MCFiles/Run3_Neutrinos_Light
mkdir -p ~/work/public/FastSim_output/eposlhc_200k && cd ~/work/public/FastSim_output/eposlhc_200k
rivet ~/work/public/CRMC/build/installed/crmc_eposlhcr_243531089_p_p_6800.hepmc \
      -a Run3_Neutrinos_Light -o Rivet.yoda
```

On HTCondor, one job per `.hepmc` file, each writing to its own folder `run_0`,
`run_1`, ...:

`job.sh`
```bash
#!/bin/bash
# $1 = input .hepmc file, $2 = output folder
source $HOME/work/public/RIVET/rivetenv.sh
export RIVET_ANALYSIS_PATH=$HOME/work/public/RIVET/FastNeutrinoFluxSimulation/RIVET_Modules/Rivet_Run3_MCFiles/Run3_Neutrinos_Light
mkdir -p $2 && cd $2
rivet $1 -a Run3_Neutrinos_Light -o Rivet.yoda
```

`job.submit`
```
executable  = job.sh
arguments   = $(infile) /afs/cern.ch/work/<initial>/<username>/public/FastSim_output/run_$(Process)
log         = xjob_$(Cluster)_$(Process).log
output      = xjob_$(Cluster)_$(Process).out
error       = xjob_$(Cluster)_$(Process).err
+JobFlavour = "tomorrow"
queue infile matching files /afs/cern.ch/work/<initial>/<username>/public/CRMC/build/installed/crmc_eposlhcr_job*_p_p_6800.hepmc
```

Replace `<initial>/<username>` with your own path. Rivet also uses one CPU per job.

CRMC and Rivet can also run at the same time without saving the `.hepmc` file, by
using a named pipe. To run the Light and Charm modules together, put both module
folders in `RIVET_ANALYSIS_PATH`, separated by `:`. The pipe cannot be created on
AFS or EOS, so it goes in the local `/tmp` (or `$TMPDIR` in an HTCondor job).

```bash
source ~/work/public/RIVET/rivetenv.sh
M=~/work/public/RIVET/FastNeutrinoFluxSimulation/RIVET_Modules/Rivet_Run3_MCFiles
export RIVET_ANALYSIS_PATH=$M/Run3_Neutrinos_Light:$M/Run3_Neutrinos_Charm
mkdir -p ~/work/public/FastSim_output/fifo_run && cd ~/work/public/FastSim_output/fifo_run
FIFO=$(mktemp -u /tmp/crmc_XXXXXX.hepmc) && mkfifo $FIFO
~/work/public/CRMC/build/installed/bin/crmc -I 2212 -P -6800 -i 2212 -p 6800 -m 0 \
    -o hepmc3 -n 200000 -f $FIFO &
rivet $FIFO -a Run3_Neutrinos_Light -a Run3_Neutrinos_Charm -o Rivet.yoda
rm $FIFO
```

**Expected output.** The example below comes from a test run of the module as
configured above (`nevent = 2000000`) on a smaller EPOS LHC file with 10k events
(`crmc_eposlhcr_3217357_p_p_6800.hepmc`). It took about 5 minutes. With 200k
events, the numbers are larger but the output looks the same.

At the end of the run, Rivet prints:

```
Event 10000 (0:05:15 elapsed)

Finished event loop at 2026-10-28 08:51:12
Cross-section = 7.746929e+10 pb
light
-- found 11974111 weighted events
-- sample 2000000 unwighted events
-- total event weight 70.9606
-- maximal event weight 0.00148299
-- effective number of events 47849.9
-- max weight events VPID: -14
-- max weight events HPID: -321
-- max weight events E: 10.5052
-- maximum number of repeated events : 58
-- realized effective number events : 34482

Rivet run completed at 2026-10-28 08:51:23, time elapsed = 0:05:26
```

`sample ... unwighted events` is always equal to `nevent`. The neutrinos are drawn
from the weighted list, so the same neutrino can appear more than once.
`maximum number of repeated events` shows how often this happens; if it is large,
generate more pp collisions.

It also writes these files in the output folder:

- **`events_light.txt`** (`events_charm.txt` for the charm module): the list of
  neutrinos, one per line, with these columns:

  | Column | Content |
  |---|---|
  | 1 | neutrino PDG ID (12 = ν<sub>e</sub>, 14 = ν<sub>μ</sub>, 16 = ν<sub>τ</sub>, negative = anti ν) |
  | 2 | parent hadron PDG ID (e.g. 211 = π⁺, 321 = K⁺) |
  | 3, 4 | x, y position at z = 480 m [m] |
  | 5 | production z position [m] |
  | 6, 7 | x, y angle [rad] |
  | 8 | energy [GeV] |
  | 9 | event weight [pb] |

  The first lines of `events_light.txt` from the test run (the file starts with an
  empty line, has `nevent` lines after that, and is about 160 MB for 2M lines):

  ```
  -14  -211   -0.1948    0.2483 24.8400 -4.2289e-04 5.5168e-04 1.2957e+02 2.7486e+02
   14   211    0.2540    0.1192 14.5990 5.7059e-04 2.1518e-04 1.7028e+01 2.7486e+02
  -14  -321    1.0136    0.4413 11.0990 2.1267e-03 9.2695e-04 1.6191e+02 2.7486e+02
  -14  -211    0.5165    0.6490  8.2990 1.0803e-03 1.3516e-03 9.5522e+01 2.7486e+02
   14   321   -0.1095   -0.1374  8.1990 -2.5270e-04 -2.9108e-04 1.3869e+02 2.7486e+02
  ```

  All events are unweighted.

  This is the file to compare with the FLUKA neutrino flux.
- **`Rivet.yoda`**: the standard Rivet output. Here it only holds the event count
  and the cross section, because these modules write events instead of histograms.

---


