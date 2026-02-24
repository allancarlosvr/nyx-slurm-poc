# 🚀 Nyx Slurm Cluster PoC  
**The Exploration Company – HPC Engineer Technical Assignment**

---

## 📌 Overview

This Proof of Concept demonstrates a fully functional **3-node Slurm cluster** built using **LXD containers** on a single Intel NUC host.

The cluster supports:

- Multi-node jobs  
- CPU-intensive workloads  
- Memory-intensive workloads  
- Multiple partitions  
- Node failure simulation  
- Job requeue & recovery  
- Deterministic networking  

---

# 🏗 Architecture

## Infrastructure Layout

```
Intel NUC Host
│
├── nyx-net (10.200.0.0/24)
│
├── head        (10.200.0.10) → slurmctld
├── compute-1   (10.200.0.11) → slurmd
└── compute-2   (10.200.0.12) → slurmd
```

### Networking

- Dedicated LXD bridge: `nyx-net`
- Static IP addressing
- Hostname resolution via `/etc/hosts`
- Full inter-node connectivity validated

---

# ⚙️ Slurm Configuration

## slurm.conf Highlights

```ini
ClusterName=nyx-cluster
SlurmctldHost=head
AuthType=auth/munge
SchedulerType=sched/backfill
SelectType=select/cons_tres

ProctrackType=proctrack/linuxproc
TaskPlugin=task/none

NodeName=compute-1 CPUs=4 Sockets=1 CoresPerSocket=4 ThreadsPerCore=1 RealMemory=2000 State=UNKNOWN
NodeName=compute-2 CPUs=4 Sockets=1 CoresPerSocket=4 ThreadsPerCore=1 RealMemory=2000 State=UNKNOWN

PartitionName=normal Nodes=compute-1,compute-2 Default=YES State=UP
PartitionName=gpu Nodes=compute-2 Default=NO State=UP
```

---

# 🔐 Munge Authentication

- `munge.key` generated on head node  
- Distributed to compute nodes  
- Permissions set to `400`  
- Ownership set to `munge:munge`  

Validation:

```bash
munge -n | unmunge
```

---

# ✅ Cluster Validation

## Node Status

```bash
sinfo
```

Expected:

```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
normal*      up   infinite      2   idle compute-[1-2]
gpu          up   infinite      1   idle compute-2
```

---

# 🧪 Multi-Node Job Test

## Submission

```bash
sbatch multinode.sbatch
```

## Script

```bash
#!/bin/bash
#SBATCH -J multinode
#SBATCH -p normal
#SBATCH -N 2
#SBATCH -n 2
#SBATCH -t 00:01:00
#SBATCH -o /shared/multinode-%j.out

echo "JobID=$SLURM_JOB_ID"
echo "Nodes=$SLURM_JOB_NODELIST"
srun hostname
```

## Expected Output

```
JobID=3
Nodes=compute-[1-2]
compute-2
compute-1
```

✔ Multi-node allocation confirmed

---

# 🔥 Resource Management Test

## CPU-Intensive Job

```bash
#!/bin/bash
#SBATCH -J cpu_stress
#SBATCH -p normal
#SBATCH -N 1
#SBATCH -c 4
#SBATCH -t 00:03:00
#SBATCH -o /shared/cpu-stress-%j.out

stress-ng --cpu 4 --timeout 150s --metrics-brief
```

## Memory-Intensive Job

```bash
#!/bin/bash
#SBATCH -J mem_stress
#SBATCH -p normal
#SBATCH -N 1
#SBATCH -c 1
#SBATCH --mem=500M
#SBATCH -t 00:03:00
#SBATCH -o /shared/mem-stress-%j.out

stress-ng --vm 1 --vm-bytes 400M --timeout 150s --metrics-brief
```

Submission:

```bash
sbatch cpu-stress.sbatch
sbatch mem-stress.sbatch
squeue
```

✔ Overlapping workloads scheduled correctly

---

# 🚨 Failure Simulation & Recovery

## Simulate Node Failure

```bash
scontrol update NodeName=compute-2 state=down reason="failure_test"
sinfo
```

Expected:

```
compute-2  down
```

Jobs enter pending or requeue state.

---

## Resume Node

```bash
scontrol update NodeName=compute-2 state=resume
sinfo
```

Expected:

```
compute-[1-2] idle
```

✔ Node recovered  
✔ Jobs resumed successfully  

---

# 🎯 GPU Partition Enforcement

## GPU Partition Job

```bash
#!/bin/bash
#SBATCH -J gpu_part
#SBATCH -p gpu
#SBATCH -N 1
#SBATCH -t 00:01:00
#SBATCH -o /shared/gpu-part-%j.out

hostname
```

Expected output:

```
compute-2
```

✔ Partition restriction validated

---

# 📊 Operational Runbook

| Command | Purpose |
|----------|----------|
| `sinfo` | View cluster state |
| `squeue` | View job queue |
| `sbatch <file>` | Submit job |
| `scontrol show job <id>` | Inspect job details |
| `scancel <id>` | Cancel job |
| `scontrol update NodeName=<node> state=down` | Simulate failure |
| `scontrol update NodeName=<node> state=resume` | Restore node |

---

# 🧠 Design Decisions

### Container Compatibility

Because LXD containers do not support the freezer cgroup namespace, the configuration uses:

```ini
ProctrackType=proctrack/linuxproc
TaskPlugin=task/none
```

In production (bare metal), cgroup enforcement would be enabled.

### Deterministic Networking

Static IP allocation ensures stable Slurm communication between nodes.

### Shared Storage

`/shared` directory mounted via LXD disk device for centralized job output access.

---

# 📈 Assignment Coverage

| Requirement | Status |
|------------|--------|
| 3-node Slurm cluster | ✅ |
| Dedicated networking | ✅ |
| Multi-node job support | ✅ |
| Multiple partitions | ✅ |
| CPU & memory workloads | ✅ |
| Node drain & resume | ✅ |
| Job requeue behavior | ✅ |
| Monitoring & user workflow | ✅ |

---

# 🏁 Conclusion

This PoC demonstrates:

- Slurm configuration fluency  
- Authentication troubleshooting  
- Deterministic cluster networking  
- Multi-node scheduling correctness  
- Resource management  
- Failure simulation and recovery  
- Operational maturity  

The cluster is fully functional, reproducible, and aligned with production HPC principles.