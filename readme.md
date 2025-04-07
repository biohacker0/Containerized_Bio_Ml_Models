# Containerized Bio ML Models

A collection of containerized machine learning models for computational biology and bioinformatics research. This repository simplifies the installation, configuration, and deployment of state-of-the-art bio ML models, allowing researchers to focus on science rather than setup.

## Overview

Setting up computational biology models often requires navigating complex dependencies, environment configurations, and installation procedures that can take days to resolve. This project provides ready-to-use Docker containers for popular bio ML models with:

- Simple commands to run each model
- Consistent interfaces across different models
- Comprehensive documentation with practical examples
- Docker Compose integration for running multiple models (planned)

## Available Models

| Model       | Description                                    | Status      |
| ----------- | ---------------------------------------------- | ----------- |
| RFdiffusion | Protein structure generation with diffusion    | Available   |
| ESMFold     | Fast and accurate protein structure prediction | Coming soon |
| AlphaFold2  | State-of-the-art protein structure prediction  | Coming soon |
| ProteinMPNN | Protein sequence design                        | Coming soon |

## System Requirements

- Docker (latest version recommended)
- NVIDIA GPU with CUDA 11.6+ support
- NVIDIA Container Toolkit
- 16GB+ RAM recommended
- Storage space for model weights

## Quick Start

Each model has its own directory with specific setup instructions and documentation.

## Documentation

Each model has its own comprehensive documentation located in its directory:

- Additional models: Coming soon

## Contributing

Contributions are welcome! You can help by:

- Adding new containerized models
- Improving documentation and examples
- Reporting bugs or issues
