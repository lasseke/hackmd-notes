# Running CTSM cases on Betzy

## Resources

Tutorials for CTSM and regional cases:
- Short tutorial gist: https://github.com/ESCOMP/CTSM/pull/1892
- Example with detailed instructions: https://gist.github.com/negin513/d71268419f808b91a82cde5530d390b1
- `mesh_maker.py` example: https://gist.github.com/negin513/dde385a28145928786e2bc498e2b5137#creating-mesh-files-for-a-wrf-domain
- CTSM user guide: https://escomp.github.io/ctsm-docs/versions/master/html/

Data:
- CLM input data: https://svn-ccsm-inputdata.cgd.ucar.edu/trunk/inputdata/lnd/clm2/
- Storage on Betzy: `/cluster/shared/noresm/inputdata`


### Get and install CTSM

Clone and checkout CTSM with version compatible for running on Betzy:

```
git clone https://github.com/NorESMhub/CTSM.git
cd CTSM
git checkout -b ctsm5.3.11-noresm_v2
```

!OBS! Checkout externals with new git-fleximod functionality:

```
./bin/git-fleximod update
```

## Regional

### 2 Create input data for region of interest

#### 2.1 Create and load conda environment

Load conda module for python dependencies and create/load environment.
Important: install into project space to avoid running into file restriction errors.
For example, on betzy with current module versions:

```
module purge
module load Miniforge3/24.1.2-0
source ${EBROOTMINIFORGE3}/bin/activate
export CONDA_PKGS_DIRS=/cluster/projects/nn9774k/conda/lassetk/package-cache
export CONDA_ENV_SRC=/cluster/projects/nn9774k/conda/lassetk

echo "Source directory for conda envs stored in var CONDA_ENV_SRC"
```

Then, create the environment with required dependencies specified in CTSM environment file:

```
cd [CTSM_ROOT]/python
conda create -p $CONDA_ENV_SRC/ctsm-env
conda install -p /cluster/projects/nn9774k/conda/lassetk/ctsm-env --file conda_env_ctsm_py_latest.txt
conda activate /cluster/projects/nn9774k/conda/lassetk/ctsm-env
```

#### 2.2 Define paths to input data files in config file

First, we need to find out which input data files we should base our extracted inputs on, depending on the chosen CTSM version.
Most default .nc files are defined in (I think):

```
[CTSM_ROOT]/bld/namelist_files/namelist_defauls_ctsm.xml
```

Next, we need to create our own config file for pointing to the correct inputs on Betzy.
It must follow the same structure defined in `[CTSM_ROOT]/tools/site_and_regional/default_data_2000.cfg`.
Example below:

```
[main]
clmforcingindir = /cluster/shared/noresm/inputdata

[datm_gswp3]
dir = atm/datm7/atm_forcing.datm7.GSWP3.0.5d.v1.c170516
domain = domain.lnd.360x720_gswp3.0v1.c170606.nc
solardir = Solar
precdir = Precip
tpqwdir = TPHWL
solartag = clmforc.GSWP3.c2011.0.5x0.5.Solr.
prectag = clmforc.GSWP3.c2011.0.5x0.5.Prec.
tpqwtag = clmforc.GSWP3.c2011.0.5x0.5.TPQWL.
solarname = CLMGSWP3v1.Solar
precname = CLMGSWP3v1.Precip
tpqwname = CLMGSWP3v1.TPQW

[surfdat]
dir = lnd/clm2/surfdata_esmf/ctsm5.3.0
surfdat_16pft = surfdata_0.9x1.25_hist_2000_16pfts_c240908.nc
surfdat_78pft = surfdata_0.9x1.25_hist_2000_78pfts_c240908.nc
mesh_dir = share/meshes/
mesh_surf = fv0.9x1.25_141008_ESMFmesh.nc

[landuse]
dir = lnd/clm2/surfdata_esmf/ctsm5.3.0
landuse_16pft = landuse.timeseries_0.9x1.25_SSP2-4.5_1850-2100_78pfts_c240908.nc
landuse_78pft = landuse.timeseries_0.9x1.25_SSP2-4.5_1850-2100_78pfts_c240908.nc

[domain]
file = share/domains/domain.lnd.fv0.9x1.25_gx1v7.151020.nc
```

OBS! Make sure the specified files actually exist in the shared cluster space on Betzy! Otherwise check if they
can be downloaded from `https://svn-ccsm-inputdata.cgd.ucar.edu/trunk/inputdata/`.


#### 2.3 Run subset_data script

Now we can run the script to subset the data (Obs: cannot run subset_data.py directly because it is missing dependencies in system path).
Adapted from [this tutorial](https://gist.github.com/negin513/d71268419f808b91a82cde5530d390b1) , but note added: `--cfg-file` and `--create-domain` (omitting threw an error)!
OBS: `--create-datm` NOT implemented for regional cases!!! See `/python/ctsm/subset_data.py` and `/python/ctsm/site_and_regional/regional_case.py`.
Coordinates correspond to Svalbard, taken from https://latitude.to/articles-by-country/no/norway/111/svalbard (18-02-2025).

```
cd [CTSM_ROOT]/tools/site_and_regional

# Path to output dir
SUBSET_OUT_DIR=/cluster/shared/noresm/inputdata/regions_from_subset_data/svalbard

# For help, run ./subset_data --help
./subset_data region --create-surface --create-mesh --create-domain --create-user-mods \
--lat1 74 --lat2 81 --lon1 10 --lon2 35 --overwrite --verbose --outdir $SUBSET_OUT_DIR \
--cfg-file /cluster/home/lassetk/ctsm_regional_test/CTSM/tools/site_and_regional/data_2000_betzy.cfg
```

_IMPORTANT!_ Due to a weird error, you need to convert the resulting mesh file to CDF5 format as follows:

```
# Load netcdf command line tools
module --quiet purge
module load NCO/5.1.9-iomkl-2022a

nccopy -k 'cdf5' $SUBSET_OUT_DIR/domain.lnd.fv0.9x1.25_gx1v7_10.0-35.0_74.0-81.0_c250219_ESMF_UNSTRUCTURED_MESH.nc \
$SUBSET_OUT_DIR/domain.lnd.fv0.9x1.25_gx1v7_10.0-35.0_74.0-81.0_c250219_ESMF_UNSTRUCTURED_MESH_TEMP.nc

rm $SUBSET_OUT_DIR/domain.lnd.fv0.9x1.25_gx1v7_10.0-35.0_74.0-81.0_c250219_ESMF_UNSTRUCTURED_MESH.nc

mv $SUBSET_OUT_DIR/domain.lnd.fv0.9x1.25_gx1v7_10.0-35.0_74.0-81.0_c250219_ESMF_UNSTRUCTURED_MESH_TEMP.nc \
$SUBSET_OUT_DIR/domain.lnd.fv0.9x1.25_gx1v7_10.0-35.0_74.0-81.0_c250219_ESMF_UNSTRUCTURED_MESH.nc
```

To plot the created mesh file, do:
```
cd [CTSM_ROOT]/tools/site_and_regional
./mesh_plotter --input /cluster/shared/noresm/inputdata/regions_from_subset_data/svalbard/domain.lnd.fv0.9x1.25_gx1v7_10.0-35.0_74.0-81.0_c250219_ESMF_UNSTRUCTURED_MESH.nc \
--outdir /cluster/home/lassetk/ctsm_regional_test/plots --overwrite --verbose
```

#### 2.3 Create and build a case

Choose a suitable compset in `[CTSM_ROOT]/cime_config/config_compsets.xml`. Used `I2000Clm60Sp` (long: `2000_DATM%GSWP3v1_CLM60%SP_SICE_SOCN_MOSART_SGLC_SWAV`)for testing.
Then:

```
cd [CTSM_ROOT]/cime/scripts
# Define where to store case folder
CASE_ROOT=/cluster/home/lassetk/ctsm_regional_test/cases
CASE_NAME=SVAL_REG
./create_newcase --case $CASE_ROOT/$CASE_NAME --mach betzy --project nn9774k --res CLM_USRDAT --compset I2000Clm60Sp --run-unsupported --user-mods-dirs /cluster/shared/noresm/inputdata/regions_from_subset_data/svalbard/user_mods
```

Then, make desired changes to run options and build the case:

```
cd $CASE_ROOT/$CASE_NAME

# Change running options
./xmlchange STOP_OPTION=nyears
./xmlchange STOP_N=2
./xmlchange CLM_FORCE_COLDSTART=on

# CHANGE OUTPUT DIR? Default is work space on betzy (good)
# ./xmlchange CIME_OUTPUT_ROOT=/blab/blub/

# Change the comp. resources (nodes, cpu/tasks) to be requested for the simulation.
# ON BETZY: to run jobs in the 'normal' queue, we need to request at least 4 nodes! Each node can use a max of 128 tasks.
# See: https://www.sigma2.no/documentation//jobs/choosing_job_types.html and https://www.sigma2.no/documentation//jobs/job_types/betzy_job_types.html#job-type-betzy
# !!!BUT!!! SVALBARD DOMAIN IS TOO SMALL (ca. 70 active grid cells as per variable 'landfrac_pft' in surface data set (ncdump -h surfdata*.nc)).
# That's why we need to use a queue that can handle smaller jobs.
# Hence:
./xmlchange NTASKS=32

./xmlchange JOB_WALLCLOCK_TIME=05:59:00 --subgroup case.run
./xmlchange JOB_WALLCLOCK_TIME=00:59:00 --subgroup case.st_archive
# ./xmlchange JOB_WALLCLOCK_TIME=05:59

# Rather than (save for larger domains):
#./xmlchange NTASKS_ATM=128,NTASKS_LND=256,NTASKS_CPL=128
# IMPORTANT: change queue XML settings too:
#./xmlchange --subgroup case.run JOB_QUEUE=normal

./xmlquery -p QUEUE

#./xmlchange NTASKS=64
# USELESS ACCORDING TO MATVEY: ./xmlchange NTHRDS=4

# Set up case
./case.setup
./preview_run

# TO DISPLAY PARALLELISM LAYOUT:
# See: https://esmci.github.io/cime/versions/master/html/users_guide/pes-threads.html
./pelayout

# Build and submit
./case.build
./case.submit
```

TO BE CONTINUED!


