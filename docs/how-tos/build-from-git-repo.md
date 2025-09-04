# Building from Git Repository

Sometimes you need to build packages from a Git repository instead of published source distributions. This is useful for:

- Development versions not yet released
- Patched versions with custom modifications  
- Packages that don't publish to PyPI
- Testing unreleased features

## Basic Git Source Configuration

Configure a package to build from Git using package settings:

```yaml title="overrides/settings/mypackage.yaml"
download_source:
  url: "https://github.com/owner/mypackage.git"
  git_options:
    tag: "v1.2.3"

changelog:
  - "Building from Git repository at tag v1.2.3"
```

## Git Options

### Building from Tags

```yaml title="overrides/settings/torch.yaml"
download_source:
  url: "https://github.com/pytorch/pytorch.git"
  git_options:
    tag: "v2.1.0"
    recursive_submodules: true

changelog:
  - "Building PyTorch from Git tag v2.1.0"
```

### Building from Branches

```yaml title="overrides/settings/mypackage.yaml"
download_source:
  url: "https://github.com/owner/mypackage.git"
  git_options:
    branch: "main"

changelog:
  - "Building from main branch"
```

### Building from Specific Commits

```yaml title="overrides/settings/mypackage.yaml"
download_source:
  url: "https://github.com/owner/mypackage.git"
  git_options:
    commit: "a1b2c3d4e5f6"

changelog:
  - "Building from specific commit a1b2c3d4e5f6"
```

## Handling Submodules

Many packages require Git submodules:

```yaml title="overrides/settings/opencv.yaml"
download_source:
  url: "https://github.com/opencv/opencv-python.git"
  git_options:
    tag: "v4.8.0"
    recursive_submodules: true

changelog:
  - "Building OpenCV with all submodules"
```

## Private Repositories

### Using SSH Keys

For private repositories, configure SSH access:

```yaml title="overrides/settings/private-package.yaml"
download_source:
  url: "git@github.com:company/private-package.git"
  git_options:
    tag: "v1.0.0"

changelog:
  - "Building from private repository"
```

### Using Personal Access Tokens

```yaml title="overrides/settings/private-package.yaml"
download_source:
  url: "https://token:${GITHUB_TOKEN}@github.com/company/private-package.git"
  git_options:
    tag: "v1.0.0"

env:
  GITHUB_TOKEN: "${GITHUB_TOKEN}"

changelog:
  - "Building from private repository with token auth"
```

## Complex Git Configurations

### Multiple Remotes

```yaml title="overrides/settings/forked-package.yaml"
download_source:
  url: "https://github.com/myorg/forked-package.git"
  git_options:
    branch: "custom-features"
    upstream_url: "https://github.com/upstream/original-package.git"

build_options:
  pre_build_commands:
    - "git remote add upstream ${upstream_url}"
    - "git fetch upstream"

changelog:
  - "Building from fork with upstream tracking"
```

### Shallow Clones

For faster clones of large repositories:

```yaml title="overrides/settings/large-package.yaml"
download_source:
  url: "https://github.com/owner/large-package.git"
  git_options:
    tag: "v2.0.0"
    depth: 1

changelog:
  - "Using shallow clone for faster download"
```

## Requirements File Integration

Specify Git sources directly in requirements files:

```text title="requirements.txt"
# Regular PyPI packages
requests>=2.28.0

# Git repository with tag
mypackage @ git+https://github.com/owner/mypackage.git@v1.2.3

# Git repository with branch
devpackage @ git+https://github.com/owner/devpackage.git@main

# Git repository with commit
testpackage @ git+https://github.com/owner/testpackage.git@a1b2c3d4
```

Then bootstrap normally:

```bash
fromager bootstrap --requirements-file requirements.txt
```

## Build Hooks for Git Sources

### Custom Clone Operations

```python title="package_plugins/git_package.py"
def download_source(req_type, package_name, version, sdist_server_url, patches_dir):
    """Custom Git clone with post-processing."""
    import subprocess
    import os
    
    # Clone repository
    subprocess.run([
        "git", "clone", 
        "https://github.com/owner/package.git",
        f"{package_name}-{version}"
    ], check=True)
    
    # Switch to specific tag/branch
    os.chdir(f"{package_name}-{version}")
    subprocess.run(["git", "checkout", version], check=True)
    
    # Initialize submodules
    subprocess.run([
        "git", "submodule", "update", 
        "--init", "--recursive"
    ], check=True)
    
    # Apply custom patches
    if patches_dir:
        for patch in sorted(patches_dir.glob("*.patch")):
            subprocess.run([
                "git", "apply", str(patch)
            ], check=True)
```

### Build System Preparation

```python title="package_plugins/cmake_package.py"
def prepare_source(req_type, package_name, version, source_dir):
    """Prepare CMake-based Git package."""
    import subprocess
    import os
    
    os.chdir(source_dir)
    
    # Generate build files
    subprocess.run([
        "cmake", "-B", "build", 
        "-DCMAKE_BUILD_TYPE=Release",
        "-DPYTHON_EXECUTABLE=" + sys.executable
    ], check=True)
```

## Troubleshooting Git Sources

### Authentication Issues

**Problem**: Git clone fails with authentication error.

**Solution**: Set up proper credentials:

```bash
# SSH key authentication
ssh-add ~/.ssh/id_rsa

# Token authentication  
export GITHUB_TOKEN="your-token-here"

# Git credential helper
git config --global credential.helper store
```

### Submodule Problems

**Problem**: Build fails because submodules are missing.

**Solution**: Ensure recursive cloning:

```yaml
git_options:
  recursive_submodules: true
  submodule_init_commands:
    - "git submodule update --init --recursive --depth 1"
```

### Large Repository Performance  

**Problem**: Git clone takes too long for large repositories.

**Solutions**:

```yaml
# Shallow clone
git_options:
  depth: 1
  
# Sparse checkout
git_options:
  sparse_checkout_paths:
    - "src/"
    - "setup.py"
    - "pyproject.toml"
```

### Version Mismatch

**Problem**: Git tag doesn't match expected package version.

**Solution**: Override version detection:

```yaml title="overrides/settings/package.yaml"
download_source:
  url: "https://github.com/owner/package.git"
  git_options:
    tag: "release-1.2.3"

project_override:
  version: "1.2.3"

changelog:
  - "Override version to match Git tag naming"
```

## CI/CD Integration

### Automated Git Builds

```yaml title=".github/workflows/build-git-packages.yml"
name: Build from Git Sources
on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          
      - name: Install fromager
        run: pip install fromager
        
      - name: Configure Git
        run: |
          git config --global user.name "CI Bot"
          git config --global user.email "ci@example.com"
          
      - name: Build packages
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          fromager bootstrap \
            --requirements-file requirements.txt \
            --wheel-server-dir wheels
            
      - name: Upload wheels
        uses: actions/upload-artifact@v4
        with:
          name: wheels
          path: wheels/
```

### Git Source Monitoring

Track upstream changes and trigger builds:

```python title="scripts/monitor-git-sources.py"
#!/usr/bin/env python3
"""Monitor Git sources for new releases."""

import requests
import yaml
from pathlib import Path

def check_new_releases():
    """Check for new Git tags in configured repositories."""
    
    settings_dir = Path("overrides/settings")
    
    for settings_file in settings_dir.glob("*.yaml"):
        with open(settings_file) as f:
            config = yaml.safe_load(f)
            
        download_source = config.get("download_source", {})
        git_url = download_source.get("url", "")
        
        if "github.com" in git_url:
            # Extract owner/repo from URL
            parts = git_url.replace(".git", "").split("/")
            owner, repo = parts[-2], parts[-1]
            
            # Check for new releases
            api_url = f"https://api.github.com/repos/{owner}/{repo}/releases/latest"
            response = requests.get(api_url)
            
            if response.status_code == 200:
                latest = response.json()
                current_tag = download_source.get("git_options", {}).get("tag")
                latest_tag = latest["tag_name"]
                
                if latest_tag != current_tag:
                    print(f"New release available for {settings_file.stem}:")
                    print(f"  Current: {current_tag}")
                    print(f"  Latest:  {latest_tag}")

if __name__ == "__main__":
    check_new_releases()
```

## Best Practices

1. **Pin to specific tags or commits** - Avoid branches for reproducible builds
2. **Use shallow clones** - Speed up downloads for large repositories
3. **Handle submodules carefully** - Ensure all dependencies are available
4. **Set up proper authentication** - Use SSH keys or tokens for private repos
5. **Monitor upstream changes** - Track new releases and security updates
6. **Test thoroughly** - Git sources may have different build requirements
7. **Document rationale** - Explain why Git sources are used instead of releases
8. **Cache repositories** - Use local mirrors for frequently accessed repos

Building from Git repositories gives you flexibility to use cutting-edge features and apply custom patches, but requires careful configuration and monitoring to maintain reliable builds.
