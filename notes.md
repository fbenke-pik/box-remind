# Putting REMIND in a box

Depending on target audience (i.e. REMIND devs, *workshop participants*), target compute environment (Desktop/laptop, AWS, HPC) and target ~~OS-level virtualization~~ container software (`docker`, `apptainer`) project structure may vary wildly. For the time being we'll focus on

## Best practices workshop user

Build a Docker image for REMIND workshops, expand to combinded REMIND/MAgPIE container

- Audience: Workshop participants
  - Needs to be able to enter and execute basic commandline tools (`ssh`, `git`, `Rscript`)
  - Has a laptop with basic user right at their disposal
  - Run pre-configured scenarios in a dedicated software environment 
- Compute env: AWS or similar
  - Run on AWS to avoid Docker install needed on participant machines
- Container: Docker
  - Bake-in `R` dependencies, clone REMIND during checkout. No 30min wait for package installation

### Show stoppers

- REMIND doesn't commit renv.lock to the repo (by design - they use renv::hydrate() not renv::restore())
- Build succeeds
  - all packages installed in /opt/remind/renv/library/ during image build
- Runtime fails 
  - REMIND's `submit.R` creates a per-run renv in output/testOneRegi/ via `renv::snapshot()` rather than `renv::restore()`
  - Lockfile contains GitHub SSH URLs (RemoteUrl: git@github.com:pik-piam/GDPuc) alongside repo URLs 
  - `renv::restore()` prioritizes GitHub SSH over repos leading to authentication fails
  - Core issue: REMIND's per-run renv isolation (lines 93-102 in submit.R) is incompatible with our "bake packages into image" approach, because the snapshot creates a lockfile with remote metadata that triggers SSH cloning.

### Basic structure

```bash
┌─────────────────────────────────────────────────────┐
│ Docker Container (Ubuntu 24.04)                     │
│                                                     │
│  ┌────────────────────────────────────────────────┐ │
│  │ /opt/remind/ (REMIND cloned)                   │ │
│  │                                                │ │
│  │  ┌──────────────────────────────────────────┐  │ │
│  │  │ Global renv                              │  │ │
│  │  │ - .Rprofile (activates renv)             │  │ │
│  │  │ - renv/activate.R                        │  │ │
│  │  │ - renv/library/ (all packages installed) │  │ │
│  │  │ - renv.lock (MISSING - not in repo)      │  │ │
│  │  └──────────────────────────────────────────┘  │ │
│  │                                                │ │
│  │  ┌──────────────────────────────────────────┐  │ │
│  │  │ Per-Run renv (output/testOneRegi/)       │  │ │
│  │  │ - Created by scripts/start/submit.R      │  │ │
│  │  │ - renv.lock (from snapshot of global)    │  │ │
│  │  │   → Has "RemoteUrl: git@github.com:..."  │  │ │
│  │  │ - renv::restore() tries GitHub SSH       │  │ │
│  │  └──────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### Example participant workflow

```bash
# Participants SSH in:
ssh workshop@your-aws-instance
# Run pre-started container:
docker exec -it remind-workshop bash
# Start run
Rscript start.R --testOneRegi
# Observe output
cd output/<example run>
```

## Best practices container dev

???

# Miscellaneous

## Project repo & workflow

- [`box-remind`](https://gitlab.pik-potsdam.de/tonnru/box-remind.git)
- [`REMIND` model branch `workshop2025`]()

## Usefull docker commands

### Build image based on specific `Dockerfile` & redirect log

```PowerShell
docker build -f Dockerfile.workshop -t remind-baked .
```
- run this command in the folder with the `Dockerfile.workshop` file and the donwloaded gams installer `linux_x64_64_sfx.exe` 
- if you want to use the current [cluster licence]((https://gitlab.pik-potsdam.de/rse/rsewiki/-/wikis/Installing-GAMS#license)), download gams 51.4.0 

### Check for images

```PowerShell
docker images remind-baked
```

### Start a new container from the image

```PowerShell
# general
docker run --name <mycontainer> -it -v ${PWD}/gamslice.txt:/opt/gams/latest/gamslice.txt remind-baked bash

# example
docker run --name falk -it -v ./gamslice.txt:/opt/gams/latest/gamslice.txt remind-baked bash
```

If one would prefer to mount the REMIND dir *outside of the container*, mount that as well

```PowerShell
docker run --rm -it \
  -v ${PWD}/remind:/opt/remind \ # Mount the goddamn project dir so CHANGES PERSIST
  -v ${PWD}/logs:/var/log/remind \ # ..maybe also for logs..
  -v ${PWD}/gamslice.txt:/opt/gams/latest/gamslice.txt \ # Provide license files?
  remind-baked bash
```

### Pause and resume working on an image

```PowerShell
# list all running and stopped containers
docker ps -a 

# start a stopped container
docker start <mycontainer>

# continue working in container
docker exec -it <mycontainer> bash
```


### Copy stuff from container to host when no bridge

```PowerShell
docker ps  # Get running container ID
docker cp <container-id>:/var/log/remind/00.log ./00.log
```

### Save changes

```PowerShell
# Get container ID
docker ps -a
# Commit changes
docker commit <container-id> remind-baked:fixed
```

### Running REMIND
- make sure `make update-renv` works
- make sure `Rscript scripts/utils/checkSetup.R` yields no warnings
- make sure `Rscript start.R --gamscompile` throws no errors
- run oneRegi together with transport `Rscript start.R -i`, then choose config `./config/tests/scenario_config_oneRegiPlus.csv` and run `testOneRegiTransport`
- check run results by hand (as we have no `rs`): `modelstats::promptAndRun("./output/*")`

### Making old workshop branch run
- restore renv from ws lockfile will fail, some must be corrected manually 
- in output folder, run `renv::install('GDPuc@1.5.3', 'mrcommons@1.65.1','mrfaocore@1.4.3','mrlandcore@1.6.3','nleqslv@3.3.5','lmomco@2.5.3','RcppArmadillo@15.0.2-2','lmom@3.2','mrdownscale@0.46.0','mrlandcore@1.6.3', 'gdx2@0.3.3')`
- afterwards, `renv::restore()` should work



### Open questions
- is it necessary to run `./gamsinst` as described [in these instructions](https://www.gams.com/50/docs/UG_UNIX_INSTALL.html)? (probably not, as the solvers are also set directly in REMIND)




