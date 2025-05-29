# COAWST Installation Guide for Slurm Cluster

## Overview
COAWST (Coupled Ocean-Atmosphere-Wave-Sediment Transport) is a modeling system that couples:
- **ROMS** (Regional Ocean Modeling System)
- **SWAN** (Simulating WAves Nearshore) 
- **WRF** (Weather Research and Forecasting)

## Prerequisites

### 1. Verify WRF Installation
```bash
# Ensure WRF is already installed (from previous guide)
module load wrf/4.5.2
which wrf.exe
```

### 2. Create COAWST Directory Structure
```bash
# Create base directory on shared storage
sudo mkdir -p /shared/COAWST
sudo chown -R slurm:slurm /shared/COAWST

# Switch to slurm user
sudo su - slurm
cd /shared/COAWST

# Create directory structure
mkdir -p {Downloads,Libraries,COAWST-3.7,Tools,Projects,Data}
```

### 3. Set Environment Variables
```bash
# Create COAWST environment file
cat > /shared/COAWST/coawst_env.sh << 'EOF'
#!/bin/bash

# COAWST Installation paths
export COAWST_DIR=/shared/COAWST
export COAWST_ROOT=$COAWST_DIR/COAWST-3.7

# Use existing WRF libraries
export WRF_DIR=/shared/WRF
export NETCDF=$WRF_DIR/Libraries/netcdf
export HDF5=$WRF_DIR/Libraries/hdf5
export PNETCDF=$WRF_DIR/Libraries/pnetcdf

# COAWST specific libraries
export MCT_LIBDIR=$COAWST_DIR/Libraries/MCT/lib
export MCT_INCDIR=$COAWST_DIR/Libraries/MCT/include
export ESMF_DIR=$COAWST_DIR/Libraries/esmf
export ESMF_LIBDIR=$ESMF_DIR/lib
export ESMF_INCDIR=$ESMF_DIR/include

# Compiler settings
export CC=mpicc
export CXX=mpicxx
export FC=mpif90
export F77=mpif77

# MPI settings
export MPICC=mpicc
export MPICXX=mpicxx
export MPIFC=mpif90
export MPIF77=mpif77
export MPIF90=mpif90

# Library paths
export LD_LIBRARY_PATH=$MCT_LIBDIR:$ESMF_LIBDIR:$NETCDF/lib:$HDF5/lib:$PNETCDF/lib:$LD_LIBRARY_PATH
export PATH=$NETCDF/bin:$HDF5/bin:$PNETCDF/bin:$ESMF_DIR/bin:$PATH

# COAWST coupling flags
export USE_MPI=on
export USE_MPIF90=on
export USE_OpenMP=
export USE_LARGE=on
export USE_NETCDF4=on

# ROMS-specific settings
export USE_ROMS=on
export USE_SWAN=on
export USE_WRF=on

# Set ROMS application (can be changed per project)
export ROMS_APPLICATION=INLET_TEST

# Optimization flags
export CPPFLAGS="-I$HDF5/include -I$NETCDF/include -I$MCT_INCDIR -I$ESMF_INCDIR"
export LDFLAGS="-L$HDF5/lib -L$NETCDF/lib -L$MCT_LIBDIR -L$ESMF_LIBDIR"
EOF

# Source the environment
source /shared/COAWST/coawst_env.sh
```

## Additional Library Installation

### 1. Install MCT (Model Coupling Toolkit)
```bash
cd /shared/COAWST/Downloads

# Download MCT
wget https://github.com/MCSclimate/MCT/archive/refs/tags/MCT_2.11.0.tar.gz
tar -xzf MCT_2.11.0.tar.gz
cd MCT-MCT_2.11.0

# Configure and build MCT
./configure --prefix=/shared/COAWST/Libraries/MCT \
    --enable-mpiserial \
    --with-blas \
    --with-lapack \
    CC=mpicc \
    FC=mpif90

make -j$(nproc)
make install
cd ..
```

### 2. Install ESMF (Earth System Modeling Framework)
```bash
# Download ESMF
wget https://github.com/esmf-org/esmf/archive/refs/tags/v8.6.0.tar.gz
tar -xzf v8.6.0.tar.gz
cd esmf-8.6.0

# Set ESMF environment variables
export ESMF_DIR=$(pwd)
export ESMF_INSTALL_PREFIX=/shared/COAWST/Libraries/esmf
export ESMF_COMM=mpiuni
export ESMF_COMPILER=gfortran
export ESMF_NETCDF=standard
export ESMF_NETCDF_INCLUDE=$NETCDF/include
export ESMF_NETCDF_LIBPATH=$NETCDF/lib
export ESMF_PNETCDF=standard
export ESMF_PNETCDF_INCLUDE=$PNETCDF/include
export ESMF_PNETCDF_LIBPATH=$PNETCDF/lib

# Build and install ESMF
make -j$(nproc)
make install
cd ..
```

### 3. Install Additional Dependencies
```bash
# Install SCRIP (Spherical Coordinate Remapping and Interpolation Package)
cd /shared/COAWST/Downloads
wget https://github.com/SCRIP-Project/SCRIP/archive/refs/heads/master.zip
unzip master.zip
cd SCRIP-master

# Build SCRIP
make
cp scrip /shared/COAWST/Tools/
cd ..
```

## COAWST Download and Installation

### 1. Download COAWST
```bash
cd /shared/COAWST/Downloads

# Download COAWST from USGS
# Note: You may need to register at https://woodshole.er.usgs.gov/operations/modeling/COAWST/
wget https://woodshole.er.usgs.gov/operations/modeling/COAWST/COAWST_v3.7.tar.gz

# Alternative: Clone from GitHub (if available)
# git clone https://github.com/jcwarner-usgs/COAWST.git

tar -xzf COAWST_v3.7.tar.gz
mv coawst /shared/COAWST/COAWST-3.7
cd /shared/COAWST/COAWST-3.7
```

### 2. Configure COAWST Build System
```bash
# Source environment
source /shared/COAWST/coawst_env.sh

# Copy and modify build script
cp Lib/ARPACK/makefile.inc.LINUX Lib/ARPACK/makefile.inc

# Edit makefile.inc for your system
cat > Lib/ARPACK/makefile.inc << 'EOF'
home = /shared/COAWST/COAWST-3.7/Lib/ARPACK
PLAT = LINUX
FC = mpif90
FFLAGS = -O2
MAKE = /usr/bin/make
SHELL = /bin/sh
CD = cd
COPY = cp
REMOVE = rm -f
RANLIB = ranlib
BLASLIB = 
LAPACKLIB = 
DIRS = $(home)/BLAS $(home)/LAPACK $(home)/UTIL $(home)/SRC
EOF

# Build ARPACK
cd Lib/ARPACK
make lib
cd ../..
```

### 3. Configure Compiler Settings
```bash
# Create custom compiler configuration
cat > Compilers/Linux-gfortran.mk << 'EOF'
# Include file for GNU Fortran compiler on Linux

ifdef USE_MPI
 FC := mpif90
else
 FC := gfortran
endif
ifdef USE_OpenMP
 FC := $(FC) -fopenmp
endif

ifdef USE_DEBUG
 FFLAGS += -g -fbounds-check -fbacktrace -finit-real=nan -ffpe-trap=invalid,zero,overflow
else
 FFLAGS += -O2
endif

FFLAGS += -fdefault-real-8 -fdefault-double-8
FFLAGS += -Waliasing -Wampersand -Wline-truncation -Wsurprising -Wno-tabs -Wunderflow
FFLAGS += -fconvert=big-endian -frecord-marker=4

ifdef USE_NETCDF4
 FFLAGS += -DHDF5
 LIBS += -lnetcdff -lnetcdf -lhdf5_hl -lhdf5 -lz
else
 LIBS += -lnetcdff -lnetcdf
endif

ifdef USE_PARALLEL_IO
 FFLAGS += -DPARALLEL_IO
 LIBS += -lpnetcdf
endif

ifdef USE_MCT
 FFLAGS += -DMCT_COUPLING
 LIBS += -lmct -lmpeu
endif

ifdef USE_ESMF
 FFLAGS += -DESMF_LIB
 LIBS += -lesmf
endif

# WRF coupling
ifdef USE_WRF
 FFLAGS += -DWRF_COUPLING
 WRF_LIB_DIR = /shared/WRF/WRF-4.5.2/main
 LIBS += $(WRF_LIB_DIR)/libwrflib.a
endif

CC := mpicc
CFLAGS := -O2

CXX := mpicxx
CXXFLAGS := -O2

# Library and Include paths
NETCDF_INCDIR ?= $(NETCDF)/include
NETCDF_LIBDIR ?= $(NETCDF)/lib
HDF5_INCDIR ?= $(HDF5)/include  
HDF5_LIBDIR ?= $(HDF5)/lib
MCT_INCDIR ?= $(MCT_INCDIR)
MCT_LIBDIR ?= $(MCT_LIBDIR)

LIBS := -L$(NETCDF_LIBDIR) -L$(HDF5_LIBDIR) -L$(MCT_LIBDIR) $(LIBS)
FFLAGS += -I$(NETCDF_INCDIR) -I$(MCT_INCDIR)
EOF
```

### 4. Create Build Script
```bash
cat > /shared/COAWST/build_coawst.sh << 'EOF'
#!/bin/bash

# Source environment
source /shared/COAWST/coawst_env.sh

cd $COAWST_ROOT

# Clean previous builds
make clean

# Set build parameters
export MY_ROOT_DIR=$COAWST_ROOT
export MY_PROJECT_DIR=$COAWST_ROOT/Projects

# Choose your application (modify as needed)
export MY_ROMS_SRC=$MY_ROOT_DIR/ROMS
export MY_CASE=INLET_TEST

# Set coupling options
export USE_ROMS=on
export USE_SWAN=on  
export USE_WRF=on
export USE_MCT=on

# Set other options
export USE_MPI=on
export USE_MPIF90=on
export USE_NETCDF4=on
export USE_PARALLEL_IO=on
export USE_MY_LIBS=on

# Build the model
make

echo "COAWST build completed. Check for coawstM executable."
EOF

chmod +x /shared/COAWST/build_coawst.sh
```

## Configure Test Cases

### 1. INLET_TEST Configuration
```bash
# Copy test case
cd /shared/COAWST/COAWST-3.7
mkdir -p Projects/Inlet_test
cp -r WRF/test/em_real/* Projects/Inlet_test/
cp ROMS/External/roms_inlet_test.in Projects/Inlet_test/
cp SWAN/INPUT/INPUT Projects/Inlet_test/swan_inlet_test.in

# Create coupled namelist
cat > Projects/Inlet_test/namelist.input << 'EOF'
&time_control
 run_days                 = 0,
 run_hours                = 6,
 run_minutes              = 0,
 run_seconds              = 0,
 start_year               = 2008,
 start_month              = 09,
 start_day                = 14,
 start_hour               = 12,
 start_minute             = 00,
 start_second             = 00,
 end_year                 = 2008,
 end_month                = 09,
 end_day                  = 14,
 end_hour                 = 18,
 end_minute               = 00,
 end_second               = 00,
 interval_seconds         = 21600,
 input_from_file          = .true.,
 history_interval         = 60,
 frames_per_outfile       = 1,
 restart                  = .false.,
 restart_interval         = 5000,
 io_form_history          = 2,
 io_form_restart          = 2,
 io_form_input            = 2,
 io_form_boundary         = 2,
 debug_level              = 0,
/

&domains
 time_step                = 10,
 time_step_fract_num      = 0,
 time_step_fract_den      = 1,
 max_dom                  = 1,
 e_we                     = 42,
 e_sn                     = 43,
 e_vert                   = 35,
 p_top_requested          = 5000,
 num_metgrid_levels       = 27,
 num_metgrid_soil_levels  = 4,
 dx                       = 4000,
 dy                       = 4000,
 grid_id                  = 1,
 parent_id                = 0,
 i_parent_start           = 1,
 j_parent_start           = 1,
 parent_grid_ratio        = 1,
 parent_time_step_ratio   = 1,
 feedback                 = 1,
 smooth_option            = 0,
/

&physics
 mp_physics               = 4,
 ra_lw_physics            = 1,
 ra_sw_physics            = 2,
 radt                     = 30,
 sf_sfclay_physics        = 2,
 sf_surface_physics       = 2,
 bl_pbl_physics           = 2,
 bldt                     = 0,
 cu_physics               = 0,
 cudt                     = 5,
 isfflx                   = 1,
 ifsnow                   = 0,
 icloud                   = 1,
 surface_input_source     = 1,
 num_soil_layers          = 5,
 maxiens                  = 1,
 maxens                   = 3,
 maxens2                  = 3,
 maxens3                  = 16,
 ensdim                   = 144,
/

&fdda
/

&dynamics
 w_damping                = 0,
 diff_opt                 = 1,
 km_opt                   = 4,
 diff_6th_opt             = 0,
 diff_6th_factor          = 0.12,
 base_temp                = 290.,
 damp_opt                 = 0,
 zdamp                    = 5000.,
 dampcoef                 = 0.01,
 khdif                    = 0,
 kvdif                    = 0,
 non_hydrostatic          = .true.,
 moist_adv_opt            = 1,
 scalar_adv_opt           = 1,
/

&bdy_control
 spec_bdy_width           = 5,
 spec_zone                = 1,
 relax_zone               = 4,
 specified                = .true.,
 nested                   = .false.,
/

&grib2
/

&namelist_quilt
 nio_tasks_per_group = 0,
 nio_groups = 1,
/
EOF
```

### 2. Create Coupling Configuration
```bash
cat > Projects/Inlet_test/coupling.in << 'EOF'
!
! Coupling configuration file
!
! Model coupling time step (seconds)
!
      dt_coupling:  60.0d0
!
! Coupling debug level
!
      debug_coupling: 0
!
! WRF-ROMS coupling parameters
!
      wrf2roms_fluxes: T
      roms2wrf_sst: T
!
! ROMS-SWAN coupling parameters  
!
      roms2swan_fields: T
      swan2roms_fields: T
!
! Export/Import field names and units
!
      export_wrf_fields: 'Tair', 'Pair', 'Qair', 'Uwind', 'Vwind', 'swrad', 'lwrad', 'rain'
      import_roms_fields: 'SST'
      export_roms_fields: 'SST', 'SSH', 'Ubar', 'Vbar'
      import_swan_fields: 'Hsig', 'Tper', 'Pdir'
EOF
```

## Build COAWST

### 1. Execute Build
```bash
# Source environment
source /shared/COAWST/coawst_env.sh

# Run build script
cd /shared/COAWST
./build_coawst.sh

# Check if build was successful
ls -la COAWST-3.7/coawstM
```

### 2. Create Module File
```bash
sudo mkdir -p /shared/modulefiles/coawst

cat > /shared/modulefiles/coawst/3.7 << 'EOF'
#%Module1.0
##
## COAWST 3.7 module
##
proc ModulesHelp { } {
    puts stderr "COAWST 3.7 - Coupled Ocean-Atmosphere-Wave-Sediment Transport"
    puts stderr "Includes ROMS, SWAN, and WRF coupling capabilities"
}

module-whatis "COAWST 3.7 - Coupled modeling system"

conflict coawst

set coawst_root "/shared/COAWST"
set coawst_version "3.7"

# Load prerequisite modules
module load wrf/4.5.2

# Set COAWST paths
setenv COAWST_DIR $coawst_root
setenv COAWST_ROOT $coawst_root/COAWST-$coawst_version

# Set coupling library paths
setenv MCT_LIBDIR $coawst_root/Libraries/MCT/lib
setenv MCT_INCDIR $coawst_root/Libraries/MCT/include
setenv ESMF_DIR $coawst_root/Libraries/esmf
setenv ESMF_LIBDIR $coawst_root/Libraries/esmf/lib
setenv ESMF_INCDIR $coawst_root/Libraries/esmf/include

# Add to PATH
prepend-path PATH $coawst_root/COAWST-$coawst_version
prepend-path PATH $coawst_root/Tools

# Add to LD_LIBRARY_PATH
prepend-path LD_LIBRARY_PATH $coawst_root/Libraries/MCT/lib
prepend-path LD_LIBRARY_PATH $coawst_root/Libraries/esmf/lib

# Set COAWST environment variables
setenv USE_MPI on
setenv USE_MPIF90 on
setenv USE_LARGE on
setenv USE_NETCDF4 on
setenv USE_ROMS on
setenv USE_SWAN on
setenv USE_WRF on
EOF
```

## Job Templates

### 1. COAWST Coupled Job Template
```bash
cat > /shared/COAWST/coawst_job_template.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=coawst_coupled
#SBATCH --output=coawst_%j.out
#SBATCH --error=coawst_%j.err
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=4
#SBATCH --time=04:00:00
#SBATCH --partition=compute

# Load COAWST module
module load coawst/3.7

# Change to run directory
cd $SLURM_SUBMIT_DIR

# Set environment
export OMP_NUM_THREADS=1

# Run COAWST
echo "Starting COAWST coupled run at $(date)"
mpirun coawstM coupling.in
echo "COAWST run completed at $(date)"
EOF
```

### 2. ROMS-only Job Template
```bash
cat > /shared/COAWST/roms_job_template.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=roms_only
#SBATCH --output=roms_%j.out
#SBATCH --error=roms_%j.err
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=8
#SBATCH --time=02:00:00
#SBATCH --partition=compute

# Load COAWST module
module load coawst/3.7

# Change to run directory
cd $SLURM_SUBMIT_DIR

# Set ROMS-only mode
export USE_ROMS=on
export USE_SWAN=off
export USE_WRF=off

# Run ROMS
echo "Starting ROMS at $(date)"
mpirun coawstM roms.in
echo "ROMS completed at $(date)"
EOF
```

### 3. SWAN-only Job Template
```bash
cat > /shared/COAWST/swan_job_template.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=swan_only
#SBATCH --output=swan_%j.out
#SBATCH --error=swan_%j.err
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=4
#SBATCH --time=01:00:00
#SBATCH --partition=compute

# Load COAWST module
module load coawst/3.7

# Change to run directory
cd $SLURM_SUBMIT_DIR

# Set SWAN-only mode
export USE_ROMS=off
export USE_SWAN=on
export USE_WRF=off

# Run SWAN
echo "Starting SWAN at $(date)"
mpirun coawstM swan.in
echo "SWAN completed at $(date)"
EOF
```

## Testing and Verification

### 1. Basic Test
```bash
# Load module
module load coawst/3.7

# Check executable
which coawstM
ldd $COAWST_ROOT/coawstM

# Test with sample case
cd /shared/COAWST/COAWST-3.7/Projects/Inlet_test
sbatch /shared/COAWST/coawst_job_template.sh
```

### 2. Model Component Tests
```bash
# Test ROMS component
cd /shared/COAWST/COAWST-3.7/ROMS/External
# Edit roms_inlet_test.in for your system
sbatch /shared/COAWST/roms_job_template.sh

# Test SWAN component  
cd /shared/COAWST/COAWST-3.7/SWAN/INPUT
# Edit INPUT file for your domain
sbatch /shared/COAWST/swan_job_template.sh
```

### 3. Performance Testing
```bash
cat > /shared/COAWST/performance_test.sh << 'EOF'
#!/bin/bash

# Test different configurations
for nodes in 2 4 8; do
    for ppn in 4 8; do
        total_cores=$((nodes * ppn))
        echo "Testing COAWST with $nodes nodes, $ppn tasks per node (total: $total_cores cores)"
        
        sbatch --nodes=$nodes --ntasks-per-node=$ppn \
               --job-name=coawst_perf_${total_cores} \
               --output=coawst_perf_${total_cores}_%j.out \
               --time=01:00:00 \
               --wrap="module load coawst/3.7; cd /shared/COAWST/test_run; mpirun coawstM coupling.in"
        
        sleep 10
    done
done
EOF

chmod +x /shared/COAWST/performance_test.sh
```

## Data and Grid Setup

### 1. Download Sample Data
```bash
mkdir -p /shared/COAWST/Data/{ROMS,SWAN,WRF,Coupling}

# ROMS grid and forcing data (example URLs - adjust as needed)
cd /shared/COAWST/Data/ROMS
# wget ftp://ftp.example.com/roms_grid.nc
# wget ftp://ftp.example.com/roms_forcing.nc

# SWAN grid and boundary data
cd /shared/COAWST/Data/SWAN
# wget ftp://ftp.example.com/swan_grid.grd
# wget ftp://ftp.example.com/swan_boundary.bnd

# WRF data (use existing WPS setup)
cd /shared/COAWST/Data/WRF
ln -s /shared/WRF/WPS_GEOG .
```

### 2. Create Grid Generation Scripts
```bash
cat > /shared/COAWST/Tools/create_grids.py << 'EOF'
#!/usr/bin/env python3
"""
COAWST Grid Generation Tool
Creates consistent grids for ROMS, SWAN, and WRF
"""

import numpy as np
import netCDF4 as nc
import matplotlib.pyplot as plt

def create_roms_grid(nx, ny, Lm, Mm, dx, dy):
    """Create a simple ROMS grid"""
    
    # Grid dimensions
    L = Lm + 1  # RHO-points in xi-direction
    M = Mm + 1  # RHO-points in eta-direction
    
    # Create coordinate arrays
    x_rho = np.arange(L) * dx
    y_rho = np.arange(M) * dy
    
    # Create 2D coordinate matrices
    x_rho_2d, y_rho_2d = np.meshgrid(x_rho, y_rho)
    
    # Create grid file
    with nc.Dataset('roms_grid.nc', 'w') as grid:
        # Dimensions
        grid.createDimension('xi_rho', L)
        grid.createDimension('eta_rho', M)
        grid.createDimension('xi_u', L-1)
        grid.createDimension('eta_u', M)
        grid.createDimension('xi_v', L)
        grid.createDimension('eta_v', M-1)
        
        # Variables
        x_var = grid.createVariable('x_rho', 'f8', ('eta_rho', 'xi_rho'))
        y_var = grid.createVariable('y_rho', 'f8', ('eta_rho', 'xi_rho'))
        
        x_var[:] = x_rho_2d
        y_var[:] = y_rho_2d
        
        # Attributes
        grid.title = 'ROMS Grid for COAWST'
        grid.type = 'ROMS grid file'
    
    print(f"Created ROMS grid: {L}x{M} points")

def create_swan_grid(nx, ny, dx, dy):
    """Create a SWAN computational grid"""
    
    with open('swan_grid.grd', 'w') as f:
        f.write('$ SWAN computational grid\n')
        f.write(f'$ Grid size: {nx} x {ny}\n')
        f.write(f'$ Grid spacing: {dx} x {dy} m\n')
        f.write('CGRID REGULAR 0.0 0.0 0.0 {} {} {} {}\n'.format(
            nx*dx, ny*dy, nx-1, ny-1))
    
    print(f"Created SWAN grid: {nx}x{ny} points")

if __name__ == "__main__":
    # Grid parameters (adjust as needed)
    nx, ny = 50, 40
    dx, dy = 2000, 2000  # 2 km resolution
    
    create_roms_grid(nx, ny, nx-2, ny-2, dx, dy)
    create_swan_grid(nx, ny, dx, dy)
    
    print("Grid generation completed")
EOF

chmod +x /shared/COAWST/Tools/create_grids.py
```
