# WRF, WRF-CHEM, and WRF-SOLAR Installation Guide

## Overview
This guide installs WRF v4.5.2, WRF-CHEM, and WRF-SOLAR on the Slurm cluster with shared storage.

## Prerequisites (Master Node - Install Once)

### 1. Create Installation Directory Structure
```bash
# Create base directory on shared storage
sudo mkdir -p /shared/WRF
sudo chown -R slurm:slurm /shared/WRF

# Switch to slurm user for installation
sudo su - slurm
cd /shared/WRF

# Create directory structure
mkdir -p {Downloads,Libraries,WRF-4.5.2,WPS-4.5,WRF-CHEM,WRF-SOLAR}
```

### 2. Set Environment Variables
```bash
# Create environment file
cat > /shared/WRF/wrf_env.sh << 'EOF'
#!/bin/bash

# WRF Installation paths
export WRF_DIR=/shared/WRF
export WRF_ROOT=$WRF_DIR/WRF-4.5.2
export WPS_ROOT=$WRF_DIR/WPS-4.5

# Library paths
export NETCDF=$WRF_DIR/Libraries/netcdf
export HDF5=$WRF_DIR/Libraries/hdf5
export PNETCDF=$WRF_DIR/Libraries/pnetcdf
export JASPER=$WRF_DIR/Libraries/jasper
export LIBPNG=$WRF_DIR/Libraries/libpng
export ZLIB=$WRF_DIR/Libraries/zlib

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

# Library paths for linking
export LD_LIBRARY_PATH=$NETCDF/lib:$HDF5/lib:$PNETCDF/lib:$JASPER/lib:$LIBPNG/lib:$ZLIB/lib:$LD_LIBRARY_PATH
export PATH=$NETCDF/bin:$HDF5/bin:$PNETCDF/bin:$PATH

# WRF specific variables
export WRFIO_NCD_LARGE_FILE_SUPPORT=1
export WRF_EM_CORE=1
export WRF_NMM_CORE=0
export WRF_CHEM=1
export WRF_KPP=1

# For WRF-SOLAR
export WRF_SOLAR=1

# Optimization flags
export CPPFLAGS="-I$HDF5/include -I$NETCDF/include"
export LDFLAGS="-L$HDF5/lib -L$NETCDF/lib -L$PNETCDF/lib"
EOF

# Source the environment
source /shared/WRF/wrf_env.sh
```

### 3. Install System Dependencies
```bash
# Install additional packages needed for WRF
sudo dnf install -y gcc-gfortran gcc-c++ \
    tcsh time wget which zlib-devel \
    libpng-devel jasper-devel \
    ksh m4 grads ncl-devel \
    python3-devel python3-numpy \
    curl-devel libxml2-devel \
    flex-devel bison byacc \
    libjpeg-turbo-devel
```

## Library Installation (Master Node)

### 1. Install ZLIB
```bash
cd /shared/WRF/Downloads
wget https://zlib.net/zlib-1.3.1.tar.gz
tar -xzf zlib-1.3.1.tar.gz
cd zlib-1.3.1

./configure --prefix=$ZLIB
make -j$(nproc)
make install
cd ..
```

### 2. Install libpng
```bash
wget http://prdownloads.sourceforge.net/libpng/libpng-1.6.39.tar.xz
tar -xf libpng-1.6.39.tar.xz
cd libpng-1.6.39

./configure --prefix=$LIBPNG
make -j$(nproc)
make install
cd ..
```

### 3. Install JasPer
```bash
wget http://www.ece.uvic.ca/~frodo/jasper/software/jasper-1.900.1.zip
unzip jasper-1.900.1.zip
cd jasper-1.900.1

./configure --prefix=$JASPER
make -j$(nproc)
make install
cd ..
```

### 4. Install HDF5
```bash
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.14/hdf5-1.14.3/src/hdf5-1.14.3.tar.gz
tar -xzf hdf5-1.14.3.tar.gz
cd hdf5-1.14.3

./configure --prefix=$HDF5 \
    --enable-fortran \
    --enable-hl \
    --enable-shared \
    CC=mpicc \
    FC=mpif90

make -j$(nproc)
make install
cd ..
```

### 5. Install Parallel NetCDF
```bash
wget https://parallel-netcdf.github.io/Release/pnetcdf-1.12.3.tar.gz
tar -xzf pnetcdf-1.12.3.tar.gz
cd pnetcdf-1.12.3

./configure --prefix=$PNETCDF \
    CC=mpicc \
    CXX=mpicxx \
    FC=mpif90 \
    F77=mpif77

make -j$(nproc)
make install
cd ..
```

### 6. Install NetCDF-C
```bash
wget https://downloads.unidata.ucar.edu/netcdf-c/4.9.2/netcdf-c-4.9.2.tar.gz
tar -xzf netcdf-c-4.9.2.tar.gz
cd netcdf-c-4.9.2

./configure --prefix=$NETCDF \
    --enable-netcdf-4 \
    --enable-shared \
    --enable-pnetcdf \
    --disable-dap \
    CC=mpicc \
    CPPFLAGS="-I$HDF5/include -I$PNETCDF/include" \
    LDFLAGS="-L$HDF5/lib -L$PNETCDF/lib"

make -j$(nproc)
make install
cd ..
```

### 7. Install NetCDF-Fortran
```bash
wget https://downloads.unidata.ucar.edu/netcdf-fortran/4.6.1/netcdf-fortran-4.6.1.tar.gz
tar -xzf netcdf-fortran-4.6.1.tar.gz
cd netcdf-fortran-4.6.1

./configure --prefix=$NETCDF \
    --enable-shared \
    CC=mpicc \
    FC=mpif90 \
    F77=mpif77 \
    CPPFLAGS="-I$NETCDF/include -I$HDF5/include" \
    LDFLAGS="-L$NETCDF/lib -L$HDF5/lib"

make -j$(nproc)
make install
cd ..
```

## WRF Installation

### 1. Download and Extract WRF
```bash
cd /shared/WRF/Downloads
wget https://github.com/wrf-model/WRF/releases/download/v4.5.2/v4.5.2.tar.gz
tar -xzf v4.5.2.tar.gz
mv WRF-4.5.2 /shared/WRF/
cd /shared/WRF/WRF-4.5.2
```

### 2. Configure WRF
```bash
# Source environment first
source /shared/WRF/wrf_env.sh

# Clean previous builds
./clean -a

# Configure WRF
./configure

# Select option 34 for dmpar (distributed memory parallel) with GNU compilers
# Select option 1 for basic nesting

# Edit configure.wrf if needed
# The configuration should automatically detect the libraries
```

### 3. Compile WRF
```bash
# Compile em_real (for real-data cases)
./compile em_real >& compile.log

# Check if compilation was successful
ls -la main/
# You should see: ndown.exe, real.exe, tc.exe, wrf.exe
```

## WPS Installation

### 1. Download and Extract WPS
```bash
cd /shared/WRF/Downloads
wget https://github.com/wrf-model/WPS/archive/v4.5.tar.gz
tar -xzf v4.5.tar.gz
mv WPS-4.5 /shared/WRF/
cd /shared/WRF/WPS-4.5
```

### 2. Configure and Compile WPS
```bash
# Source environment
source /shared/WRF/wrf_env.sh

# Configure WPS
./configure
# Select option 3 for Linux x86_64, gfortran (dmpar)

# Edit configure.wps to ensure correct paths
sed -i 's|WRF_DIR.*=.*|WRF_DIR = /shared/WRF/WRF-4.5.2|' configure.wps

# Compile WPS
./compile >& compile.log

# Check compilation
ls -la *.exe
# You should see: geogrid.exe, metgrid.exe, ungrib.exe
```

## WRF-CHEM Setup

### 1. Download KPP (Kinetic Pre-Processor)
```bash
cd /shared/WRF/Downloads
wget https://github.com/KineticPreProcessor/KPP/archive/refs/tags/v2.2.3.tar.gz
tar -xzf v2.2.3.tar.gz
mv KPP-2.2.3 /shared/WRF/KPP
cd /shared/WRF/KPP

# Compile KPP
make
export PATH=$PATH:/shared/WRF/KPP/bin
```

### 2. Configure WRF for CHEM
```bash
cd /shared/WRF/WRF-4.5.2

# Clean previous build
./clean -a

# Set environment for CHEM
export WRF_CHEM=1
export WRF_KPP=1

# Configure WRF with CHEM
./configure
# Select the same options as before (34 for dmpar GNU)

# Compile with CHEM
./compile em_real >& compile_chem.log

# Verify CHEM compilation
ls -la main/
# Should see the same executables with CHEM capabilities
```

## WRF-SOLAR Setup

### 1. Download WRF-SOLAR
```bash
cd /shared/WRF/Downloads
wget https://github.com/NCAR/WRF-SOLAR/archive/refs/heads/release-v4.5.2.tar.gz
tar -xzf release-v4.5.2.tar.gz
mv WRF-SOLAR-release-v4.5.2 /shared/WRF/WRF-SOLAR
cd /shared/WRF/WRF-SOLAR
```

### 2. Configure and Compile WRF-SOLAR
```bash
# Source environment
source /shared/WRF/wrf_env.sh
export WRF_SOLAR=1

# Clean and configure
./clean -a
./configure
# Select option 34 for dmpar GNU

# Compile WRF-SOLAR
./compile em_real >& compile_solar.log

# Verify compilation
ls -la main/
```

## Create Module Files

### 1. Install Environment Modules
```bash
sudo dnf install -y environment-modules
```

### 2. Create WRF Module
```bash
sudo mkdir -p /shared/modulefiles/wrf

cat > /shared/modulefiles/wrf/4.5.2 << 'EOF'
#%Module1.0
##
## WRF 4.5.2 module
##
proc ModulesHelp { } {
    puts stderr "WRF 4.5.2 - Weather Research and Forecasting Model"
    puts stderr "Includes WRF-CHEM and WRF-SOLAR capabilities"
}

module-whatis "WRF 4.5.2 with CHEM and SOLAR"

conflict wrf

set wrf_root "/shared/WRF"
set wrf_version "4.5.2"

# Set WRF paths
setenv WRF_DIR $wrf_root
setenv WRF_ROOT $wrf_root/WRF-$wrf_version
setenv WPS_ROOT $wrf_root/WPS-4.5

# Set library paths
setenv NETCDF $wrf_root/Libraries/netcdf
setenv HDF5 $wrf_root/Libraries/hdf5
setenv PNETCDF $wrf_root/Libraries/pnetcdf
setenv JASPER $wrf_root/Libraries/jasper
setenv LIBPNG $wrf_root/Libraries/libpng
setenv ZLIB $wrf_root/Libraries/zlib

# Add to PATH
prepend-path PATH $wrf_root/WRF-$wrf_version/main
prepend-path PATH $wrf_root/WPS-4.5
prepend-path PATH $wrf_root/Libraries/netcdf/bin
prepend-path PATH $wrf_root/KPP/bin

# Add to LD_LIBRARY_PATH
prepend-path LD_LIBRARY_PATH $wrf_root/Libraries/netcdf/lib
prepend-path LD_LIBRARY_PATH $wrf_root/Libraries/hdf5/lib
prepend-path LD_LIBRARY_PATH $wrf_root/Libraries/pnetcdf/lib
prepend-path LD_LIBRARY_PATH $wrf_root/Libraries/jasper/lib
prepend-path LD_LIBRARY_PATH $wrf_root/Libraries/libpng/lib
prepend-path LD_LIBRARY_PATH $wrf_root/Libraries/zlib/lib

# Set WRF environment variables
setenv WRF_EM_CORE 1
setenv WRF_NMM_CORE 0
setenv WRFIO_NCD_LARGE_FILE_SUPPORT 1
EOF
```

### 3. Setup Module Environment
```bash
# Add module path to all users
echo 'module use /shared/modulefiles' | sudo tee -a /etc/profile.d/modules.sh

# Source modules for current session
source /etc/profile.d/modules.sh
module use /shared/modulefiles
```

## Create Job Templates

### 1. WRF Job Template
```bash
cat > /shared/WRF/wrf_job_template.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=wrf_run
#SBATCH --output=wrf_%j.out
#SBATCH --error=wrf_%j.err
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=4
#SBATCH --time=02:00:00
#SBATCH --partition=compute

# Load WRF module
module load wrf/4.5.2

# Change to run directory
cd $SLURM_SUBMIT_DIR

# Run WRF
echo "Starting WRF at $(date)"
mpirun wrf.exe
echo "WRF completed at $(date)"
EOF
```

### 2. WPS Job Template
```bash
cat > /shared/WRF/wps_job_template.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=wps_run
#SBATCH --output=wps_%j.out
#SBATCH --error=wps_%j.err
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --time=01:00:00
#SBATCH --partition=compute

# Load WRF module
module load wrf/4.5.2

# Change to run directory
cd $SLURM_SUBMIT_DIR

# Run WPS components
echo "Running geogrid.exe at $(date)"
./geogrid.exe

echo "Running ungrib.exe at $(date)"
./ungrib.exe

echo "Running metgrid.exe at $(date)"
./metgrid.exe

echo "WPS completed at $(date)"
EOF
```

### 3. WRF-CHEM Job Template
```bash
cat > /shared/WRF/wrf_chem_job_template.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=wrf_chem
#SBATCH --output=wrf_chem_%j.out
#SBATCH --error=wrf_chem_%j.err
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=4
#SBATCH --time=04:00:00
#SBATCH --partition=compute

# Load WRF module
module load wrf/4.5.2

# Set CHEM environment
export WRF_CHEM=1

# Change to run directory
cd $SLURM_SUBMIT_DIR

# Run WRF-CHEM
echo "Starting WRF-CHEM at $(date)"
mpirun wrf.exe
echo "WRF-CHEM completed at $(date)"
EOF
```

## Verification and Testing

### 1. Test WRF Installation
```bash
# Load module
module load wrf/4.5.2

# Check executables
which wrf.exe
which real.exe
which geogrid.exe

# Test with ideal case
cd /shared/WRF
mkdir -p test_run
cd test_run

# Copy test case (em_quarter_ss)
cp -r $WRF_ROOT/test/em_quarter_ss/* .

# Edit namelist.input if needed
# Run ideal case
mpirun -np 4 ideal.exe
mpirun -np 4 wrf.exe
```

### 2. Download Geographic Data
```bash
# Create directory for geographic data
mkdir -p /shared/WRF/WPS_GEOG

# Download geographic data (this is large, ~50GB)
# You can download from: https://www2.mmm.ucar.edu/wrf/users/download/get_sources_wps_geog.html
# For testing, use low-resolution data first

# Update WPS namelist.wps to point to this directory
# geog_data_path = '/shared/WRF/WPS_GEOG'
```

### 3. Performance Testing
```bash
# Create performance test script
cat > /shared/WRF/performance_test.sh << 'EOF'
#!/bin/bash

# Test different node/core configurations
for nodes in 1 2 4; do
    for ppn in 2 4 8; do
        echo "Testing $nodes nodes with $ppn tasks per node"
        
        sbatch --nodes=$nodes --ntasks-per-node=$ppn \
               --job-name=perf_${nodes}n_${ppn}ppn \
               --output=perf_${nodes}n_${ppn}ppn_%j.out \
               --wrap="module load wrf/4.5.2; cd /shared/WRF/test_run; mpirun wrf.exe"
        
        sleep 5
    done
done
EOF

chmod +x /shared/WRF/performance_test.sh
```

## Maintenance and Updates

### 1. Regular Tasks
```bash
# Check disk usage
df -h /shared

# Monitor running jobs
squeue
sacct

# Check module availability
module avail
```

### 2. Backup Script
```bash
cat > /shared/WRF/backup_wrf.sh << 'EOF'
#!/bin/bash

BACKUP_DIR="/shared/backups/WRF_$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Backup namelists and important configs
find /shared/WRF -name "namelist.*" -exec cp {} $BACKUP_DIR/ \;
find /shared/WRF -name "*.sh" -exec cp {} $BACKUP_DIR/ \;

# Backup module files
cp -r /shared/modulefiles/wrf $BACKUP_DIR/

echo "Backup completed to $BACKUP_DIR"
EOF

chmod +x /shared/WRF/backup_wrf.sh
```

## Troubleshooting

### Common Issues
1. **Library linking errors**: Check LD_LIBRARY_PATH and library installations
2. **MPI errors**: Verify OpenMPI installation and module loading
3. **Memory issues**: Adjust Slurm job memory allocation
4. **File system errors**: Check NFS mount and permissions

### Useful Commands
```bash
# Check WRF compilation
tail -100 /shared/WRF/WRF-4.5.2/compile.log

# Test libraries
ldd /shared/WRF/WRF-4.5.2/main/wrf.exe

# Monitor job resources
sstat -j JOBID --format=JobID,MaxRSS,MaxVMSize,NTasks

# Check node health
scontrol show nodes
```

This installation provides a complete WRF suite with chemistry and solar radiation capabilities, optimized for your Slurm cluster environment.
