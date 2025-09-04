# Ensuring Repeatable Builds

Repeatable builds are crucial for production deployments, debugging, and compliance. Fromager provides several mechanisms to ensure that you can reproduce the exact same wheels across different machines and time periods.

## Understanding Build Reproducibility

### Why Reproducibility Matters

1. **Security** - Verify that builds haven't been tampered with
2. **Debugging** - Reproduce issues in different environments
3. **Compliance** - Meet regulatory requirements for software traceability
4. **Quality Assurance** - Ensure consistent behavior across deployments

### Factors Affecting Reproducibility

- **Timestamps** in built files
- **Build environment variations** (paths, environment variables)
- **Dependency version resolution** changes over time
- **Random elements** in build processes
- **System-specific paths** and configurations

## Version Pinning Strategies

### Complete Version Locking

Create comprehensive constraint files:

```text title="constraints-locked.txt"
# Core dependencies with exact versions
setuptools==69.2.0
wheel==0.42.0
pip==24.0

# Application dependencies
requests==2.31.0
urllib3==2.0.7
certifi==2024.2.2
charset-normalizer==3.3.2
idna==3.6

# Build dependencies
build==1.0.3
packaging==24.0
pyproject-hooks==1.0.0
tomli==2.0.1
```

### Version Ranges with Upper Bounds

Balance stability with security updates:

```text title="constraints-stable.txt"
# Allow patch updates but pin major.minor
requests>=2.31.0,<2.32.0
numpy>=1.26.0,<1.27.0
pandas>=2.1.0,<2.2.0

# Pin build tools exactly
setuptools==69.2.0
wheel==0.42.0
```

### Automated Version Locking

Generate locked versions from successful builds:

```python title="scripts/generate-lockfile.py"
#!/usr/bin/env python3
"""Generate locked constraint file from successful build."""

import json
import subprocess
from pathlib import Path

def generate_lockfile(build_log_dir):
    """Extract exact versions from build logs."""
    
    # Parse build logs to extract versions
    built_packages = {}
    
    log_dir = Path(build_log_dir)
    for log_file in log_dir.glob("*-build.log"):
        package_name = log_file.stem.replace("-build", "")
        
        # Extract version from log (implementation depends on log format)
        with open(log_file) as f:
            content = f.read()
            # Parse version from build output
            # This is simplified - actual parsing depends on log format
            for line in content.split('\n'):
                if f"Successfully built {package_name}" in line:
                    # Extract version
                    version = line.split()[-1]
                    built_packages[package_name] = version
                    break
    
    # Generate constraints file
    with open("constraints-locked.txt", "w") as f:
        f.write("# Auto-generated locked constraints\n")
        f.write(f"# Generated from build logs in {build_log_dir}\n\n")
        
        for package, version in sorted(built_packages.items()):
            f.write(f"{package}=={version}\n")
    
    print(f"Generated constraints-locked.txt with {len(built_packages)} packages")

if __name__ == "__main__":
    import sys
    build_log_dir = sys.argv[1] if len(sys.argv) > 1 else "logs"
    generate_lockfile(build_log_dir)
```

## Build Environment Standardization

### Container-Based Builds

Use containers for consistent build environments:

```dockerfile title="Dockerfile.build"
FROM python:3.11.8-slim

# Install system dependencies with exact versions
RUN apt-get update && apt-get install -y \
    gcc=4:12.2.0-3ubuntu1 \
    g++=4:12.2.0-3ubuntu1 \
    make=4.3-4.1build1 \
    git=1:2.39.2-1ubuntu1 \
    && rm -rf /var/lib/apt/lists/*

# Set reproducible environment variables
ENV PYTHONHASHSEED=0
ENV SOURCE_DATE_EPOCH=1640995200
ENV BUILD_DATE=2022-01-01T00:00:00Z

# Install exact Python toolchain
RUN pip install --no-cache-dir \
    setuptools==69.2.0 \
    wheel==0.42.0 \
    build==1.0.3

WORKDIR /build
```

### Reproducible Environment Setup

```bash title="scripts/setup-reproducible-env.sh"
#!/bin/bash
set -euo pipefail

# Set reproducible environment variables
export PYTHONHASHSEED=0
export SOURCE_DATE_EPOCH="1640995200"  # 2022-01-01 00:00:00 UTC
export LANG=C.UTF-8
export LC_ALL=C.UTF-8

# Disable pip cache and randomization
export PIP_NO_CACHE_DIR=1
export PIP_DISABLE_PIP_VERSION_CHECK=1

# Set consistent paths
export HOME="/tmp/build-home"
export XDG_CACHE_HOME="$HOME/.cache"

# Create consistent directory structure
mkdir -p "$HOME"

# Install exact tool versions
pip install --no-cache-dir \
    setuptools==69.2.0 \
    wheel==0.42.0 \
    fromager==0.62.0

echo "Reproducible environment configured"
```

## Build Configuration Management

### Reproducible Build Settings

```yaml title="overrides/settings-reproducible.yaml"
changelog:
  - "Reproducible build configuration"
  - "All timestamps normalized to SOURCE_DATE_EPOCH"

# Global reproducibility settings
build_options:
  env:
    # Normalize timestamps
    SOURCE_DATE_EPOCH: "1640995200"
    
    # Disable randomization
    PYTHONHASHSEED: "0"
    
    # Consistent locale
    LANG: "C.UTF-8"
    LC_ALL: "C.UTF-8"
    
    # Disable caching
    PIP_NO_CACHE_DIR: "1"
    SETUPTOOLS_DISABLE_NORMALIZATION: "1"

# Per-package reproducibility
packages:
  numpy:
    build_options:
      env:
        # Ensure deterministic BLAS linking
        NPY_NUM_BUILD_JOBS: "1"
        NPY_ENABLE_CPU_FEATURES: "none"
        
  torch:
    build_options:
      env:
        # Disable CUDA randomization
        PYTORCH_DETERMINISTIC_BUILD: "1"
        
  # Packages with timestamp issues
  wheel:
    build_options:
      post_build_commands:
        - "find . -name '*.whl' -exec python -m wheel unpack {} +"
        - "find . -name '*.whl-info' -exec touch -d '@${SOURCE_DATE_EPOCH}' {} +"
        - "find . -name '*.whl-info' -exec python -m wheel pack {} +"
```

### Hash Verification

Track file hashes for verification:

```python title="scripts/verify-reproducible.py"
#!/usr/bin/env python3
"""Verify build reproducibility by comparing hashes."""

import hashlib
import json
import zipfile
from pathlib import Path

def calculate_wheel_hash(wheel_path):
    """Calculate reproducible hash of wheel contents."""
    
    hashes = {}
    
    with zipfile.ZipFile(wheel_path) as zf:
        for info in sorted(zf.infolist(), key=lambda x: x.filename):
            # Skip timestamp-sensitive files
            if info.filename.endswith(('.pyc', '.pyo')):
                continue
                
            # Read file content
            content = zf.read(info.filename)
            
            # Calculate hash
            file_hash = hashlib.sha256(content).hexdigest()
            hashes[info.filename] = file_hash
    
    # Calculate overall wheel hash
    combined = json.dumps(hashes, sort_keys=True)
    wheel_hash = hashlib.sha256(combined.encode()).hexdigest()
    
    return wheel_hash, hashes

def verify_builds(build1_dir, build2_dir):
    """Compare two build outputs for reproducibility."""
    
    build1_wheels = list(Path(build1_dir).glob("*.whl"))
    build2_wheels = list(Path(build2_dir).glob("*.whl"))
    
    # Match wheels by name (ignoring timestamps in filename)
    wheel_pairs = []
    for w1 in build1_wheels:
        # Find matching wheel in build2
        base_name = "-".join(w1.stem.split("-")[:-1])  # Remove build number
        matching = [w2 for w2 in build2_wheels if base_name in w2.stem]
        if matching:
            wheel_pairs.append((w1, matching[0]))
    
    results = {}
    for w1, w2 in wheel_pairs:
        hash1, files1 = calculate_wheel_hash(w1)
        hash2, files2 = calculate_wheel_hash(w2)
        
        package_name = w1.stem.split("-")[0]
        results[package_name] = {
            "reproducible": hash1 == hash2,
            "hash1": hash1,
            "hash2": hash2,
            "differing_files": []
        }
        
        # Find differing files
        if hash1 != hash2:
            for filename in set(files1.keys()) | set(files2.keys()):
                if files1.get(filename) != files2.get(filename):
                    results[package_name]["differing_files"].append(filename)
    
    return results

def main():
    import sys
    
    if len(sys.argv) != 3:
        print("Usage: verify-reproducible.py <build1_dir> <build2_dir>")
        sys.exit(1)
    
    build1_dir, build2_dir = sys.argv[1], sys.argv[2]
    results = verify_builds(build1_dir, build2_dir)
    
    print("Reproducibility Verification Results:")
    print("=" * 40)
    
    reproducible_count = 0
    total_count = len(results)
    
    for package, result in results.items():
        status = "✓ REPRODUCIBLE" if result["reproducible"] else "✗ NON-REPRODUCIBLE"
        print(f"{package}: {status}")
        
        if result["reproducible"]:
            reproducible_count += 1
        else:
            print(f"  Differing files: {result['differing_files']}")
    
    print("=" * 40)
    print(f"Summary: {reproducible_count}/{total_count} packages reproducible")
    
    if reproducible_count == total_count:
        print("🎉 All packages are reproducible!")
        sys.exit(0)
    else:
        print("⚠️  Some packages are not reproducible")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

## Source Pinning and Archival

### Source Code Archival

Archive exact source versions:

```yaml title="overrides/settings/source-archival.yaml" 
changelog:
  - "Source code archival for reproducibility"

packages:
  torch:
    download_source:
      url: "https://github.com/pytorch/pytorch.git"
      git_options:
        commit: "a1b2c3d4e5f6789012345678901234567890abcd"
        archive: true  # Create tarball of exact commit
        
  numpy:
    download_source:
      url: "https://archive.example.com/numpy-1.26.0.tar.gz"
      sha256: "exact_hash_of_archived_source"
```

### Dependency Mirroring

Create local mirrors of dependencies:

```bash title="scripts/mirror-sources.sh"
#!/bin/bash

# Create local mirror of source distributions
MIRROR_DIR="source-mirror"
mkdir -p "$MIRROR_DIR"

# Download and verify source distributions
while read -r package version; do
    echo "Mirroring $package==$version"
    
    # Download source distribution
    pip download \
        --no-binary :all: \
        --dest "$MIRROR_DIR" \
        --no-deps \
        "$package==$version"
        
    # Calculate and store hash
    find "$MIRROR_DIR" -name "${package}-${version}*" -type f | while read -r file; do
        sha256sum "$file" >> "$MIRROR_DIR/checksums.sha256"
    done
    
done < requirements.txt

echo "Source mirror created in $MIRROR_DIR"
```

## Automated Reproducibility Testing

### CI/CD Reproducibility Checks

```yaml title=".github/workflows/reproducible-builds.yml"
name: Reproducible Builds Test

on:
  pull_request:
  schedule:
    - cron: '0 2 * * 0'  # Weekly on Sunday

jobs:
  test-reproducibility:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          
      - name: Install fromager
        run: pip install fromager
        
      - name: First build
        run: |
          ./scripts/setup-reproducible-env.sh
          fromager \
            --logs-dir build1/logs \
            bootstrap \
            --requirements-file requirements.txt \
            --constraints-file constraints-locked.txt \
            --work-dir build1/work \
            --wheel-server-dir build1/wheels
            
      - name: Second build (clean environment)
        run: |
          # Clean environment for second build
          rm -rf ~/.cache/pip
          ./scripts/setup-reproducible-env.sh
          
          fromager \
            --logs-dir build2/logs \
            bootstrap \
            --requirements-file requirements.txt \
            --constraints-file constraints-locked.txt \
            --work-dir build2/work \
            --wheel-server-dir build2/wheels
            
      - name: Verify reproducibility
        run: |
          python scripts/verify-reproducible.py \
            build1/wheels build2/wheels
            
      - name: Upload results
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: reproducibility-test-results
          path: |
            build1/wheels/
            build2/wheels/
            build*/logs/
```

### Regular Reproducibility Monitoring

```python title="scripts/monitor-reproducibility.py"
#!/usr/bin/env python3
"""Monitor reproducibility over time."""

import json
import subprocess
import tempfile
from datetime import datetime
from pathlib import Path

def run_reproducibility_test():
    """Run reproducibility test and return results."""
    
    with tempfile.TemporaryDirectory() as tmpdir:
        tmpdir = Path(tmpdir)
        
        # Run two builds
        for build_num in [1, 2]:
            build_dir = tmpdir / f"build{build_num}"
            build_dir.mkdir()
            
            subprocess.run([
                "fromager",
                "--logs-dir", str(build_dir / "logs"),
                "bootstrap",
                "--requirements-file", "requirements.txt",
                "--constraints-file", "constraints-locked.txt",
                "--work-dir", str(build_dir / "work"),
                "--wheel-server-dir", str(build_dir / "wheels")
            ], check=True)
        
        # Verify reproducibility
        result = subprocess.run([
            "python", "scripts/verify-reproducible.py",
            str(tmpdir / "build1" / "wheels"),
            str(tmpdir / "build2" / "wheels")
        ], capture_output=True, text=True)
        
        return {
            "timestamp": datetime.now().isoformat(),
            "reproducible": result.returncode == 0,
            "output": result.stdout,
            "errors": result.stderr
        }

def main():
    """Run monitoring and store results."""
    
    results_file = Path("reproducibility-history.json")
    
    # Load existing results
    if results_file.exists():
        with open(results_file) as f:
            history = json.load(f)
    else:
        history = []
    
    # Run test
    result = run_reproducibility_test()
    history.append(result)
    
    # Save results
    with open(results_file, "w") as f:
        json.dump(history, f, indent=2)
    
    # Report status
    if result["reproducible"]:
        print("✅ Reproducibility test passed")
    else:
        print("❌ Reproducibility test failed")
        print(result["output"])
        
    # Keep only last 30 results
    if len(history) > 30:
        history = history[-30:]
        with open(results_file, "w") as f:
            json.dump(history, f, indent=2)

if __name__ == "__main__":
    main()
```

## Best Practices

1. **Version locking** - Pin exact versions of all dependencies
2. **Environment standardization** - Use containers or consistent setup scripts
3. **Source archival** - Archive exact source code versions
4. **Hash verification** - Track and verify file hashes across builds
5. **Regular testing** - Continuously test reproducibility
6. **Documentation** - Document reproducibility requirements and processes
7. **Toolchain stability** - Pin build tools and compilers
8. **Minimal builds** - Reduce non-deterministic elements in build processes

Reproducible builds provide confidence in your software supply chain and enable reliable debugging and compliance processes.
