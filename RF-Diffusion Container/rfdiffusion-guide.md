# RFdiffusion: Complete User Guide

A guide for using this containerized version of RFdiffusion, a powerful protein structure generation tool developed by the Baker Lab that enables unconditional protein generation, motif scaffolding, binder design, and symmetric oligomer creation.

This is a cleaner version of the official repository, making it easy to set up, understand, and run experiments.

## Table of Contents

- [Setup Scripts](#setup-scripts)
- [System Requirements](#system-requirements)
- [Basic Usage](#basic-usage)
- [Task-Specific Examples](#task-specific-examples)
  - [Unconditional Protein Generation](#1-unconditional-monomer-generation)
  - [Motif Scaffolding](#2-motif-scaffolding)
  - [Binder Design (PPI)](#3-binder-design-ppi)
  - [Symmetric Oligomer Design](#4-symmetric-oligomer-design)
  - [Structure Diversification](#5-structure-diversification-partial-diffusion)
- [Understanding the Contig Format](#understanding-the-contig-format)
- [Model Checkpoints](#model-checkpoints)
- [Advanced Options](#advanced-options)
- [Output Files](#output-files)
- [Troubleshooting](#troubleshooting)
- [Performance Tips](#performance-tips)
- [Common Workflows](#common-workflows)

Container Image:

```bash
docker pull ghcr.io/biohacker0/rfdiffusion-new:local
```

## Setup Scripts

Save the following script as `setup_rfdiffusion.sh` in your home directory to automate the setup process:

```bash
#!/bin/bash
# Setup script for RFdiffusion

# Configuration
IMAGE_NAME="rfdiffusion:local"
GHCR_IMAGE="ghcr.io/biohacker0/rfdiffusion-new:local"

# Create directory structure
mkdir -p $HOME/rfdiffusion/{inputs,outputs,models,scripts}
cd $HOME/rfdiffusion

# Download sample PDB file
echo "Downloading sample PDB file to inputs directory..."
wget -P $HOME/rfdiffusion/inputs https://files.rcsb.org/download/5TPN.pdb

# Pull image
if ! docker image inspect $IMAGE_NAME >/dev/null 2>&1; then
  echo "Pulling $GHCR_IMAGE..."
  docker pull $GHCR_IMAGE
  docker tag $GHCR_IMAGE $IMAGE_NAME
fi

# Create models download script
cat > scripts/download_models.sh << 'EOF'
#!/bin/bash
MODELS_DIR=${1:-"$HOME/rfdiffusion/models"}
mkdir -p $MODELS_DIR
cd $MODELS_DIR

echo "Downloading RFdiffusion model weights to $MODELS_DIR..."

wget -c http://files.ipd.uw.edu/pub/RFdiffusion/6f5902ac237024bdd0c176cb93063dc4/Base_ckpt.pt
wget -c http://files.ipd.uw.edu/pub/RFdiffusion/e29311f6f1bf1af907f9ef9f44b8328b/Complex_base_ckpt.pt

read -p "Download additional models for advanced features? (y/n) " answer
if [[ $answer == "y" ]]; then
  wget -c http://files.ipd.uw.edu/pub/RFdiffusion/60f09a193fb5e5ccdc4980417708dbab/Complex_Fold_base_ckpt.pt
  wget -c http://files.ipd.uw.edu/pub/RFdiffusion/74f51cfb8b440f50d70878e05361d8f0/InpaintSeq_ckpt.pt
  wget -c http://files.ipd.uw.edu/pub/RFdiffusion/76d00716416567174cdb7ca96e208296/InpaintSeq_Fold_ckpt.pt
  wget -c http://files.ipd.uw.edu/pub/RFdiffusion/5532d2e1f3a4738decd58b19d633b3c3/ActiveSite_ckpt.pt
  wget -c http://files.ipd.uw.edu/pub/RFdiffusion/12fc204edeae5b57713c5ad7dcb97d39/Base_epoch8_ckpt.pt
  wget -c http://files.ipd.uw.edu/pub/RFdiffusion/f572d396fae9206628714fb2ce00f72e/Complex_beta_ckpt.pt
fi

echo "Model download complete!"
EOF

# Create run wrapper script
cat > scripts/run_rfdiffusion.sh << 'EOF'
#!/bin/bash
# Run RFdiffusion with standard paths

MODELS_DIR=${MODELS_DIR:-"$HOME/rfdiffusion/models"}
INPUTS_DIR=${INPUTS_DIR:-"$HOME/rfdiffusion/inputs"}
OUTPUTS_DIR=${OUTPUTS_DIR:-"$HOME/rfdiffusion/outputs"}
OUTPUT_NAME=${1:-"rfd_output"}
shift

for dir in "$MODELS_DIR" "$INPUTS_DIR" "$OUTPUTS_DIR"; do
  if [ ! -d "$dir" ]; then
    echo "Error: Directory $dir does not exist."
    exit 1
  fi
done

if [ ! "$(ls -A $MODELS_DIR)" ]; then
  echo "No models found in $MODELS_DIR. Run ./scripts/download_models.sh first."
  exit 1
fi

docker run -it --rm --gpus all \
  -v $MODELS_DIR:/models:ro \
  -v $INPUTS_DIR:/inputs:ro \
  -v $OUTPUTS_DIR:/outputs:rw \
  rfdiffusion:local \
  inference.output_prefix=/outputs/$OUTPUT_NAME \
  inference.model_directory_path=/models \
  "$@"

echo "Results saved to $OUTPUTS_DIR/$OUTPUT_NAME"
EOF

# Create examples script
cat > scripts/examples.sh << 'EOF'
#!/bin/bash
# RFdiffusion examples

function show_examples() {
  echo "======================================"
  echo "RFdiffusion Example Commands"
  echo "======================================"
  echo
  echo "1. Generate a basic 150-residue protein:"
  echo "./scripts/run_rfdiffusion.sh basic_protein 'contigmap.contigs=[150-150]' inference.num_designs=5"
  echo
  echo "2. Scaffold a motif using 5TPN.pdb:"
  echo "./scripts/run_rfdiffusion.sh motif_scaffold inference.input_pdb=/inputs/5TPN.pdb 'contigmap.contigs=[10-40/A163-181/10-40]' inference.num_designs=5"
  echo
  echo "3. Design a protein binder using 5TPN.pdb as target:"
  echo "./scripts/run_rfdiffusion.sh binder inference.input_pdb=/inputs/5TPN.pdb 'contigmap.contigs=[A163-181/0 100-100]' 'ppi.hotspot_res=[A170,A173,A174]' inference.num_designs=5"
  echo
  echo "4. Generate a symmetric tetramer:"
  echo "./scripts/run_rfdiffusion.sh symmetric_tetramer --config-name symmetry inference.symmetry=tetrahedral 'contigmap.contigs=[360]' inference.num_designs=1"
  echo
  echo "5. Design with reduced noise (better quality, less diversity):"
  echo "./scripts/run_rfdiffusion.sh high_quality 'contigmap.contigs=[150-150]' denoiser.noise_scale_ca=0.5 denoiser.noise_scale_frame=0.5 inference.num_designs=3"
  echo
  echo "Run these commands from your rfdiffusion directory."
  echo "Note: Ensure the contig map matches your PDB file's chain and residue numbers."
  echo "Check residues with: grep \"^ATOM\" inputs/5TPN.pdb | awk '{print \$5, \$6}' | sort | uniq"
}

show_examples
EOF

# Make scripts executable
chmod +x scripts/*.sh

echo "RFdiffusion setup complete!"
echo "Run ./scripts/download_models.sh to download model weights."
echo "Sample PDB file (5TPN.pdb) has been downloaded to the inputs directory."
echo "See ./scripts/examples.sh for usage examples."
```

Make it executable and run it:

```bash
chmod +x setup_rfdiffusion.sh
./setup_rfdiffusion.sh
cd ~/rfdiffusion
./scripts/download_models.sh
./scripts/examples.sh
```

## System Requirements

- **Docker**: Latest stable version
- **GPU**: NVIDIA GPU with CUDA 11.6+ support
- **NVIDIA Container Toolkit**: Required for GPU access in containers
- **Memory**: 16GB RAM minimum, 32GB+ recommended for larger proteins
- **GPU Memory**:
  - 8GB for small proteins (<150 residues)
  - 16GB+ for larger proteins or complexes
- **Storage**: ~10GB for model files and container, plus space for outputs

### Checking Your Setup

```bash
# Check Docker
docker --version

# Check NVIDIA drivers
nvidia-smi

# Check NVIDIA Container Toolkit
docker run --rm --gpus all nvidia/cuda:11.6.2-base-ubuntu20.04 nvidia-smi
```

## Basic Usage

### Command Structure

```bash
docker run -it --rm --gpus all \
  -v /path/to/models:/models:ro \
  -v /path/to/inputs:/inputs:ro \
  -v /path/to/outputs:/outputs:rw \
  rfdiffusion:local \
  inference.output_prefix=/outputs/my_output \
  inference.model_directory_path=/models \
  [additional arguments...]
```

### Essential Arguments

| Argument                         | Description                              | Example                | Required?                  |
| -------------------------------- | ---------------------------------------- | ---------------------- | -------------------------- |
| `inference.output_prefix`        | Path for output files                    | `/outputs/test`        | Yes                        |
| `inference.model_directory_path` | Path to models directory                 | `/models`              | Yes                        |
| `contigmap.contigs`              | Defines protein structure to generate    | `[150-150]`            | Yes                        |
| `inference.num_designs`          | Number of designs to generate            | `10`                   | Optional (default=1)       |
| `inference.input_pdb`            | Input PDB file (for scaffolding/binding) | `/inputs/my_motif.pdb` | Required for certain tasks |

### Using the Wrapper Script

After setting up with `setup_rfdiffusion.sh`:

```bash
# Format: ./scripts/run_rfdiffusion.sh OUTPUT_NAME [arguments...]
./scripts/run_rfdiffusion.sh my_design 'contigmap.contigs=[150-150]' inference.num_designs=5
```

## Task-Specific Examples

### 1. Unconditional Monomer Generation

Generate completely new protein structures without any template:

```bash
./scripts/run_rfdiffusion.sh monomer \
  'contigmap.contigs=[150-150]' \
  inference.num_designs=10
```

**Parameters to Experiment With:**

- **Length**: Change `[150-150]` to desired protein length
- **Length range**: Use `[100-200]` to sample lengths between 100-200 residues
- **Quality vs. diversity**: Add `denoiser.noise_scale_ca=0.5 denoiser.noise_scale_frame=0.5` for better quality but less diversity

### 2. Motif Scaffolding

Design a protein that incorporates a functional motif:

```bash
./scripts/run_rfdiffusion.sh motif_scaffold \
  inference.input_pdb=/inputs/5TPN.pdb \
  'contigmap.contigs=[10-40/A163-181/10-40]' \
  inference.num_designs=10
```

**Key Considerations:**

- `A163-181`: References residues 163-181 on chain A of input PDB
- The model will build 10-40 residues before and 10-40 residues after the motif
- For very small motifs (fewer than 5 residues), use the ActiveSite model:
  ```bash
  inference.ckpt_override_path=/models/ActiveSite_ckpt.pt
  ```

### 3. Binder Design (PPI)

Design a protein that binds to a specific target:

```bash
./scripts/run_rfdiffusion.sh binder \
  inference.input_pdb=/inputs/5TPN.pdb \
  'contigmap.contigs=[A163-181/0 100-100]' \
  'ppi.hotspot_res=[A170,A173,A174]' \
  denoiser.noise_scale_ca=0.5 \
  denoiser.noise_scale_frame=0.5 \
  inference.num_designs=10
```

**Best Practices:**

- **Target preparation**: Truncate target to only include the binding region plus ~10Å buffer
- **Hotspots**: Choose 3-6 surface residues with hydrophobic character to define binding interface
- **Noise reduction**: Lower noise scales (0.5) generally improves binder quality
- **Diverse topologies**: To generate binders with more β-sheet content:
  ```bash
  inference.ckpt_override_path=/models/Complex_beta_ckpt.pt
  ```

### 4. Symmetric Oligomer Design

Create symmetric protein assemblies:

```bash
./scripts/run_rfdiffusion.sh c4_symmetric \
  --config-name symmetry \
  inference.symmetry=c4 \
  'contigmap.contigs=[120]' \
  inference.num_designs=5
```

**Important**: For symmetric designs, the total length must be exactly divisible by the number of subunits. For example:

- C4 symmetry: Length must be divisible by 4
- D2 symmetry: Length must be divisible by 4
- Tetrahedral: Length must be divisible by 12

**Supported Symmetries:**

- Cyclic: `c2`, `c3`, `c4`, etc.
- Dihedral: `d2`, `d3`, etc.
- Tetrahedral: `tetrahedral`
- Octahedral: `octahedral`
- Icosahedral: `icosahedral`

**Oligomer Contacts Potential**: To improve packing in oligomers:

```bash
potentials.guiding_potentials=["type:olig_contacts,weight_intra:1.0,weight_inter:0.5"]
potentials.guide_scale=2.0
potentials.guide_decay=quadratic
```

### 5. Structure Diversification (Partial Diffusion)

Create variations of an existing protein structure:

```bash
./scripts/run_rfdiffusion.sh diversify \
  inference.input_pdb=/inputs/starting_structure.pdb \
  'contigmap.contigs=[120-120]' \
  diffuser.partial_T=20 \
  inference.num_designs=10
```

**Important**: The contig length must exactly match the length of the input protein.

**Tuning Diversity**:

- `diffuser.partial_T`: Controls the noise level (diversity)
  - Lower values (10-15): Subtle variations, maintains overall fold
  - Medium values (20-25): Moderate diversity
  - Higher values (30+): Significant diversity, may alter fold

## Understanding the Contig Format

The `contigmap.contigs` argument defines the protein structure using a specialized format:

### Basic Patterns

| Pattern  | Explanation                              | Example                  |
| -------- | ---------------------------------------- | ------------------------ |
| `[N]`    | Fixed length N                           | `[150]`                  |
| `[N-M]`  | Length range from N to M                 | `[100-150]`              |
| `[N/M]`  | Sections N and M connected               | `[10-40/A163-181/10-40]` |
| `/0 `    | Chain break (space required)             | `[A1-100/0 B1-100]`      |
| `A10-25` | Residues 10-25 from chain A of input PDB | `[10-20/A10-25/10-20]`   |

### Visual Examples

```
[10-40/A163-181/10-40]
 ▲      ▲         ▲
 │      │         │
 │      │         └── Build 10-40 residues after motif
 │      └─────────── Use residues 163-181 from chain A of input PDB
 └─────────────── Build 10-40 residues before motif

[A163-181/0 100-100]
 ▲          ▲ ▲
 │          │ │
 │          │ └── Build a 100-residue protein
 │          └──── Chain break (separate chain)
 └───────────── Use residues 163-181 from chain A of input PDB as target
```

**Important**: Always check that your contig map references chains and residue numbers that actually exist in your PDB file. You can check available residues with:

```bash
grep "^ATOM" inputs/5TPN.pdb | awk '{print $5, $6}' | sort | uniq
```

## Model Checkpoints

RFdiffusion automatically selects the appropriate checkpoint for your task, but you can override with `inference.ckpt_override_path`.

| Checkpoint                  | Size  | Purpose                   | When to Use                                 |
| --------------------------- | ----- | ------------------------- | ------------------------------------------- |
| `Base_ckpt.pt`              | 1.5GB | Default model             | Unconditional generation, basic scaffolding |
| `Complex_base_ckpt.pt`      | 1.5GB | Binder design             | Standard binder design with hotspots        |
| `ActiveSite_ckpt.pt`        | 1.5GB | Small motif scaffolding   | When scaffolding small motifs (<5 residues) |
| `Complex_beta_ckpt.pt`      | 1.5GB | Diverse binder topologies | For binders with more β-sheet content       |
| `InpaintSeq_ckpt.pt`        | 1.5GB | Sequence inpainting       | When using `contigmap.inpaint_seq`          |
| `Complex_Fold_base_ckpt.pt` | 1.5GB | Fold-conditioned binders  | When using fold conditioning                |

## Advanced Options

### Sequence Inpainting

Hide sequence identity of specific residues when scaffolding:

```bash
./scripts/run_rfdiffusion.sh inpaint_seq \
  inference.input_pdb=/inputs/my_structure.pdb \
  'contigmap.contigs=[10-20/A10-25/10-20]' \
  'contigmap.inpaint_seq=[A15-20]' \
  inference.num_designs=5
```

### Auxiliary Potentials

Guide the diffusion process with additional constraints:

```bash
./scripts/run_rfdiffusion.sh with_potentials \
  'contigmap.contigs=[150-150]' \
  potentials.guiding_potentials=["type:monomer_ROG,weight:1.0"] \
  potentials.guide_scale=2.0 \
  potentials.guide_decay=quadratic \
  inference.num_designs=5
```

**Available Potentials:**

- `monomer_ROG`: Reduces radius of gyration (creates more compact proteins)
- `olig_contacts`: Optimizes inter-chain contacts in oligomers
- `substrate_contacts`: Optimizes contacts with substrate/target
- `helix_bias`: Encourages helical secondary structure
- `sheet_bias`: Encourages β-sheet/strand secondary structure

### Controlling Diffusion Parameters

```bash
# Reduce noise for better quality but less diversity
denoiser.noise_scale_ca=0.5 denoiser.noise_scale_frame=0.5

# Reduce timesteps for faster generation (default=50)
diffuser.T=20

# Stop diffusion early for faster generation
inference.final_step=0.2
```

## Output Files

For each design, RFdiffusion generates:

| File          | Description             | Notes                                                 |
| ------------- | ----------------------- | ----------------------------------------------------- |
| `*.pdb`       | Final protein structure | All designed regions are poly-glycine (no sidechains) |
| `*.trb`       | Metadata file           | Contains contig mapping, run parameters               |
| `/traj/*.pdb` | Trajectory files        | Visualize diffusion process in molecular viewers      |

**Important**: Since RFdiffusion only designs backbones, you'll need to use sequence design tools like ProteinMPNN to add amino acid sequences to the poly-glycine output.

## Troubleshooting

### Common Issues and Solutions

| Problem                  | Possible Cause         | Solution                                                       |
| ------------------------ | ---------------------- | -------------------------------------------------------------- |
| Container fails to start | GPU drivers issue      | Verify with `nvidia-smi` that GPUs are detected                |
| "CUDA out of memory"     | Protein too large      | Reduce input size or use GPU with more memory                  |
| Motif not preserved      | Motif too small        | Use `ActiveSite_ckpt.pt` checkpoint                            |
| Poor binder designs      | Unsuitable target site | Ensure target has hydrophobic residues, choose better hotspots |
| Incorrect symmetry       | Wrong contig length    | Ensure total length is divisible by number of subunits         |

### Error Messages Explained

| Error                                         | Explanation           | Fix                                              |
| --------------------------------------------- | --------------------- | ------------------------------------------------ |
| `ValueError: Contig string yielded 0 contigs` | Invalid contig format | Check contig syntax and enclosing quotes         |
| `FileNotFoundError: models/Base_ckpt.pt`      | Missing model files   | Run download script for models                   |
| `RuntimeError: No CUDA GPUs are available`    | GPU access issue      | Ensure `--gpus all` is included and drivers work |

## Performance Tips

1. **Optimize Input Size**:

   - Runtime scales as O(N²) where N is total residue count
   - Truncate large targets to just the region of interest

2. **Reduce Computation Time**:

   - Use fewer timesteps: `diffuser.T=20` (default=50)
   - Early stopping: `inference.final_step=0.2`

3. **Quality vs. Diversity**:
   - Reduce noise for better quality: `denoiser.noise_scale_ca=0.5 denoiser.noise_scale_frame=0.5`
   - Increase noise (up to 1.0) for more diversity

## Common Workflows

### Complete Binder Design Pipeline

1. **Prepare Target**:

   - Truncate target to binding region plus ~10Å buffer
   - Identify 3-6 potential hotspot residues at interface

2. **Generate Backbones** (start with 50-100):

   ```bash
   ./scripts/run_rfdiffusion.sh binder_v1 \
     inference.input_pdb=/inputs/target_truncated.pdb \
     'contigmap.contigs=[A163-181/0 100-100]' \
     'ppi.hotspot_res=[A170,A173,A174]' \
     denoiser.noise_scale_ca=0.5 \
     denoiser.noise_scale_frame=0.5 \
     inference.num_designs=50
   ```

3. **Design Sequences**:

   - Use ProteinMPNN to assign sequences to the poly-glycine backbones

4. **Evaluate Designs**:
   - Use AlphaFold2 to predict structure and assess binding interface
