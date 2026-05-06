# GCP Slurm — state3

Deployed with [cluster toolkit](https://github.com/GoogleCloudPlatform/cluster-toolkit) in the `arc-state3` GCP project.

## Connecting

```bash
gcloud compute ssh hpcslurmst-slurm-login-001 --project=arc-state3 --zone=us-east1-b --tunnel-through-iap
```

## Storage

| Path | Type | Size | Access |
|------|------|------|--------|
| `/home` | Filestore NFS (BASIC_HDD) | 5 TB | Read/write, persistent |
| `/data` | Filestore NFS (BASIC_HDD) | 10 TB | Read/write, persistent |
| `/mnt/gcs/state3-data-curation` | GCS bucket (gcsfuse) | Unlimited | Read/write |
| `/scratch` | Local NVMe RAID | 3 TB | Read/write, **ephemeral** |

### Recommended: Use GCS for primary data storage

**Prefer `/mnt/gcs/state3-data-curation` or direct `gcloud storage` commands over Filestore for large datasets.** GCS storage is cheaper, scales without limits, and is accessible from anywhere — not just the cluster.

Copying data into `/data` before a job is faster for I/O-intensive pipelines (GCSFuse adds overhead for small random reads). Use `gcloud storage` to stage data:

```bash
# Copy data from GCS into /data for a pipeline run
gcloud storage cp -r "gs://state3-data-curation/my-experiment/**" /data/my-experiment/

# Copy results back to GCS when done
gcloud storage cp -r /data/my-experiment/results/ gs://state3-data-curation/my-experiment/results/
```

For large transfers, run these on the `debug` partition or the login node rather than inside a compute job.

### Data Transfer from a Local Machine

```bash
# Upload a folder to the login node
gcloud compute scp --recurse ./your_folder/ hpcslurmst-slurm-login-001:~ \
  --zone=us-east1-b --project=arc-state3 --tunnel-through-iap

# Or upload directly to GCS (faster, no cluster needed)
gcloud storage cp -r ./your_folder/ gs://state3-data-curation/your_folder/
```

## Compute

```bash
$ sinfo
PARTITION    AVAIL  TIMELIMIT  NODES  STATE NODELIST
compute         up   infinite     20  idle~ hpcslurmst-computenodeset-[0-19]
computelarge    up   infinite     20  idle~ hpcslurmst-computelargeno-[0-19]
debug*          up   infinite      4  idle~ hpcslurmst-debugnodeset-[0-3]
```

The partitions are **dynamic** — at rest, no nodes are running. When a job is submitted, Slurm auto-scales by spinning up the required nodes. Nodes shut down after 1 minute of inactivity.

**Expect a few minutes of delay when the cluster is "cold" and a node needs to provision.**

| Partition | Machine | vCPUs | RAM | Local Scratch | Notes |
|-----------|---------|-------|-----|---------------|-------|
| `debug` (default) | n2-standard-4 | 4 | 16 GB | — | On-demand, non-preemptible. Nodes stay up after jobs end. Use for interactive work and data transfer. |
| `compute` | c2d-standard-32 | 32 | 128 GB | 3 TB NVMe (`/scratch`) | SPOT (preemptible). Use for standard pipeline runs. |
| `computelarge` | c2d-standard-56 | 56 | 224 GB | 3 TB NVMe (`/scratch`) | SPOT (preemptible). Use for memory-intensive jobs. |

> **Note on SPOT partitions:** `compute` and `computelarge` use preemptible VMs for cost savings. Jobs may be interrupted if GCP reclaims capacity. **Batch jobs (`sbatch`) are automatically requeued when a node is preempted** — Slurm detects the node failure and re-schedules the job on a new node. Interactive `srun` sessions are not requeued and will need to be restarted manually. For long-running jobs, write intermediate checkpoints to `/data` so your pipeline can resume from where it left off rather than restarting from scratch.

### Local Scratch (`/scratch`)

Compute nodes have 8 × 375 GB NVMe SSDs RAIDed together and mounted at `/scratch` (3 TB, world-writable). This is fast local storage — use it for temporary working files during a job. **Data on `/scratch` is lost when the node shuts down.**

```bash
# Example: write job temp files to scratch
export TMPDIR=/scratch/$SLURM_JOB_ID
mkdir -p $TMPDIR
```

## Running Jobs

### Interactive session

Use the `debug` partition for interactive work — it uses on-demand VMs so there's no preemption risk:

```bash
srun -p debug --pty bash
```

For an interactive session on a compute node (e.g. to test a pipeline before batch submission):

```bash
srun -p compute --pty bash
```

### Batch jobs

Create a job script and submit with `sbatch`:

```bash
#!/bin/bash
#SBATCH --job-name=my-job
#SBATCH --partition=compute
#SBATCH --nodes=1
#SBATCH --cpus-per-task=32
#SBATCH --mem=120G
#SBATCH --requeue
#SBATCH --output=/data/logs/%j.out
#SBATCH --error=/data/logs/%j.err

# Stage input data to local scratch
mkdir -p /scratch/$SLURM_JOB_ID
gcloud storage cp -r gs://state3-data-curation/my-input/ /scratch/$SLURM_JOB_ID/

# Run pipeline
my-pipeline --input /scratch/$SLURM_JOB_ID/my-input --output /scratch/$SLURM_JOB_ID/output

# Copy results to GCS
gcloud storage cp -r /scratch/$SLURM_JOB_ID/output/ gs://state3-data-curation/results/
```

```bash
sbatch my-job.sh
```

### Useful Slurm commands

```bash
squeue                          # show all running/pending jobs
squeue -u $USER                 # show your jobs
scancel <job_id>                # cancel a job
scontrol show job <job_id>      # detailed job info
sinfo -R                        # show node failure reasons
```

### Requesting resources

```bash
# Single node, all cores on compute partition
srun -p compute -N 1 --cpus-per-task=32 --pty bash

# Large memory job on computelarge
sbatch --partition=computelarge --mem=200G --cpus-per-task=56 my-job.sh

# Multi-node job
sbatch --partition=compute --nodes=4 my-mpi-job.sh
```

## Containers

> **Container support is not enabled by default.** The base SchedMD Slurm image does not include Apptainer. To use containers, a cluster admin must add Apptainer via a custom image build or startup script (see [toolkit example](../community/examples/hpc-slurm6-apptainer.yaml)).

**Apptainer** (the successor to Singularity) is the recommended container runtime. Docker is not available on compute nodes. `srun --container-image` (pyxis/enroot) is also not available — that requires a GPU-specific image.

### Pulling images

```bash
# Pull from Docker Hub (saves as a .sif file)
apptainer pull docker://ubuntu:22.04
apptainer pull docker://continuumio/miniconda3

# Pull from GitHub Container Registry
apptainer pull docker://ghcr.io/org/image:tag
```

Store `.sif` files in `/data/containers/` so they persist across sessions. Pull images on the login node or a `debug` node — avoid pulling inside a SPOT compute job.

> **Note:** `srun --container-image` / `--container-name` (pyxis/enroot) and `docker run` are **not available** on this cluster. Apptainer is the supported container runtime.

### Running containers

```bash
# Run a command inside the container
apptainer exec --bind /data,/scratch my-image.sif my-command --arg

# Interactive shell
apptainer shell --bind /data,/scratch my-image.sif
```

### Container batch job

```bash
#!/bin/bash
#SBATCH --job-name=container-job
#SBATCH --partition=compute
#SBATCH --cpus-per-task=32
#SBATCH --mem=120G
#SBATCH --requeue
#SBATCH --output=/data/logs/%j.out

SIF=/data/containers/my-pipeline.sif
export TMPDIR=/scratch/$SLURM_JOB_ID
mkdir -p $TMPDIR

gcloud storage cp -r gs://state3-data-curation/my-input/ $TMPDIR/

apptainer exec \
  --bind /data:/data \
  --bind /scratch:/scratch \
  $SIF my-pipeline \
    --input $TMPDIR/my-input \
    --output $TMPDIR/output \
    --threads $SLURM_CPUS_PER_TASK

gcloud storage cp -r $TMPDIR/output/ gs://state3-data-curation/results/
```

## Claude Code

[Claude Code](https://code.claude.com/docs/en/quickstart) is available on the cluster. Install it in your home directory (persists across sessions):

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Then launch from any node:

```bash
claude
```

See the [quickstart guide](https://code.claude.com/docs/en/quickstart) for usage.

### state3-hpc Plugin

The `state3-hpc` Claude Code plugin gives Claude built-in knowledge of this cluster — partitions, storage layout, GCS patterns, job submission, and more. Install it once from the Arc plugin marketplace:

```bash
# 1. Add the Arc marketplace (one-time setup)
/plugin marketplace add ArcInstitute/arc-claude-plugins

# 2. Install the state3-hpc plugin
/plugin install state3-hpc@arc-claude-plugins
```

Once installed, Claude will automatically load cluster-specific context whenever you ask about Slurm jobs, storage, or GCP on this cluster.
