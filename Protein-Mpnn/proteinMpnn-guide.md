# ProteinMPNN: Complete Setup Guide

This containerized version of ProteinMPNN, a neural network for protein sequence design developed by the Dauparas lab, lets you generate amino acid sequences for given protein backbones with a single command. Use it for monomer design, multi-chain complexes, symmetric designs, and more.

This setup is cleaner than the official repository, saving you from dependency hell so you can focus on research.

## Table of Contents

- [Container Image](#container-image)
- [Setup Scripts](#setup-scripts)
- [System Requirements](#system-requirements)
- [Basic Usage](#basic-usage)
- [Task-Specific Examples](#task-specific-examples)
  - [Monomer Design](#monomer-design)
  - [Multi-Chain Design](#multi-chain-design)
  - [Fixed Residue Design](#fixed-residue-design)
  - [Symmetric Design](#symmetric-design)
  - [Batch Processing Multiple PDBs](#batch-processing-multiple-pdbs)
- [Model Weights](#model-weights)
- [Advanced Options](#advanced-options)
- [Output Files](#output-files)
- [Troubleshooting](#troubleshooting)
- [Performance Tips](#performance-tips)

## Container Image

Pull the image from GitHub Container Registry:

```bash
docker pull ghcr.io/biohacker0/protein_mpnn-new:local
```

## Setup Scripts

Save the following script as setup_proteinmpnn.sh in your home directory to automate setup:

```bash
#!/bin/bash
# Setup script for ProteinMPNN

# Configuration
IMAGE_NAME="protein_mpnn_new:local"
GHCR_IMAGE="ghcr.io/biohacker0/protein_mpnn-new:local"

# Create directory structure
mkdir -p $HOME/protein_mpnn/{inputs,outputs,models,scripts}
cd $HOME/protein_mpnn

# Download sample PDB file
echo "Downloading sample PDB file to inputs directory..."
wget -P $HOME/protein_mpnn/inputs https://files.rcsb.org/download/5TPN.pdb

# Pull image
if ! docker image inspect $IMAGE_NAME >/dev/null 2>&1; then
  echo "Pulling $GHCR_IMAGE..."
  docker pull $GHCR_IMAGE
  docker tag $GHCR_IMAGE $IMAGE_NAME
fi

# Create models download script
cat > scripts/download_models.sh << 'EOF'
#!/bin/bash
MODELS_DIR=${1:-"$HOME/protein_mpnn/models"}
mkdir -p $MODELS_DIR
cd $MODELS_DIR

echo "Downloading ProteinMPNN model weights to $MODELS_DIR..."

# Vanilla model weights
wget -c https://raw.githubusercontent.com/CyrusBiotechnology/ProteinMPNN-docker/main/vanilla_model_weights/v_48_020.pt

read -p "Download additional models (soluble, CA-only)? (y/n) " answer
if [[ $answer == "y" ]]; then
  # Vanilla models
  wget -c https://raw.githubusercontent.com/CyrusBiotechnology/ProteinMPNN-docker/main/vanilla_model_weights/v_48_002.pt
  wget -c https://raw.githubusercontent.com/CyrusBiotechnology/ProteinMPNN-docker/main/vanilla_model_weights/v_48_010.pt
  wget -c https://raw.githubusercontent.com/CyrusBiotechnology/ProteinMPNN-docker/main/vanilla_model_weights/v_48_030.pt
  # Soluble models
  wget -c https://raw.githubusercontent.com/dauparas/ProteinMPNN/main/soluble_model_weights/v_48_002.pt
  wget -c https://raw.githubusercontent.com/dauparas/ProteinMPNN/main/soluble_model_weights/v_48_010.pt
  wget -c https://raw.githubusercontent.com/dauparas/ProteinMPNN/main/soluble_model_weights/v_48_020.pt
  wget -c https://raw.githubusercontent.com/dauparas/ProteinMPNN/main/soluble_model_weights/v_48_030.pt
  # CA-only models
  wget -c https://raw.githubusercontent.com/dauparas/ProteinMPNN/main/ca_model_weights/v_48_002.pt
  wget -c https://raw.githubusercontent.com/dauparas/ProteinMPNN/main/ca_model_weights/v_48_010.pt
  wget -c https://raw.githubusercontent.com/dauparas/ProteinMPNN/main/ca_model_weights/v_48_020.pt
fi

echo "Model download complete!"
EOF

# Create run wrapper script
cat > scripts/run_proteinmpnn.sh << 'EOF'
#!/bin/bash
# Run ProteinMPNN with standard paths

MODELS_DIR=${MODELS_DIR:-"$HOME/protein_mpnn/models"}
INPUTS_DIR=${INPUTS_DIR:-"$HOME/protein_mpnn/inputs"}
OUTPUTS_DIR=${OUTPUTS_DIR:-"$HOME/protein_mpnn/outputs"}
OUTPUT_NAME=${1:-"mpnn_output"}
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
  protein_mpnn_new:local \
  --path_to_model_weights /models \
  --out_folder /outputs/$OUTPUT_NAME \
  "$@"

echo "Results saved to $OUTPUTS_DIR/$OUTPUT_NAME"
EOF

# Create batch processing script
cat > scripts/batch_proteinmpnn.sh << 'EOF'
#!/bin/bash
# Batch process multiple PDB files with ProteinMPNN

INPUTS_DIR=${INPUTS_DIR:-"$HOME/protein_mpnn/inputs"}
OUTPUTS_DIR=${OUTPUTS_DIR:-"$HOME/protein_mpnn/outputs"}
MODELS_DIR=${MODELS_DIR:-"$HOME/protein_mpnn/models"}

for pdb_file in $INPUTS_DIR/*.pdb; do
  if [ -f "$pdb_file" ]; then
    output_name=$(basename "$pdb_file" .pdb)
    echo "Processing $pdb_file..."
    ./run_proteinmpnn.sh "$output_name" --pdb_path "/inputs/$(basename "$pdb_file")" "$@"
  fi
done
EOF

# Create examples script
cat > scripts/examples.sh << 'EOF'
#!/bin/bash
# ProteinMPNN examples

function show_examples() {
  echo "======================================"
  echo "ProteinMPNN Example Commands"
  echo "======================================"
  echo
  echo "1. Design sequences for a monomer (5TPN.pdb):"
  echo "./scripts/run_proteinmpnn.sh monomer --pdb_path /inputs/5TPN.pdb --num_seq_per_target 5 --sampling_temp '0.1'"
  echo
  echo "2. Design a multi-chain complex (specify chains):"
  echo "./scripts/run_proteinmpnn.sh multichain --pdb_path /inputs/5TPN.pdb --pdb_path_chains 'A' --num_seq_per_target 5"
  echo
  echo "3. Fix specific residues (requires JSONL file):"
  echo "./scripts/run_proteinmpnn.sh fixed --pdb_path /inputs/5TPN.pdb --fixed_positions_jsonl /inputs/fixed.jsonl --num_seq_per_target 5"
  echo
  echo "4. Design with soluble model weights:"
  echo "./scripts/run_proteinmpnn.sh soluble --pdb_path /inputs/5TPN.pdb --use_soluble_model --num_seq_per_target 5"
  echo
  echo "5. Process multiple PDB files in a directory:"
  echo "./scripts/batch_proteinmpnn.sh --num_seq_per_target 5 --sampling_temp '0.1'"
  echo
  echo "Run these commands from your protein_mpnn directory."
  echo "Note: Ensure PDB files are in the inputs directory and model weights are in the models directory."
}

show_examples
EOF

# Make scripts executable
chmod +x scripts/*.sh

echo "ProteinMPNN setup complete!"
echo "Run ./scripts/download_models.sh to download model weights."
echo "Sample PDB file (5TPN.pdb) has been downloaded to the inputs directory."
echo "See ./scripts/examples.sh for usage examples."
```

Make it executable and run:

```bash
chmod +x setup_proteinmpnn.sh
./setup_proteinmpnn.sh
cd ~/protein_mpnn
./scripts/download_models.sh
./scripts/examples.sh
```

## System Requirements

- Docker: Latest stable version
- GPU: NVIDIA GPU with CUDA 11.6+ support
- NVIDIA Container Toolkit: Required for GPU access
- Memory: 8GB RAM minimum, 16GB+ recommended
- GPU Memory: 4GB for small proteins, 8GB+ for larger complexes
- Storage: ~2GB for model weights and container, plus space for outputs

Check Your Setup:

```bash
# Verify Docker
docker --version

# Verify NVIDIA drivers
nvidia-smi

# Verify NVIDIA Container Toolkit
docker run --rm --gpus all nvidia/cuda:11.6.2-base-ubuntu20.04 nvidia-smi
```

## Basic Usage

Command Structure:

```bash
docker run -it --rm --gpus all \
  -v /path/to/models:/models:ro \
  -v /path/to/inputs:/inputs:ro \
  -v /path/to/outputs:/outputs:rw \
  protein_mpnn_new:local \
  --path_to_model_weights /models \
  --out_folder /outputs/my_output \
  --pdb_path /inputs/my_pdb.pdb \
  [additional arguments...]
```

Essential Arguments:

| Argument                | Description                                    | Example            | Required?                |
| ----------------------- | ---------------------------------------------- | ------------------ | ------------------------ |
| --path_to_model_weights | Path to model weights directory                | /models            | Yes                      |
| --out_folder            | Output directory for sequences                 | /outputs/test      | Yes                      |
| --pdb_path              | Input PDB file path                            | /inputs/my_pdb.pdb | Yes (unless using JSONL) |
| --num_seq_per_target    | Number of sequences to generate                | 5                  | Optional (default=1)     |
| --sampling_temp         | Sampling temperature (higher = more diversity) | 0.1                | Optional (default=0.1)   |

Using the Wrapper Script:

```bash
# Format: ./scripts/run_proteinmpnn.sh OUTPUT_NAME [arguments...]
./scripts/run_proteinmpnn.sh my_design --pdb_path /inputs/5TPN.pdb --num_seq_per_target 5
```

Batch Processing:
To process multiple PDBs:

```bash
./scripts/batch_proteinmpnn.sh --num_seq_per_target 5 --sampling_temp "0.1"
```

## Task-Specific Examples

### Monomer Design

Generate sequences for a single protein chain:

```bash
./scripts/run_proteinmpnn.sh monomer --pdb_path /inputs/5TPN.pdb --num_seq_per_target 5 --sampling_temp "0.1"
```

### Multi-Chain Design

Design sequences for specific chains:

```bash
./scripts/run_proteinmpnn.sh multichain --pdb_path /inputs/5TPN.pdb --pdb_path_chains "A" --num_seq_per_target 5
```

### Fixed Residue Design

Fix specific residues (requires a JSONL file, see [Advanced Options](#advanced-options)):

```bash
./scripts/run_proteinmpnn.sh fixed --pdb_path /inputs/5TPN.pdb --fixed_positions_jsonl /inputs/fixed.jsonl --num_seq_per_target 5
```

### Symmetric Design

Tie residues for symmetric design (requires a JSONL file):

```bash
./scripts/run_proteinmpnn.sh symmetric --pdb_path /inputs/5TPN.pdb --tied_positions_jsonl /inputs/tied.jsonl --num_seq_per_target 5
```

### Score-Only Mode

Score an existing protein-sequence pair without generating new sequences:

```bash
./scripts/run_proteinmpnn.sh score_only --pdb_path /inputs/5TPN.pdb --score_only 1
```

### Score from FASTA

Score using a provided FASTA sequence:

```bash
./scripts/run_proteinmpnn.sh score_fasta --pdb_path /inputs/5TPN.pdb --path_to_fasta "/inputs/sequence.fasta" --score_only 1
```

### Homooligomer Design

Design a homooligomer with tied chains:

```bash
./scripts/run_proteinmpnn.sh homooligomer --pdb_path /inputs/homooligomer.pdb --tied_positions_jsonl /inputs/tied_chains.jsonl
```

### PSSM-guided Design

Use position-specific scoring matrix (PSSM) to guide design:

```bash
./scripts/run_proteinmpnn.sh pssm_guided --pdb_path /inputs/5TPN.pdb --pssm_jsonl /inputs/pssm.jsonl --pssm_multi 0.5
```

### Batch Processing Multiple PDBs

Process all PDBs in the inputs directory:

```bash
./scripts/batch_proteinmpnn.sh --num_seq_per_target 5 --sampling_temp "0.1"
```

## Model Weights

ProteinMPNN requires pre-trained model weights:

### Available Models:

- **Vanilla Models**: General-purpose protein design

  - `v_48_002.pt`, `v_48_010.pt`, `v_48_020.pt`, `v_48_030.pt`
  - Default is `v_48_020.pt`
  - The number after `v_48_` indicates noise level (e.g., `v_48_010` = 0.10Å noise)

- **Soluble Models**: Specifically for soluble proteins

  - `v_48_002.pt`, `v_48_010.pt`, `v_48_020.pt`, `v_48_030.pt` in the soluble_model_weights directory
  - Enable with `--use_soluble_model`

- **CA-only Models**: For coarse-grained structures with only alpha carbons
  - `v_48_002.pt`, `v_48_010.pt`, `v_48_020.pt` in the ca_model_weights directory
  - Enable with `--ca_only`

Run `./scripts/download_models.sh` to download weights. By default, only `v_48_020.pt` (vanilla) is downloaded, with an option to download additional models.

## Advanced Options

All command line arguments from the original ProteinMPNN:

```
--suppress_print          0 for False, 1 for True
--ca_only                 Parse CA-only structures and use CA-only models (default: false)
--path_to_model_weights   Path to model weights folder
--model_name              ProteinMPNN model name: v_48_002, v_48_010, v_48_020, v_48_030
--use_soluble_model       Flag to load ProteinMPNN weights trained on soluble proteins only
--seed                    If set to 0 then a random seed will be picked
--save_score              0 for False, 1 for True; save score=-log_prob to npy files
--path_to_fasta           Score provided input sequence in fasta format (e.g. GGGGGG/PPPPS/WWW for chains A, B, C)
--save_probs              0 for False, 1 for True; save MPNN predicted probabilities per position
--score_only              0 for False, 1 for True; score input backbone-sequence pairs
--conditional_probs_only  0 for False, 1 for True; output conditional probabilities p(s_i given the rest)
--conditional_probs_only_backbone  0 for False, 1 for True; output conditional probabilities p(s_i given backbone)
--unconditional_probs_only  0 for False, 1 for True; output unconditional probabilities p(s_i given backbone)
--backbone_noise          Standard deviation of Gaussian noise to add to backbone atoms
--num_seq_per_target      Number of sequences to generate per target
--batch_size              Batch size; reduce if running out of GPU memory
--max_length              Max sequence length
--sampling_temp           Sampling temperature for amino acids (e.g. "0.1", "0.2 0.25 0.5")
--out_folder              Path to output folder
--pdb_path                Path to a single PDB to be designed
--pdb_path_chains         Define which chains need to be designed for a single PDB
--jsonl_path              Path to folder with parsed pdb into jsonl
--chain_id_jsonl          Path to dictionary specifying which chains to design/fix
--fixed_positions_jsonl   Path to dictionary with fixed positions: {"A": [10, 20]}
--omit_AAs                Specify which amino acids to omit (e.g. 'AC' omits alanine and cystine)
--bias_AA_jsonl           Path to dictionary for AA composition bias (e.g. {"A": -1.1, "F": 0.7})
--bias_by_res_jsonl       Path to dictionary with per position bias
--omit_AA_jsonl           Path to dictionary specifying amino acids to omit at specific positions
--pssm_jsonl              Path to dictionary with PSSM
--pssm_multi              Value between [0.0, 1.0], 0.0 means do not use PSSM, 1.0 ignores MPNN predictions
--pssm_threshold          Value between -inf + inf to restrict per position AAs
--pssm_log_odds_flag      0 for False, 1 for True
--pssm_bias_flag          0 for False, 1 for True
--tied_positions_jsonl    Path to dictionary with tied positions: {"A": [[10, 20]]} to tie positions 10 and 20
```

Common example use cases:

1. **Soluble Protein Design**: `--use_soluble_model`
2. **CA-only Design**: `--ca_only` (for coarse-grained structures)
3. **Fixed Residues**: `--fixed_positions_jsonl /inputs/fixed.jsonl`
4. **Position Symmetry**: `--tied_positions_jsonl /inputs/tied.jsonl`
5. **Amino Acid Bias**: `--bias_AA_jsonl /inputs/bias.jsonl`
6. **PSSM Guidance**: `--pssm_jsonl /inputs/pssm.jsonl --pssm_multi 0.5`

See the official ProteinMPNN repo for detailed JSONL file formats.

## Output Files

Outputs are saved in --out_folder (e.g., /outputs/my_output):

- Sequence files: FASTA files (seqs.fa) with designed sequences.
- Score files: If --save_score=1, scores in .npy format.
- Probability files: If --save_probs=1, per-position probabilities.

Example seqs.fa:

```
>3HTN, score=1.1705, global_score=1.2045, designed_chains=['A'], model_name=v_48_020
NMYSYKKIGNKYIV
```

Output header fields explained:

- `score` - Average over designed residues negative log probability of sampled amino acids
- `global_score` - Average over all residues in all chains negative log probability of sampled/fixed amino acids
- `fixed_chains` - Chains that were not designed (fixed)
- `designed_chains` - Chains that were redesigned
- `model_name` - Model name that was used to generate results, e.g. `v_48_020`
- `git_hash` - GitHub version that was used to generate outputs
- `seed` - Random seed
- `T=0.1` - Temperature used to sample sequences (from sampling_temp parameter)
- `sample` - Sequence sample number (1, 2, 3...)

## Troubleshooting

- IsADirectoryError: Ensure --pdb_path points to a single PDB file. Use batch_proteinmpnn.sh for multiple files.
- GPU Memory Issues: Add --batch_size 1 to reduce memory usage.
- Missing Weights: Run ./scripts/download_models.sh and verify /models contains weights.
- Invalid PDB: Check PDB file format with grep "^ATOM" inputs/5TPN.pdb.
- ModuleNotFoundError: Rebuild the image with the latest Dockerfile if numpy or biopython errors occur.

## Performance Tips

- Use --batch_size > 1 on high-memory GPUs (16GB+).
- Download only necessary weights (e.g., v_48_020.pt) to save space.
- Pre-validate PDB files for complex tasks.
