# Working with Pre-release Versions

Pre-release versions (alpha, beta, release candidates) are often needed for testing new features, getting early access to bug fixes, or working with cutting-edge development. Fromager provides several ways to handle pre-release packages.

## Understanding Pre-release Versions

### Version Types

- **Alpha** (`1.0.0a1`, `1.0.0a2`) - Early development versions
- **Beta** (`1.0.0b1`, `1.0.0b2`) - Feature-complete but may have bugs
- **Release Candidates** (`1.0.0rc1`, `1.0.0rc2`) - Nearly final versions
- **Development** (`1.0.0.dev1`) - Ongoing development snapshots

### Version Ordering

Python follows PEP 440 for version ordering:

```
1.0.0.dev1 < 1.0.0a1 < 1.0.0b1 < 1.0.0rc1 < 1.0.0
```

## Enabling Pre-release Support

### Global Pre-release Configuration

Allow pre-releases globally in constraints:

```text title="constraints-prerelease.txt"
# Allow pre-releases for all packages
--pre

# Or specify pre-release for specific packages
torch>=2.2.0a0
tensorflow>=2.15.0b1
```

### Package-Specific Pre-release Settings

Enable pre-releases for specific packages:

```yaml title="overrides/settings/torch.yaml"
resolver_dist:
  include_prerelease: true

changelog:
  - "Enabled pre-release versions for PyTorch"
```

### Requirements File Pre-releases

Specify pre-release versions directly:

```text title="requirements-dev.txt"
# Explicit pre-release versions
torch==2.2.0a1+cpu
transformers==4.36.0.dev0

# Allow any pre-release
numpy>=1.26.0a1

# Mixed stable and pre-release
requests>=2.31.0
pandas>=2.1.0b1
```

## Building Pre-release Packages

### Bootstrap with Pre-releases

```bash
# Enable pre-releases globally
fromager bootstrap \
  --requirements-file requirements-dev.txt \
  --constraints-file constraints-prerelease.txt

# Or use pip-style pre-release flag
echo "--pre" > constraints.txt
fromager bootstrap \
  --requirements-file requirements.txt \
  --constraints-file constraints.txt
```

### Variant-Based Pre-release Builds

Create variants for stable vs pre-release builds:

```yaml title="overrides/settings/torch.yaml"
variants:
  stable:
    changelog:
      - "Stable release build"
    resolver_dist:
      include_prerelease: false
      
  preview:
    changelog:
      - "Pre-release/preview build"
    resolver_dist:
      include_prerelease: true
```

Build different variants:

```bash
# Stable build
fromager --variant stable bootstrap torch

# Pre-release build  
fromager --variant preview bootstrap torch
```

## Version Selection Strategies

### Pinning Specific Pre-releases

```yaml title="overrides/settings/experimental-package.yaml"
download_source:
  version: "2.0.0b3"
  
changelog:
  - "Pinned to beta 3 for stability testing"
```

### Version Ranges with Pre-releases

```text title="constraints-ml-dev.txt"
# Allow betas but not alphas
torch>=2.1.0b1,<2.2.0a0

# Allow any 2.0 pre-releases
tensorflow>=2.0.0a1,<2.1.0

# Stable dependencies with pre-release ML stack
numpy>=1.24.0,<2.0.0
pandas>=2.0.0,<3.0.0
scikit-learn>=1.3.0,<2.0.0
```

## Handling Pre-release Dependencies

### Dependency Resolution with Pre-releases

```yaml title="overrides/settings/transformers.yaml"
# Package that depends on pre-release PyTorch
resolver_dist:
  include_prerelease: true
  prerelease_dependencies:
    - torch
    - torchvision

changelog:
  - "Allow pre-release dependencies for compatibility"
```

### Mixed Stable and Pre-release Stacks

```text title="requirements-mixed.txt"
# Core libraries - stable versions
numpy>=1.24.0
pandas>=2.0.0
requests>=2.31.0

# ML libraries - allow pre-releases
torch>=2.1.0a0
transformers>=4.35.0b0
accelerate>=0.24.0.dev0

# Development tools - latest pre-releases OK
pytest>=7.4.0
black>=23.9.0a0
```

## CI/CD with Pre-releases

### Separate Pre-release Pipelines

```yaml title=".github/workflows/prerelease.yml"
name: Pre-release Testing
on:
  schedule:
    - cron: '0 6 * * *'  # Daily at 6 AM
  workflow_dispatch:

jobs:
  test-prerelease:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          
      - name: Build pre-release stack
        run: |
          echo "--pre" > constraints.txt
          fromager bootstrap \
            --requirements-file requirements.txt \
            --constraints-file constraints.txt \
            --wheel-server-dir wheels-prerelease
            
      - name: Test pre-release builds
        run: |
          pip install --find-links wheels-prerelease \
            --pre torch transformers
          python -c "
import torch
import transformers
print(f'PyTorch: {torch.__version__}')  
print(f'Transformers: {transformers.__version__}')
"
          
      - name: Run integration tests
        run: |
          python -m pytest tests/ -k "prerelease"
```

### Matrix Testing Stable vs Pre-release

```yaml title=".github/workflows/version-matrix.yml"
name: Version Matrix Testing
on: push

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        version-type: [stable, prerelease]
        include:
          - version-type: stable
            constraints: constraints-stable.txt
            
          - version-type: prerelease
            constraints: constraints-prerelease.txt
            
    steps:
      - uses: actions/checkout@v4
      
      - name: Build ${{ matrix.version-type }} stack
        run: |
          fromager bootstrap \
            --requirements-file requirements.txt \
            --constraints-file ${{ matrix.constraints }}
```

## Monitoring Pre-release Availability

### Automated Pre-release Detection

```python title="scripts/check-prereleases.py"
#!/usr/bin/env python3
"""Check for new pre-release versions."""

import requests
import json
from packaging import version

def check_prereleases(package_name):
    """Check PyPI for pre-release versions."""
    
    url = f"https://pypi.org/pypi/{package_name}/json"
    response = requests.get(url)
    
    if response.status_code != 200:
        return []
    
    data = response.json()
    releases = data["releases"]
    
    prereleases = []
    for ver, files in releases.items():
        if files and version.parse(ver).is_prerelease:
            prereleases.append(ver)
    
    # Sort by version
    prereleases.sort(key=version.parse, reverse=True)
    return prereleases[:5]  # Latest 5 pre-releases

def main():
    packages = ["torch", "tensorflow", "transformers", "numpy"]
    
    for pkg in packages:
        prereleases = check_prereleases(pkg)
        if prereleases:
            print(f"{pkg}: {', '.join(prereleases)}")

if __name__ == "__main__":
    main()
```

### Pre-release Update Notifications

```python title="scripts/prerelease-notifier.py"
#!/usr/bin/env python3
"""Notify about new pre-releases."""

import json
import smtplib
from email.mime.text import MimeText
from datetime import datetime, timedelta

def check_and_notify():
    """Check for new pre-releases and send notifications."""
    
    # Load previous check results
    try:
        with open("prerelease-cache.json") as f:
            previous = json.load(f)
    except FileNotFoundError:
        previous = {}
    
    # Check current pre-releases
    current = {}
    packages = ["torch", "tensorflow", "transformers"]
    
    for pkg in packages:
        current[pkg] = check_prereleases(pkg)
    
    # Find new pre-releases
    new_releases = {}
    for pkg, versions in current.items():
        prev_versions = set(previous.get(pkg, []))
        new_versions = [v for v in versions if v not in prev_versions]
        if new_versions:
            new_releases[pkg] = new_versions
    
    # Send notifications
    if new_releases:
        send_notification(new_releases)
    
    # Save current state
    with open("prerelease-cache.json", "w") as f:
        json.dump(current, f)

def send_notification(new_releases):
    """Send email notification about new pre-releases."""
    
    subject = f"New Pre-releases Available - {datetime.now().strftime('%Y-%m-%d')}"
    
    body = "New pre-release versions detected:\n\n"
    for pkg, versions in new_releases.items():
        body += f"{pkg}:\n"
        for version in versions:
            body += f"  - {version}\n"
        body += "\n"
    
    body += "Consider updating your pre-release builds."
    
    # Send email (configure SMTP settings)
    # ... email sending logic here

if __name__ == "__main__":
    check_and_notify()
```

## Testing Pre-release Stability

### Automated Testing Pipeline

```bash title="scripts/test-prerelease-stability.sh"
#!/bin/bash

# Test pre-release packages for stability
PACKAGES=("torch" "transformers" "accelerate")

for pkg in "${PACKAGES[@]}"; do
    echo "Testing pre-release stability for $pkg"
    
    # Build with latest pre-release
    fromager build \
        --package "$pkg" \
        --allow-prerelease \
        --wheel-server-dir "wheels-test"
    
    # Install and run basic tests
    pip install --find-links wheels-test --pre "$pkg"
    
    # Run package-specific tests
    python -c "
import $pkg
print(f'$pkg version: {$pkg.__version__}')

# Basic functionality test
try:
    # Package-specific smoke tests
    if '$pkg' == 'torch':
        import torch
        x = torch.randn(2, 2)
        print('PyTorch basic test: OK')
    elif '$pkg' == 'transformers':
        from transformers import pipeline
        print('Transformers import: OK')
except Exception as e:
    print(f'Test failed: {e}')
    exit(1)
"
    
    echo "$pkg pre-release test completed"
done
```

### Regression Testing

```python title="tests/test_prerelease_regression.py"
"""Regression tests for pre-release versions."""

import pytest
import importlib
import subprocess

PRERELEASE_PACKAGES = [
    "torch",
    "transformers", 
    "accelerate",
]

@pytest.mark.parametrize("package", PRERELEASE_PACKAGES)
def test_prerelease_import(package):
    """Test that pre-release packages can be imported."""
    try:
        importlib.import_module(package)
    except ImportError as e:
        pytest.fail(f"Failed to import {package}: {e}")

@pytest.mark.parametrize("package", PRERELEASE_PACKAGES)  
def test_prerelease_basic_functionality(package):
    """Test basic functionality of pre-release packages."""
    
    if package == "torch":
        import torch
        # Basic tensor operations
        x = torch.randn(2, 2)
        y = torch.randn(2, 2)
        z = torch.matmul(x, y)
        assert z.shape == (2, 2)
        
    elif package == "transformers":
        from transformers import AutoTokenizer
        # Basic tokenizer test
        tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
        tokens = tokenizer("Hello world")
        assert "input_ids" in tokens

def test_prerelease_compatibility():
    """Test compatibility between pre-release packages."""
    
    try:
        import torch
        import transformers
        
        # Test basic compatibility
        model_name = "bert-base-uncased"
        from transformers import AutoModel
        model = AutoModel.from_pretrained(model_name)
        
        # Ensure model works with current PyTorch
        inputs = torch.randint(0, 1000, (1, 10))
        outputs = model(inputs)
        assert outputs is not None
        
    except Exception as e:
        pytest.fail(f"Compatibility test failed: {e}")
```

## Best Practices

1. **Separate environments** - Use different directories/environments for pre-release builds
2. **Version pinning** - Pin specific pre-release versions for reproducibility  
3. **Extensive testing** - Pre-releases may have breaking changes or bugs
4. **Monitoring** - Track new pre-release availability automatically
5. **Fallback plans** - Have stable versions ready if pre-releases fail
6. **Documentation** - Document why pre-releases are needed and their risks
7. **Communication** - Inform team members about pre-release usage
8. **Regular updates** - Keep pre-release versions current as new ones are released

Pre-release versions provide access to the latest features but require careful handling and thorough testing to ensure stability in your builds.
