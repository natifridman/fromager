# Handling Multiple Versions

Sometimes you need to build multiple versions of the same package for different use cases, environments, or compatibility requirements. Fromager supports this through build variants and careful dependency management.

## Understanding Version Conflicts

### Common Scenarios

1. **Different Python versions** - Package built for Python 3.9 vs 3.11
2. **CUDA versions** - ML packages for different CUDA toolkit versions
3. **CPU vs GPU** - Same package optimized for different hardware
4. **Development vs Production** - Debug builds vs optimized builds
5. **Platform variations** - Windows vs Linux specific builds

## Using Build Variants

Build variants allow you to create different versions of packages with distinct configurations:

### Basic Variant Configuration

```yaml title="overrides/settings/torch.yaml"
variants:
  cpu:
    changelog:
      - "CPU-only build of PyTorch"
    build_options:
      env:
        USE_CUDA: "0"
        USE_DISTRIBUTED: "1"
        
  cuda121:
    changelog:
      - "CUDA 12.1 build of PyTorch"
    build_options:
      env:
        USE_CUDA: "1"
        CUDA_HOME: "/usr/local/cuda-12.1"
        TORCH_CUDA_ARCH_LIST: "7.0;7.5;8.0;8.6;9.0"

  cuda118:
    changelog:
      - "CUDA 11.8 build of PyTorch"  
    build_options:
      env:
        USE_CUDA: "1"
        CUDA_HOME: "/usr/local/cuda-11.8"
        TORCH_CUDA_ARCH_LIST: "7.0;7.5;8.0;8.6"
```

### Building Specific Variants

```bash
# Build CPU version
fromager --variant cpu bootstrap torch

# Build CUDA 12.1 version  
fromager --variant cuda121 bootstrap torch

# Build all variants
for variant in cpu cuda121 cuda118; do
  fromager --variant $variant bootstrap torch
done
```

## Version-Specific Requirements

### Requirements Files with Variants

Create separate requirements files for different scenarios:

```text title="requirements-cpu.txt"
torch==2.1.0+cpu
torchvision==0.16.0+cpu
torchaudio==2.1.0+cpu
```

```text title="requirements-cuda121.txt"  
torch==2.1.0+cu121
torchvision==0.16.0+cu121
torchaudio==2.1.0+cu121
```

```text title="requirements-cuda118.txt"
torch==2.1.0+cu118
torchvision==0.16.0+cu118
torchaudio==2.1.0+cu118
```

### Constraint Files for Version Management

```text title="constraints-ml-cpu.txt"
# CPU-only ML stack
torch==2.1.0+cpu
tensorflow-cpu==2.13.0
numpy>=1.21.0,<2.0.0
```

```text title="constraints-ml-gpu.txt"  
# GPU-enabled ML stack
torch==2.1.0+cu121
tensorflow-gpu==2.13.0
numpy>=1.21.0,<2.0.0
```

## Directory Organization

Organize builds by variant to avoid conflicts:

```
work-dir/
├── cpu/
│   ├── wheels/
│   ├── work-dir/
│   └── logs/
├── cuda121/
│   ├── wheels/
│   ├── work-dir/
│   └── logs/
└── cuda118/
    ├── wheels/
    ├── work-dir/
    └── logs/
```

### Automated Multi-Variant Builds

```bash title="scripts/build-all-variants.sh"
#!/bin/bash

VARIANTS=("cpu" "cuda121" "cuda118")
BASE_DIR="$(pwd)"

for variant in "${VARIANTS[@]}"; do
    echo "Building variant: $variant"
    
    # Create variant-specific directories
    mkdir -p "build-${variant}"/{wheels,work-dir,logs}
    
    # Build with variant-specific settings
    fromager \
        --variant "$variant" \
        --logs-dir "build-${variant}/logs" \
        bootstrap \
        --requirements-file "requirements-${variant}.txt" \
        --constraints-file "constraints-${variant}.txt" \
        --work-dir "build-${variant}/work-dir" \
        --wheel-server-dir "build-${variant}/wheels"
        
    echo "Completed variant: $variant"
done

echo "All variants built successfully!"
```

## Version-Specific Package Settings

### Conditional Settings Based on Variants

```yaml title="overrides/settings/numpy.yaml"
variants:
  default:
    changelog:
      - "Standard NumPy build"
    build_options:
      env:
        NPY_NUM_BUILD_JOBS: "4"
        
  mkl:
    changelog:
      - "NumPy with Intel MKL optimization"
    build_options:
      env:
        NPY_NUM_BUILD_JOBS: "4"
        USE_MKL: "1"
        MKL_ROOT: "/opt/intel/mkl"
        
  openblas:
    changelog:
      - "NumPy with OpenBLAS backend"
    build_options:
      env:
        NPY_NUM_BUILD_JOBS: "4"
        NPY_USE_BLAS_ILP64: "1"
        OPENBLAS_ROOT: "/usr/local/openblas"
```

### Python Version Specific Builds

```yaml title="overrides/settings/package.yaml"
variants:
  py39:
    changelog:
      - "Built for Python 3.9"
    build_options:
      python_version: "3.9"
      
  py311:
    changelog:
      - "Built for Python 3.11"
    build_options:
      python_version: "3.11"
      
  py312:
    changelog:
      - "Built for Python 3.12"
    build_options:
      python_version: "3.12"
```

## Managing Version Dependencies

### Cross-Variant Dependencies

Sometimes variants of one package depend on specific variants of others:

```yaml title="overrides/settings/torchvision.yaml"
variants:
  cpu:
    dependencies:
      torch: "cpu"
    changelog:
      - "TorchVision CPU build depending on PyTorch CPU"
      
  cuda121:
    dependencies:
      torch: "cuda121"
    changelog:
      - "TorchVision CUDA 12.1 build"
      
  cuda118:
    dependencies:
      torch: "cuda118"
    changelog:
      - "TorchVision CUDA 11.8 build"
```

### Version Resolution Strategies

```yaml title="overrides/settings.yaml"
changelog:
  - "Global settings for multi-version handling"

# Default variant selection
default_variant: "cpu"

# Variant compatibility matrix
variant_compatibility:
  cuda121:
    python_version: ">=3.8,<3.12"
    cuda_version: "12.1"
    
  cuda118:
    python_version: ">=3.7,<3.11"  
    cuda_version: "11.8"
    
  cpu:
    python_version: ">=3.8,<3.13"
```

## Advanced Multi-Version Patterns

### Matrix Builds

Build all combinations of variants:

```python title="scripts/matrix-build.py"
#!/usr/bin/env python3
"""Build all variant combinations."""

import subprocess
import itertools
from pathlib import Path

# Define variant dimensions
PYTHON_VERSIONS = ["3.9", "3.10", "3.11"]
CUDA_VERSIONS = ["cpu", "cuda118", "cuda121"]
PACKAGES = ["torch", "torchvision", "torchaudio"]

def build_matrix():
    """Build all combinations."""
    
    for py_ver, cuda_ver in itertools.product(PYTHON_VERSIONS, CUDA_VERSIONS):
        variant_name = f"py{py_ver.replace('.', '')}-{cuda_ver}"
        
        print(f"Building variant: {variant_name}")
        
        # Create requirements for this combination
        requirements = []
        for package in PACKAGES:
            if cuda_ver == "cpu":
                requirements.append(f"{package}==2.1.0+cpu")
            else:
                requirements.append(f"{package}==2.1.0+{cuda_ver}")
        
        # Write requirements file
        req_file = f"requirements-{variant_name}.txt"
        with open(req_file, "w") as f:
            f.write("\n".join(requirements))
        
        # Build this variant
        subprocess.run([
            "fromager",
            "--variant", variant_name,
            "bootstrap",
            "--requirements-file", req_file,
            "--wheel-server-dir", f"wheels-{variant_name}"
        ], check=True)

if __name__ == "__main__":
    build_matrix()
```

### Selective Version Building

Build only compatible combinations:

```python title="scripts/selective-build.py"
#!/usr/bin/env python3
"""Build only valid variant combinations."""

import json

# Define compatibility matrix
COMPATIBILITY = {
    "torch": {
        "2.1.0+cpu": {"python": ["3.8", "3.9", "3.10", "3.11"]},
        "2.1.0+cu118": {"python": ["3.8", "3.9", "3.10", "3.11"]},  
        "2.1.0+cu121": {"python": ["3.8", "3.9", "3.10", "3.11"]},
    }
}

def is_compatible(package, version, python_version):
    """Check if package version is compatible with Python version."""
    if package not in COMPATIBILITY:
        return True
        
    pkg_versions = COMPATIBILITY[package]
    if version not in pkg_versions:
        return False
        
    return python_version in pkg_versions[version]["python"]

def build_compatible_versions():
    """Build only compatible combinations."""
    
    python_versions = ["3.8", "3.9", "3.10", "3.11"]
    torch_versions = ["2.1.0+cpu", "2.1.0+cu118", "2.1.0+cu121"]
    
    for py_ver in python_versions:
        for torch_ver in torch_versions:
            if is_compatible("torch", torch_ver, py_ver):
                variant = f"py{py_ver.replace('.', '')}-{torch_ver.split('+')[1]}"
                print(f"Building compatible variant: {variant}")
                # ... build logic here

if __name__ == "__main__":
    build_compatible_versions()
```

## Testing Multiple Versions

### Automated Testing

```bash title="scripts/test-variants.sh"
#!/bin/bash

VARIANTS=("cpu" "cuda121" "cuda118")

for variant in "${VARIANTS[@]}"; do
    echo "Testing variant: $variant"
    
    # Install wheels from variant build
    pip install --force-reinstall \
        --find-links "build-${variant}/wheels" \
        --no-index torch torchvision
    
    # Run variant-specific tests
    python -c "
import torch
print(f'PyTorch version: {torch.__version__}')
print(f'CUDA available: {torch.cuda.is_available()}')

if '$variant' != 'cpu' and torch.cuda.is_available():
    print(f'CUDA version: {torch.version.cuda}')
    print(f'GPU count: {torch.cuda.device_count()}')
"
    
    echo "Variant $variant test completed"
done
```

### Continuous Integration Matrix

```yaml title=".github/workflows/multi-version.yml"
name: Multi-Version Builds
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: [3.9, 3.10, 3.11]
        variant: [cpu, cuda118, cuda121]
        exclude:
          # CUDA 12.1 not supported on Python 3.9
          - python-version: 3.9
            variant: cuda121
            
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
          
      - name: Install fromager
        run: pip install fromager
        
      - name: Build variant
        run: |
          fromager \
            --variant ${{ matrix.variant }} \
            bootstrap torch
            
      - name: Test build
        run: |
          pip install --find-links wheels torch
          python -c "import torch; print(torch.__version__)"
```

## Best Practices

1. **Clear naming conventions** - Use consistent variant names across packages
2. **Compatibility matrices** - Document which versions work together
3. **Separate build directories** - Avoid conflicts between variants
4. **Automated testing** - Verify each variant works correctly
5. **Resource planning** - Consider storage and compute requirements
6. **Version pinning** - Pin compatible versions across the stack
7. **Documentation** - Clearly document what each variant provides
8. **CI/CD integration** - Automate multi-version builds and testing

Managing multiple versions requires careful planning and organization, but provides the flexibility to support diverse deployment scenarios and user requirements.
