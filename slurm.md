# Slurm 23.11.5 Setup Guide - Rocky Linux 9.5

## Prerequisites (All Nodes)

### 1. Add Slurm User and Group
```bash
sudo groupadd -g 1002 slurm
sudo useradd -m -c "Slurm Workload Manager" -d /var/lib/slurm -u 1002 -g slurm -s /bin/bash slurm
```

### 2. Install Required Packages
```bash
# Install EPEL and development tools
sudo dnf install -y epel-release
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y munge munge-libs munge-devel

# Install dependencies for Slurm
sudo dnf install -y rpm-build libtool hwloc hwloc-devel \
    numactl numactl-devel lua lua-devel readline-devel \
    rrdtool-devel ncurses-devel man2html libibmad libibumad \
    pmix-devel libevent-devel json-c-devel http-parser-devel \
    dbus-devel gtk2-devel glib2-devel libssh2-devel mysql-devel \
    hdf5-devel freeipmi-devel
```

### 3. Install OpenMPI
```bash
sudo dnf install -y openmpi openmpi-devel
echo 'export PATH=$PATH:/usr/lib64/openmpi/bin' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib64/openmpi/lib' >> ~/.bashrc
source ~/.bashrc
```

## NFS Setup (Master Node Only)

### 1. Install and Configure NFS Server
```bash
sudo dnf install -y nfs-utils

# Create shared directory
sudo mkdir -p /shared
sudo chown slurm:slurm /shared
sudo chmod 755 /shared

# Configure NFS exports
sudo tee /etc/exports << EOF
/shared 10.10.140.0/24(rw,sync,no_root_squash,no_subtree_check)
EOF

# Start and enable NFS services
sudo systemctl enable --now rpcbind nfs-server
sudo exportfs -ra
```

### 2. Firewall Configuration (Master)
```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload
```

## NFS Client Setup (Compute Nodes Only)

### 1. Install NFS Client
```bash
sudo dnf install -y nfs-utils

# Create mount point
sudo mkdir -p /shared

# Add to fstab for persistent mounting
echo "10.10.140.40:/shared /shared nfs defaults 0 0" | sudo tee -a /etc/fstab

# Mount the share
sudo mount -a
```

## Munge Setup (All Nodes)

### 1. Generate Munge Key (Master Node Only)
```bash
sudo /usr/sbin/create-munge-key -r
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
```

### 2. Copy Munge Key to Compute Nodes
```bash
# On master node
sudo scp /etc/munge/munge.key root@10.10.140.41:/etc/munge/
sudo scp /etc/munge/munge.key root@10.10.140.42:/etc/munge/
sudo scp /etc/munge/munge.key root@10.10.140.43:/etc/munge/

# On each compute node, set proper permissions
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
```

### 3. Start Munge Service (All Nodes)
```bash
sudo systemctl enable --now munge
sudo systemctl status munge
```

## Slurm Installation (All Nodes)

### 1. Download and Compile Slurm
```bash
cd /tmp
wget https://download.schedmd.com/slurm/slurm-23.11.5.tar.bz2
tar xjf slurm-23.11.5.tar.bz2
cd slurm-23.11.5

# Configure build
./configure --prefix=/usr/local/slurm \
    --sysconfdir=/etc/slurm \
    --with-systemdsystemunitdir=/etc/systemd/system \
    --enable-pam \
    --with-pam_dir=/lib64/security \
    --without-shared-libslurm \
    --with-pmix \
    --with-hwloc \
    --with-json

# Compile and install
make -j$(nproc)
sudo make install

# Create necessary directories
sudo mkdir -p /etc/slurm
sudo mkdir -p /var/spool/slurm/ctld
sudo mkdir -p /var/spool/slurm/d
sudo mkdir -p /var/log/slurm

# Set ownership
sudo chown slurm:slurm /var/spool/slurm/ctld
sudo chown slurm:slurm /var/spool/slurm/d
sudo chown slurm:slurm /var/log/slurm
```

### 2. Add Slurm to PATH
```bash
echo 'export PATH=$PATH:/usr/local/slurm/bin:/usr/local/slurm/sbin' | sudo tee /etc/profile.d/slurm.sh
source /etc/profile.d/slurm.sh
```

## Slurm Configuration

### Master Node Configuration

#### 1. Create slurm.conf (Master Node)
```bash
sudo tee /etc/slurm/slurm.conf << 'EOF'
# Slurm Configuration File
ClusterName=rocky-cluster
ControlMachine=master
ControlAddr=10.10.140.40

# Authentication
AuthType=auth/munge
CryptoType=crypto/munge

# Scheduling
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core

# Accounting
AccountingStorageType=accounting_storage/none
JobAcctGatherType=jobacct_gather/none

# Logging
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid

# State preservation
StateSaveLocation=/var/spool/slurm/ctld
SlurmdSpoolDir=/var/spool/slurm/d

# Process tracking (without cgroups as requested)
ProctrackType=proctrack/linuxproc

# MPI support
MpiDefault=pmix

# Timeouts
SlurmctldTimeout=120
SlurmdTimeout=300
InactiveLimit=0
MinJobAge=300
KillWait=30
Waittime=0

# Scheduling parameters
SchedulerTimeSlice=5
DefMemPerCPU=1024
MaxJobCount=10000
MaxArraySize=1000

# Default resources
DefMemPerNode=1024

# Node definitions
NodeName=compute01 NodeAddr=10.10.140.41 CPUs=4 Sockets=1 CoresPerSocket=4 ThreadsPerCore=1 RealMemory=7900 State=UNKNOWN
NodeName=compute02 NodeAddr=10.10.140.42 CPUs=4 Sockets=1 CoresPerSocket=4 ThreadsPerCore=1 RealMemory=7900 State=UNKNOWN
NodeName=compute03 NodeAddr=10.10.140.43 CPUs=4 Sockets=1 CoresPerSocket=4 ThreadsPerCore=1 RealMemory=7900 State=UNKNOWN

# Partition definitions
PartitionName=compute Nodes=compute[01-03] Default=YES MaxTime=INFINITE State=UP
EOF
```

#### 2. Create Systemd Service for Controller (Master Node)
```bash
sudo tee /etc/systemd/system/slurmctld.service << 'EOF'
[Unit]
Description=Slurm controller daemon
After=network.target munge.service
Wants=munge.service

[Service]
Type=notify
EnvironmentFile=-/etc/default/slurmctld
ExecStart=/usr/local/slurm/sbin/slurmctld -D $SLURMCTLD_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
PIDFile=/var/run/slurmctld.pid
LimitNOFILE=65536
LimitMEMLOCK=infinity
LimitSTACK=infinity
User=slurm
Group=slurm

[Install]
WantedBy=multi-user.target
EOF
```

### Compute Nodes Configuration

#### 1. Copy slurm.conf to Compute Nodes
```bash
# On master node
sudo scp /etc/slurm/slurm.conf root@10.10.140.41:/etc/slurm/
sudo scp /etc/slurm/slurm.conf root@10.10.140.42:/etc/slurm/
sudo scp /etc/slurm/slurm.conf root@10.10.140.43:/etc/slurm/
```

#### 2. Create Systemd Service for Compute Nodes
```bash
# Run this on each compute node
sudo tee /etc/systemd/system/slurmd.service << 'EOF'
[Unit]
Description=Slurm node daemon
After=network.target munge.service
Wants=munge.service

[Service]
Type=notify
EnvironmentFile=-/etc/default/slurmd
ExecStart=/usr/local/slurm/sbin/slurmd -D $SLURMD_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
PIDFile=/var/run/slurmd.pid
KillMode=process
LimitNOFILE=131072
LimitMEMLOCK=infinity
LimitSTACK=infinity
User=root
Group=root

[Install]
WantedBy=multi-user.target
EOF
```

## Firewall Configuration

### Master Node
```bash
sudo firewall-cmd --permanent --add-port=6817/tcp  # slurmctld
sudo firewall-cmd --permanent --add-port=6818/tcp  # slurmd
sudo firewall-cmd --permanent --add-port=6819/tcp  # slurmdbd (if needed)
sudo firewall-cmd --reload
```

### Compute Nodes
```bash
sudo firewall-cmd --permanent --add-port=6818/tcp  # slurmd
sudo firewall-cmd --reload
```

## Starting Services

### Master Node
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now slurmctld
sudo systemctl status slurmctld
```

### Compute Nodes
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now slurmd
sudo systemctl status slurmd
```

## Verification and Testing

### 1. Check Cluster Status
```bash
sinfo
squeue
scontrol show nodes
```

### 2. Test Job Submission
```bash
# Simple test job
echo '#!/bin/bash
echo "Hello from $(hostname)"
sleep 10' > /shared/test_job.sh

chmod +x /shared/test_job.sh
sbatch /shared/test_job.sh
```

### 3. Test MPI Job
```bash
# Create MPI test program
cat > /shared/mpi_hello.c << 'EOF'
#include <mpi.h>
#include <stdio.h>

int main(int argc, char** argv) {
    MPI_Init(NULL, NULL);
    int world_size;
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);
    int world_rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
    char processor_name[MPI_MAX_PROCESSOR_NAME];
    int name_len;
    MPI_Get_processor_name(processor_name, &name_len);
    printf("Hello world from processor %s, rank %d out of %d processors\n",
           processor_name, world_rank, world_size);
    MPI_Finalize();
    return 0;
}
EOF

# Compile MPI program
mpicc -o /shared/mpi_hello /shared/mpi_hello.c

# Create MPI job script
cat > /shared/mpi_job.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=mpi_test
#SBATCH --output=mpi_test_%j.out
#SBATCH --error=mpi_test_%j.err
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=2
#SBATCH --time=00:05:00

module load mpi/openmpi-x86_64
mpirun /shared/mpi_hello
EOF

chmod +x /shared/mpi_job.sh
sbatch /shared/mpi_job.sh
```

## Troubleshooting

### Common Commands
```bash
# Check logs
sudo tail -f /var/log/slurm/slurmctld.log
sudo tail -f /var/log/slurm/slurmd.log

# Restart services
sudo systemctl restart slurmctld  # on master
sudo systemctl restart slurmd     # on compute nodes

# Check node states
scontrol show nodes
scontrol update NodeName=compute01 State=RESUME  # if node is down
```

### Node States
- **IDLE**: Ready for jobs
- **DOWN**: Node is down
- **DRAIN**: Node is draining jobs
- **UNKNOWN**: Initial state, should change to IDLE

This setup provides a complete Slurm 23.11.5 cluster with OpenMPI support, NFS shared storage, and proper authentication via Munge, without cgroups configuration as requested.
