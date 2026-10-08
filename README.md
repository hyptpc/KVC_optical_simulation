KVC optical simulation
======================

Geant4 simulation of the optical photon response of the KVC
(quartz radiator read out by MPPCs).


## Requirements

- Geant4 (with UI/Vis drivers)
- ROOT
- CMake

Set up the Geant4 and ROOT environments before building, e.g.

```shell
source /path/to/root/bin/thisroot.sh
source /path/to/geant4/bin/geant4.sh
```


## How to build

```shell
git clone git@github.com:hyptpc/KVC_optical_simulation.git
cd KVC_optical_simulation
./build.sh
```

The executable is installed to `bin/KVCOpticalSim`.
To remove the build directory (`.build/`) and `bin/`, run

```shell
./clean.sh
```

After changing header files, a clean rebuild (`./clean.sh && ./build.sh`) is recommended.


## How to use

Arguments of ConfFile and OutputName are necessary.
G4Macro is an optional argument.

```shell
./bin/KVCOpticalSim [ConfFile] [OutputName] (G4Macro)
./bin/KVCOpticalSim conf/default.conf foo.root run.mac
```

- With a macro, the macro is executed in batch mode (e.g. `run.mac` runs `/run/beamOn 5000`).
- Without a macro, an interactive session starts and `vis.mac` (and `gui.mac` for GUI sessions)
  are executed. Run it from the top directory so that these macros are found.

```shell
./bin/KVCOpticalSim conf/default.conf foo.root
```


## Conf file

A conf file is a plain text file with one `key value` pair per line.
Lines starting with `#` can be used as comments.

```
particle         kaon-
momentum         0.735
input_beam_file  beam/beam_K_seg1_run00344.root
```

Example conf files are placed in `conf/`:

| File                          | Description                                         |
|-------------------------------|-----------------------------------------------------|
| `default.conf`                | Default setting (K beam from a ROOT beam file)      |
| `k_setup.conf`, `pi_setup.conf` | K / pi beam from the particle gun (no beam file)  |
| `pbar_seg*.conf`              | Anti-proton beam for each segment                   |

### Beam

| Key               | Description |
|-------------------|-------------|
| `particle`        | Geant4 particle name (e.g. `kaon-`, `pi-`, `anti_proton`) |
| `momentum`        | Beam momentum in GeV/c (used by the particle gun, i.e. when no beam file is given) |
| `input_beam_file` | ROOT beam file, or `none` to use the particle gun (see below) |
| `beam_y_offset`   | Offset of the beam position in y [mm] |
| `decay`           | `1`: decay physics on, `0`: decay physics removed (for all particles) |
| `seed`            | (optional) Fixed random seed. If not given, the seed is randomized |
| `production_cut`  | Production cut (range) in mm for all particles (default 0.1 mm, e- threshold ~135 keV in quartz) |

### Geometry

| Key                   | Description |
|-----------------------|-------------|
| `quartz_thickness`    | Thickness of the quartz radiator [mm] (must be larger than 6 mm) |
| `do_segmentize`       | `1`: single segment (26 mm wide), `0`: full width (104 mm) |
| `wrapper_thickness`   | Thickness of the wrapper [mm] |
| `air_layer_thickness` | Thickness of the air gap between the quartz and the wrapper [mm] |

### Optical properties

| Key | Description |
|-----|-------------|
| `quartz_finish` | Quartz surface finish, `0`: polished, `1`: ground |
| `Quartz_A_Alpha`, `Quartz_B_Alpha` | `sigma_alpha` of the quartz surface for quartz A (normal, polished; `quartz_finish 0`) and quartz B (frosted; `quartz_finish 1`), respectively |
| `quartz_specularSpike`, `quartz_specularLobe`, `quartz_backScatter`, `quartz_diffuseLobe` | Unified-model constants of the quartz surface |
| `quartz_boundary_reflectivity` | Reflectivity of the quartz surface (not applied if negative) |
| `quartz_abs_scale` | (optional) Scale factor for the quartz absorption length |
| `wrap_type` | Wrapper model, `0`: Teflon, `1`: specular wrapper (Mylar by default, Teflon/EJ-510 reflectivity with `is_teflon 1`/`is_paint 1`), `2`: EJ-510 (volume reflection), `3`: transmissive Teflon |
| `teflon_*` | Parameters of the Teflon wrapper (`sigma_alpha`, unified-model constants, `teflon_reflectivity_scale`) |
| `ej510_*` | Parameters of the EJ-510 wrapper (`sigma_alpha`, unified-model constants) |
| `qe_scale` | Scale factor for the MPPC photon detection efficiency |

Each surface takes its parameters in the same format,
`<material>_sigma_alpha` (for the quartz: `Quartz_A_Alpha` / `Quartz_B_Alpha`),
`<material>_specularSpike`, `<material>_specularLobe`, `<material>_backScatter` and `<material>_diffuseLobe`,
with `<material>` = `quartz`, `teflon` or `ej510`.
All of them are passed to Geant4 in the same way, but depending on the surface model
Geant4 ignores some of them:

| Surface | Geant4 model | Used | Ignored |
|---------|--------------|------|---------|
| Quartz, `quartz_finish 0` | `dielectric_dielectric`, `polished` | `quartz_boundary_reflectivity` | `Quartz_A_Alpha`, unified-model constants |
| Quartz, `quartz_finish 1` | `dielectric_dielectric`, `ground` | `Quartz_B_Alpha`, unified-model constants, `quartz_boundary_reflectivity` | - |
| `wrap_type 0` (Teflon) | `dielectric_dielectric`, `groundfrontpainted` | reflectivity (`teflon_reflectivity_scale`) | `teflon_sigma_alpha`, unified-model constants |
| `wrap_type 1` (Mylar) | `dielectric_metal`, `polished` | reflectivity (aluminized Mylar) | (no keys) |
| `wrap_type 1` with `is_teflon 1` / `is_paint 1` | `dielectric_metal`, `ground` | `teflon_*` / `ej510_*` sigma_alpha and unified-model constants, reflectivity | - |
| `wrap_type 2` (EJ-510, or Teflon with `is_teflon 1`) | `dielectric_dielectric`, `groundfrontpainted` | reflectivity | sigma_alpha, unified-model constants |
| `wrap_type 3` (transmissive Teflon) | `dielectric_dielectric`, `ground` | `teflon_sigma_alpha`, unified-model constants | (no reflectivity: Fresnel) |

Notes:
- A front-painted surface reflects with probability given by the reflectivity, always as Lambertian (diffuse) reflection.
- In the unified model the diffuse lobe is the remainder 1 - (spike + lobe + backscatter); `*_diffuseLobe` is not read by Geant4.
  If spike + lobe + backscatter + diffuse of a surface in use is not 1, a warning (`UnifiedConstantsSum`) is printed at start-up.
- The quartz surface (`quartz_finish`) is applied to the faces facing the air gap, i.e. only when `air_layer_thickness > 0`.
  The upper / lower end faces of the quartz (MPPC side) are always polished.

See `src/DetectorConstruction.cc` for the details of each wrapper model.

### Beam file

When `input_beam_file` is given, the primary particle is randomly sampled from the
TTree `tree` in the file, which has the following branches:

| Branch         | Description |
|----------------|-------------|
| `px, py, pz`   | Momentum [MeV/c] |
| `vx, vy, vz`   | Position [mm] (`vz` is relative to the upstream surface of the quartz) |

A relative path is resolved against the directory of the conf file,
e.g. `beam/xxx.root` in `conf/default.conf` refers to `conf/beam/xxx.root`.
An absolute path can also be used.
Beam files used in the example conf files are placed in `conf/beam/`.


## Output

The output ROOT file contains the TTree `tree` with one entry per event.
If several `/run/beamOn` are executed in one session, all events are stored in the same tree.
Main branches:

| Branch            | Description |
|-------------------|-------------|
| `evnum`           | Event number in the output file (continues over several `/run/beamOn`) |
| `event_id`        | Geant4 event ID (restarts from 0 at each `/run/beamOn`) |
| `npe`             | Number of detected photoelectrons |
| `n_cherenkov_gen` | Number of Cherenkov photons generated in the quartz (1.37-3.87 eV) |
| `n_delta_e`       | Number of delta electrons generated in the quartz |
| `beam_*`          | Energy, momentum and position of the primary particle |
| `nhit_mppc`       | Number of photons reaching the MPPCs (detected or not) |
| `nTrapped_Air`    | Number of photons generated in the quartz that are lost outside the quartz (absorbed or leaving the world), excluding photons reaching the MPPCs |
| `seg`, `detect_flag`, `wave_length`, `time`, `pos_*` | Information of each photon reaching the MPPCs (`detect_flag`: 1 if detected, 0 otherwise) |

See `src/AnaManager.cc` for the full list of branches.
