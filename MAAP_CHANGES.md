# Changes from upstream

Upstream: https://github.com/SmithB/ATL1415, branch `main` at commit 048b66f78bae353ca57f5e7a31a352ccf2fdc0a4 (2026-06-05), by Ben Smith with Suzanne Dickinson, Benjamin Jelley and Tyler Sutterley (see `CONTRIBUTORS.md`), MIT License ("Copyright (c) 2021 SmithB"). The newest upstream tag, v2.1.0 (b77926d, 2026-03-12), is two commits older; `pyproject.toml` still says version 2.0.0.

This copy starts from upstream's file tree at 048b66f without its git history. It is unchanged except for the items below. The solver settings, the default arguments, the region files, the masks and the LICENSE are untouched. Upstream has no `.github/` folder, so nothing was left out.

## Masks and Git LFS

The masks (`masks/**/*.tif`, `*.h5`, `*.db`) are Git LFS files, and this copy keeps the pointer files exactly as upstream committed them. A plain clone gets 130-byte pointers; the real files come from upstream's LFS store, for example `https://media.githubusercontent.com/media/SmithB/ATL1415/main/masks/RGI_reduced/06_rgi60_Iceland_reduced.db`.

`masks/EGM2008_geoid_h.nc` is a special case: the committed file is an LFS pointer (133 bytes, oid `sha256:1d421492...`, 85,694,896 bytes), but `.gitattributes` line 5 excludes that path from LFS (`-filter=lfs`), so `git lfs pull` leaves the pointer in place and the geoid has to be downloaded by hand from `https://media.githubusercontent.com/media/SmithB/ATL1415/main/masks/EGM2008_geoid_h.nc`. It is left as it is here.

## 2026-10-25: LSsurf and SMBcorr install over https

`pyproject.toml` required `LSsurf @ git+ssh://git@github.com/smithb/LSsurf.git` and `SMBcorr @ git+ssh://git@github.com/smithb/SMBcorr.git`. Cloning over SSH needs a key registered on GitHub, so for everyone else, and in any container build, `pip install .` stopped with `Permission denied (publickey)`. Both repositories are public, so they now install over https, as pointCollection already did.

Upstream named no commit for either, so each install took whatever the default branch held that day. They are now pinned to the commits these results were checked with, which are also the current heads of their default branches:

- LSsurf `04abfbb1030f39ad27d429f3aadda8e88943e10c` (2026-09-24)
- SMBcorr `9e99f7ba608e13dc20a9759593da8764324f6c02` (2024-02-29)

pointCollection is left unpinned as upstream had it (it was at `1c23042b91892522a3786b627767176f610f69a3`, 2026-10-18, for the check below). LSsurf in turn installs PySPQR (`sparseqr`) from its default branch. No processing changes.

## 2026-10-25: `decimate_data` works on numpy 2.4 and newer

numpy 2.4 removed `np.in1d`. `decimate_data` in `ATL1415/ATL11_to_ATL15.py` called it for every 5 km bin denser than the point target (1.2 million points per tile), so dense Greenland and Antarctic tiles stopped with `AttributeError` on current numpy; small tiles like the Iceland one below never reach that line. It now uses `np.isin`, which gives the same mask for these 1-D arrays. Results do not change.

## 2026-10-25: other numpy names removed in numpy 2

Same kind of fix in the rest of the package, each with its documented equivalent, so results do not change:

- `ATL1415/ATL11_to_ATL15.py`, `mask_components_by_time`: `np.NaN` to `np.nan` (removed in numpy 2.0; the function is not called on the tile path today).
- `ATL1415/assign_firn_variable.py`, the firn-mask lines: `np.NaN` to `np.nan`.
- `ATL1415/scripts/make_ATL1415_queue.py` and `ATL1415/scripts/make_GL_ATL1415_queue.py`, `pad_mask_canvas`: `np.in1d` to `np.isin`.
- `ATL1415/scripts/make_GL_ATL1415_queue.py`, the region-file reader: `np.float` to `float` (removed in numpy 1.24).

The superseded copies in `scripts/old/` are left alone. Not changed here because they live in the sibling repositories: `SMBcorr/assign_firn_variable.py` (the version the tile solve actually imports) still uses `np.NaN` on the firn-correction path, and pointCollection still uses `np.in1d` and `np.Inf` in a few helpers (`data.new_rows`, `tools/calc_bias_and_R.py`, `tools/along_track_coords.py`, `ATL11/crossover_data.py`, `glah12/campaign_bias_correction.py`), none of which the Iceland tile solve reaches.

## 2026-10-25: README

Install steps (the README had none).

## 2026-10-25: masks fetched from the original project's Git LFS store

The masks under `masks/` are Git LFS files (about 974 MB in all). This copy does not hold the LFS objects itself, so `.lfsconfig` points Git LFS at the original repository's LFS store (`https://github.com/SmithB/ATL1415.git/info/lfs`). A clone with Git LFS installed then downloads the masks from there, exactly as a clone of the original does. Without `.lfsconfig`, Git LFS would look for the objects in this copy and fail. No mask file changed.

## 2026-10-25: release 006 parameter file

`default_args/` shipped parameter files only up to `rel_003.txt`, whose time span (2018.75 to 2023.25) and settings belong to release 003. `default_args/rel_006_0331.txt` is copied unchanged from the author's `on_s3` branch (commit fa0afc5ce5b6abdda92967facca1fd42a299a0b5). These are the parameters of the ATL14/ATL15 release 006 products now at NSIDC (`-t=2018.75,2026.5`, `--reference_time=2020`, `--max_iterations=3`). The code on `main` ignores three of its keys: `--ATL11xo_version`, `--previous_product_top` (a path on the authors' cluster) and `--t_crop`. This is the parameter set the Check below used. No code changed.

## 2026-10-25: grid spacing in the release 006 parameter file

On `on_s3` the grid spacing of the release 006 products is not in `rel_006_0331.txt` but in a separate file, `default_args/quarterly.txt` (`-g=100,1000,0.25`: the 100 m DEM of ATL14 and the 1 km quarterly height change of ATL15), which was not copied with it. A region set up from `rel_006_0331.txt`, `north.txt` and `IS.txt` alone therefore fell back to the built-in default in `ATL11_to_ATL15.py` (`-g=250,4000,1`, a 4 km yearly grid). On that grid the 10 km averaging operator in LSsurf does not line up (10 km is two and a half 4 km cells, and every scale but the largest is centered half a cell off), and the Iceland tile stopped in `LSsurf/grid_functions.py` `sum_cell_area` with `ValueError: cannot reshape array of size 576 into shape (7,7,9)`. The Check below had passed only because its index file also set `-g`.

`rel_006_0331.txt` now carries the `-g=100,1000,0.25` line itself, as `rel_001.txt` to `rel_003.txt` already do, so it is complete on its own. Runs that already passed this grid spacing are unchanged.

## Check

On 2026-10-25, in a fresh conda environment with the packages in `environment.yml` (Python 3.12, conda-forge: numpy 2.5.4, scipy 1.18.1, GDAL 3.13.3, h5py 3.16.0, SuiteSparse), from a fresh clone of this repository: `pip install .` built ATL1415 and fetched LSsurf, SMBcorr, pointCollection and PySPQR over https with no SSH key (57 s). The default branch has no tests (`testpaths = ["test"]` names a folder that does not exist).

Then one 60 km tile, the Iceland smoke tile (E1260 N-2620, Myrdalsjokull), from the 16 ATL11 v007 granules that touch it (621 MB, NSIDC) and the Iceland RGI mask and EGM2008 geoid from the LFS store, with the release 006 parameters (`rel_006_0331.txt`, as in `default_args/` on upstream's `on_s3` branch; `local.txt` sets `--ATL14_root` and `--mask_dir`, and `index_007_0332.txt` sets `--ATL11_index` and, for this first check, `-g=100,1000,0.25`):

```sh
make_ATL11_index <root>/ATL14_processing 007_0332 <data>/ATL11 None
bash index_queue.txt; bash post_queue.txt
setup_ATL1415_region.py local.txt rel_006_0331.txt default_args/north.txt default_args/IS.txt index_007_0332.txt
ATL11_to_ATL15.py --xy0 1260000 -2620000 --prelim @ATL14_processing/rel006/north/IS/input_args_IS.txt
ATL11_to_ATL15.py --xy0 1260000 -2620000 --prelim @ATL14_processing/rel006/north/IS/input_args_IS.txt --calc_error_for_xy
```

The solve took 9 min 33 s on one core with 5.7 GiB peak memory, and the error step 2 min 57 s with 4.6 GiB. The output is the same as from the unchanged code with the https fix applied by hand: the same edit counts and sigma_hat (4.68), and the same comparison with the official ATL14_IS_0331_100m_006_01 and ATL15_IS_0331_3mo_1km_006_01 granules over the central 40 km of the tile, to every printed digit. ATL14 `h` against `z0` (60,826 nodes): median difference -0.105 m, r = 0.99924. ATL15 `delta_h` against `dz` from 2019.25 on (17,574 cell-epochs): median difference -0.015 m, RMS 0.669 m, r = 0.990.

On 2026-10-25, after the grid-spacing change, the same tile again from a fresh clone, this time in an environment solved from `environment.yml` with no version pins (Python 3.15.1, numpy 2.5.4, scipy 1.18.1), with an index file that sets only `--ATL11_index`. The composed `input_args_IS.txt` has the same keys and values as above, the solve took 9 min 41 s (5.8 GiB) and the error step 2 min 57 s (4.6 GiB), and the comparison with the official granules is identical to every printed digit. Before the change the same setup stopped after 11 s with the `sum_cell_area` error.
